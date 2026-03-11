# Scout

AI-powered multi-agent system that analyzes your GitHub profile and CV to generate ranked, market-aligned project ideas with full PRDs.

## How it works

Scout runs four specialized agents in sequence:

1. **Analyst** — extracts skills, stack, and gaps from your GitHub profile and CV
2. **Researcher** — gathers current AI/ML market trends via DuckDuckGo, GitHub Search, and HN
3. **Strategist** — ranks project ideas by portfolio impact + market demand
4. **Specifier** — writes a mini PRD for each of the top 3 ideas

Output: a structured markdown report with your profile summary, market context, ranked ideas, and three actionable PRDs.

## Quickstart

### Docker Compose (recommended)

```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx

# CPU (default)
docker compose --profile cpu up -d

# Nvidia GPU
docker compose --profile gpu-nvidia up -d

# AMD GPU
docker compose --profile gpu-amd up -d
```

API available at `http://localhost:8000`. Ollama pulls `qwen2.5:9b` automatically on first run.

### Local development

**Prerequisites**: Python 3.11+, [uv](https://docs.astral.sh/uv/), [Ollama](https://ollama.com)

```bash
git clone <repo-url> && cd scout
uv sync --dev
ollama pull qwen2.5:9b

cp .env.example .env   # set GITHUB_TOKEN, OLLAMA_HOST, etc.

uv run uvicorn scout.api.main:app --reload --port 8000
```

## Usage

```bash
# Submit a job
curl -X POST http://localhost:8000/jobs \
  -F "github_username=torvalds" \
  -F "cv_file=@/path/to/cv.pdf"
# → { "job_id": "abc123", "status": "pending", "status_url": "/jobs/abc123/status" }

# Poll status
curl http://localhost:8000/jobs/abc123/status

# Retrieve report (markdown or json)
curl http://localhost:8000/jobs/abc123/result
curl "http://localhost:8000/jobs/abc123/result?format=json"

# Clean up
curl -X DELETE http://localhost:8000/jobs/abc123
```

Full pipeline takes up to ~15 minutes. Poll every 10–30 seconds.

## API reference

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/jobs` | Submit job (`github_username` + `cv_file` PDF) |
| `GET` | `/jobs/{id}/status` | Per-agent status: `pending → running → completed/failed` |
| `GET` | `/jobs/{id}/result` | Scout report (`?format=markdown\|json`) |
| `DELETE` | `/jobs/{id}` | Remove job from store |
| `GET` | `/health` | Liveness + dependency check |

## Development

```bash
uv run pytest                  # all tests
uv run pytest tests/unit/      # unit only
uv run ruff check . && uv run ruff format .   # lint + format
uv run mlflow ui --port 5000   # observability UI
```

## Troubleshooting

**Ollama unreachable** — run `ollama serve` and verify with `curl http://localhost:11434/api/tags`.

**GitHub rate limit** — set `GITHUB_TOKEN` in `.env`. Without it, the 60 req/hr unauthenticated limit is exhausted quickly.

**Image-only PDF rejected** — the CV must contain extractable text. Convert with OCR first: `ocrmypdf input.pdf output.pdf`.

**Job stuck in `running`** — check the MLflow UI at `http://localhost:5000` and the uvicorn console for errors.
