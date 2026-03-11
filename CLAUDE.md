# scout Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-03-11

## Active Technologies
- Python 3.11+ + FastAPI 0.110+, CrewAI ~0.80.0, LiteLLM (bundled with CrewAI), httpx, PyMuPDF (fitz), MLflow, duckduckgo-search, crewai-tools, ruff, uv (001-scout-multiagent-system)
- In-memory `dict[str, AnalysisJob]` with `threading.RLock`; CV files saved to `UPLOAD_DIR` on disk (001-scout-multiagent-system)

- Python 3.11+ + FastAPI, CrewAI ~0.80.0 (with LiteLLM/Ollama), httpx, PyMuPDF (fitz), MLflow, ruff (001-scout-multiagent-system)

## Project Structure

```text
src/scout/
├── api/          # FastAPI app and routes
├── agents/       # CrewAI agent definitions
├── pipeline/     # Crew assembly and task definitions
├── models/       # Pydantic data models
├── services/     # GitHub, PDF, and market data clients
├── store/        # In-memory job store
└── tracking/     # MLflow helpers

tests/
├── unit/
├── integration/
└── contract/
```

## Commands

```bash
# Install
uv sync --dev

# Run service
uv run uvicorn scout.api.main:app --reload --port 8000

# Test
uv run pytest

# Lint / format
uv run ruff check .
uv run ruff format .
```

## Code Style

Python 3.11+: Follow standard conventions

## Recent Changes
- 001-scout-multiagent-system: Added Python 3.11+ + FastAPI 0.110+, CrewAI ~0.80.0, LiteLLM (bundled with CrewAI), httpx, PyMuPDF (fitz), MLflow, duckduckgo-search, crewai-tools, ruff, uv

- 001-scout-multiagent-system: Added Python 3.11+ + FastAPI, CrewAI ~0.80.0 (with LiteLLM/Ollama), httpx, PyMuPDF (fitz), MLflow, ruff

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
