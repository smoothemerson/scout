# Quickstart: Scout

## Option A — Docker Compose (recommended)

### Prerequisites
- [Docker](https://docs.docker.com/get-docker/) + Docker Compose v2

### Start the full stack

```bash
# CPU (default — works on any machine)
docker compose --profile cpu up -d

# Nvidia GPU
docker compose --profile gpu-nvidia up -d

# AMD GPU
docker compose --profile gpu-amd up -d
```

The Scout API is available at `http://localhost:8000`. Ollama pulls `qwen2.5:9b` automatically on first run.

Set your GitHub token before starting:
```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
docker compose --profile cpu up -d
```

---

## Option B — Local Development

### Prerequisites

- Python 3.11+, [uv](https://docs.astral.sh/uv/)
- [Ollama](https://ollama.com) running locally
- A GitHub Personal Access Token (PAT) with `public_repo` read scope

---

## 1. Clone and install

```bash
git clone <repo-url>
cd scout
uv sync --dev
```

---

## 2. Pull the LLM model

```bash
ollama pull qwen2.5:9b
```

Verify Ollama is reachable:

```bash
curl http://localhost:11434/api/tags
```

---

## 3. Configure environment

```bash
cp .env.example .env
```

Edit `.env`:

```env
GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=qwen2.5:9b
MLFLOW_TRACKING_URI=./mlruns
UPLOAD_DIR=./uploads
```

---

## 4. Start the service

```bash
uv run uvicorn scout.api.main:app --reload --port 8000
```

The API is now available at `http://localhost:8000`.

Optional: start the MLflow UI in a separate terminal:

```bash
uv run mlflow ui --backend-store-uri ./mlruns --port 5000
```

---

## 5. Submit your first job

```bash
curl -X POST http://localhost:8000/jobs \
  -F "github_username=torvalds" \
  -F "cv_file=@/path/to/your-cv.pdf"
```

Response:

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "pending",
  "status_url": "/jobs/550e8400-e29b-41d4-a716-446655440000/status"
}
```

---

## 6. Poll for completion

```bash
# Poll every 30 seconds
watch -n 30 curl -s http://localhost:8000/jobs/550e8400-e29b-41d4-a716-446655440000/status
```

---

## 7. Retrieve the report

```bash
# Markdown report (default)
curl http://localhost:8000/jobs/550e8400-e29b-41d4-a716-446655440000/result

# JSON report
curl "http://localhost:8000/jobs/550e8400-e29b-41d4-a716-446655440000/result?format=json"

# Save to file
curl http://localhost:8000/jobs/550e8400-e29b-41d4-a716-446655440000/result \
  -o scout-report.md
```

---

## Running Tests

```bash
# All tests
uv run pytest

# Unit tests only
uv run pytest tests/unit/

# Integration tests only
uv run pytest tests/integration/

# Contract tests only
uv run pytest tests/contract/
```

---

## Development

```bash
# Lint and format
uv run ruff check .
uv run ruff format .

# Type check
uv run mypy src/
```

---

## Troubleshooting

**Ollama not reachable**
```
curl http://localhost:11434/api/tags
# If unreachable: start Ollama with `ollama serve`
```

**GitHub rate limit errors**
Ensure `GITHUB_TOKEN` is set in `.env` (or as an environment variable for Docker Compose). Unauthenticated calls exhaust the 60 req/hr limit on any profile with >10 repos.

**Image-only PDF error**
The CV PDF must contain extractable text. PDFs created by scanning paper documents without OCR will be rejected. Convert with OCR first (e.g., `ocrmypdf input.pdf output.pdf`).

**Job stuck in `running` with no progress**
Check the MLflow UI at `http://localhost:5000` for the latest run's agent logs. Also check the uvicorn console for exception output.
