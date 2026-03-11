# Implementation Plan: Scout — Multi-Agent Portfolio Opportunity Finder

**Branch**: `001-scout-multiagent-system` | **Date**: 2026-03-11 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-scout-multiagent-system/spec.md`

## Summary

Scout is a FastAPI web service that orchestrates four sequential CrewAI agents (Analyst → Researcher → Strategist → Specifier) to analyze a developer's GitHub profile and uploaded CV PDF, producing a structured markdown report with three prioritized project ideas and full mini PRDs. The pipeline runs asynchronously behind a submit-and-poll HTTP API. All agents share a single Ollama LLM instance (`qwen2.5:9b` via LiteLLM). The system ships as a Docker Compose application with CPU, AMD GPU, and Nvidia GPU hardware profiles.

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**: FastAPI 0.110+, CrewAI ~0.80.0, LiteLLM (bundled with CrewAI), httpx, PyMuPDF (fitz), MLflow, duckduckgo-search, crewai-tools, ruff, uv
**Storage**: In-memory `dict[str, AnalysisJob]` with `threading.RLock`; CV files saved to `UPLOAD_DIR` on disk
**Testing**: pytest + pytest-asyncio
**Target Platform**: Linux server via Docker Compose; local dev via `uv run uvicorn`
**Project Type**: web-service
**Performance Goals**: Full pipeline ≤15 min (SC-001); status update ≤10 s of state transition (SC-005); validation error response ≤5 s (SC-006)
**Constraints**: Localhost only, no authentication, no database, single-user/low-concurrency; MLflow via `autolog()` only — no manual logging (FR-013)
**Scale/Scope**: Single-user service; no multi-tenancy; no persistent job history across restarts

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

The constitution file (`/.specify/memory/constitution.md`) is an unfilled template — no project-specific gates are defined. No violations to evaluate.

**Post-design check**: All design decisions align with spec requirements and clarifications. No architectural complexity beyond what the problem warrants (single project, in-memory store, direct service calls — no extra abstractions added).

## Project Structure

### Documentation (this feature)

```text
specs/001-scout-multiagent-system/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── http-api.md      # Phase 1 output
└── tasks.md             # Phase 2 output (/speckit.tasks — NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
src/scout/
├── api/
│   ├── main.py           # FastAPI app factory; calls mlflow.autolog() at startup
│   ├── routes/
│   │   ├── jobs.py       # POST /jobs, GET /jobs/{id}/status, GET /jobs/{id}/result, DELETE /jobs/{id}
│   │   └── health.py     # GET /health (Ollama + MLflow liveness)
│   └── deps.py           # Shared FastAPI dependencies (job store, services)
├── agents/
│   ├── analyst.py        # Analyst Agent (GitHub + PDF tools)
│   ├── researcher.py     # Researcher Agent (DuckDuckGo + GitHub Search + HN tools)
│   ├── strategist.py     # Strategist Agent
│   └── specifier.py      # Specifier Agent
├── pipeline/
│   ├── crew.py           # Crew assembly (Process.sequential), kickoff_async wrapper, agent status updates
│   └── tasks.py          # CrewAI Task definitions with context + output_pydantic
├── models/
│   ├── job.py            # AnalysisJob, JobStatus, AgentStatus, AgentName enums
│   ├── profile.py        # UserProfile, LanguageStat, CommitActivity, SkillGap, GapSource
│   ├── market.py         # MarketSnapshot, TrendingRepo, HiringSignal, WebInsight
│   ├── ideas.py          # ProjectIdea
│   ├── prd.py            # ProjectPRD
│   └── report.py         # ScoutReport
├── services/
│   ├── github.py         # httpx async GitHub REST client with exponential backoff (FR-016)
│   ├── pdf.py            # PyMuPDF CV extraction; image-only + encrypted PDF detection
│   └── market.py         # DuckDuckGo + GitHub Search + HN Firebase data gathering; degradation handling
├── store/
│   └── jobs.py           # In-memory JobStore with threading.RLock
└── tracking/
    └── mlflow.py         # mlflow.autolog() initialization (called once at startup)

tests/
├── unit/
│   ├── test_github_service.py   # Retry logic, 404 vs 403/429 handling
│   ├── test_pdf_service.py      # Text extraction, image-only, encrypted detection
│   ├── test_market_service.py   # Degradation handling when sources unreachable
│   ├── test_job_store.py        # Thread-safe reads/writes
│   └── test_models.py           # Pydantic model validation and constraints
├── integration/
│   ├── test_pipeline.py         # Full crew execution against real Ollama (requires OLLAMA_HOST)
│   └── test_api_jobs.py         # FastAPI TestClient full job lifecycle
└── contract/
    └── test_http_api.py         # HTTP contract tests (request/response shapes, status codes)

docker-compose.yml               # Scout API + Ollama (cpu / gpu-amd / gpu-nvidia profiles)
Dockerfile                       # Multi-stage uv build; entrypoint: uvicorn
.env.example                     # GITHUB_TOKEN, OLLAMA_HOST, OLLAMA_MODEL, MLFLOW_TRACKING_URI, UPLOAD_DIR
pyproject.toml                   # uv project, ruff config (line-length=100, py311 target), pytest config
```

**Structure Decision**: Single-project layout under `src/scout/`. Module boundaries map 1:1 to functional boundaries in the spec (`api/`, `agents/`, `pipeline/`, `models/`, `services/`, `store/`, `tracking/`). This matches the layout established in `CLAUDE.md`.

## Complexity Tracking

No complexity violations. Single-project layout, in-memory store (no ORM/database), direct service calls (no repository pattern), no external message queue. Each choice is the minimum sufficient for the stated constraints.
