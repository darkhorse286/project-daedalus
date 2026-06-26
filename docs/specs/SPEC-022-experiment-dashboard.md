# SPEC-022: Experiment Dashboard

## Metadata

**Feature ID:** SPEC-022

**Title:** Experiment Dashboard

**Status:** Draft

**Author:** Darkhorse286

**Created:** 2026-06-25

**Last Updated:** 2026-06-25

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-002, ADR-006, ADR-009, ADR-012, ADR-013

**Related Specs:** SPEC-001, SPEC-003, SPEC-005, SPEC-006, SPEC-007, SPEC-008, SPEC-009, SPEC-020, SPEC-021

---

# Problem Statement

SPEC-021 introduces a Web UI providing browser-accessible job submission, status monitoring, and experiment observation. SPEC-021 FR-8 exposes experiment state and trial tables as read-only views; the experiment summary (FR-8.3) and benchmark summary (FR-8.4) are presented as raw JSON payloads available for download.

Without a structured visualization layer:

- The experiment summary artifact (SPEC-020 FR-14 Artifact 3) contains per-(problem, solver) quality statistics, runtime statistics, cross-solver rankings, and outcome distributions — but is presented as an opaque JSON blob with no structured presentation.
- Comparing solver outcomes across trials requires manual interpretation of the summary JSON rather than a readable comparison interface.
- The project's core thesis — that most routing problems should not use expensive quantum-adjacent backends because classical heuristics are sufficient — cannot be demonstrated to a technical audience through a browser-based interface without external tooling.
- The benchmark summary (SPEC-020 FR-14 Artifact 4) aggregates cross-experiment evidence but is not presented in a form that supports reading the comparative conclusion directly.
- Evidence reports (SPEC-009) are accessible individually but not navigable from trial context; no cross-trial view links trials to their respective evidence artifacts.
- Scheduler decision information (SPEC-009 Section 3) is embedded in individual evidence reports; there is no aggregate view of scheduler targeting behavior across an experiment's trials.

The Experiment Dashboard provides a browser-based evidence visualization and comparison interface on top of the already-produced artifacts and API endpoints. It does not introduce new backend capabilities, modify existing API contracts, or duplicate experiment orchestration responsibilities.

---

# Business Value

- Makes the experiment summary artifact immediately useful: cross-solver quality comparison, outcome distributions, and runtime statistics are presented as structured, readable views without requiring JSON export and manual analysis
- Demonstrates the project thesis directly in the browser: the Dashboard surfaces that classical solvers achieved lower total route distance in N of M trials without external tooling
- Provides a navigable evidence chain from experiment → trial → job → evidence report, enabling a reviewer to drill from aggregate results to individual execution evidence
- Signals full-stack engineering and evidence-driven system design as portfolio artifacts
- Enables technical stakeholders to read comparative experiment outcomes without CLI access or JSON literacy

---

# Employer Signaling

- Frontend Development
- Full-Stack Engineering
- Evidence-Driven System Design
- Data Visualization
- API Contract Consumption

---

# Domain Concept

The Experiment Dashboard is a browser-based visualization and navigation layer for the evidence artifacts produced by the Project DAEDALUS optimization runtime and experiment harness. It is a consumer of SPEC-008 API endpoints and the artifacts those endpoints expose.

The Dashboard does not produce evidence. It does not compute statistics. It does not run experiments. It presents the artifacts that the runtime, experiment harness, and evidence generation pipeline have already produced.

From an architectural perspective, the Dashboard is a browser client in the system context, closely related to the Web UI (SPEC-021):

```
Browser [User]
  → Experiment Dashboard
    → Daedalus API (SPEC-008)
      → (existing artifacts: ExperimentSummary, BenchmarkSummary, evidence reports, trial records)
```

The Dashboard renders the structured experiment summary from SPEC-008 FR-23 as a tabular comparison view. It renders the benchmark summary from SPEC-008 FR-24 as a cross-experiment evidence view. It renders the trial results from SPEC-008 FR-22 as a navigable comparison matrix. It links trials to their evidence reports via the job detail chain (SPEC-008 FR-8, FR-12, FR-13).

The Dashboard does not define new API endpoints. All data it displays originates from existing SPEC-008 endpoints.

**Experiment context and scheduler targeting:** All experiment trials are submitted with an explicit `backend_id` per ADR-013 Decision 2. Every trial in an experiment uses the targeted execution path (`selection_mode = explicitly_targeted`). The Dashboard presents this as context: in experiment mode, the experiment harness directs each trial to a specific backend; the Scheduler validates eligibility and executes without policy-based selection. The evidence report for each trial (SPEC-009 Section 3) captures this in the rendered scheduler decision section.

---

# Scope and Responsibility Boundary

| Responsibility | Owner |
|---|---|
| HTTP API contract for all endpoints consumed by the Dashboard | SPEC-008 |
| Experiment summary artifact content and schema | SPEC-020 |
| Benchmark summary artifact content and schema | SPEC-020 |
| Trial record content and field definitions | SPEC-008 FR-22 |
| Quality stats aggregate content and computation | SPEC-020 FR-12, FR-13 |
| Evidence report generation and content | SPEC-009 |
| Evidence report serving | SPEC-008 FR-12, FR-13 |
| Experiment orchestration and trial submission | SPEC-016 (CLI) |
| Job submission and individual job monitoring | SPEC-021 (Web UI) |
| Basic experiment observation (status view, raw trial table, raw summary JSON) | SPEC-021 (Web UI) FR-8 |
| Presenting structured experiment summary as tabular comparison views | **SPEC-022 (this spec)** |
| Presenting trial comparison matrix organized by (problem_config, solver) | **SPEC-022 (this spec)** |
| Presenting cross-solver quality comparison and outcome distribution | **SPEC-022 (this spec)** |
| Presenting structured benchmark summary and cross-experiment evidence | **SPEC-022 (this spec)** |
| Navigating from trial to job detail and evidence report | **SPEC-022 (this spec)** |
| Presenting scheduler targeting context at the experiment level | **SPEC-022 (this spec)** |
| Presenting execution metadata in a sortable, filterable table | **SPEC-022 (this spec)** |
| Downloading evidence artifacts (reports, summary JSON) | **SPEC-022 (this spec)** |

The Dashboard does not:

- Submit routing jobs, experiment manifests, or benchmark manifests (SPEC-021 and CLI responsibilities)
- Trigger trial evidence collection (CLI responsibility per ADR-012 Decision 5)
- Compute or recompute quality statistics, rankings, or outcome distributions — all aggregate data is consumed from the SPEC-008 FR-23 experiment summary artifact produced at experiment completion
- Modify any experiment, trial, or job record state; all Dashboard interactions are read-only
- Access PostgreSQL, RabbitMQ, the report volume, the Worker, or any infrastructure component directly
- Perform SPEC-001 domain validation
- Generate, modify, or re-process HTML evidence reports
- Expose OpenTelemetry, Prometheus, or Grafana data

---

# Requirements

## FR-1: Scope and Component Responsibilities

**Description:**
SPEC-022 defines the Dashboard's responsibilities and explicitly allocates adjacent responsibilities it must not perform.

| Component | Responsibilities at This Boundary | Explicitly Not Responsible For |
|---|---|---|
| **Dashboard** (SPEC-022) | HTTP GET requests to SPEC-008 read endpoints; structured rendering of experiment summary, benchmark summary, and trial data; cross-trial and cross-solver comparison views; navigation from experiment to job to evidence report | Writing to any API resource; experiment orchestration; evidence computation; report generation; job submission |
| **API** (SPEC-008) | All evidence artifact computation, persistence, and serving; endpoint contract enforcement | Rendering comparison views; computing statistics on demand |
| **CLI** (SPEC-016) | Experiment orchestration; trial submission; evidence collection | Dashboard interaction |
| **Web UI** (SPEC-021) | Job submission; individual job monitoring; basic experiment observation (FR-8) | Dashboard comparison and visualization views |

**Acceptance Criteria:**
- No Dashboard code path issues any HTTP method other than GET against the Daedalus API
- The Dashboard does not emit OpenTelemetry spans or inject `traceparent` headers into API requests
- The Dashboard does not access PostgreSQL, RabbitMQ, the report volume, the Worker, or the Python Solver Adapter directly
- The Dashboard does not submit experiment manifests, trigger evidence collection, or modify any record state

---

## FR-2: Experiment Entry and Browser

**Description:**
The Dashboard provides an entry point for accessing experiment data. The primary entry path is by `experiment_id`. An experiment list view is contingent on OQ-3.

**Experiment lookup:**
The user enters an `experiment_id` (UUID) via an entry form. The Dashboard issues `GET /v1/experiments/{experiment_id}` (SPEC-008 FR-20) and, on HTTP 200, navigates to the Experiment Overview (FR-3).

A UUID format check is applied client-side before the API request is issued. An entry that fails the UUID format check is rejected with an inline error before an API call is made.

**Experiment list (contingent on OQ-3):**
If a `GET /v1/experiments` list endpoint is defined (SPEC-008 OQ-1, SPEC-021 OQ-1), the Dashboard provides a browseable list of experiments ordered most-recently-created first. The list displays:

| Column | Source | Notes |
|---|---|---|
| Experiment Name | `experiment_name` | |
| Benchmark | `benchmark_id` | Links to FR-8 benchmark view |
| Status | `status` | Visually differentiated |
| Trial Progress | `trial_counts` summary | Completed trials / planned |
| Reproducibility | `reproducibility_class` | |
| Submitted At | `submitted_at` | ISO 8601 UTC |

**Fallback when OQ-3 is unresolved:**
If no list endpoint is available, the experiment ID entry form is the initial view. No other Dashboard capability is degraded by the absence of a list endpoint.

**Acceptance Criteria:**
- The experiment ID entry form applies a UUID format check before issuing an API request; a malformed entry displays an inline error without an API call
- A valid UUID entry issues `GET /v1/experiments/{experiment_id}` and, on HTTP 200, navigates to FR-3
- `EXPERIMENT_NOT_FOUND` (HTTP 404) displays a not-found message; the entry form remains accessible
- When the list endpoint is available (OQ-3 resolved), experiments are listed ordered most-recently-created first with the columns defined above
- The `status` column is visually differentiated across `Created`, `Running`, `Completed`, and `Failed`

---

## FR-3: Experiment Overview

**Description:**
The Experiment Overview is the landing view for a single experiment. It presents the top-level experiment state and provides navigation to comparison and detail views.

**Data source:** `GET /v1/experiments/{experiment_id}` (SPEC-008 FR-20)

**Fields displayed:**

| Field | Condition | Notes |
|---|---|---|
| `experiment_id` | Always | Full UUID; copyable |
| `experiment_name` | Always | |
| `benchmark_id` | Always | Links to FR-8 benchmark view |
| `status` | Always | Visually differentiated: `Created`, `Running`, `Completed`, `Failed` |
| `planned_trial_count` | Always | |
| `effective_trial_count` | When set | |
| `workload_mode` | Always | `Fixed` or `Generated` |
| `reproducibility_class` | Always | With explanatory note per FR-5 reproducibility annotation |
| `submitted_at` | Always | ISO 8601 UTC |
| `started_at` | When Running or later | ISO 8601 UTC |
| `completed_at` | When `Completed` | ISO 8601 UTC |
| `failed_at` | When `Failed` | ISO 8601 UTC |
| `failure_reason` | When `Failed` | |
| `trial_counts` | Always | Status category breakdown |

**Trial counts display:**
The `trial_counts` object is displayed as a compact breakdown showing: `pending`, `submitted`, `completed`, `scheduler_rejected`, `harness_error`, and `evidence_collected` counts. The `executing` count is reserved for future use (always 0 at MVP scope per SPEC-008 FR-20) and is not displayed at MVP scope.

**Navigation from overview:**
- Link to Trial Matrix (FR-4) for this experiment
- Link to Solver Comparison (FR-5) when `status = Completed`
- Link to Experiment Summary Detail (FR-6) when `status = Completed`
- Link to Benchmark Summary (FR-8) for this experiment's `benchmark_id`

**Auto-refresh:**
For non-terminal experiments (`Created`, `Running`), the overview auto-refreshes by re-issuing `GET /v1/experiments/{experiment_id}` on a defined interval. Auto-refresh stops when `status` transitions to `Completed` or `Failed`.

**Acceptance Criteria:**
- All fields from SPEC-008 FR-20 are displayed, conditioned on `status` appropriately
- `completed_at` and `failed_at` are mutually exclusive; only the applicable field is displayed
- Navigation links to FR-5 and FR-6 appear only when `status = Completed`
- Auto-refresh updates the displayed `trial_counts` and `status` without a full page reload for non-terminal experiments
- Auto-refresh stops when the experiment reaches `Completed` or `Failed`
- A `404 Not Found` for the initial `GET /v1/experiments/{experiment_id}` displays a not-found message and allows returning to the FR-2 entry form

---

## FR-4: Trial Matrix View

**Description:**
The Trial Matrix View presents all trials for an experiment organized as a comparison matrix, enabling cross-solver and cross-problem inspection in a single view.

**Data source:** `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22). No additional per-trial API calls are issued to render the matrix.

**Matrix organization:**
Rows are indexed by `(problem_config_index, repetition_index)` — one row per unique (problem, repetition) combination. Columns correspond to the distinct `backend_id` values in the experiment's solver set. Each cell represents the trial at that `(problem_config_index, repetition_index, backend_id)` intersection.

**Cell content:**

| Item | Condition | Source field |
|---|---|---|
| Trial status indicator | Always | `trial_status`; visually differentiated |
| Solver outcome | When evidence collected | `solver_outcome` |
| Hindsight quality (km) | When available | `hindsight_quality`; null displayed as "—" |
| Execution duration (ms) | When available | `execution_duration_ms`; null displayed as "—" |
| Evidence status | Always | `evidence_status` |
| Rejection or error reason | When applicable | `scheduler_rejection_reason` or `harness_failure_reason` |
| Job link | When `job_id` non-null | Navigates to SPEC-021 FR-6 job detail view for that `job_id` |
| Report link | When `trial_status = Completed` and `job_id` non-null | Navigates to FR-7; availability confirmed at navigation time via SPEC-008 FR-12 |

**Best-quality highlight:**
Within each row, the cell with the lowest `hindsight_quality` among trials with `quality_comparison_eligible = true` is visually indicated. This renders the per-row solver ranking visible without navigating to the summary view.

`hindsight_quality` is dimensionally comparable only within the same routing problem (SPEC-007 FR-7; ADR-012 Decision 3 instance-sharing invariant). The highlight is scoped per row — same `(problem_config_index, repetition_index)` — and does not imply any cross-row comparison. Cells with `quality_comparison_eligible = false` or null `hindsight_quality` are not included in the row-level highlight.

**Acceptance Criteria:**
- The matrix is derived entirely from a single `GET /v1/experiments/{experiment_id}/trials` response; no additional `GET /v1/jobs/{job_id}` calls are issued while rendering matrix cells
- Rows are ordered by `problem_config_index` ascending, then `repetition_index` ascending
- Columns correspond to solver backends in the experiment's solver set
- Within each row, the cell with the lowest `hindsight_quality` among `quality_comparison_eligible = true` trials is visually indicated; cells with `quality_comparison_eligible = false` or null `hindsight_quality` are excluded from the row-level highlight
- Cells with `job_id = null` render a placeholder with no job link or report link
- `SchedulerRejected` cells render `scheduler_rejection_reason`; `HarnessError` cells render `harness_failure_reason`
- Auto-refresh for non-terminal experiments re-fetches trial data at the same interval as FR-3 auto-refresh

---

## FR-5: Solver Comparison View

**Description:**
The Solver Comparison View presents per-solver aggregate statistics derived from the experiment summary artifact. It is the primary evidence view for the project's thesis claim about solver backend performance across the evaluated workload.

**Data source:** `GET /v1/experiments/{experiment_id}/summary` (SPEC-008 FR-23). Available only when `status = Completed`.

**Unavailable state:**
When the experiment has not yet reached `Completed`, this view displays a "Summary not yet available" message with a link back to FR-3. `EXPERIMENT_SUMMARY_NOT_FOUND` (HTTP 404) is the expected condition; it is presented as a not-yet-available state, not an error.

**Per-solver quality statistics:**
A comparison table with one row per solver backend presents the following fields derived from the QualityStatsAggregate entries in the summary payload (field labels; exact field names are per SPEC-020 FR-12, FR-14):

| Column | Description |
|---|---|
| Backend | `backend_id` |
| Sample Count | Number of trials contributing to the aggregate |
| Mean Hindsight Quality (km) | Mean `hindsight_quality` across eligible trials; lower is better |
| Std Dev (km) | Standard deviation of `hindsight_quality` |
| Min (km) | Minimum `hindsight_quality` |
| Max (km) | Maximum `hindsight_quality` |
| Eligible Trial Count | Trials with `quality_comparison_eligible = true` |

**Per-solver runtime statistics:**

| Column | Description |
|---|---|
| Mean Duration (ms) | Mean `execution_duration_ms` |
| Std Dev (ms) | Standard deviation |
| Min (ms) | Minimum |
| Max (ms) | Maximum |

**Per-solver outcome distribution:**
For each solver, the count of each `solver_outcome` value across all trials is displayed: `Succeeded`, `Infeasible`, `Timeout`, `Cancelled`, `Failed`, `ContractViolation`.

**Cross-solver ranking:**
The cross-solver ranking from the summary payload is presented without recomputation. A note is displayed alongside the ranking: "Rankings are based on quality-comparison-eligible trials sharing the same routing problem instance (SPEC-007 FR-7)." When no quality-comparison-eligible trials exist across any (problem, repetition) intersection, a note is displayed indicating that quality comparison is not applicable for this experiment.

**Reproducibility annotation:**
The experiment's `reproducibility_class` is displayed with a contextual note:

| Class | Note |
|---|---|
| `fully_reproducible` | All solvers are deterministic or seeded-stochastic; results are reproducible under the ADR-010 seed policy |
| `partially_reproducible` | One or more solvers are stochastic with seeds; variance across repetitions is expected and reflects solver behavior |
| `non_reproducible` | One or more solvers include hardware execution; shot outcomes are inherently non-deterministic and cannot be reproduced from seed |

**Targeted execution note:**
A note is displayed indicating that all trials in this experiment used explicitly targeted backend execution (`selection_mode = explicitly_targeted` per ADR-013). The Scheduler validated each backend's eligibility but did not score or select among candidates. This context distinguishes experiment-mode execution from standard policy-selected execution.

**Download:**
The full experiment summary JSON payload is offered for download without transformation.

**Acceptance Criteria:**
- The Solver Comparison View is accessible only when `GET /v1/experiments/{experiment_id}/summary` returns HTTP 200
- `EXPERIMENT_SUMMARY_NOT_FOUND` displays a not-yet-available message with a link to FR-3; the message is distinct from a not-found error
- The quality statistics comparison table has one row per solver backend in the experiment's solver set
- The outcome distribution is displayed per solver
- The cross-solver ranking is derived from the summary payload without recomputation by the Dashboard
- The cross-solver ranking note is displayed alongside the ranking
- When no quality-comparison-eligible trials exist, a note explains that quality comparison is not applicable; the quality statistics table is still displayed for runtime and outcome data
- The `reproducibility_class` annotation is always displayed
- The targeted execution note is always displayed
- The experiment summary JSON is offered for download

---

## FR-6: Experiment Summary Detail

**Description:**
The Experiment Summary Detail presents the full experiment summary artifact from SPEC-008 FR-23 in a structured, readable form. It surfaces the per-(problem, solver) scope aggregates and the experiment-level aggregate beyond what FR-5's per-solver view shows.

**Data source:** `GET /v1/experiments/{experiment_id}/summary` (SPEC-008 FR-23). Shares the same fetch as FR-5 when accessed in the same session.

**Per-(problem, solver) aggregates:**
Each QualityStatsAggregate artifact in the summary is presented in a table row. Logical columns include: `problem_config_index`, `backend_id`, sample count, mean hindsight quality (km), quality std dev, outcome distribution, mean execution duration (ms). Exact field names are per SPEC-020 FR-12, FR-14.

**Experiment-level aggregate:**
The per-experiment aggregate stats covering all trials across all problems and all solvers are presented as a distinct summary row, visually differentiated from the per-(problem, solver) rows.

**Cross-solver comparison section:**
The cross-solver comparison from the summary payload is presented as a ranked table. The ranking and methodology are those computed by the API at experiment completion; the Dashboard does not recompute them.

**Acceptance Criteria:**
- All QualityStatsAggregate artifacts in the payload are displayed, one row per `(problem_config_index, backend_id)` scope
- The experiment-level aggregate is displayed as a visually distinct summary row
- The cross-solver comparison reflects the summary payload without Dashboard-side recomputation
- `EXPERIMENT_SUMMARY_NOT_FOUND` displays a not-available message with a link to FR-3

---

## FR-7: Evidence Report Navigation and Rendering

**Description:**
The Dashboard provides navigation from a trial to its HTML evidence report. This enables drill-down from aggregate views to the individual execution evidence backing each data point.

**Navigation trigger:**
From the Trial Matrix (FR-4) and Execution Metadata View (FR-9), each trial with `trial_status = Completed` and `job_id` non-null provides a report navigation link.

**Report access flow:**
1. The Dashboard issues `GET /v1/jobs/{job_id}/report` (SPEC-008 FR-12) for the trial's `job_id`.
2. On HTTP 200: the Dashboard fetches the report content via `GET /v1/reports/{report_id}` using the `report_url` from the FR-12 response (SPEC-008 FR-13).
3. The HTML report content is rendered in-browser without modification.
4. A download option allows saving the HTML report file to disk.

**Report metadata displayed:**
Alongside the report content:

| Field | Source |
|---|---|
| `report_id` | SPEC-008 FR-12 response |
| `generated_at` | SPEC-008 FR-12 response; ISO 8601 UTC |
| `report_format` | SPEC-008 FR-12 response |
| `job_id` | SPEC-008 FR-12 response; links back to job detail (SPEC-021 FR-6) |

**Scheduler decision context:**
The HTML evidence report rendered in this view includes Section 3 (Scheduler Decision, SPEC-009 FR-4), which presents the full scheduler decision record including `selection_mode`, selected backend, predicted values, confidence score, candidate scores, and rejection reasons. The Dashboard does not extract, re-render, or summarize these fields separately; the evidence report is the authoritative per-trial presentation of scheduler decision detail.

**Not-available state:**
When `GET /v1/jobs/{job_id}/report` returns HTTP 404, the view displays a not-available message explaining that the report may not have been produced (job in `Failed` state, report generation failed, or job still executing). The Trial Matrix view remains accessible.

**Acceptance Criteria:**
- Every Trial Matrix cell and Execution Metadata row with `trial_status = Completed` and `job_id` non-null provides a report navigation link
- The report HTML content is rendered in-browser byte-identical to the SPEC-008 FR-13 response body
- The Dashboard does not inject any content into the rendered report HTML
- A download option allows saving the HTML file to disk
- The four metadata fields are displayed alongside the report content
- `job_id` links to the job detail view (SPEC-021 FR-6 or equivalent)
- HTTP 404 on report discovery displays a not-available message without error state; the Trial Matrix remains accessible
- When report content is rendered in a frame within the Dashboard, appropriate browser sandboxing is applied per Security Considerations

---

## FR-8: Benchmark Summary View

**Description:**
The Benchmark Summary View presents the benchmark summary artifact from SPEC-008 FR-24 in a structured form that supports reading cross-experiment evidence.

**Data source:** `GET /v1/benchmarks/{benchmark_id}/summary` (SPEC-008 FR-24)

**Benchmark lookup:**
The user enters a `benchmark_id` string via a lookup form. No benchmark list view is included at MVP scope.

**Research context section (when benchmark manifest exists):**
When the summary payload includes benchmark manifest fields, the following are displayed:

| Field | Notes |
|---|---|
| `benchmark_name` | |
| `research_question` | |
| `hypothesis` | |
| `null_hypothesis` | |
| `controls` | Bulleted list |
| `independent_variables` | Bulleted list |
| `dependent_variables` | Bulleted list |

When the summary payload does not include research context fields (no benchmark manifest was submitted), this section is omitted.

**Member experiment table:**
All experiments referenced in the summary payload are listed:

| Column | Notes |
|---|---|
| Experiment Name | |
| Status | |
| Reproducibility Class | |
| Planned / Completed Trials | |
| Link | Navigates to FR-3 for that experiment |

**Cross-experiment comparison section:**
When the summary payload includes cross-experiment quality comparison data, it is presented as a structured table. The comparison is derived from the payload without recomputation by the Dashboard.

**Download:**
The full benchmark summary JSON payload is offered for download without transformation.

**Not-available state:**
`BENCHMARK_NOT_FOUND` (HTTP 404) — no member experiment has yet completed — displays a not-yet-available message, not a generic not-found error.

**Acceptance Criteria:**
- The benchmark summary is accessible by `benchmark_id` entry
- The research context section is displayed when the payload includes manifest fields; it is omitted when absent
- The member experiment table displays all experiments in the payload; each entry links to FR-3 for that experiment
- The cross-experiment comparison section is displayed when the payload includes such data; it is omitted otherwise
- The benchmark summary JSON is offered for download
- `BENCHMARK_NOT_FOUND` displays a not-yet-available message distinct from a not-found error

---

## FR-9: Execution Metadata View

**Description:**
The Execution Metadata View presents per-trial execution detail for an experiment, focused on timing and outcome metadata. It complements the Trial Matrix (FR-4) by providing a flat, sortable table suitable for identifying outliers in duration or outcome.

**Data source:** `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22). No additional per-trial API calls are issued.

**Table columns:**

| Column | Source field | Notes |
|---|---|---|
| Trial ID | `trial_id` | Abbreviated UUID; full value accessible |
| Problem Config | `problem_config_index` | |
| Repetition | `repetition_index` | |
| Solver Set Index | `solver_set_index` | |
| Backend | `backend_id` | |
| Trial Status | `trial_status` | Visually differentiated |
| Solver Outcome | `solver_outcome` | When evidence collected; blank otherwise |
| Duration (ms) | `execution_duration_ms` | When available; null displayed as "—" |
| Solution Present | `solution_present` | When available |
| Evidence Status | `evidence_status` | When terminal |
| Job Link | `job_id` | Links to job detail when non-null; blank when null |
| Report | When `trial_status = Completed` | Link navigates to FR-7 |

**Sortable columns:** The table is client-side sortable by `execution_duration_ms`, `backend_id`, `trial_status`, `solver_outcome`, `problem_config_index`, and `repetition_index`. Sorting does not issue additional API requests.

**Acceptance Criteria:**
- All trial rows are derived from a single `GET /v1/experiments/{experiment_id}/trials` response; no per-trial API calls are issued
- The table is sortable by the designated columns without additional API requests
- `job_id` entries render a link to job detail when non-null; null entries render a blank placeholder
- Null `execution_duration_ms` values are displayed as "—"
- Report links are rendered for trials with `trial_status = Completed`

---

## FR-10: Observability Metadata Display

**Description:**
The Dashboard surfaces system observability metadata that is accessible through the SPEC-008 API. It does not query the OpenTelemetry Collector, Prometheus, or Grafana.

**Job timing (from SPEC-008 FR-8):**
When the user navigates to job detail from any Dashboard view (via a `job_id` link), the following timing fields from `GET /v1/jobs/{job_id}` are displayed:

| Field | Condition | Notes |
|---|---|---|
| `created_at` | Always | ISO 8601 UTC |
| `updated_at` | Always | ISO 8601 UTC |
| `completed_at` | When `status = Completed` | ISO 8601 UTC |
| `failed_at` | When `status = Failed` | ISO 8601 UTC |

**Experiment timeline (from SPEC-008 FR-20):**
The FR-3 Experiment Overview displays `submitted_at`, `started_at`, and `completed_at` to provide a view of the experiment's total wall-clock duration.

**`request_id` propagation:**
The `request_id` from SPEC-008 FR-14 error responses is displayed in all API error conditions to support operator correlation with API observability data. The `request_id` is propagated unchanged from the API error body.

**Acceptance Criteria:**
- Job timing fields are displayed in any job detail view reached from the Dashboard
- The experiment timeline fields are displayed in FR-3
- The Dashboard does not query OTel, Prometheus, or Grafana
- `request_id` is displayed for all API error responses; it is not regenerated or altered by the Dashboard

---

## FR-11: Cross-View Navigation

**Description:**
The Dashboard provides consistent navigation across its views, enabling the user to move between experiment, trial, job, and report contexts without losing position.

**Navigation paths:**

| From | To | Trigger |
|---|---|---|
| Experiment Browser (FR-2) | Experiment Overview (FR-3) | Row selection or entry form submission |
| Experiment Overview (FR-3) | Trial Matrix (FR-4) | Navigation link |
| Experiment Overview (FR-3) | Solver Comparison (FR-5) | Navigation link (when `status = Completed`) |
| Experiment Overview (FR-3) | Experiment Summary Detail (FR-6) | Navigation link (when `status = Completed`) |
| Experiment Overview (FR-3) | Benchmark Summary (FR-8) | `benchmark_id` link |
| Trial Matrix (FR-4) | Execution Metadata View (FR-9) | Navigation link |
| Trial Matrix (FR-4) | Job Detail (SPEC-021 FR-6) | `job_id` link per cell |
| Trial Matrix (FR-4) | Evidence Report (FR-7) | Report link per cell |
| Solver Comparison (FR-5) | Trial Matrix (FR-4) | Navigation link |
| Execution Metadata (FR-9) | Job Detail (SPEC-021 FR-6) | `job_id` link per row |
| Execution Metadata (FR-9) | Evidence Report (FR-7) | Report link per row |
| Evidence Report (FR-7) | Job Detail (SPEC-021 FR-6) | `job_id` metadata link |
| Benchmark Summary (FR-8) | Experiment Overview (FR-3) | Per-experiment row link |

**Breadcrumb navigation:**
All views except the top-level entry view (FR-2) display a breadcrumb trail reflecting the navigation path. The breadcrumb allows returning to any ancestor view. The breadcrumb is derived from navigation context, not from a separate API call.

**Acceptance Criteria:**
- All navigation links defined above are present and functional when their preconditions are met
- Navigation links conditioned on `status = Completed` are not visible for non-terminal experiments
- A breadcrumb trail is present on all views except the FR-2 entry view
- Breadcrumb navigation does not re-issue API requests for data that was already fetched and is still current

---

## FR-12: API Error Display

**Description:**
All SPEC-008 API error responses are surfaced to the user in a consistent, accessible presentation. The Dashboard propagates SPEC-008 FR-14 structured error information without discarding fields or adding fabricated detail.

**Error presentation:**

| HTTP status | `error_code` | Dashboard presentation |
|---|---|---|
| 404 | `EXPERIMENT_NOT_FOUND` | "No experiment found with this ID. Verify the experiment_id and try again." |
| 404 | `EXPERIMENT_SUMMARY_NOT_FOUND` | "Experiment summary is not yet available. The experiment must reach Completed status before the summary can be viewed." |
| 404 | `BENCHMARK_NOT_FOUND` | "No benchmark summary is available yet. A member experiment must complete before the summary can be viewed." |
| 404 | Any other | Contextual not-found message identifying the resource type and supplied identifier |
| 500 | `INTERNAL_ERROR` | "An unexpected server error occurred. Reference request_id {request_id} for support." |
| Network error | — | "Unable to reach the Daedalus API. Verify the API is running and accessible at the configured URL." |

`EXPERIMENT_SUMMARY_NOT_FOUND` and `EXPERIMENT_NOT_FOUND` produce distinct messages. `BENCHMARK_NOT_FOUND` produces a not-yet-available message rather than a generic not-found error.

**`request_id` display:**
The SPEC-008 FR-14 `request_id` is displayed with every API error to support operator correlation with API observability data. It is propagated unchanged from the error body.

**Content safety:**
The Dashboard never renders stack traces, connection strings, PostgreSQL error codes, or internal component names. The `message` field from SPEC-008 FR-14 is safe to display per that specification's own security constraints.

**Acceptance Criteria:**
- Every non-2xx API response produces a visible, user-readable error message
- `EXPERIMENT_SUMMARY_NOT_FOUND` and `EXPERIMENT_NOT_FOUND` produce distinct user-visible messages
- `BENCHMARK_NOT_FOUND` produces a not-yet-available message, not a generic not-found error
- HTTP 500 responses do not expose stack traces, connection strings, or internal component names
- The `request_id` from SPEC-008 FR-14 is displayed for all API errors; it is not regenerated or altered
- Network-level errors produce a connectivity message distinct from HTTP-level errors

---

# Non-Requirements

- The Dashboard does not submit routing jobs, experiment manifests, or benchmark manifests
- The Dashboard does not trigger trial evidence collection (CLI responsibility per ADR-012 Decision 5)
- The Dashboard does not compute or recompute quality statistics, runtime statistics, or rankings; all aggregate data is consumed from the SPEC-008 FR-23 experiment summary artifact produced at experiment completion
- The Dashboard does not provide real-time chart rendering (time-series, histograms, scatter plots) at MVP scope
- The Dashboard does not surface OpenTelemetry trace data, Prometheus metrics, or Grafana dashboard content
- The Dashboard does not support Generated Mode workload configuration (SPEC-008 OQ-7 deferred)
- The Dashboard does not support pagination for large trial sets at MVP scope
- The Dashboard does not support multi-tenant authentication or authorization at MVP scope
- The Dashboard does not cancel jobs or modify any persistent state; all interactions are read-only
- The Dashboard does not generate, transform, or re-process HTML evidence reports
- The Dashboard does not access the report volume directly
- The Dashboard does not provide a trial detail view beyond what is accessible via job detail (SPEC-021 FR-6) and evidence report (FR-7)
- The Dashboard does not provide a Generated Mode experiment configuration view (SPEC-008 OQ-7 deferred)
- The Dashboard does not provide a benchmark list view at MVP scope (no list endpoint for benchmarks exists in SPEC-008)
- The Dashboard does not display the raw `experiment_seed` or per-trial `trial_seed` values; seeds are reproducibility inputs, not display data

---

# Assumptions

1. The Daedalus API is reachable by the browser at a known base URL before the Dashboard is used. The default is `http://localhost:5000`, consistent with the Docker Compose port mapping (architecture.md).
2. No authentication or authorization is required at MVP scope, consistent with SPEC-008 Security Considerations. The Dashboard mirrors the API's MVP authentication posture.
3. The API emits CORS headers permitting requests from the Dashboard's browser origin, as required by SPEC-021 Architectural Impact. The specific CORS configuration depends on OQ-2 (serving mechanism).
4. The experiment summary artifact (SPEC-008 FR-23, SPEC-020 FR-14 Artifact 3) contains all fields required for FR-5 and FR-6 rendering: per-(problem, solver) QualityStatsAggregates with quality and runtime statistics, outcome distributions, cross-solver comparison, and reproducibility annotation. The payload structure is authoritative in SPEC-020 FR-12, FR-13, FR-14.
5. The benchmark summary artifact (SPEC-008 FR-24, SPEC-020 FR-14 Artifact 4) contains per-experiment summary fields and cross-experiment comparison data sufficient for FR-8 rendering.
6. The `GET /v1/experiments/{experiment_id}/trials` response (SPEC-008 FR-22) provides all trial fields required for the Trial Matrix (FR-4) and Execution Metadata View (FR-9) without additional per-trial API calls.
7. The Docker Compose environment is the expected deployment context. The Dashboard is accessible at a browser-reachable address within that environment.
8. The Dashboard may be implemented as part of the same application as SPEC-021 (Web UI) or as a separate browser application. This is an implementation planning decision resolved by OQ-1. The functional requirements of this specification apply regardless of the implementation relationship.

---

# Constraints

1. The Dashboard must not access any DAEDALUS infrastructure component other than the Daedalus API. All system interaction is through SPEC-008 HTTP endpoints.
2. The Dashboard must not issue any HTTP method other than GET against the API. All Dashboard interactions are read-only.
3. The Dashboard must not compute quality statistics, rankings, or outcome distributions independently. All aggregate data is consumed from the SPEC-008 FR-23 experiment summary artifact. Independent recomputation risks divergence from the authoritative summary and is prohibited.
4. The Dashboard must not modify, transform, or augment HTML evidence report content before rendering. The report is the Report Generator's artifact (SPEC-009); the Dashboard is a transparent renderer.
5. The Dashboard must not expose internal system identifiers prohibited by SPEC-008 FR-7: `execution_seed`, `decision_id`, and solver run identifiers. These fields do not appear in any SPEC-008 response and therefore cannot appear in Dashboard displays.
6. The Dashboard must not fabricate, regenerate, or alter `request_id` values from SPEC-008 FR-14 error responses; these are propagated unchanged per FR-12.
7. The Dashboard technology stack and serving mechanism are governed by OQ-1 and OQ-2. No implementation technology is prescribed by this specification.

---

# Inputs

| Input | Source | Format | Notes |
|---|---|---|---|
| Experiment ID for lookup | User text entry or list navigation | UUID string | UUID format check applied client-side |
| Benchmark ID for summary lookup | User text entry | String | No format constraint beyond non-empty |
| Experiment status response | `GET /v1/experiments/{experiment_id}` (SPEC-008 FR-20) | JSON | Rendered in FR-3 |
| Trial results response | `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22) | JSON | Rendered in FR-4, FR-9 |
| Experiment summary response | `GET /v1/experiments/{experiment_id}/summary` (SPEC-008 FR-23) | JSON | Rendered in FR-5, FR-6 |
| Benchmark summary response | `GET /v1/benchmarks/{benchmark_id}/summary` (SPEC-008 FR-24) | JSON | Rendered in FR-8 |
| Job status response | `GET /v1/jobs/{job_id}` (SPEC-008 FR-8) | JSON | For job detail via FR-10 |
| Report metadata response | `GET /v1/jobs/{job_id}/report` (SPEC-008 FR-12) | JSON | For FR-7 report navigation |
| Report HTML content | `GET /v1/reports/{report_id}` (SPEC-008 FR-13) | `text/html` | Rendered verbatim in FR-7 |

---

# Outputs

| Output | Consumer | Format | Notes |
|---|---|---|---|
| Rendered experiment overview | Browser user | Browser UI | SPEC-008 FR-20 fields; FR-3 |
| Rendered trial matrix | Browser user | Browser UI | SPEC-008 FR-22 fields; FR-4 |
| Rendered solver comparison | Browser user | Browser UI | SPEC-008 FR-23 summary artifact; FR-5 |
| Rendered experiment summary detail | Browser user | Browser UI | SPEC-008 FR-23 summary artifact; FR-6 |
| Rendered benchmark summary | Browser user | Browser UI | SPEC-008 FR-24 summary artifact; FR-8 |
| Rendered execution metadata table | Browser user | Browser UI | SPEC-008 FR-22 fields; FR-9 |
| Rendered evidence report | Browser user | HTML in browser | SPEC-008 FR-13 content; unmodified; FR-7 |
| Downloaded experiment summary JSON | Browser user | JSON file | SPEC-008 FR-23 payload; unmodified |
| Downloaded benchmark summary JSON | Browser user | JSON file | SPEC-008 FR-24 payload; unmodified |
| Downloaded evidence report HTML | Browser user | HTML file | SPEC-008 FR-13 content; unmodified |

---

# Failure Modes

### API Unreachable

**Condition:** Browser cannot reach the Daedalus API at the configured base URL (TCP failure, DNS failure, API container not running).
**Behavior:** Network error is surfaced per FR-12. The Dashboard displays a connectivity error message and the attempted URL. No API state is modified.
**Recovery:** The user verifies Docker Compose is running and the API is healthy.

---

### Experiment Not Found (HTTP 404, `EXPERIMENT_NOT_FOUND`)

**Condition:** `GET /v1/experiments/{experiment_id}` returns HTTP 404.
**Behavior:** FR-3 displays a not-found message. The FR-2 entry form remains accessible.
**Recovery:** The user verifies the `experiment_id` is correct.

---

### Experiment Summary Not Available (HTTP 404, `EXPERIMENT_SUMMARY_NOT_FOUND`)

**Condition:** `GET /v1/experiments/{experiment_id}/summary` returns HTTP 404 because the experiment has not yet completed.
**Behavior:** FR-5 and FR-6 display a not-yet-available message with a link to FR-3. No error state is displayed; this is the expected condition for in-progress experiments.
**Recovery:** The user waits for the experiment to complete and auto-refreshes the overview.

---

### Benchmark Not Found (HTTP 404, `BENCHMARK_NOT_FOUND`)

**Condition:** No member experiment under the `benchmark_id` has reached `Completed` status.
**Behavior:** FR-8 displays a not-yet-available message.
**Recovery:** The user waits for a member experiment to complete.

---

### Trial Evidence Not Collected

**Condition:** Trials are in `Completed` trial_status but `evidence_status = Missing` or `evidence_status = Error`.
**Behavior:** The Trial Matrix (FR-4) displays the `evidence_status` for affected cells. `hindsight_quality` and `execution_duration_ms` are null, displayed as "—". No best-quality highlight is applied to cells with null `hindsight_quality`. The Dashboard cannot recover from this condition.
**Recovery:** Evidence collection failure requires investigation via CLI or API. No Dashboard action is available.

---

### Report Not Available (HTTP 404 from `GET /v1/jobs/{job_id}/report`)

**Condition:** Report metadata is not available for a trial's job (report generation failed, job in `Failed` state, or job still executing).
**Behavior:** FR-7 displays a not-available message. The Trial Matrix and other views remain accessible.
**Recovery:** For `Failed` jobs, no evidence report will be produced. For report generation failures, operator investigation is required.

---

### Auto-Refresh API Error

**Condition:** An auto-refresh request (`GET /v1/experiments/{experiment_id}` or `GET /v1/experiments/{experiment_id}/trials`) returns a non-200 response or a network error.
**Behavior:** Per FR-12. The last successfully received data remains displayed. The error is surfaced per FR-12.
**Recovery:** If transient, the next successful refresh restores current state. If persistent, the user must reload.

---

# Architectural Impact

| Component | Impact |
|---|---|
| **API** (SPEC-008) | CORS header configuration required. Already required by SPEC-021 Architectural Impact; no additional API endpoint or contract change required by this specification. No new endpoints are introduced. |
| **Dashboard** | New browser component. Consumes SPEC-008 GET endpoints. No existing component behavior is modified. |
| **Docker Compose topology** | May require a new service entry when implemented as a separate application (OQ-1 Option 2). No change if implemented as part of the SPEC-021 Web UI (OQ-1 Option 1). Depends on OQ-1 and OQ-2. |
| **Worker** | None. The Dashboard has no direct interaction with the Worker. |
| **Scheduler** | None. The Dashboard has no direct interaction with the Scheduler. |
| **CLI** | None. The Dashboard does not replace or modify CLI responsibilities. |
| **Observability** | Indirect. API GET calls from the Dashboard browser produce spans owned by SPEC-008 FR-17 (`api.status_poll`, `api.report_serve`, etc.). No new spans are introduced. Dashboard-issued calls are indistinguishable from CLI-issued GET calls at the Prometheus metric level. |
| **Security** | The Dashboard introduces a browser-accessible surface. No authentication at MVP scope. Path traversal for report IDs is prevented by the API's UUID-lookup mechanism (SPEC-008 FR-13). |
| **System Context diagram** | An addition to architecture.md is required when the Dashboard is deployed separately from the SPEC-021 Web UI. If implemented as part of SPEC-021, no separate System Context node is added. Resolution depends on OQ-1. |

---

# Testability

1. **Unit: Trial matrix organization** — Given a trials response with 3 `problem_config_index` values, 2 `repetition_index` values, and 2 distinct `backend_id` values (12 trials), the Trial Matrix renders 6 rows and 2 columns, with each cell corresponding to the correct `(problem_config_index, repetition_index, backend_id)` intersection.

2. **Unit: Best-quality highlight** — Given two trials in the same row with `hindsight_quality = 12.5` and `hindsight_quality = 10.2`, both with `quality_comparison_eligible = true`, the cell with `hindsight_quality = 10.2` is visually indicated as best; the cell with `12.5` is not.

3. **Unit: Best-quality highlight — eligibility scoping** — A `hindsight_quality` value in one row does not contribute to the best-quality highlight in a different row. The highlight is scoped per `(problem_config_index, repetition_index)`.

4. **Unit: Best-quality highlight — ineligible cells excluded** — A cell with `quality_comparison_eligible = false` is not included in the row-level highlight even if it has the lowest numeric `hindsight_quality` in the row.

5. **Unit: Trial matrix — null job_id** — A trial with `job_id = null` renders a placeholder cell without a job link or report link.

6. **Unit: Trial matrix — SchedulerRejected** — A trial with `trial_status = SchedulerRejected` renders `scheduler_rejection_reason` in its cell.

7. **Unit: No per-trial API calls** — The Trial Matrix renders entirely from a single `GET /v1/experiments/{experiment_id}/trials` response. No `GET /v1/jobs/{job_id}` calls are issued while rendering the matrix.

8. **Integration: Experiment overview navigation links** — For a `Completed` experiment, the overview displays links to Trial Matrix, Solver Comparison, Experiment Summary Detail, and Benchmark Summary views. For a `Running` experiment, links to Solver Comparison and Experiment Summary Detail are absent.

9. **Integration: Solver comparison — summary not yet available** — For a `Running` experiment, `GET /v1/experiments/{experiment_id}/summary` returns HTTP 404 with `EXPERIMENT_SUMMARY_NOT_FOUND`; the Solver Comparison view displays a not-yet-available message with a link to FR-3; no error state is displayed.

10. **Integration: Solver comparison table population** — For a `Completed` experiment with two solver backends, the Solver Comparison view renders a table with two rows, one per backend, each populated with quality and runtime statistics from the summary payload.

11. **Integration: Solver comparison — no quality comparison eligible** — When no trial has `quality_comparison_eligible = true`, the cross-solver ranking note states that quality comparison is not applicable; the outcome distribution and runtime statistics tables are still displayed.

12. **Integration: Reproducibility annotation** — The Solver Comparison view displays the experiment's `reproducibility_class` with the corresponding explanatory note for each class value.

13. **Integration: Targeted execution note** — The Solver Comparison view displays a note that all trials used explicitly targeted backend execution (`selection_mode = explicitly_targeted`).

14. **Integration: Report navigation from Trial Matrix** — A cell with `trial_status = Completed` and `job_id` non-null provides a report link; activating the link issues `GET /v1/jobs/{job_id}/report` and, on HTTP 200, fetches and renders the report HTML from the returned `report_url`.

15. **Integration: Report rendered without modification** — The HTML content from `GET /v1/reports/{report_id}` is rendered byte-identical to the API response body. No Dashboard-injected content appears in the rendered output.

16. **Integration: Report not available message** — When `GET /v1/jobs/{job_id}/report` returns HTTP 404, the FR-7 view displays a not-available message. The Trial Matrix view is still accessible.

17. **Integration: Benchmark summary — research context displayed** — When the summary payload includes benchmark manifest fields (`research_question`, `hypothesis`, etc.), the Research Context section is displayed. When these fields are absent from the payload, the section is omitted.

18. **Integration: Benchmark summary — member experiment links** — Each member experiment entry in the benchmark summary table provides a link to FR-3 for that experiment.

19. **Integration: Benchmark not found** — Entering a `benchmark_id` for which no member experiment has completed returns `BENCHMARK_NOT_FOUND` (HTTP 404); the Dashboard displays a not-yet-available message, not a generic not-found error.

20. **Integration: Experiment summary download** — For a `Completed` experiment, activating the download option in FR-5 triggers a file save of the SPEC-008 FR-23 JSON payload without transformation.

21. **Integration: Benchmark summary download** — Activating the download option in FR-8 triggers a file save of the SPEC-008 FR-24 JSON payload without transformation.

22. **Integration: Breadcrumb from evidence report** — From the FR-7 Evidence Report view, the breadcrumb displays the path to the Trial Matrix and Experiment Overview; activating a breadcrumb step navigates to that view.

23. **Integration: Auto-refresh stops on completion** — For an experiment in `Running` status with auto-refresh active in FR-3 and FR-4, when the experiment transitions to `Completed`, no further polling requests are issued to either endpoint.

24. **Unit: Error display — EXPERIMENT_SUMMARY_NOT_FOUND vs EXPERIMENT_NOT_FOUND** — An HTTP 404 with `error_code = EXPERIMENT_SUMMARY_NOT_FOUND` produces a "summary not yet available" message; an HTTP 404 with `error_code = EXPERIMENT_NOT_FOUND` produces a "not found" message. The two messages are textually distinct.

25. **Unit: Error display — request_id propagated** — An HTTP 500 response from the API includes a `request_id` in the SPEC-008 FR-14 body; the Dashboard displays this `request_id` unchanged in the rendered error message.

26. **Unit: Error display — no internal details** — An HTTP 500 response does not result in stack traces, connection strings, or internal component names appearing in any rendered Dashboard view.

27. **Integration: Execution metadata sortable** — Activating the sort control for `execution_duration_ms` in FR-9 reorders the table rows without issuing a new API request.

28. **Unit: UUID format check on entry** — An experiment_id entry that fails the UUID format check produces a client-side error without issuing a `GET /v1/experiments/{experiment_id}` API request.

29. **Integration: Per-(problem, solver) table in FR-6** — For a `Completed` experiment with 2 problem configs and 2 backends, the Experiment Summary Detail view renders 4 rows in the per-(problem, solver) table plus 1 experiment-level aggregate row.

30. **Unit: Constraint — no non-GET requests** — Inspect all network requests issued by any Dashboard view. Verify no POST, PUT, PATCH, or DELETE methods are issued to the Daedalus API.

---

# Observability Requirements

Operational questions the Dashboard and its API interactions must support:

1. Is the API accessible from the browser? Answered by connectivity error handling (FR-12) and `daedalus_api_status_poll_total` in SPEC-008 FR-17.
2. Are experiment summaries being produced? Answered by `daedalus_api_experiments_completed_total` in SPEC-008 FR-17.
3. Are trial evidence records present for completed trials? Answered by `daedalus_api_trials_evidence_collected_total` in SPEC-008 FR-17.
4. Are evidence reports accessible via the API? Answered by `api.report_serve` spans from SPEC-008 FR-17 and `daedalus_report_generation_total` from SPEC-009 Observability Requirements.

The Dashboard does not introduce new Prometheus metrics or OpenTelemetry spans. Observability for the system is owned by SPEC-008 FR-17, SPEC-005 FR-19, and ADR-006.

---

# Security Considerations

**No authentication at MVP scope:**
Consistent with SPEC-008 Security Considerations, no authentication layer exists at MVP scope. The Dashboard assumes a trusted local development environment. Authentication and authorization must be implemented before any exposure to untrusted networks or production deployment.

**Input handling:**
User input (experiment_id, benchmark_id) is transmitted to the API as URL path parameters only. The API is the trust boundary. The Dashboard does not construct SQL queries, filesystem paths, or queue payloads from user input. The client-side UUID format check on experiment_id entry prevents malformed strings from being sent to the API but does not substitute for API-level validation.

**Report rendering:**
HTML evidence reports are produced by the Report Generator (SPEC-009), served by the API (SPEC-008 FR-13), and rendered by the browser. The report content is authored by the system, not by untrusted user input. The Dashboard must not inject user-supplied content into rendered report HTML. When report content is rendered in an embedded frame within the Dashboard page, the frame must apply appropriate browser sandboxing to restrict script execution, form submission, and top-level navigation originating from report content. The specific sandboxing policy is an implementation planning decision resolved alongside OQ-1 and OQ-2; it must be consistent with the sandboxing approach defined by SPEC-021 Security Considerations.

**CORS:**
The API must be configured to emit CORS headers permitting requests from the Dashboard's origin. Already required by SPEC-021 Architectural Impact. If the Dashboard is co-deployed with the SPEC-021 Web UI (OQ-1 Option 1), no additional CORS origin is required.

**Internal identifier isolation:**
The Dashboard must not display `execution_seed`, `decision_id`, or solver run identifiers. These fields do not appear in any SPEC-008 response.

**Path traversal:**
Report retrieval (FR-7) uses the `report_url` from SPEC-008 FR-12, which resolves to a server-side UUID-based lookup (SPEC-008 FR-13). The Dashboard does not construct file paths from user input.

---

# Performance Considerations

**Trial results response size:**
`GET /v1/experiments/{experiment_id}/trials` returns all trials in a single response. At MVP trial counts (small experiments with a modest solver set and repetition count), rendering the full matrix in-browser without pagination is safe. Pagination is deferred per Non-Requirements.

**Experiment summary response size:**
The summary payload includes QualityStatsAggregate artifacts per (problem, solver) scope, cross-solver comparison, and experiment-level aggregates. At MVP experiment sizes this payload is expected to be small enough for in-browser rendering without streaming.

**Auto-refresh polling:**
FR-3 and FR-4 auto-refresh issues two API calls per cycle: `GET /v1/experiments/{experiment_id}` and `GET /v1/experiments/{experiment_id}/trials`. At single-developer MVP scale this is not a concern. Polling intervals must be defined during implementation to balance responsiveness with API load; 10-second intervals are consistent with SPEC-021 FR-8 experiment polling.

**Solver comparison view:**
The Solver Comparison View (FR-5) derives its content from the already-computed experiment summary artifact. No server-side computation is triggered by the Dashboard. Rendering performance is bounded by browser-side table layout for the number of (problem, solver) pairings.

**No per-trial API calls:**
The Trial Matrix (FR-4) and Execution Metadata View (FR-9) are both derived from a single `GET /v1/experiments/{experiment_id}/trials` response. This prevents the N×M API call pattern that would arise from fetching each trial's job detail individually.

---

# Documentation Updates Required

- **docs/architecture.md**: When the Dashboard is deployed as a separate application (OQ-1 Option 2), the System Context diagram should include the Dashboard as a new client node parallel to the CLI and Web UI (SPEC-021). If the Dashboard is implemented as part of the SPEC-021 Web UI application (OQ-1 Option 1), no separate architecture node is added. Resolution depends on OQ-1.

- **SPEC-020 (Benchmark and Experiment Harness)**: The Scope and Responsibility Boundary table in SPEC-020 already references "Dashboard visualization of benchmark and experiment summaries | SPEC-022 (pending specification)." When SPEC-022 is accepted, SPEC-020 should be updated to replace "(pending specification)" with the accepted status reference.

- **SPEC-021 (Web UI)**: SPEC-021 FR-8 provides basic experiment observation (raw status view, raw trial table, raw summary JSON). SPEC-022 provides the structured visualization and comparison layer. When SPEC-022 is accepted, SPEC-021 Documentation Updates Required should note that structured experiment visualization is owned by SPEC-022 and that SPEC-021 FR-8 represents the basic observation baseline. If OQ-1 resolves to a single unified application, the two specifications' FR-8 and SPEC-022 views should be reconciled in a joint implementation planning session.

---

# Open Questions

### OQ-1: Relationship to SPEC-021 Web UI

**Question:** Is the Experiment Dashboard implemented as a set of additional views within the SPEC-021 Web UI application, or as a separately deployed browser application?

**Why it matters:** The answer determines deployment unit boundaries (Docker Compose service, build pipeline), CORS configuration (same origin vs. separate), and whether a separate System Context node is added to architecture.md. It also determines whether the FR-2 experiment browser reuses FR-5 of SPEC-021 or stands alone.

**Options under consideration:**
1. **Part of SPEC-021:** Dashboard views are added to the Web UI application. Single deployment unit, same origin, shared technology stack. No additional CORS configuration.
2. **Separate application:** Dashboard has its own deployment unit. Different origin from the Web UI; API CORS headers must permit both origins.

**Owner:** Project Owner decision. The ADR governing SPEC-021 OQ-2 (technology stack) should address this relationship simultaneously.

**Blocking:** Blocking for implementation planning. Not blocking for Draft status.

---

### OQ-2: Technology Stack and Serving Mechanism

**Question:** What browser-side technology and serving mechanism are used for the Dashboard?

**Why it matters:** If the Dashboard is part of the SPEC-021 Web UI (OQ-1 Option 1), this question is answered by SPEC-021 OQ-2. If implemented separately, an independent ADR is required.

**Owner:** Resolved by the ADR governing SPEC-021 OQ-2 if OQ-1 selects Option 1; otherwise requires a separate ADR.

**Blocking:** Not blocking for Draft status. Blocking for implementation.

---

### OQ-3: Experiment List Endpoint Dependency

**Question:** The Experiment Browser (FR-2) benefits from a `GET /v1/experiments` list endpoint. This endpoint is referenced in SPEC-021 OQ-1 and SPEC-016 OQ-1 but is not currently defined in SPEC-008. Should this endpoint be defined as a SPEC-008 amendment before Dashboard implementation begins, or does the Dashboard use the entry-form fallback?

**Why it matters:** Without a list endpoint, the Dashboard entry experience is a text entry form. A list endpoint would make experiments discoverable without requiring the user to know the `experiment_id` in advance.

**Options:**
1. Define `GET /v1/experiments` as a SPEC-008 amendment. This resolves SPEC-021 OQ-1, SPEC-016 OQ-1, and SPEC-022 OQ-3 simultaneously.
2. Implement the entry-form fallback at MVP scope; defer the list endpoint.

**Owner:** Project Owner decision on SPEC-008 amendment scope. Resolution aligns with SPEC-021 OQ-1.

**Blocking:** Blocking for full FR-2 implementation. The entry-form fallback in FR-2 is not blocked.

---

### OQ-4: API Base URL Discovery

**Question:** How does the Dashboard browser application discover the Daedalus API base URL at runtime?

**Why it matters:** Same as SPEC-021 OQ-4. If the Dashboard is implemented as part of the SPEC-021 Web UI (OQ-1 Option 1), this question is answered by SPEC-021 OQ-4. If implemented separately, the same options apply: build-time environment variable, runtime configuration file, or same-origin derivation.

**Owner:** Resolved by SPEC-021 OQ-4 if OQ-1 selects Option 1; otherwise addressed by the same ADR governing OQ-2.

**Blocking:** Not blocking for Draft status. Blocking for implementation.

---

# Acceptance Checklist

- [ ] Problem is clearly defined
- [ ] Domain concept is defined
- [ ] Scope and responsibility boundary is defined
- [ ] FR-1 component responsibilities defined
- [ ] FR-2 experiment entry and browser defined
- [ ] FR-3 experiment overview defined
- [ ] FR-4 trial matrix view defined
- [ ] FR-5 solver comparison view defined
- [ ] FR-6 experiment summary detail defined
- [ ] FR-7 evidence report navigation and rendering defined
- [ ] FR-8 benchmark summary view defined
- [ ] FR-9 execution metadata view defined
- [ ] FR-10 observability metadata display defined
- [ ] FR-11 cross-view navigation defined
- [ ] FR-12 API error display defined
- [ ] Non-requirements documented
- [ ] Assumptions explicit
- [ ] Constraints documented
- [ ] Inputs and outputs defined
- [ ] Failure modes defined
- [ ] Architectural impact documented
- [ ] Testability cases exist for each FR
- [ ] Observability requirements stated
- [ ] Security considerations documented
- [ ] Performance considerations documented
- [ ] Documentation updates identified
- [ ] OQ-1 documented (relationship to SPEC-021 Web UI)
- [ ] OQ-2 documented (technology stack)
- [ ] OQ-3 documented (experiment list endpoint dependency)
- [ ] OQ-4 documented (API base URL discovery)

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-12) are implemented and acceptance criteria pass
- OQ-1 resolved: Dashboard relationship to SPEC-021 Web UI determined (same application or separate deployment)
- OQ-2 resolved: technology stack and serving mechanism defined via ADR before implementation begins
- OQ-3 resolved: either `GET /v1/experiments` is defined in a SPEC-008 amendment and the full experiment browser is implemented, or the entry-form fallback per FR-2 is in place
- OQ-4 resolved: API base URL discovery mechanism defined
- The Dashboard accesses the API solely through SPEC-008 HTTP GET endpoints; no POST, PUT, PATCH, or DELETE requests are issued; no direct database, queue, or Worker access exists
- The Trial Matrix (FR-4) renders without per-trial API calls; all cell data is derived from a single `GET /v1/experiments/{experiment_id}/trials` response
- The Solver Comparison View (FR-5) derives all statistics from the SPEC-008 FR-23 payload without independent recomputation
- The cross-solver ranking note and targeted execution note are displayed in FR-5 on every invocation
- Evidence report HTML is rendered verbatim without modification (FR-7)
- Per-FR-12: `EXPERIMENT_SUMMARY_NOT_FOUND`, `EXPERIMENT_NOT_FOUND`, and `BENCHMARK_NOT_FOUND` produce distinct user-visible messages
- `request_id` from SPEC-008 FR-14 error responses is displayed in all error conditions
- No OpenTelemetry spans, `traceparent` headers, or internal DAEDALUS identifiers (`execution_seed`, `decision_id`) appear in any Dashboard output
- CORS headers on the API permit browser requests from the Dashboard's serving origin
- Engineering review passes
- Architecture review passes
- Specification status is updated to Verified
