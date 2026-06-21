# SPEC-004: Solver Contract

## Metadata

**Feature ID:** SPEC-004

**Title:** Solver Contract

**Status:** Accepted

**Author:** Darkhorse286

**Created:** 2026-06-08

**Last Updated:** 2026-06-20 (Revision 2 — Post-acceptance documentation alignment; OQ-1 closed referencing SPEC-017)

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-001, ADR-005, ADR-007, ADR-008, ADR-009, ADR-010

**Related Specs:** SPEC-001, SPEC-003, (pending) Evidence Log Specification, (pending) Core Quality Evaluation Specification

---

# Problem Statement

ADR-008 (Solver Contract and Backend Neutrality) establishes that all solver backends must be accessed through a normalized solver contract, and that the Scheduler operates exclusively on normalized candidates. ADR-008 explicitly defers the concrete interface definition, stating: "The concrete interface definition is a blocking dependency for all solver implementations." Without a concrete contract, no solver backend can be implemented, the Worker has no defined interface for solver invocation, and quality evaluation and regret analysis cannot be specified.

SPEC-004 provides the concrete realization of ADR-008. It defines the data exchange semantics, structural constraints, and behavioral requirements of the interface between the Worker and all optimization backends. It does not define execution strategy, persistence, or quality evaluation.

---

# Business Value

- Unblocks all MVP solver backend implementations (nearest-neighbor, greedy insertion, QUBO simulated annealing)
- Enables the Worker to invoke any backend without special-casing
- Ensures all solver outputs share a normalized structure, enabling Core to evaluate quality and regret on a consistent basis
- Establishes that adding a new backend requires no changes to the Scheduler, Worker, or Core contract-handling logic
- Demonstrates production-grade interface contract design as a portfolio artifact

---

# Employer Signaling

- System Design
- API Contract Design
- Reliability Engineering
- Optimization

---

# Domain Concept

The Solver Contract is a normalized interface through which the Worker invokes optimization backends. Every registered backend — whether a classical heuristic, a QUBO-adjacent stochastic solver, or a Python adapter process — presents the same request-response schema to the Worker. Backend-specific execution strategies, internal data structures, and optimization algorithms are hidden behind this contract. The contract defines what goes in and what comes out. It does not define how the optimization is performed.

The Solver Contract is not an execution engine. It is a data exchange specification.

---

# Requirements

### FR-1: Responsibility Scope and Component Boundaries

**Description:**
SPEC-004 defines the Solver Contract's scope and explicitly allocates responsibilities across all components that interact with it.

| Component | Responsibilities at This Boundary | Explicitly Not Responsible For |
|---|---|---|
| **Scheduler** (SPEC-003) | Backend selection; producing the decision record identifying the selected backend | Solver invocation; solution evaluation; persisting solver results |
| **Solver Contract** (SPEC-004) | Request schema; response schema; outcome model; structural constraints on the route plan; extension metadata mechanism; versioning semantics | Worker orchestration strategy; quality evaluation; result persistence; Python adapter transport protocol |
| **Solver Backend** | Implementing the contract; accepting every SolverRequest; returning a conforming SolverResponse for every invocation; honoring the execution timeout; reporting the honest outcome | Persisting results; evaluating route quality; re-validating the routing problem; selecting which problem to solve |
| **Worker** (architecture.md) | Invoking the selected backend; enforcing the execution timeout externally; signaling cancellation; persisting the SolverResponse; constructing a Failed response if the backend does not respond | Backend selection; defining the contract; route quality evaluation; regret calculation |
| **Core** (architecture.md) | Post-execution quality evaluation; route feasibility verification (time windows); regret calculation; passing the validated routing problem to the Worker for solver invocation | Invoking solver backends directly; persisting solver results |
| **Evidence Log** (pending specification) | Defining the persistence schema for SolverResponse fields | Producing solver responses; invoking backends |

**Acceptance Criteria:**
- No component performs responsibilities allocated to another component at this boundary
- Adding a new backend requires only: implementing the SolverContract, registering a capability profile (SPEC-003 FR-4), and documenting its seed usage policy in its own specification (that it accepts and uses the Worker-provided `execution_seed` as its exclusive entropy source via PCG64, per ADR-010 Decision 1 and FR-11.3)
- No changes to the Scheduler, Worker, or Core contract-handling logic are required to add a new backend

---

### FR-2: Solver Request

**Description:**
The SolverRequest is the complete input delivered by the Worker to a backend at invocation time. It contains the routing problem, the execution configuration, and correlation identifiers. No backend receives additional request fields outside of this schema.

| Field | Type | Required | Description |
|---|---|---|---|
| `contract_version` | uint32 | Required | The contract version being used. Current value: 1. See FR-14 for versioning semantics. Backends receiving an unsupported value must return Failed with failure_code = ContractVersionMismatch. |
| `routing_problem` | RoutingProblem | Required | The SPEC-001 C++ domain representation of the routing problem. Validated by Core before invocation (ADR-009). Contains all fields defined in SPEC-001 FR-14: coordinates, demands, vehicle fleet, capacity, time windows, service durations, and seed. |
| `execution_timeout_ms` | uint32 | Required | Maximum execution budget in milliseconds. Must be a positive integer (> 0). The Worker enforces this externally. Solvers should self-terminate by this deadline. See FR-10. |
| `execution_seed` | uint64 | Required | PRNG seed for this solver execution. Derived from the routing problem seed (SPEC-001 FR-6) using a uniform derivation policy owned by the Worker (FR-11.3). Solver specifications document seed *usage* policy, not derivation. This is the exclusive permitted entropy source for all reproducibility-critical stochastic operations within the execution. See FR-11. |
| `job_id` | UUID | Required | Job identifier. For trace correlation only. Does not affect solver execution. |
| `decision_id` | UUID | Required | Scheduler decision record identifier (SPEC-003 FR-10). For trace correlation and evidence only. Does not affect solver execution. |
| `backend_id` | string | Required | Identifier of the backend being invoked, matching the `backend_id` in the registered capability profile (SPEC-003 FR-4). For correlation only. Does not affect solver execution. |

The `routing_problem` field contains the complete routing problem including all geographic coordinates. The solver computes pairwise distances from coordinates using the Haversine formula (SPEC-001 FR-5). No precomputed distance matrix is provided.

**Acceptance Criteria:**
- A SolverRequest omitting any required field is a Worker construction error, not a backend error
- Backends must accept any structurally valid SolverRequest and return a SolverResponse
- Correlation fields (job_id, decision_id, backend_id) have no effect on solver execution output

---

### FR-3: Solver Response

**Description:**
The SolverResponse is the complete output produced by a backend for every invocation. A SolverResponse is always produced — including on failure. A backend that terminates without producing a SolverResponse is handled by the Worker, which constructs a Failed SolverResponse on the backend's behalf. That fallback behavior is Worker orchestration and is outside this contract.

| Field | Type | Present When | Description |
|---|---|---|---|
| `contract_version` | uint32 | Always | Must equal the `contract_version` from the corresponding SolverRequest. |
| `outcome` | SolverOutcome | Always | The execution outcome. Defined in FR-4. |
| `solution` | RoutePlan | See FR-7, FR-9, FR-10 | The route plan. Defined in FR-5. |
| `statistics` | ExecutionStatistics | Always | Execution timing data. Defined in FR-6. |
| `failure` | SolverFailureDetail | When outcome ≠ Succeeded | Structured failure information. Defined in FR-8. Absent on Succeeded. |
| `extension_metadata` | map\<string, string\> | Optional | Backend-specific metadata. Defined in FR-13. |

**Acceptance Criteria:**
- Every backend invocation produces a structurally complete SolverResponse
- `contract_version` in the response equals `contract_version` in the corresponding request
- `failure` is absent when outcome = Succeeded
- `failure` is present when outcome ≠ Succeeded
- Core and Worker consume only the normalized fields; extension_metadata does not alter their behavior

---

### FR-4: Solver Outcome Model

**Description:**
The SolverOutcome enumeration is the primary status indicator in every SolverResponse. It is a closed set. Backends must report the honest outcome.

| Value | Meaning | `solution` Present? | `failure` Present? |
|---|---|---|---|
| `Succeeded` | A complete, capacity-valid route plan was found and verified by the backend. | Yes — required | No |
| `Infeasible` | The backend determined the problem has no feasible solution. This is a strong claim indicating the backend has proven infeasibility, not merely failed to find a solution within the time budget. | No | Yes |
| `Timeout` | Execution exceeded `execution_timeout_ms`. The solver self-terminated or was terminated by the Worker. | Optional — see FR-10 | Yes |
| `Cancelled` | The Worker cancelled the execution before the deadline. | Optional — see FR-9 | Yes |
| `Failed` | An unexpected internal execution error occurred, or the request was rejected (e.g., ContractVersionMismatch). | No | Yes |

Heuristic backends (nearest-neighbor, greedy insertion, simulated annealing) are not required to support the `Infeasible` outcome. These backends either find a complete solution or terminate (Timeout or Cancelled). `Infeasible` is reserved for backends capable of proving infeasibility (e.g., exact solvers). Returning `Infeasible` when the backend merely ran out of time is a contract violation.

**Acceptance Criteria:**
- A backend that has not found a complete solution and has run out of time must return `Timeout`, not `Infeasible`
- A backend that finds a complete, capacity-valid solution must return `Succeeded`, not any other value
- The honest outcome is reported in all cases; misreporting outcome is a contract violation

---

### FR-5: Route Plan Structure

**Description:**
A RoutePlan is the solution payload. It represents a complete assignment of all stops across all vehicles.

**Structure:**
- `routes`: A sequence of exactly `vehicle_count` route entries, indexed 0 through `vehicle_count − 1`.
- Route at index `i` corresponds to vehicle `i`.
- Each route entry is an ordered sequence of stop IDs (non-negative integers matching SPEC-001 FR-4 stop identifiers).
- An empty sequence at route index `i` is valid and means vehicle `i` is unused.

**Depot exclusion:** Routes do not include the depot. All vehicles implicitly depart from and return to the depot (SPEC-001 FR-3). A route enumerates only the intermediate stops visited by that vehicle between depot departure and return. The depot has no stop_id in the SPEC-001 domain model (SPEC-001 FR-3) and is never represented in a RoutePlan route sequence. The completeness requirement in condition 4 below applies only to the stop_ids defined in the routing problem's stop list; the depot is not a stop and is not subject to that check.

**Structural validity requirements (contract-level constraints):**
The backend must verify all of the following before returning outcome = Succeeded. A solution present in a Timeout or Cancelled response must also satisfy these requirements.

1. `routes.size()` equals `vehicle_count` from the routing problem.
2. Every stop ID in every route is a valid stop ID from the routing problem.
3. No stop ID appears in more than one route.
4. Every stop ID from the routing problem appears in exactly one route.
5. For each route: the sum of `demand` values for stops in that route does not exceed `capacity_per_vehicle`.

**Core-verified constraints (not contract-level — verified by Core during quality evaluation, not by the backend):**
- Time window feasibility: for each time-windowed stop, the simulated arrival time is within \[time_window_open, time_window_close\].
- Route continuity and service duration compliance per SPEC-001 FR-9 and FR-16.

Returning a RoutePlan that violates structural validity requirements (1–5) when outcome = Succeeded, Timeout, or Cancelled is a contract violation. The Worker performs post-receipt structural validation on all responses that include a solution.

**Partial assignments are not permitted.** A solution in which some stops are unassigned is not a valid RoutePlan under any outcome. If the backend cannot produce a complete assignment before the deadline, it must return Timeout or Cancelled with no solution — not a partially-assigned route plan. See OQ-3.

**Acceptance Criteria:**
- A Succeeded response always contains a structurally valid RoutePlan
- A Timeout or Cancelled response with a solution present contains a structurally valid RoutePlan
- No RoutePlan contains an unassigned stop
- No RoutePlan assigns the same stop to more than one vehicle
- No route in any RoutePlan has a total stop demand exceeding vehicle capacity

---

### FR-6: Execution Statistics

**Description:**
ExecutionStatistics fields capture timing and effort data for every invocation.

| Field | Type | Required | Description |
|---|---|---|---|
| `execution_duration_ms` | uint64 | Required | Wall-clock execution time from backend invocation to SolverResponse production, in milliseconds. Must be a non-negative integer. |
| `solution_count` | uint32 | Optional | Number of distinct complete feasible solutions found during execution. For iterative improvement solvers, this is the number of times a new best solution was found. Absent if not tracked. |

Backend-specific statistics (annealing energy, insertion attempts, acceptance rates) belong in `extension_metadata` (FR-13), not in ExecutionStatistics.

`execution_duration_ms` is excluded from the determinism invariant (FR-11) and is not expected to be identical across identical invocations.

**Acceptance Criteria:**
- `execution_duration_ms` is present in every SolverResponse
- `execution_duration_ms` is a non-negative integer

---

### FR-7: Success Semantics

**Description:**
A Succeeded outcome means the backend found a complete, capacity-valid solution and verified it satisfies all structural validity requirements (FR-5, conditions 1–5) before returning.

The solution in a Succeeded response is the backend's best complete feasible solution found within the execution budget. Backends are expected to optimize, not merely find any valid solution. For heuristic backends, "best" is defined by the backend's internal objective function. The normalized quality metric used for evidence and regret analysis is computed independently by Core from the route plan.

A Succeeded response does not guarantee time window feasibility. Core evaluates time window feasibility as part of quality evaluation. A backend that returns Succeeded with a route that violates time windows has returned a structurally valid but time-infeasible solution; this is a solver quality defect, not a contract violation. Core records the feasibility outcome in the quality evaluation.

**Acceptance Criteria:**
- `solution` is present and structurally valid (FR-5 requirements 1–5 satisfied)
- `failure` is absent
- `contract_version` in the response equals `contract_version` in the request
- The backend has verified structural validity before returning Succeeded

---

### FR-8: Failure Semantics

**Description:**
SolverFailureDetail provides structured information for every non-Succeeded outcome.

**SolverFailureDetail structure:**

| Field | Type | Required | Description |
|---|---|---|---|
| `failure_code` | SolverFailureCode | Required | Structured code identifying the failure category. |
| `failure_message` | string | Required | Human-readable description. Not for programmatic consumption. |

**SolverFailureCode enumeration (forward-extensible):** New values may be added as non-breaking changes (FR-14). Consumers must treat any unrecognized value as `InternalError`.

| Code | Used With | Meaning |
|---|---|---|
| `InfeasibleProblem` | `Infeasible` | Backend determined no feasible solution exists. |
| `ExecutionTimeout` | `Timeout` | Execution exceeded `execution_timeout_ms`. |
| `ExecutionCancelled` | `Cancelled` | Worker cancelled the execution. |
| `ContractVersionMismatch` | `Failed` | `contract_version` in SolverRequest is not supported by this backend. |
| `InternalError` | `Failed` | Unexpected execution error (solver library failure, assertion, memory error, etc.). |

When outcome = Succeeded, `failure` is absent.
When outcome ≠ Succeeded, `failure` is present with the appropriate `failure_code`.

**Acceptance Criteria:**
- `failure` contains a `failure_code` consistent with the `outcome`
- `failure_message` is non-empty on every failure
- `failure` is absent on Succeeded

---

### FR-9: Cancellation Semantics

**Description:**
The Worker may cancel a solver execution at any time by signaling the backend to terminate before the execution deadline. The signal mechanism is implementation planning and is not defined by this contract.

**Cancellation behavior obligations:**
1. The backend must respond to a cancellation signal and terminate. The maximum response time to a cancellation signal is an implementation planning concern outside this contract.
2. Upon cancellation, the backend sets `outcome = Cancelled` and `failure_code = ExecutionCancelled`.
3. The backend MAY include a `solution` in the Cancelled response. If a solution is included, it must satisfy all FR-5 structural validity requirements. A solution present in a Cancelled response is the best complete feasible solution the backend had found at the time of cancellation.
4. If no complete feasible solution existed at the time of cancellation, the `solution` field is absent.
5. The Worker decides whether to use a partial solution from a Cancelled response. That decision is Worker orchestration, outside this contract.

When a Cancelled response includes a solution, Core evaluates quality normally. The `outcome = Cancelled` code is recorded in the evidence record, attributing the quality result to a partial execution.

**Acceptance Criteria:**
- A Cancelled response with a solution present contains a structurally valid RoutePlan
- A Cancelled response without a solution has no `solution` field
- `failure` is present in all Cancelled responses

---

### FR-10: Timeout Behavior

**Description:**
`execution_timeout_ms` in the SolverRequest is the maximum execution budget.

**Timeout behavior obligations:**
1. The backend should self-terminate before or at the `execution_timeout_ms` deadline.
2. The Worker enforces the timeout externally. If the backend does not self-terminate before the deadline, the Worker terminates it and constructs a Timeout SolverResponse on its behalf. A Worker-constructed Timeout response has no `solution` field.
3. A backend that self-terminates due to timeout sets `outcome = Timeout` and `failure_code = ExecutionTimeout`.
4. The backend MAY include a `solution` in the self-produced Timeout response. If included, it must satisfy all FR-5 structural validity requirements. A solution in a Timeout response is the best complete feasible solution found before the deadline.
5. If no complete feasible solution was found before the deadline, the `solution` field is absent.

A Timeout response that includes a solution is not lower quality than a Succeeded response from the perspective of the contract. Core evaluates the route plan's quality and time window feasibility regardless of `outcome`. The `outcome` code is recorded in the evidence log.

For iterative improvement heuristics (e.g., simulated annealing), anytime behavior — where the backend maintains a running best-complete-solution and can return it at any point — is the preferred implementation approach.

**Quantum hardware anytime exception:** For `quantum_hardware` category backends (SPEC-011 FR-2.1) that implement anytime behavior, `Timeout` with `failure_code = ExecutionTimeout` may be produced by a provider infrastructure failure during hardware execution when hardware execution has already begun and partial results may be recoverable. This exception applies only when the backend has invested real QPU execution time and best-so-far preservation is required. It does not apply to infrastructure failures before hardware execution begins (provider outage at job submission or during queue wait produce `Failed` with `InternalError`). It does not generalize infrastructure failures into `Timeout` for any other backend category. A backend using this exception must distinguish the infrastructure-failure Timeout from a time-budget Timeout through an extension_metadata field; for SPEC-019 (Quantum Hardware Solver Backend), the distinguishing field is `qaoa.hardware.failure_classification = "provider_outage"`.

`execution_timeout_ms = 0` is invalid. The Worker must always provide a positive value.

**Acceptance Criteria:**
- A Timeout response with a solution present contains a structurally valid RoutePlan
- A self-produced Timeout response has `failure` present with `failure_code = ExecutionTimeout`
- `execution_timeout_ms = 0` is never sent; if received, the backend returns Failed with InternalError

---

### FR-11: Determinism and Reproducibility

**Description:**
All solver executions must satisfy the following requirements, which bind ADR-010 to the solver contract.

1. **Reproducibility invariant:** Given identical inputs — same `routing_problem`, same `execution_seed`, same `contract_version` — a backend must produce an identical `solution` on every invocation. `execution_duration_ms` is excluded from this invariant.

2. **Seed authority:** `execution_seed` in the SolverRequest is the exclusive permitted entropy source for all reproducibility-critical stochastic operations within the execution. System time, process ID, OS random sources (`/dev/urandom`, `getrandom`, `std::random_device`), and hardware entropy are prohibited in reproducibility-critical paths (ADR-010 Decision 4). Backends must not use `routing_problem.seed` (SPEC-001 FR-6) as a PRNG entropy source. The problem seed is a scenario reproducibility identifier; `execution_seed` is the solver execution entropy source. Backends have access to both fields in the SolverRequest and must use only `execution_seed`.

3. **Seed derivation:** `execution_seed` is computed by the Worker from the routing problem seed (SPEC-001 FR-6) using a uniform derivation policy that applies identically to all backends regardless of `backend_id`. The derivation policy is owned by the Worker specification. The specific derivation formula is an implementation planning decision; at MVP scope all backends receive the same derivation. Each solver specification documents its seed *usage* policy — that it uses the Worker-provided `execution_seed` as its exclusive entropy source via PCG64 (ADR-010 Decision 1) — but does not define a custom derivation from the problem seed. The backend accepts and uses the provided `execution_seed`; it does not derive its own seed from the problem seed.

4. **PRNG policy:** All stochastic computations within a solver execution must use PCG64 seeded from `execution_seed` (ADR-010 Decision 1). `std::uniform_int_distribution` and `std::normal_distribution` are prohibited in reproducibility-critical paths. The Box-Muller transform and bias-free integer sampling algorithms are the approved distribution methods (ADR-010 Decision 3).

5. **Classical deterministic backends:** Backends with no stochastic operations (e.g., nearest-neighbor, greedy insertion) produce identical outputs for identical inputs regardless of `execution_seed` by construction. These backends must still accept the `execution_seed` field without error, for contract uniformity, but do not use it.

6. **Outcome code reproducibility:** `Timeout` and `Cancelled` outcomes depend on timing and external signals and carry no reproducibility obligation. The reproducibility invariant applies only when **both** invocations produce `outcome = Succeeded`. A determinism test that compares a Succeeded response against a Timeout response does not constitute a reproducibility failure.

**Acceptance Criteria:**
- Two identical SolverRequests produce identical SolverResponse solutions when outcome = Succeeded on both
- No stochastic backend uses entropy sources other than `execution_seed`
- Classical backends accept `execution_seed` without error

---

### FR-12: Backend Neutrality

**Description:**
The Solver Contract enforces ADR-008 Backend Neutrality through the following structural properties.

1. **Uniform request:** Every registered backend receives the same SolverRequest field structure. No backend receives additional fields not present in the contract definition.

2. **Uniform response:** Every backend returns the same SolverResponse field structure. Core and Worker consume the normalized fields. Backend-specific data is confined to `extension_metadata` (FR-13).

3. **No identity routing:** The Worker and Core must not branch on `backend_id` in contract handling logic. All SolverResponses are processed through the same contract handling path regardless of which backend produced them.

4. **Additive backend registration:** Adding a new backend requires only: implementing SolverContract, registering a capability profile in the backend registry (SPEC-003 FR-4), and documenting the seed usage policy in the solver specification (ADR-010 Decision 1 — use Worker-provided `execution_seed` as exclusive entropy source via PCG64). No changes to the Scheduler, Worker, Core contract-handling logic, or any existing backend are required.

5. **Extension metadata isolation:** `extension_metadata` does not affect normalized contract semantics. Core and Worker must not fail, reject, or alter behavior based on `extension_metadata` contents or absence.

**Acceptance Criteria:**
- A new backend passes all contract conformance tests defined in the Testability section without any changes to Scheduler, Worker, or Core
- Confirmed by code review: no branch on `backend_id` exists in Worker or Core contract processing logic

---

### FR-13: Extension Metadata

**Description:**
`extension_metadata` is an optional field in the SolverResponse providing a structured extension point for backend-specific execution outputs.

**Structure:** `map<string, string>` — string keys mapped to string values.

**Purpose:**
- QUBO-style backends may report annealing-specific data (e.g., `"qubo.final_energy"`, `"qubo.annealing_steps"`, `"qubo.acceptance_rate"`)
- Classical backends may report algorithm data (e.g., `"nn.route_extensions_considered"`, `"gi.insertion_attempts"`)
- Extension metadata is passed through to the evidence log for backend-specific reporting

**Constraints:**
- Keys must be non-empty strings
- Values must be non-empty strings
- `extension_metadata` must not contain routing problem raw data (geographic coordinate arrays, full stop lists) — per SPEC-001 Security Considerations on log safety
- `extension_metadata` must not replicate any normalized SolverResponse field (outcome, statistics, failure fields)
- The set of recognized keys per backend is documented in each solver's specification; unrecognized keys must be silently ignored by generic consumers (Core, Worker)
- Compliance with the raw-data prohibition is not enforceable at the contract level. It is enforced through code review and backend specification acceptance. Each backend specification must explicitly affirm that its `extension_metadata` schema excludes raw problem data. Automated enforcement is deferred beyond MVP scope.

**Acceptance Criteria:**
- Core and Worker produce identical behavior regardless of whether `extension_metadata` is present, absent, or contains unknown keys
- A backend that omits `extension_metadata` is contract-conformant

---

### FR-14: Contract Versioning and Compatibility

**Description:**
`contract_version` in SolverRequest and SolverResponse identifies the schema version in use.

**Current version:** `1`

**Version handling obligations:**
- A backend that receives `contract_version = 1` processes the request and returns `contract_version = 1` in the response.
- A backend that receives an unsupported `contract_version` must return `outcome = Failed` and `failure_code = ContractVersionMismatch` immediately, without processing the routing problem.
- The Worker must only send SolverRequests with `contract_version` values the target backend is documented to support.

**Non-breaking changes** (do NOT increment `contract_version`):
- Adding optional fields to `extension_metadata` schemas (backend-specific, no contract impact)
- Adding new `SolverFailureCode` values (unknown codes treated as `InternalError` by existing consumers)
- Adding optional fields to `ExecutionStatistics`
- Clarifying field descriptions without changing semantics

**Breaking changes** (MUST increment `contract_version`):
- Removing any required field from SolverRequest or SolverResponse
- Changing the type or semantics of any existing required field
- Removing values from the `SolverOutcome` enumeration
- Changing RoutePlan structural validity requirements (FR-5)
- Changing the `execution_seed` contract (type, authority, or derivation obligations)

**Breaking change procedure:**
1. `contract_version` is incremented
2. All backend specifications implementing this contract are updated with the new version
3. This specification is revised with an explicit change record documenting the previous and replacement definition

**Acceptance Criteria:**
- A backend receiving `contract_version = 1` returns `contract_version = 1` in the response
- A backend receiving an unsupported version returns `ContractVersionMismatch` without processing

---

### FR-15: Telemetry

**Description:**
The `solver.execute` OTel span (defined in architecture.md) is emitted by the Worker for every solver invocation. The Worker owns span emission; the backend does not produce OTel spans.

**Span: `solver.execute`**
- Scope: Begins immediately before the SolverRequest is dispatched to the backend; ends when the SolverResponse is received and structurally validated by the Worker. When the Worker constructs a SolverResponse due to backend termination (Worker-enforced timeout) or process failure (unresponsive backend), the span ends when the Worker completes constructing the response. The `execution_duration_ms` span attribute is the Worker-measured elapsed time from dispatch to construction completion in these cases.
- Required attributes:

| Attribute | Description |
|---|---|
| `backend_id` | The `backend_id` from the SolverRequest |
| `job_id` | Job identifier for correlation |
| `decision_id` | Scheduler decision identifier (SPEC-003 FR-10) for correlation |
| `problem_size_class` | Size class of the routing problem, from workload features (SPEC-003 FR-3) |
| `outcome` | `SolverOutcome` value from the response |
| `execution_duration_ms` | From `statistics.execution_duration_ms` |
| `stop_count` | Number of stops in the routing problem |

- Span status rules:
  - `outcome = Succeeded`: span status **OK**
  - `outcome = Infeasible` or `outcome = Failed`: span status **Error**
  - `outcome = Timeout` or `outcome = Cancelled` with no solution: span status **Error**
  - `outcome = Timeout` or `outcome = Cancelled` with solution present: span status **Unset** — the solver produced a usable result within the execution boundary; this is not a system failure. The `outcome` attribute distinguishes these cases in queries without inflating error-rate metrics.
- `extension_metadata` contents are not included as span attributes (cardinality control)
- The span is correlatable with the Scheduler decision via `decision_id` and with the job via `job_id`

**Acceptance Criteria:**
- `solver.execute` is emitted for every SolverRequest dispatch, successful or not
- All required attributes are present on every emission
- The span is correlatable with both the job and the Scheduler decision

---

# Non-Requirements

- The Solver Contract does not define Worker execution strategy (retry logic, backoff, result caching, queue management)
- The Solver Contract does not define how SolverResponse fields are persisted to PostgreSQL (Evidence Log Specification responsibility)
- The Solver Contract does not define route quality evaluation or the normalized quality metric (Core Quality Evaluation Specification responsibility)
- The Solver Contract does not define regret calculation (Core responsibility)
- The Solver Contract does not define the Python adapter transport protocol. The transport and wire format are defined by SPEC-017 (Accepted); the behavioral contract is owned by SPEC-017. ADR-005 is Accepted.
- The Solver Contract does not define the specific formula by which the Worker derives `execution_seed` from the problem seed (Worker specification responsibility; the uniformity policy is defined in FR-11.3)
- The Solver Contract does not define solver timeout response time obligations beyond "must respond to cancellation" (implementation planning concern)
- The Solver Contract does not define QUBO problem formulation or annealing schedule (backend-internal concerns)
- The Solver Contract does not define multi-vehicle route optimization strategy; route planning is solver-internal
- The Solver Contract does not define a normalized quality metric; quality is computed by Core from the RoutePlan

---

# Assumptions

1. All MVP solver backends can express their optimization output as an ordered stop sequence per vehicle satisfying FR-5 structural requirements.
2. A normalized quality metric meaningful for cross-backend comparison can be computed by Core from the RoutePlan (ordered stop sequences) and the routing problem (coordinates, capacity, time windows, service durations). The specific metric is defined in the Core Quality Evaluation Specification.
3. Solver backends can self-terminate within a bounded time when `execution_timeout_ms` expires. The specific response time bound is a solver implementation concern.
4. The Python adapter process realizes the same logical SolverContract through JSON over HTTP (SPEC-017 FR-2). The logical contract schema applies uniformly; wire-format representations for all SolverRequest and SolverResponse fields are defined in SPEC-017 FR-3 and FR-4.
5. MVP backends (nearest-neighbor, greedy insertion, QUBO simulated annealing) can all produce capacity-valid, complete route plans as their primary output type within reasonable time budgets.
6. The QUBO simulated annealing backend can implement anytime behavior: maintaining a running best-complete-solution that has been decoded from the QUBO binary assignment space and repair-processed into a capacity-valid RoutePlan satisfying FR-5 structural requirements, returnable at any point before deadline. This is an implementation risk: a partial annealing state may not decode into a complete, feasible route plan, in which case the backend returns Timeout or Cancelled with no solution. This risk must be validated and explicitly addressed during QUBO backend specification.

---

# Constraints

1. No solver backend may bypass the normalized SolverContract. All backends must produce SolverResponses conforming to this specification (ADR-008).
2. Backends must not use entropy sources other than `execution_seed` for reproducibility-critical stochastic operations (ADR-010 Decision 4).
3. The contract operates on the C++ routing problem domain representation (ADR-001, SPEC-001 FR-14).
4. `extension_metadata` must not contain routing problem raw data (geographic coordinate arrays, full stop lists), to comply with SPEC-001 Security Considerations on log safety.
5. The Python adapter transport contract is outside this specification's scope. The transport and behavioral contract are defined by SPEC-017 (Accepted). ADR-005 is Accepted with the JSON over HTTP transport decision.

---

# Inputs

The SolverRequest is the complete input. See FR-2 for the field table.

The `routing_problem` is:
- A validated SPEC-001 routing problem in C++ domain representation
- Validated by Core before solver invocation (ADR-009)
- Immutable for the duration of solver execution
- The solver does not re-validate the problem

---

# Outputs

The SolverResponse is the complete output. See FR-3 for the field table.

A SolverResponse is always produced — even on failure. The output is consumed by:
- The Worker (for timeout enforcement, structural validation, persistence)
- Core (for quality evaluation and regret calculation, via the Worker)
- The Evidence Log (persistence schema ownership)

---

# Failure Modes

### ContractVersionMismatch

**Condition:** The SolverRequest contains a `contract_version` the backend does not support.
**Behavior:** Backend returns `outcome = Failed`, `failure_code = ContractVersionMismatch`, without processing the routing problem.
**Fallback:** Worker logs the failure; job processing fails with a structured error; manual investigation required.

---

### MalformedRoutePlan

**Condition:** A backend returns `outcome = Succeeded` (or Timeout/Cancelled with solution present) but the RoutePlan violates FR-5 structural requirements (missing stops, duplicate stops, wrong route count, capacity violation).
**Behavior:** The Worker detects the violation during post-receipt structural validation and treats the invocation as Failed.
**Fallback:** Worker records the structural violation in the evidence log; job fails.
**Note:** MalformedRoutePlan detection is a Worker responsibility; it is a backend contract violation.

---

### ExecutionTimeout (Worker-Enforced)

**Condition:** Backend does not self-terminate before `execution_timeout_ms` expires.
**Behavior:** Worker terminates the backend and constructs a Timeout SolverResponse on its behalf with no `solution` field.
**Fallback:** Any in-progress solution in the backend at termination time is not recoverable; the job records a Timeout with no route plan.
**Worker-constructed response fields:** `contract_version` = the value sent in the SolverRequest; `outcome = Timeout`; `failure` present with `failure_code = ExecutionTimeout`; `solution` absent; `statistics.execution_duration_ms` = the Worker's measured wall-clock time from SolverRequest dispatch to backend termination.

---

### InternalError

**Condition:** Backend encounters an unexpected execution error.
**Behavior:** Backend returns `outcome = Failed`, `failure_code = InternalError`, with `failure_message` describing the error.
**Fallback:** Worker records the failure; job fails.

---

### Unresponsive Backend (Process Crash)

**Condition:** Backend process crashes or becomes unresponsive without producing a SolverResponse.
**Behavior:** Worker detects unresponsiveness (via timeout or process exit signal), constructs a Failed SolverResponse on behalf of the backend.
**Fallback:** Worker records the failure; job fails.
**Note:** This fallback is Worker orchestration behavior. SPEC-004 defines only that a SolverResponse is always consumed by the Worker; the Worker handles backends that fail to produce one.
**Worker-constructed response fields:** `contract_version` = the value sent in the SolverRequest; `outcome = Failed`; `failure` present with `failure_code = InternalError`; `solution` absent; `statistics.execution_duration_ms` = the Worker's measured wall-clock time from SolverRequest dispatch to the detected failure point.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer | Yes — provides the concrete solver interface promised by ADR-008; SPEC-001 FR-15 dependent constraint is now satisfiable |
| API Layer | None — no new API surface |
| Persistence | Yes — SolverResponse fields inform the solver_runs table schema (Evidence Log Specification owns the schema) |
| Solver Runtime | Yes — all MVP solver backends must implement SolverContract |
| Observability | Yes — solver.execute span attributes defined in FR-15 |
| Configuration | Minimal — `contract_version` is a fixed constant at this stage |
| Security | Yes — `extension_metadata` raw-data prohibition; `execution_seed` exclusivity enforcement |

**SPEC-001 FR-15:** The dependent constraint stating "The concrete solver contract interface is a dependent output of this spec and will be defined during implementation planning" is satisfied by SPEC-004. SPEC-001 FR-15 can be marked resolved.

**ADR-008:** The concrete interface promised by ADR-008 is now defined. ADR-008 status can be advanced to Accepted following SPEC-004 acceptance.

---

# Testability

Every solver backend must pass the following contract conformance tests before it is considered contract-conformant. These are test contracts, not test implementations.

1. **Unit: Structural validity of Succeeded responses** — Given a valid SolverRequest, a Succeeded SolverResponse contains a RoutePlan where: (a) `routes.size()` = `vehicle_count`; (b) every stop ID appears exactly once across all routes; (c) no route's total demand exceeds `capacity_per_vehicle`.

2. **Unit: Determinism invariant** — Given two identical SolverRequests (same `routing_problem`, same `execution_seed`), the backend produces identical `solution` fields in both responses. `execution_duration_ms` is excluded from comparison.

3. **Unit: Timeout self-termination** — Given a SolverRequest where `execution_timeout_ms` is set to a value shorter than the backend requires for a complete search, the backend self-terminates and returns `outcome = Timeout` within a reasonable additional delay after the deadline. If a `solution` is included, it satisfies FR-5 structural requirements.

4. **Unit: Cancellation response** — When the Worker signals cancellation, the backend returns `outcome = Cancelled`. If a `solution` is present, it satisfies FR-5 structural requirements.

5. **Unit: ContractVersionMismatch** — A SolverRequest with an unsupported `contract_version` causes the backend to return `outcome = Failed` with `failure_code = ContractVersionMismatch` without processing the routing problem.

6. **Unit: Extension metadata isolation** — Core and Worker contract handling logic produces identical behavior regardless of whether `extension_metadata` is present, absent, empty, or contains unknown keys.

7. **Unit: Trivial case coverage** — A routing problem with one stop and one vehicle produces a Succeeded response with a single route containing that stop.

8. **Unit: Empty vehicle routes** — A routing problem with more vehicles than stops produces a Succeeded response where some routes are empty (no stops). Empty routes are valid.

9. **Unit: No partial assignments** — No Succeeded, Timeout, or Cancelled response contains a RoutePlan with unassigned stops. Every stop in the routing problem appears in exactly one route.

10. **Integration: solver.execute span emission** — A `solver.execute` span is emitted for every SolverRequest, successful or not, with all required attributes present and correlatable via `decision_id`.

11. **Integration: Backend neutrality** — Adding a new backend and registering its capability profile (SPEC-003 FR-4) produces no changes to Scheduler, Worker, or Core contract-handling code. Verified by code review.

12. **Property: Capacity validity across the workload** — For all Succeeded responses produced by every MVP backend across the full SPEC-002 synthetic workload: no route has total demand exceeding `capacity_per_vehicle`. This property must hold across all problem sizes and constraint configurations.

---

# Observability Requirements

Operational questions the `solver.execute` span and persisted SolverResponse fields must answer:

1. For a given job, which backend was selected and what was the execution outcome?
2. How long did solver execution take, disaggregated by backend and problem size class?
3. How often do solver executions timeout or fail, and for which backends?
4. For a given Scheduler decision (`decision_id`), what was the corresponding solver outcome?
5. Is there a correlation between `problem_size_class` and execution duration per backend?
6. How often do Timeout responses include a partial solution vs. no solution?

These are answered by the `solver.execute` span (FR-15) and SolverResponse fields persisted by the Worker through the Evidence Log.

Backend-specific questions (QUBO energy convergence, annealing acceptance rates) are answered by `extension_metadata` in the evidence record. These are not normalized observability requirements.

---

# Security Considerations

**Seed authority (ADR-010 Decision 4):** Backends must not use system entropy sources. Using `std::random_device`, `/dev/urandom`, system time, or process ID in reproducibility-critical paths causes non-reproducible outputs that undermine evidence integrity and violate the architecture.md reproducibility principle.

**Extension metadata safety:** `extension_metadata` must not contain routing problem raw data (geographic coordinates, stop lists, solver input arrays). This prevents sensitive input data from appearing in evidence records, logs, and traces that may have broader read access than the raw routing problem.

**Input trust:** The routing problem arrives at the backend pre-validated by Core (ADR-009). Backends may trust the problem's structural validity. Backends must not re-validate the problem to avoid introducing divergence from Core's canonical validation.

**Python adapter channel:** The transport channel between the Worker and the Python adapter process must be secured in production deployments. MVP Docker Compose security posture is accepted risk per ADR-005.

---

# Performance Considerations

**Execution budget discipline:** Backends that approach the `execution_timeout_ms` deadline should prefer returning a best-found solution over extending the search. Anytime behavior — maintaining a running best-complete-solution returnable at any point — is the preferred implementation approach for iterative improvement solvers.

**RoutePlan structural validation:** The Worker validates FR-5 structural requirements after receiving a Succeeded response. For N stops, this is O(N). No performance concern at MVP scale (maximum 76+ stops per Large class problem).

**Distance computation:** Each backend computes its own pairwise distance matrix from geographic coordinates using Haversine. For N stops plus one depot, this is O(N²). For Large-class problems (76+ stops), this must be measured and its contribution to `execution_duration_ms` must be reported in evidence.

**Extension metadata size:** `extension_metadata` is intended for compact summary values, not large data structures. Backends must not use `extension_metadata` to transmit large arrays or binary data.

---

# Documentation Updates Required

- **ADR-008**: The concrete solver contract interface promised by this ADR is now defined by SPEC-004. ADR-008 may be advanced to Accepted status following SPEC-004 acceptance. At that time, ADR-008 Decision 2 must be revised to read: "A solver returns a route plan, execution duration, and execution statistics. Quality metrics are computed by Core from the normalized route plan output. Execution cost at MVP scope is represented by the static `cost_profile` declared in the backend capability profile (SPEC-003 FR-4)." This revision aligns ADR-008 with the architectural model where Core owns quality evaluation via the normalized contract output.
- **SPEC-001 FR-15**: The dependent constraint ("The concrete solver contract interface is a dependent output of this spec and will be defined during implementation planning") is now satisfied by SPEC-004.
- **SPEC-001 FR-6**: Updated to reflect that the problem seed does not control solver execution directly; the Worker derives `execution_seed` from it (applied in this revision cycle).
- **SPEC-003 FR-4**: The backend capability profile must be extended with a `supported_contract_version` field (uint32, required, current value: 1). The Worker reads this field before dispatch and validates that the version it intends to send matches the declared value. A pre-dispatch mismatch is a Worker configuration error. A post-dispatch mismatch detected by the backend produces `ContractVersionMismatch` per FR-14 (applied in this revision cycle).
- **docs/architecture.md**: No changes required. SPEC-004 conforms to the architecture principles and required spans.
- **Each solver specification** (nearest-neighbor, greedy insertion, QUBO simulated annealing): Must document its seed *usage* policy — that it accepts and uses the Worker-provided `execution_seed` as its exclusive entropy source via PCG64 (ADR-010 Decision 1). Solver specifications do not define a custom derivation from the problem seed; the derivation is the Worker's responsibility. Must also identify which `SolverOutcome` values the backend supports.

---

# Open Questions

### OQ-1: Python Adapter Transport Realization

**Status: RESOLVED — SPEC-017 Accepted (2026-06-20)**

**Resolution:** SPEC-017 (Python Solver Adapter) is Accepted. The Python adapter transport realization is fully defined: JSON over HTTP is the accepted transport (SPEC-017 FR-2, resolving ADR-005 OQ-1). Wire-format representations for all SolverRequest and SolverResponse fields, including the RoutePlan structure, are specified in SPEC-017 FR-3 and FR-4. The cross-language execution boundary is defined by SPEC-017 FR-1. Python adapter behavior is owned by SPEC-017; individual Python backend specifications (SPEC-018+) are children of SPEC-017 and conform to SPEC-011 framework requirements through the mechanisms defined there. ADR-005 is Accepted.

**Ownership:** SPEC-017 owns the transport definition and adapter behavioral contract.

---

### OQ-2: Actual Execution Cost Reporting

**Question:** Should solver backends report actual per-execution measured cost in the SolverResponse, or is the capability profile's declared `cost_profile` (SPEC-003 FR-4) sufficient for evidence purposes?

**Why it matters:** ADR-008 Decision 2 originally stated that "A solver returns a result that includes... execution cost." That language will be revised when ADR-008 advances to Accepted (see Documentation Updates Required), replacing solver-reported cost with the static `cost_profile` at MVP scope. The open question is whether a future iteration with usage-based backends requires actual per-execution cost in the SolverResponse. If actual per-execution cost varies — for example, a cloud-based backend with usage-based pricing — the static `cost_profile` estimate diverges from actual cost, making regret analysis less accurate.

**MVP position:** All MVP backends are local. `cost_profile` is a static proxy. No actual cost measurement mechanism is defined at MVP scope. The Scheduler uses `cost_profile` for BudgetCapped filtering (SPEC-003 FR-5 Phase 2); the decision record records `predicted_cost` (SPEC-003 FR-10). Both are satisfied by the static `cost_profile`.

**If actual cost reporting is needed:** A `reported_cost` field would be added to `ExecutionStatistics` in a non-breaking or breaking change depending on whether it is required vs. optional. This is a SPEC-004 change.

**Ownership:** SPEC-004. Resolution required before any backend with usage-based pricing is introduced.

**Blocking:** Not blocking SPEC-004 acceptance at MVP scope.

---

### OQ-3: Partial Route Plan Semantics

**Question:** Should the contract permit solvers to return partially-assigned route plans (where some stops are unassigned) on Timeout or Cancelled outcomes?

**Why it matters:** Some VRP solver approaches build routes incrementally and may not have assigned all stops when interrupted. Under the current FR-5 definition, partial assignments are prohibited in any solution payload. A solver that can only produce partial assignments must return Timeout or Cancelled with no solution — not a partially-assigned plan. This means the Worker and Core receive no solution to evaluate.

**Current position:** SPEC-004 prohibits partial assignments in any solution payload. A solution is either structurally complete (all stops assigned, capacity valid) or absent. This keeps Core quality evaluation logic simple and consistent.

**If partial assignments are needed:** A new `SolverOutcome` value (e.g., `PartialSolution`) with relaxed FR-5 requirements would be required, plus a Core quality evaluation extension to handle incomplete plans. This is a breaking contract change.

**MVP assumption:** All MVP backends (nearest-neighbor, greedy insertion, QUBO simulated annealing) are expected to produce complete route assignments or no assignment. Partial assignment support is not required for MVP.

**Ownership:** SPEC-004. Resolution required if a backend is introduced that cannot produce complete assignments.

**Blocking:** Not blocking SPEC-004 acceptance.

---

### OQ-4: RoutePlan Structural Validation on Timeout and Cancelled Responses

**Status: CLOSED**

**Resolution:** The SPEC-004 contract requirement is resolved: any solution present in a Timeout or Cancelled response must satisfy all FR-5 structural validity requirements. This is stated in FR-5 and is a SPEC-004 contract obligation, not an open question.

The remaining question — what the Worker does upon detecting a structural violation in a Timeout or Cancelled solution (discard the solution and retain the Timeout outcome? treat the invocation as Failed?) — is Worker orchestration behavior outside SPEC-004's responsibility boundary per FR-1. It is deferred to the Worker specification.

**Ownership:** Worker specification.

---

# Acceptance Checklist

- [x] Problem is clearly defined
- [x] Domain concept is defined
- [x] Responsibility scope and component boundaries are defined (FR-1)
- [x] Solver request is defined (FR-2)
- [x] Solver response is defined (FR-3)
- [x] Solver outcome model is defined (FR-4)
- [x] Route plan structure is defined (FR-5)
- [x] Execution statistics are defined (FR-6)
- [x] Success semantics are defined (FR-7)
- [x] Failure semantics are defined (FR-8)
- [x] Cancellation semantics are defined (FR-9)
- [x] Timeout behavior is defined (FR-10)
- [x] Determinism expectations are defined (FR-11)
- [x] Backend neutrality requirements are defined (FR-12)
- [x] Extension metadata is defined (FR-13)
- [x] Contract versioning and compatibility are defined (FR-14)
- [x] Telemetry requirements are defined (FR-15)
- [x] Non-requirements are documented
- [x] Assumptions are explicit
- [x] Failure modes are defined
- [x] Observability requirements exist
- [x] Security considerations exist
- [x] Documentation updates are identified
- [x] OQ-1 resolved — Python adapter transport realization. Resolved by SPEC-017 (Accepted 2026-06-20); JSON over HTTP transport and wire format defined in SPEC-017 FR-2 through FR-4. ADR-005 Accepted.
- [ ] OQ-2 resolved — actual execution cost reporting (non-blocking for MVP)
- [ ] OQ-3 resolved — partial route plan semantics (non-blocking for MVP)
- [x] OQ-4 closed — Contract requirement resolved in SPEC-004 (solutions in Timeout/Cancelled responses must satisfy FR-5 structural requirements). Worker handling behavior upon violation is a Worker specification concern.

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-15) are implemented and acceptance criteria pass
- All MVP solver backends (nearest-neighbor, greedy insertion, QUBO simulated annealing) implement and pass contract conformance tests defined in the Testability section
- `solver.execute` OTel span is emitted and verifiable in the test environment
- ADR-008 status is updated to Accepted following this specification being accepted
- SPEC-001 FR-15 dependent constraint is marked resolved
- OQ-1 is resolved — Python adapter transport and wire format defined in SPEC-017 (Accepted 2026-06-20); JSON over HTTP transport specified in SPEC-017 FR-2 through FR-4; individual Python backend specifications (SPEC-018+) may now be written
- Engineering review passes
- Specification status is updated to Verified
