# Feature Specification

## Metadata

**Feature ID:** SPEC-021

**Title:** Daedalus Web UI

**Status:** Proposed

**Author:** Darkhorse286

**Created:** 2026-06-25

**Last Updated:** 2026-06-25

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-002, ADR-006, ADR-009, ADR-012, ADR-013

**Related Specs:** SPEC-001, SPEC-003, SPEC-005, SPEC-006, SPEC-007, SPEC-008, SPEC-009, SPEC-012, SPEC-016, SPEC-020

---

# Problem Statement

Project DAEDALUS defines an evidence-driven optimization runtime that selects solver backends, executes routing jobs, and produces HTML evidence reports explaining each scheduling decision. Currently, interacting with the system requires either authoring raw HTTP requests or using the Daedalus CLI (SPEC-016). Absent a browser-based interface:

- Stakeholders without CLI access cannot observe system state — pending jobs, completed executions, solver outcomes — without developer assistance.
- Evidence reports require CLI invocation (`daedalus report show`) or direct HTTP interaction to retrieve. There is no in-browser experience for viewing the project's core output artifact.
- Job submission requires knowledge of the SPEC-001 JSON request format and the SPEC-008 endpoint contract.
- Demonstrating the project thesis ("most routing problems should never touch a quantum computer") to a non-technical audience requires a developer workstation with Docker Compose and the CLI installed.
- The scheduler configuration model (SPEC-003 FR-6 objective modes) has no human-readable presentation; it is only visible through raw API responses.

A Web UI provides a browser-based interaction layer that exposes the capabilities already defined by accepted specifications, without introducing new backend behavior, modifying the API contract, or bypassing the Control Plane.

---

# Business Value

- Enables demonstration of the full optimization runtime from any browser without CLI installation or knowledge of the HTTP API contract
- Makes HTML evidence reports immediately viewable in-browser, turning the project's core output into a navigable artifact rather than a file to download
- Provides a visual job dashboard showing system state: pending jobs, solver outcomes, execution status, and report availability
- Signals full-stack engineering and system integration capability as a portfolio artifact
- Makes the project thesis demonstrable to a non-technical audience: a stakeholder can open a browser, submit a routing problem, watch the scheduler select a solver, and read the evidence report explaining the decision

---

# Employer Signaling

- Frontend Development
- Full-Stack Engineering
- System Integration
- API Contract Consumption
- Developer Experience

---

# Domain Concept

The Web UI is a browser-based client for the Daedalus API. It is a consumer of the SPEC-008 API contract, not a backend component. It observes what the system has produced and provides form-based access to a subset of the operations defined by SPEC-008.

The Web UI does not submit experiment manifests, orchestrate trial loops, collect evidence, or select solver backends. Those responsibilities are owned by the CLI (SPEC-016), the API (SPEC-008), and the Scheduler respectively. The Web UI reads what those components have produced and provides a browser-accessible submission path for individual routing jobs.

From an architectural perspective, the Web UI adds a new browser client to the System Context. It accesses the API through the same HTTP interface available to the CLI and any other caller:

```
Browser [User]
  → Web UI
    → Daedalus API (SPEC-008, http://localhost:5000 by default)
      → (existing system: Worker, Scheduler, PostgreSQL, RabbitMQ, Report Generator)
```

No new API endpoints are defined by this specification. No existing API endpoints are modified. The Web UI is purely a frontend consumer of the established SPEC-008 contract.

---

# Scope and Responsibility Boundary

| Responsibility | Owner |
|---|---|
| HTTP API contract for all endpoints consumed by the Web UI | SPEC-008 |
| Routing problem domain model and validation | SPEC-001 |
| Scheduler configuration semantics and objective modes | SPEC-003 |
| Job execution, solver selection, quality evaluation, evidence production | SPEC-005, Worker |
| Report generation and report content | SPEC-009 |
| Experiment orchestration loop: trial submission, polling, evidence collection triggers | SPEC-016 (CLI) |
| Benchmark manifest submission | SPEC-016 (CLI) |
| Experiment and benchmark durable state | SPEC-008, SPEC-012 |
| Consuming SPEC-008 API endpoints via browser for job submission, status monitoring, and report access | **SPEC-021 (this spec)** |
| Displaying experiment and trial observation data (read-only) | **SPEC-021 (this spec)** |
| Presenting scheduler configuration selection during job submission | **SPEC-021 (this spec)** |
| Presenting backend targeting selection during job submission | **SPEC-021 (this spec)** |

The Web UI does not:

- Submit experiment manifests or drive the experiment trial orchestration loop (CLI responsibility per ADR-012 Decision 1)
- Trigger trial evidence collection (CLI responsibility per ADR-012 Decision 5)
- Submit benchmark manifests (CLI responsibility per SPEC-016 FR-17)
- Access PostgreSQL, RabbitMQ, the report volume, or any infrastructure component directly
- Perform SPEC-001 domain validation independently of the API (domain validation authority belongs to the API and Core per ADR-009; the Web UI applies structural input checks only)
- Generate, modify, or re-process HTML evidence reports (Report Generator and API responsibilities)
- Select solver backends (Scheduler responsibility)
- Modify experiment or trial state

---

# Requirements

## FR-1: Scope and Component Responsibilities

**Description:**
SPEC-021 defines the Web UI's responsibilities and explicitly allocates adjacent responsibilities it must not perform.

| Component | Responsibilities at This Boundary | Explicitly Not Responsible For |
|---|---|---|
| **Web UI** (SPEC-021) | HTTP request formation to SPEC-008 endpoints; user input collection and client-side structural validation; rendering API responses in browser; job status polling; in-browser report rendering; experiment and trial state observation | SPEC-001 domain validation (API + Core); solver selection; experiment orchestration; evidence collection triggers; report generation; direct infrastructure access |
| **API** (SPEC-008) | All data validation, persistence, job queue publication, experiment state management, and error response production | Rendering UI; driving polling loops; managing user session state |
| **CLI** (SPEC-016) | Experiment orchestration, trial submission loop, evidence collection, benchmark submission | Web browser interaction; browser state management |

**Acceptance Criteria:**
- No Web UI code path bypasses the API to access PostgreSQL, RabbitMQ, the report volume, the Worker, or the Python Solver Adapter directly
- Client-side validation is structural only: required fields present, numeric fields parseable, UUID format checks. SPEC-001 domain validation (coordinate ranges, capacity feasibility, time window ordering, demand constraints) is enforced by the API, not the Web UI
- The Web UI does not emit OpenTelemetry spans or interact with the OpenTelemetry Collector
- The Web UI does not submit experiment manifests, trigger evidence collection, or modify any experiment or trial record state

---

## FR-2: Job Submission Form

**Description:**
The Web UI provides a form for submitting a routing job via `POST /v1/jobs` (SPEC-008 FR-2). The form collects all fields required to compose a SPEC-008 FR-2 request body.

**Routing problem fields** (per SPEC-001, serialized per SPEC-001 FR-11):

| Field | Type | Required | Client-side check |
|---|---|---|---|
| `seed` | Non-negative integer | Yes | Non-empty; parseable as non-negative 64-bit integer |
| `vehicle_count` | Positive integer | Yes | Non-empty; parseable as integer ≥ 1 |
| `capacity_per_vehicle` | Positive integer | Yes | Non-empty; parseable as integer ≥ 1 |
| `average_vehicle_speed_kmh` | Positive decimal | Yes | Non-empty; parseable as a positive number |
| Depot `latitude` | Decimal | Yes | Non-empty; parseable as a number |
| Depot `longitude` | Decimal | Yes | Non-empty; parseable as a number |
| Stops | Repeated rows | Yes | At least one row present; each row has all required fields non-empty and parseable |

**Per-stop fields** (per SPEC-001):

| Field | Type | Required | Client-side check |
|---|---|---|---|
| `id` | Non-negative integer | Yes | Non-empty; parseable as non-negative integer; duplicate-free within form (structural consistency aid to prevent malformed submissions; domain uniqueness is enforced by the API per SPEC-001 FR-4 and ADR-009) |
| `latitude` | Decimal | Yes | Non-empty; parseable as a number |
| `longitude` | Decimal | Yes | Non-empty; parseable as a number |
| `demand` | Non-negative integer | Yes | Non-empty; parseable as non-negative integer |
| `time_window_open` | Non-negative integer | No | When present, parseable as non-negative integer; both open and close must be present or both absent (structural check only; ordering enforced by API) |
| `time_window_close` | Non-negative integer | No | When present, parseable as non-negative integer; presence paired with `time_window_open` |
| `service_duration` | Non-negative integer | No | When present, parseable as non-negative integer |

The form supports adding and removing stop rows dynamically. A minimum of one stop row must be present before submission can be attempted.

**Job configuration fields:**

| Field | Type | Required | Notes |
|---|---|---|---|
| `scheduler_config_id` | Selection or UUID text | No | Populated from the scheduler configuration panel (FR-3); absent causes the API to use the default configuration (SPEC-008 FR-2); when supplied, must be a valid UUID format |
| `backend_id` | Selection or text | No | Optional backend targeting directive (FR-4); absent means standard scheduler path (SPEC-008 FR-2, ADR-013) |

**Submission flow:**
1. User fills the form and submits.
2. The Web UI performs client-side structural checks. Any structural failures are displayed before the API request is issued.
3. The Web UI serializes the form fields into a SPEC-008 FR-2 JSON request body.
4. The Web UI issues `POST /v1/jobs` to the API.
5. On HTTP 202: navigate to the Job Detail View (FR-6) for the returned `job_id`. Display the `job_id` and `problem_id` to the user.
6. On HTTP 400: display the structured validation errors from the SPEC-008 FR-14 `field_errors` array. Each error entry is associated with the corresponding form field where a field mapping exists. All `field_errors` entries are displayed; none are silently dropped.
7. On HTTP 5xx: display a generic error message consistent with FR-9. Internal server details are not surfaced.

**Acceptance Criteria:**
- The form collects all SPEC-001 required fields and submits them as a SPEC-008 FR-2 JSON request body
- Client-side structural checks block submission on parseable-type failures and missing required fields; they do not block submission on SPEC-001 domain constraint violations (coordinate ranges, capacity feasibility, time windows) — those are enforced by the API
- A submission producing HTTP 202 navigates to the Job Detail View (FR-6) for the returned `job_id`; the `job_id` and `problem_id` are displayed
- A submission producing HTTP 400 displays every `field_errors` entry from the SPEC-008 FR-14 error body; no entry is silently dropped
- Stop rows can be added and removed dynamically; the form prevents submission when fewer than one stop row is present
- `scheduler_config_id` and `backend_id` are optional; their absence causes the API to apply defaults without the Web UI producing an error
- HTTP 5xx responses display a generic error message without exposing internal server details, stack traces, or connection strings
- The form does not prevent submission on SPEC-001 domain violations detectable only by the API (such as capacity infeasibility or coordinate out-of-range); those errors are surfaced in the HTTP 400 response after the attempt

---

## FR-3: Scheduler Configuration Selection

**Description:**
The Web UI provides a scheduler configuration selection panel integrated with the job submission form (FR-2). It presents the available configurations from the API and optionally allows creation of new configurations.

**Configuration list:**
When the job submission form is displayed, the Web UI fetches the current scheduler configurations via `GET /v1/scheduler-configs` (SPEC-008 FR-10). The result is rendered as a selectable list. The default configuration — identified as the first entry in the response per SPEC-008 FR-10 — is visually distinguished. The user may select any listed configuration or leave the selection unset (causing the API to use the default implicitly per SPEC-008 FR-2).

**Configuration detail on selection:**
When a configuration is selected, its `objective_mode` and, for parameterized modes, its `mode_parameters` are displayed to the user. The seven objective modes are:

| `objective_mode` | Parameters displayed |
|---|---|
| `CheapestValid` | None |
| `FastestValid` | None |
| `Balanced` | `cost_weight`, `latency_weight`, `quality_weight` |
| `BestQuality` | None |
| `DeadlineAware` | `deadline_seconds` |
| `BudgetCapped` | `budget_limit` |
| `Experimental` | None |

**Configuration creation:**
The Web UI provides a form for creating a new scheduler configuration via `POST /v1/scheduler-configs` (SPEC-008 FR-10). The creation form collects:

| Field | Input type | Required | Notes |
|---|---|---|---|
| `objective_mode` | Selection from the seven defined modes | Yes | Selects one of the seven modes defined in SPEC-003 FR-6 |
| Mode parameters | Conditional inputs revealed by selected mode | Conditional | Fields are shown or hidden based on the selected mode; irrelevant fields are not submitted |

Mode-specific parameter inputs:

| Mode | Parameter inputs |
|---|---|
| `CheapestValid`, `FastestValid`, `BestQuality`, `Experimental` | No parameter inputs required |
| `Balanced` | `cost_weight` (decimal), `latency_weight` (decimal), `quality_weight` (decimal); sum = 1.0 is enforced by the API, not the UI |
| `DeadlineAware` | `deadline_seconds` (positive number) |
| `BudgetCapped` | `budget_limit` (positive number) |

On HTTP 201 from configuration creation: the new configuration is appended to the selection list and may be immediately selected for job submission without reloading the page.

**Acceptance Criteria:**
- The scheduler configuration list is populated from `GET /v1/scheduler-configs` before the submission form accepts user input
- The default configuration is visually identified in the list
- Selecting a configuration passes its `scheduler_config_id` in the `POST /v1/jobs` request body
- Leaving the configuration unselected submits the job without `scheduler_config_id`; the API default is used (SPEC-008 FR-2)
- The configuration creation form reveals only the parameter inputs required for the selected `objective_mode`; parameter fields for other modes are not visible and are not submitted
- A configuration created through the creation form is immediately available for selection without a page reload
- HTTP 400 from `POST /v1/scheduler-configs` displays `field_errors` without surfacing internal server details
- An unknown `scheduler_config_id` submitted with a job (HTTP 400, `error_code = SCHEDULER_CONFIG_NOT_FOUND`) is displayed as a validation error on the submission form

---

## FR-4: Backend Targeting

**Description:**
The job submission form (FR-2) includes an optional `backend_id` field for targeted job execution (SPEC-008 FR-2, ADR-013). When provided, the `backend_id` is forwarded to the Scheduler, which validates eligibility for that specific backend and routes execution to it without policy-based fallback.

**Backend selection:**
The Web UI presents the known MVP solver backends as a selectable list alongside a free-text entry option. The list is static, defined by the architecture and not queried from the API at runtime (backend capability profiles are not persisted in PostgreSQL per SPEC-012 FR-2 scope).

**Known backends at MVP scope** (per architecture.md Solver Backend Inventory):

| Display Name | `backend_id` | Category |
|---|---|---|
| Nearest Neighbor | `nearest-neighbor` | Classical deterministic |
| Greedy Insertion | `greedy-insertion` | Classical deterministic |
| QUBO Simulated Annealing | `qubo-simulated-annealing` | Quantum-inspired stochastic |
| QAOA — Local Simulator | `qaoa-qiskit` | Quantum-inspired stochastic |

The `qaoa-hardware` backend is not included in the selection list at MVP scope. Its execution is deferred per ADR-007. Free-text entry of `qaoa-hardware` as a `backend_id` is permitted by the API (SPEC-008 FR-3 accepts any non-empty string) and is not blocked by the UI.

**Submission behavior (ADR-013):**
- When a `backend_id` is selected or entered, it is included in the `POST /v1/jobs` request body as the targeting directive (targeted path per SPEC-008 FR-2 and ADR-013).
- When no `backend_id` is selected and the free-text field is empty, the `backend_id` field is absent from the request body (standard path per SPEC-008 FR-2).
- The Web UI enforces that if the `backend_id` field is actively populated, its value is non-empty before submission is attempted. An empty-string `backend_id` must not be submitted (SPEC-008 FR-3 rejects empty strings with HTTP 400).
- The Web UI does not perform Scheduler eligibility checks. The API accepts any non-empty `backend_id` string; eligibility is determined by the Scheduler at execution time (ADR-013, SPEC-008 FR-3).

**Acceptance Criteria:**
- When a `backend_id` is selected from the list or entered in the free-text field, it is included in the `POST /v1/jobs` request body
- When no `backend_id` is selected and the free-text field is empty, the `POST /v1/jobs` request body omits the `backend_id` field entirely
- An empty-string `backend_id` is not submitted; the Web UI enforces non-empty if the field is populated before the API request is issued
- The Web UI does not validate `backend_id` against a capability profile registry or perform any eligibility check; the API and Scheduler are the authorities
- A `NoEligibleSolver` result (job transitions to `Failed` with the Scheduler rejecting the targeted backend) is surfaced in the Job Detail View (FR-6) without any special-casing at the submission form; the submission itself returned HTTP 202 successfully
- The `qaoa-hardware` backend is not presented in the selection list; it may be entered via free text without UI-level rejection

---

## FR-5: Job Dashboard

**Description:**
The Web UI provides a dashboard view listing recent jobs and their current lifecycle status, giving the user a summary of system activity without requiring knowledge of specific `job_id` values.

**Data source dependency:**
The job dashboard requires a `GET /v1/jobs` list endpoint in the API. This endpoint is referenced in SPEC-016 OQ-1 but is not currently defined in SPEC-008. The dashboard view is contingent on this endpoint. See OQ-1.

**Dashboard columns** (pending OQ-1 resolution):

| Column | Source field | Notes |
|---|---|---|
| Job ID | `job_id` | Abbreviated for display; full UUID accessible from the Job Detail View |
| Status | `status` | Visually differentiated by lifecycle state: Pending, Processing, Completed, Failed |
| Solver Outcome | `solver_outcome` | Displayed when `status = Completed`; blank for other states |
| Created At | `created_at` | ISO 8601 UTC timestamp |
| Report | `report_available` | Visible link to the evidence report view (FR-7) when `true`; otherwise absent |

**Auto-refresh:**
The dashboard auto-refreshes periodically by re-querying the job list endpoint. Auto-refresh is active while the dashboard view is displayed and pauses or stops when the user navigates away.

**Job selection:**
Each job row is interactive. Selecting a job navigates to the Job Detail View (FR-6) for that job.

**Fallback when OQ-1 is unresolved:**
If the `GET /v1/jobs` list endpoint is not available at implementation time, the dashboard renders a job lookup form instead: the user enters a `job_id` directly and navigates to the Job Detail View. The submission form (FR-2) remains accessible. The fallback does not require any new API endpoint.

**Acceptance Criteria:**
- When the `GET /v1/jobs` list endpoint exists (OQ-1 resolved), the dashboard displays recent jobs with the columns defined above
- Jobs are ordered most-recently-created first
- `status` is rendered with visual differentiation between Pending, Processing, Completed, and Failed
- `solver_outcome` is present for `Completed` jobs and absent for non-terminal or `Failed` jobs
- `report_available = true` renders a link that navigates to FR-7 for that job
- Auto-refresh does not disrupt in-progress interactions on the page (such as text entry in the submission form)
- When the `GET /v1/jobs` list endpoint is not available (OQ-1 unresolved), the dashboard falls back to a job-ID lookup form with no degradation of the submission form (FR-2)

---

## FR-6: Job Detail View and Cancellation

**Description:**
The Web UI provides a detail view for a single job, identified by `job_id`. This view is the primary destination after successful job submission (FR-2) and from the job dashboard (FR-5). It consumes `GET /v1/jobs/{job_id}` (SPEC-008 FR-8).

**Fields displayed:**

| Field | Condition | Source |
|---|---|---|
| `job_id` | Always | SPEC-008 FR-8; full UUID; copyable |
| `problem_id` | Always | SPEC-008 FR-8; full UUID; copyable |
| `scheduler_config_id` | Always | SPEC-008 FR-8; full UUID; copyable |
| `status` | Always | SPEC-008 FR-8; visually differentiated by state |
| `created_at` | Always | SPEC-008 FR-8; ISO 8601 UTC |
| `updated_at` | Always | SPEC-008 FR-8; ISO 8601 UTC |
| `completed_at` | When `status = Completed` | SPEC-008 FR-8 |
| `failed_at` | When `status = Failed` | SPEC-008 FR-8 |
| `solver_outcome` | When `status = Completed` | SPEC-008 FR-8; displayed prominently |
| `failure_reason` | When `status = Failed` | SPEC-008 FR-8 |
| `report_available` | Always | SPEC-008 FR-8; link to FR-7 when `true` |

**Solver outcome guidance:**
Per SPEC-008 FR-9, `status = Completed` does not indicate solver success. When `status = Completed` and `solver_outcome` is `Infeasible`, `Timeout`, `Cancelled`, `Failed`, or `ContractViolation`, the view presents an explanatory note distinguishing the solver-level outcome from a Worker lifecycle failure. This is consistent with the SPEC-008 FR-9 caller guidance. A `solver_outcome = Failed` on a `Completed` job is distinct from `status = Failed`; the view must not conflate them.

**Status polling:**
For non-terminal jobs (`Pending`, `Processing`), the view auto-polls `GET /v1/jobs/{job_id}` periodically. Polling continues until the job transitions to `Completed` or `Failed`, then stops. The display updates without a full page reload.

**Report access:**
When `report_available = true`, the view displays a prominent link to the Evidence Report View (FR-7) for this job.

**Job cancellation:**
The view provides a cancellation action for non-terminal jobs. When activated, the Web UI issues `POST /v1/jobs/{job_id}/cancel` (SPEC-008 FR-11).

| API response | Web UI behavior |
|---|---|
| HTTP 202 | Display the cancellation acknowledgement message; note that cancellation is not guaranteed (consistent with SPEC-008 FR-11 semantics) |
| HTTP 409 (`JOB_ALREADY_TERMINAL`) | Display a message indicating the job reached a terminal state before cancellation was recorded |
| HTTP 404 | Display a not-found error (should not occur if the view was reached from a valid `job_id`) |

The Web UI does not poll for the job to transition to `Cancelled`; it acknowledges the intake and allows the existing auto-poll loop to observe the eventual status.

**Acceptance Criteria:**
- The view displays all fields defined above, rendering only the fields appropriate for the current `status`
- For non-terminal jobs, auto-polling updates the displayed status without a full page reload
- Polling stops when the job reaches `Completed` or `Failed`
- `solver_outcome` is always displayed prominently for `Completed` jobs
- `failure_reason` is always displayed for `Failed` jobs
- The view presents an explanatory note when `solver_outcome` on a `Completed` job indicates a non-success outcome (`Infeasible`, `Timeout`, `Cancelled`, `Failed`, `ContractViolation`), distinguishing solver-level outcome from Worker lifecycle failure
- The report link is displayed and navigates to FR-7 when `report_available = true`
- The cancellation action is visible for non-terminal jobs; it is not displayed or is disabled for terminal jobs
- A successful cancellation request (HTTP 202) displays an acknowledgement noting that cancellation is not guaranteed
- A cancellation request for a terminal job (HTTP 409) displays the conflict reason without causing an error state in the view
- A `404 Not Found` response for the initial `GET /v1/jobs/{job_id}` (unknown `job_id`) displays a clear not-found message

---

## FR-7: Evidence Report Access

**Description:**
The Web UI provides in-browser access to the HTML evidence report produced by the Report Generator (SPEC-009) for a completed job. The Web UI is a consumer of the report-serving contract defined by SPEC-008 FR-12 and FR-13.

**Report discovery:**
The Web UI fetches report metadata via `GET /v1/jobs/{job_id}/report` (SPEC-008 FR-12). On HTTP 200, the response provides the `report_id`, `report_url`, `generated_at`, and `report_format`.

**Report retrieval:**
The Web UI fetches the report content via the `report_url` returned by discovery, which resolves to `GET /v1/reports/{report_id}` (SPEC-008 FR-13). The API serves the content with `Content-Type: text/html`.

**In-browser rendering:**
The HTML report content is rendered in the browser. The rendering mechanism is an implementation planning decision. The Web UI must not modify the report content. The Report Generator owns the report content; SPEC-008 FR-13 confirms the API does not transform it; the Web UI must similarly serve as a transparent renderer.

**Download option:**
The user can save the HTML report file to disk. The download uses the report file served by the API without modification.

**Metadata display:**
The view displays the following report metadata alongside or prior to the report content:

| Field | Source |
|---|---|
| `report_id` | SPEC-008 FR-12 response |
| `generated_at` | SPEC-008 FR-12 response; ISO 8601 UTC |
| `report_format` | SPEC-008 FR-12 response (`html` at MVP scope) |
| `job_id` | SPEC-008 FR-12 response; link back to Job Detail View (FR-6) |

**Not-available state:**
When `GET /v1/jobs/{job_id}/report` returns HTTP 404, the view displays a message indicating the report is not yet available or was not produced for this job. This covers three conditions: job still executing, report generation failed, and `Failed` job (which does not produce an evidence report per SPEC-005 FR-17).

**Acceptance Criteria:**
- For a job with `report_available = true`, the Web UI fetches report metadata via `GET /v1/jobs/{job_id}/report` and then fetches the report content via the returned `report_url`
- The report HTML content is rendered in-browser without modification
- A download option allows the user to save the HTML file to disk
- A `404 Not Found` on report discovery displays a contextual not-available message
- The four metadata fields (`report_id`, `generated_at`, `report_format`, `job_id`) are displayed alongside the report content
- The `job_id` metadata entry links back to the Job Detail View (FR-6)
- The Web UI does not generate, transform, or re-process the report content

---

## FR-8: Experiment Observation

**Description:**
The Web UI provides read-only observation of experiments that were orchestrated by the CLI (SPEC-016). The Web UI is not an experiment orchestration executor; that role belongs to the CLI (ADR-012 Decision 1). The Web UI provides browser-accessible visibility into experiment state already persisted by the API (ADR-012 Decision 2).

The experiment observation views are read-only. No write operations to experiment, trial, or benchmark records are performed by the Web UI.

**Experiment lookup:**
The user enters an `experiment_id` to access a specific experiment. No experiment list view is included at MVP scope. No experiment list endpoint is currently defined in SPEC-008; experiment access is by identifier only, mirroring the job list dependency captured in OQ-1.

### FR-8.1: Experiment Status View

Consumes `GET /v1/experiments/{experiment_id}` (SPEC-008 FR-20).

Fields displayed:

| Field | Condition | Notes |
|---|---|---|
| `experiment_id` | Always | Full UUID; copyable |
| `experiment_name` | Always | |
| `benchmark_id` | Always | |
| `status` | Always | `Created`, `Running`, `Completed`, or `Failed`; visually differentiated |
| `planned_trial_count` | Always | |
| `effective_trial_count` | When set | |
| `workload_mode` | Always | `Fixed` or `Generated` |
| `reproducibility_class` | Always | `fully_reproducible`, `partially_reproducible`, or `non_reproducible` |
| `submitted_at` | Always | ISO 8601 UTC |
| `started_at` | When Running or later | ISO 8601 UTC |
| `completed_at` | When `status = Completed` | ISO 8601 UTC |
| `failed_at` | When `status = Failed` | ISO 8601 UTC |
| `failure_reason` | When `status = Failed` | |
| `trial_counts` | Always | Per-category breakdown: pending, submitted, completed, scheduler_rejected, harness_error, evidence_collected |

Navigation from the experiment status view:
- Link to the trial results view (FR-8.2) for this experiment
- Link to the experiment summary view (FR-8.3) when `status = Completed`

**Auto-refresh:**
For experiments in non-terminal states (`Created`, `Running`), the view auto-refreshes periodically. Auto-refresh stops when `status = Completed` or `status = Failed`.

### FR-8.2: Trial Results View

Consumes `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22).

Displays a table of trial records with the following columns:

| Column | Source field | Notes |
|---|---|---|
| Trial ID | `trial_id` | Abbreviated; full UUID accessible |
| Problem ID | `problem_id` | |
| Backend | `backend_id` | |
| Repetition | `repetition_index` | |
| Config | `problem_config_index` | |
| Trial Status | `trial_status` | Visually differentiated |
| Job ID | `job_id` | Link to Job Detail View (FR-6) when non-null; blank when null (not yet submitted) |
| Evidence Status | `evidence_status` | Blank before collection |
| Outcome | `solver_outcome` | When evidence collected |
| Quality | `hindsight_quality` | When available; per SPEC-007 FR-7 |
| Duration (ms) | `execution_duration_ms` | When available |

The table does not require additional per-job API calls beyond the single `GET /v1/experiments/{experiment_id}/trials` call. All column data is present in the SPEC-008 FR-22 response.

**Auto-refresh:**
For experiments in non-terminal states, the trial results view auto-refreshes at the same interval as the experiment status view (FR-8.1).

### FR-8.3: Experiment Summary

Consumes `GET /v1/experiments/{experiment_id}/summary` (SPEC-008 FR-23). Available only when `status = Completed`.

The summary JSON payload from the API is displayed and offered for download. The Web UI does not transform the summary content.

When the experiment exists but is not yet completed (HTTP 404, `error_code = EXPERIMENT_SUMMARY_NOT_FOUND`), the view displays a not-yet-available message. When the experiment does not exist (HTTP 404, `error_code = EXPERIMENT_NOT_FOUND`), the view displays a not-found message.

### FR-8.4: Benchmark Summary

Consumes `GET /v1/benchmarks/{benchmark_id}/summary` (SPEC-008 FR-24). Accessed via a benchmark ID lookup form.

The summary JSON payload is displayed and offered for download. The Web UI does not transform the content.

When no member experiment has yet completed (HTTP 404, `error_code = BENCHMARK_NOT_FOUND`), the view displays a not-yet-available message.

**Acceptance Criteria:**
- The experiment status view (FR-8.1) displays all fields from SPEC-008 FR-20, rendered appropriately for each `status`
- `trial_counts` is rendered as a breakdown displaying all trial status categories
- The trial results table (FR-8.2) displays all columns defined above, derived solely from the SPEC-008 FR-22 response without additional per-job API calls
- `job_id` entries in the trial table are clickable links to the Job Detail View (FR-6) when non-null; entries with `job_id = null` display a blank or placeholder indicator
- The experiment summary (FR-8.3) is offered for download as JSON when `status = Completed`
- `EXPERIMENT_SUMMARY_NOT_FOUND` (experiment exists but not complete) and `EXPERIMENT_NOT_FOUND` (unknown ID) produce distinct user-visible messages
- The benchmark summary (FR-8.4) is offered for download when available; `BENCHMARK_NOT_FOUND` produces a not-yet-available message
- Auto-refresh applies to experiment status and trial results views for non-terminal experiments
- The Web UI performs no write operations against experiment, trial, or benchmark resources; all FR-8 interactions are read-only

---

## FR-9: API Error Display

**Description:**
All API error responses (SPEC-008 FR-14 format) are surfaced to the user in a consistent, accessible presentation. The Web UI propagates the structured error information from SPEC-008 FR-14 without discarding fields or adding fabricated detail.

**Error presentation by HTTP status:**

| HTTP status | `error_code` | Web UI presentation |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Display each `field_errors` entry; associate with the corresponding form field where possible; show a summary for errors without a field mapping |
| 400 | `INVALID_SCHEDULER_CONFIG` | Display `field_errors` entries on the scheduler configuration creation form |
| 400 | `SCHEDULER_CONFIG_NOT_FOUND` | Display on the job submission form at the scheduler config field |
| 404 | Any | Contextual not-found message identifying the resource type and ID |
| 409 | `JOB_ALREADY_TERMINAL` | "This job has already reached a terminal state; cancellation is not applicable" |
| 409 | Any other | Display the `message` field from the SPEC-008 FR-14 error body |
| 500 | `INTERNAL_ERROR` | "An unexpected server error occurred. Please try again or contact the system operator." Reference the `request_id` for support correlation. |
| Network error | — | "Unable to reach the Daedalus API. Verify the API is running and the URL is correct." |

**`request_id` display:**
The SPEC-008 FR-14 `request_id` is displayed alongside every API error response to enable correlation with API observability data (ADR-006). This value is not regenerated by the Web UI; it is propagated unchanged from the API error body.

**Content safety:**
The Web UI never renders stack traces, connection strings, PostgreSQL error codes, or internal component names from API error responses. The `message` field from SPEC-008 FR-14 is safe to display per that specification's own security constraints.

**Acceptance Criteria:**
- Every non-2xx API response produces a visible, user-readable error message
- HTTP 400 `field_errors` are displayed; no entry is silently dropped
- Field-level errors are associated with the offending form field where a mapping exists
- HTTP 500 responses do not expose internal server details, stack traces, or connection strings
- The `request_id` from the SPEC-008 FR-14 error body is displayed for API errors
- Network-level errors (TCP/DNS failure) produce a connectivity error message; they do not display as generic HTTP 500 errors
- The Web UI does not fabricate, transform, or regenerate `request_id` values; it propagates the value from the API response

---

## FR-10: Observability

**Description:**
The Web UI is a browser client. It does not emit OpenTelemetry spans. All telemetry for DAEDALUS system behavior is provided by the API (SPEC-008 FR-17) and Worker (SPEC-005 FR-19).

API calls issued by the Web UI to the API will produce `job.submit`, `api.status_poll`, `api.report_serve`, and related spans owned by the API per SPEC-008 FR-17. The Web UI does not contribute additional spans and does not participate in trace context propagation.

**Acceptance Criteria:**
- The Web UI does not emit OpenTelemetry spans
- The Web UI does not inject `traceparent`, `tracestate`, or other distributed-tracing headers into API requests
- `request_id` values from SPEC-008 FR-14 error responses are displayed in the UI per FR-9, supporting operator correlation with API traces

---

# Non-Requirements

- The Web UI does not submit experiment manifests or drive the experiment trial orchestration loop (CLI responsibility per ADR-012 Decision 1)
- The Web UI does not trigger trial evidence collection (CLI responsibility per ADR-012 Decision 5)
- The Web UI does not submit benchmark manifests (CLI responsibility per SPEC-016 FR-17)
- The Web UI does not create routing problems for Generated Mode experiments (deferred pending SPEC-008 OQ-7)
- The Web UI does not support multi-tenant authentication or authorization at MVP scope (consistent with SPEC-008 Security Considerations; no auth layer exists at MVP scope)
- The Web UI does not generate, transform, or re-process HTML evidence reports (Report Generator and API responsibilities)
- The Web UI does not access PostgreSQL, RabbitMQ, the report volume, the Worker, or the Python Solver Adapter directly
- The Web UI does not configure or register solver backend capability profiles (not an API capability per SPEC-012 FR-2 scope)
- The Web UI does not support real-time push notifications; all live status updates use polling
- The Web UI does not support bulk job submission
- The Web UI does not support routing problem versioning, amendment, or deletion (SPEC-001 constraint)
- The Web UI does not cancel jobs from the experiment observation views (cancellation is only available from the Job Detail View, FR-6)
- Pagination for large result sets is not required at MVP scope
- The Web UI does not provide a terminal or CLI emulation interface
- The Web UI does not display experiment orchestration progress during an active CLI `experiment run` session; it shows persisted API state only
- The Web UI does not support Generated Mode workload set configuration at MVP scope (SPEC-008 OQ-7 deferred)

---

# Assumptions

1. The Daedalus API is reachable by the browser at a known base URL before the Web UI is used. The default is `http://localhost:5000`, consistent with the Docker Compose port mapping (architecture.md).
2. No authentication or authorization is required at MVP scope. All API endpoints are accessible without credentials. The Web UI mirrors the API's MVP authentication posture (SPEC-008 Security Considerations).
3. The API is configured to emit Cross-Origin Resource Sharing (CORS) headers permitting requests from the Web UI's origin. Without CORS headers, browser security policy will block API calls from the Web UI. This is a required API configuration change (see Architectural Impact).
4. HTML evidence reports produced by the Report Generator (SPEC-009) are well-formed HTML and render correctly in a modern browser without transformation.
5. The SPEC-008 FR-10 `GET /v1/scheduler-configs` response list places the default configuration as the first entry. The Web UI relies on this ordering to identify the default configuration for visual distinction.
6. All experiment state is owned by the API (ADR-012 Decision 2). At MVP scope, the CLI is the expected experiment orchestration executor (ADR-012 Decision 1). The Web UI displays whatever experiment state is present in the API regardless of how it was produced. The Web UI has no mechanism to create experiment state.
7. The Docker Compose environment is the expected deployment context. The Web UI is accessible at a browser-reachable address within that environment.

---

# Constraints

1. The Web UI must not access any DAEDALUS infrastructure component other than the Daedalus API. All system interaction is through SPEC-008 HTTP endpoints.
2. The Web UI must not perform SPEC-001 domain validation. Client-side checks are limited to structural input validation (type parseability, required fields, UUID format). Domain constraint enforcement is the responsibility of the API and Core per ADR-009.
3. The Web UI must not expose internal system identifiers that SPEC-008 prohibits from API responses: `execution_seed`, `decision_id`, solver run identifiers (SPEC-008 FR-7). These fields do not appear in any SPEC-008 response and therefore cannot appear in the Web UI.
4. The Web UI must not modify, transform, or augment HTML evidence report content before rendering. The report is the Report Generator's artifact; the Web UI is a transparent renderer.
5. The Web UI must not fabricate, regenerate, or alter `request_id` values from SPEC-008 FR-14 error responses; these are propagated unchanged per FR-9.
6. The Web UI technology stack is governed by a separate ADR. No implementation technology is prescribed by this specification.
7. The Web UI technology stack is governed by OQ-2. The Web UI serving mechanism and Docker Compose integration are governed by OQ-3. No container topology is prescribed by this specification.

---

# Inputs

| Input | Source | Format | Notes |
|---|---|---|---|
| Routing problem fields | Browser user input | Form fields serialized to SPEC-008 FR-2 JSON body | Structural validation in UI; domain validation by API |
| Scheduler configuration selection | User selection from `GET /v1/scheduler-configs` list | `scheduler_config_id` UUID | Optional; absent uses API default |
| Backend targeting directive | User selection or free text | `backend_id` string | Optional; absent means standard scheduler path |
| Job ID for status or report lookup | User text entry or navigation from dashboard | UUID string | Structural UUID format check in UI |
| Experiment ID for observation | User text entry | UUID string | Structural UUID format check in UI |
| Benchmark ID for summary lookup | User text entry | String | |
| API responses | Daedalus API (SPEC-008) | JSON per SPEC-008 FR-6, FR-8, FR-10, FR-12, FR-14, FR-20, FR-22, FR-23, FR-24 | Trusted; rendered without transformation |
| Report HTML content | Daedalus API `GET /v1/reports/{report_id}` | `text/html` (SPEC-008 FR-13) | Rendered verbatim; not modified |

---

# Outputs

| Output | Consumer | Format | Notes |
|---|---|---|---|
| Job submission request | Daedalus API `POST /v1/jobs` | JSON per SPEC-008 FR-2 | Formed from FR-2 form input |
| Scheduler config creation request | Daedalus API `POST /v1/scheduler-configs` | JSON per SPEC-008 FR-10 | Formed from FR-3 creation form |
| Job cancellation request | Daedalus API `POST /v1/jobs/{job_id}/cancel` | Empty body per SPEC-008 FR-11 | From FR-6 cancellation action |
| Status poll request | Daedalus API `GET /v1/jobs/{job_id}` | Path parameter | Auto-issued by FR-6 for non-terminal jobs |
| Report discovery request | Daedalus API `GET /v1/jobs/{job_id}/report` | Path parameter | From FR-7 |
| Report retrieval request | Daedalus API `GET /v1/reports/{report_id}` | Path parameter | From FR-7; resolves `report_url` |
| Experiment status request | Daedalus API `GET /v1/experiments/{experiment_id}` | Path parameter | From FR-8.1 |
| Trial results request | Daedalus API `GET /v1/experiments/{experiment_id}/trials` | Path parameter | From FR-8.2 |
| Experiment summary request | Daedalus API `GET /v1/experiments/{experiment_id}/summary` | Path parameter | From FR-8.3 |
| Benchmark summary request | Daedalus API `GET /v1/benchmarks/{benchmark_id}/summary` | Path parameter | From FR-8.4 |
| Rendered job submission result | Browser user | Browser UI | HTTP 202 → navigate to FR-6 |
| Rendered job detail | Browser user | Browser UI | SPEC-008 FR-8 response fields |
| Rendered evidence report | Browser user | HTML in browser | SPEC-008 FR-13 content; unmodified |
| Downloaded report file | Browser user | HTML file | SPEC-008 FR-13 content; unmodified |
| Rendered experiment state | Browser user | Browser UI | SPEC-008 FR-20, FR-22 response fields |
| Downloaded summary JSON | Browser user | JSON file | SPEC-008 FR-23 or FR-24 payload; unmodified |

---

# Failure Modes

### API Unreachable

**Condition:** Browser cannot reach the Daedalus API at the configured base URL (TCP failure, DNS failure, API container not running).
**Behavior:** Network error is surfaced per FR-9. The Web UI displays a connectivity error message and the attempted URL. No API state is modified.
**Recovery:** The user verifies that Docker Compose is running and the API is healthy.

---

### Job Submission Validation Failure (HTTP 400)

**Condition:** `POST /v1/jobs` returns HTTP 400 with `field_errors`.
**Behavior:** All `field_errors` entries are displayed per FR-9. The submission form remains populated so the user can correct and resubmit. No job record is created (SPEC-008 FR-3).
**Recovery:** User corrects the flagged fields and resubmits.

---

### Scheduler Configuration Not Found (HTTP 400, `SCHEDULER_CONFIG_NOT_FOUND`)

**Condition:** The `scheduler_config_id` selected during job submission references no stored configuration at the time the request is processed.
**Behavior:** HTTP 400 is received; the error is displayed on the submission form per FR-9. This can occur if a configuration was deleted between list fetch and submission (not currently possible at MVP scope per SPEC-008 Non-Requirements, but handled defensively).
**Recovery:** User re-fetches the configuration list and reselects.

---

### Job Not Found (HTTP 404)

**Condition:** `GET /v1/jobs/{job_id}` returns HTTP 404 for a `job_id` the user or UI attempted to access.
**Behavior:** The Job Detail View (FR-6) displays a not-found message. Auto-polling stops.
**Recovery:** User verifies the `job_id` is correct.

---

### Report Not Available (HTTP 404 from `GET /v1/jobs/{job_id}/report`)

**Condition:** Report metadata is not available: job is still executing, report generation failed, or the job is `Failed` (no evidence report produced per SPEC-005 FR-17).
**Behavior:** The Evidence Report View (FR-7) displays a not-available message explaining that the report is not yet available or was not produced.
**Recovery:** For in-progress jobs, the user waits for the job to complete. For `Failed` jobs, no report will be produced.

---

### Targeted Backend Ineligible (`NoEligibleSolver`)

**Condition:** The job was submitted with a `backend_id` targeting directive (FR-4). The Scheduler determines the targeted backend is ineligible. The job transitions to `Failed` with `failure_reason` indicating `NoEligibleSolver`.
**Behavior:** Submission returned HTTP 202 successfully. The Job Detail View (FR-6) shows `status = Failed` and the `failure_reason` when the job reaches terminal state via polling. The Web UI does not distinguish targeted-path `NoEligibleSolver` from standard-path `NoEligibleSolver` at the status level, consistent with SPEC-008 FR-8 behavior.
**Recovery:** The user resubmits without a `backend_id` (standard path) or selects a different `backend_id`.

---

### Report File Missing from Volume (HTTP 404 from `GET /v1/reports/{report_id}`)

**Condition:** Report metadata exists but the HTML file is absent from the report volume.
**Behavior:** The Evidence Report View (FR-7) receives HTTP 404 from the API and displays a not-available message. This is an infrastructure condition requiring operator investigation.
**Recovery:** Operator investigation required. The Web UI cannot recover from this condition.

---

### Experiment Not Found (HTTP 404, `EXPERIMENT_NOT_FOUND`)

**Condition:** `GET /v1/experiments/{experiment_id}` returns HTTP 404 for the entered `experiment_id`.
**Behavior:** The experiment status view (FR-8.1) displays a not-found message.
**Recovery:** User verifies the `experiment_id` is correct.

---

### Auto-Refresh API Error

**Condition:** An auto-refresh poll (`GET /v1/jobs/{job_id}`, `GET /v1/experiments/{experiment_id}`) returns a non-200 response or fails with a network error.
**Behavior:** The error is surfaced per FR-9. Auto-refresh may pause or retry at the next attempt depending on the implementation. The last successfully received data remains displayed.
**Recovery:** If the error is transient, the next successful poll restores the current state. If persistent, the user must reload.

---

# Architectural Impact

| Component | Impact |
|---|---|
| **API** (SPEC-008) | Requires CORS header configuration. Browser security policy requires the API to emit `Access-Control-Allow-Origin` and related CORS headers for requests from the Web UI's browser origin. This is a required deployment configuration change; it does not modify any API endpoint contract or introduce new endpoints. CORS configuration scope is defined by OQ-3 (serving mechanism determines the origin). |
| **Web UI** | New component. Consumes SPEC-008 endpoints from a browser context. No existing component behavior is modified. |
| **Docker Compose topology** | New service entry required when the Web UI is containerized. The exact topology addition depends on OQ-3 (serving mechanism). The API's published port must remain accessible from the browser. |
| **Worker** | None. The Web UI has no direct interaction with the Worker. |
| **Scheduler** | None. The Web UI has no direct interaction with the Scheduler. |
| **CLI** | None. The Web UI does not replace or modify CLI responsibilities. CLI remains the primary experiment orchestration interface. |
| **Observability** | Indirect. API calls from the browser produce spans owned by SPEC-008 FR-17 (`job.submit`, `api.status_poll`, etc.). The Web UI does not introduce new spans. Prometheus metrics defined in SPEC-008 FR-17 will reflect Web UI-driven API calls alongside CLI-driven calls; no metric label distinguishes the caller. |
| **Security** | The Web UI introduces a browser-accessible surface. No authentication exists at MVP scope per SPEC-008 Security Considerations. The browser's same-origin policy provides cross-site request isolation. The Web UI must not expose internal identifiers prohibited by SPEC-008 FR-7. Path traversal for report IDs is already prevented by the API's SPEC-008 FR-13 UUID-lookup mechanism. |
| **System Context diagram** | Addition required. The Web UI is a new client node in the architecture.md System Context diagram, parallel to the CLI. |

**SPEC-008 CORS dependency:**
This specification requires the API to be configured to emit CORS response headers permitting requests from the browser origin serving the Web UI. This is an operational and deployment configuration requirement on the API, not a change to any API endpoint contract defined by SPEC-008. The specific origins permitted are determined by OQ-3. This requirement must be noted in the Documentation Updates Required section of SPEC-008 when SPEC-021 is accepted.

---

# Testability

1. **Unit: Form serialization — required field present** — A job submission form with all required routing problem fields populated produces a `POST /v1/jobs` request body that includes `routing_problem` with all expected fields.

2. **Unit: Form serialization — optional fields absent** — A job submission form with no `scheduler_config_id` selected and no `backend_id` entered produces a request body without those fields.

3. **Unit: Form serialization — backend_id present** — A job submission form with `backend_id = "qubo-simulated-annealing"` selected produces a request body with `backend_id = "qubo-simulated-annealing"`.

4. **Unit: Client-side structural validation — missing required field** — A submission attempt with `vehicle_count` empty does not issue a `POST /v1/jobs` API call; the form displays a required-field error for `vehicle_count`.

5. **Unit: Client-side structural validation — non-parseable numeric field** — A submission attempt with `average_vehicle_speed_kmh = "abc"` does not issue an API call; the form displays a parse error.

6. **Unit: Client-side structural validation — empty backend_id blocked** — A submission attempt with the `backend_id` field populated with an empty string does not issue an API call; the form enforces non-empty.

7. **Unit: Client-side validation does not block domain errors** — A submission with `depot.latitude = 95.0` (out of range per SPEC-001 FR-5) passes client-side structural checks and issues the API call; the HTTP 400 `field_errors` entry for `depot.latitude` is surfaced post-submission.

8. **Integration: Successful submission → Job Detail View** — A valid routing problem submission produces HTTP 202 from the API and navigates the browser to the Job Detail View for the returned `job_id`. The `job_id` and `problem_id` from the API response are displayed.

9. **Integration: Validation failure display** — A submission that returns HTTP 400 with two `field_errors` entries displays both entries in the UI without dropping either.

10. **Integration: Scheduler config list populated** — When the submission form is displayed, `GET /v1/scheduler-configs` has been called and the returned configurations are shown in the selection list.

11. **Integration: Default config identified** — The default scheduler configuration (first entry per SPEC-008 FR-10) is visually distinguished in the selection list.

12. **Integration: Config selection passed to submission** — Selecting a non-default configuration from the list causes the subsequent job submission to include that configuration's `scheduler_config_id` in the request body.

13. **Integration: Job detail status polling** — For a non-terminal job, the Job Detail View polls `GET /v1/jobs/{job_id}` and updates the displayed `status` when the API reflects a state change, without a full page reload.

14. **Integration: Polling stops on terminal state** — When the displayed job transitions to `Completed` or `Failed`, the Job Detail View stops issuing status poll requests.

15. **Integration: solver_outcome prominently displayed** — A `Completed` job with `solver_outcome = Infeasible` shows `solver_outcome` prominently and includes an explanatory note distinguishing solver-level outcome from Worker lifecycle failure.

16. **Integration: Report link visible and functional** — A `Completed` job with `report_available = true` shows a link in the Job Detail View; activating the link navigates to the Evidence Report View for that `job_id` and fetches the report metadata.

17. **Integration: Report rendered without modification** — The Evidence Report View fetches the HTML content from `GET /v1/reports/{report_id}` and renders it; the content is byte-identical to the API response body.

18. **Integration: Report not available message** — For a `Failed` job (no evidence report produced), the Evidence Report View receives HTTP 404 and displays a not-available message; no report content is shown.

19. **Integration: Cancellation acknowledged** — Activating the cancellation action on a non-terminal job issues `POST /v1/jobs/{job_id}/cancel`; a HTTP 202 response displays the cancellation acknowledgement message noting cancellation is not guaranteed.

20. **Integration: Cancellation conflict displayed** — Activating the cancellation action on a `Completed` job returns HTTP 409; the UI displays a conflict message indicating the job is already terminal, without causing an error state in the view.

21. **Integration: Experiment status fields displayed** — Entering a known `experiment_id` on the experiment status view retrieves and renders all fields from the SPEC-008 FR-20 response, including the `trial_counts` breakdown.

22. **Integration: Trial results table — null job_id** — A trial with `job_id = null` in the SPEC-008 FR-22 response displays a blank or placeholder in the Job ID column; no link is rendered.

23. **Integration: Trial results table — job_id link** — A trial with a non-null `job_id` renders a clickable link in the Job ID column that navigates to the Job Detail View for that `job_id`.

24. **Integration: Experiment summary download** — For a `Completed` experiment, the experiment summary view provides a download option that triggers a file save of the SPEC-008 FR-23 JSON payload.

25. **Integration: Experiment not found** — An unknown `experiment_id` returns HTTP 404 (`EXPERIMENT_NOT_FOUND`) from the API; the experiment status view displays a not-found message.

26. **Integration: Experiment summary not yet available** — An experiment in `Running` status returns HTTP 404 (`EXPERIMENT_SUMMARY_NOT_FOUND`) from `GET /v1/experiments/{id}/summary`; the view displays a not-yet-available message distinct from the not-found message.

27. **Integration: Benchmark not found** — An unknown or pending `benchmark_id` returns HTTP 404 (`BENCHMARK_NOT_FOUND`); the benchmark summary view displays a not-yet-available message.

28. **Unit: Error display — request_id propagated** — An HTTP 500 response from the API contains a `request_id` in the SPEC-008 FR-14 error body; the displayed error in the Web UI includes this `request_id` unchanged.

29. **Unit: Error display — no internal details** — An HTTP 500 response body does not result in stack traces, connection strings, or internal component names appearing in the rendered UI.

30. **Integration: CORS — API calls reach the API** — With CORS headers configured on the API, a browser-issued `POST /v1/jobs` from the Web UI origin reaches the API and returns HTTP 202 without being blocked by browser same-origin policy.

31. **Integration: Dashboard auto-refresh stops on navigation** — While the job dashboard (FR-5) is auto-refreshing, navigating away from the dashboard view stops further polling requests against the job list endpoint; no additional calls are issued after navigation.

32. **Integration: Experiment polling stops on terminal state** — For an experiment with auto-refresh active across FR-8.1 and FR-8.2, when the experiment transitions to `status = Completed` or `status = Failed`, the auto-refresh stops issuing further requests to `GET /v1/experiments/{experiment_id}` and `GET /v1/experiments/{experiment_id}/trials`.

---

# Observability Requirements

Operational questions the Web UI and its API interactions must support:

1. How many job submissions originate from the Web UI vs. the CLI? — Not directly distinguishable from API-side spans at MVP scope; no caller-identity label is defined in SPEC-008 FR-17. This is an accepted limitation.
2. Is the API accessible from the browser? — Answered by browser-side network error handling (FR-9, connectivity error message) and `daedalus_api_jobs_submitted_total` counter in SPEC-008 FR-17.
3. Are report files rendering successfully? — Answered by `daedalus_api_status_poll_total` and `api.report_serve` spans from SPEC-008 FR-17 (API-side); browser rendering errors are surfaced to the user in the Evidence Report View (FR-7).
4. What fraction of job submissions from any client fail validation? — Answered by `daedalus_api_submissions_rejected_total` in SPEC-008 FR-17.

The Web UI does not introduce new Prometheus metrics or OpenTelemetry spans. Observability for the system as a whole is owned by SPEC-008 FR-17, SPEC-005 FR-19, and ADR-006.

---

# Security Considerations

**No authentication at MVP scope:**
Consistent with SPEC-008 Security Considerations, no authentication or authorization layer exists at MVP scope. The Web UI assumes a trusted local development environment. All API endpoints are accessible without credentials. Authentication and authorization must be implemented before any exposure to untrusted networks or production deployment.

**Input handling:**
All user input collected by the Web UI is transmitted to the API as HTTP request payloads. The API is the trust boundary and performs all validation before any side effects occur (SPEC-008 FR-3). The Web UI must not construct SQL queries, filesystem paths, or queue payloads from user input; all input is serialized to JSON and sent to the API.

**Report rendering:**
HTML evidence reports are produced by the Report Generator (SPEC-009), served by the API (SPEC-008 FR-13), and rendered by the browser. The report content is authored by the system, not by untrusted user input. The Web UI must not inject user-supplied content into rendered report HTML.

When report content is rendered in an embedded frame within the Web UI page, the frame must apply appropriate browser sandboxing to restrict script execution, form submission, and top-level navigation originating from report content. The specific sandboxing policy is an implementation planning decision resolved alongside OQ-2. SPEC-009 governs safe generation of report HTML content and is the authority for what the report may contain; the Web UI governs the rendering container and is responsible for the sandbox policy applied to it.

**CORS configuration:**
The API must be configured to emit CORS headers permitting requests from the Web UI's origin. The permitted origin should be scoped narrowly to the Web UI's serving address within the Docker Compose environment. Wildcard `*` CORS is acceptable only in a fully trusted local development environment; it must not be used in any environment with network exposure.

**Internal identifier isolation:**
The Web UI must not expose `execution_seed`, `decision_id`, or solver run identifiers in any rendered output. These fields do not appear in any SPEC-008 response body and therefore cannot appear in the Web UI's display.

**Path traversal:**
Report retrieval (FR-7) uses the `report_url` from the API's SPEC-008 FR-12 response, which resolves to a server-side UUID-based lookup (SPEC-008 FR-13). The Web UI does not construct file paths from user input; path traversal prevention is owned by the API's existing `report_id` UUID-lookup mechanism.

---

# Performance Considerations

**Polling overhead:**
The Job Detail View (FR-6) polls at 3-second intervals for non-terminal jobs. The Experiment Observation view (FR-8) polls at 10-second intervals for non-terminal experiments. Each poll is a single `GET` request to the API, which in turn performs a single PostgreSQL read (SPEC-008 Performance Considerations). Polling overhead is bounded by the number of concurrent Web UI sessions; at MVP single-developer scope this is not a concern.

**Configuration list fetch:**
`GET /v1/scheduler-configs` is called when the submission form is loaded (FR-3). The response is small (few configurations at MVP scope). The list may be cached within a user session; invalidation on configuration creation ensures the new configuration appears in the list. Cache lifetime is an implementation planning detail.

**Trial results table:**
`GET /v1/experiments/{experiment_id}/trials` returns all trial records for an experiment in a single response. At MVP trial counts (single-digit to low-double-digit experiments with a modest solver set and repetition count), rendering the full table in-browser without pagination is safe. Pagination is deferred per Non-Requirements.

**Report rendering:**
HTML report file sizes are bounded by SPEC-009 report content. Reports are served in full from the API's report volume (SPEC-008 FR-13). No streaming or chunked rendering is required at MVP scope.

---

# Documentation Updates Required

- **docs/architecture.md**: The System Context diagram should be updated to include the Web UI as a new client node parallel to the CLI, with an arrow from the Web UI to the Daedalus API. The MVP Container Topology diagram should be updated when OQ-3 (serving mechanism) is resolved to reflect the new container or serving configuration.
- **SPEC-008 (API / Control Plane)**: A documentation update is required to note that the API must be configured to emit CORS response headers permitting browser requests from the Web UI's origin. This is a deployment configuration requirement, not an endpoint contract change. The update should appear in the SPEC-008 Constraints or Assumptions section. No new SPEC-008 functional requirement is introduced.
- **README.md**: The "Deferred" section lists "Full dashboard." SPEC-021 introduces a scoped MVP Web UI. The README should be updated to reflect that a scoped Web UI is in MVP scope when SPEC-021 is accepted, while noting that the full dashboard remains a post-MVP capability.

---

# Open Questions

### OQ-1: Job List API Endpoint

**Question:** The job dashboard (FR-5) requires a `GET /v1/jobs` list endpoint. This endpoint is referenced in SPEC-016 OQ-1 but is not currently defined in SPEC-008. Should this endpoint be defined as part of the SPEC-008 amendment that enables the Web UI, or should the dashboard fall back to job-ID lookup until the endpoint exists?

**Why it matters:** The job dashboard is the primary at-a-glance view of system activity. Without a job list endpoint, the dashboard must fall back to a job-ID lookup form, which reduces the out-of-the-box discoverability of system state.

**Options under consideration:**
1. Define `GET /v1/jobs` (with optional `status` and `limit` query parameters) as a SPEC-008 amendment before Web UI implementation begins. This unblocks both the Web UI dashboard and the SPEC-016 `job list` command (SPEC-016 FR-9 OQ-1).
2. Implement the fallback job-ID lookup form for the dashboard at MVP scope; defer the list endpoint to a SPEC-008 amendment triggered by a future requirement.

**Owner:** Project Owner decision on SPEC-008 amendment scope. Resolution required before Web UI dashboard implementation begins.

**Blocking:** Blocking for FR-5 full implementation. FR-5 fallback behavior (job-ID lookup form) is not blocked.

---

### OQ-2: Web UI Technology Stack

**Question:** What browser-side technology (framework, build toolchain) is used to implement the Web UI?

**Why it matters:** The technology choice affects the build pipeline, static asset serving, and the Docker Compose service definition. ADR-001 (C++) and ADR-002 (C#) established technology choices for the runtime and API respectively. A parallel decision record for the Web UI is appropriate.

**Owner:** An ADR is required before implementation begins. Criteria: employer-signaling value of the chosen stack, compatibility with the Docker Compose development environment, and alignment with the project's portfolio thesis.

**Blocking:** Blocking for Web UI implementation. Not blocking for Draft or Proposed spec status.

---

### OQ-3: Web UI Serving Mechanism and Docker Compose Integration

**Question:** How is the Web UI served within the Docker Compose environment, and what origin does it run from for CORS purposes?

**Options under consideration:**
1. **Separate `web-ui` container:** A dedicated container (e.g., Nginx serving static build output) added to Docker Compose. The container serves the Web UI at a distinct port (e.g., `http://localhost:3000`). The API must emit CORS headers for this origin.
2. **Served from the API container:** Static Web UI assets are served from the existing `api` container at a sub-path (e.g., `http://localhost:5000/ui`). No CORS configuration is required (same origin). This couples the Web UI build artifact to the API container build.
3. **Developer-served (outside Docker Compose):** The Web UI is run with a development server on the developer machine (e.g., `http://localhost:3000`) outside Docker Compose, similar to the CLI. The API must emit CORS headers for this origin. The Web UI is not deployed as a container.

**Why it matters:** The serving mechanism determines the CORS origin that the API must permit, the Docker Compose topology change, and whether the Web UI has its own container lifecycle.

**Owner:** Project Owner decision. Resolution required before API CORS configuration can be specified and before the Docker Compose topology can be updated in architecture.md.

**Blocking:** Blocking for Docker Compose topology finalization and API CORS configuration. Not blocking for Draft or Proposed spec status.

---

### OQ-4: API Base URL Discovery

**Question:** How does the Web UI browser application discover the Daedalus API base URL at runtime?

**Why it matters:** The Web UI must know the API base URL to issue any API request. Assumption 1 states the default is `http://localhost:5000`, but defines no mechanism by which this URL is communicated to the browser application. For a browser-hosted application, the URL cannot be read from environment variables at runtime as a CLI would; it must be embedded at build time, fetched from a well-known location, or derived from the current origin. The correct mechanism depends on the serving approach resolved in OQ-3: if the Web UI is served from the API container at a sub-path (OQ-3 Option 2), the API URL is the same origin and no external configuration is required; if the Web UI is served separately (OQ-3 Options 1 or 3), the API URL must be communicated to the browser application through a defined channel.

**Options under consideration:**
1. Build-time environment variable (e.g., `VITE_API_BASE_URL` or equivalent): the API URL is baked into the static build artifact at build time. Simple; requires a rebuild to change the URL.
2. Runtime configuration file: a `config.json` served alongside the static assets at a fixed relative path; the browser fetches it on startup before issuing API calls. Allows URL reconfiguration without a rebuild.
3. Same-origin derivation: the Web UI derives the API URL from `window.location.origin`. Only applicable under OQ-3 Option 2 (serving from the API container).

**Owner:** Resolution is a dependency of OQ-3. The correct mechanism cannot be selected until the serving approach is known. The selection may be addressed within the same ADR that resolves OQ-3.

**Blocking:** Not blocking for Draft or Proposed spec status. Blocking for Web UI implementation.

---

# Acceptance Checklist

- [ ] Problem is clearly defined
- [ ] Domain concept is defined
- [ ] Scope and responsibility boundary is defined
- [ ] FR-1 component responsibilities are defined
- [ ] FR-2 job submission form is defined
- [ ] FR-3 scheduler configuration selection is defined
- [ ] FR-4 backend targeting is defined
- [ ] FR-5 job dashboard is defined (contingent on OQ-1)
- [ ] FR-6 job detail view and cancellation is defined
- [ ] FR-7 evidence report access is defined
- [ ] FR-8 experiment observation (status, trials, summary, benchmark) is defined
- [ ] FR-9 API error display is defined
- [ ] FR-10 observability position is defined
- [ ] Non-requirements are documented
- [ ] Assumptions are explicit
- [ ] Constraints are documented
- [ ] Inputs and outputs are defined
- [ ] Failure modes are defined
- [ ] Architectural impact is documented
- [ ] Testability cases exist for each FR
- [ ] Observability requirements are stated
- [ ] Security considerations are documented
- [ ] Performance considerations are documented
- [ ] Documentation updates are identified
- [ ] OQ-1 documented (job list endpoint dependency)
- [ ] OQ-2 documented (technology stack)
- [ ] OQ-3 documented (serving mechanism and CORS)
- [ ] OQ-4 documented (API base URL discovery)

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-10) are implemented and acceptance criteria pass
- OQ-1 resolved: either `GET /v1/jobs` is defined in a SPEC-008 amendment and the full dashboard is implemented, or the fallback job-ID lookup form is implemented per FR-5 fallback behavior
- OQ-2 resolved: Web UI technology stack is selected via ADR before implementation begins
- OQ-3 resolved: serving mechanism and Docker Compose integration are defined; CORS configuration on the API is applied and verified
- OQ-4 resolved: API base URL discovery mechanism is defined; the Web UI locates the API at runtime without manual configuration in standard deployment
- The Web UI accesses the API solely through SPEC-008 HTTP endpoints; no direct database, queue, or Worker access exists
- Client-side structural validation is in place; no SPEC-001 domain validation is duplicated in the Web UI
- The job submission form (FR-2) produces correct SPEC-008 FR-2 JSON request bodies for all field combinations (with and without `scheduler_config_id`, with and without `backend_id`)
- Auto-polling for non-terminal jobs (FR-6) and non-terminal experiments (FR-8) stops when terminal state is reached
- Evidence report HTML content is rendered verbatim without modification (FR-7)
- All API error `field_errors` entries are surfaced without loss (FR-9)
- `request_id` from SPEC-008 FR-14 error responses is displayed in all error conditions (FR-9)
- No OpenTelemetry spans, `traceparent` headers, or internal DAEDALUS identifiers (`execution_seed`, `decision_id`) appear in any Web UI output
- CORS headers on the API permit browser requests from the Web UI's serving origin
- Engineering review passes
- Architecture review passes
- Specification status is updated to Verified
