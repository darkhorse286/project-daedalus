# SPEC-005: Worker Execution Lifecycle

## Metadata

**Feature ID:** SPEC-005

**Title:** Worker Execution Lifecycle

**Status:** Proposed

**Author:** Darkhorse286

**Created:** 2026-06-09

**Last Updated:** 2026-06-12

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-001, ADR-003, ADR-004, ADR-006, ADR-008, ADR-009, ADR-010

**Related Specs:** SPEC-001, SPEC-003, SPEC-004, SPEC-006, SPEC-007

---

# Problem Statement

The architecture defines the Daedalus Worker as the C++ execution service responsible for consuming jobs, invoking Core, executing solver backends, enforcing timeouts, persisting results, and generating evidence reports. No specification currently defines the precise lifecycle states, state transitions, behavioral obligations, failure handling, idempotency requirements, or telemetry contract governing the Worker's execution of a routing optimization job.

Without a concrete lifecycle specification:

- The Worker has no defined obligation for when to acknowledge a RabbitMQ message, creating ambiguity about job loss on crash.
- The execution seed derivation ownership is asserted in SPEC-004 FR-11.3 but the policy has no authoritative home.
- Timeout enforcement obligations and the source of the execution time budget have no formal definition.
- Failure classification — which failures cause a job to retry, which dead-letter, and which produce partial evidence — is unspecified.
- Idempotency requirements for at-least-once message delivery have no formal owner.
- The handoff responsibilities between the Worker and the pending Evidence Log Specification are undefined.

SPEC-005 provides the authoritative lifecycle definition for the Daedalus Worker. It closes the gap between the abstract architecture description and the implementable behavior contract.

---

# Business Value

- Eliminates implementation ambiguity for the Worker component and makes it independently implementable
- Establishes the Worker as the authoritative owner of execution seed derivation, closing the gap left open by SPEC-004 FR-11.3
- Defines reliable job execution semantics that prevent silent job loss on Worker crash
- Provides testable acceptance criteria for every stage of the execution lifecycle
- Demonstrates production-grade reliability engineering: explicit failure classification, idempotency, and observable lifecycle states

---

# Employer Signaling

- System Design
- Distributed Systems
- Reliability Engineering
- Observability

---

# Domain Concept

The Worker is the execution orchestrator for routing optimization jobs. It is the sole consumer of the routing-jobs queue. It owns no optimization logic, no scheduling policy, no routing problem validation authority, and no evidence persistence schema. The Worker's responsibility is coordination: it activates the right component at each stage of job execution, enforces execution contracts against solver backends, and ensures every job produces an observable evidence record regardless of solver outcome.

The Worker bridges the asynchronous queue boundary established by ADR-003 with the synchronous solver execution path. The API places a job on the queue and returns immediately; the Worker picks it up and drives it through to completion. Every stage of the execution lifecycle is observable through the OpenTelemetry spans defined in architecture.md and this specification.

The Worker is not an optimizer. It does not interpret routing problems, evaluate solution quality, or adjust scheduling decisions. When the Worker receives a decision from Core, it executes that decision without modification. When the Worker receives a response from a solver backend, it validates it structurally and hands it to Core for quality evaluation without interpretation.

A routing optimization job that completes with a solver outcome of `Infeasible`, `Timeout`, `Cancelled`, or `Failed` is not a Worker failure. The Worker has succeeded in its lifecycle responsibility when it produces a complete evidence record — even if the solver did not find a solution. The Worker fails only when it cannot complete its orchestration responsibilities and cannot produce any evidence for the job.

---

# Requirements

### FR-1: Responsibility Scope and Component Boundaries

**Description:**
SPEC-005 defines the Worker's responsibilities and explicitly allocates the adjacent responsibilities it must not perform.

| Component | Responsibilities at This Boundary | Explicitly Not Responsible For |
|---|---|---|
| **Worker** (SPEC-005) | Job consumption; problem and configuration loading; Core invocation coordination; solver contract invocation; execution seed derivation; execution timeout enforcement; cancellation signal dispatch; SolverResponse post-receipt structural validation; decision record persistence; quality evaluation handoff to Core; evidence persistence handoff; report generation invocation; job lifecycle state management; all required OTel span emissions | Routing problem domain validation; workload feature computation; solver backend selection; solution quality evaluation; regret calculation; evidence persistence schema definition; report content definition |
| **Core** (architecture.md) | Authoritative domain validation (ADR-009); workload feature extraction; Scheduler invocation; quality evaluation and regret calculation; computing `actual_outcome` and `hindsight_quality` values and returning them in `QualityEvaluationResult` | Solver invocation; timeout enforcement; job state management; message acknowledgment |
| **Scheduler** (SPEC-003) | Backend eligibility filtering; candidate scoring; backend selection; decision record production | Execution; persistence; timeout enforcement |
| **Solver Backend** (SPEC-004) | Executing the routing optimization within the SolverRequest contract; self-termination by `execution_timeout_ms`; reporting the honest outcome | Persisting results; evaluating quality; selecting the problem to solve |
| **Evidence Log** (SPEC-006) | Defining the persistence schema for all job execution artifacts | Producing or evaluating any artifact |
| **API** (architecture.md) | Job submission; routing problem validation for fast caller feedback (ADR-009); scheduler config validation; job status endpoint | Solver execution; Core invocation; evidence persistence |

**Acceptance Criteria:**
- No Worker code path performs domain validation, workload feature computation, or quality evaluation
- No Worker code path invokes the Scheduler directly; all Scheduler invocations go through Core
- Adding a new solver backend requires no modifications to Worker lifecycle behavior, only backend registration and backend implementation artifacts
- The Worker produces a structured evidence handoff payload for every completed job (successful or not)

---

### FR-2: Job Lifecycle State Model

**Description:**
Every routing optimization job has a lifecycle state persisted in PostgreSQL. The Worker is the authoritative writer of job lifecycle state after the job transitions from `Pending`. The API sets `Pending` at submission time.

**Job lifecycle states:**

| State | Meaning | Terminal? |
|---|---|---|
| `Pending` | Job message is in the routing-jobs queue; Worker has not yet consumed it | No |
| `Processing` | Worker has consumed the message and execution is in progress | No |
| `Completed` | Worker has finished its lifecycle responsibilities and produced an evidence record. This state is set regardless of solver outcome; `Succeeded`, `Infeasible`, `Timeout`, `Cancelled`, and `Failed` solver outcomes all produce a `Completed` job. | Yes |
| `Failed` | Worker could not complete its orchestration responsibilities before producing any evidence. A structured failure record is persisted. | Yes |

The solver outcome is not encoded in the job lifecycle state. It is recorded on the solver run record (part of the evidence record). A job that reaches `Completed` has an associated solver run record carrying the outcome. A job that reaches `Failed` does not have a solver run record.

**Allowed state transitions:**

```
Pending → Processing    (Worker consumes message from queue)
Processing → Completed  (Worker finishes lifecycle, evidence persisted)
Processing → Failed     (pre-evidence failure: load failure, Core validation rejection, NoEligibleSolver)
```

There is no transition from `Failed` back to `Pending`. A `Failed` job requires manual intervention via the dead-letter queue strategy (FR-13). There is no transition from `Completed` to any other state.

**Worker-internal execution stages:**

The following stages occur within the `Processing` state. They are not persisted as separate job states but are each covered by a named OTel span (FR-19):

1. **Loading** — Worker loads routing problem and scheduler configuration from PostgreSQL
2. **Scheduling** — Worker invokes Core; Core validates, extracts features, runs Scheduler, returns decision record
3. **Persisting-Decision** — Worker persists the Scheduler decision record
4. **Executing** — Worker dispatches SolverRequest; enforces timeout; awaits SolverResponse
5. **Validating-Response** — Worker performs post-receipt structural validation on the SolverResponse
6. **Evaluating** — Worker passes the SolverResponse to Core for quality evaluation; awaits quality result
7. **Persisting-Results** — Worker persists solver run record and quality evaluation
8. **Reporting** — Worker generates the evidence report
9. **Completing** — Worker marks job `Completed`, ACKs the RabbitMQ message

**Acceptance Criteria:**
- The job lifecycle state is updated to `Processing` before any execution work begins
- A job that reaches a terminal state (`Completed` or `Failed`) never transitions again
- A job with `outcome = Infeasible`, `Timeout`, `Cancelled`, or `Failed` at the solver level is marked `Completed` at the job level, not `Failed`
- A job is marked `Failed` only for pre-evidence failures that prevent any solver run record from being produced

---

### FR-3: Job Consumption (RabbitMQ)

**Description:**
The Worker is the sole consumer of the `routing-jobs` queue (ADR-003). It consumes jobs using the AMQP consumer acknowledgment model. One job is processed at a time per Worker instance at MVP scope.

**Message structure:** A routing-jobs message contains at minimum:
- `job_id` (UUID): the unique job identifier, assigned by the API at submission
- `problem_id` (UUID): the routing problem identifier, assigned by the API at submission
- `scheduler_config_id` (UUID): the scheduler configuration identifier, either the submitted value or the default

**Consume behavior:**
1. The Worker receives a message from the `routing-jobs` queue.
2. The Worker does NOT acknowledge the message immediately on receipt. The message remains unacknowledged in the broker until the Worker explicitly ACKs or NACKs it.
3. The Worker transitions the job to `Processing` in PostgreSQL.
4. The Worker proceeds with the execution lifecycle (FR-4 through FR-18).
5. The message is acknowledged (ACK) or negatively acknowledged (NACK) only after the job reaches a terminal state (FR-18).

**ACK strategy — at-least-once delivery:**
The Worker holds the RabbitMQ message acknowledgment until the job reaches a terminal state. If the Worker process terminates before ACKing, the broker redelivers the message and the job is re-executed (subject to FR-14 idempotency requirements). This ensures routing jobs survive Worker crashes at the cost of requiring idempotent execution.

**NACK and dead-letter routing:** Defined in FR-13.

**Acceptance Criteria:**
- The Worker does not ACK a message until the job has reached a terminal state (`Completed` or `Failed`)
- The Worker does not process more than one job per Worker instance concurrently at MVP scope
- A Worker crash before ACK results in message redelivery by the broker; the redelivered message is processed per FR-14 idempotency rules
- The `job_id`, `problem_id`, and `scheduler_config_id` are extracted from the message before any database access occurs; a malformed message that cannot be parsed is rejected (NACK, dead-letter) without modifying any persistent state

---

### FR-4: Problem and Scheduler Configuration Loading

**Description:**
After consuming a message, the Worker loads the routing problem and the scheduler configuration from PostgreSQL before proceeding to Core invocation.

**Loading sequence:**
1. Load the routing problem record by `problem_id`.
2. Load the scheduler configuration record by `scheduler_config_id`.
3. Construct the C++ domain representation of the routing problem from the loaded record (SPEC-001 FR-14). This construction is a Worker responsibility; Core receives the C++ representation, not raw PostgreSQL records.

**Failure handling at loading stage:**
- If PostgreSQL is unavailable: transient failure. The Worker NACKs the message with requeue semantics (FR-13). The job remains in `Processing` state; the status is not updated to `Failed` until the retry limit is reached.
- If the routing problem record does not exist in PostgreSQL: permanent failure. The API should have persisted the record before publishing the message; its absence indicates a data integrity fault. The Worker marks the job `Failed`, records a structured load failure, NACKs the message without requeue (dead-letter), and emits a structured log event.
- If the scheduler configuration record does not exist: permanent failure. Same handling as missing routing problem record.
- If the C++ domain representation cannot be constructed from the loaded record (corrupt record): permanent failure. The Worker marks the job `Failed` and dead-letters the message.

The Worker does not perform domain validation during loading. Loading produces a C++ domain representation and passes it to Core. Core performs authoritative validation (ADR-009).

**Acceptance Criteria:**
- Both the routing problem and the scheduler configuration are loaded from PostgreSQL before Core is invoked
- The C++ domain representation is constructed from the PostgreSQL record, not from the raw JSON submission form
- A PostgreSQL connection failure during loading triggers a NACK with requeue, not an immediate job failure
- A missing record triggers a job failure with a structured failure record identifying the missing entity and its identifier
- The `problem.load` OTel span covers the load and C++ domain representation construction for the routing problem (FR-19)

---

### FR-5: Core Invocation and Scheduler Decision

**Description:**
After loading, the Worker invokes Core with the routing problem and the resolved scheduler configuration. Core is responsible for authoritative domain validation (ADR-009), workload feature extraction, and Scheduler invocation. The Worker does not invoke the Scheduler directly.

**Worker invocation call:**
The Worker passes to Core:
1. The C++ domain representation of the routing problem (constructed in FR-4)
2. The resolved scheduler configuration object (loaded in FR-4)

The Worker does not pass workload features; Core computes them. The Worker does not pass the backend registry; it is accessible within Core's execution context (SPEC-003 Assumption 3).

**Possible Core responses:**

| Response type | Cause | Worker action |
|---|---|---|
| Decision record with `decision_status = Selected` | Scheduler selected a backend | Worker proceeds to FR-6 |
| Core validation rejection | Routing problem failed Core authoritative validation (ADR-009); divergence from API validation | Worker marks job `Failed`; persists a structured validation failure record; dead-letters message |
| Decision record with `decision_status = NoEligibleSolver` | All registered backends failed eligibility filtering | Worker marks job `Failed`; persists the decision record (which carries full rejection reasons per SPEC-003 FR-9); dead-letters message |
| Decision record with `decision_status = InvalidConfiguration` | Scheduler configuration specifies an unrecognized mode or missing required parameters | Worker marks job `Failed`; persists the decision record; dead-letters message |
| Decision record with `decision_status = IncompleteFeaturesError` or `OutOfRangeFeaturesError` | Core contract violation in feature computation | Worker marks job `Failed`; persists the decision record; dead-letters message |
| Infrastructure failure (Core unavailable) | Core process failure | Transient failure; Worker NACKs with requeue |
| Unexpected response type or unclassified error from Core | Core programming error or unknown contract violation | Treated as a transient failure; Worker NACKs with requeue. If the retry limit is exhausted, the message is dead-lettered. Worker persists a structured `CoreContractViolation` failure record before NACKing. |

The Worker emits a `job.consume` span that is the parent of the full execution lifecycle (FR-19). Core emits `features.extract` and `scheduler.score_solvers` as child spans. The Worker does not emit these spans; they are Core's telemetry obligation.

**Acceptance Criteria:**
- The Worker never invokes the Scheduler directly; all Scheduler invocations are initiated by Core
- The Worker passes a resolved scheduler configuration object to Core, not a bare `scheduler_config_id`
- The Worker does not modify the routing problem or the scheduler configuration before passing them to Core
- All non-`Selected` decision outcomes from Core result in job `Failed` with a structured failure record
- A Core infrastructure failure (as distinct from a domain failure) triggers a NACK with requeue, not an immediate permanent failure
- The `features.extract` and `scheduler.score_solvers` spans are emitted by Core; the Worker does not emit them

---

### FR-6: Decision Record Persistence

**Description:**
When Core returns a decision record with `decision_status = Selected`, the Worker persists the decision record to PostgreSQL before invoking the solver backend. This ordering guarantees that every solver invocation has a corresponding decision record in the evidence store, even if the Worker crashes after invocation but before completing the full lifecycle.

When Core returns a non-`Selected` decision record, the Worker also persists it. The decision record is evidence for why the job failed (NoEligibleSolver, InvalidConfiguration, etc.) and must be queryable.

**Persistence obligation:**
- The decision record is persisted atomically: either the full record is written or nothing is written. Partial record persistence is not acceptable.
- The persistence schema is defined by SPEC-006 (Evidence Log). SPEC-005 defines the ordering and ownership obligation; schema details are deferred to SPEC-006.
- The Worker does not modify the decision record after receiving it from Core. The `actual_outcome` and `hindsight_quality` reserved fields (SPEC-003 FR-10) are absent (null) when persisted at this stage; they are populated by Core during quality evaluation (FR-15).

**Acceptance Criteria:**
- The decision record is persisted before the solver is invoked (for `Selected` outcomes)
- The decision record is persisted even when `decision_status != Selected`
- A persistence failure at this stage triggers a NACK with requeue (transient failure), not a permanent job failure
- The persisted decision record contains the `decision_id` that will be included in the SolverRequest as a correlation identifier (SPEC-004 FR-2)

---

### FR-7: Execution Seed Derivation

**Description:**
The Worker is the authoritative owner of the `execution_seed` derivation policy. This ownership was established in SPEC-004 FR-11.3 and is formalized here. No solver backend and no other component derives the `execution_seed`.

**Policy:**
1. The `execution_seed` is derived from the routing problem seed (SPEC-001 FR-6), which is a 64-bit non-negative integer.
2. The derivation applies identically to all registered backends regardless of `backend_id`. There is no per-backend derivation strategy.
3. The derivation is deterministic and does not mix in any additional entropy (system time, process ID, OS random sources, or hardware entropy are prohibited per ADR-010 Decision 4).
4. The derivation produces a `uint64` value as required by SPEC-004 FR-2.
5. The derivation formula is: `execution_seed = RoutingProblem.seed`. No transformation is applied. (ODR-4)

**Derivation timing:**
The `execution_seed` is derived after the Scheduler decision is received and before the SolverRequest is constructed. It is derived once per job execution.

**Reproducibility consequence:**
Because the derivation is deterministic and backend-neutral, any two executions of the same routing problem (same problem seed) will produce the same `execution_seed` for any backend. This is required by the reproducibility invariant (SPEC-004 FR-11) and ADR-010.

**Acceptance Criteria:**
- No solver backend implementation includes execution seed derivation logic
- No other Worker code path (feature extraction invocation, quality evaluation invocation) derives or modifies the execution seed
- Given the same routing problem seed, the Worker produces the same `execution_seed` on every invocation
- The execution seed derivation does not use any entropy source other than the problem seed
- Given the same routing problem, the Worker sets `execution_seed = RoutingProblem.seed`; no additional derivation steps are performed (ODR-4)

---

### FR-8: Contract Version Pre-dispatch Validation

**Description:**
Before constructing the SolverRequest, the Worker reads the `supported_contract_version` field from the selected backend's capability profile (SPEC-003 FR-4) and validates that it matches the contract version the Worker intends to send.

**Validation:**
- The Worker reads `supported_contract_version` from the backend's capability profile.
- The Worker compares it against the current `contract_version` the Worker uses (currently 1 per SPEC-004 FR-14).
- If they match: the Worker constructs and dispatches the SolverRequest.
- If they do not match: this is a Worker configuration error. The Worker must not dispatch the SolverRequest. The Worker marks the job `Failed` with a structured pre-dispatch mismatch error, persists the failure record, and dead-letters the message. The solver backend is not invoked.

This pre-dispatch check is distinct from the post-dispatch `ContractVersionMismatch` failure described in SPEC-004 FR-14, which occurs if the backend rejects the `contract_version` in the SolverRequest. The pre-dispatch check prevents the dispatch from occurring at all when a version mismatch is known ahead of time.

**Acceptance Criteria:**
- The pre-dispatch version check always executes before the SolverRequest is dispatched
- A pre-dispatch mismatch produces a job `Failed` result without invoking the solver backend
- The pre-dispatch mismatch is logged and traced; the failure record identifies the backend, the declared `supported_contract_version`, and the Worker's intended `contract_version`

---

### FR-9: Solver Contract Invocation

**Description:**
After the pre-dispatch version check passes, the Worker constructs a SolverRequest and dispatches it to the selected backend.

**SolverRequest construction (per SPEC-004 FR-2):**

| Field | Source |
|---|---|
| `contract_version` | Worker constant: current value 1 (SPEC-004 FR-14) |
| `routing_problem` | C++ domain representation loaded in FR-4 |
| `execution_timeout_ms` | Derived per the Timeout Budget Policy (OQ-1). Must be a positive integer. |
| `execution_seed` | Derived per FR-7 |
| `job_id` | From the consumed job message (FR-3) |
| `decision_id` | From the persisted decision record (FR-6) |
| `backend_id` | `selected_backend_id` from the Scheduler decision record |

**Dispatch mechanism:**
- C++ in-process backends: invoked as a direct function call within the Worker process. No IPC overhead.
- Python adapter backend: dispatched over the IPC transport defined by ADR-005 (transport contract unresolved; Python adapter dispatch is out of scope for this specification until ADR-005 is resolved).

The dispatch mechanism is backend-specific implementation detail; the SolverRequest structure is identical regardless of dispatch mechanism (ADR-008 Backend Neutrality).

**Awaiting the SolverResponse:**
The Worker blocks awaiting the SolverResponse until either:
- The backend produces a SolverResponse, or
- The external timeout enforcement fires (FR-10).

The Worker does not interpret or route on `backend_id` when processing the SolverResponse. All SolverResponses are processed through the same post-receipt path (FR-11).

**Acceptance Criteria:**
- The Worker constructs a SolverRequest with all required fields populated before dispatch
- The Worker does not branch on `backend_id` in contract handling logic after dispatch (ADR-008 Backend Neutrality)
- The `execution_seed` in the SolverRequest is the value derived in FR-7, not the problem seed
- The `decision_id` in the SolverRequest matches the `decision_id` of the persisted Scheduler decision record
- A SolverRequest with `execution_timeout_ms = 0` is never dispatched; the Worker must ensure the timeout budget policy produces a value > 0 before dispatch

---

### FR-10: Execution Timeout Enforcement

**Description:**
The Worker enforces the execution time budget externally, independent of whether the backend self-terminates.

**Behavior:**
1. The Worker starts a timer when the SolverRequest is dispatched.
2. If the backend produces a SolverResponse before the timer fires, the timer is cancelled and the Worker proceeds with the response.
3. If the timer fires before the backend responds: the Worker terminates the backend and constructs a Timeout SolverResponse on the backend's behalf (per SPEC-004 Failure Modes — ExecutionTimeout Worker-Enforced).

**Worker-constructed Timeout response fields (per SPEC-004 Failure Modes):**
- `contract_version`: the value sent in the SolverRequest
- `outcome`: `Timeout`
- `failure`: present with `failure_code = ExecutionTimeout`
- `solution`: absent
- `statistics.execution_duration_ms`: the Worker's measured wall-clock time from SolverRequest dispatch to backend termination

**Timer duration:**
The timer duration equals `execution_timeout_ms` from the SolverRequest. This is the same value the backend received as its self-termination deadline.

**Backend termination mechanism for Worker-enforced timeout:**
The termination mechanism is backend-specific and is an implementation planning concern. For C++ in-process backends, it may be a thread interrupt or cooperative cancellation flag. For the Python adapter process, termination mechanism depends on the ADR-005 transport contract. The mechanism must not leave the Worker process in an inconsistent state.

**Relationship to self-terminating backends:**
If the backend self-terminates before `execution_timeout_ms` expires and returns a Timeout SolverResponse, the Worker accepts it normally; the Worker's external timer is not fired. The Worker validates the self-produced Timeout response through the same post-receipt validation path (FR-11) as any other response.

**Acceptance Criteria:**
- The Worker's external timeout fires if and only if the backend has not produced a SolverResponse by `execution_timeout_ms` milliseconds after dispatch
- A Worker-constructed Timeout response contains all fields specified above
- A Worker-constructed Timeout response carries no `solution` field (the in-progress backend state is not recoverable by the Worker)
- The `execution_duration_ms` in a Worker-constructed response is the Worker's measured elapsed time from dispatch to termination, not a backend-reported value
- The Worker's backend termination operation completes within a bounded time after the external timeout fires. If the backend does not terminate within this bounded window, the Worker proceeds to construct the Timeout SolverResponse and continues the lifecycle. The specific termination timeout value and mechanism are implementation planning concerns; the requirement for boundedness is not.

---

### FR-11: SolverResponse Post-receipt Validation

**Description:**
The Worker validates every received SolverResponse structurally before passing it to Core for quality evaluation. This validation applies to both backend-produced responses and Worker-constructed responses (which the Worker has already ensured are correct by construction).

**Validation checks:**
1. `contract_version` echo: the response's `contract_version` must equal the `contract_version` in the corresponding SolverRequest. A mismatch is `ContractVersionMismatch`.
2. RoutePlan structural validity (when `solution` is present): verify SPEC-004 FR-5 requirements:
   - `routes.size()` equals `vehicle_count` from the routing problem
   - Every stop ID in every route is a valid stop ID from the routing problem
   - No stop ID appears in more than one route
   - Every stop ID appears in exactly one route
   - No route's total demand exceeds `capacity_per_vehicle`
3. Presence constraints: `failure` must be present when `outcome != Succeeded`; `failure` must be absent when `outcome = Succeeded` (SPEC-004 FR-3).

**On structural violation:**
If any validation check fails, the Worker treats the invocation as a solver backend contract violation. The Worker does not discard the record silently: it produces a structured violation record, records the violation in the evidence log, marks the solver run as `ContractViolation`, and proceeds to evidence persistence. The job still reaches `Completed`. This is not a Worker lifecycle failure.

The Worker does not attempt to correct or compensate for a structurally invalid RoutePlan. Core quality evaluation is not invoked if the RoutePlan is structurally invalid.

**Acceptance Criteria:**
- Post-receipt validation runs on every SolverResponse before Core quality evaluation is invoked
- A structurally invalid RoutePlan is recorded as a contract violation; the job still completes with a `Completed` status and a structured contract violation record
- A `contract_version` echo mismatch is treated as `ContractVersionMismatch`
- Worker-constructed responses are not re-validated (they are correct by construction), but the Worker must ensure they conform before constructing them

---

### FR-12: Cancellation Handling

**Description:**
The Worker may receive an external cancellation signal directing it to terminate the current solver execution before the timeout deadline. The cancellation signal mechanism — how the cancellation intent reaches the Worker — is an open question (OQ-2).

**Worker behavior on receiving a cancellation signal:**
1. The Worker signals the backend to terminate using a backend-appropriate mechanism (implementation planning).
2. The Worker awaits the backend's Cancelled SolverResponse. If the backend does not respond to the cancellation signal within a bounded time, the Worker applies Worker-enforced termination (FR-10 procedure) and constructs a Cancelled response.
3. If the backend produces a Cancelled SolverResponse with a solution present, the Worker validates the solution structurally (FR-11) and passes it to Core for quality evaluation (FR-15).
4. If the backend produces a Cancelled SolverResponse with no solution, Core quality evaluation is not invoked (no route plan to evaluate).

**Worker-constructed Cancelled response fields:**
- `contract_version`: the value sent in the SolverRequest
- `outcome`: `Cancelled`
- `failure`: present with `failure_code = ExecutionCancelled`
- `solution`: absent
- `statistics.execution_duration_ms`: the Worker's measured wall-clock time from dispatch to construction

**Cancellation is not a Worker lifecycle failure:**
A Cancelled outcome results in a `Completed` job with a Cancelled solver outcome in the evidence record. The job is not marked `Failed`.

**Acceptance Criteria:**
- The Worker responds to a cancellation signal during the `Executing` stage only; a cancellation signal received outside this stage is logged and ignored
- A Worker-constructed Cancelled response is produced when the backend does not produce its own Cancelled response within a bounded time after the cancellation signal
- A Cancelled outcome results in job `Completed`, not `Failed`
- The cancellation signal mechanism (OQ-2) is defined before implementation begins

---

### FR-13: Retry and Dead-Letter Behavior

**Description:**
Not all Worker failures warrant dead-lettering. The Worker classifies failures into transient failures (retry via requeue) and permanent failures (dead-letter without requeue). This classification determines the NACK policy.

**Transient failures (NACK with requeue):**
The following failures are transient: the condition may resolve without changing the job or its routing problem. The Worker NACKs the message with requeue=true, allowing the broker to redeliver.

| Failure condition | Stage | Rationale |
|---|---|---|
| PostgreSQL unavailable during problem/config load | FR-4 | Infrastructure is down; retry when it recovers |
| PostgreSQL unavailable during decision record persistence | FR-6 | Infrastructure is down; retry when it recovers |
| PostgreSQL unavailable during result persistence | FR-16 | Infrastructure is down; retry when it recovers |
| Core process unavailable (infrastructure failure, not domain failure) | FR-5 | Infrastructure is down; retry when it recovers |

**Permanent failures (NACK without requeue, dead-letter):**
The following failures are permanent: re-executing the same job with the same inputs will produce the same failure.

| Failure condition | Stage | Rationale |
|---|---|---|
| Routing problem record not found in PostgreSQL | FR-4 | Data integrity fault; record was never written or was deleted |
| Scheduler configuration record not found in PostgreSQL | FR-4 | Configuration reference is invalid |
| Corrupt routing problem record; C++ representation cannot be constructed | FR-4 | Data is irrecoverably corrupt |
| Malformed job message (missing required fields, cannot be parsed) | FR-3 | Message is permanently invalid |
| Core validation rejection (ADR-009) | FR-5 | Routing problem is permanently invalid per Core authoritative validation |
| `NoEligibleSolver` from Scheduler | FR-5 | No backend can handle this problem+configuration combination |
| `InvalidConfiguration` from Scheduler | FR-5 | Scheduler configuration is permanently invalid |
| Pre-dispatch contract version mismatch | FR-8 | Worker configuration is inconsistent with backend registration |
| Retry limit exhausted for a transient failure | Any | Maximum NACK+requeue count reached (broker-configured) |

**Post-execution outcomes (not failures; no NACK — see FR-18):**
Solver execution outcomes of `Infeasible`, `Timeout`, `Cancelled`, `Failed` (including `InternalError` and `ContractVersionMismatch` post-dispatch), and Worker-constructed responses are not permanent failures at the job level. They all produce evidence records. The job reaches `Completed`. The Worker ACKs the message.

**Maximum retry count:**
The maximum number of requeue attempts before the broker routes to dead-letter is a RabbitMQ configuration parameter, not a Worker behavior. The Worker always NACKs with requeue for transient failures; the broker enforces the retry limit. This is an implementation planning concern (OQ-4).

**Acceptance Criteria:**
- Transient failure conditions result in NACK with requeue=true; permanent failure conditions result in NACK with requeue=false (dead-letter routing)
- A permanent failure always produces a structured failure record in PostgreSQL before the NACK is issued, identifying the failure type, the job_id, and the failure reason
- Solver execution outcomes (Infeasible, Timeout, Cancelled, Failed) do not trigger NACK; the Worker ACKs after evidence is persisted

---

### FR-14: Idempotency Requirements

**Description:**
Because the Worker uses at-least-once message delivery (FR-3), a job may be executed more than once if the Worker crashes before ACKing. All Worker persistence operations must be idempotent so that re-execution of the same job produces the same result without creating duplicate records or corrupting existing state.

**Idempotency check on message receipt:**
When the Worker receives a message, it checks the job's current lifecycle state in PostgreSQL before beginning execution:
1. If the job is in state `Completed` or `Failed` (already terminal): the Worker ACKs the message and discards it without re-executing. This handles the case where the Worker completed the job and persisted the terminal state but crashed before ACKing the message.
2. If the job is in state `Processing`: the job is in a non-terminal state from a prior execution attempt. The Worker re-executes from the beginning (see below).
3. If the job is in state `Pending` or not found: this is the first execution. The Worker proceeds normally.

**Re-execution from the beginning:**
When re-executing a job from `Processing` state, the Worker treats it as a fresh execution. It loads the problem and configuration, invokes Core, constructs the SolverRequest, and invokes the solver again. The execution produces the same result as the original because:
- `execution_seed` derivation is deterministic from the problem seed (FR-7)
- Solver execution is deterministic from `execution_seed` (SPEC-004 FR-11 reproducibility invariant, when both executions produce `Succeeded`)
- Core quality evaluation is deterministic
- Persistence operations use upsert semantics (see below)

**Idempotent persistence operations:**
All Worker write operations to PostgreSQL must use upsert semantics (insert-or-update) rather than insert-only semantics. `job_id` is the natural idempotency key for all Worker persistence operations. A second write of the same record for the same `job_id` updates the existing record rather than creating a duplicate. The specific upsert mechanism is an implementation planning concern; the behavioral requirement is that a second write for the same `job_id` produces no duplicate records.

A re-execution on message redelivery may produce a new `decision_id` from Core (the Scheduler generates a new identifier on each invocation). This is expected: the new decision record replaces the prior attempt's record via `job_id`-keyed upsert. The `decision_id` is a correlation identifier within a single execution attempt, not a stable identifier across re-executions.

**Acceptance Criteria:**
- On receipt of a message for a job already in `Completed` or `Failed` state, the Worker ACKs and discards without modifying any state
- On receipt of a message for a job in `Processing` state, the Worker re-executes from the beginning using idempotent operations
- No Worker persistence operation creates duplicate records when called more than once for the same `job_id`
- The idempotency check reads the job state from PostgreSQL before transitioning to `Processing`

---

### FR-15: Core Quality Evaluation Handoff

**Description:**
After the SolverResponse passes post-receipt validation (FR-11), the Worker passes the solution to Core for quality evaluation and regret calculation. This step is the Worker's responsibility to invoke; the evaluation logic belongs to Core.

**Worker inputs to Core quality evaluation:**
- The SolverResponse (complete, including `outcome`, `solution` RoutePlan, `statistics`, `failure_detail`, and `extension_metadata`) — SPEC-007 FR-3
- The routing problem (for time window feasibility verification and regret context)
- The Scheduler decision record (for regret context: predicted vs. actual)
- The average vehicle speed (`RoutingProblem.average_vehicle_speed_kmh` per ODR-1); required for route simulation — SPEC-007 FR-3

**Core quality evaluation produces (`QualityEvaluationResult` — SPEC-007 FR-9):**
- `actual_outcome` (`ActualOutcomeClassification`, SPEC-007 FR-4): classification of the actual solver outcome; input-derived subfields (`solver_outcome`, `solution_present`, `time_window_constrained`) are always non-null on any invocation, including infrastructure failure (SPEC-007 FR-13)
- `hindsight_quality` (float64, SPEC-007 FR-7): total route distance in km; null when simulation could not be completed
- `quality_metrics` (`QualityMetrics`, SPEC-007 FR-6): route quality metrics; null on infrastructure failure
- `regret_analysis` (`RegretAnalysis`, SPEC-007 FR-8): actual vs. predicted quality, cost, and latency
- `evaluation_metadata` (`EvaluationMetadata`, SPEC-007 FR-12): evaluation provenance

**When quality evaluation is invoked:**
- `outcome = Succeeded`: always invoke; RoutePlan is present and structurally valid
- `outcome = Timeout` or `Cancelled` with solution present: invoke; RoutePlan has passed structural validation
- `outcome = Infeasible`, `Failed`, `Timeout` with no solution, `Cancelled` with no solution: do not invoke quality evaluation; there is no route plan to evaluate. Regret calculation in the absence of a solution is defined by SPEC-007 (Core Quality Evaluation, Accepted).

**Failure handling:**
If Core quality evaluation fails due to an infrastructure failure (Core process unavailable, unclassified error), Core returns a partial `QualityEvaluationResult` per SPEC-007 FR-13. The Worker writes this partial result to the evidence record: input-derived `actual_outcome` subfields (`solver_outcome`, `solution_present`, `time_window_constrained`) are non-null; simulation-derived `actual_outcome` subfields and computed quality metrics (`hindsight_quality`, `quality_metrics`) are null. The quality evaluation record is persisted with `evaluation_status = Failed` per SPEC-006 FR-7.4. The job proceeds to FR-16 and FR-17. The failure is captured in the `result.evaluate` span status as Error and in a structured log event (`worker.quality.evaluation.failed`). The Worker does not re-invoke the solver. Retrying quality evaluation specifically — without re-executing the solver — is deferred to implementation planning and is not required at MVP scope.

**Dependency:**
The interface between the Worker and Core for quality evaluation — what is passed and what is returned — is defined by SPEC-007 (Core Quality Evaluation, Accepted). SPEC-007 FR-9 defines the `QualityEvaluationResult` returned to the Worker; SPEC-007 FR-10 establishes the determinism invariant. This specification establishes the Worker's obligation to invoke the evaluation; SPEC-007 is authoritative for evaluation interface details.

**Acceptance Criteria:**
- Core quality evaluation is invoked for every SolverResponse that includes a structurally valid RoutePlan
- Core quality evaluation is not invoked when no RoutePlan is present in the response
- The Worker does not perform quality evaluation or regret calculation itself; it delegates entirely to Core
- The quality evaluation result and the updated decision record fields (`actual_outcome`, `hindsight_quality`) are persisted in FR-16

---

### FR-16: Evidence Persistence Handoff

**Description:**
After quality evaluation is complete (or skipped per FR-15), the Worker persists all execution artifacts to PostgreSQL. SPEC-006 (Evidence Log) defines the persistence schema. This specification defines the Worker's persistence obligations and the artifact set.

**Artifacts the Worker must persist:**
1. Updated job lifecycle state (to `Completed` or `Failed`)
2. Scheduler decision record (including `actual_outcome` and `hindsight_quality` if populated by Core)
3. Solver run record, including:
   - All SolverResponse fields: `outcome`, `statistics`, `failure` (if present), `extension_metadata` (if present)
   - `execution_seed` used in the SolverRequest (for reproducibility verification)
   - `backend_id` of the invoked backend
   - `contract_version` used
   - Structural validation result (if a violation was detected in FR-11)
4. Quality evaluation result (if Core quality evaluation was invoked)
5. If a pre-execution failure occurred: a structured failure record identifying the failure type and context

**Persistence ordering:**
Evidence artifacts are persisted together as an atomic unit where possible. If atomicity is not achievable across all artifacts (for example, if some require different PostgreSQL tables), partial persistence must be recoverable through idempotent re-execution (FR-14).

**Dependency:**
The schema for each artifact is defined by SPEC-006 (Evidence Log). SPEC-005 does not define column names, table names, or data types. SPEC-006 has a dependency on SPEC-005 for the artifact set definition and specifies that all artifact tables support `job_id`-keyed upsert semantics as the idempotency mechanism, consistent with FR-14.

**Acceptance Criteria:**
- All listed artifacts are persisted before the report generation stage (FR-17)
- A persistence failure at this stage triggers a NACK with requeue (transient failure), not an immediate permanent job failure
- The solver run record always includes `execution_seed` so that deterministic replay is possible
- The `actual_outcome` and `hindsight_quality` fields on the decision record are persisted with whatever value Core provided; `actual_outcome` is null only when Core quality evaluation was not invoked (no RoutePlan present); on infrastructure failure, input-derived `actual_outcome` subfields are non-null per SPEC-007 FR-13

---

### FR-17: Report Generation

**Description:**
After evidence is persisted, the Worker invokes the report generator to produce an HTML evidence report for the job.

**Worker obligations:**
1. Invoke the report generator with the persisted evidence record for this job (job_id, decision record, solver run record, quality evaluation result).
2. The report generator writes the HTML report to the report volume (per architecture.md container topology).
3. The Worker emits the `report.generate` span covering the report generation call (FR-19).

**Failure handling:**
If report generation fails, the Worker logs a structured error and proceeds to job completion (FR-18) without marking the job `Failed`. The evidence has already been persisted; the job is not retried because of a report generation failure. The report generation failure is observable through the span status and structured logs.

**The report content and format** are defined by the Report Generator component specification (not yet authored). SPEC-005 defines only that the Worker is the invoker.

**Acceptance Criteria:**
- Report generation is invoked for every job that reaches the reporting stage, regardless of solver outcome
- A report generation failure does not change the job's terminal state from `Completed` to `Failed`
- A report generation failure is logged with a structured error event and captured in the `report.generate` span status

---

### FR-18: Job Completion and Message Acknowledgment

**Description:**
After the report is generated (or report generation has been attempted), the Worker performs the final job completion sequence.

**Completion sequence:**
1. Update job lifecycle state to `Completed` in PostgreSQL.
2. Emit the `job.complete` OTel span (FR-19) with the terminal state and solver outcome.
3. ACK the RabbitMQ message.

**ACK timing:**
The RabbitMQ ACK is issued as the last step of the completion sequence, after the job state is updated to `Completed` in PostgreSQL. This ordering guarantees that a Worker crash between state update and ACK causes a redelivery that is handled idempotently (FR-14: the job is already in a terminal state, the Worker ACKs and discards).

**ACK for permanent failures:**
When a permanent failure causes the job to transition directly to `Failed` (FR-13), the Worker:
1. Persists the structured failure record.
2. Updates the job lifecycle state to `Failed`.
3. Emits the `job.complete` span with the failure status.
4. Issues a NACK with requeue=false (dead-letter routing).

There is no ACK for a permanently failed job; the NACK routes it to the dead-letter queue.

**Acceptance Criteria:**
- The RabbitMQ ACK is the last operation in the completion sequence for a `Completed` job
- The job state in PostgreSQL is `Completed` before the ACK is issued
- A crash between the PostgreSQL update and the ACK produces a redelivery that is correctly identified as a terminal-state job and discarded
- The `job.complete` span is emitted before the ACK is issued

---

### FR-19: Telemetry

**Description:**
The Worker is responsible for emitting the required OpenTelemetry spans for the stages of the execution lifecycle it owns. The Worker does not emit spans for stages owned by Core.

**Span ownership:**

| Span | Owner | When emitted |
|---|---|---|
| `job.consume` | Worker | Covers the full job execution lifecycle from message receipt to ACK/NACK |
| `problem.load` | Worker | Covers FR-4: problem and config load from PostgreSQL |
| `features.extract` | Core | Emitted by Core during FR-5 |
| `scheduler.score_solvers` | Scheduler (via Core) | Emitted by Core/Scheduler during FR-5; defined in SPEC-003 FR-15 |
| `solver.execute` | Worker | Covers FR-9/FR-10: SolverRequest dispatch through SolverResponse receipt; defined in SPEC-004 FR-15 |
| `result.evaluate` | Worker | Covers FR-15: Core quality evaluation handoff |
| `worker.evidence.persist` | Worker | Covers FR-16: evidence artifact persistence to PostgreSQL; required by SPEC-006 FR-18 |
| `report.generate` | Worker | Covers FR-17: report generation |
| `job.complete` | Worker | Covers FR-18: terminal state transition and ACK |

**Worker-defined span attribute requirements:**

**`job.consume`** (parent span for full lifecycle):
| Attribute | Description |
|---|---|
| `job_id` | Job identifier |
| `problem_id` | Routing problem identifier |
| `scheduler_config_id` | Scheduler configuration identifier |
| `terminal_state` | `Completed` or `Failed` |
| `solver_outcome` | `SolverOutcome` value from the solver run record; absent if job reached `Failed` before solver invocation |

**`problem.load`**:
| Attribute | Description |
|---|---|
| `job_id` | Job identifier |
| `problem_id` | Routing problem identifier |
| `stop_count` | Number of stops in the loaded routing problem |
| `vehicle_count` | Vehicle count |
| `seed` | Problem seed value |
| `outcome` | `Success` or `Failure` |

**`result.evaluate`**:
| Attribute | Description |
|---|---|
| `job_id` | Job identifier |
| `decision_id` | Scheduler decision identifier |
| `solver_outcome` | `SolverOutcome` value from the solver run |
| `evaluation_invoked` | Boolean: whether Core quality evaluation was invoked |
| `outcome` | `Success` or `Failure` |

**`report.generate`**:
| Attribute | Description |
|---|---|
| `job_id` | Job identifier |
| `outcome` | `Success` or `Failure` |

**`job.complete`**:
| Attribute | Description |
|---|---|
| `job_id` | Job identifier |
| `terminal_state` | `Completed` or `Failed` |
| `solver_outcome` | As above |

**`worker.evidence.persist`**:
| Attribute | Description |
|---|---|
| `job_id` | Job identifier |
| `artifact_types_written` | List of artifact types successfully written |
| `outcome` | `Success` or `Failure` |

**`solver.execute`** is defined in SPEC-004 FR-15. The Worker emits it per that definition.

**`worker.evidence.persist`** observability requirements are declared in SPEC-006 FR-18. This specification is the authoritative definition of span attributes and status rules.

**Span context propagation:**
All Worker spans are children of `job.consume`. Core-emitted spans (`features.extract`, `scheduler.score_solvers`) must propagate the trace context received from the Worker so they appear as children of `job.consume` in the trace. The specific context propagation mechanism is an implementation planning concern.

**Span status rules for Worker-owned spans:**

| Span | Status on success | Status on failure |
|---|---|---|
| `job.consume` | OK when `terminal_state = Completed`; Error when `terminal_state = Failed` | Error |
| `problem.load` | OK | Error |
| `result.evaluate` | OK | Error |
| `worker.evidence.persist` | OK | Error |
| `report.generate` | OK | Unset (report generation failure does not propagate to job failure) |
| `job.complete` | OK when `terminal_state = Completed`; Error when `terminal_state = Failed` | Error |

**Acceptance Criteria:**
- All listed Worker-owned spans are emitted on every job execution, including failed jobs
- All required attributes are present on every span emission
- `solver.execute` is emitted per SPEC-004 FR-15; no attributes are added beyond that definition
- Core-emitted spans appear as children of `job.consume` in distributed traces via context propagation
- `report.generate` span status is `Unset` on report generation failure (not `Error`), preserving the `job.consume` parent span's `OK` status for a `Completed` job that had a report generation failure

---

# Non-Requirements

- The Worker does not perform domain validation (ADR-009; Core responsibility)
- The Worker does not compute workload features (Core responsibility)
- The Worker does not evaluate solver eligibility or score candidates (Scheduler/Core responsibility)
- The Worker does not evaluate route quality or calculate regret (Core responsibility; interface defined by SPEC-007, Core Quality Evaluation)
- The Worker does not define the evidence persistence schema (Evidence Log Specification responsibility)
- The Worker does not define the HTML report content or format (Report Generator component responsibility)
- The Worker does not select which solver to invoke; it executes the Scheduler's decision exactly as given
- The Worker does not retry a solver invocation with a different backend if the first invocation fails or times out; retry decisions are the caller's responsibility (SPEC-003 FR-9)
- The Worker does not implement multi-concurrency at MVP scope; one job is processed at a time per Worker instance
- The Worker does not implement load balancing across multiple Worker instances; that is an infrastructure concern
- The Worker does not own the Python adapter transport contract (ADR-005 deferred; the Worker invokes the Python adapter over whatever transport ADR-005 defines, but the transport definition is not a Worker responsibility)
- The Worker does not define the dead-letter queue reprocessing strategy; that is an operational concern addressed after the dead-letter queue is populated (ADR-003)

---

# Assumptions

1. The API persists the routing problem record to PostgreSQL before publishing the job message to RabbitMQ (SPEC-001 FR-12). The Worker assumes the record exists when it loads by `problem_id`.
2. The `scheduler_config_id` in the job message references an existing scheduler configuration record. The API validates this at submission time (SPEC-001 FR-13). If the record is missing at Worker load time, it is treated as a permanent failure (FR-4).
3. Core is available as an in-process C++ library within the Worker process, not as a separate network service. Core invocation has no network latency or IPC overhead at MVP scope.
4. The backend registry is populated before the Worker processes its first job. An empty registry is a configuration fault, not a condition the Worker compensates for.
5. MVP solver backends (nearest-neighbor, greedy insertion, QUBO simulated annealing) are C++ in-process implementations invoked via direct function call. The Python adapter backend dispatch is not in scope until ADR-005 is resolved.
6. At most one Worker instance is running at any given time at MVP scope. Multi-Worker concurrency and job ownership protocols are deferred.
7. SPEC-006 (Evidence Log, Proposed) defines a persistence schema compatible with the artifact set described in FR-16. SPEC-006 must be accepted before SPEC-005 implementation is complete.
8. SPEC-007 (Core Quality Evaluation, Accepted) defines the Worker-invocable quality evaluation interface compatible with the handoff described in FR-15. The `QualityEvaluationResult` interface (SPEC-007 FR-9) satisfies this assumption.
9. Core quality evaluation is deterministic — given the same routing problem, route plan, solver response, scheduler decision, and travel speed, Core produces identical quality evaluation results on every invocation. The idempotency model defined in FR-14 depends on this property. This guarantee is established by SPEC-007 FR-10 (Accepted).

---

# Constraints

1. The Worker must not modify the routing problem or the Scheduler decision record (ADR-009 validation authority; SPEC-003 Backend Neutrality).
2. The `execution_seed` must be derived from the problem seed using only deterministic operations with no additional entropy (ADR-010 Decision 4; FR-7).
3. The Worker must use at-least-once message delivery semantics (ADR-003); ACK is held until terminal state is reached.
4. All Worker persistence operations must be idempotent (FR-14); upsert semantics are required.
5. The Worker must not branch on `backend_id` in contract handling, response processing, or result handoff logic (ADR-008 Backend Neutrality).
6. Adding a new solver backend must not require modification of any Worker lifecycle behavior (ADR-008 Backend Neutrality; FR-1 Acceptance Criteria).
7. The Worker must emit all required OTel spans on every job execution, including failures (ADR-006; FR-19).
8. The Worker must be implemented in C++ (ADR-001).
9. OTel span emission must use non-blocking export. If the OpenTelemetry Collector is unavailable or the export buffer is full, spans are dropped. Span emission failure must never cause the Worker to halt job processing or transition a job to `Failed`.

---

# Inputs

| Input | Source | Format | Notes |
|---|---|---|---|
| Job message | RabbitMQ `routing-jobs` queue | AMQP message with `job_id`, `problem_id`, `scheduler_config_id` fields | At-least-once delivery |
| Routing problem record | PostgreSQL (loaded by Worker) | PostgreSQL record; Worker constructs C++ domain representation (SPEC-001 FR-14) | Must exist before message is consumed |
| Scheduler configuration | PostgreSQL (loaded by Worker) | Scheduler configuration object (SPEC-003 FR-13) | Resolved before Core invocation |
| Scheduler decision record | Core (produced during FR-5) | SPEC-003 FR-10 decision record schema | Produced for every invocation, successful or failed |
| SolverResponse | Solver backend (produced during FR-9/FR-10) | SPEC-004 FR-3 SolverResponse schema | Always produced; Worker constructs fallback if backend is unresponsive |
| Quality evaluation result | Core (produced during FR-15) | SPEC-007 FR-9 (QualityEvaluationResult) | Absent if solver produced no RoutePlan |

---

# Outputs

| Output | Consumer | Format | Notes |
|---|---|---|---|
| Updated job lifecycle state | PostgreSQL (read by API for status polling) | Job status enum: `Pending`, `Processing`, `Completed`, `Failed` | Updated at each transition |
| Persisted decision record | PostgreSQL (read by Evidence Log, Report Generator) | SPEC-003 FR-10 schema | Includes `actual_outcome` and `hindsight_quality` after quality evaluation |
| Persisted solver run record | PostgreSQL (read by Evidence Log, Report Generator) | Per FR-16 artifact list | Always present for `Completed` jobs; absent for `Failed` jobs |
| Evidence report | Report volume (served by API) | HTML (format defined by Report Generator specification) | One report per `Completed` job |
| OTel spans | OpenTelemetry Collector | Traces per FR-19 | All required spans plus `solver.execute` per SPEC-004 FR-15 |
| Structured log events | Stdout (JSON) | Per ADR-006 | At each lifecycle stage |
| RabbitMQ ACK or NACK | RabbitMQ broker | AMQP acknowledgment | ACK on `Completed`; NACK on permanent failure |

---

# Failure Modes

### Problem or Configuration Load Failure

**Condition:** PostgreSQL unavailable, routing problem record missing, scheduler configuration record missing, or corrupt record preventing C++ representation construction.
**Behavior:** Worker marks job `Failed` (for missing/corrupt records) or issues NACK with requeue (for infrastructure unavailability). Structured failure record persisted for permanent failures.
**Fallback:** Dead-letter queue for permanent failures. Manual investigation required.

---

### Core Validation Rejection (ADR-009 Divergence)

**Condition:** Core rejects the routing problem as invalid after the API accepted it.
**Behavior:** Job transitions to `Failed`. Structured validation error recorded. Decision record persisted with `decision_status` reflecting the failure. Message dead-lettered.
**Fallback:** None. This is a detectable fault between API and Core validation implementations (ADR-009 Consequence). Manual investigation required.

---

### NoEligibleSolver

**Condition:** Scheduler finds no backend eligible for the problem and configuration combination.
**Behavior:** Job transitions to `Failed`. The `NoEligibleSolver` decision record (carrying per-backend rejection reasons) is persisted. Message dead-lettered.
**Fallback:** None. The caller must resubmit with a different scheduler configuration or routing problem.

---

### Pre-dispatch Contract Version Mismatch

**Condition:** The selected backend's declared `supported_contract_version` does not match the Worker's current contract version.
**Behavior:** Job transitions to `Failed`. Solver is not invoked. Structured mismatch record persisted. Message dead-lettered.
**Fallback:** None. This is a Worker configuration error requiring manual correction.

---

### Execution Timeout (Worker-Enforced)

**Condition:** Backend does not self-terminate before `execution_timeout_ms` expires.
**Behavior:** Worker terminates the backend and constructs a Timeout SolverResponse (SPEC-004 Failure Modes — ExecutionTimeout Worker-Enforced). Evidence record persisted. Job reaches `Completed`.
**Fallback:** No solution is available. The evidence record reflects the Timeout outcome.

---

### Unresponsive Backend (Process Crash)

**Condition:** Backend process crashes or becomes unresponsive without producing a SolverResponse.
**Behavior:** Worker constructs a Failed SolverResponse (SPEC-004 Failure Modes — Unresponsive Backend). Evidence record persisted. Job reaches `Completed`.
**Fallback:** Evidence record reflects the `Failed` solver outcome with `InternalError`.

---

### Solver Contract Violation (Structural Validation Failure)

**Condition:** Backend returns a response that fails FR-11 post-receipt structural validation.
**Behavior:** Worker records the structural violation. Core quality evaluation is not invoked for a structurally invalid RoutePlan. Evidence record persisted with violation notation. Job reaches `Completed`.
**Fallback:** The violation record provides diagnostic information for investigation of the backend implementation.

---

### Evidence Persistence Failure

**Condition:** PostgreSQL unavailable when Worker attempts to persist evidence artifacts (FR-16).
**Behavior:** Transient failure. Worker issues NACK with requeue. Idempotency guarantees (FR-14) ensure re-execution produces the same result and persists correctly.
**Fallback:** Redelivery and idempotent re-execution.

---

### Report Generation Failure

**Condition:** Report generator fails or is unavailable.
**Behavior:** Worker logs a structured error and emits `report.generate` with span status `Unset`. Worker proceeds to job completion. Job reaches `Completed`. No NACK or requeue is triggered.
**Fallback:** Evidence is persisted; the report will be absent or incomplete. Manual report regeneration may be possible from the persisted evidence record.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer | Yes — Worker is the primary C++ execution service; execution seed derivation is a new Worker-owned policy |
| API Layer | Indirect — API reads job status written by Worker; no new API surface from this spec |
| Persistence | Yes — Worker writes job lifecycle state, decision records, solver run records, quality evaluations; schema defined by SPEC-006 (Evidence Log) |
| Solver Runtime | Yes — Worker is the invoker of all solver backends via SolverContract (SPEC-004); no changes to existing solver specs |
| Observability | Yes — Worker emits four new spans (job.consume, problem.load, result.evaluate, report.generate, job.complete) and enforces span context propagation |
| Configuration | Yes — execution timeout budget policy (OQ-1) and maximum retry count (OQ-4) are Worker configuration concerns |
| Security | Yes — Worker is a trust boundary for inbound job messages; messages must be treated as untrusted (malformed message handling in FR-3) |
| Deployment | Yes — Worker container must have access to PostgreSQL, RabbitMQ, Core libraries, solver backends, Python adapter process (when applicable), and report volume |

**SPEC-004 FR-11.3:** Execution seed derivation ownership explicitly transferred to SPEC-005. FR-7 is the authoritative definition.

**ADR-003:** Dead-letter queue processing strategy is partially defined in FR-13. Reprocessing strategy for jobs already in the dead-letter queue remains an operational concern (OQ-4 context).

---

# Testability

Every Worker lifecycle behavior must be proven before this feature is considered complete. These are test contracts, not test implementations.

1. **Unit: Idempotency — terminal state discard** — When the Worker receives a message for a job already in `Completed` state, it ACKs and discards without modifying any persistent state.

2. **Unit: Idempotency — `Processing` state re-execution** — When the Worker receives a message for a job in `Processing` state (simulating a crash recovery), it re-executes from the beginning and produces the same terminal state and evidence artifacts as the first execution.

3. **Unit: Execution seed derivation — determinism** — Given the same problem seed, the Worker produces the same `execution_seed` on every invocation.

4. **Unit: Execution seed derivation — backend neutrality** — For two different `backend_id` values receiving the same problem seed, the Worker provides the identical `execution_seed` in both SolverRequests.

5. **Unit: Pre-dispatch version check — mismatch** — When the selected backend's `supported_contract_version` does not match the Worker's current contract version, no SolverRequest is dispatched, the job transitions to `Failed`, and a structured mismatch record is persisted.

6. **Unit: Timeout enforcement** — When a C++ in-process backend does not respond within `execution_timeout_ms`, the Worker constructs a Timeout SolverResponse with the required fields, the job reaches `Completed`, and a solver run record with `outcome = Timeout` is persisted.

7. **Unit: Cancellation during execution** — When the Worker receives a cancellation signal during the `Executing` stage, the Worker terminates the backend and either uses the backend's Cancelled response or constructs one. The job reaches `Completed` with `outcome = Cancelled`.

8. **Unit: Post-receipt structural validation — valid response** — A SolverResponse with a structurally valid RoutePlan passes FR-11 validation and proceeds to Core quality evaluation.

9. **Unit: Post-receipt structural validation — structural violation** — A SolverResponse with a RoutePlan that violates FR-5 structural requirements (e.g., missing stop) produces a contract violation record. Core quality evaluation is not invoked. The job still reaches `Completed`.

10. **Unit: Permanent failure — missing problem record** — When the routing problem record does not exist in PostgreSQL, the job transitions to `Failed`, a structured failure record is persisted, and a NACK with requeue=false is issued.

11. **Unit: Permanent failure — Core validation rejection** — A Core validation rejection (ADR-009 divergence) transitions the job to `Failed` with a structured record, and the message is dead-lettered.

12. **Unit: Permanent failure — NoEligibleSolver** — A `NoEligibleSolver` decision from the Scheduler transitions the job to `Failed`, the decision record (with per-backend rejection reasons) is persisted, and the message is dead-lettered.

13. **Unit: Report generation failure — job still completes** — A report generation failure produces a structured error log and `report.generate` span with `Unset` status, but the job reaches `Completed`, not `Failed`.

14. **Integration: Full execution lifecycle — Succeeded outcome** — A valid routing problem is consumed from the queue, Core is invoked, a backend is selected and produces `Succeeded`, quality evaluation completes, evidence is persisted, a report is generated, the job reaches `Completed`, and the message is ACKed.

15. **Integration: Full execution lifecycle — Timeout outcome** — A valid routing problem produces a Timeout solver outcome. Evidence is persisted reflecting the Timeout. The job reaches `Completed`. The message is ACKed.

16. **Integration: OTel span emission — full lifecycle** — A complete job execution produces all Worker-owned spans (job.consume, problem.load, result.evaluate, report.generate, job.complete, solver.execute) with all required attributes. Core-emitted spans (features.extract, scheduler.score_solvers) appear as children of job.consume in the distributed trace.

17. **Integration: OTel span emission — failed job** — A permanent failure (e.g., Core validation rejection) produces job.consume with Error status, problem.load with the appropriate status, and job.complete with Error status. The message is NACKed.

18. **Property: Backend neutrality** — Replacing the selected backend with a different backend, while keeping the routing problem and problem seed identical, produces the same `execution_seed` value in the SolverRequest. No Worker code path produces different behavior based on `backend_id`.

19. **Unit: Transient failure — NACK with requeue** — When PostgreSQL is unavailable during the loading stage, the Worker issues a NACK with requeue=true. The job status in PostgreSQL is not updated to `Failed`. No permanent failure record is persisted. The job may be re-executed on redelivery.

20. **Unit: ACK ordering guarantee — crash-after-Completed** — Given a simulated Worker crash immediately after writing `Completed` to PostgreSQL and before issuing the ACK, the subsequent message redelivery is identified as a terminal-state job via the idempotency check (FR-14). The Worker ACKs the redelivered message and discards it without modifying any persistent state.

---

# Observability Requirements

Operational questions the Worker lifecycle spans, logs, and persisted records must answer:

1. For a given job, what lifecycle state is it currently in?
2. How long did each stage of the lifecycle take (loading, scheduling, executing, evaluating, persisting, reporting)?
3. For a given job that reached `Failed`, at which stage did it fail and what was the failure reason?
4. How often do jobs fail due to NoEligibleSolver, Core validation rejection, or infrastructure failures?
5. What is the distribution of solver outcomes (`Succeeded`, `Timeout`, `Infeasible`, etc.) across all completed jobs?
6. For a given `decision_id`, what was the actual solver outcome compared to the predicted outcome?
7. How often does the Worker-enforced timeout fire, vs. the backend self-terminating?
8. What is the end-to-end job latency from queue consumption to ACK, disaggregated by solver outcome?

These are answered by:
- The `job.consume` span (end-to-end latency, terminal state, solver outcome)
- Child spans (stage-level latency)
- Persisted decision records and solver run records
- Structured log events at each lifecycle stage transition
- Prometheus metrics defined below

---

### Structured Log Events

The Worker emits the following structured log events at each lifecycle stage. All events are emitted as JSON to stdout (ADR-006). Events must not include routing problem raw data (geographic coordinate arrays, full stop lists) per SPEC-001 Security Considerations. Job identifiers, stop counts, and outcome codes are safe to include.

| Event name | Stage | Required fields | Notes |
|---|---|---|---|
| `worker.job.consumed` | FR-3 | `job_id`, `problem_id`, `scheduler_config_id`, `delivery_count` | Emitted before idempotency check; `delivery_count` > 1 indicates redelivery |
| `worker.job.terminal.discard` | FR-14 | `job_id`, `terminal_state`, `delivery_count` | Emitted when idempotency check identifies a terminal-state job; no re-execution occurs |
| `worker.problem.loaded` | FR-4 (success) | `job_id`, `problem_id`, `stop_count`, `vehicle_count`, `seed` | `seed` is the problem seed; must not include `execution_seed` |
| `worker.load.failed` | FR-4 (permanent failure) | `job_id`, `problem_id`, `scheduler_config_id`, `failure_reason`, `missing_entity_type`, `missing_entity_id` | Emitted before NACK is issued |
| `worker.decision.received` | FR-5 | `job_id`, `decision_id`, `decision_status`, `selected_backend_id` | `selected_backend_id` present only when `decision_status = Selected` |
| `worker.core.unexpected_response` | FR-5 | `job_id`, `response_type`, `delivery_count` | Emitted on catch-all CoreContractViolation path |
| `worker.solver.invoked` | FR-9 | `job_id`, `decision_id`, `backend_id`, `contract_version`, `execution_timeout_ms` | Must not include `execution_seed` (Security Considerations) |
| `worker.solver.response.received` | FR-11 | `job_id`, `decision_id`, `outcome`, `execution_duration_ms`, `response_source` | `response_source`: `backend` or `worker_constructed` |
| `worker.quality.evaluation.skipped` | FR-15 | `job_id`, `decision_id`, `solver_outcome`, `reason` | Emitted when no RoutePlan is present; `reason` identifies the outcome class |
| `worker.quality.evaluation.failed` | FR-15 (infrastructure failure) | `job_id`, `decision_id`, `error_type` | Quality fields will be null in persisted record |
| `worker.evidence.persisted` | FR-16 | `job_id`, `decision_id`, `artifacts_written`, `duration_ms` | `artifacts_written`: list of artifact types successfully written; `duration_ms`: persistence operation duration in milliseconds |
| `worker.evidence.upsert.collision` | FR-16 | `job_id`, `artifact_type` | Emitted when a `job_id`-keyed upsert updates an existing record (re-execution detected) |
| `worker.evidence.persistence.failed` | FR-16 | `job_id`, `artifact_type`, `failure_class` | Emitted when an artifact persistence operation fails; `failure_class` identifies the failure type per SPEC-005 FR-13 |
| `worker.report.generation.failed` | FR-17 | `job_id`, `error_type` | Report generation failure; job still completes |
| `worker.job.completed` | FR-18 | `job_id`, `terminal_state`, `solver_outcome` | `solver_outcome` absent when job reached `Failed` before solver invocation |

---

### Metrics

The Worker exposes the following Prometheus metrics (ADR-006). All label value sets are enumerated below to ensure bounded cardinality.

| Metric name | Type | Labels | Description |
|---|---|---|---|
| `daedalus_worker_jobs_consumed_total` | Counter | — | Total jobs consumed from the routing-jobs queue |
| `daedalus_worker_jobs_terminal_total` | Counter | `terminal_state` (`Completed`, `Failed`), `solver_outcome` (`Succeeded`, `Infeasible`, `Timeout`, `Cancelled`, `Failed`, `ContractViolation`, `none`) | Total jobs reaching a terminal state; `solver_outcome = none` when job reached `Failed` before solver invocation |
| `daedalus_worker_job_duration_seconds` | Histogram | `solver_outcome` (same enumeration as above) | End-to-end job duration from message receipt to ACK or NACK |
| `daedalus_worker_timeout_events_total` | Counter | `source` (`worker_enforced`, `backend_self`) | Timeout events by source: Worker externally enforced vs. backend self-terminated before external timeout fired |
| `daedalus_worker_dead_letter_events_total` | Counter | `failure_reason` (`missing_record`, `core_validation_rejection`, `no_eligible_solver`, `invalid_configuration`, `contract_version_mismatch`, `core_contract_violation`, `transient_failure_exhausted`) | Jobs routed to the dead-letter queue, labeled by failure reason |

---

# Security Considerations

**Untrusted message content:**
The Worker must treat RabbitMQ message content as untrusted input. A malformed message (missing required fields, wrong types, excessively long field values) must be rejected with a structured error before any database access occurs (FR-3). The Worker must not panic or produce undefined behavior on malformed message content.

**Execution seed isolation:**
The Worker must not expose the `execution_seed` in logs, traces, or `extension_metadata`. The `execution_seed` is derived from the problem seed and its exposure enables trivial reproduction of solver outputs, which may be a security consideration in non-MVP deployments. At MVP scope with synthetic routing problems, this is an accepted risk.

**Backend termination authority:**
The Worker has authority to terminate C++ in-process backends when timeout fires. Termination logic must be bounded: it must not hang indefinitely if the backend does not respond to termination. A bounded termination timeout is an implementation planning concern.

**Python adapter channel security:**
The transport channel between the Worker and the Python adapter process must be secured in production deployments. Docker Compose internal networking is accepted risk at MVP scope (ADR-005).

**Log safety:**
Structured log events must not include routing problem raw data (geographic coordinate arrays, full stop lists) per SPEC-001 Security Considerations. Job identifiers, stop counts, and outcome codes are safe to log.

---

# Performance Considerations

**Execution overhead:**
The Worker's orchestration overhead (loading, seed derivation, SolverRequest construction, response validation, evidence persistence, report generation) must be measured and reported separately from the solver's `execution_duration_ms`. The `job.consume` span's total duration includes both Worker overhead and solver execution time; the `solver.execute` child span isolates solver execution. Evidence reports must distinguish between solver execution time and total job time.

**Evidence persistence latency:**
PostgreSQL write latency for evidence persistence (FR-16) adds to total job latency. This must be measured during implementation. Connection pool configuration is an implementation planning concern.

**Post-receipt validation cost:**
FR-11 structural validation is O(N) in stop count. For Large-class problems (76+ stops), this cost must be measured and its contribution to total job latency must be reported.

**Report generation latency:**
Report generation occurs after evidence is persisted. Its contribution to total job latency must be measured. If report generation becomes a bottleneck, asynchronous report generation is a future architectural option; this is out of scope for SPEC-005.

---

# Documentation Updates Required

- **docs/architecture.md**: Worker Responsibilities section should be updated to reflect SPEC-005 as the authoritative source for Worker lifecycle behavior. The existing list in architecture.md is a summary; SPEC-005 is the binding contract.
- **ADR-003**: Dead-letter reprocessing strategy is partially addressed by FR-13. The remaining operational strategy for reprocessing dead-lettered jobs should be documented during Worker implementation planning.
- **SPEC-004 FR-11.3**: The execution seed derivation ownership reference to the Worker specification is satisfied by SPEC-005 FR-7. SPEC-004's "Non-Requirements" section can be updated to reference SPEC-005 rather than "Worker specification (future)."
- **SPEC-006 (Evidence Log, Proposed)**: Defines the persistence schema for the artifact set described in FR-16. The schema supports `job_id`-keyed upsert semantics as the idempotency mechanism for all artifact tables, consistent with SPEC-005 FR-14. SPEC-005 is a listed dependency of SPEC-006. SPEC-006 must advance to Accepted before SPEC-005 implementation is complete.
- **SPEC-007 (Core Quality Evaluation, Accepted)**: Defines the Worker-invocable quality evaluation interface compatible with FR-15 (`QualityEvaluationResult`, SPEC-007 FR-9). Establishes the determinism invariant (SPEC-007 FR-10) on which the FR-14 idempotency model depends. SPEC-005 is a listed dependency of SPEC-007 and has been updated to reference SPEC-007 as authoritative.
- **ADR-010 Decision 4**: Revised to reflect Worker-owned execution seed derivation (SPEC-005 FR-7) and the approved derivation policy `execution_seed = RoutingProblem.seed` (ODR-4). The prior text ("Solver-specific derivation strategies are outside the scope of this ADR") has been replaced. This revision was a pre-condition for SPEC-005 advancing to Accepted — satisfied.
- **Report Generator Specification (pending)**: Must define the Worker-invocable interface for evidence report generation, the report file format and storage location, and idempotency behavior on re-invocation with the same `job_id`. SPEC-005 FR-17 implementation is blocked until this specification is accepted.

---

# Open Questions

### OQ-1: Execution Timeout Budget Policy

**Question:** How does the Worker determine the value of `execution_timeout_ms` in the SolverRequest?

**Why it matters:** `execution_timeout_ms` is a required field in the SolverRequest (SPEC-004 FR-2). The Worker constructs the SolverRequest and must provide a value. No existing specification defines how this value is derived. Possible approaches include:
- (a) Derived from the selected backend's `latency_profile[problem_size_class]` declared in the capability profile (SPEC-003 FR-4) — this makes the capability profile the single source of truth for execution time budget.
- (b) A configurable multiplier of `latency_profile[problem_size_class]` — allows tolerance above the declared estimate.
- (c) Taken from `deadline_seconds` in the scheduler configuration when the active mode is `DeadlineAware` — but this only covers one objective mode.
- (d) A new `execution_timeout_ms` field added to the scheduler configuration, independent of objective mode.
- (e) A Worker-level global configuration parameter that applies to all jobs.

**Architectural consequence:** The answer determines whether capability profiles carry execution budget information (option a/b), whether the scheduler configuration is extended (option d), or whether a new configuration layer is introduced (option e). Option (a) or (b) is architecturally preferred because it co-locates declared latency and enforcement budget in the capability profile.

**Owner:** Project Owner decision. Resolution of OQ-1 will require a revision to SPEC-003: options (a) and (b) require extending the capability profile (SPEC-003 FR-4) with an explicit execution budget field; option (d) requires adding a new field to the scheduler configuration schema (SPEC-003 FR-13). The OQ-1 resolution must be coordinated with a SPEC-003 revision — it is not a Worker-only implementation planning decision.

**Blocking:** Blocking for SPEC-005 implementation. `FR-9 (Solver Contract Invocation)` cannot be fully implemented until the source of `execution_timeout_ms` is defined. Not blocking for SPEC-005 Draft status.

---

### OQ-2: Cancellation Signal Mechanism and Authorization

**Questions:**
1. What actor or system is authorized to issue a cancellation for an in-flight job?
2. Through what interface does that actor's cancellation intent reach the Worker?
3. What signal does the Worker send to a C++ in-process backend to request cancellation?

**Why it matters:** FR-12 defines the Worker's behavior when a cancellation signal is received, but neither the authorization source nor the delivery mechanism is defined. The authorization question (sub-question 1) is upstream of the mechanism questions (sub-questions 2 and 3): the mechanism cannot be resolved without first knowing who is authorized to issue a cancellation and through what interface. Additionally, if cancellations can originate from multiple sources (API caller request, operator intervention, system-level deadline), the source should be captured in the evidence record to distinguish caller-initiated from system-enforced cancellations.

The mechanism sub-questions depend on resolution of sub-question 1, and further:
- For C++ in-process backends: the Worker signal mechanism may be a cooperative cancellation flag, a thread interrupt, or a signal handler. This is Worker implementation specification territory.
- For the Python adapter backend: the mechanism depends on the ADR-005 transport contract (currently unresolved).

**Owner:** API specification for sub-questions 1 and 2 (external cancellation interface and authorization); Worker implementation specification for sub-question 3 (C++ in-process signal mechanism); ADR-005 for the Python adapter mechanism.

**Blocking:** Blocking for FR-12 implementation. Not blocking for SPEC-005 Draft status or C++ solver implementation (which can proceed with timeout enforcement only).

---

### OQ-3: Execution Seed Derivation Formula — RESOLVED (ODR-4)

**Resolution:** `execution_seed = RoutingProblem.seed`. No transformation is applied. Incorporated in FR-7 Policy item 5 and FR-7 Acceptance Criteria.

**Decision rationale (ODR-4):** Maximum reproducibility; simplest implementation; consistent with ADR-010 deterministic randomness policy; eliminates unnecessary derivation complexity; backend-neutral. Approved by Project Owner.

**Blocking:** Was blocking for Accepted status. Resolved — no longer blocking.

---

### OQ-4: Dead-Letter Queue Reprocessing Strategy

**Question:** How should operators reprocess jobs that have been routed to the dead-letter queue?

**Why it matters:** ADR-003 explicitly deferred dead-letter queue processing strategy to Worker implementation planning. FR-13 defines when jobs are dead-lettered but does not define how they are reprocessed. Options include: manual resubmission via the API, an administrative endpoint to replay dead-lettered messages, operator tooling to inspect and selectively requeue, or simply discarding with a notification.

**Owner:** Operations specification or Worker implementation planning.

**Blocking:** Not blocking for SPEC-005 acceptance or implementation. The dead-letter queue receives failed jobs regardless of whether a reprocessing strategy exists.

---

# Acceptance Checklist

- [x] Problem is clearly defined
- [x] Domain concept is defined
- [x] Responsibility scope and component boundaries are defined (FR-1)
- [x] Job lifecycle state model is defined (FR-2)
- [x] Job consumption behavior is defined (FR-3)
- [x] Problem and configuration loading behavior is defined (FR-4)
- [x] Core invocation and Scheduler decision consumption are defined (FR-5)
- [x] Decision record persistence is defined (FR-6)
- [x] Execution seed derivation ownership is established (FR-7)
- [x] Contract version pre-dispatch validation is defined (FR-8)
- [x] Solver contract invocation behavior is defined (FR-9)
- [x] Timeout enforcement is defined (FR-10)
- [x] SolverResponse post-receipt validation is defined (FR-11)
- [x] Cancellation handling is defined (FR-12)
- [x] Retry and dead-letter behavior is defined (FR-13)
- [x] Idempotency requirements are defined (FR-14)
- [x] Core quality evaluation handoff is defined (FR-15)
- [x] Evidence persistence handoff is defined (FR-16)
- [x] Report generation is defined (FR-17)
- [x] Job completion and message acknowledgment are defined (FR-18)
- [x] Telemetry requirements are defined (FR-19)
- [x] Non-requirements are documented
- [x] Assumptions are explicit
- [x] Failure modes are defined
- [x] Observability requirements exist
- [x] Security considerations exist
- [x] Documentation updates are identified
- [ ] OQ-1 resolved — execution timeout budget policy (blocking for implementation; Project Owner decision)
- [ ] OQ-2 resolved — cancellation signal mechanism (blocking for FR-12 implementation)
- [x] OQ-3 resolved — execution seed derivation formula: `execution_seed = RoutingProblem.seed` (ODR-4)
- [ ] OQ-4 resolved — dead-letter reprocessing strategy (non-blocking; operational concern)

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-19) are implemented and acceptance criteria pass
- OQ-1 (execution timeout budget policy) is resolved and incorporated in FR-9/FR-10
- OQ-2 (cancellation signal mechanism) is resolved and incorporated in FR-12
- OQ-3 (execution seed derivation formula) is resolved: `execution_seed = RoutingProblem.seed` (ODR-4) — satisfied
- All test contracts defined in the Testability section pass
- All required OTel spans (job.consume, problem.load, result.evaluate, report.generate, job.complete) are emitted and verifiable in the test environment
- SPEC-006 (Evidence Log) is accepted and the persistence schema is compatible with the FR-16 artifact set
- SPEC-007 (Core Quality Evaluation) is accepted and the evaluation interface is compatible with FR-15 — satisfied
- Engineering review passes
- Specification status is updated to Verified
