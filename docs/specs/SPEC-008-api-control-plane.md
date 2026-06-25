# SPEC-008: API / Control Plane

## Metadata

**Feature ID:** SPEC-008

**Title:** API / Control Plane

**Status:** Accepted

**Author:** Darkhorse286

**Created:** 2026-06-12

**Last Updated:** 2026-06-25

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-002, ADR-003, ADR-004, ADR-006, ADR-009, ADR-011, ADR-012, ADR-013

**Related Specs:** SPEC-001, SPEC-003, SPEC-005, SPEC-006, SPEC-009, SPEC-012, SPEC-016, SPEC-020

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
| **API** (SPEC-008) | Request receipt; structural and fast domain validation; routing problem persistence; job creation; queue publication; job status visibility; scheduler configuration creation and retrieval; cancellation request intake; report metadata and file serving; `job.submit` span emission; error response production; experiment manifest validation and persistence; benchmark manifest validation and persistence; experiment lifecycle state management; trial evidence collection (reads Evidence Log tables, writes to experiment trial records per ADR-012 Decision 5); experiment artifact computation (QualityStatsAggregates, ExperimentSummary); benchmark summary artifact computation | Solver execution; backend selection; workload feature extraction; quality evaluation; report generation; writing to Evidence Log artifact tables (SPEC-006 FR-1.3 write authority is Worker-only); routing problem generation (SPEC-002); Worker execution behavior in experiment context |
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
| `routing_problem` | object | Conditional | A routing problem per SPEC-001 FR-11 JSON serialization contract. Required when `problem_id` is absent. Mutually exclusive with `problem_id`. |
| `problem_id` | UUID string | Conditional | References an existing routing problem record. When provided, `routing_problem` must be absent. Used in experiment context to submit multiple jobs against the same routing problem (ADR-012 Decision 3, SPEC-020 FR-4 instance-sharing invariant). |
| `scheduler_config_id` | UUID string | No | References an existing scheduler configuration (SPEC-003 FR-13). When absent, the default configuration is used (SPEC-003 FR-14). |
| `backend_id` | string | No | Backend targeting directive (ADR-013). When absent, the Scheduler selects by policy (standard path — behavior unchanged). When present, the Scheduler validates eligibility for the specified backend and directs execution to it without policy-based fallback (targeted path). The experiment harness always provides `backend_id` for trial job submissions (ADR-013 Decision 2); standard callers may omit it. The API validates `backend_id` as a non-empty string; no existence check against the Scheduler capability profile registry is performed (registry not persisted in PostgreSQL per SPEC-012 FR-2 scope). |

**Submission mode:**
Exactly one of `routing_problem` or `problem_id` must be present. When neither or both are provided, the request is rejected with HTTP 400. When `problem_id` is provided, the job references an existing routing problem record without creating a new one, enabling the instance-sharing invariant required by experiment cross-solver comparisons (ADR-012 Decision 3, SPEC-012 FR-6). The one-problem-one-job constraint of FR-7 applies only to standard (non-experiment) submissions; in experiment context, multiple jobs may reference the same `problem_id` per ADR-012 Decision 3.

**Backend targeting paths (ADR-013):**

| Path | Condition | Scheduler behavior |
|---|---|---|
| Standard path | `backend_id` absent | Scheduler evaluates all eligible backends, scores candidates, and selects by policy |
| Targeted path | `backend_id` present | Scheduler validates eligibility for the specified backend only; selects it if eligible; records `selection_mode = explicitly_targeted`; no fallback if ineligible |

The targeted path is transparent to the API: the API forwards `backend_id` in the job record and queue message without performing eligibility evaluation or backend selection. The API does not distinguish experiment-context submissions from standard submissions; it does not inspect why a `backend_id` was provided.

**Routing problem fields** (per SPEC-001):

| Field | Type | Required | Constraint |
|---|---|---|---|
| `seed` | uint64 | Yes | Non-negative 64-bit integer (SPEC-001 FR-6) |
| `vehicle_count` | positive integer | Yes | ≥ 1 (SPEC-001 FR-2) |
| `capacity_per_vehicle` | positive integer | Yes | ≥ 1 (SPEC-001 FR-2) |
| `average_vehicle_speed_kmh` | float64 | Yes | > 0.0; required for all routing problem submissions (SPEC-001 FR-17; ODR-1) |
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
- A request with neither `routing_problem` nor `problem_id` is rejected with HTTP 400 before any persistence occurs
- A request with both `routing_problem` and `problem_id` is rejected with HTTP 400
- A `problem_id` that is not a valid UUID format is rejected with HTTP 400
- A `problem_id` that does not reference an existing record in `routing_problems` is rejected with HTTP 400 with `error_code = PROBLEM_NOT_FOUND`
- A request with an unrecognized top-level field is accepted; the field is ignored (permissive structural parsing)
- A `scheduler_config_id` that is not a valid UUID format is rejected with HTTP 400
- When `routing_problem` is provided, the object must satisfy the schema above; any missing required field or type violation is rejected with HTTP 400 identifying the offending field(s)
- When `problem_id` is provided, no routing problem field validation is performed
- A `backend_id` that is an empty string is rejected with HTTP 400; absence of `backend_id` is not a validation failure on any submission path
- A valid `backend_id` (non-empty string) is accepted without any existence check against the Scheduler capability profile registry

---

### FR-3: Submission Request Validation

**Description:**
The API performs two validation layers before persisting the routing problem. Both layers must pass before persistence begins. Validation implements the API's side of the dual-validation policy (ADR-009).

**Layer 1: Structural validation**

JSON schema conformance, field type checking, and required field presence. Applied before domain validation.

Structural validation failures are immediately rejected with HTTP 400 and a structured error identifying each offending field (FR-14).

**Layer 2: Fast domain validation**

Domain validation rules derived from SPEC-001, applied for fast caller feedback (ADR-009). The API implements these rules independently in C#; Core re-validates in C++ as the authoritative validator (ADR-009).

When `problem_id` is provided instead of `routing_problem` (experiment context), fast domain validation is skipped entirely. The routing problem was already validated at its original submission time. Proceed directly to the `problem_id` existence check and scheduler configuration validation.

| Rule | SPEC-001 Reference | Rejection message |
|---|---|---|
| `vehicle_count` ≥ 1 | FR-2 | `vehicle_count must be a positive integer` |
| `capacity_per_vehicle` ≥ 1 | FR-2 | `capacity_per_vehicle must be a positive integer` |
| `average_vehicle_speed_kmh` > 0.0 | FR-17 | `average_vehicle_speed_kmh must be a positive number` |
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

**`problem_id` existence validation (experiment context):**

| Rule | Description |
|---|---|
| `problem_id` exists in PostgreSQL | A provided `problem_id` that references no record in `routing_problems` is rejected: HTTP 400, `error_code = PROBLEM_NOT_FOUND` |

**`backend_id` validation (when present):**

| Rule | Description |
|---|---|
| `backend_id` non-empty string | A provided `backend_id` that is an empty string is rejected: HTTP 400, `backend_id must be a non-empty string` |
| No existence check | `backend_id` is not validated against the Scheduler capability profile registry; capability profiles are not persisted in PostgreSQL (SPEC-012 FR-2 scope). An unregistered `backend_id` is detected at Scheduler execution time, not at submission time (ADR-013 Accepted Risks). |
| Absence not a failure | Absent `backend_id` means standard-path execution; no validation failure is produced |

**Scheduler configuration validation:**

| Rule | Description |
|---|---|
| `scheduler_config_id` exists in PostgreSQL | A provided `scheduler_config_id` that references no stored configuration is rejected: HTTP 400, `scheduler_config_id not found` |
| No validation when absent | Absent `scheduler_config_id` uses the default configuration (SPEC-003 FR-14); no existence check is required |

**Validation ordering (routing_problem path — standard submission):**

1. Structural validation — all failures collected and returned together.
2. Fast domain validation — applied only when structural validation passes. All domain failures collected and returned together in a single response. `backend_id` non-empty check applied here when `backend_id` is present.
3. Scheduler configuration existence check — applied only when domain validation passes.

**Validation ordering (problem_id path — experiment context):**

1. Structural validation — validate `problem_id` is a valid UUID format; validate `routing_problem` is absent. `backend_id` non-empty check applied here when `backend_id` is present.
2. `problem_id` existence check — validate the referenced routing problem exists in PostgreSQL.
3. Scheduler configuration existence check — applied only when step 2 passes.

**Divergence from Core:**

If the API accepts a routing problem that Core subsequently rejects (ADR-009 divergence), the job transitions to `Failed` with a `CoreValidationRejection` failure record (SPEC-005 FR-5). The caller receives HTTP 202 Accepted and discovers the failure by polling job status. This divergence is an observable fault condition requiring investigation (ADR-009).

**Acceptance Criteria:**
- Structural validation failures are returned as HTTP 400 before any database access
- Fast domain validation failures are returned as HTTP 400 before any database access (routing_problem path only)
- All validation failures in a layer are collected and returned in a single response — callers receive all errors, not just the first
- A `scheduler_config_id` that references no stored configuration is rejected with HTTP 400 — the system does not silently substitute the default (SPEC-001 FR-13)
- A submission that passes all validation stages proceeds to FR-4 persistence
- When `problem_id` is provided, fast domain validation is skipped entirely; only `problem_id` existence, `backend_id` non-empty check (when present), and scheduler config validation apply
- A `problem_id` that references no record in `routing_problems` is rejected with HTTP 400 with `error_code = PROBLEM_NOT_FOUND` before any persistence
- A `backend_id` present but empty string is rejected with HTTP 400; `backend_id` absent is not a validation failure
- The API does not validate `backend_id` against the Scheduler capability profile registry
- The API does not re-run Core's authoritative validation in C#; it implements the fast-feedback approximation defined in ADR-009

---

### FR-4: Job Creation and Persistence

**Description:**
After all validation passes, the API creates the job and persists the routing problem to PostgreSQL before publishing to the queue.

**Creation sequence — routing_problem path (standard submission):**

1. Generate `problem_id`: UUID, system-generated (SPEC-001 FR-1).
2. Generate `job_id`: UUID, system-generated. The `job_id` is the caller's correlation handle and the Evidence Log root key (SPEC-006 FR-2).
3. Resolve `scheduler_config_id`: use the provided value, or the default configuration's ID (SPEC-003 FR-14).
4. Persist the routing problem record to PostgreSQL with `problem_id`. All SPEC-001 fields are persisted. Persistence completes before queue publication (SPEC-001 FR-12).
5. Create the job record in PostgreSQL with `job_id`, `problem_id`, `scheduler_config_id`, `backend_id` (nullable — stored when present in the request, NULL when absent), `status = Pending`, `cancellation_requested = false`, and `created_at` timestamp (SPEC-006 FR-4.2).
6. Steps 4 and 5 execute atomically: either both succeed or neither is committed. If persistence fails, no job message is published and HTTP 500 is returned to the caller.

**Creation sequence — problem_id path (experiment context):**

1. Use the caller-provided `problem_id`. No routing problem record is created.
2. Generate `job_id`: UUID, system-generated.
3. Resolve `scheduler_config_id`: use the provided value, or the default configuration's ID (SPEC-003 FR-14).
4. Create the job record in PostgreSQL with `job_id`, the provided `problem_id`, `scheduler_config_id`, `backend_id` (nullable — stored when present in the request, NULL when absent), `status = Pending`, `cancellation_requested = false`, and `created_at` timestamp.
5. Step 4 executes atomically. If persistence fails, no job message is published and HTTP 500 is returned.

Note: In the problem_id path, the `problem_id` returned in the response and in the queue message is the caller-supplied value, not a system-generated one. The `problem_id` in the response is unchanged from the input. Multiple jobs may reference the same `problem_id` via this path without violating referential integrity (SPEC-012 FR-6: no UNIQUE constraint on `jobs.problem_id`).

**Acceptance Criteria:**
- In the routing_problem path: `problem_id` is a UUID generated by the API and returned in the response
- In the problem_id path: `problem_id` in the response equals the caller-provided value; no routing problem record is created
- `job_id` is a UUID generated by the API and returned in the response
- In the routing_problem path: the routing problem record is persisted to PostgreSQL before the job message is published to RabbitMQ
- The job record initial status is `Pending`
- The job record initializes `cancellation_requested = false`; the field must exist in the record at creation so that FR-11 can write `true` to it when a cancellation request is accepted
- When `backend_id` is present in the request, the job record stores the value in the nullable `backend_id` column; when absent, the column is NULL
- A PostgreSQL failure during persistence causes HTTP 500; no queue message is published
- The `problem_id` appears on all subsequent log events, spans, and evidence records for the job

---

### FR-5: Queue Publication

**Description:**
After successful persistence, the API publishes a job message to the RabbitMQ routing-jobs queue (ADR-003).

**Message content:**

| Field | Value | Present |
|---|---|---|
| `job_id` | The UUID generated in FR-4 | Always |
| `problem_id` | The UUID generated in FR-4 | Always |
| `scheduler_config_id` | The resolved configuration identifier | Always |
| `backend_id` | The backend targeting directive from the request | When `backend_id` was present in the submission request; absent from the message when the request did not include `backend_id` |

Absence of `backend_id` in the queue message signals standard-path execution to the Worker; presence signals targeted-path execution. The Worker forwards `backend_id` to the Scheduler when present (SPEC-005 FR-5).

**AMQP message metadata:**
At publication time, the API injects W3C TraceContext (`traceparent`, optionally `tracestate`) from the active `job.submit` span context into the AMQP message `application_headers`. This is message-level metadata, distinct from the message payload body. The trace context schema carried in `application_headers` is owned by ADR-011.

**Publication semantics:**

- The message is published only after both the routing problem record and the job record are successfully persisted (FR-4).
- If publication fails, the API returns HTTP 500. The job record has already been persisted with `status = Pending`. The job message was not published; the job will not be executed unless requeued by an operator. This is an infrastructure failure condition.
- The API does not retry queue publication on failure. Recovery from failed publication is an operational concern.
- The API does not wait for the Worker to process the job. Publication is fire-and-forget from the API's perspective.

**Acceptance Criteria:**
- No job message is published unless routing problem and job records are committed to PostgreSQL
- The published message payload body contains `job_id`, `problem_id`, and `scheduler_config_id` on every submission
- When `backend_id` is present in the submission request, the published message payload includes `backend_id`; when absent from the request, `backend_id` is not included in the message
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
- UUID, generated by the API at submission time in the routing_problem path (FR-4), or supplied by the caller in the problem_id path (FR-2)
- Identifies the routing problem record in PostgreSQL
- Appears on all log events, spans, and evidence records (SPEC-001 FR-1)
- Stable for the lifetime of the problem record
- **Cardinality constraint (scoped):** In standard job submissions (routing_problem path), each routing problem is associated with exactly one job. In experiment context (problem_id path, ADR-012 Decision 3), a routing problem may be referenced by multiple jobs. This relaxation applies only when the `problem_id` path in FR-2 is used. The SPEC-012 FR-6 schema already supports multiple jobs referencing the same `problem_id`; no UNIQUE constraint exists on `jobs.problem_id`. The instance-sharing invariant is a correctness requirement for cross-solver quality comparisons under SPEC-007 FR-7 (SPEC-020 FR-4, ADR-012 Decision 3).

**`backend_id` — the backend targeting directive (ADR-013):**
- An optional string field supplied by the caller on `POST /v1/jobs`
- When present, stored in the job record (nullable `backend_id` column per SPEC-012) and included in the queue message payload
- A routing directive for the Scheduler: it indicates which backend to target; it is not an experiment reference and does not link the job to any experiment or trial (ADR-013 Decision 7)
- `backend_id` is not a public lifecycle identifier in the same sense as `job_id` or `problem_id`; it is not returned in submission responses or status responses

**Internal identifiers not exposed in API responses:**
- `decision_id` — Scheduler decision record key (SPEC-003 internal)
- `execution_seed` — derived by the Worker (SPEC-005 FR-7); must not appear in API responses or logs (SPEC-005 Security Considerations)
- Solver run record identifiers
- Quality evaluation record identifiers

**Acceptance Criteria:**
- Every job submission response includes both `job_id` and `problem_id`
- In the routing_problem path: `job_id` and `problem_id` are distinct UUIDs generated by the API
- In the problem_id path: `job_id` is a new UUID; `problem_id` equals the caller-supplied value
- No API response includes `decision_id`, `execution_seed`, or internal solver execution identifiers
- A caller who polls `GET /v1/jobs/{job_id}` receives the same `problem_id` that was returned at submission
- The `jobs` table does not contain an `experiment_id` column; experiment-to-job linkage is held in `experiment_trials.job_id` (SPEC-012 FR-21.4, ADR-012 Decision 4)
- `backend_id` is a job record field and queue message field, not a submission response field; it is stored at job creation but is not part of the caller-facing identifier contract

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

**Note — `NoEligibleSolver` path distinction:** The `failure_reason` field reflects the coarse failure classification from the failure record (`failure_class`). The status response does not distinguish a targeted-path `NoEligibleSolver` (Scheduler rejected the specified backend with `selection_mode = explicitly_targeted`) from a standard-path `NoEligibleSolver` (Scheduler found no eligible backend by policy). Both produce the same `failure_reason`. Callers requiring this distinction — e.g., the experiment harness determining whether targeted ineligibility occurred — should consult the evidence report (`GET /v1/jobs/{job_id}/report`) or, in experiment context, the trial evidence state returned by the collect-evidence endpoint (FR-21), which includes the Scheduler decision record with `selection_mode`.

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
The API reads the report file from the report volume shared between the Worker and API containers (architecture.md container topology). The API does not generate the file; it reads and serves a file that the Report Generator wrote. The API locates the report file by looking up the report metadata record (SPEC-006 FR-9) for the given `report_id`; it does not construct file paths directly from the `report_id` or any other caller-supplied value. File naming conventions, storage structure, and file location on the report volume are owned by SPEC-009 (FR-6, FR-7) — SPEC-008 is a consumer of that contract, not its owner.

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
| 400 | `SCHEDULER_CONFIG_NOT_FOUND` | `scheduler_config_id` in job submission or experiment manifest references no stored configuration |
| 400 | `PROBLEM_NOT_FOUND` | `problem_id` in job submission (problem_id path, FR-2) references no record in `routing_problems` |
| 400 | `EXPERIMENT_CONFIG_INVALID` | Experiment manifest is structurally invalid, references unresolvable resources (Fixed Mode problem_ids), or fails domain validation (FR-18) |
| 400 | `BENCHMARK_CONFIG_INVALID` | Benchmark manifest is structurally invalid or missing required fields (FR-19) |
| 404 | `JOB_NOT_FOUND` | No job with the given `job_id` |
| 404 | `REPORT_NOT_FOUND` | No report for the given job or report_id |
| 404 | `SCHEDULER_CONFIG_NOT_FOUND` | No configuration with the given `scheduler_config_id` (GET endpoint) |
| 404 | `EXPERIMENT_NOT_FOUND` | No experiment with the given `experiment_id` |
| 404 | `TRIAL_NOT_FOUND` | No trial with the given `trial_id` in the specified experiment |
| 404 | `BENCHMARK_NOT_FOUND` | No benchmark summary exists for the given `benchmark_id` (no member experiment has yet completed) |
| 404 | `EXPERIMENT_SUMMARY_NOT_FOUND` | Experiment exists but has not yet reached `Completed` status; summary not yet produced |
| 409 | `JOB_ALREADY_TERMINAL` | Cancellation requested for a terminal job |
| 409 | `BENCHMARK_ALREADY_EXISTS` | Benchmark manifest submission uses a `benchmark_id` already in use (FR-19) |
| 409 | `JOB_NOT_TERMINAL` | Evidence collection requested for a trial whose job has not yet reached terminal state (FR-21) |
| 409 | `TRIAL_ALREADY_SUBMITTED` | Trial submission linkage requested for a trial whose `job_id` is already set; trial has already been linked to a job (FR-25) |
| 409 | `JOB_PROBLEM_MISMATCH` | Trial submission linkage requested with a `job_id` whose `problem_id` does not match the trial's `problem_id`; instance-sharing invariant violation (ADR-012 Decision 3) (FR-25) |
| 409 | `JOB_BACKEND_MISMATCH` | Trial submission linkage requested with a `job_id` whose `backend_id` (non-null) does not match the trial's `backend_id`; evidence integrity invariant violation (ADR-013, POD-1) (FR-25) |
| 409 | `TRIAL_NOT_SUBMITTED` | Evidence collection requested for a trial whose `job_id` is null; trial has not been linked to a job via FR-25 (FR-21) |
| 409 | `EXPERIMENT_NOT_ACTIVE` | Trial submission linkage requested for an experiment whose status is not `Created` or `Running`; trial submission is not permitted on terminal experiments (FR-25) |
| 500 | `INTERNAL_ERROR` | Unexpected server error (PostgreSQL unavailable, publication failure, evidence collection infrastructure failure, etc.) |

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
- `POST /v1/experiments`
- `GET /v1/experiments/{experiment_id}`
- `GET /v1/experiments/{experiment_id}/trials`
- `POST /v1/experiments/{experiment_id}/trials/{trial_id}/submit`
- `POST /v1/experiments/{experiment_id}/trials/{trial_id}/collect-evidence`
- `GET /v1/experiments/{experiment_id}/summary`
- `POST /v1/benchmarks`
- `GET /v1/benchmarks/{benchmark_id}/summary`

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

**Experiment manifest submission (`POST /v1/experiments`):** Not idempotent. Each call creates a new experiment with a new `experiment_id`.

**Benchmark manifest submission (`POST /v1/benchmarks`):** Idempotent on `benchmark_id`. A second submission of the same `benchmark_id` returns `409 Conflict`; the manifest is not updated.

**Experiment status retrieval (`GET /v1/experiments/{experiment_id}`):** Idempotent.

**Trial results retrieval (`GET /v1/experiments/{experiment_id}/trials`):** Idempotent.

**Trial submission linkage (`POST /v1/experiments/{experiment_id}/trials/{trial_id}/submit`):** Not idempotent. A trial may be linked to exactly one `job_id`. A call for a trial with an existing `job_id` returns `409 Conflict` with `error_code = TRIAL_ALREADY_SUBMITTED`; a call for an experiment whose status is not in `{Created, Running}` returns `409 Conflict` with `error_code = EXPERIMENT_NOT_ACTIVE`.

**Trial evidence collection trigger (`POST /v1/experiments/{experiment_id}/trials/{trial_id}/collect-evidence`):** Idempotent. Repeated calls for the same `trial_id` when evidence has already been collected return `200 OK` with the existing evidence state without re-reading or re-writing. This supports CLI retry on network failure and CLI restart during experiment recovery (ADR-012 SPEC-008-R1 idempotency requirement). The upsert approach aligns with SPEC-012 FR-12's idempotency pattern.

**Experiment summary retrieval (`GET /v1/experiments/{experiment_id}/summary`):** Idempotent.

**Benchmark summary retrieval (`GET /v1/benchmarks/{benchmark_id}/summary`):** Idempotent.

**Acceptance Criteria:**
- Two identical `POST /v1/jobs` requests produce two distinct `job_id` values; in the routing_problem path, they also produce two distinct `problem_id` values; in the problem_id path, both reference the same `problem_id`
- `GET` requests produce no side effects on any persistent state
- A `POST /v1/jobs/{job_id}/cancel` on an already-terminal job returns `409 Conflict`, not `202 Accepted`
- A second `POST /v1/benchmarks` with the same `benchmark_id` returns `409 Conflict`
- Repeated `POST /v1/experiments/{experiment_id}/trials/{trial_id}/collect-evidence` calls return `200 OK` and do not duplicate evidence records
- A call to `POST /v1/experiments/{experiment_id}/trials/{trial_id}/submit` for a trial whose `job_id` is already set returns HTTP 409 with `error_code = TRIAL_ALREADY_SUBMITTED`
- A call to `POST /v1/experiments/{experiment_id}/trials/{trial_id}/submit` for an experiment whose `status` is not in `{Created, Running}` returns HTTP 409 with `error_code = EXPERIMENT_NOT_ACTIVE`

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
| `experiment.submit` | API | Covers experiment manifest validation and persistence (FR-18) |
| `experiment.trial_submit` | API | Covers trial job linkage: validates job/trial integrity and sets `experiment_trials.job_id` (FR-25) |
| `experiment.evidence_collect` | API | Covers Evidence Log read and experiment trial record write for one trial (FR-21) |
| `experiment.summarize` | API | Covers experiment summary artifact generation when experiment reaches `Completed` (FR-21 auto-completion path) |

**`job.submit` span attributes (required):**

| Attribute | Description |
|---|---|
| `job_id` | The generated job identifier |
| `problem_id` | The generated problem identifier |
| `scheduler_config_id` | The resolved scheduler configuration identifier |
| `backend_id` | The backend targeting directive when present in the request; omitted from the span when absent (ADR-013) |
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
| `api.job.submitted` | POST /v1/jobs | `job_id`, `problem_id`, `scheduler_config_id`, `stop_count`, `vehicle_count`, `backend_id` (when present) |
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
| `daedalus_api_submissions_rejected_total` | Counter | `rejection_reason` (`validation_error`, `config_not_found`, `problem_not_found`) | Total submissions rejected at validation |
| `daedalus_api_job_submission_duration_seconds` | Histogram | `outcome` (`success`, `validation_failure`, `server_error`) | End-to-end submission request duration |
| `daedalus_api_status_poll_total` | Counter | `job_status` (`Pending`, `Processing`, `Completed`, `Failed`, `not_found`) | Status poll outcomes |
| `daedalus_api_cancellations_total` | Counter | `outcome` (`accepted`, `already_terminal`, `not_found`) | Cancellation request outcomes |
| `daedalus_api_experiments_submitted_total` | Counter | — | Total experiment manifest submissions receiving 201 Created |
| `daedalus_api_experiments_completed_total` | Counter | `reproducibility_class` | Total experiments that reached `Completed` status |
| `daedalus_api_trials_evidence_collected_total` | Counter | `evidence_status` (`Collected`, `Missing`, `Error`) | Trial evidence collection outcomes |

**Acceptance Criteria:**
- `job.submit` is emitted on every job submission attempt, successful or not
- All required `job.submit` attributes are present on every emission; `backend_id` is included in the span when present in the submission request and omitted when absent
- `job.submit` span status is `Error` on any 4xx or 5xx response
- The `job.submit` span context is injected as W3C TraceContext in AMQP message `application_headers` on every successful publication (FR-5)
- `experiment.submit` is emitted on every experiment manifest submission attempt
- `experiment.trial_submit` is emitted on every trial submission linkage call
- `experiment.evidence_collect` is emitted on every trial evidence collection call
- `experiment.summarize` is emitted when an experiment auto-transitions to `Completed`
- Structured log events do not include routing problem coordinate data

---

### FR-18: Experiment Manifest Submission

**Description:**
The API accepts experiment manifest submissions from the CLI orchestrator (SPEC-016). The experiment manifest is the complete, immutable configuration for one experiment (SPEC-020 FR-3). On submission, the API validates the manifest, persists the experiment record and trial records (Fixed Mode), and returns the experiment identifier. All durable experiment state is API-backed; the CLI is the orchestration executor but not the state authority (ADR-012 Decisions 1 and 2).

**Endpoint:** `POST /v1/experiments`

**Request body fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `experiment_name` | string | Yes | Human-readable experiment name (SPEC-020 FR-1) |
| `benchmark_id` | string | Yes | Grouping identifier associating this experiment with a benchmark (SPEC-020 FR-1) |
| `experiment_seed` | uint64 | Yes | Top-level entropy source for trial seed derivation (SPEC-020 FR-8) |
| `workload_set` | object | Yes | WorkloadSetDefinition; see below (SPEC-020 FR-4) |
| `solver_set` | array of strings | Yes | Ordered list of `backend_id` values; minimum 1 element (SPEC-020 FR-5) |
| `repetition_count` | positive integer | Yes | Number of executions per (problem_config, solver) pairing; minimum 1 (SPEC-020 FR-7) |
| `execution_timeout_ms_per_trial` | positive integer | Yes | Per-trial execution timeout in milliseconds; must be positive (SPEC-020 FR-3) |
| `seed_derivation_algorithm_version` | string | Yes | Version identifier for the seed derivation algorithm used by the CLI to compute per-trial seeds from `experiment_seed` (SPEC-012 FR-20.4). Allows reproducibility verification across CLI versions. |
| `reproducibility_class` | string | Yes | Reproducibility classification for this experiment; derived by the CLI from the union of `determinism_class` values across `solver_set` backends (SPEC-020 FR-9). One of `"fully_reproducible"`, `"partially_reproducible"`, or `"non_reproducible"`. |
| `scheduler_config_id` | UUID string | No | Scheduler configuration applied to every trial; defaults to default configuration when absent (SPEC-003 FR-14) |
| `hardware_policy` | object | No | Retry and failure handling for `quantum_hardware` backend trials (SPEC-020 FR-10); defaults apply when absent |

**WorkloadSetDefinition — Fixed Mode:**

| Field | Type | Required | Description |
|---|---|---|---|
| `mode` | string | Yes | Must be `"fixed"` |
| `problem_ids` | array of UUID strings | Yes | One or more existing routing problem identifiers (SPEC-020 FR-4 Mode A). Minimum 1 element. Each must reference a record in `routing_problems`. |

**WorkloadSetDefinition — Generated Mode:**

| Field | Type | Required | Description |
|---|---|---|---|
| `mode` | string | Yes | Must be `"generated"` |
| `generation_config` | object | Yes | SPEC-002 generation configuration: scenario type, stop count range, fleet parameters, time window parameters |
| `instance_count` | positive integer | Yes | Number of distinct routing problem instances per repetition; minimum 1 (SPEC-020 FR-4 Mode B) |

**Validation:**

| Rule | Condition | Rejection |
|---|---|---|
| `experiment_name` non-empty | | `experiment_name must be a non-empty string` |
| `benchmark_id` non-empty | | `benchmark_id must be a non-empty string` |
| `experiment_seed` ≥ 0 | uint64 range | `experiment_seed must be a non-negative 64-bit integer` |
| `solver_set` non-empty | | `solver_set must contain at least one backend_id` |
| `repetition_count` ≥ 1 | | `repetition_count must be a positive integer` |
| `execution_timeout_ms_per_trial` > 0 | | `execution_timeout_ms_per_trial must be a positive integer` |
| `workload_set.mode` recognized | Must be `"fixed"` or `"generated"` | `workload_set.mode must be fixed or generated` |
| `scheduler_config_id` valid UUID format (if provided) | | `scheduler_config_id is not a valid UUID` |
| `scheduler_config_id` exists in PostgreSQL (if provided) | | `scheduler_config_id not found` |
| Fixed Mode: `problem_ids` non-empty | | `problem_ids must contain at least one element` |
| Fixed Mode: all `problem_ids` resolvable in PostgreSQL | Any `problem_id` absent from `routing_problems` | `problem_ids contains unresolvable identifiers: [{id}, ...]`; error_code = `EXPERIMENT_CONFIG_INVALID` |
| Generated Mode: `instance_count` ≥ 1 | | `instance_count must be a positive integer` |
| Generated Mode: `generation_config` structurally valid per SPEC-002 | Missing or invalid fields | `generation_config: {field}` |
| `seed_derivation_algorithm_version` non-empty | | `seed_derivation_algorithm_version must be a non-empty string` |
| `reproducibility_class` recognized value | Must be one of `"fully_reproducible"`, `"partially_reproducible"`, `"non_reproducible"` | `reproducibility_class must be one of: fully_reproducible, partially_reproducible, non_reproducible` |

**Fixed Mode problem_id validation:** The API validates every `problem_id` in the Fixed Mode `problem_ids` list against `routing_problems` before any persistence occurs. A single unresolvable `problem_id` causes the entire submission to be rejected. No partial experiment is created. All unresolvable identifiers are collected and returned together in the response.

**Backend capability-profile validation gap (F-13):** The `solver_set` backend identifiers are not validated against Scheduler capability profiles at experiment submission time. Scheduler capability profiles are not persisted in PostgreSQL (SPEC-012 FR-2 scope); the API has no query path to verify that a `backend_id` is registered and eligible. SPEC-020 FR-5 places pre-submission eligibility verification on the CLI (SPEC-016). A trial job submitted for an unregistered or ineligible `backend_id` will fail at the Scheduler stage, not at the experiment submission stage. This is an accepted MVP constraint.

**Computed manifest fields:**

| Field | Computation |
|---|---|
| `experiment_id` | UUID, system-generated |
| `planned_trial_count` | `problem_count × |solver_set| × repetition_count`, where `problem_count = |problem_ids|` (Fixed) or `instance_count` (Generated) |
| `workload_mode` | Derived from `workload_set.mode` |
| `reproducibility_class` | Supplied by CLI in the request body. The CLI derives this from the union of backend `determinism_class` values in `solver_set` per SPEC-020 FR-9. The API validates the supplied value against recognized classes and persists it; no independent derivation occurs at the API layer. |

**Persistence behavior:**

*Fixed Mode:* The API validates all `problem_ids`, creates the experiment record (SPEC-012 FR-20), and creates all trial records (SPEC-012 FR-21) in `Pending` trial_status atomically. The experiment is created in `Created` status. `planned_trial_count` equals `|problem_ids| × |solver_set| × repetition_count`.

*Generated Mode:* The API creates the experiment record (SPEC-012 FR-20) in `Created` status. Trial records are deferred until the CLI provides the generated routing problem identifiers (see OQ-7). The experiment cannot transition to `Running` until trial records are registered.

**Worker experiment-unawareness:** The `jobs` table does not receive an `experiment_id` column. The Worker has no visibility into the experiment. Experiment-to-job linkage is held exclusively in `experiment_trials.job_id` (SPEC-012 FR-21.4, ADR-012 Decision 4).

**HTTP status:** `201 Created`

**Response body:**

| Field | Type | Description |
|---|---|---|
| `experiment_id` | UUID string | System-generated experiment identifier |
| `benchmark_id` | string | The benchmark grouping identifier |
| `status` | string | Initial status: `"Created"` |
| `planned_trial_count` | integer | Computed trial count |
| `workload_mode` | string | `"Fixed"` or `"Generated"` |
| `reproducibility_class` | string | `"fully_reproducible"`, `"partially_reproducible"`, or `"non_reproducible"` |
| `experiment_url` | string | Absolute URL: `GET /v1/experiments/{experiment_id}` |

**Acceptance Criteria:**
- A request with missing required fields is rejected with HTTP 400 before any persistence
- A request with an unrecognized `reproducibility_class` value is rejected with HTTP 400
- Fixed Mode: any `problem_id` not found in `routing_problems` causes HTTP 400 with `error_code = EXPERIMENT_CONFIG_INVALID` and a list of all unresolvable identifiers; no experiment record is created
- Fixed Mode: the experiment record and all trial records are created atomically; a PostgreSQL failure causes HTTP 500 and no partial state is committed
- Generated Mode: only the experiment record is created at submission time; trial records are deferred (OQ-7)
- `planned_trial_count` is computed correctly per the formula above
- No trial is submitted by the API at experiment submission time
- The `jobs` table receives no `experiment_id` column

---

### FR-19: Benchmark Manifest Submission

**Description:**
The API accepts benchmark manifest submissions. A benchmark manifest records the research question, hypothesis, and variable definitions for a named research initiative (SPEC-020 FR-2). The manifest is immutable after submission. Submitting a benchmark manifest before experiments are submitted is optional; experiments may reference a `benchmark_id` without a prior benchmark manifest (SPEC-020 FR-2).

**Endpoint:** `POST /v1/benchmarks`

**Request body fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `benchmark_id` | string | Yes | User-supplied identifier; serves as the lookup key. Must be unique across all submitted benchmark manifests. |
| `benchmark_name` | string | Yes | Human-readable benchmark name |
| `research_question` | string | Yes | The question the benchmark attempts to answer |
| `hypothesis` | string | Yes | Expected outcome |
| `null_hypothesis` | string | Yes | The condition that holds if the proposed improvement does not exist |
| `controls` | array of strings | Yes | Variables held constant across experiments; minimum 1 element |
| `independent_variables` | array of strings | Yes | Variables that change between experiments; minimum 1 element |
| `dependent_variables` | array of strings | Yes | Variables measured; minimum 1 element |

**Validation:**

| Rule | Rejection |
|---|---|
| All string fields non-empty | `{field} must be a non-empty string` |
| `controls`, `independent_variables`, `dependent_variables` non-empty arrays | `{field} must contain at least one element` |
| `benchmark_id` not already in `benchmark_manifests` | HTTP 409, `error_code = BENCHMARK_ALREADY_EXISTS` |

**HTTP status:**
- `201 Created` on success
- `400 Bad Request` for structural or field validation failures
- `409 Conflict` when `benchmark_id` already exists in `benchmark_manifests`

**Response body (201):**

| Field | Type | Description |
|---|---|---|
| `benchmark_id` | string | The submitted identifier |
| `benchmark_name` | string | |
| `created_at` | ISO 8601 UTC | Timestamp of creation |

**Acceptance Criteria:**
- A benchmark manifest with an existing `benchmark_id` is rejected with HTTP 409; the existing manifest is not modified
- The manifest is persisted to `benchmark_manifests` (SPEC-012 FR-19) before the response is returned
- All required fields must be non-empty; any missing or empty field is rejected with HTTP 400
- Submitting a benchmark manifest after experiments have already referenced the `benchmark_id` is valid; the manifest provides research context retroactively

---

### FR-20: Experiment Status Retrieval

**Description:**
The API exposes an endpoint for retrieving the current status and trial progress of an experiment.

**Endpoint:** `GET /v1/experiments/{experiment_id}`

**Response body:**

| Field | Type | Present | Description |
|---|---|---|---|
| `experiment_id` | UUID string | Always | |
| `benchmark_id` | string | Always | |
| `experiment_name` | string | Always | |
| `status` | string | Always | Current `ExperimentStatus` (SPEC-020 FR-11): `"Created"`, `"Running"`, `"Completed"`, or `"Failed"` |
| `planned_trial_count` | integer | Always | From manifest |
| `effective_trial_count` | integer | When set | Set when partial workload generation failure reduces actual trial count below `planned_trial_count` |
| `workload_mode` | string | Always | `"Fixed"` or `"Generated"` |
| `reproducibility_class` | string | Always | |
| `submitted_at` | ISO 8601 UTC | Always | Experiment record creation timestamp |
| `started_at` | ISO 8601 UTC | When Running or later | Set when first trial is submitted |
| `completed_at` | ISO 8601 UTC | When `status = Completed` | Set when experiment transitions to `Completed` |
| `failed_at` | ISO 8601 UTC | When `status = Failed` | Set when experiment transitions to `Failed` |
| `failure_reason` | string | When `status = Failed` | |
| `trial_counts` | object | Always | Trial count breakdown; see below |

**`trial_counts` object:**

| Field | Type | Description |
|---|---|---|
| `pending` | integer | Trials in `Pending` trial_status |
| `submitted` | integer | Trials in `Submitted` trial_status |
| `executing` | integer | Trials in `Executing` trial_status. Reserved for future progress reporting; always `0` at MVP scope because no SPEC-008 endpoint transitions `trial_status` to `Executing` |
| `completed` | integer | Trials in `Completed` trial_status |
| `scheduler_rejected` | integer | Trials in `SchedulerRejected` trial_status |
| `harness_error` | integer | Trials in `HarnessError` trial_status |
| `evidence_collected` | integer | Trials with `evidence_status = Collected` |

**HTTP status codes:**
- `200 OK`: experiment found
- `404 Not Found`: no experiment with the given `experiment_id`; `error_code = EXPERIMENT_NOT_FOUND`

**Acceptance Criteria:**
- Status reflects the current PostgreSQL `experiments` record state at the time of the request
- `trial_counts` is derived from `experiment_trials` records for this experiment
- The API does not cache experiment status; every request reads from PostgreSQL
- `completed_at` and `failed_at` are mutually exclusive

---

### FR-25: Trial Submission Linkage

**Description:**
After creating an experiment and receiving trial records (FR-18), the CLI submits one job per backend per trial via `POST /v1/jobs` and receives a `job_id` for each. The CLI then calls this endpoint to link each trial record to its corresponding job. This link is required before evidence collection (FR-21 step 2). This endpoint is the mechanism by which `experiment_trials.job_id` is set and by which the experiment transitions from `Created` to `Running` (ADR-012 Decision 1).

**Endpoint:** `POST /v1/experiments/{experiment_id}/trials/{trial_id}/submit`

**Request body:**

| Field | Type | Required | Description |
|---|---|---|---|
| `job_id` | UUID string | Yes | The `job_id` returned by `POST /v1/jobs` for this trial's backend |

**Validation:**

| Rule | Condition | Rejection |
|---|---|---|
| Experiment exists | `experiment_id` not found in `experiments` | HTTP 404, `error_code = EXPERIMENT_NOT_FOUND` |
| Experiment is active | `experiments.status` not in `{Created, Running}` | HTTP 409, `error_code = EXPERIMENT_NOT_ACTIVE` |
| Trial belongs to experiment | `trial_id` not found in `experiment_trials` for this experiment | HTTP 404, `error_code = TRIAL_NOT_FOUND` |
| Trial not yet submitted | `experiment_trials.job_id` is not null | HTTP 409, `error_code = TRIAL_ALREADY_SUBMITTED` |
| Job exists | `job_id` not found in `jobs` | HTTP 404, `error_code = JOB_NOT_FOUND` |
| Problem identity | `jobs.problem_id ≠ experiment_trials.problem_id` | HTTP 409, `error_code = JOB_PROBLEM_MISMATCH` |
| Backend identity | `jobs.backend_id` is non-null and `jobs.backend_id ≠ experiment_trials.backend_id` | HTTP 409, `error_code = JOB_BACKEND_MISMATCH` |

**Behavior:**

The API performs the following sequence:

1. Validates experiment exists; returns HTTP 404 with `error_code = EXPERIMENT_NOT_FOUND` if not.
2. Validates `experiments.status ∈ {Created, Running}`; returns HTTP 409 with `error_code = EXPERIMENT_NOT_ACTIVE` if not. This prevents trial submission linkage on experiments that have already reached a terminal status (`Completed`, `Failed`).
3. Validates trial belongs to experiment; returns HTTP 404 with `error_code = TRIAL_NOT_FOUND` if not.
4. Validates `experiment_trials.job_id` is null; returns HTTP 409 with `error_code = TRIAL_ALREADY_SUBMITTED` if already set.
5. Validates the referenced `job_id` exists in `jobs`; returns HTTP 404 with `error_code = JOB_NOT_FOUND` if not.
6. Validates `jobs.problem_id` equals `experiment_trials.problem_id`; returns HTTP 409 with `error_code = JOB_PROBLEM_MISMATCH` if not. This enforces the instance-sharing invariant (ADR-012 Decision 3): all backends for a given trial share the same routing problem.
7. When `jobs.backend_id` is non-null: validates `jobs.backend_id` equals `experiment_trials.backend_id`; returns HTTP 409 with `error_code = JOB_BACKEND_MISMATCH` if they differ. This enforces the evidence integrity invariant: a trial job must be directed at the same backend the trial is assigned to, ensuring evidence is produced by the claimed backend (ADR-013, POD-1). When `jobs.backend_id` is null (standard-path job), this check is skipped.
8. Sets `experiment_trials.job_id = job_id` and transitions `trial_status` from `Pending` to `Submitted`.
9. If this is the first submitted trial for the experiment (experiment is in `Created` status): transitions `experiments.status` from `Created` to `Running` and sets `experiments.started_at`.
10. Returns HTTP 200 with the updated trial state.

**HTTP status:**
- `200 OK`: trial successfully linked to job
- `404 Not Found`: experiment not found (`EXPERIMENT_NOT_FOUND`), trial not found (`TRIAL_NOT_FOUND`), or job not found (`JOB_NOT_FOUND`)
- `409 Conflict`: experiment is not in an active state (`EXPERIMENT_NOT_ACTIVE`), trial already has a `job_id` (`TRIAL_ALREADY_SUBMITTED`), `job.problem_id ≠ trial.problem_id` (`JOB_PROBLEM_MISMATCH`), or `job.backend_id` (non-null) `≠ trial.backend_id` (`JOB_BACKEND_MISMATCH`)

**Response body (200):**

| Field | Type | Description |
|---|---|---|
| `trial_id` | UUID string | |
| `experiment_id` | UUID string | |
| `job_id` | UUID string | The linked `job_id` |
| `trial_status` | string | `"Submitted"` after successful linkage |
| `experiment_status` | string | Current experiment status; `"Running"` if this was the first submitted trial |

**Observability:** The `experiment.trial_submit` span is emitted for each call (FR-17).

**Acceptance Criteria:**
- A call that successfully links a trial to a job returns HTTP 200 with `trial_status = "Submitted"`
- A call for an experiment whose `status` is not `Created` or `Running` returns HTTP 409 with `error_code = EXPERIMENT_NOT_ACTIVE`
- A call for a trial whose `job_id` is already set returns HTTP 409 with `error_code = TRIAL_ALREADY_SUBMITTED`
- A call with a `job_id` whose `problem_id` does not match the trial's `problem_id` returns HTTP 409 with `error_code = JOB_PROBLEM_MISMATCH`
- A call with a `job_id` whose `backend_id` (non-null) does not match the trial's `backend_id` returns HTTP 409 with `error_code = JOB_BACKEND_MISMATCH`
- A call with a `job_id` whose `backend_id` is null (standard-path job) does not trigger backend identity validation; the linkage proceeds if all other checks pass
- The first successful trial submission for an experiment transitions `experiments.status` from `Created` to `Running` and sets `started_at`
- The linkage and status transitions are atomic; a PostgreSQL failure leaves no partial state
- The `experiment.trial_submit` span is emitted on every call

---

### FR-21: Trial Evidence Collection Trigger

**Description:**
When the CLI determines (via job status polling) that a trial's associated job has reached a terminal state, the CLI calls this endpoint to trigger API-mediated evidence collection. The API reads the relevant Evidence Log records for the trial's `job_id` and writes the collected evidence to the experiment trial record (SPEC-012 FR-21). This implements ADR-012 Decision 5.

**Endpoint:** `POST /v1/experiments/{experiment_id}/trials/{trial_id}/collect-evidence`

**Request body:** Empty or absent.

**Evidence collection behavior:**

The API performs the following sequence:

1. Validates that the experiment exists and the trial belongs to it.
2. Checks `experiment_trials.evidence_status`. If `Collected` or `Missing`, returns HTTP 200 with the existing evidence state without proceeding further (idempotency short-circuit; precedes all Evidence Log reads).
3. Validates that the trial's `job_id` is non-null. If null, the trial has not been linked to a job via FR-25 and evidence collection cannot proceed; returns HTTP 409 with `error_code = TRIAL_NOT_SUBMITTED`.
4. Validates that the trial's associated job has reached a terminal state (`jobs.status = Completed` or `jobs.status = Failed`). If the job is not yet terminal, returns HTTP 409 with `error_code = JOB_NOT_TERMINAL`.
5. Reads `solver_run_records` for the `job_id`: `solver_outcome`, `execution_duration_ms`, `route_plan`, `extension_metadata`. Absent when `jobs.status = Failed` at an early failure stage. `solution_present` is derived as `route_plan IS NOT NULL`; no additional table read is performed.
6. Reads `quality_evaluation_records` for the `job_id`: `hindsight_quality`, `quality_metrics` (for `quality_comparison_eligible`, `time_window_feasible`). Absent when no quality evaluation was performed.
7. Writes collected evidence to the `experiment_trials` row: updates `trial_status` to `Completed`, sets evidence scalar columns and `evidence_payload`, sets `evidence_status`, sets `evidence_collected_at`. If no `solver_run_records` exist for the `job_id` and the job's status is `Completed` (job completed but Worker wrote no Evidence Log records — abnormal but possible), `evidence_status` is set to `Missing`.
8. Checks whether all trials for the experiment are in a terminal `trial_status` (`Completed`, `SchedulerRejected`, or `HarnessError`). If yes, and only if `experiments.status = 'Running'`, triggers the experiment completion sequence (see below). If `experiments.status` is already `Completed`, step 8 is a no-op; the experiment completion sequence is not triggered. This guard prevents a duplicate ExperimentSummary INSERT (SPEC-012 FR-22 UNIQUE constraint) when an `Error`-status trial is retried after the experiment has already transitioned to `Completed`.

**Experiment auto-completion:**
When step 7 determines all trials are terminal, the API automatically:
- Computes QualityStatsAggregate artifacts for all (problem, solver) pairings with sufficient evidence (SPEC-020 FR-12, FR-13); verifies no existing `experiment_artifacts` row for `(experiment_id, problem_id, backend_id)` before INSERT per SPEC-012 FR-22.4 (application-layer idempotency guard)
- Computes the ExperimentSummary artifact (SPEC-020 FR-14 Artifact 3)
- Persists both to `experiment_artifacts` (SPEC-012 FR-22)
- Transitions the experiment to `Completed` and sets `completed_at`
- Triggers benchmark summary update: recomputes the `benchmark_summaries` record for the experiment's `benchmark_id` (SPEC-012 FR-23)

The experiment status in the response reflects the post-collection state. The auto-completion is triggered within the collect-evidence endpoint, making the Running → Completed transition robust to the specific CLI interruption sub-case where all trials complete but the CLI exits before initiating a separate completion trigger (ADR-012 Accepted Risks).

**Idempotency:**
This endpoint's idempotency behavior varies by `evidence_status`:

1. **`Collected`:** Idempotent. Repeated calls return HTTP 200 with the existing evidence state without re-reading Evidence Log tables or re-writing trial evidence. This supports CLI retry on network failure and CLI restart during experiment recovery (ADR-012 SPEC-008-R1 idempotency requirement). The idempotency check precedes all Evidence Log reads.
2. **`Error`:** Retryable. A prior call that resulted in `evidence_status = Error` indicates a transient failure during Evidence Log reads. Repeated calls re-attempt Evidence Log reads and may produce a different `evidence_status` on success. The CLI may retry evidence collection for `Error` trials.
3. **`Missing`:** Terminal. A prior call that resulted in `evidence_status = Missing` indicates Evidence Log records are absent for a completed job. Repeated calls return HTTP 200 with the existing `Missing` state without re-reading Evidence Log tables. `Missing` evidence is not retryable.

**SPEC-006 FR-1.3 compliance:** The API reads from Evidence Log tables (`solver_run_records`, `quality_evaluation_records`) and writes to `experiment_trials` (an experiment persistence table per SPEC-012 FR-21, not an Evidence Log table). No write to any Evidence Log table occurs. Worker-only write authority on Evidence Log tables is preserved (SPEC-006 FR-1.3).

**ADR-011 trace propagation:** The `experiment.evidence_collect` span is emitted for each call. W3C TraceContext is propagated within the span scope per ADR-011 requirements for all new CLI→API experiment endpoint interactions.

**HTTP status:**
- `200 OK`: evidence collected (or already collected — idempotent)
- `404 Not Found`: experiment not found (`EXPERIMENT_NOT_FOUND`) or trial not found (`TRIAL_NOT_FOUND`)
- `409 Conflict`: trial has no linked job (`TRIAL_NOT_SUBMITTED`) or trial's job is not yet in terminal state (`JOB_NOT_TERMINAL`)

**Response body (200):**

| Field | Type | Description |
|---|---|---|
| `trial_id` | UUID string | |
| `experiment_id` | UUID string | |
| `trial_status` | string | `"Completed"` after evidence collection |
| `evidence_status` | string | `"Collected"`, `"Missing"`, or `"Error"` |
| `solver_outcome` | string | From Evidence Log; null if no solver run record |
| `hindsight_quality` | float | From Evidence Log; null when not available |
| `quality_comparison_eligible` | boolean | From Evidence Log; null when not available |
| `execution_duration_ms` | integer | From Evidence Log; null when not available |
| `solution_present` | boolean | `true` if a valid route plan was produced; `false` if the solver returned no valid solution; null if no solver run record exists |
| `evidence_collected_at` | ISO 8601 UTC | Timestamp of evidence collection |
| `experiment_status` | string | Current experiment status after this collection; may be `"Completed"` if this was the last terminal trial |

**Acceptance Criteria:**
- Repeated calls for a trial with `evidence_status = Collected` return HTTP 200 with the existing evidence state without modification
- A call when the trial's job is not yet in terminal state returns HTTP 409 with `error_code = JOB_NOT_TERMINAL`
- A collect-evidence request for a trial whose `job_id` is null returns HTTP 409 with `error_code = TRIAL_NOT_SUBMITTED`
- Evidence is read from `solver_run_records` and `quality_evaluation_records`; no Evidence Log table is written
- When this call causes the last trial to reach terminal trial_status, the experiment auto-transitions to `Completed` and `experiment_status = "Completed"` in the response
- Auto-completion produces one QualityStatsAggregate artifact for each eligible (problem, solver) scope and does not produce duplicate artifacts for an existing scope (SPEC-012 FR-22.4)
- The ExperimentSummary artifact and benchmark summary update are computed atomically with the experiment status transition
- `experiment_status` in the response reflects the post-collection state of the experiment

---

### FR-22: Trial Results Retrieval

**Description:**
The API exposes an endpoint for retrieving the trial results collection for an experiment. This is the API realization of SPEC-020 FR-14 Artifact 2 (Trial Results Collection), implemented as a query over `experiment_trials` rather than a separate materialized artifact (SPEC-012 FR-22.3).

**Endpoint:** `GET /v1/experiments/{experiment_id}/trials`

**Query parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `status` | string | No | Filter by `trial_status`; accepts comma-separated values (e.g., `Completed,HarnessError`) |
| `backend_id` | string | No | Filter by `backend_id` |

**Response body:**

| Field | Type | Description |
|---|---|---|
| `experiment_id` | UUID string | |
| `trials` | array | Array of trial record objects (see below) |
| `total_count` | integer | Total trial count for the experiment, unfiltered |

**Trial record fields:**

| Field | Type | Present | Description |
|---|---|---|---|
| `trial_id` | UUID string | Always | |
| `problem_id` | UUID string | Always | |
| `backend_id` | string | Always | |
| `repetition_index` | integer | Always | |
| `problem_config_index` | integer | Always | |
| `solver_set_index` | integer | Always | |
| `trial_seed` | uint64 | Always | |
| `trial_type` | string | Always | `"reproducible"` or `"non_reproducible"` |
| `trial_status` | string | Always | Current `TrialStatus` |
| `job_id` | UUID string | When submitted | Null before submission and for `SchedulerRejected` trials |
| `evidence_status` | string | When terminal | `"Collected"`, `"Missing"`, `"Error"`, or null before evidence collection |
| `solver_outcome` | string | When evidence collected | Null before collection or when not available |
| `hindsight_quality` | float | When evidence collected | Null when not available |
| `quality_comparison_eligible` | boolean | When evidence collected | Null before collection |
| `execution_duration_ms` | integer | When evidence collected | Null when not available |
| `retry_count` | integer | Always | 0 for non-hardware trials |
| `harness_failure_reason` | string | When `HarnessError` | |
| `scheduler_rejection_reason` | string | When `SchedulerRejected` | |

**HTTP status codes:**
- `200 OK`
- `404 Not Found`: no experiment with the given `experiment_id`; `error_code = EXPERIMENT_NOT_FOUND`

**Acceptance Criteria:**
- The response contains all trials for the experiment across all statuses when no filter is applied
- `total_count` reflects the total trial count regardless of applied filters
- The API does not cache trial results; every request reads from PostgreSQL
- Filtering by `status` or `backend_id` is applied server-side

---

### FR-23: Experiment Summary Retrieval

**Description:**
The API exposes an endpoint for retrieving the experiment summary artifact (SPEC-020 FR-14 Artifact 3). The summary is computed and persisted when the experiment reaches `Completed` status via the FR-21 auto-completion path.

**Endpoint:** `GET /v1/experiments/{experiment_id}/summary`

**Response body:**
The complete experiment summary payload per SPEC-020 FR-14 Artifact 3: experiment identity fields, trial results collection, per-(problem, solver) aggregate evidence, per-experiment aggregate evidence, cross-solver comparison, QualityStats and RuntimeStats, outcome distribution, and reproducibility annotation. The payload is the `artifact_payload` of the `ExperimentSummary` record in `experiment_artifacts` (SPEC-012 FR-22).

**HTTP status codes:**
- `200 OK`: experiment is `Completed` and summary is available
- `404 Not Found` with `error_code = EXPERIMENT_NOT_FOUND`: no experiment with the given `experiment_id`
- `404 Not Found` with `error_code = EXPERIMENT_SUMMARY_NOT_FOUND`: experiment exists but has not yet reached `Completed` status

**Acceptance Criteria:**
- The summary is available only after the experiment reaches `Completed` status
- Before `Completed`, the endpoint returns 404 with `error_code = EXPERIMENT_SUMMARY_NOT_FOUND`
- The summary payload contains all fields defined in SPEC-020 FR-14 Artifact 3

---

### FR-24: Benchmark Summary Retrieval

**Description:**
The API exposes an endpoint for retrieving the benchmark summary artifact (SPEC-020 FR-14 Artifact 4). The summary is produced when the first member experiment under the `benchmark_id` reaches `Completed` status and updated with each subsequent completion.

**Endpoint:** `GET /v1/benchmarks/{benchmark_id}/summary`

**Response body:**
The complete benchmark summary payload per SPEC-020 FR-14 Artifact 4: benchmark definition fields (from `benchmark_manifests` if a manifest was submitted; omitted otherwise), the list of all experiments under the `benchmark_id` with their status and per-solver summaries, and cross-experiment comparability notes. The payload is the `summary_payload` from the `benchmark_summaries` record (SPEC-012 FR-23).

**HTTP status codes:**
- `200 OK`: a benchmark summary exists for this `benchmark_id`
- `404 Not Found` with `error_code = BENCHMARK_NOT_FOUND`: no `benchmark_summaries` record exists for this `benchmark_id` (no member experiment has yet reached `Completed` status)

**Acceptance Criteria:**
- The summary reflects all experiments under the `benchmark_id` that have reached `Completed` status at the time of the request
- The summary is updated each time a new member experiment completes (via FR-21 auto-completion path)
- Returns 404 before any member experiment reaches `Completed` status
- When a `benchmark_manifests` record exists for the `benchmark_id`, the research context fields are included in the response; when absent, the response contains only experiment-level evidence

---

# Non-Requirements

- The API does not perform solver execution, backend selection, workload feature computation, or quality evaluation (Core and Scheduler responsibilities)
- The API does not generate HTML evidence reports (Report Generator responsibility)
- The API does not write Evidence Log artifact records — job status record creation is the API's responsibility; solver run records, quality evaluation records, failure records, and report metadata records are Worker responsibilities (SPEC-005, SPEC-006)
- The API does not define the Evidence Log persistence schema (SPEC-006 responsibility)
- The API does not define Worker execution behavior (SPEC-005 responsibility)
- The API does not define Worker internal cancellation mechanics beyond writing the `cancellation_requested` flag (Worker implementation planning concern)
- The API does not define report generation behavior (SPEC-009, Report Generator)
- The API does not define report file naming conventions, report storage structure, or report volume layout — these are owned by SPEC-009 (FR-6, FR-7)
- The API does not provide report retrieval compatibility guarantees independently; retrieval compatibility is contingent on SPEC-009 maintaining a stable file location captured in the report metadata record (SPEC-006 FR-9)
- The API does not support multi-tenant isolation, authentication, or authorization at MVP scope (deferred per README)
- The API does not expose internal execution identifiers (`decision_id`, `execution_seed`, solver run identifiers)
- The API does not support routing problem versioning, amendment, or deletion (SPEC-001)
- The API does not support scheduler configuration updates or deletion at MVP scope (OQ-4)
- The API does not implement pagination for list endpoints at MVP scope (OQ-5)
- The API does not expose solver backend registration or capability profile management
- The API does not support bulk job submission
- The API does not support webhook or push-based job completion notifications; callers poll for status
- The API does not submit trial jobs on behalf of the CLI — job submission is performed by the CLI via the existing `POST /v1/jobs` endpoint (ADR-012 Decision 1)
- The API does not poll job status to determine trial completion — job status polling is performed by the CLI; the CLI calls `POST /v1/experiments/{experiment_id}/trials/{trial_id}/collect-evidence` when a job is terminal (ADR-012 Decision 5)
- The API does not write to Evidence Log tables (`solver_run_records`, `quality_evaluation_records`, `solver_failure_records`, `report_metadata_records`) — Worker-only write authority is preserved (SPEC-006 FR-1.3)
- The API does not define experiment cancellation at MVP scope
- The API does not paginate trial listing endpoints at MVP scope
- The API does not generate routing problems in Generated Mode experiments — routing problem creation in Generated Mode is a CLI responsibility; the mechanism is deferred (OQ-7)

---

# Assumptions

1. PostgreSQL is available and reachable before the API processes its first request. A PostgreSQL unavailability is treated as a transient infrastructure failure.
2. RabbitMQ is available and reachable before the API publishes its first job message. A RabbitMQ unavailability during publication is an infrastructure failure.
3. The default scheduler configuration (SPEC-003 FR-14: `Balanced` mode, equal weights) is seeded into PostgreSQL at API startup before any requests are processed. The API does not lazily create it.
4. The report volume defined in the Docker Compose container topology (architecture.md) is mounted and readable by the API container. The API does not write to the report volume.
5. The routing problem records, job records, and report metadata records written to PostgreSQL by the Worker are readable by the API. Both the API and Worker connect to the same PostgreSQL instance.
6. The `average_vehicle_speed_kmh` field belongs to the routing problem domain, formally defined in SPEC-001 FR-17 per Project Owner Decision ODR-1. The field is required for all routing problem submissions.
7. Routing problems in the MVP are primarily synthetic (SPEC-002). Real-world coordinate submissions are architecturally supported but are not explicitly tested.
8. All `experiment_trials` records for a given experiment are accessible through a single PostgreSQL query using the `experiment_id` foreign key. The trial count for any experiment is small enough that full in-memory evaluation of terminal status (step 7 of FR-21) is safe at MVP scope.
9. A benchmark summary (`benchmark_summaries`) record is always initialized or updated when a member experiment reaches `Completed` status. The API does not need to handle a race condition where two experiments under the same `benchmark_id` complete simultaneously at MVP scope. Concurrent experiment completions under the same `benchmark_id` are not an expected CLI usage pattern at MVP scope; a single CLI session drives one experiment at a time. This is an accepted MVP risk.
10. No authentication or authorization is required at MVP scope. No authentication layer exists at MVP scope — not within the API, and not as an external proxy or gateway sitting in front of it. All API endpoints are accessible without credentials. Multi-tenant security is deferred to post-MVP implementation (README). The MVP is assumed to run in a trusted local development environment.

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
| Job record | PostgreSQL | SPEC-006 FR-4.2 (schema); ODR-6 | `job_id`, `problem_id`, `scheduler_config_id`, `backend_id` (nullable), `status`, `cancellation_requested`, `created_at`, `updated_at`, `completed_at`, `failed_at` | FR-3 (config validation), FR-8 (status response), FR-11 (terminal state check) | Created by API; `status`, `updated_at`, `completed_at`, `failed_at` updated by Worker (SPEC-005); `backend_id` written at job creation, NULL when not supplied (ADR-013, SPEC-012) |
| Solver run record | PostgreSQL | SPEC-006 FR-6.2 | `solver_outcome` and associated fields | FR-8 (status response — `solver_outcome` field for `Completed` jobs) | Written by Worker (SPEC-005); absent for `Failed` and non-terminal jobs |
| Failure record | PostgreSQL | SPEC-006 FR-8 | Failure classification fields including `failure_reason` | FR-8 (status response — `failure_reason` field for `Failed` jobs) | Written by Worker (SPEC-005); absent for `Completed` and non-terminal jobs |
| Report metadata record | PostgreSQL | SPEC-006 FR-9.2 | `report_id`, `job_id`, `report_format`, `generated_at`, `generation_status` | FR-8 (existence check for `report_available`), FR-12 (report discovery response), FR-13 (file path lookup for report serving) | Written by Worker (SPEC-005 FR-17); may be absent if report generation failed or job is not yet terminal |
| Scheduler configuration record | PostgreSQL | SPEC-003 FR-13 | Objective mode and mode parameters | FR-3 (existence check at submission validation), FR-10 (configuration retrieval endpoints) | Written by API at creation time (FR-10) and seeded at startup (SPEC-003 FR-14) |
| Report file | Report volume | SPEC-009 (FR-6, FR-7) | HTML | FR-13 (report retrieval response body) | Written by Report Generator; located via report metadata record lookup — not by direct path construction from caller input |

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
| Job message | RabbitMQ `routing-jobs` | `{job_id, problem_id, scheduler_config_id, backend_id (optional)}` | Consumed by Worker (SPEC-005); `backend_id` included when present in the submission request (ADR-013) |
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

### Experiment Manifest Validation Failure

**Condition:** Experiment manifest submission fails structural or domain validation (missing required fields, unrecognized `workload_set.mode`, invalid `scheduler_config_id`).
**Behavior:** HTTP 400. No experiment record is created. All validation errors are collected and returned in `field_errors`.
**Fallback:** Caller must correct the manifest and resubmit.

---

### Fixed Mode Problem Not Found

**Condition:** One or more `problem_ids` in the experiment manifest's Fixed Mode `workload_set` do not reference existing records in `routing_problems`.
**Behavior:** HTTP 400 with `error_code = EXPERIMENT_CONFIG_INVALID`. All unresolvable identifiers are listed. No experiment record is created.
**Fallback:** Caller must verify the `problem_ids` are correct and that the routing problem records were successfully persisted before experiment submission.

---

### Evidence Collection on Non-Terminal Job

**Condition:** `POST /v1/experiments/{experiment_id}/trials/{trial_id}/collect-evidence` is called for a trial whose associated job has not yet reached a terminal state (`Completed` or `Failed`).
**Behavior:** HTTP 409 with `error_code = JOB_NOT_TERMINAL`. No evidence is read or written.
**Fallback:** Caller (CLI) must verify job status before triggering evidence collection.

---

### Experiment Persistence Failure

**Condition:** PostgreSQL write fails during experiment manifest submission (experiment record or trial record creation).
**Behavior:** HTTP 500. The transaction is rolled back atomically. No partial experiment or trial records are committed.
**Fallback:** Caller should retry the submission. A retry after a clean rollback is safe.

---

### Experiment Auto-Completion Failure

**Condition:** The experiment auto-completion sequence (ExperimentSummary artifact computation or benchmark summary update) fails after the last trial reaches terminal status.
**Behavior:** HTTP 500 is returned from the collect-evidence call. The trial's evidence may have been written before the failure; the idempotency mechanism in FR-21 (already-`Collected` check) ensures a retry does not re-read Evidence Log tables. The experiment may remain in `Running` status if the transition write failed. Operator inspection of PostgreSQL state is required.
**Fallback:** Retry the collect-evidence call. If the trial is already `Collected`, the idempotent path re-runs the all-terminal check and re-attempts the completion sequence.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer | Indirect — API implements fast domain validation rules derived from SPEC-001; these rules must be kept consistent with Core's authoritative C++ validation (ADR-009) |
| API Layer | Yes — SPEC-008 is the primary definition of the API layer behavior |
| Persistence | Yes — API writes routing problem records and job records to PostgreSQL; reads job status and report metadata from PostgreSQL; writes and reads experiment state tables (experiments, experiment_trials, experiment_artifacts, benchmark_manifests, benchmark_summaries) per SPEC-012 FR-19 through FR-23 |
| Solver Runtime | None — API has no direct interaction with solver backends |
| Observability | Yes — API owns `job.submit`, `experiment.submit`, `experiment.evidence_collect`, and `experiment.summarize` spans; introduces API-layer Prometheus metrics |
| Security | Yes — API is the external trust boundary; all caller input is untrusted; authentication/authorization deferred |
| Configuration | Yes — API creates and retrieves scheduler configuration records; seeds default configuration at startup |
| Experiment State | Yes — API is the durable state authority for all experiment lifecycle state (ADR-012 Decision 2); CLI is the orchestration executor but holds no durable state |
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

22. **Unit: Experiment manifest — missing required field** — A `POST /v1/experiments` body with `solver_set` absent returns HTTP 400 with a `field_errors` entry identifying `solver_set`.

23. **Unit: Experiment manifest — Fixed Mode unknown problem_id** — A `POST /v1/experiments` with Fixed Mode `workload_set` containing one valid and one non-existent `problem_id` returns HTTP 400 with `error_code = EXPERIMENT_CONFIG_INVALID` listing the unresolvable identifier; no experiment record is created.

24. **Unit: Experiment manifest — Fixed Mode all problem_ids unresolvable** — A `POST /v1/experiments` with Fixed Mode `workload_set` where all `problem_ids` are absent from `routing_problems` returns HTTP 400 listing all unresolvable identifiers.

25. **Integration: Experiment submission — Fixed Mode happy path** — A valid `POST /v1/experiments` with Fixed Mode returns HTTP 201 with `experiment_id`, `status = "Created"`, `planned_trial_count` matching `|problem_ids| × |solver_set| × repetition_count`, and all trial records in `Pending` trial_status in PostgreSQL.

26. **Unit: Benchmark submission — duplicate benchmark_id** — A second `POST /v1/benchmarks` with the same `benchmark_id` returns HTTP 409 with `error_code = BENCHMARK_ALREADY_EXISTS`; the original manifest is unchanged.

27. **Integration: Experiment status retrieval** — `GET /v1/experiments/{experiment_id}` returns HTTP 200 with `trial_counts` reflecting the current trial_status distribution.

28. **Integration: Experiment status — not found** — `GET /v1/experiments/{unknown_id}` returns HTTP 404 with `error_code = EXPERIMENT_NOT_FOUND`.

29. **Integration: Trial evidence collection — non-terminal job** — `POST /v1/experiments/{id}/trials/{id}/collect-evidence` when the trial's job has `status = Processing` returns HTTP 409 with `error_code = JOB_NOT_TERMINAL`.

30. **Integration: Trial evidence collection — idempotent** — Calling `POST /v1/experiments/{id}/trials/{id}/collect-evidence` twice for a trial with `evidence_status = Collected` returns HTTP 200 on both calls with identical values for all evidence fields in the response body.

31. **Integration: Trial evidence collection — auto-completion** — When the collect-evidence call causes the last pending trial to become terminal, the response body contains `experiment_status = "Completed"` and `GET /v1/experiments/{experiment_id}` reflects `status = "Completed"`.

32. **Unit: Trial evidence collection — no experiment_id on jobs table** — After a trial job is submitted, the `jobs` table record for that `job_id` has no `experiment_id` column. The experiment-to-job linkage exists only in `experiment_trials.job_id`.

33. **Integration: Trial results retrieval** — `GET /v1/experiments/{experiment_id}/trials` returns HTTP 200 with a `trials` array containing one entry per trial, and `total_count` matching `planned_trial_count`.

34. **Integration: Experiment summary — before completion** — `GET /v1/experiments/{experiment_id}/summary` when the experiment is in `Running` status returns HTTP 404 with `error_code = EXPERIMENT_SUMMARY_NOT_FOUND`.

35. **Integration: Experiment summary — after completion** — `GET /v1/experiments/{experiment_id}/summary` after auto-completion returns HTTP 200 with a summary payload containing all fields from SPEC-020 FR-14 Artifact 3.

36. **Integration: Benchmark summary — before any experiment completes** — `GET /v1/benchmarks/{benchmark_id}/summary` before any member experiment reaches `Completed` returns HTTP 404 with `error_code = BENCHMARK_NOT_FOUND`.

37. **Integration: Benchmark summary — after first experiment completes** — `GET /v1/benchmarks/{benchmark_id}/summary` after the first member experiment completes returns HTTP 200.

38. **Integration: Trial submission linkage — success** — `POST /v1/experiments/{id}/trials/{id}/submit` with a valid `job_id` whose `problem_id` matches the trial's `problem_id` returns HTTP 200 with `trial_status = "Submitted"`.

39. **Integration: Trial submission linkage — already submitted** — `POST /v1/experiments/{id}/trials/{id}/submit` for a trial whose `job_id` is already set returns HTTP 409 with `error_code = TRIAL_ALREADY_SUBMITTED`.

40. **Integration: Trial submission linkage — problem mismatch** — `POST /v1/experiments/{id}/trials/{id}/submit` with a `job_id` whose `problem_id` does not match the trial's `problem_id` returns HTTP 409 with `error_code = JOB_PROBLEM_MISMATCH`.

41. **Integration: Trial submission linkage — first trial transitions experiment to Running** — `POST /v1/experiments/{id}/trials/{id}/submit` for the first trial in a `Created` experiment returns `experiment_status = "Running"` in the response body, and `GET /v1/experiments/{id}` reflects `status = "Running"`.

42. **Unit: Trial evidence collection — null job_id** — `POST /v1/experiments/{id}/trials/{id}/collect-evidence` for a trial whose `job_id` is null (trial not yet linked via FR-25) returns HTTP 409 with `error_code = TRIAL_NOT_SUBMITTED`.

43. **Integration: Trial submission linkage — experiment completed** — `POST /v1/experiments/{id}/trials/{id}/submit` for an experiment whose `status = Completed` returns HTTP 409 with `error_code = EXPERIMENT_NOT_ACTIVE`.

44. **Unit: backend_id absent — standard path** — A valid `POST /v1/jobs` submission without `backend_id` returns HTTP 202; the published queue message does not contain a `backend_id` field; the job record has `backend_id = NULL` in PostgreSQL.

45. **Unit: backend_id present — targeted path** — A valid `POST /v1/jobs` submission with `backend_id = "qubo-simulated-annealing"` returns HTTP 202; the published queue message contains `backend_id = "qubo-simulated-annealing"`; the job record stores `backend_id = "qubo-simulated-annealing"` in PostgreSQL.

46. **Unit: backend_id empty string — rejected** — A `POST /v1/jobs` submission with `backend_id = ""` returns HTTP 400 with `error_code = VALIDATION_ERROR` and a `field_errors` entry identifying `backend_id` before any persistence occurs.

47. **Unit: backend_id in job.submit span** — A `POST /v1/jobs` submission with `backend_id` present produces a `job.submit` span that includes the `backend_id` attribute matching the submitted value.

48. **Unit: backend_id absent — span attribute omitted** — A `POST /v1/jobs` submission without `backend_id` produces a `job.submit` span that does not include a `backend_id` attribute.

49. **Integration: Trial submission linkage — backend_id mismatch** — `POST /v1/experiments/{id}/trials/{id}/submit` with a `job_id` whose `backend_id` (non-null) does not match the trial's `backend_id` returns HTTP 409 with `error_code = JOB_BACKEND_MISMATCH`.

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
9. How many experiment manifests are submitted, and what fraction fail Fixed Mode validation?
10. How long does the experiment auto-completion sequence (ExperimentSummary computation, benchmark summary update) take?
11. For a given `experiment_id`, how many trials are in each status at a given moment?
12. How often is `POST /v1/experiments/{id}/trials/{id}/collect-evidence` called for non-terminal jobs (JOB_NOT_TERMINAL rejections)?
13. For a given `job_id`, was the job submitted with a `backend_id` (targeted path) or without (standard path)? Answered by the `backend_id` attribute on the `job.submit` span and the `backend_id` field in the `api.job.submitted` log event.

These are answered by:
- The `job.submit` span (submission latency, validation outcome, scheduler config)
- The `api.status_poll` span (polling frequency, status distribution)
- The `experiment.submit` span (experiment manifest submission latency, Fixed Mode validation outcome, planned_trial_count)
- The `experiment.evidence_collect` span (evidence collection latency, idempotent vs. new collection, auto-completion trigger)
- The `experiment.summarize` span (summary computation latency, benchmark summary update outcome)
- Structured log events at each API stage
- Prometheus metrics: submission totals, rejection totals, poll outcomes, submission duration histogram, experiment submission totals, evidence collection totals, JOB_NOT_TERMINAL rejection totals

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
Structured log events must not include routing problem raw data (geographic coordinate arrays, full stop lists). Job identifiers, stop counts, and outcome codes are safe to log. Consistent with SPEC-001 Security Considerations. For experiment-related log events, `experiment_id` is safe to log, but raw problem data (route coordinates, stop lists) from collected evidence payloads must not appear in log events.

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

**Experiment auto-completion transaction scope:**
When the last trial reaches a terminal state, the collect-evidence endpoint executes an experiment auto-completion sequence that includes: Evidence Log reads (outside the commit transaction), then atomic write of QualityStatsAggregate artifacts, ExperimentSummary artifact, experiment status transition to `Completed`, and benchmark summary upsert. The transaction write scope grows with the number of (problem, solver) pairings in the experiment. At MVP trial counts this is not expected to cause performance issues, but the commit latency for large experiments will be higher than for single-trial evidence collection calls. This must be measured during implementation.

**Connection pooling:**
PostgreSQL and RabbitMQ connection pool sizes are implementation planning concerns. Defaults are assumed sufficient for the MVP single-instance workload.

---

# Documentation Updates Required

- **docs/architecture.md**: The API Responsibilities section accurately describes the API's role at a high level. SPEC-008 is the authoritative binding contract. ADR-013 requires architecture.md updates (Experiment Execution Flow sequence diagram, RabbitMQ section payload description, removal of SPEC-016 OQ-6 from Implementation Blockers); those are architecture.md's governance scope and are not owned by this specification.
- **SPEC-001 (revision complete)**: `average_vehicle_speed_kmh` is formally defined in SPEC-001 FR-17 as a required routing problem field per ODR-1. SPEC-008 FR-2 and FR-3 now reference SPEC-001 FR-17 as the authoritative definition.
- **SPEC-003**: The scheduler configuration endpoint contract defined in SPEC-008 FR-10 is the API's implementation of the configuration persistence model implied by SPEC-003 FR-13. No SPEC-003 changes are required.
- **SPEC-005 FR-12**: The cancellation delivery mechanism is resolved (ODR-5). SPEC-008 FR-11 (API writes `cancellation_requested`) and SPEC-005 FR-12 (Worker reads flag at pre-execution check) are consistent. No further cross-spec updates required for cancellation.
- **SPEC-006 FR-4 (Job Record Persistence Schema)**: Resolved by ODR-6. SPEC-006 FR-4.2 has been updated to add `scheduler_config_id` and `cancellation_requested`, and SPEC-008 FR-4 and FR-8 have been aligned to SPEC-006's authoritative field names (`created_at`, `updated_at`, `completed_at`, `failed_at`). Engineering Review findings B-2 and B-3 are resolved. SPEC-006 is the authoritative schema owner per ODR-6; future job record field additions require a SPEC-006 revision.
- **CLI Specification (pending)**: The Daedalus CLI (README.md) submits jobs through the API. The CLI specification must reference SPEC-008 for the HTTP contract it wraps.
- **SPEC-009 (Report Generator, Accepted)**: SPEC-008 FR-13 serves files written by the Report Generator to the shared report volume. The API is a consumer of the report storage contract; SPEC-009 is the owner. SPEC-009 FR-6 defines the file naming convention (`{report_id}.html`); SPEC-009 FR-7 defines the flat directory storage structure on the report volume; SPEC-009 FR-10 establishes that the report metadata record (SPEC-006 FR-9) stores an explicit `file_path` field from which the API locates the physical file. SPEC-008 FR-13 implementation dependency on the Report Generator Specification is satisfied.
- **ADR-012 (Experiment Execution Architecture, Accepted)**: ADR-012 Decision 2 named "SPEC-008-R1" as the mechanism for experiment API contract definition. SPEC-008 has been amended in place as the revision vehicle for those requirements (FR-18 through FR-24). ADR-012 reference to "SPEC-008-R1" should be understood as this amendment to SPEC-008. No separate SPEC-008-R1 document exists or is required.
- **SPEC-020 (Benchmark and Experiment Harness, Accepted)**: OQ-1 (persistence schema for experiment tables) is resolved by SPEC-012 FR-19 through FR-23. SPEC-020 should be updated to mark OQ-1 as resolved and reference SPEC-012 FR-19 through FR-23.
- **SPEC-020 FR-3 (Experiment Manifest Fields)**: The `seed_derivation_algorithm_version` field (SPEC-012 FR-20.4: `manifest_payload` column) is absent from the SPEC-020 FR-3 manifest field table. SPEC-020 FR-3 should be updated to include `seed_derivation_algorithm_version` as a manifest field.
- **SPEC-016 (CLI Specification, pending)**: The CLI specification must reference SPEC-008 FR-18 through FR-25 for the experiment and benchmark HTTP contracts it wraps. The CLI is the orchestration executor (ADR-012 Decision 1); the CLI specification must document the collect-evidence call pattern, the experiment submission flow, the trial submission linkage call (FR-25), and the polling loop that drives evidence collection.
- **SPEC-020 FR-4 (inconsistency — pre-creation rejection vs. transition-to-Failed):** SPEC-020 FR-4 states that an unresolvable `problem_id` is an experiment configuration error that transitions the experiment to `Failed`. FR-18 specifies pre-creation rejection: submission is rejected before any experiment record is created, so no `Failed` transition occurs. Pre-creation rejection is the preferred behavior (no partial state). SPEC-020 FR-4 should be updated to clarify that unresolvable `problem_ids` cause HTTP 400 rejection before experiment record creation, not a post-creation `Failed` transition.
- **SPEC-020 FR-2 (inconsistency — manifest-must-precede vs. retroactive-is-valid):** SPEC-020 FR-2 first acceptance criterion states that the benchmark manifest must precede any experiment submission under that `benchmark_id`. A later acceptance criterion and FR-19 behavior allow retroactive manifest submission. This is an internal SPEC-020 inconsistency. SPEC-020 FR-2 should be revised to consistently define whether retroactive benchmark manifest submission is supported and under what conditions.

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

### OQ-7: Generated Mode Routing Problem Creation and Trial Record Registration

**This question covers two distinct sub-problems that must both be resolved before Generated Mode harness implementation can proceed.**

**Sub-problem A: Routing problem creation without job creation**

**Question:** In Generated Mode experiments, the CLI must create routing problems before submitting trial jobs. The existing `POST /v1/jobs` endpoint always creates a routing problem and a job atomically. There is no endpoint that creates a routing problem without also creating a job. How does the CLI create routing problems for Generated Mode experiments so that the resulting `problem_ids` can be used in trial job submissions (via the `problem_id` path in FR-2)?

**Why it matters:** The instance-sharing invariant (ADR-012 Decision 3, SPEC-020 FR-4) requires that for each `(problem_config_index, repetition_index)` pair, all backends share one `problem_id`. To implement this, the CLI must first create the routing problem, then submit one job per backend referencing that `problem_id` using the FR-2 `problem_id` path. Currently, `POST /v1/jobs` is the only way to create a routing problem record, and it always creates a job simultaneously.

**Options under consideration:**
1. Add a `POST /v1/routing-problems` endpoint that creates a routing problem without creating a job. The CLI uses this for Generated Mode problem creation; `POST /v1/jobs` continues to serve the single-job (non-experiment) path.
2. Extend `POST /v1/jobs` with an `experiment_trial_context` field that signals the API to create the problem without immediately submitting a job. The first backend submission creates the problem; subsequent backend submissions in the same (problem_config_index, repetition_index) reuse it via the `problem_id` path.
3. Require that Generated Mode experiments use pre-created problem_ids (effectively treating them as Fixed Mode from the API's perspective). The CLI generates the routing problems before experiment submission and submits them as Fixed Mode. This is architecturally equivalent to Fixed Mode with CLI-generated inputs.

**Sub-problem B: Trial record registration after generated problems exist**

**Question:** FR-18 specifies that in Generated Mode, trial records are deferred until the CLI provides generated routing problem identifiers. Once the CLI has created routing problems (Sub-problem A), how does the CLI register the trial records in `experiment_trials`? No endpoint currently exists to create trial records for an existing experiment. Without trial records, the experiment cannot transition to `Running`, the FR-25 trial submission linkage endpoint has no trial records to link, and FR-21 evidence collection has no trial records to write.

**Why it matters:** SPEC-012 FR-21 defines `experiment_trials` with `problem_id NOT NULL`. Trial records must be registered with their `problem_id` before any job can be linked via FR-25. An endpoint or mechanism is required that creates the corresponding `experiment_trials` rows for a Generated Mode experiment after routing problems are known.

**Options under consideration:**
1. Add a `POST /v1/experiments/{experiment_id}/trials` endpoint that accepts a batch of trial registration tuples `(problem_config_index, repetition_index, backend_id, problem_id)`. The CLI calls this after creating routing problems for each `(problem_config_index, repetition_index)` group.
2. Define trial registration as part of the routing problem creation response: the problem creation endpoint (Sub-problem A Option 1) automatically creates trial records in the experiment context when an `experiment_id` is provided.
3. Treat Generated Mode trial record registration as an atomic extension of Sub-problem A Option 2: a single request creates the routing problem and registers trial records for all backends in one operation.

**Owner:** Both sub-problems are blocking for Generated Mode experiment harness implementation. A decision on Sub-problem A constrains the viable options for Sub-problem B. A decision is required before the CLI specification (SPEC-016) can define the Generated Mode execution flow. An ADR-012 review trigger (per ADR-012 Decision 1 note on non-CLI submission) may be required depending on the chosen options.

**Blocking:** Not blocking for Fixed Mode experiments or for MVP harness scope if Generated Mode is deferred. Both sub-problems must be resolved before Generated Mode experiment harness implementation (SPEC-020 FR-4 Mode B) can proceed.

---

### OQ-8: CLI Notification Mechanism for SchedulerRejected and HarnessError Trials

**Question:** For trials where the CLI fails to submit a job (Scheduler rejects before job creation, or harness-level error prevents job_id acquisition), there is no `job_id` on the trial record. The collect-evidence endpoint (`POST /v1/experiments/{experiment_id}/trials/{trial_id}/collect-evidence`) requires a non-null `job_id` to read Evidence Log records. How does the CLI update the trial record to `SchedulerRejected` or `HarnessError` trial_status so that the all-trials-terminal check in FR-21 step 7 can evaluate correctly?

**Why it matters:** The experiment auto-completion logic in FR-21 step 7 checks whether all trials are in a terminal trial_status (`Completed`, `SchedulerRejected`, or `HarnessError`). If some trials are permanently stuck in `Pending` or `Submitted` because the CLI had no mechanism to report a submission-time failure, the experiment will never auto-complete.

**Options under consideration:**
1. Add a `PATCH /v1/experiments/{experiment_id}/trials/{trial_id}` endpoint that allows the CLI to set `trial_status = SchedulerRejected` or `HarnessError` with a `failure_reason` field. The CLI calls this before abandoning the trial.
2. Extend the collect-evidence endpoint body to accept an optional `failure_report` object. If present and `job_id` is null, the endpoint marks the trial as `HarnessError` or `SchedulerRejected` instead of reading Evidence Log records.
3. Have the CLI call the collect-evidence endpoint with a synthetic (pre-agreed) signal in the request body to communicate submission-time failure to the API.

**Owner:** This question is blocking for full experiment harness implementation if any trial can fail before job submission. In the happy path (all jobs submitted successfully), this question does not block MVP harness operation.

**Blocking:** Not blocking for MVP scope if `SchedulerRejected` and `HarnessError` terminal states are treated as out-of-scope for the initial harness implementation. Blocking for complete experiment lifecycle handling.

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
- [x] Experiment manifest submission is defined (FR-18)
- [x] Benchmark manifest submission is defined (FR-19)
- [x] Experiment status retrieval is defined (FR-20)
- [x] Trial submission linkage is defined (FR-25)
- [x] Trial evidence collection trigger is defined (FR-21)
- [x] Trial results retrieval is defined (FR-22)
- [x] Experiment summary retrieval is defined (FR-23)
- [x] Benchmark summary retrieval is defined (FR-24)
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
- [ ] OQ-7 resolved — Generated Mode routing problem creation (Sub-problem A) and trial record registration (Sub-problem B) (blocking for Generated Mode harness)
- [ ] OQ-8 resolved — CLI notification mechanism for SchedulerRejected and HarnessError trials (blocking for full lifecycle handling)
- [x] ADR-013 applied — optional `backend_id` field added to FR-2, FR-3, FR-4, FR-5, FR-7; observability updated in FR-17; testability cases 44–48 added
- [x] POD-1 applied — FR-25 backend identity validation added (step 7); `JOB_BACKEND_MISMATCH` added to FR-14 error codes; testability case 49 added; standard-path null bypass documented in FR-25 acceptance criteria
- [x] POD-2 applied — FR-8 clarifying note added: targeted-path vs standard-path `NoEligibleSolver` distinction is not surfaced in status response; callers requiring this distinction directed to evidence report or FR-21 collect-evidence

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-25) are implemented and acceptance criteria pass
- OQ-1 (cancellation delivery mechanism) is resolved: PostgreSQL `cancellation_requested` flag; API writes, Worker reads at pre-execution check (ODR-5) — satisfied
- OQ-2 (Pending-state cancellation) is resolved: soft-cancel model; Pending jobs remain in queue; Worker cancels at pre-execution check (ODR-5) — satisfied
- OQ-6 (trace context propagation) is resolved and incorporated in FR-17 — satisfied
- OQ-7 Sub-problem A (routing problem creation without job creation) and Sub-problem B (trial record registration) are resolved before Generated Mode harness implementation begins
- OQ-8 (SchedulerRejected/HarnessError trial notification) is resolved before full experiment lifecycle implementation begins
- ADR-013 backend targeting contract (FR-2, FR-3, FR-4, FR-5, FR-7) is implemented: optional `backend_id` accepted at submission, stored in job record, forwarded in queue message; empty-string rejection enforced; no capability registry check performed; `jobs.backend_id ≠ experiment_trials.backend_id` check enforced at FR-25 trial submission linkage; `JOB_BACKEND_MISMATCH` (HTTP 409) returned on mismatch
- All test contracts defined in the Testability section pass (items 1 through 49)
- The `job.submit`, `experiment.submit`, `experiment.trial_submit`, `experiment.evidence_collect`, and `experiment.summarize` spans are emitted and verifiable in the OpenTelemetry Collector
- `job.submit` span includes `backend_id` attribute when present in the submission request
- SPEC-001 FR-17 defines `average_vehicle_speed_kmh` (ODR-1) and SPEC-008 FR-2 references SPEC-001 FR-17
- The `jobs` table has no `experiment_id` column; Worker experiment-unawareness is verified by schema inspection
- The `jobs` table has a nullable `backend_id` column (SPEC-012); existing rows have NULL (standard-path backward compatibility)
- Fixed Mode experiment submission rejects unresolvable `problem_ids` atomically (no partial experiment state)
- Evidence collection is idempotent: repeated calls for a `Collected` trial return HTTP 200 without re-reading Evidence Log tables
- Experiment auto-completes to `Completed` when the last trial reaches terminal trial_status
- Engineering review passes
- Architecture review passes
- Specification status is updated to Verified
