---

description: "Task list for Scout — Multi-Agent Portfolio Opportunity Finder"
---

# Tasks: Scout — Multi-Agent Portfolio Opportunity Finder

**Input**: Design documents from `/specs/001-scout-multiagent-system/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/http-api.md ✅, quickstart.md ✅

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2…US5)
- Exact file paths included in every task description

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Initialize pyproject.toml with Python 3.11+, FastAPI 0.110+, CrewAI ~0.80.0, httpx, PyMuPDF (fitz), MLflow, duckduckgo-search, crewai-tools, ruff, pytest, pytest-asyncio under uv project layout
- [ ] T002 [P] Create src/scout/ package directory structure with empty `__init__.py` files: api/, api/routes/, agents/, pipeline/, models/, services/, store/, tracking/
- [ ] T003 [P] Create tests/ directory structure with empty `__init__.py` files: tests/unit/, tests/integration/, tests/contract/
- [ ] T004 [P] Create .env.example with GITHUB_TOKEN, OLLAMA_HOST, OLLAMA_MODEL, MLFLOW_TRACKING_URI, UPLOAD_DIR variables

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core Pydantic models, job store, MLflow init, and FastAPI app shell that ALL user stories depend on

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T005 [P] Create AnalysisJob, JobStatus, AgentStatus, AgentName Pydantic models with state machine and all fields (id, status, github_username, cv_file_path, agent_statuses, result, error, created_at, started_at, completed_at) in src/scout/models/job.py
- [ ] T006 [P] Create UserProfile, LanguageStat, CommitActivity, SkillGap, GapSource Pydantic models in src/scout/models/profile.py
- [ ] T007 [P] Create MarketSnapshot, TrendingRepo, HiringSignal, WebInsight Pydantic models with degraded/degraded_note fields in src/scout/models/market.py
- [ ] T008 [P] Create ProjectIdea Pydantic model with rank, title, rationale, market_signals, profile_evidence, technology_direction, estimated_impact fields in src/scout/models/ideas.py
- [ ] T009 [P] Create ProjectPRD Pydantic model with project_idea_rank, title, problem_statement, target_user, core_features (3–5 items validated), mvp_scope, success_criteria (≥1 item validated) fields in src/scout/models/prd.py
- [ ] T010 Create ScoutReport Pydantic model with job_id, generated_at, user_profile, market_snapshot, all_project_ideas, top_project_ideas (exactly 3), prds (exactly 3), markdown fields in src/scout/models/report.py
- [ ] T011 Implement thread-safe JobStore with threading.RLock, dict[str, AnalysisJob] backing store, and create/get/update/delete methods in src/scout/store/jobs.py
- [ ] T012 [P] Implement MLflow autolog initialization function (mlflow.set_tracking_uri + mlflow.autolog() — no other MLflow calls permitted) in src/scout/tracking/mlflow.py
- [ ] T013 Create FastAPI app factory with lifespan calling init_tracking() at startup; include uploads directory creation from UPLOAD_DIR env var in src/scout/api/main.py
- [ ] T014 [P] Create FastAPI shared dependency functions for JobStore injection in src/scout/api/deps.py

**Checkpoint**: Foundation ready — user story implementation can now begin

---

## Phase 3: User Story 1 — Full Pipeline Execution (Priority: P1) 🎯 MVP

**Goal**: Four agents execute sequentially (Analyst → Researcher → Strategist → Specifier), each consuming the previous agent's structured output, producing a complete ScoutReport with three project ideas.

**Independent Test**: Submit a real GitHub username and PDF via `POST /jobs`; poll `/jobs/{id}/status` until `completed`; retrieve report via `GET /jobs/{id}/result`. Verify the response contains a ScoutReport with three project ideas and three PRD sections.

- [ ] T015 [P] [US1] Implement async GitHub REST client using httpx with PAT authentication, parallel repo fetching (GET /users/{username}, /users/{username}/repos, /repos/{owner}/{repo}/languages, /repos/{owner}/{repo}/topics, /repos/{owner}/{repo}/stats/contributors), and exponential backoff in src/scout/services/github.py
- [ ] T016 [P] [US1] Implement CV PDF text extraction using PyMuPDF with block-level reading-order sorting; detect and raise for image-only PDFs (empty text + images present) and encrypted PDFs in src/scout/services/pdf.py
- [ ] T017 [P] [US1] Implement market data gathering client with DuckDuckGo (duckduckgo-search), GitHub Search API (trending AI/ML repos by topic+recency), and HN Firebase/Algolia API (Who is Hiring threads) in src/scout/services/market.py
- [ ] T018 [P] [US1] Define Analyst agent with goal, backstory, and tool assignments (GitHub profile fetcher, PDF text extractor) using LiteLLM model string `ollama/qwen2.5:9b` and OLLAMA_HOST env var as api_base in src/scout/agents/analyst.py
- [ ] T019 [P] [US1] Define Researcher agent with goal, backstory, and DuckDuckGoSearchRun + GitHub Search tools (crewai-tools) using LiteLLM `ollama/qwen2.5:9b` in src/scout/agents/researcher.py
- [ ] T020 [P] [US1] Define Strategist agent with goal and backstory for ranking ≥5 project ideas by portfolio impact using LiteLLM `ollama/qwen2.5:9b` in src/scout/agents/strategist.py
- [ ] T021 [P] [US1] Define Specifier agent with goal and backstory for producing three mini PRDs using LiteLLM `ollama/qwen2.5:9b` in src/scout/agents/specifier.py
- [ ] T022 [US1] Define four CrewAI Task objects (analyst_task, researcher_task, strategist_task, specifier_task) with description, context chains, and output_pydantic=UserProfile/MarketSnapshot/list[ProjectIdea]/list[ProjectPRD] in src/scout/pipeline/tasks.py
- [ ] T023 [US1] Assemble CrewAI Crew with Process.sequential, implement kickoff_async wrapper that updates AnalysisJob.agent_statuses (pending→running→completed/failed) and AnalysisJob.status on transitions in src/scout/pipeline/crew.py
- [ ] T024 [US1] Implement ScoutReport assembly and markdown renderer (profile summary, market context, ranked ideas, three PRD sections) after all agents complete; set job.result and job.status=completed in src/scout/pipeline/crew.py
- [ ] T025 [US1] Implement POST /jobs endpoint: save uploaded PDF to UPLOAD_DIR, create AnalysisJob, call asyncio.create_task(run_pipeline()), return 202 with job_id/status/status_url/result_url in src/scout/api/routes/jobs.py
- [ ] T026 [US1] Implement GET /jobs/{job_id}/result endpoint returning ScoutReport as text/markdown (default) or application/json based on ?format= query param; return 409 for non-completed jobs in src/scout/api/routes/jobs.py

**Checkpoint**: At this point, the full pipeline is functional — submit a job via POST /jobs and receive a Scout Report

---

## Phase 4: User Story 2 — Profile Analysis (Priority: P2)

**Goal**: The Analyst agent's output (skills map, stack summary, identified gaps) accurately reflects the developer's actual profile and is visible in the Scout Report.

**Independent Test**: Submit a known GitHub username (e.g., a profile with clear language distribution) and a matching CV PDF. Inspect `report.user_profile` for correct languages, frameworks, domains, commit_activity, and at least one gap in gap_analysis. Verify the report's profile summary section matches.

- [ ] T027 [US2] Enhance GitHub service to fetch per-repo language bytes and compute percentage across all public repos; fetch repo topics; fetch contributor stats with 202-retry handling in src/scout/services/github.py
- [ ] T028 [US2] Enhance PDF service to use PyMuPDF block-level x/y sorting for multi-column CV reading order; extract skills, experience domains, and education sections as structured text in src/scout/services/pdf.py
- [ ] T029 [US2] Implement gap analysis prompt instructions in the Analyst task description: detect cv_without_github (skills on CV absent from GitHub), github_without_cv (GitHub languages/tools absent from CV), and inferred gaps; produce SkillGap list in src/scout/pipeline/tasks.py
- [ ] T030 [US2] Add expected_output validation to analyst_task requiring all UserProfile fields populated: languages, frameworks, domains, commit_activity, cv_skills, cv_experience_domains, cv_education, gap_analysis in src/scout/pipeline/tasks.py
- [ ] T031 [US2] Render profile summary section in markdown report including languages, top frameworks, domains, commit activity, and gap analysis table in src/scout/pipeline/crew.py

**Checkpoint**: Profile analysis is transparent and verifiable in the Scout Report

---

## Phase 5: User Story 3 — Market Trend Research (Priority: P3)

**Goal**: The Researcher agent gathers current AI/ML market intelligence from multiple sources; findings are reflected in project rankings and visible in the report; degraded coverage is reported transparently.

**Independent Test**: Submit any valid profile and inspect the report's market context section. Verify it references at least one trending GitHub repo and one hiring signal. Disable one market source and confirm the pipeline completes with `MarketSnapshot.degraded=True` and a human-readable degraded_note.

- [ ] T032 [US3] Implement GitHub Search API queries in market service for trending AI/ML repos: `topic:llm+pushed:>7-days-ago`, `topic:rag`, `topic:mlops` sorted by stars, collecting TrendingRepo entries in src/scout/services/market.py
- [ ] T033 [US3] Implement HN Firebase API queries (latest "Who is Hiring?" thread) and HN Algolia search to extract technology mention counts as HiringSignal entries in src/scout/services/market.py
- [ ] T034 [US3] Implement DuckDuckGo search integration in market service for WebInsight entries (queries: "AI ML developer jobs 2026", "top AI open source projects 2026") in src/scout/services/market.py
- [ ] T035 [US3] Implement degradation handling in market service: catch exceptions per source independently, set MarketSnapshot.degraded=True and populate degraded_note listing failed sources if any fail; proceed with available sources in src/scout/services/market.py
- [ ] T036 [US3] Render market context section in markdown report including top_domains, trending repos table, hiring signals table, and degraded-coverage notice when MarketSnapshot.degraded is True in src/scout/pipeline/crew.py

**Checkpoint**: Market research is grounded in current data; single-source failures do not break the pipeline

---

## Phase 6: User Story 4 — Project Specification Output (Priority: P4)

**Goal**: For each of the top three ranked project ideas, the Specifier agent produces a complete mini PRD that stands alone as an actionable specification.

**Independent Test**: Inspect the three PRD sections of a completed report. Each must contain a problem statement, target user, 3–5 core features, MVP scope, and at least one success criterion — independently readable without referencing other report sections.

- [ ] T037 [US4] Configure Specifier task description to produce complete PRDs with all required fields: problem_statement (one paragraph), target_user (persona description), core_features (3–5 items), mvp_scope (included and excluded scope), success_criteria (≥1 measurable metric) in src/scout/pipeline/tasks.py
- [ ] T038 [US4] Add Pydantic field validators to ProjectPRD: core_features length between 3 and 5, success_criteria minimum length 1 in src/scout/models/prd.py
- [ ] T039 [US4] Add Strategist task description instruction to produce ≥5 ProjectIdea candidates with at least 1 market_signals entry and 1 profile_evidence entry each before selecting top 3 in src/scout/pipeline/tasks.py
- [ ] T040 [US4] Render three PRD sections in markdown report as self-contained subsections (Problem, Target User, Core Features, MVP Scope, Success Criteria) with clear headings in src/scout/pipeline/crew.py

**Checkpoint**: Each PRD section is independently readable and complete; all three projects have full specifications

---

## Phase 7: User Story 5 — API Interface (Priority: P5)

**Goal**: Scout exposes a production-quality programmatic interface: validated input, clear errors, job polling, DELETE cleanup, and health liveness check.

**Independent Test**: Use `curl` to: (1) submit a valid job and receive 202 with job_id; (2) submit with missing github_username and receive 422 with field error; (3) poll /status and see agent_statuses update; (4) request /result before completion and receive 409; (5) DELETE a job and confirm 204; (6) GET /health and confirm 200.

- [ ] T041 [P] [US5] Implement input validation in POST /jobs before pipeline starts (SC-006 ≤5s): verify github_username non-empty + GitHub API resolves to public profile with ≥1 repo; verify cv_file content-type=application/pdf; verify PDF yields non-empty text via PyMuPDF; return 422 with error/message/field on failure in src/scout/api/routes/jobs.py
- [ ] T042 [P] [US5] Implement GET /jobs/{job_id}/status endpoint returning job_id, status, agent_statuses dict, created_at, started_at, completed_at, error; return 404 for unknown job_id in src/scout/api/routes/jobs.py
- [ ] T043 [US5] Implement DELETE /jobs/{job_id} endpoint removing job from JobStore; return 204 on success, 404 if not found in src/scout/api/routes/jobs.py
- [ ] T044 [P] [US5] Implement GET /health endpoint checking Ollama liveness (GET OLLAMA_HOST/api/tags) and MLflow liveness; return 200 with status=ok or 503 with status=degraded and per-dependency breakdown in src/scout/api/routes/health.py
- [ ] T045 [US5] Register jobs_router and health_router in FastAPI app; include API routers with appropriate prefixes in src/scout/api/main.py
- [ ] T046 [P] [US5] Implement consistent error response schema with error/message/field keys for all error codes: invalid_input (400/422), not_found (404), job_not_complete (409), internal_error (500) in src/scout/api/routes/jobs.py
- [ ] T047 [US5] Implement GitHub rate-limit detection (HTTP 429 response or X-RateLimit-Remaining=0) in exponential backoff loop (3 attempts, 2^attempt seconds); on exhaustion set job.status=failed with human-readable rate-limit error message in src/scout/services/github.py
- [ ] T048 [US5] Implement clean pipeline failure handling in crew.py: catch all exceptions, mark current agent as failed, set job.error to human-readable message, set job.status=failed, set job.completed_at; ensure no partial ScoutReport is set on job.result in src/scout/pipeline/crew.py

**Checkpoint**: Full API contract satisfied — job submission, polling, result retrieval, deletion, and health checks all function correctly

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Docker deployment, test suite, config finalization, and quickstart validation

- [ ] T049 Create multi-stage Dockerfile: builder stage runs `uv sync --no-dev --frozen`, runtime stage copies .venv and src/; entrypoint `uv run uvicorn scout.api.main:app --host 0.0.0.0 --port 8000`
- [ ] T050 Create docker-compose.yml with scout-api service (build, ports 8000:8000, env vars from .env) and Ollama service with cpu/gpu-amd/gpu-nvidia profiles; OLLAMA_HOST=http://ollama:11434 within Docker network
- [ ] T051 [P] Configure ruff in pyproject.toml: line-length=100, target-version=py311, enable E/W/I/UP rule sets
- [ ] T052 [P] Configure pytest and pytest-asyncio in pyproject.toml: testpaths=tests, asyncio_mode=auto
- [ ] T053 [P] Write unit tests for GitHub service retry logic (mock httpx responses), 404 vs 403/429 handling, and parallel fetch in tests/unit/test_github_service.py
- [ ] T054 [P] Write unit tests for PDF service text extraction, image-only detection (empty text + images present), and encrypted PDF detection in tests/unit/test_pdf_service.py
- [ ] T055 [P] Write unit tests for market service degradation handling when one or more sources raise exceptions in tests/unit/test_market_service.py
- [ ] T056 [P] Write unit tests for thread-safe JobStore concurrent reads/writes using threading.Thread in tests/unit/test_job_store.py
- [ ] T057 [P] Write unit tests for Pydantic model validation: ProjectPRD core_features bounds, ScoutReport top_project_ideas/prds length constraints, AnalysisJob state transitions in tests/unit/test_models.py
- [ ] T058 Write contract tests for all HTTP API endpoints (request/response shapes, status codes per contracts/http-api.md) using FastAPI TestClient in tests/contract/test_http_api.py
- [ ] T059 Run quickstart.md local dev validation: uv sync --dev, configure .env, start uvicorn, submit test job, poll for completion

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately; T002, T003, T004 parallel
- **Foundational (Phase 2)**: Depends on Phase 1 — BLOCKS all user stories
  - T005–T009 parallel (independent model files)
  - T010 depends on T005–T009 (ScoutReport references all other models)
  - T011 depends on T005 (JobStore stores AnalysisJob)
  - T012 parallel with T011
  - T013 depends on T011, T012
  - T014 depends on T011
- **User Stories (Phase 3–7)**: All depend on Phase 2 completion; stories can proceed in priority order
- **Polish (Phase 8)**: Depends on all desired user stories complete

### User Story Dependencies

- **US1 (Phase 3, P1)**: Can start after Phase 2 — no dependency on other stories; T015–T021 parallel; T022 depends on T015–T021; T023–T026 depend on T022
- **US2 (Phase 4, P2)**: Can start after Phase 3 — enhances Analyst agent and services implemented in US1
- **US3 (Phase 5, P3)**: Can start after Phase 3 — enhances market service implemented in US1
- **US4 (Phase 6, P4)**: Can start after Phase 3 — validates and enriches Specifier/Strategist implemented in US1
- **US5 (Phase 7, P5)**: Can start after Phase 3 — adds full API contract quality on top of basic endpoints in US1

### Within Each User Story

- Models before services
- Services before agent definitions
- Agent definitions before pipeline task/crew assembly
- Core implementation before validation/enrichment tasks
- Within Phase 3: T015–T021 [P] → T022 → T023 → T024 → T025 → T026

### Parallel Opportunities

- All setup tasks marked [P] can run simultaneously
- All model files in Phase 2 (T005–T009) can run simultaneously
- In Phase 3, all services (T015–T017) and all agent definitions (T018–T021) can run simultaneously
- All polish/test tasks marked [P] can run simultaneously
- US2–US4 can be started in parallel once Phase 3 is complete (different files per story)

---

## Parallel Example: Phase 3 (User Story 1)

```bash
# Launch all services and agent definitions together (different files):
Task T015: Implement GitHub service in src/scout/services/github.py
Task T016: Implement PDF service in src/scout/services/pdf.py
Task T017: Implement market service in src/scout/services/market.py
Task T018: Define Analyst agent in src/scout/agents/analyst.py
Task T019: Define Researcher agent in src/scout/agents/researcher.py
Task T020: Define Strategist agent in src/scout/agents/strategist.py
Task T021: Define Specifier agent in src/scout/agents/specifier.py

# Then sequentially:
Task T022: Define pipeline tasks in src/scout/pipeline/tasks.py
Task T023: Assemble crew in src/scout/pipeline/crew.py
Task T024: Add markdown renderer in src/scout/pipeline/crew.py
Task T025: Add POST /jobs endpoint in src/scout/api/routes/jobs.py
Task T026: Add GET /jobs/{id}/result endpoint in src/scout/api/routes/jobs.py
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: User Story 1 (Full Pipeline Execution)
4. **STOP and VALIDATE**: Submit a real GitHub username + PDF; confirm Scout Report produced
5. Deploy/demo the working pipeline

### Incremental Delivery

1. Setup + Foundational → foundation ready
2. US1 (Phase 3) → full pipeline working end-to-end → **MVP demo**
3. US2 (Phase 4) → profile analysis is accurate and transparent
4. US3 (Phase 5) → market research grounded in current data
5. US4 (Phase 6) → each PRD is a complete actionable specification
6. US5 (Phase 7) → API is production-quality (validation, error handling, polling)
7. Polish (Phase 8) → Docker deployment, full test suite

### Parallel Team Strategy

With multiple developers after Phase 2 completes:

- Developer A: US1 services + agents (T015–T021 in parallel)
- Developer B: US1 pipeline assembly (T022–T024, unblocked after T015–T021)
- Developer C: US1 API endpoints (T025–T026, unblocked after T022–T024)
- After Phase 3: Developer A → US2, Developer B → US3, Developer C → US4/US5

---

## Notes

- [P] tasks operate on different files with no dependencies on incomplete tasks — safe to run simultaneously
- [Story] label maps each task to its user story for independent traceability
- Each user story phase produces an independently testable increment before the next begins
- Commit after each task or logical group of parallel tasks
- Stop at each **Checkpoint** to validate the story independently before continuing
- MLflow: `mlflow.autolog()` is the only permitted MLflow call (FR-013) — no manual logging anywhere
- All agents use LiteLLM model string `ollama/qwen2.5:9b`; `api_base` from `OLLAMA_HOST` env var
- Job store uses `threading.RLock` (not asyncio locks) because pipeline runs in thread executor context
- File uploads saved to UPLOAD_DIR before enqueueing — background task needs a stable path
