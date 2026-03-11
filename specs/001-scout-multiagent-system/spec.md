# Feature Specification: Scout — Multi-Agent Portfolio Opportunity Finder

**Feature Branch**: `001-scout-multiagent-system`
**Created**: 2026-03-11
**Status**: Draft

## Overview

Scout is a multi-agent system that takes a developer's GitHub profile and CV (PDF) as input and produces a ranked, fully-specified set of project ideas tailored to their skills, gaps, and current market demand. Rather than generating generic suggestions, Scout scouts the actual landscape — analyzing the user's real profile and real market trends — and returns actionable project opportunities ready to build.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Full Pipeline Execution (Priority: P1)

A developer submits their GitHub username and CV PDF to Scout. The system automatically coordinates four specialized agents in sequence: one analyzes the developer's profile and skills, another researches current market and AI/ML trends, a third ranks project opportunities by portfolio impact, and a fourth writes a brief product specification for each top idea. The developer receives a structured markdown report with three prioritized project ideas, each with a rationale, recommended technology direction, and immediate next steps.

**Why this priority**: This is the core value proposition of the entire system. Without this end-to-end flow, Scout has no value. All other stories are enhancements.

**Independent Test**: Submit a real GitHub username and PDF CV. The system should return a complete markdown report with three project ideas within an acceptable wait time, without any manual intervention.

**Acceptance Scenarios**:

1. **Given** a valid GitHub username and a valid CV PDF, **When** a user submits them to Scout, **Then** the system returns a markdown report containing exactly three prioritized project ideas with justification, recommended direction, and next steps for each.
2. **Given** a valid GitHub username and a valid CV PDF, **When** the pipeline runs, **Then** all four agents execute in the correct sequence (Analyst → Researcher → Strategist → Specifier) and each agent's output is used as input to the next.
3. **Given** a valid submission, **When** the report is delivered, **Then** each project idea includes: a title, a one-paragraph justification tied to the user's profile and market trends, a recommended technology direction (without mandating specific tools), and a list of concrete next steps.

---

### User Story 2 - Profile Analysis (Priority: P2)

A developer can verify that Scout has correctly understood their technical profile before the full pipeline runs. The Analyst agent's output (skills map, stack summary, identified gaps) is accessible in the final report to give the user confidence that recommendations are grounded in their actual profile.

**Why this priority**: If the profile analysis is wrong, all downstream recommendations are invalid. Users need to trust the foundation. This story makes the Analyst's output transparent.

**Independent Test**: Submit a GitHub username and CV. Inspect the report section summarizing the user's detected skills, primary stack, and identified gaps. Verify the information matches the actual profile.

**Acceptance Scenarios**:

1. **Given** a GitHub profile, **When** the Analyst agent processes it, **Then** the report includes a skills summary listing detected languages, frameworks, and domains, sourced from repository analysis and commit history.
2. **Given** a CV PDF, **When** the Analyst agent processes it, **Then** the report includes skills and experience extracted from the CV merged with the GitHub analysis, with no duplication.
3. **Given** a profile with clear strengths and visible gaps (e.g., no ML projects despite data science CV), **When** the Analyst runs, **Then** the gap analysis section identifies at least one actionable skill gap.

---

### User Story 3 - Market Trend Research (Priority: P3)

The Researcher agent gathers current information about what is in demand in the AI/ML job market and developer ecosystem. Its findings are reflected in project recommendations, so users can trust the suggestions are relevant as of today.

**Why this priority**: Without current market data, Scout's recommendations would be generic and time-insensitive. This story ensures the system's outputs are grounded in present-day demand.

**Independent Test**: Submit any profile and inspect the report for evidence that market trends influenced the project rankings — e.g., a recommendation mentioning a currently trending technology or hiring demand signal.

**Acceptance Scenarios**:

1. **Given** any valid submission, **When** the Researcher agent runs, **Then** the report includes a market context section summarizing current hiring trends in AI/ML and top trending project types on GitHub.
2. **Given** a set of market findings, **When** the Strategist uses them, **Then** the ranking rationale for each project idea explicitly references at least one market signal (e.g., "companies are actively hiring for X", "GitHub trending shows demand for Y").

---

### User Story 4 - Project Specification Output (Priority: P4)

For each of the top three ranked project ideas, Scout produces a mini Product Requirements Document (PRD) — a concise structured document with enough detail for the developer to immediately begin planning or implementing.

**Why this priority**: The PRD output is what converts a vague idea into an actionable item. It is the terminal deliverable and makes Scout more valuable than a simple list generator.

**Independent Test**: Inspect the three PRD sections of the report. Each should stand alone as an actionable specification without needing additional research or clarification.

**Acceptance Scenarios**:

1. **Given** the top three ranked project ideas, **When** the Specifier agent processes them, **Then** each produces a PRD section with: problem statement, target user, core features (3–5), MVP scope, and success metrics.
2. **Given** a PRD section, **When** a developer reads it, **Then** they can identify the project's purpose, who it is for, what the MVP includes, and how to measure success — without needing to read any other section of the report.

---

### User Story 5 - API Interface (Priority: P5)

Scout exposes a programmatic interface that accepts inputs and returns the report, enabling integration with other tools, frontends, or automation pipelines.

**Why this priority**: A usable API makes Scout composable and accessible beyond manual use. This enables future integrations (e.g., CLI, web UI, CI/CD triggers).

**Independent Test**: Use an API client to submit a GitHub username and CV PDF and receive a well-structured JSON or markdown response without opening any UI or running any scripts manually.

**Acceptance Scenarios**:

1. **Given** a running Scout service, **When** a client POSTs a GitHub username and CV file to the appropriate endpoint, **Then** the service accepts the request, runs the pipeline, and returns the markdown report in the response.
2. **Given** an invalid request (missing GitHub username or invalid file format), **When** the client submits, **Then** the API returns a clear error response indicating what is missing or invalid.
3. **Given** a long-running pipeline, **When** the client submits a request, **Then** the API returns immediately with a job identifier and the client can poll a status endpoint for completion and retrieve the result when ready.

---

### Edge Cases

- What happens when a GitHub profile is private or has no public repositories?
- What happens when the CV PDF is corrupted, password-protected, or contains only images (no extractable text)?
- When GitHub rate limits are hit during profile analysis, the system retries with exponential backoff (up to 3 attempts). If all retries are exhausted, the job fails with a clear error message indicating the rate limit issue. Partial profiles are not returned.
- When market trend sources (DuckDuckGo, GitHub Search, HN Firebase) are unavailable or return empty results, the Researcher proceeds with whichever sources respond and includes a degraded-coverage notice in the Market Snapshot section. The pipeline does not fail due to source unavailability alone.
- When the pipeline fails or times out partway through, the job transitions to `failed` with a clear error message. No partial output is returned. The user must resubmit a new job.
- How does the system handle a GitHub username that does not exist?
- What if the CV and GitHub profile appear to be from completely different people or domains (very low coherence)?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST accept a GitHub username and a CV file as the two primary inputs for initiating an analysis job.
- **FR-002**: System MUST validate inputs before starting the pipeline: the GitHub username must resolve to a public profile with at least one public repository, and the CV must be a readable document file.
- **FR-003**: System MUST execute four agents in a fixed sequence: Analyst → Researcher → Strategist → Specifier, with each agent's output passed as context to the next.
- **FR-004**: The Analyst agent MUST extract from the GitHub profile: primary programming languages, detected frameworks and tools, repository domains (e.g., web, data, ML, DevOps), and commit activity patterns.
- **FR-005**: The Analyst agent MUST extract from the CV: listed skills, work experience domains, education background, and any explicitly mentioned technologies or methodologies.
- **FR-006**: The Analyst agent MUST produce a merged skills profile and a gap analysis identifying areas present in the CV but absent from GitHub activity, or vice versa.
- **FR-007**: The Researcher agent MUST gather current information on AI/ML job market demand and trending open-source project categories.
- **FR-008**: The Strategist agent MUST rank project ideas by their projected portfolio impact, combining the user's profile gaps with market demand signals, and MUST produce at least five ranked candidate ideas before passing the top three to the Specifier.
- **FR-009**: The Specifier agent MUST produce a mini PRD for each of the top three project ideas, including: problem statement, target user, core features (3–5 items), MVP scope definition, and measurable success criteria.
- **FR-010**: System MUST produce a final markdown report that includes: a user profile summary, a market trends summary, the ranked list of all candidate ideas with rationale, and the three full PRD sections.
- **FR-011**: System MUST expose an HTTP endpoint that accepts the inputs, initiates the pipeline asynchronously, and provides a mechanism to retrieve the final report upon completion.
- **FR-012**: System MUST track and expose the execution status of each agent in the pipeline (pending, running, completed, failed) so users or integrations can monitor progress.
- **FR-013**: System MUST enable MLflow observability via `mlflow.autolog()` only — no manual MLflow logging calls (e.g., `mlflow.log_metric`, `mlflow.log_param`) are permitted.
- **FR-014**: System MUST return a meaningful error response if any required input is missing, unreadable, or invalid, without starting the pipeline.
- **FR-015**: System MUST complete the full pipeline and return a result within a reasonable time for a synchronous user experience; for longer runs, asynchronous job polling MUST be supported.
- **FR-016**: When a GitHub API rate limit is encountered during profile analysis, the system MUST retry the failed request with exponential backoff (up to 3 attempts). If all retries fail, the job MUST transition to `failed` with a human-readable error identifying the rate limit as the cause. Partial profiles MUST NOT be returned.
- **FR-017**: If the pipeline fails or times out at any stage, the job MUST transition to `failed`. No partial output MUST be returned or exposed via the result endpoint. The result endpoint MUST return `409 Conflict` for a failed job, directing the user to the status endpoint for the error details.
- **FR-018**: System MUST expose a DELETE endpoint to remove a job from the in-memory store. Jobs are never evicted automatically; the DELETE endpoint is the only supported cleanup mechanism (outside of process restart).

### Key Entities

- **Analysis Job**: Represents a single end-to-end Scout run. Has a unique identifier, status (pending/running/completed/failed), creation timestamp, input references, and output report. Stored with a strong reference in the in-memory job dict; no automatic TTL or eviction — jobs persist until explicit deletion or process restart.
- **User Profile**: The merged representation of the developer's GitHub data and CV content. Includes skill map, stack summary, activity patterns, and gap analysis.
- **Market Snapshot**: The Researcher's output at a point in time — a summary of AI/ML hiring trends and trending project categories, with a timestamp indicating when it was gathered.
- **Project Idea**: A ranked candidate project opportunity. Includes title, rationale, estimated portfolio impact score, and references to the profile gaps and market signals that motivate it.
- **Project PRD**: A concise product requirements document for one project idea. Includes problem statement, target user, feature list, MVP scope, and success criteria.
- **Scout Report**: The final deliverable — a structured markdown document that compiles the User Profile summary, Market Snapshot, ranked Project Ideas, and three Project PRDs.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer with a public GitHub profile and a text-readable CV can receive a complete Scout Report within 15 minutes of submitting their inputs.
- **SC-002**: Each Scout Report contains exactly three project ideas, each with a complete PRD section (problem statement, target user, features, MVP scope, success criteria) — no partial or empty sections.
- **SC-003**: At least 80% of users who review their Scout Report agree that the identified skills and gaps accurately reflect their actual profile (measured via post-use survey or spot-check validation).
- **SC-004**: Each ranked project idea explicitly references at least one specific market demand signal and at least one gap or strength from the user's profile — linking recommendations to evidence.
- **SC-005**: The pipeline status endpoint reflects real-time agent execution state, with status updates occurring within 10 seconds of an agent transitioning states.
- **SC-006**: The system returns a clear, human-readable error within 5 seconds when provided an invalid GitHub username or unreadable CV file — without starting the pipeline.
- **SC-007**: A developer unfamiliar with Scout can submit their inputs and read a full report without requiring documentation or support, based on self-evident API responses and report structure.

## Clarifications

### Session 2026-03-11

- Q: Which web search tool should the Researcher agent use for real-time market intelligence? → A: DuckDuckGo (`duckduckgo-search` Python library, no API key required), replacing Brave Search API.
- Q: What happens when GitHub rate limits are hit during profile analysis? → A: Retry with exponential backoff (up to 3 attempts), then fail the job with a clear error.
- Q: When the pipeline fails or times out partway through, is partial output returned or is the job failed cleanly? → A: Fail the job cleanly; no partial output; user must resubmit.
- Q: Does the HTTP API require authentication? → A: No authentication; the service binds to localhost only.
- Q: How should MLflow be used for pipeline observability? → A: Use only `mlflow.autolog()` — no other MLflow functions or manual logging calls.
- Q: Which Python package manager should be used by default? → A: `uv` (not `pip`).
- Q: Does the in-memory job store have automatic TTL or cleanup? → A: No TTL or automatic cleanup; jobs accumulate indefinitely. Explicit cleanup must be triggered externally (e.g., via a DELETE endpoint or process restart).
- Q: Which Ollama model should be used as the default for all agents? → A: `qwen2.5:9b`.
- Q: Is Docker Compose a required deployment deliverable? → A: Yes — ship `docker-compose.yml` including both the Scout API service and Ollama (CPU/AMD/Nvidia profiles).
- Q: What is the default Docker Compose hardware profile for first-time users? → A: `cpu`; GPU profiles (`gpu-amd`, `gpu-nvidia`) are opt-in alternatives.
- Q: How should CrewAI agents address the Ollama LLM via LiteLLM? → A: Model string `ollama/qwen2.5:9b`; `api_base` resolved from `OLLAMA_HOST` env var (e.g., `http://ollama:11434` in Docker, `http://localhost:11434` locally).
- Q: What happens when market trend sources (DuckDuckGo, GitHub Search, HN Firebase) are unavailable? → A: Proceed with whichever sources respond; include a degraded-coverage notice in the Market Snapshot section of the report.

## Assumptions

- GitHub profiles analyzed are public; private profiles are out of scope for the initial version.
- CV files are assumed to be PDFs containing extractable text (not scanned images). Image-only PDFs are treated as invalid input.
- "Market trends" are gathered dynamically at pipeline execution time using DuckDuckGo web search (via the `duckduckgo-search` Python library, no API key required), GitHub Search API, and the HN Firebase API. The freshness of the data depends on source availability. If one or more sources are unreachable, the Researcher continues with available data and notes degraded coverage in the Market Snapshot; the pipeline does not fail solely due to source unavailability.
- The system runs as a single-user or low-concurrency service initially; high-scale multi-tenancy is out of scope.
- The report is the primary deliverable; no persistent user account, history, or dashboard is included in scope.
- The five candidate project ideas generated by the Strategist before passing top three to the Specifier is a reasonable default; this threshold may be adjusted during planning.
- Asynchronous job polling is the default delivery model for the API, given that full pipeline execution may take several minutes.
- The HTTP API requires no authentication. The service is intended to bind to localhost only; network exposure is out of scope.
- `uv` is the default Python package manager. `pip` is not used.
- The default Ollama model for all agents is `qwen2.5:9b`. The `OLLAMA_MODEL` environment variable MUST be set to `qwen2.5:9b` in all deployment configurations.
- A `docker-compose.yml` is a required deliverable. It MUST include the Scout API service and the Ollama service with CPU, AMD GPU, and Nvidia GPU profiles (matching the provided configuration). The Scout API container MUST connect to Ollama via the `OLLAMA_HOST` environment variable (default: `ollama:11434` within the Docker network).
- The `cpu` profile is the documented default for first-time users. GPU profiles (`gpu-amd`, `gpu-nvidia`) are opt-in alternatives and MUST be documented separately.
- All CrewAI agents MUST use LiteLLM model string `ollama/qwen2.5:9b`. The `api_base` URL MUST be read from the `OLLAMA_HOST` environment variable, defaulting to `http://localhost:11434` for local development and `http://ollama:11434` within Docker Compose.
- The in-memory job store has no TTL or automatic cleanup. Jobs accumulate indefinitely until explicitly deleted via a DELETE endpoint or process restart. Operators are responsible for periodic cleanup.
