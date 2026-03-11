# HTTP API Contract: Scout

**Phase**: 1 — Design
**Branch**: `001-scout-multiagent-system`
**Date**: 2026-03-11
**Base URL**: `http://localhost:8000`

---

## Overview

Scout exposes a REST API with three primary resources:
- **Jobs** — submit and monitor pipeline runs
- **Health** — service liveness check

All job submission uses `multipart/form-data` (required for file upload).
All responses use `application/json`.

---

## Endpoints

### POST /jobs

Submit a new analysis job. Validates inputs, saves the CV to disk, creates the job record, and enqueues the pipeline. Returns immediately with the job identifier.

**Request**: `Content-Type: multipart/form-data`

| Field | Type | Required | Description |
|---|---|---|---|
| `github_username` | string (form field) | Yes | GitHub username (e.g., `torvalds`). Must resolve to a public profile with ≥1 public repo. |
| `cv_file` | file (PDF) | Yes | CV document. Must be `application/pdf` with extractable text (not image-only). Max size: 20MB. |

**Responses**:

```
HTTP 202 Accepted
Content-Type: application/json

{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "pending",
  "created_at": "2026-03-11T10:00:00Z",
  "status_url": "/jobs/550e8400-e29b-41d4-a716-446655440000/status",
  "result_url": "/jobs/550e8400-e29b-41d4-a716-446655440000/result"
}
```

```
HTTP 422 Unprocessable Entity
Content-Type: application/json

{
  "error": "invalid_input",
  "message": "GitHub username 'xyz' does not exist or has no public repositories.",
  "field": "github_username"
}
```

```
HTTP 422 Unprocessable Entity
Content-Type: application/json

{
  "error": "invalid_input",
  "message": "The uploaded PDF contains no extractable text. Image-only PDFs are not supported.",
  "field": "cv_file"
}
```

```
HTTP 400 Bad Request
Content-Type: application/json

{
  "error": "invalid_input",
  "message": "File type not supported. Only PDF files are accepted.",
  "field": "cv_file"
}
```

**Validation rules** (applied before pipeline starts, SC-006: response within 5 seconds):
1. `github_username` must be non-empty.
2. GitHub API `GET /users/{username}` must return a 200 with `public_repos >= 1`.
3. `cv_file` must have `Content-Type: application/pdf`.
4. CV PDF must yield non-empty text from PyMuPDF extraction on at least one page.

---

### GET /jobs/{job_id}/status

Poll the execution status of a job. Returns the current state, per-agent breakdown, and timing information.

**Path parameter**: `job_id` — UUID4 string

**Responses**:

```
HTTP 200 OK
Content-Type: application/json

{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "running",
  "agent_statuses": {
    "analyst": "completed",
    "researcher": "running",
    "strategist": "pending",
    "specifier": "pending"
  },
  "created_at": "2026-03-11T10:00:00Z",
  "started_at": "2026-03-11T10:00:02Z",
  "completed_at": null,
  "error": null
}
```

```
HTTP 200 OK
Content-Type: application/json

{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "agent_statuses": {
    "analyst": "completed",
    "researcher": "completed",
    "strategist": "completed",
    "specifier": "completed"
  },
  "created_at": "2026-03-11T10:00:00Z",
  "started_at": "2026-03-11T10:00:02Z",
  "completed_at": "2026-03-11T10:08:45Z",
  "error": null
}
```

```
HTTP 200 OK
Content-Type: application/json

{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "failed",
  "agent_statuses": {
    "analyst": "completed",
    "researcher": "failed",
    "strategist": "pending",
    "specifier": "pending"
  },
  "created_at": "2026-03-11T10:00:00Z",
  "started_at": "2026-03-11T10:00:02Z",
  "completed_at": "2026-03-11T10:03:11Z",
  "error": "Researcher agent failed: GitHub API rate limit exceeded after 3 retries."
}
```

```
HTTP 404 Not Found
Content-Type: application/json

{
  "error": "not_found",
  "message": "Job '550e8400-e29b-41d4-a716-446655440000' not found."
}
```

**Status update guarantee**: Status transitions reflected within 10 seconds of agent state change (SC-005).

---

### GET /jobs/{job_id}/result

Retrieve the completed Scout report for a job. Only available when `status == "completed"`.

**Path parameter**: `job_id` — UUID4 string

**Query parameter**: `format` — `markdown` (default) | `json`

**Responses**:

```
HTTP 200 OK (format=markdown)
Content-Type: text/markdown

# Scout Report: torvalds

## Your Profile Summary
...

## Market Context
...

## Ranked Project Ideas
...

## Project PRD 1: ...
...
```

```
HTTP 200 OK (format=json)
Content-Type: application/json

{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "generated_at": "2026-03-11T10:08:45Z",
  "github_username": "torvalds",
  "user_profile": { ... },
  "market_snapshot": { ... },
  "all_project_ideas": [ ... ],
  "top_project_ideas": [ ... ],
  "prds": [ ... ],
  "markdown": "# Scout Report: torvalds\n..."
}
```

```
HTTP 404 Not Found
Content-Type: application/json

{
  "error": "not_found",
  "message": "Job '550e8400-e29b-41d4-a716-446655440000' not found."
}
```

```
HTTP 409 Conflict
Content-Type: application/json

{
  "error": "job_not_complete",
  "message": "Job is still running. Current status: running.",
  "status_url": "/jobs/550e8400-e29b-41d4-a716-446655440000/status"
}
```

---

### GET /health

Liveness check for the service and its external dependencies.

**Response**:

```
HTTP 200 OK
Content-Type: application/json

{
  "status": "ok",
  "dependencies": {
    "ollama": "ok",
    "mlflow": "ok"
  }
}
```

```
HTTP 503 Service Unavailable
Content-Type: application/json

{
  "status": "degraded",
  "dependencies": {
    "ollama": "unreachable",
    "mlflow": "ok"
  }
}
```

---

## Error Response Schema

All error responses follow a consistent structure:

```json
{
  "error": "<error_code>",      // machine-readable code
  "message": "<human_message>", // human-readable description (SC-007)
  "field": "<field_name>"       // optional, present for input validation errors
}
```

**Error codes**:

| Code | HTTP Status | Description |
|---|---|---|
| `invalid_input` | 400 / 422 | Request validation failure |
| `not_found` | 404 | Job ID does not exist |
| `job_not_complete` | 409 | Result requested before job is done |
| `internal_error` | 500 | Unexpected server error |

---

## Polling Example

```
POST /jobs
  github_username=torvalds
  cv_file=@cv.pdf
→ 202 { "job_id": "abc123", "status_url": "/jobs/abc123/status" }

GET /jobs/abc123/status
→ 200 { "status": "running", "agent_statuses": { "analyst": "running", ... } }

GET /jobs/abc123/status   (poll every 10–30 seconds)
→ 200 { "status": "completed", ... }

GET /jobs/abc123/result
→ 200 (markdown report)
```
