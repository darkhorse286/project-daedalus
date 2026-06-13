# SPEC-008: API / Control Plane

## Metadata

**Feature ID:** SPEC-008

**Title:** API / Control Plane

**Status:** Proposed

**Author:** Darkhorse286

**Created:** 2026-06-12

**Last Updated:** 2026-06-12

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-002, ADR-003, ADR-004, ADR-006, ADR-009, ADR-011

**Related Specs:** SPEC-001, SPEC-003, SPEC-005, SPEC-006

---

# Problem Statement

The architecture defines the Daedalus API as the C# ASP.NET Core control plane responsible for routing job submission, job status visibility, scheduler configuration creation and retrieval, cancellation intake, and report serving. No specification currently defines the precise endpoint contracts, request and response models, validation behavior, error response format, idempotency expectations, or observability obligations governing the API's external-facing behavior.

Without a concrete API specification:

- Callers have no defined contract for submitting routing problems, associating scheduler configurations, or correlating submissions with asynchronous outcomes.
- The boundary between what the API validates (fast feedback, ADR-009) and what Core validates (authoritative, ADR-009) is not formally documented at the API layer.
- The externally observable job state model is not bounded — the API could invent states inconsistent with the SPEC-005 lifecycle.
- The cancellation request interface has no intake contract.
- The report serving model is undefined, leaving the architecture's "API serves report metadata and links" responsibility un-specified.
- The observability obligations for the `job.submit` span (required by architecture.md and ADR-006) are unowned.

SPEC-008 provides the authoritative contract definition for the Daedalus API. It closes the gap between the architecture description and the implementable HTTP interface.

---

# Business Value

- Makes the API independently implementable against a concrete, testable contract
- Ensures caller-facing validation feedback is consistent with SPEC-001 domain rules and ADR-009 dual-validation policy
- Defines the external job lifecycle view, preventing state model drift from SPEC-005
- Establishes report discovery and retrieval interfaces so the HTML evidence reports become accessible artifacts
- Provides the `job.submit` span definition required to close the observability chain from submission through Worker completion
- Demonstrates production-grade API contract design, input validation, and asynchronous pattern handling as portfolio artifacts

---

# Employer Signaling

- System Design
- Distributed Systems
- API Contract Design
- Reliability Engineering
- Observability

---

# Domain Concept

The Daedalus API is the external integration boundary. It accepts routing problems from callers, validates them, persists them, and hands them off to the asynchronous execution system. It does not execute optimization, evaluate solution quality, or produce evidence. It makes the evidence produced by the Worker and Core observable and retrievable.

The API is a stateless control plane in the sense that it holds no optimization state. All job state lives in PostgreSQL. All asynchronous execution state lives in the Worker's interaction with RabbitMQ and PostgreSQL. The API reads and writes PostgreSQL but does not touch the Worker directly.

A routing job in the API's view is an asynchronous work item: submitted synchronously, acknowledged immediately, executed asynchronously, and queryable for status. The API does not wait for execution to complete. A `job_id` is the caller's correlation handle across this asynchronous boundary.

---

# Requirements

### FR-1: Responsibility Scope and Component Boundaries

**Description:**
SPEC-008 defines the API's responsibilities and explicitly allocates the adjacent responsibilities it must not perform.

| Component | Responsibilities at This Boundary | Explicitly Not Responsible For |
|---|---|---|
| **API** (SPEC-008) | Request receipt; structural and fast domain validation; routing problem persistence; job creation; queue publication; job status visibility; scheduler configuration creation and retrieval; cancellation request intake; report metadata and file serving; `job.submit` span emission; error response production | Solver execution; backend selection; workload feature extraction; quality evaluation; report generation; evidence record persistence; Worker execution behavior |
| **Worker** (SPEC-005) | Job consumption; solver execution; evidence persistence; report generation invocation; job lifecycle state updates | Job submission handling; external API contract; queue publication |
| **Core** (architecture.md) | Authoritative domain validation (ADR-009); workload feature extraction; quality evaluation and regret calculation | External API; persistence; queue interaction |
| **PostgreSQL** (ADR-004) | Durable persistence for all jobs, problems, configurations, and evidence records | Computation; execution; API contract |
| **RabbitMQ** (ADR-003) | Asynchronous execution boundary between API and Worker; at-least-once delivery | Validation; execution; reporting |

**Acceptance Criteria:**
- No API code path performs solver execution, backend selection, workload feature computation, or quality evaluation
- No API code path writes Evidence Log records (job record initial creation is API responsibility; Evidence Log artifacts are Worker responsibility per SPEC-005 and SPEC-006)
- No API code path publishes directly to any queue other than the routing-jobs queue
- The API does not construct or interpret SolverRequests or SolverResponses

---

### FR-2: Job Submission Request Model

**Description:**
A job submission request is an HTTP POST to the job submission endpoint. The request body is a JSON document containing a routing problem and an optional scheduler configuration identifier.

**Endpoint:** `POST /v1/jobs`

**Request body fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `routing_problem` | object | Yes | A routing problem per SPEC-001 FR-11 JSON serialization contract |
| `scheduler_config_id` | UUID string | No | References an existing scheduler configuration (SPEC-003 FR-13). When absent, the default configuration is used (SPEC-003 FR-14). |

**Routing problem fields** (per SPEC-001):

| Field | Type | Required | Constraint |
|---|---|---|---|
| `seed` | uint64 | Yes | Non-negative 64-bit integer (SPEC-001 FR-6) |
| `vehicle_count` | positive integer | Yes | ≥ 1 (SPEC-001 FR-2) |
| `capacity_per_vehicle` | positive integer | Yes | ≥ 1 (SPEC-001 FR-2) |
| `average_vehicle_speed_kmh` | float64 | Yes | > 0.0; required for route simulation (ODR-1; SPEC-001 revision pending) |
| `depot` | object | Yes | `{latitude, longitude}` — geographic coordinates (SPEC-001 FR-3, FR-5) |
| `stops` | array | Yes | One or more stop objects; must not be empty (SPEC-001 FR-4) |

**Stop object fields:**

| Field | Type | Required | Constraint |
|---|---|---|---|
| `id` | non-negative integer | Yes | Unique within the problem (SPEC-001 FR-4) |
| `latitude` | float64 | Yes | ∈ [−90.0, 90.0] (SPEC-001 FR-5) |
| `longitude` | float64 | Yes | ∈ [−180.0, 180.0] (SPEC-001 FR-5) |
| `demand` | non-negative integer | Yes | In units of capacity (SPEC-001 FR-4) |
| `time_window_open` | non-negative integer | No | Seconds from route start; if present, `time_window_close` must also be present (SPEC-001 FR-9) |
| `time_window_close` | non-negative integer | No | Seconds from route start; must be strictly greater than `time_window_open` (SPEC-001 FR-9) |
| `service_duration` | non-negative integer | No | Seconds; defaults to 0 when absent (SPEC-001 FR-16) |

**Content-Type:** `application/json`

**Acceptance Criteria:**
- A request without `routing_problem` is rejected with HTTP 400 before any persistence occurs
- A request with an unrecognized top-level field is accepted; the field is ignored (permissive structural parsing)
- A `scheduler_config_id` that is not a valid UUID format is rejected with HTTP 400
- The `routing_problem` object must satisfy the schema above; any missing required field or type violation is rejected with HTTP 400 identifying the offending field(s)

---

### FR-3: Submission Request Validation

**Description:**
The API performs two validation layers before persisting the routing problem. Both layers must pass before persistence begins. Validation implements the API's side of the dual-validation policy (ADR-009).

**Layer 1: Structural validation**

JSON schema conformance, field type checking, and required field presence. Applied before domain validation.

Structural validation failures are immediately rejected with HTTP 400 and a structured error identifying each offending field (FR-14).

**Layer 2: Fast domain validation**

Domain validation rules derived from SPEC-001, applied for fast caller feedback (ADR-009). The API implements these rules independently in C#; Core re-validates in C++ as the authoritative validator (ADR-009).

| Rule | SPEC-001 Reference | Rejection message |
|---|---|---|
| `vehicle_count` ≥ 1 | FR-2 | `vehicle_count must be a positive integer` |
| `capacity_per_vehicle` ≥ 1 | FR-2 | `capacity_per_vehicle must be a positive integer` |
| `average_vehicle_speed_kmh` > 0.0 | ODR-1 | `average_vehicle_speed_kmh must be a positive number` |
| Depot latitude ∈ [−90.0, 90.0] | FR-5 | `depot.latitude is out of range` |
| Depot longitude ∈ [−180.0, 180.0] | FR-5 | `depot.longitude is out of range` |
| At least one stop | FR-4 | `stops must contain at least one stop` |
| No duplicate stop `id` values | FR-4 | `duplicate stop id: {id}` |
| Each stop latitude ∈ [−90.0, 90.0] | FR-5 | `stops[{id}].latitude is out of range` |
| Each stop longitude ∈ [−180.0, 180.0] | FR-5 | `stops[{id}].longitude is out of range` |
| `demand` ≥ 0 for each stop | FR-4 | `stops[{id}].demand must be non-negative` |
| `service_duration` ≥ 0 for each stop (when present) | FR-16 | `stops[{id}].service_duration must be non-negative` |
| If `time_window_open` present then `time_window_close` must be present (and vice versa) | FR-9 | `stops[{id}]: time_window_open and time_window_close must both be present or both absent` |
| `time_window_open` < `time_window_close` (when both present) | FR-9 | `stops[{id}].time_window_open must be less than time_window_close` |
| Total stop demand ≤ total fleet capacity (`vehicle_count × capacity_per_vehicle`) | FR-8 | `total stop demand ({N}) exceeds total fleet capacity ({M})` |

**Scheduler configuration validation:**

| Rule | Description |
|---|---|
| `scheduler_config_id` exists in PostgreSQL | A provided `scheduler_config_id` that references no stored configuration is rejected: HTTP 400, `scheduler_config_id not found` |
| No validation when absent | Absent `scheduler_config_id` uses the default configuration (SPEC-003 FR-14); no existence check is required |

**Validation ordering:**

1. Structural validation — all failures collected and returned together.
2. Fast domain validation — applied only when structural validation passes. All domain failures collected and returned together in a single response.
3. Scheduler configuration existence check — applied only when domain validation passes.

**Divergence from Core:**

If the API accepts a routing problem that Core subsequently rejects (ADR-009 divergence), the job transitions to `Failed` with a `CoreValidationRejection` failure record (SPEC-005 FR-5). The caller receives HTTP 202 Accepted and discovers the failure by polling job status. This divergence is an observable fault condition requiring investigation (ADR-009).

**Acceptance Criteria:**
- Structural validation failures are returned as HTTP 400 before any database access
- Fast domain validation failures are returned as HTTP 400 before any database access
- All validation failures in a layer are collected and returned in a single response — callers receive all errors, not just the first
- A `scheduler_config_id` that references no stored configuration is rejected with HTTP 400 — the system does not silently substitute the default (SPEC-001 FR-13)
- A submission that passes all three validation stages proceeds to FR-4 persistence
- The API does not re-run Core's authoritative validation in C#; it implements the fast-feedback approximation defined in ADR-009

---

### FR-4: Job Creation and Persistence

**Description:**
After all validation passes, the API creates the job and persists the routing problem to PostgreSQL before publishing to the queue.

**Creation sequence:**

1. Generate `problem_id`: UUID, system-generated (SPEC-001 FR-1).
2. Generate `job_id`: UUID, system-generated. The `job_id` is the caller's correlation handle and the Evidence Log root key (SPEC-006 FR-2).
3. Resolve `scheduler_config_id`: use the provided value, or the default configuration's ID (SPEC-003 FR-14).
4. Persist the routing problem record to PostgreSQL with `problem_id`. All SPEC-001 fields are persisted. Persistence completes before queue publication (SPEC-001 FR-12).
5. Create the job record in PostgreSQL with `job_id`, `problem_id`, `scheduler_config_id`, `status = Pending`, `cancellation_requested = false`, and `created_at` timestamp (SPEC-006 FR-4.2).
6. Steps 4 and 5 execute atomically: either both succeed or neither is committed. If persistence fails, no job message is published and HTTP 500 is returned to the caller.

**Acceptance Criteria:**
- `problem_id` is a UUID generated by the API and returned in the response
- `job_id` is a UUID generated by the API and returned in the response
- The routing problem record is persisted to PostgreSQL before the job message is published to RabbitMQ
- The job record initial status is `Pending`
- The job record initializes `cancellation_requested = false`; the field must exist in the record at creation so that FR-11 can write `true` to it when a cancellation request is accepted
- A PostgreSQL failure during persistence causes HTTP 500; no queue message is published
- The `problem_id` appears on all subsequent log events, spans, and evidence records for the job

---

### FR-5: Queue Publication

**Description:**
After successful persistence, the API publishes a job message to the RabbitMQ routing-jobs queue (ADR-003).

**Message content:**

| Field | Value |
|---|---|
| `job_id` | The UUID generated in FR-4 |
| `problem_id` | The UUID generated in FR-4 |
| `scheduler_config_id` | The resolved configuration identifier |

**AMQP message metadata:**
At publication time, the API injects W3C TraceContext (`traceparent`, optionally `tracestate`) from the active `job.submit` span context into the AMQP message `application_headers`. This is message-level metadata, distinct from the message payload body. The payload fields (`job_id`, `problem_id`, `scheduler_config_id`) are unchanged. The trace context schema carried in `application_headers` is owned by ADR-011.

**Publication semantics:**

- The message is published only after both the routing problem record and the job record are successfully persisted (FR-4).
- If publication fails, the API returns HTTP 500. The job record has already been persisted with `status = Pending`. The job message was not published; the job will not be executed unless requeued by an operator. This is an infrastructure failure condition.
- The API does not retry queue publication on failure. Recovery from failed publication is an operational concern.
- The API does not wait for the Worker to process the job. Publication is fire-and-forget from the API's perspective.

**Acceptance Criteria:**
- No job message is published unless routing problem and job records are committed to PostgreSQL
- The published message payload body contains exactly `job_id`, `problem_id`, and `scheduler_config_id`
- The published AMQP message includes W3C TraceContext (`traceparent`, optionally `tracestate`) injected from the active `job.submit` span context into the AMQP message `application_headers` (ADR-011)
- A RabbitMQ publication failure after successful PostgreSQL persistence causes HTTP 500; the persisted job record remains with `status = Pending`
- The API publishes to the `routing-jobs` queue as defined by ADR-003; no other queue is used for job dispatch

---

### FR-6: Submission Response

**Description:**
On successful submission, the API returns HTTP 202 Accepted with a response body containing identifiers and status polling information.

**HTTP status:** `202 Accepted`

**Response body:**

| Field | Type | Description |
|---|---|---|
| `job_id` | UUID string | The job correlation handle; used for all subsequent status polling and cancellation requests |
| `problem_id` | UUID string | The routing problem identifier; used for evidence log retrieval and cross-job comparison |
| `status` | string | Initial state: `"Pending"` |
| `status_url` | string | Absolute URL for polling job status: `GET /v1/jobs/{job_id}` |

**Acceptance Criteria:**
- HTTP 202 is returned only when all of the following have succeeded: structural validation, fast domain validation, scheduler configuration existence check, PostgreSQL persistence, and RabbitMQ publication
- The `job_id` in the response matches the `job_id` in the published queue message
- The `status_url` is a resolvable URL for the job status endpoint (FR-8)
- The response does not include internal execution identifiers (`decision_id`, `execution_seed`, solver run identifiers)

---

### FR-7: Job Identifier Model

**Description:**
FR-7 defines the primary job-lifecycle public identifiers: `job_id` and `problem_id`. These are system-generated UUIDs that form the caller's correlation handles across the asynchronous submission boundary. Additional public identifiers appear in other API endpoints: `scheduler_config_id` (returned by job submission, job status, and scheduler configuration endpoints — FR-6, FR-8, FR-10) and `report_id` (returned by report discovery and retrieval endpoints — FR-12, FR-13); these are defined in their respective requirements. All public identifiers exposed by the API are system-generated UUIDs. Internal execution identifiers are not exposed in any API response.

**`job_id`** — the job execution identifier:
- UUID, generated by the API at submission time (FR-4)
- The caller's primary correlation handle across the asynchronous boundary
- The Evidence Log root key for all artifacts produced by this job (SPEC-006 FR-2)
- The Worker's idempotency key (SPEC-005 FR-14)
- Stable for the lifetime of the job

**`problem_id`** — the routing problem identifier:
- UUID, generated by the API at submission time (FR-4)
- Identifies the routing problem record in PostgreSQL
- Appears on all log events, spans, and evidence records (SPEC-001 FR-1)
- Stable for the lifetime of the problem record
- At MVP scope, each problem is associated with exactly one job (SPEC-001)

**Internal identifiers not exposed in API responses:**
- `decision_id` — Scheduler decision record key (SPEC-003 internal)
- `execution_seed` — derived by the Worker (SPEC-005 FR-7); must not appear in API responses or logs (SPEC-005 Security Considerations)
- Solver run record identifiers
- Quality evaluation record identifiers

**Acceptance Criteria:**
- Every job submission response includes both `job_id` and `problem_id`
- `job_id` and `problem_id` are distinct UUIDs for every submission
- No API response includes `decision_id`, `execution_seed`, or internal solver execution identifiers
- A caller who polls `GET /v1/jobs/{job_id}` receives the same `problem_id` that was returned at submission

---

### FR-8: Job Status Endpoint

**Description:**
The API exposes an endpoint for polling the current state of a job.

**Endpoint:** `GET /v1/jobs/{job_id}`

**Response body:**

The response includes both the job lifecycle state (`status`) and, for completed jobs, the solver execution result (`solver_outcome`). These are distinct fields with distinct semantics: `status` reflects the Worker's execution lifecycle phase (FR-9); `solver_outcome` reflects what the solver returned. A `Completed` status does not indicate solver success — it means the Worker finished its execution lifecycle. Callers should inspect `solver_outcome` to determine the optimization result (FR-9 Caller guidance).

| Field | Type | Present | Description |
|---|---|---|---|
| `job_id` | UUID string | Always | The job identifier |
| `problem_id` | UUID string | Always | The associated routing problem identifier |
| `scheduler_config_id` | UUID string | Always | The scheduler configuration used for this job |
| `status` | string | Always | Current job lifecycle state (FR-9); reflects Worker execution phase, not solver outcome |
| `created_at` | ISO 8601 UTC | Always | Immutable timestamp set by API at job creation (SPEC-006 FR-4.2) |
| `updated_at` | ISO 8601 UTC | Always | Timestamp of the most recent state transition; equals `created_at` for `Pending` jobs |
| `completed_at` | ISO 8601 UTC | When `status = Completed` | Set when status transitions to `Completed`; absent for non-terminal and `Failed` jobs (SPEC-006 FR-4.2) |
| `failed_at` | ISO 8601 UTC | When `status = Failed` | Set when status transitions to `Failed`; absent for non-terminal and `Completed` jobs (SPEC-006 FR-4.2) |
| `solver_outcome` | string | When `status = Completed` | The solver outcome code from the solver run record. Always present for `Completed` jobs. Absent for `Failed` jobs. |
| `failure_reason` | string | When `status = Failed` | The Worker failure classification; absent for `Completed` jobs |
| `report_available` | boolean | Always | Whether a report metadata record exists for this job |

**HTTP status codes:**
- `200 OK`: job found, status returned
- `404 Not Found`: no job with the given `job_id` exists

**Acceptance Criteria:**
- Status reflects the current PostgreSQL job record state at the time of the request
- `created_at` is always present and matches the timestamp set at job creation; it never changes across polls
- `updated_at` is always present; for a `Pending` job it equals `created_at`
- A `Completed` job always includes `completed_at` and never includes `failed_at`
- A `Failed` job always includes `failed_at` and never includes `completed_at`
- Non-terminal jobs (`Pending`, `Processing`) include neither `completed_at` nor `failed_at`
- A `Completed` job always includes `solver_outcome`; the field is absent only for `Failed` jobs
- A `Failed` job always includes `failure_reason` derived from the failure record (SPEC-006 FR-8)
- `report_available = true` only when a report metadata record (SPEC-006 FR-9) exists for the job
- The API does not cache job status; every polling request reads from PostgreSQL

---

### FR-9: Job Status Model

**Description:**
The externally observable job state model is derived directly from the SPEC-005 Worker execution lifecycle. The API does not introduce additional states.

| State | Caller-visible meaning | Terminal? |
|---|---|---|
| `Pending` | Job is in the routing-jobs queue; Worker has not yet consumed it | No |
| `Processing` | Worker has consumed the message and execution is in progress | No |
| `Completed` | Worker completed its execution lifecycle and produced an evidence record. This state does not indicate solver success; solver outcomes of `Succeeded`, `Infeasible`, `Timeout`, `Cancelled`, `Failed`, and `ContractViolation` all produce a `Completed` job. | Yes |
| `Failed` | Worker could not complete its orchestration responsibilities. No complete evidence record was produced. | Yes |

**State visibility constraint:** The API reads job state from the PostgreSQL job record written by the Worker (SPEC-005 FR-2). The API does not infer or invent state transitions. The `Pending` state reflects the job record's PostgreSQL value only — it does not confirm queue presence. A job whose queue publication failed (FR-5, RabbitMQ Unavailable failure mode) also shows `Pending` in the status response; the HTTP 500 returned at submission time is the only caller-visible signal distinguishing a publication-failed job from one awaiting execution. The two conditions produce identical status responses.

**Caller guidance:**
Callers must not assume that `Completed` means the solver found a valid solution. Callers should inspect `solver_outcome` to determine the optimization result and consult the evidence report for full decision context. The complete set of possible `solver_outcome` values for `Completed` jobs is: `Succeeded`, `Infeasible`, `Timeout`, `Cancelled`, `Failed`, and `ContractViolation`. `ContractViolation` indicates the solver returned a structurally invalid response that failed post-receipt validation (SPEC-005 FR-11).

**Acceptance Criteria:**
- The API exposes exactly the four states defined above and no others
- A `Completed` status is returned for any job where the Worker set the terminal `Completed` state in PostgreSQL, regardless of solver outcome
- A `Failed` status is returned for any job where the Worker set the terminal `Failed` state in PostgreSQL
- The API never transitions a job from one state to another; all state transitions are owned by the Worker (SPEC-005)

---

### FR-10: Scheduler Configuration Management

**Description:**
The API provides endpoints for creating and retrieving scheduler configurations. A scheduler configuration specifies the optimization objective mode and its parameters (SPEC-003 FR-13).

**Endpoints:**

`POST /v1/scheduler-configs` — Create a new scheduler configuration.

`GET /v1/scheduler-configs` — List all stored scheduler configurations.

`GET /v1/scheduler-configs/{scheduler_config_id}` — Retrieve a specific scheduler configuration.

**POST /v1/scheduler-configs request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `objective_mode` | string | Yes | One of the seven modes defined in SPEC-003 FR-6 |
| `mode_parameters` | object | Conditional | Required for parameterized modes (`Balanced`, `DeadlineAware`, `BudgetCapped`). May be omitted or `{}` for non-parameterized modes. `null` is not accepted. See SPEC-003 FR-6 Parameter Schemas |

Valid `objective_mode` values: `CheapestValid`, `FastestValid`, `Balanced`, `BestQuality`, `DeadlineAware`, `BudgetCapped`, `Experimental`.

Required `mode_parameters` by mode:

| Mode | Required parameters |
|---|---|
| `CheapestValid` | None |
| `FastestValid` | None |
| `Balanced` | `cost_weight` (float ∈ [0.0, 1.0]), `latency_weight` (float ∈ [0.0, 1.0]), `quality_weight` (float ∈ [0.0, 1.0]); sum must equal 1.0 within tolerance |
| `BestQuality` | None |
| `DeadlineAware` | `deadline_seconds` (positive number) |
| `BudgetCapped` | `budget_limit` (positive number) |
| `Experimental` | None |

**`mode_parameters` for non-parameterized modes** (`CheapestValid`, `FastestValid`, `BestQuality`, `Experimental`):

| Supplied value | Behavior |
|---|---|
| Omitted entirely | Accepted; treated as equivalent to `{}` |
| Empty object `{}` | Accepted |
| `null` | Structurally invalid; rejected with HTTP 400 |
| Non-empty object with unrecognized fields | Accepted; unrecognized fields are silently ignored |

`mode_parameters = null` is invalid regardless of mode. The `null` case is a structural error distinct from omission; callers must either omit the field or supply an object.

**POST response:**
- `201 Created` on success
- Response body: `{scheduler_config_id: UUID, objective_mode: string, mode_parameters: object, created_at: ISO 8601 UTC}`

**Validation for POST:**
- `objective_mode` must be one of the seven recognized values; unrecognized values → HTTP 400
- `mode_parameters = null` is structurally invalid for all modes → HTTP 400
- `Balanced` mode: all three weight fields must be present and their sum must equal 1.0 within floating-point tolerance (SPEC-003 FR-13); any absent weight → HTTP 400
- `DeadlineAware` mode: `deadline_seconds` must be present and positive; absent or non-positive → HTTP 400
- `BudgetCapped` mode: `budget_limit` must be present and positive; absent or non-positive → HTTP 400
- Non-parameterized modes (`CheapestValid`, `FastestValid`, `BestQuality`, `Experimental`): `mode_parameters` may be omitted, `{}`, or a non-empty object; unrecognized fields are ignored

**GET /v1/scheduler-configs response:**
- `200 OK`
- Response body: array of configuration objects `[{scheduler_config_id, objective_mode, mode_parameters, created_at}]`
- Includes the default configuration (SPEC-003 FR-14) as the first entry

**GET /v1/scheduler-configs/{id} response:**
- `200 OK` with configuration object
- `404 Not Found` when no configuration with that ID exists

**Default configuration:**
The default scheduler configuration (SPEC-003 FR-14: `Balanced` mode with equal weights) is seeded at API startup and is always present. Its `scheduler_config_id` is stable across restarts. Callers may reference it explicitly or omit `scheduler_config_id` from job submissions to use it implicitly.

**Acceptance Criteria:**
- A created configuration is immediately retrievable via `GET /v1/scheduler-configs/{id}`
- The default configuration is always present and retrievable
- An unrecognized `objective_mode` is rejected with HTTP 400 before any persistence occurs
- A `Balanced` configuration with any weight absent is rejected with HTTP 400
- `mode_parameters = null` is rejected with HTTP 400 for all modes
- A non-parameterized mode (`CheapestValid`, `FastestValid`, `BestQuality`, `Experimental`) with `mode_parameters` omitted is accepted
- A non-parameterized mode with `mode_parameters = {}` is accepted
- A non-parameterized mode with an unrecognized field in `mode_parameters` is accepted; the unrecognized field is ignored
- A `scheduler_config_id` in a job submission that references a configuration created via this endpoint is accepted
- Configurations are not deleted or modified after creation at MVP scope (OQ-4)

---

### FR-11: Cancellation Request Intake

**Description:**
The API accepts a cancellation request for an in-progress job. The request is an intake operation: the API records the cancellation intent and acknowledges it. The API does not guarantee that the job will be cancelled.

**Endpoint:** `POST /v1/jobs/{job_id}/cancel`

**Request body:** Empty or absent.

**Response:**
- `202 Accepted`: cancellation intent recorded; job execution may or may not be cancelled depending on its current state and Worker behavior
- `404 Not Found`: no job with the given `job_id` exists
- `409 Conflict`: the job is already in a terminal state (`Completed` or `Failed`); cancellation is not applicable

**202 Accepted response body:**

```json
{
  "job_id": "UUID string",
  "status": "Pending | Processing",
  "cancellation_requested": true
}
```

| Field | Type | Description |
|---|---|---|
| `job_id` | UUID string | The job for which cancellation was recorded |
| `status` | string | The job's lifecycle state at the moment the cancellation request was accepted (`Pending` or `Processing`) |
| `cancellation_requested` | boolean | Always `true` in a 202 response; confirms the cancellation intent was successfully written to the job record |

The `202 Accepted` response reflects point-in-time state. It does not guarantee that execution has stopped or will stop; it confirms only that the cancellation intent was recorded. Actual cancellation behavior is owned by the Worker (SPEC-005 FR-12, ODR-5): a `Pending` job may still execute before the Worker observes the flag; a `Processing` job may complete before any in-execution cancellation path is evaluated.

**Cancellation semantics (ODR-5):**

The API's cancellation interface is the external intake for the SPEC-005 FR-12 cancellation signal. On accepting a cancellation request for a non-terminal job, the API writes `cancellation_requested = true` to the PostgreSQL job record. This flag is the delivery mechanism: the Worker reads it at the pre-execution check point (after persisting the Scheduler decision record, before dispatching the SolverRequest). If the flag is set at that point, the Worker terminates the execution lifecycle without invoking a solver and records a `Cancelled` outcome (ODR-5; SPEC-005 FR-12).

**Terminal state check:**
- The API checks the job's current state before acknowledging a cancellation request.
- If the job is `Completed` or `Failed`, the API returns `409 Conflict` with a message indicating the job is already terminal.
- If the job is `Pending` or `Processing`, the API writes `cancellation_requested = true` and returns `202 Accepted`.

**Cancellation of Pending jobs (ODR-5):**
A job in `Pending` state has a message in the routing-jobs queue that has not yet been consumed by the Worker. Pending jobs are not withdrawn from the queue (ODR-5). The API writes `cancellation_requested = true` to the job record. When the Worker eventually consumes the message and reaches the pre-execution check (SPEC-005 FR-12), it observes the flag and terminates the lifecycle with a `Cancelled` outcome without invoking a solver. This preserves RabbitMQ ownership boundaries and at-least-once delivery guarantees.

**Acceptance Criteria:**
- A cancellation request for a terminal job returns `409 Conflict` without modifying any state
- A cancellation request for a non-terminal job returns `202 Accepted` and writes `cancellation_requested = true` to the PostgreSQL job record
- The `202 Accepted` response body includes `job_id`, `status` (the job lifecycle state at the moment of acceptance), and `cancellation_requested = true`
- A `202 Accepted` response does not guarantee cancellation; it acknowledges the intake and sets the flag
- The API does not remove messages from the routing-jobs queue as part of cancellation handling (ODR-5)
- All Worker cancellation behavior is defined in SPEC-005 FR-12

---

### FR-12: Report Discovery

**Description:**
The API exposes an endpoint for discovering the report metadata associated with a completed job.

**Endpoint:** `GET /v1/jobs/{job_id}/report`

**Response:**
- `200 OK`: report metadata returned
- `404 Not Found`: no report exists for this job (job not found, job not yet completed, or report generation failed)

**Response body (when 200):**

| Field | Type | Description |
|---|---|---|
| `report_id` | UUID string | Stable identifier for the report artifact (SPEC-006 FR-9) |
| `job_id` | UUID string | The job this report describes |
| `report_format` | string | Report format; `html` for MVP |
| `generated_at` | ISO 8601 UTC | Timestamp of report generation (SPEC-006 FR-9) |
| `report_url` | string | URL for retrieving the report file: `GET /v1/reports/{report_id}` |

**Acceptance Criteria:**
- Report metadata is returned from the SPEC-006 FR-9 report metadata record for the given `job_id`
- `report_url` is a resolvable URL that retrieves the report file via FR-13
- If no report metadata record exists for the job (job in progress, generation failed, or job never completed), the response is `404 Not Found`
- The API does not generate the report; it reads the metadata record written by the Worker (SPEC-005 FR-17)

---

### FR-13: Report Retrieval

**Description:**
The API serves the evidence report file for a given report identifier.

**Endpoint:** `GET /v1/reports/{report_id}`

**Response:**
- `200 OK`: report file served
  - `Content-Type: text/html` for HTML reports
  - Body: the HTML report file content
- `404 Not Found`: no report with this `report_id` exists or the file is absent from the report volume

**File serving:**
The API reads the report file from the report volume shared between the Worker and API containers (architecture.md container topology). The API does not generate the file; it reads and serves a file that the Report Generator wrote. The API locates the report file by looking up the report metadata record (SPEC-006 FR-9) for the given `report_id`; it does not construct file paths directly from the `report_id` or any other caller-supplied value. File naming conventions, storage structure, and file location on the report volume are owned by the Report Generator Specification (pending) — SPEC-008 is a consumer of that contract, not its owner.

**Acceptance Criteria:**
- A `report_id` returned by FR-12 resolves to the corresponding report file via this endpoint
- A `report_id` that references no file on the report volume returns `404 Not Found`
- The response body is the complete HTML report file
- The API does not transform or reprocess the report file before serving

---

### FR-14: Error Response Model

**Description:**
All API error responses use a consistent structured JSON format. This model applies to all endpoints and all error conditions.

**Error response schema:**

```json
{
  "error_code": "string",
  "message": "string",
  "request_id": "UUID string",
  "field_errors": [
    {
      "field": "string",
      "message": "string"
    }
  ]
}
```

| Field | Present | Description |
|---|---|---|
| `error_code` | Always | Machine-readable error classification code |
| `message` | Always | Human-readable description of the error |
| `request_id` | Always | A UUID generated per request; logged alongside the request span to support correlating error responses with observability data for the same request |
| `field_errors` | When validation fails | One entry per invalid field; empty array for non-validation errors |

**Error codes:**

| HTTP Status | `error_code` | Condition |
|---|---|---|
| 400 | `VALIDATION_ERROR` | One or more structural or domain validation failures (FR-3) |
| 400 | `INVALID_SCHEDULER_CONFIG` | Scheduler configuration body is structurally invalid or has missing required parameters (FR-10) |
| 400 | `SCHEDULER_CONFIG_NOT_FOUND` | `scheduler_config_id` in job submission references no stored configuration |
| 404 | `JOB_NOT_FOUND` | No job with the given `job_id` |
| 404 | `REPORT_NOT_FOUND` | No report for the given job or report_id |
| 404 | `SCHEDULER_CONFIG_NOT_FOUND` | No configuration with the given `scheduler_config_id` (GET endpoint) |
| 409 | `JOB_ALREADY_TERMINAL` | Cancellation requested for a terminal job |
| 500 | `INTERNAL_ERROR` | Unexpected server error (PostgreSQL unavailable, publication failure, etc.) |

**Acceptance Criteria:**
- Every non-2xx response body conforms to this schema
- `field_errors` is always present in 400 responses, even when empty
- `request_id` is present on every error response and correlates to the `job.submit` or relevant request span
- Error responses do not include stack traces, internal component names, or connection strings
- The `message` field is safe to return to external callers; it does not expose internal state

---

### FR-15: API Versioning

**Description:**
The API uses URL path versioning. All endpoints defined in this specification are prefixed with `/v1/`. This prefix is the version indicator for the MVP.

**Version prefix:** `/v1/`

**Scope:** All endpoints defined in this specification:
- `POST /v1/jobs`
- `GET /v1/jobs/{job_id}`
- `POST /v1/jobs/{job_id}/cancel`
- `GET /v1/jobs/{job_id}/report`
- `GET /v1/reports/{report_id}`
- `POST /v1/scheduler-configs`
- `GET /v1/scheduler-configs`
- `GET /v1/scheduler-configs/{scheduler_config_id}`

**Version evolution:** A breaking change to any request or response contract requires a version increment (e.g., `/v2/`). A breaking change is defined as: removing a required field, changing a field's type, removing an endpoint, or changing the semantics of an existing field in a non-backward-compatible way. Adding optional response fields is not a breaking change.

The specific version increment strategy and multi-version serving are deferred to implementation planning (OQ-3).

**Acceptance Criteria:**
- All endpoints respond under the `/v1/` prefix
- A request without the `/v1/` prefix is not served by these endpoints
- An unrecognized path under `/v1/` returns HTTP 404

---

### FR-16: Idempotency

**Description:**
The API's idempotency behavior varies by endpoint class.

**Submission (`POST /v1/jobs`):** Not idempotent. Each submission creates a new routing problem and a new job with new UUIDs. Two identical routing problem bodies submitted sequentially produce two distinct jobs with distinct `job_id` and `problem_id` values. This is consistent with SPEC-001: "Each submission creates a new problem with a new problem_id. No routing problem is shared between jobs."

**Status polling (`GET /v1/jobs/{job_id}`):** Idempotent. Repeated requests return the current state without side effects.

**Cancellation (`POST /v1/jobs/{job_id}/cancel`):** Conditionally idempotent. Repeated cancellation requests for the same non-terminal job return `202 Accepted` each time; whether subsequent requests have additional effect depends on the Worker's cancellation state. Cancellation requests for a terminal job always return `409 Conflict`.

**Configuration creation (`POST /v1/scheduler-configs`):** Not idempotent. Each request creates a new configuration with a new `scheduler_config_id`, even if the objective mode and parameters are identical to an existing configuration.

**Configuration retrieval (`GET /v1/scheduler-configs`, `GET /v1/scheduler-configs/{id}`):** Idempotent.

**Report retrieval (`GET /v1/jobs/{job_id}/report`, `GET /v1/reports/{report_id}`):** Idempotent.

**Acceptance Criteria:**
- Two identical `POST /v1/jobs` requests produce two distinct `job_id` values and two distinct `problem_id` values
- `GET` requests produce no side effects on any persistent state
- A `POST /v1/jobs/{job_id}/cancel` on an already-terminal job returns `409 Conflict`, not `202 Accepted`

---

### FR-17: Observability

**Description:**
The API is responsible for emitting the required OpenTelemetry spans for the stages of the request lifecycle it owns. The required `job.submit` span is defined in architecture.md and ADR-006.

**Span ownership:**

| Span | Owner | When emitted |
|---|---|---|
| `job.submit` | API | Covers the full job submission lifecycle: from request receipt through HTTP 202 response. Required by architecture.md. |
| `api.status_poll` | API | Covers each `GET /v1/jobs/{job_id}` request |
| `api.cancel_request` | API | Covers each `POST /v1/jobs/{job_id}/cancel` request |
| `api.config_create` | API | Covers each `POST /v1/scheduler-configs` request |
| `api.report_serve` | API | Covers each `GET /v1/reports/{report_id}` request |

**`job.submit` span attributes (required):**

| Attribute | Description |
|---|---|
| `job_id` | The generated job identifier |
| `problem_id` | The generated problem identifier |
| `scheduler_config_id` | The resolved scheduler configuration identifier |
| `stop_count` | Number of stops in the submitted routing problem |
| `vehicle_count` | Vehicle count |
| `validation_passed` | Boolean: whether validation succeeded |
| `outcome` | `Success` (202 returned) or `Failure` (4xx/5xx returned) |

**`job.submit` span status:**
- `OK` when HTTP 202 is returned
- `Error` when HTTP 400 or 500 is returned

**`api.status_poll` span attributes:**

| Attribute | Description |
|---|---|
| `job_id` | The polled job identifier |
| `job_status` | The current status value returned |
| `outcome` | `Success` (200) or `NotFound` (404) |

**Structured log events:**

| Event name | Endpoint | Required fields |
|---|---|---|
| `api.job.submitted` | POST /v1/jobs | `job_id`, `problem_id`, `scheduler_config_id`, `stop_count`, `vehicle_count` |
| `api.job.validation_failed` | POST /v1/jobs | `problem_id` (not yet assigned), `failure_count`, `failure_codes` |
| `api.job.publish_failed` | POST /v1/jobs | `job_id`, `problem_id`, `error_type` |
| `api.job.cancel_requested` | POST /v1/jobs/{id}/cancel | `job_id`, `current_status` |
| `api.report.served` | GET /v1/reports/{id} | `report_id`, `job_id` |

**Log safety:** Structured log events must not include routing problem raw data (geographic coordinate arrays, full stop lists). Job identifiers, stop counts, and outcome codes are safe to log. Consistent with SPEC-005 Security Considerations and SPEC-001 Security Considerations.

**Trace context propagation:**
The API propagates the `job.submit` span context to the Worker's `job.consume` span via W3C TraceContext (`traceparent`, optionally `tracestate`) injected into the AMQP message `application_headers` at queue publication time (FR-5). This is the authoritative architectural mechanism for cross-process trace continuity between the API and Worker (ADR-011). The API owns trace context injection at publication; Worker extraction and trace relationship establishment are owned by SPEC-005 FR-3 and FR-19. FR-17 does not redefine ADR-011; ADR-011 is the authoritative decision governing cross-process trace context propagation for Project DAEDALUS.

**Prometheus metrics:**

| Metric name | Type | Labels | Description |
|---|---|---|---|
| `daedalus_api_jobs_submitted_total` | Counter | — | Total job submissions receiving 202 Accepted |
| `daedalus_api_submissions_rejected_total` | Counter | `rejection_reason` (`validation_error`, `config_not_found`) | Total submissions rejected at validation |
| `daedalus_api_job_submission_duration_seconds` | Histogram | `outcome` (`success`, `validation_failure`, `server_error`) | End-to-end submission request duration |
| `daedalus_api_status_poll_total` | Counter | `job_status` (`Pending`, `Processing`, `Completed`, `Failed`, `not_found`) | Status poll outcomes |
| `daedalus_api_cancellations_total` | Counter | `outcome` (`accepted`, `already_terminal`, `not_found`) | Cancellation request outcomes |

**Acceptance Criteria:**
- `job.submit` is emitted on every job submission attempt, successful or not
- All required `job.submit` attributes are present on every emission
- `job.submit` span status is `Error` on any 4xx or 5xx response
- The `job.submit` span context is injected as W3C TraceContext in AMQP message `application_headers` on every successful publication (FR-5)
- Structured log events do not include routing problem coordinate data

---

# Non-Requirements

- The API does not perform solver execution, backend selection, workload feature computation, or quality evaluation (Core and Scheduler responsibilities)
- The API does not generate HTML evidence reports (Report Generator responsibility)
- The API does not write Evidence Log artifact records — job status record creation is the API's responsibility; solver run records, quality evaluation records, failure records, and report metadata records are Worker responsibilities (SPEC-005, SPEC-006)
- The API does not define the Evidence Log persistence schema (SPEC-006 responsibility)
- The API does not define Worker execution behavior (SPEC-005 responsibility)
- The API does not define Worker internal cancellation mechanics beyond writing the `cancellation_requested` flag (Worker implementation planning concern)
- The API does not define report generation behavior (Report Generator Specification, pending)
- The API does not define report file naming conventions, report storage structure, or report volume layout — these are owned by the Report Generator Specification (pending)
- The API does not provide report retrieval compatibility guarantees independently; retrieval compatibility is contingent on the Report Generator Specification maintaining a stable file location captured in the report metadata record (SPEC-006 FR-9)
- The API does not support multi-tenant isolation, authentication, or authorization at MVP scope (deferred per README)
- The API does not expose internal execution identifiers (`decision_id`, `execution_seed`, solver run identifiers)
- The API does not support routing problem versioning, amendment, or deletion (SPEC-001)
- The API does not support scheduler configuration updates or deletion at MVP scope (OQ-4)
- The API does not implement pagination for list endpoints at MVP scope (OQ-5)
- The API does not expose solver backend registration or capability profile management
- The API does not support bulk job submission
- The API does not support webhook or push-based job completion notifications; callers poll for status

---

# Assumptions

1. PostgreSQL is available and reachable before the API processes its first request. A PostgreSQL unavailability is treated as a transient infrastructure failure.
2. RabbitMQ is available and reachable before the API publishes its first job message. A RabbitMQ unavailability during publication is an infrastructure failure.
3. The default scheduler configuration (SPEC-003 FR-14: `Balanced` mode, equal weights) is seeded into PostgreSQL at API startup before any requests are processed. The API does not lazily create it.
4. The report volume defined in the Docker Compose container topology (architecture.md) is mounted and readable by the API container. The API does not write to the report volume.
5. The routing problem records, job records, and report metadata records written to PostgreSQL by the Worker are readable by the API. Both the API and Worker connect to the same PostgreSQL instance.
6. The `average_vehicle_speed_kmh` field belongs to the routing problem domain (ODR-1). SPEC-001 will be revised to add this field. Until that revision is accepted, this field is specified here as a required submission field and is persisted with the routing problem record.
7. Routing problems in the MVP are primarily synthetic (SPEC-002). Real-world coordinate submissions are architecturally supported but are not explicitly tested.
8. No authentication or authorization is required at MVP scope. No authentication layer exists at MVP scope — not within the API, and not as an external proxy or gateway sitting in front of it. All API endpoints are accessible without credentials. Multi-tenant security is deferred to post-MVP implementation (README). The MVP is assumed to run in a trusted local development environment.

---

# Constraints

1. The API must be implemented in C# with ASP.NET Core (ADR-002).
2. The API must persist routing problems and job records to PostgreSQL before publishing to RabbitMQ (SPEC-001 FR-12; ADR-004).
3. The API must publish to the `routing-jobs` queue in RabbitMQ (ADR-003).
4. The API must implement the dual-validation model defined in ADR-009: fast domain validation for caller feedback, deferring authoritative domain validation to Core.
5. Validation must run before any PostgreSQL write. An invalid submission must not produce any database record.
6. The `job.submit` span is required by architecture.md and must be emitted on every submission attempt (ADR-006).
7. Internal execution identifiers must not be included in any API response.
8. The API must not block on job execution. HTTP 202 Accepted must be returned after queue publication, not after job completion.
9. Error responses must conform to the FR-14 schema and must not expose internal implementation details.

---

# Inputs

**Caller requests:**

| Input | Source | Format | Notes |
|---|---|---|---|
| Job submission request | HTTP caller | JSON body per FR-2 | Untrusted; validated before persistence |
| Scheduler configuration creation request | HTTP caller | JSON body per FR-10 | Untrusted; validated before persistence |
| Cancellation request | HTTP caller | Empty body; `job_id` in path | `POST /v1/jobs/{job_id}/cancel` |
| Status poll request | HTTP caller | `job_id` in path | `GET /v1/jobs/{job_id}`; read-only |
| Report discovery request | HTTP caller | `job_id` in path | `GET /v1/jobs/{job_id}/report`; read-only |
| Scheduler configuration retrieval request | HTTP caller | `scheduler_config_id` in path (single) or no body (list) | `GET /v1/scheduler-configs/{id}` and `GET /v1/scheduler-configs`; read-only |
| Report retrieval request | HTTP caller | `report_id` in path | `GET /v1/reports/{report_id}`; read-only |

**Externally-owned data artifacts read at runtime:**

| Input | Source | Authoritative Owner | Format | Used By | Notes |
|---|---|---|---|---|---|
| Job record | PostgreSQL | SPEC-006 FR-4.2 (schema); ODR-6 | `job_id`, `problem_id`, `scheduler_config_id`, `status`, `cancellation_requested`, `created_at`, `updated_at`, `completed_at`, `failed_at` | FR-3 (config validation), FR-8 (status response), FR-11 (terminal state check) | Created by API; `status`, `updated_at`, `completed_at`, `failed_at` updated by Worker (SPEC-005) |
| Solver run record | PostgreSQL | SPEC-006 FR-6.2 | `solver_outcome` and associated fields | FR-8 (status response — `solver_outcome` field for `Completed` jobs) | Written by Worker (SPEC-005); absent for `Failed` and non-terminal jobs |
| Failure record | PostgreSQL | SPEC-006 FR-8 | Failure classification fields including `failure_reason` | FR-8 (status response — `failure_reason` field for `Failed` jobs) | Written by Worker (SPEC-005); absent for `Completed` and non-terminal jobs |
| Report metadata record | PostgreSQL | SPEC-006 FR-9.2 | `report_id`, `job_id`, `report_format`, `generated_at`, `generation_status` | FR-8 (existence check for `report_available`), FR-12 (report discovery response), FR-13 (file path lookup for report serving) | Written by Worker (SPEC-005 FR-17); may be absent if report generation failed or job is not yet terminal |
| Scheduler configuration record | PostgreSQL | SPEC-003 FR-13 | Objective mode and mode parameters | FR-3 (existence check at submission validation), FR-10 (configuration retrieval endpoints) | Written by API at creation time (FR-10) and seeded at startup (SPEC-003 FR-14) |
| Report file | Report volume | Report Generator Specification (pending) | HTML | FR-13 (report retrieval response body) | Written by Report Generator; located via report metadata record lookup — not by direct path construction from caller input |

---

# Outputs

| Output | Consumer | Format | Notes |
|---|---|---|---|
| HTTP 202 submission response | HTTP caller | JSON per FR-6 | Contains `job_id`, `problem_id`, `status_url` |
| HTTP 400 validation response | HTTP caller | JSON per FR-14 | Contains `field_errors` for all failures |
| HTTP 500 infrastructure error | HTTP caller | JSON per FR-14 | Does not expose internal details |
| Job status response | HTTP caller | JSON per FR-8 | Reads current PostgreSQL state |
| Report metadata response | HTTP caller | JSON per FR-12 | |
| Report file | HTTP caller | `text/html` | Served from report volume |
| Cancellation acknowledgement | HTTP caller | JSON per FR-14 error or 202 body | |
| Routing problem record | PostgreSQL | Per SPEC-001 | Persisted before queue publication |
| Job record | PostgreSQL | `job_id`, `problem_id`, `scheduler_config_id`, `status = Pending`, `cancellation_requested = false`, `created_at` (SPEC-006 FR-4.2) | Initial creation; Worker updates status |
| Job message | RabbitMQ `routing-jobs` | `{job_id, problem_id, scheduler_config_id}` | Consumed by Worker (SPEC-005) |
| OTel spans | OpenTelemetry Collector | Traces per FR-17 | `job.submit` and secondary spans |
| Structured log events | Stdout (JSON) | Per ADR-006 | At each lifecycle stage |

---

# Failure Modes

### Structural Validation Failure

**Condition:** Request body does not conform to the JSON schema (missing required field, wrong type, malformed JSON).
**Behavior:** HTTP 400 returned immediately. No database access occurs. `job.submit` span status Error.
**Fallback:** Caller must correct the request and resubmit.

---

### Domain Validation Failure

**Condition:** Request body passes structural validation but fails a fast domain validation rule (SPEC-001 FR-5, FR-8, FR-9, FR-16).
**Behavior:** HTTP 400 with all failing rules collected in `field_errors`. No database access occurs.
**Fallback:** Caller must correct the routing problem and resubmit.

---

### Scheduler Configuration Not Found

**Condition:** Provided `scheduler_config_id` references no stored configuration.
**Behavior:** HTTP 400 with `error_code = SCHEDULER_CONFIG_NOT_FOUND`. No persistence occurs.
**Fallback:** Caller must use a valid `scheduler_config_id` or omit it.

---

### PostgreSQL Unavailable During Persistence

**Condition:** PostgreSQL connection fails during routing problem or job record persistence.
**Behavior:** HTTP 500. No queue message is published. The routing problem and job records may be partially written; the API should attempt to roll back. This is a transient infrastructure failure.
**Fallback:** Caller should retry the submission. The system does not auto-retry.

---

### RabbitMQ Unavailable During Publication

**Condition:** PostgreSQL persistence succeeds but RabbitMQ is unavailable when the API attempts to publish.
**Behavior:** HTTP 500. The job record exists in PostgreSQL with `status = Pending` but no queue message was published. The job will not be executed without operator intervention.
**Fallback:** Operator must detect the stranded job and republish the message, or the caller must resubmit (which creates a new job). This is an infrastructure failure condition requiring operator action.

---

### Job Not Found (Status Poll, Cancellation, Report Discovery)

**Condition:** The provided `job_id` references no job in PostgreSQL.
**Behavior:** HTTP 404.
**Fallback:** Caller must verify the `job_id` is correct.

---

### Report File Missing from Volume

**Condition:** A report metadata record exists for the job but the report file is absent from the report volume.
**Behavior:** HTTP 404 on `GET /v1/reports/{report_id}`.
**Fallback:** Manual investigation required. The report file may have been deleted or the volume may be misconfigured.

---

### Cancellation of Terminal Job

**Condition:** Cancellation is requested for a job already in `Completed` or `Failed` state.
**Behavior:** HTTP 409 Conflict. No state is modified.
**Fallback:** No action required; the job is already complete.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer | Indirect — API implements fast domain validation rules derived from SPEC-001; these rules must be kept consistent with Core's authoritative C++ validation (ADR-009) |
| API Layer | Yes — SPEC-008 is the primary definition of the API layer behavior |
| Persistence | Yes — API writes routing problem records and job records to PostgreSQL; reads job status and report metadata from PostgreSQL |
| Solver Runtime | None — API has no direct interaction with solver backends |
| Observability | Yes — API owns `job.submit` span and additional request spans; introduces API-layer Prometheus metrics |
| Security | Yes — API is the external trust boundary; all caller input is untrusted; authentication/authorization deferred |
| Configuration | Yes — API creates and retrieves scheduler configuration records; seeds default configuration at startup |
| Deployment | Yes — API container requires PostgreSQL access, RabbitMQ access, and report volume mount (read-only) |

**SPEC-001 FR-12:** "Persistence and job creation are atomic. If persistence fails, the job is not published." Satisfied by FR-4 atomicity requirement.

**SPEC-001 FR-13:** Default scheduler configuration is satisfied by SPEC-003 FR-14. The API validates `scheduler_config_id` existence at submission time (ADR-009).

**ADR-009 dual validation:** The API implements the fast-feedback layer. Core implements the authoritative layer. The API's fast domain validation rules are maintained in C# independently of Core's C++ implementation. Divergence is a detectable fault condition.

---

# Testability

1. **Unit: Structural validation — missing required field** — A submission body with `routing_problem` absent returns HTTP 400 with `error_code = VALIDATION_ERROR` and a `field_errors` entry identifying `routing_problem` before any database call.

2. **Unit: Structural validation — type mismatch** — A submission body where `vehicle_count` is a string returns HTTP 400 identifying `routing_problem.vehicle_count`.

3. **Unit: Domain validation — capacity infeasibility** — A submission where total stop demand exceeds total fleet capacity returns HTTP 400 with a capacity infeasibility message.

4. **Unit: Domain validation — time window inversion** — A submission where a stop has `time_window_open ≥ time_window_close` returns HTTP 400 identifying the offending stop.

5. **Unit: Domain validation — out-of-range coordinate** — A submission with a depot latitude of 95.0 returns HTTP 400 identifying `depot.latitude`.

6. **Unit: Domain validation — multiple failures collected** — A submission with three separate domain violations returns HTTP 400 with three entries in `field_errors`, not just the first.

7. **Unit: Scheduler config validation — unknown mode** — `POST /v1/scheduler-configs` with `objective_mode = "Unknown"` returns HTTP 400 with `error_code = INVALID_SCHEDULER_CONFIG`.

8. **Unit: Scheduler config validation — Balanced missing weight** — `POST /v1/scheduler-configs` with `objective_mode = Balanced` and only two weight fields returns HTTP 400.

9. **Unit: Scheduler config not found at submission** — A submission with a `scheduler_config_id` referencing no stored configuration returns HTTP 400 with `error_code = SCHEDULER_CONFIG_NOT_FOUND`.

10. **Unit: Job identifier model** — Two sequential valid submissions produce responses with distinct `job_id` and distinct `problem_id` values.

11. **Unit: No internal identifiers** — The submission response (HTTP 202) does not include `decision_id`, `execution_seed`, or any solver run identifier.

12. **Integration: Happy path — submission to status** — A valid submission returns HTTP 202 with `job_id`. A subsequent `GET /v1/jobs/{job_id}` returns HTTP 200 with `status = Pending` or later states.

13. **Integration: Status transitions** — Given a job that advances through `Pending → Processing → Completed`, polling at each stage returns the correct status value.

14. **Integration: Completed job with solver_outcome** — A job that reaches `Completed` has `solver_outcome` present in the status response.

15. **Integration: Report discovery** — After report generation, `GET /v1/jobs/{job_id}/report` returns HTTP 200 with `report_url` populated.

16. **Integration: Report retrieval** — `GET /v1/reports/{report_id}` using the `report_url` from FR-12 returns HTTP 200 with `Content-Type: text/html` and a non-empty body.

17. **Integration: Cancellation of terminal job** — After a job reaches `Completed`, `POST /v1/jobs/{job_id}/cancel` returns `409 Conflict`.

18. **Integration: Default configuration** — A submission without `scheduler_config_id` uses the default configuration. The status response includes the default configuration's `scheduler_config_id`.

19. **Integration: OTel span emission** — A valid submission produces a `job.submit` span with all required attributes in the OpenTelemetry Collector.

20. **Integration: OTel span — validation failure** — A submission failing domain validation produces a `job.submit` span with `outcome = Failure` and span status Error.

21. **Unit: PostgreSQL persistence before publication** — Given a test harness that accepts a PostgreSQL write and then fails RabbitMQ publication, the routing problem and job records exist in PostgreSQL with no queue message published, and the API returns HTTP 500.

---

# Observability Requirements

Operational questions the API spans, logs, and metrics must answer:

1. How many job submissions are received per unit time?
2. What fraction of submissions fail validation, and for which reasons?
3. What is the end-to-end latency of a job submission from request to 202 response?
4. For a given `job_id`, what is the current job status?
5. How often does queue publication fail after successful PostgreSQL persistence?
6. How often are cancellation requests submitted for jobs that are already terminal?
7. What is the distribution of solver outcomes across Completed jobs visible to the API?
8. Are report files being served successfully?

These are answered by:
- The `job.submit` span (submission latency, validation outcome, scheduler config)
- The `api.status_poll` span (polling frequency, status distribution)
- Structured log events at each API stage
- Prometheus metrics: submission totals, rejection totals, poll outcomes, submission duration histogram

---

# Security Considerations

**External trust boundary:**
The API is the sole external trust boundary for Project DAEDALUS. All caller-provided input — request bodies, path parameters, query parameters — must be treated as untrusted. Validation must run before any database access or queue publication.

**Input validation:**
All submission inputs are validated before persistence (FR-3). Malformed JSON, type violations, and domain rule violations are rejected before any side effects occur. The API must not panic or produce undefined behavior on malformed input.

**No authentication or authorization at MVP scope:**
No authentication layer exists at MVP scope — not within the API, and not as an external proxy or gateway. All API endpoints are accessible without credentials. This is not a case of auth being delegated externally; it is simply absent. Multi-tenant security is deferred to post-MVP implementation (README). This is an accepted risk for a local development and demonstration environment. Authentication and authorization must be implemented before any production deployment or exposure to untrusted networks.

**Internal identifier isolation:**
`execution_seed` must not appear in API responses, logs, or traces (SPEC-005 Security Considerations). `decision_id` and solver run identifiers are internal and must not be included in API responses (FR-7).

**Log safety:**
Structured log events must not include routing problem raw data (geographic coordinate arrays, full stop lists). Job identifiers, stop counts, and outcome codes are safe to log. Consistent with SPEC-001 Security Considerations.

**Database credentials:**
PostgreSQL and RabbitMQ credentials must be injected via environment variables. Credentials must not be committed to source control or logged. Connection strings must not appear in error responses.

**Report serving:**
The report volume is shared between the API (read) and Worker (write) containers. The API must serve only files within the designated report volume directory. Path traversal vulnerabilities in the `report_id` parameter must be prevented: `report_id` is a UUID, and the file is located by UUID lookup against the report metadata record, not by direct filesystem path construction from the request parameter.

---

# Performance Considerations

**Submission latency:**
The `job.submit` span covers request receipt through HTTP 202. This duration includes JSON parsing, structural validation, domain validation, two PostgreSQL writes (problem record + job record), and one RabbitMQ publication. Submission should complete in well under 1 second under normal infrastructure conditions. This must be measured during implementation.

**Status polling:**
`GET /v1/jobs/{job_id}` is a single PostgreSQL read. Latency is bounded by PostgreSQL query performance. The API does not cache status; every poll hits the database.

**Report serving:**
HTML report files are served from the report volume. File size is bounded by the report content. The API does not buffer or transform the file. Latency is bounded by disk read performance and HTTP transfer time.

**Connection pooling:**
PostgreSQL and RabbitMQ connection pool sizes are implementation planning concerns. Defaults are assumed sufficient for the MVP single-instance workload.

---

# Documentation Updates Required

- **docs/architecture.md**: The API Responsibilities section accurately describes the API's role at a high level. SPEC-008 is the authoritative binding contract. No architecture.md changes are required.
- **SPEC-001**: Must be revised to add `average_vehicle_speed_kmh` as a required routing problem field per ODR-1. Until that revision is accepted, SPEC-008 FR-2 and Assumption 6 define the API's treatment of this field.
- **SPEC-003**: The scheduler configuration endpoint contract defined in SPEC-008 FR-10 is the API's implementation of the configuration persistence model implied by SPEC-003 FR-13. No SPEC-003 changes are required.
- **SPEC-005 FR-12**: The cancellation delivery mechanism is resolved (ODR-5). SPEC-008 FR-11 (API writes `cancellation_requested`) and SPEC-005 FR-12 (Worker reads flag at pre-execution check) are consistent. No further cross-spec updates required for cancellation.
- **SPEC-006 FR-4 (Job Record Persistence Schema)**: Resolved by ODR-6. SPEC-006 FR-4.2 has been updated to add `scheduler_config_id` and `cancellation_requested`, and SPEC-008 FR-4 and FR-8 have been aligned to SPEC-006's authoritative field names (`created_at`, `updated_at`, `completed_at`, `failed_at`). Engineering Review findings B-2 and B-3 are resolved. SPEC-006 is the authoritative schema owner per ODR-6; future job record field additions require a SPEC-006 revision.
- **CLI Specification (pending)**: The Daedalus CLI (README.md) submits jobs through the API. The CLI specification must reference SPEC-008 for the HTTP contract it wraps.
- **Report Generator Specification (pending)**: SPEC-008 FR-13 serves files written by the Report Generator to the shared report volume. The API is a consumer of the report storage contract; the Report Generator Specification is the owner. The Report Generator Specification must define the following before SPEC-008 FR-13 can be fully implemented: (1) the file naming convention and storage structure on the report volume; and (2) the mechanism by which the API locates the physical file — specifically, whether the report metadata record (SPEC-006 FR-9) stores an explicit `file_path` field, or whether `report_id` is sufficient to derive the file location via a naming convention owned by the Report Generator Specification.

---

# Open Questions

### OQ-1: Cancellation Delivery Mechanism — RESOLVED (ODR-5)

**Resolution:** The API writes `cancellation_requested = true` to the PostgreSQL job record when a cancellation request is accepted (FR-11). The Worker reads this flag at the pre-execution check point (after Scheduler decision persistence, before SolverRequest dispatch) per SPEC-005 FR-12. This is the sole delivery mechanism for external cancellation. Incorporated in FR-11 (API sets flag) and SPEC-005 FR-12 (Worker reads flag).

**No further action required.**

---

### OQ-2: Pending Job Cancellation — RESOLVED (ODR-5)

**Resolution:** Soft-cancel model. Pending jobs remain in the routing-jobs queue; the API does not manipulate queue messages. The API writes `cancellation_requested = true` to the PostgreSQL job record. The Worker observes the flag at the pre-execution check (after Scheduler decision persistence, before SolverRequest dispatch) and terminates the lifecycle with a `Cancelled` outcome without invoking a solver (SPEC-005 FR-12, ODR-5). This preserves RabbitMQ ownership boundaries and at-least-once delivery guarantees.

**No further action required.**

---

### OQ-3: API Versioning Strategy

**Question:** If a breaking API change is required, how are multiple API versions served concurrently?

**Why it matters:** SPEC-008 FR-15 defines URL path versioning (`/v1/`) but does not define how a `/v2/` would coexist with `/v1/`. Options include: routing both versions through the same API process with version-aware controllers; deploying separate API processes; or using a gateway layer for version routing.

**Owner:** Implementation planning — to be decided when API versioning becomes a concrete requirement (i.e., before a breaking change is needed). If the chosen strategy has deployment cost implications (e.g., long-running parallel deployments, gateway infrastructure), Project Owner input is required before committing.

**Blocking:** Not blocking for MVP. Not blocking for Draft or Proposed status.

---

### OQ-4: Scheduler Configuration Mutability

**Question:** Should stored scheduler configurations be updatable or deletable after creation?

**Why it matters:** A configuration may need to be corrected (wrong weights, wrong objective mode). However, if a configuration is in use by in-flight jobs, updating or deleting it mid-execution could produce inconsistent evidence. Two jobs referencing the same `scheduler_config_id` with different effective configurations would be non-comparable.

**Recommendation:** Read-only after creation at MVP scope. Callers who want a modified configuration create a new one. This preserves evidence log comparability.

**Owner:** Project Owner decision. Escalation path: if the read-only-after-creation recommendation is not accepted, an ODR (Owner Decision Record) is required to define the configuration mutability policy, including the handling of in-flight jobs that reference the modified or deleted configuration.

**Blocking:** Not blocking for Draft or Proposed status. The current FR-10 assumes read-only after creation.

---

### OQ-5: Scheduler Configuration List Pagination — RESOLVED

**Resolution:** No pagination for scheduler configuration list endpoints at MVP scope. The Non-Requirements section of this specification is the authoritative answer: "The API does not implement pagination for list endpoints at MVP scope (OQ-5)." The question is answered within this specification and requires no further action.

**No further action required.**

---

### OQ-6: Trace Context Correlation Between API and Worker — RESOLVED (ADR-011)

**Resolution:** W3C TraceContext (`traceparent`, optionally `tracestate`) propagated in AMQP message `application_headers` at queue publication time. The API injects the active `job.submit` span context into the AMQP message `application_headers` at publication (FR-5); the Worker extracts it from the consumed message and establishes a navigable trace relationship between `job.consume` and the `job.submit` context (SPEC-005 FR-3, FR-19). ADR-011 is the authoritative architectural decision governing cross-process trace context propagation for Project DAEDALUS.

**No further action required.**

---

# Acceptance Checklist

- [x] Problem is clearly defined
- [x] Domain concept is defined
- [x] Responsibility scope and component boundaries are defined (FR-1)
- [x] Job submission request model is defined (FR-2)
- [x] Submission validation behavior is defined (FR-3)
- [x] Job creation and persistence behavior is defined (FR-4)
- [x] Queue publication behavior is defined (FR-5)
- [x] Submission response model is defined (FR-6)
- [x] Job identifier model is defined (FR-7)
- [x] Job status endpoint is defined (FR-8)
- [x] Job status model is defined and derived from SPEC-005 (FR-9)
- [x] Scheduler configuration creation and retrieval is defined (FR-10)
- [x] Cancellation request intake is defined (FR-11)
- [x] Report discovery is defined (FR-12)
- [x] Report retrieval is defined (FR-13)
- [x] Error response model is defined (FR-14)
- [x] Versioning expectations are defined (FR-15)
- [x] Idempotency expectations are defined (FR-16)
- [x] Observability requirements are defined (FR-17)
- [x] Non-requirements are documented
- [x] Assumptions are explicit
- [x] Failure modes are defined
- [x] Observability requirements exist
- [x] Security considerations exist
- [x] Documentation updates are identified
- [x] OQ-1 resolved — cancellation delivery mechanism: PostgreSQL `cancellation_requested` flag written by API, read by Worker at pre-execution check (ODR-5)
- [x] OQ-2 resolved — Pending-state cancellation: soft-cancel model; Pending jobs remain in queue; Worker cancels at pre-execution check (ODR-5)
- [ ] OQ-3 resolved — API versioning strategy (non-blocking)
- [ ] OQ-4 resolved — scheduler configuration mutability (non-blocking; read-only assumed)
- [x] OQ-5 resolved — configuration list pagination: no pagination at MVP scope (Non-Requirements)
- [x] OQ-6 resolved — trace context propagation mechanism: W3C TraceContext in AMQP message `application_headers`; Worker establishes navigable trace relationship (ADR-011)

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-17) are implemented and acceptance criteria pass
- OQ-1 (cancellation delivery mechanism) is resolved: PostgreSQL `cancellation_requested` flag; API writes, Worker reads at pre-execution check (ODR-5) — satisfied
- OQ-2 (Pending-state cancellation) is resolved: soft-cancel model; Pending jobs remain in queue; Worker cancels at pre-execution check (ODR-5) — satisfied
- OQ-6 (trace context propagation) is resolved and incorporated in FR-17
- All test contracts defined in the Testability section pass
- The `job.submit` span is emitted and verifiable in the OpenTelemetry Collector on every submission attempt
- SPEC-001 is revised to include `average_vehicle_speed_kmh` (ODR-1) and SPEC-008 FR-2 is updated to reference SPEC-001 directly
- Engineering review passes
- Architecture review passes
- Specification status is updated to Verified
