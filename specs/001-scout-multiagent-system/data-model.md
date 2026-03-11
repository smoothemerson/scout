# Data Model: Scout Multi-Agent System

**Phase**: 1 — Design
**Branch**: `001-scout-multiagent-system`
**Date**: 2026-03-11

---

## Entities

### 1. AnalysisJob

Represents a single end-to-end Scout pipeline run.

| Field | Type | Description | Constraints |
|---|---|---|---|
| `id` | `str` | UUID4, job identifier | Unique, immutable |
| `status` | `JobStatus` | Overall pipeline state | Enum (see below) |
| `github_username` | `str` | GitHub username submitted | Non-empty, max 39 chars |
| `cv_file_path` | `str` | Absolute path to uploaded CV on disk | Non-empty, file must exist at job creation |
| `agent_statuses` | `dict[AgentName, AgentStatus]` | Per-agent execution state | Keys: analyst, researcher, strategist, specifier |
| `result` | `ScoutReport \| None` | Final report, populated on completion | None until completed |
| `error` | `str \| None` | Human-readable error message | None unless status=failed |
| `created_at` | `datetime` | UTC timestamp of job creation | Immutable |
| `started_at` | `datetime \| None` | UTC timestamp when pipeline started | Set when first agent begins |
| `completed_at` | `datetime \| None` | UTC timestamp of terminal state | Set on completed or failed |

**State machine**:

```
pending → running → completed
                 ↘ failed
```

`pending`: Job record created, pipeline not yet started.
`running`: At least one agent is executing.
`completed`: All four agents finished, `result` is populated.
`failed`: An unrecoverable error occurred; `error` is populated.

**AgentStatus enum**: `pending | running | completed | failed`
**AgentName enum**: `analyst | researcher | strategist | specifier`
**JobStatus enum**: `pending | running | completed | failed`

---

### 2. UserProfile

The merged representation of the developer's GitHub data and CV content. Output of the Analyst agent.

| Field | Type | Description |
|---|---|---|
| `languages` | `list[LanguageStat]` | Programming languages with usage percentage |
| `frameworks` | `list[str]` | Detected frameworks and tools from repos and CV |
| `domains` | `list[str]` | Repository domains (e.g., web, data, ml, devops) |
| `commit_activity` | `CommitActivity` | Weekly commit patterns summary |
| `cv_skills` | `list[str]` | Skills explicitly listed in CV |
| `cv_experience_domains` | `list[str]` | Work experience domains from CV |
| `cv_education` | `str` | Education background summary |
| `gap_analysis` | `list[SkillGap]` | Identified gaps between CV and GitHub activity |

**LanguageStat**:

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Language name (e.g., "Python") |
| `percentage` | `float` | % of total bytes across public repos |
| `repo_count` | `int` | Number of repos using this language |

**CommitActivity**:

| Field | Type | Description |
|---|---|---|
| `total_commits_last_year` | `int` | Sum across all public repos |
| `most_active_days` | `list[str]` | Day names with highest commit frequency |
| `active_repos_count` | `int` | Repos with at least 1 commit in last 90 days |

**SkillGap**:

| Field | Type | Description |
|---|---|---|
| `skill` | `str` | The gap skill area (e.g., "ML model deployment") |
| `source` | `GapSource` | Where the gap was detected |
| `description` | `str` | One-sentence explanation of the gap |

**GapSource enum**: `cv_without_github | github_without_cv | inferred`

---

### 3. MarketSnapshot

The Researcher agent's output. Represents market intelligence gathered at pipeline execution time.

| Field | Type | Description |
|---|---|---|
| `gathered_at` | `datetime` | UTC timestamp when data was collected |
| `trending_repos` | `list[TrendingRepo]` | Top trending GitHub repos in AI/ML domains |
| `hiring_signals` | `list[HiringSignal]` | Signals from HN hiring threads and job APIs |
| `web_insights` | `list[WebInsight]` | Key findings from DuckDuckGo search queries |
| `top_domains` | `list[str]` | Most active domains (e.g., "llm-agents", "rag", "mlops") |
| `degraded` | `bool` | `True` if one or more market sources were unreachable during collection |
| `degraded_note` | `str \| None` | Human-readable description of which sources failed; `None` if not degraded |

**TrendingRepo**:

| Field | Type | Description |
|---|---|---|
| `name` | `str` | `owner/repo` slug |
| `description` | `str` | Repo description |
| `stars` | `int` | Current star count |
| `topics` | `list[str]` | GitHub topics |
| `language` | `str \| None` | Primary language |

**HiringSignal**:

| Field | Type | Description |
|---|---|---|
| `source` | `str` | Source identifier (e.g., "hn-who-is-hiring-2026-03") |
| `technology` | `str` | Mentioned technology or skill (e.g., "RAG", "PyTorch") |
| `mention_count` | `int` | How many job posts mention this |

**WebInsight**:

| Field | Type | Description |
|---|---|---|
| `query` | `str` | Search query that produced this insight |
| `summary` | `str` | LLM-synthesized summary of search results |
| `urls` | `list[str]` | Source URLs referenced |

---

### 4. ProjectIdea

A ranked candidate project opportunity. Output of the Strategist agent.

| Field | Type | Description |
|---|---|---|
| `rank` | `int` | Position in ranked list (1 = highest impact) |
| `title` | `str` | Short project title |
| `rationale` | `str` | One-paragraph justification |
| `market_signals` | `list[str]` | Specific market signals supporting this idea |
| `profile_evidence` | `list[str]` | Specific profile strengths/gaps supporting this idea |
| `technology_direction` | `str` | Recommended tech direction (not specific tools) |
| `estimated_impact` | `str` | Qualitative impact description (e.g., "high", "medium") |

**Constraints**:
- The Strategist produces at least 5 `ProjectIdea` objects before passing the top 3 to the Specifier.
- Each idea must reference at least 1 `market_signals` entry and at least 1 `profile_evidence` entry (FR-004, SC-004).

---

### 5. ProjectPRD

A mini Product Requirements Document for one project idea. Output of the Specifier agent.

| Field | Type | Description |
|---|---|---|
| `project_idea_rank` | `int` | References the parent `ProjectIdea.rank` |
| `title` | `str` | Project title (matches `ProjectIdea.title`) |
| `problem_statement` | `str` | One-paragraph problem description |
| `target_user` | `str` | Primary user persona |
| `core_features` | `list[str]` | 3–5 core feature descriptions |
| `mvp_scope` | `str` | What is included and excluded from the MVP |
| `success_criteria` | `list[str]` | Measurable success metrics |

**Constraints**:
- `core_features` must have 3–5 items (FR-009).
- `success_criteria` must have at least 1 item.

---

### 6. ScoutReport

The final deliverable. Compiled by the pipeline orchestrator after all four agents complete.

| Field | Type | Description |
|---|---|---|
| `job_id` | `str` | References the parent `AnalysisJob.id` |
| `generated_at` | `datetime` | UTC timestamp |
| `user_profile` | `UserProfile` | Full Analyst output |
| `market_snapshot` | `MarketSnapshot` | Full Researcher output |
| `all_project_ideas` | `list[ProjectIdea]` | All 5+ ranked ideas from Strategist |
| `top_project_ideas` | `list[ProjectIdea]` | Top 3 ideas passed to Specifier |
| `prds` | `list[ProjectPRD]` | One PRD per top-3 idea |
| `markdown` | `str` | Fully rendered markdown report |

**Constraints**:
- `top_project_ideas` must have exactly 3 items (FR-010, SC-002).
- `prds` must have exactly 3 items, one per `top_project_ideas` entry.

---

## State Transitions

### AnalysisJob lifecycle

```
POST /jobs received
  └─→ AnalysisJob created (status=pending, all agent_statuses=pending)
      └─→ asyncio.create_task() called
          └─→ crew.kickoff_async() starts
              ├─→ agent_statuses["analyst"] = running
              │   └─→ analyst completes → agent_statuses["analyst"] = completed
              │       └─→ agent_statuses["researcher"] = running
              │           └─→ researcher completes → agent_statuses["researcher"] = completed
              │               └─→ agent_statuses["strategist"] = running
              │                   └─→ strategist completes → agent_statuses["strategist"] = completed
              │                       └─→ agent_statuses["specifier"] = running
              │                           └─→ specifier completes → agent_statuses["specifier"] = completed
              │                               └─→ ScoutReport assembled
              │                                   └─→ job.result = report
              │                                       └─→ job.status = completed
              └─→ (on any exception)
                  └─→ current agent → failed
                      └─→ job.status = failed, job.error = message
```

---

## Pydantic Output Models for CrewAI Tasks

The following Pydantic models are passed as `output_pydantic` on CrewAI Tasks to enforce structured handoffs between agents:

| Agent | Task Output Model |
|---|---|
| Analyst | `UserProfile` |
| Researcher | `MarketSnapshot` |
| Strategist | `list[ProjectIdea]` (5+ items) |
| Specifier | `list[ProjectPRD]` (3 items) |
