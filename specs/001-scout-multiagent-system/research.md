# Research: Scout Multi-Agent System

**Phase**: 0 — Pre-design research
**Branch**: `001-scout-multiagent-system`
**Date**: 2026-03-11

---

## 1. Agent Framework: CrewAI Sequential Pipeline

**Decision**: CrewAI `~0.80.0` with `Process.sequential`

**Rationale**: Sequential process is the simplest execution model for a fixed Analyst → Researcher → Strategist → Specifier pipeline. Each task completes fully before the next begins, which is required when downstream agents depend on upstream output. No orchestrator LLM is needed (unlike `Process.hierarchical`), reducing cost and latency.

**Alternatives considered**:
- `Process.hierarchical`: Uses a manager LLM to dynamically delegate tasks — adds non-determinism and overhead. Not appropriate for a fixed 4-step pipeline.
- CrewAI Flows (`@crew` decorated): Better for complex branching pipelines with conditional routing between crews. Upgrade path if stages need independent async triggers or timeouts.

---

## 2. Context Passing Between Agents

**Decision**: Use `context=[upstream_task]` on each downstream `Task`. Use `output_pydantic=<Model>` on tasks for structured handoffs.

**Rationale**: The `context` list injects upstream task string output into the next task's prompt automatically — no manual output threading. `output_pydantic` enforces a schema on the LLM's response, making downstream parsing deterministic. This is critical when the Strategist must consume structured Researcher findings.

**Alternatives considered**:
- Plain `expected_output` string descriptions: Less reliable with local LLMs; no schema enforcement.
- Shared state via a custom tool (Redis/file): Adds infrastructure; bypasses CrewAI's built-in context plumbing.
- CrewAI Flows `state` objects: More explicit but more boilerplate for a linear pipeline.

---

## 3. LLM Integration: Ollama via LiteLLM

**Decision**: `LLM(model="ollama/qwen2.5:14b", base_url="http://localhost:11434")` assigned to each Agent's `llm` parameter.

**Rationale**: CrewAI delegates all LLM calls to LiteLLM since ~0.60.x. LiteLLM natively supports Ollama via the `ollama/<model>` prefix — no custom wrapper needed. Setting `base_url` avoids accidental calls to OpenAI endpoints.

**Note on model name**: The plan specifies `qwen3.5:9b`. As of the August 2025 knowledge cutoff, the stable Ollama model is `qwen2.5` (7B/14B variants). `qwen3` may have been released by March 2026. Use the exact model tag as specified (`qwen3.5:9b` or whatever is available in the local Ollama registry) — the LiteLLM provider prefix `ollama/` works for any model tag.

**Alternatives considered**:
- `langchain_community.llms.Ollama` via legacy `llm` parameter: Deprecated path in 0.80.x.
- `llama.cpp` with OpenAI-compatible server (`openai/` prefix): Works but adds another local server process.

---

## 4. Async Execution

**Decision**: `crew.kickoff_async()` at the CrewAI level; `asyncio.create_task()` at the FastAPI job management level.

**Rationale**: `kickoff_async()` is the official async entry point (supported since ~0.70.x). At the application layer, `asyncio.create_task` creates a `Task` object with a reference that can track state (running → completed/failed) — unlike `BackgroundTasks` which is fire-and-forget with no handle. Must keep a strong reference to the task (store in a dict/set) to prevent GC before completion.

**Alternatives considered**:
- `asyncio.to_thread()` wrapping synchronous `kickoff()`: Works but bypasses native async plumbing; use only if blocking I/O forces it.
- Celery + Redis: Production-grade for multi-worker distribution. Out of scope for single-user service.
- FastAPI `BackgroundTasks`: Designed for lightweight side effects, not trackable long-running jobs.

---

## 5. HTTP Framework and Job Pattern

**Decision**: FastAPI with the submit-and-poll pattern:
- `POST /jobs` (multipart/form-data) → returns `job_id` (HTTP 202)
- `GET /jobs/{id}/status` → returns `{status, agent_statuses, timestamps}`
- `GET /jobs/{id}/result` → returns markdown report when completed

**Rationale**: Decouples submission from execution; no persistent connection required. `UploadFile` (not `bytes`) handles PDF uploads efficiently — buffers small files in memory, spills to disk for large files. File must be saved to disk before enqueueing so the background task has a stable path.

**Key constraint**: File uploads use `multipart/form-data`. Additional parameters (github_username) must be form fields, not JSON body.

**Alternatives considered**:
- SSE (Server-Sent Events): Better UX for real-time progress but more stateful.
- WebSockets: Overkill for one-way push.
- Webhooks (callback URL): Good for server-to-server; not needed for direct API consumers.

---

## 6. Job Store

**Decision**: In-memory `dict[str, JobRecord]` for initial implementation. SQLite via `aiosqlite` as the upgrade path if crash-recovery or job history is needed.

**Rationale**: Zero setup, zero latency, no serialization overhead. For a single-user service where jobs can be resubmitted after crashes, this is the right tradeoff. Writes from the background coroutine to the dict are safe without locks since asyncio is single-threaded. If `run_in_executor` is used for blocking PDF parsing, use `call_soon_threadsafe` to update the job store.

**Alternatives considered**:
- Redis: Requires a separate process; overengineered for single-user.
- SQLite (immediate): Adds persistence from day one. Adopt when cross-restart visibility becomes a requirement.

---

## 7. GitHub API Client

**Decision**: `httpx` (async) with direct REST calls. Authenticated with a Personal Access Token (PAT).

**Rationale**: Async concurrency is significant when fetching per-repo data in parallel (e.g., languages + topics for 30+ repos). PyGithub is synchronous-only and would bottleneck sequential calls. Unauthenticated rate limit (60 req/hr) is exhausted almost immediately on users with >10 repos. PAT provides 5,000 req/hr.

**Endpoints used**:
- `GET /users/{username}` — profile metadata
- `GET /users/{username}/repos?per_page=100` — repository list
- `GET /repos/{owner}/{repo}/languages` — language bytes
- `GET /repos/{owner}/{repo}/topics` — repository topics
- `GET /repos/{owner}/{repo}/stats/contributors` — commit activity (may return 202; must retry)

**Alternatives considered**:
- PyGithub: Synchronous, handles pagination automatically. Right choice for simple scripts; too slow for concurrent pipeline use.

---

## 8. PDF Text Extraction

**Decision**: PyMuPDF (`fitz`) for CV text extraction.

**Rationale**: CVs frequently use multi-column layouts. PyMuPDF's coordinate-aware block extraction (`page.get_text("blocks")`) enables reading-order reconstruction by sorting blocks by x/y position. Native table detection since 1.23+ (`page.find_tables()`). Fastest of the options (C binding to MuPDF). Image-only detection: if `page.get_text()` returns empty and `len(page.get_images()) > 0`, the PDF is image-only — return a `422 Unprocessable Entity` with a clear message.

**Alternatives considered**:
- pdfplumber: Better table extraction for structured documents; slower (pure Python). Good fallback if PyMuPDF table detection proves insufficient.
- pypdf: Pure Python, lightweight, but poor multi-column handling (text extracted in PDF object order). Avoid for CV parsing.

---

## 9. Market Data Sources

**Decision**: Three-source combination — GitHub Search API + HN Firebase/Algolia API + DuckDuckGo (Researcher agent tool).

**Rationale**:
- **GitHub Search API** (`/search/repositories?q=topic:llm+pushed:>DATE&sort=stars`): Official, free, PAT-authenticated (30 req/min). Primary source for trending repos by topic + recency. Simulates `github.com/trending` by querying `pushed:>7-days-ago` sorted by stars.
- **HN Firebase API + Algolia**: Official, free, no auth. Monthly "Who is Hiring?" threads provide qualitative AI/ML hiring signal. Parse with LLM (trivial with an agent).
- **DuckDuckGo** (`duckduckgo-search` Python library): The Researcher agent's "escape hatch" for real-time web search. No API key required. Free, no rate limit enforcement beyond reasonable use. Covers anything not in structured APIs — new tool releases, salary discussions, conference announcements. CrewAI has built-in DuckDuckGo tool support via `crewai-tools`.

**Rejected sources**:
- Brave Search API: Replaced by DuckDuckGo (user preference; DuckDuckGo requires no API key and is simpler to set up).
- LinkedIn API: Enterprise partnership required; not accessible.
- Indeed API: Deprecated to new applicants since 2022–2023.
- github-trending-api scrapers: Fragile HTML scraping; use Search API instead.
- levels.fyi: No API; ToS-violating to scrape.
- SerpAPI (`google_jobs`): $75/month; not justified for single-user service.

**Optional addition**:
- Adzuna API (free tier, 50 req/day): Quantitative job posting counts by role/technology. Worth adding if job volume metrics are needed.

---

## 10. Observability / Pipeline Tracking

**Decision**: `mlflow.autolog()` called once at service startup — no manual MLflow logging calls anywhere in the codebase.

**Rationale**: FR-013 explicitly prohibits manual MLflow logging (`mlflow.log_metric`, `mlflow.log_param`, `mlflow.log_text`, etc.). `mlflow.autolog()` captures LiteLLM/OpenAI-compatible LLM call traces automatically. In local mode it writes to a file-based SQLite store under `./mlruns` with no server required.

**Pattern**:
```python
# scout/tracking/mlflow.py — called once in app startup
import mlflow

def init_tracking(tracking_uri: str = "./mlruns") -> None:
    mlflow.set_tracking_uri(tracking_uri)
    mlflow.autolog()   # Only MLflow call permitted by FR-013
```

**Coverage caveat**: MLflow autolog coverage of CrewAI/LiteLLM 0.80.x is not guaranteed for all event types. If autolog silently omits a call, that is acceptable — observability is best-effort and must not couple to pipeline correctness. Do not add manual logging to fill gaps.

**Alternatives considered**:
- Phoenix (Arize): In-process, OpenTelemetry-native, purpose-built for LLM traces. Best alternative if autolog proves insufficient.
- Langfuse (self-hosted): Excellent LLM observability UI; requires Docker + Postgres. Upgrade path for team/production use.
- Structured SQLite logging: Minimum viable; no overhead. Consider if MLflow autolog adds measurable latency.

---

## 11. Code Quality

**Decision**: `ruff` for linting and formatting (replaces flake8 + black + isort).

**Rationale**: Single fast tool covering formatting, import sorting, and lint checks. Specified as part of the tech stack. Configure via `pyproject.toml`.

---

## 12. Docker Compose Deployment

**Decision**: Ship `docker-compose.yml` at repo root with three Ollama hardware profiles (`cpu`, `gpu-amd`, `gpu-nvidia`) plus the Scout API service. Default documented profile: `cpu`.

**Rationale**: Provides a one-command full-stack launch regardless of hardware. The Ollama service configuration (YAML anchors, profile variants) was provided directly by the project owner and is treated as a fixed constraint, not a design decision.

**Scout API service additions to `docker-compose.yml`**:
```yaml
scout-api:
  build: .
  ports:
    - "8000:8000"
  environment:
    - OLLAMA_HOST=http://ollama:11434
    - OLLAMA_MODEL=${OLLAMA_MODEL:-qwen2.5:9b}
    - GITHUB_TOKEN=${GITHUB_TOKEN}
    - MLFLOW_TRACKING_URI=./mlruns
    - UPLOAD_DIR=./uploads
  networks:
    default:
      aliases:
        - scout-api
```

**Profile-specific `depends_on`**: The `scout-api` service must declare `depends_on` per profile using Docker Compose conditional expressions, or three profile-specific `scout-api` variants (matching the Ollama pattern). Simplest approach: use a single `scout-api` with no hard `depends_on` and rely on the init container's `sleep 3` to ensure Ollama is ready.

**Dockerfile**: Multi-stage build — builder stage runs `uv sync --no-dev`, runtime stage copies `.venv` and source. Entrypoint: `uv run uvicorn scout.api.main:app --host 0.0.0.0 --port 8000`.

**First-time usage**:
```bash
docker compose --profile cpu up -d        # CPU (default)
docker compose --profile gpu-nvidia up -d # Nvidia GPU
docker compose --profile gpu-amd up -d    # AMD GPU
```

---

## Resolution Summary

| Topic | Resolved Decision |
|---|---|
| Agent framework | CrewAI ~0.80.0, Process.sequential |
| Context passing | `Task.context` + `output_pydantic` |
| LLM | Ollama via LiteLLM (`ollama/qwen2.5:14b` or specified `qwen3.5:9b`) |
| Async | `kickoff_async()` + `asyncio.create_task` |
| HTTP framework | FastAPI, submit-and-poll, multipart upload |
| Job store | In-memory dict → SQLite upgrade path |
| GitHub client | httpx async + PAT |
| PDF extraction | PyMuPDF (fitz) |
| Market data | GitHub Search API + HN API + DuckDuckGo (`duckduckgo-search`) |
| Observability | MLflow `autolog()` only (FR-013) |
| Code quality | ruff |
| Deployment | Docker Compose, `cpu`/`gpu-amd`/`gpu-nvidia` profiles, default `cpu` |
| Default LLM model | `qwen2.5:9b` via `OLLAMA_MODEL` env var |
