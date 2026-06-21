# SPEC-017: Python Solver Adapter

## Metadata

**Feature ID:** SPEC-017

**Title:** Python Solver Adapter

**Status:** Accepted

**Author:** Darkhorse286

**Created:** 2026-06-20

**Last Updated:** 2026-06-20 (Accepted — Engineering Review, Owner Decision Review, Architecture Review, and Acceptance Review complete; all blocking findings resolved; ADR-005 advanced to Accepted; OQ-4 resolved)

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-001, ADR-003, ADR-004, ADR-005, ADR-006, ADR-008, ADR-009, ADR-010, ADR-011

**Related Specs:** SPEC-001, SPEC-003, SPEC-004, SPEC-005, SPEC-006, SPEC-011

---

# Problem Statement

SPEC-004 OQ-1 identifies the Python adapter transport realization as a blocking dependency for any Python-based backend implementation. ADR-005 deferred the transport decision, establishing only that Python is confined to a separate container process accessible from the Daedalus Worker. As of this specification, no transport protocol has been defined between the Worker and the Python adapter, no wire format has been specified for the SolverRequest and SolverResponse across the process boundary, and no behavioral contract governs Python environment management, timeout ownership, cancellation, reproducibility, or observability for Python-based solver backends.

Without SPEC-017:

- No Python backend (QAOA, Qiskit-based, future Python frameworks) can be specified or implemented
- The Worker has no defined dispatch path for Python-backed Scheduler decisions
- The reproducibility obligations of ADR-010 have no Python-language equivalent
- The observability contract has no Python adapter chapter
- SPEC-004 OQ-1 remains unresolved and blocks Python adapter backend implementation

SPEC-017 defines the observable behavioral contract for the Python Solver Adapter. It resolves ADR-005 OQ-1, specifies the JSON wire format, defines the process model, establishes timeout and cancellation obligations, and maps ADR-010 reproducibility requirements to Python-native equivalents. Individual Python backend solver specifications (SPEC-018 and beyond) are children of this specification and conform to the SPEC-011 framework requirements through the mechanisms defined here.

---

# Business Value

- Unblocks all Python-based solver backends: QAOA, Qiskit-based solvers, future Python quantum frameworks
- Resolves ADR-005 OQ-1 and SPEC-004 OQ-1 — two previously blocking open questions
- Enables portfolio demonstration of Python-based quantum-inspired solver integration within a production-style runtime without compromising the Worker's execution lifecycle ownership
- Preserves the Worker's authoritative ownership of timeout enforcement, seed derivation, and execution lifecycle (SPEC-005)
- Establishes the Python-side reproducibility contract, extending the ADR-010 reproducibility guarantees across the adapter process boundary

---

# Employer Signaling

- System Design
- Distributed Systems
- AI Engineering
- Reliability Engineering
- Observability

SPEC-017 demonstrates the ability to specify a cross-language adapter contract that preserves a formal solver interface (SPEC-004) across a process boundary. The challenge is non-trivial: the adapter must honor timeout and cancellation signals mediated through an HTTP transport, preserve reproducibility guarantees defined in a C++-centric policy (ADR-010) within a Python runtime, and maintain the Worker's observability model without emitting telemetry of its own. Specifying these obligations as observable behaviors — without prescribing Python internal implementation — requires clear separation of contract from implementation.

---

# Domain Concept

The Python Solver Adapter is a long-running HTTP service running in a separate Docker Compose container process. When the Scheduler selects a Python-based solver backend, the Worker dispatches a SolverRequest to the adapter over HTTP, the adapter executes the appropriate Python solver, and the adapter returns a SolverResponse. The logical SolverContract (SPEC-004) is preserved across the process boundary: the adapter presents the same normalized request-response interface to the Worker as any in-process C++ backend, expressed through a JSON wire format.

The adapter is not an algorithmic backend. It is a transport and dispatch layer. The specific optimization algorithms — QAOA circuits, Qiskit execution, other Python frameworks — belong to individual Python backend solver specifications (SPEC-018 and beyond), which are children of this specification and conform to the SPEC-011 framework requirements.

The adapter is stateless between requests. Each HTTP request is an independent solver execution. The Worker's execution lifecycle ownership (SPEC-005) is unchanged: the Worker enforces timeouts externally, owns execution seed derivation, holds the RabbitMQ acknowledgment until the job is complete, and emits all required OTel spans. The adapter is an extension of the solver execution path, not an expansion of execution ownership.

Python is confined to the adapter role per ADR-005. The adapter is not the runtime, scheduler, evidence layer, or Worker.

---

# Requirements

### FR-1: Responsibility Scope and Component Boundaries

**Description:**
SPEC-017 defines the Python Solver Adapter's responsibilities and explicitly allocates adjacent responsibilities to the correct owners.

| Component | Responsibilities at This Boundary | Explicitly Not Responsible For |
|---|---|---|
| **Worker** (SPEC-005) | Dispatching SolverRequest JSON to adapter via HTTP POST; enforcing execution timeout externally via HTTP client timeout; signaling in-execution cancellation by aborting the HTTP connection; constructing fallback SolverResponse on adapter failure or HTTP client timeout; emitting all required OTel spans; pre-execution cancellation check before dispatch | Launching or stopping the adapter process; managing the Python environment; internal adapter routing between backends |
| **Python Solver Adapter** (SPEC-017) | Receiving and deserializing SolverRequest JSON; routing to the appropriate Python solver by `backend_id`; self-terminating the active Python solver before `execution_timeout_ms` expires; handling client disconnect (in-execution cancellation); serializing and returning SolverResponse JSON (HTTP 200); managing HTTP service lifecycle; managing Python dependencies | Execution seed derivation; RabbitMQ lifecycle; evidence persistence; quality evaluation; OTel span emission; launching as a subprocess of the Worker |
| **Individual Python Backend** (SPEC-018+) | Implementing the logical SolverContract in Python; consuming `execution_seed` for NumPy PCG64 seeding; producing a RoutePlan conforming to SPEC-004 FR-5; declaring `extension_metadata` per its specification | HTTP transport; JSON serialization; timeout monitoring; health checking; client disconnect detection |
| **Worker** (timeout authority, SPEC-005 FR-10) | External timeout enforcement: HTTP client timeout fires if adapter has not responded within `execution_timeout_ms + transport_overhead_buffer_ms`; Worker constructs Timeout SolverResponse on HTTP timeout (no solution) | Adapter-internal self-termination logic |

**Acceptance Criteria:**
- The Worker does not launch, stop, or restart the adapter process
- The adapter does not derive `execution_seed`; it receives it in the SolverRequest JSON and forwards it to the Python solver unchanged
- The adapter does not emit OpenTelemetry spans (per SPEC-011 FR-9.2)
- Adding a new Python backend requires only: an accepted individual Python backend specification (SPEC-018+), a backend implementation, and registration in the adapter's routing table — no changes to the Worker, Core, Scheduler, or SPEC-004 contract handling logic are required
- The Worker does not branch on `backend_id` in contract handling or result processing; the adapter performs all backend routing internally

---

### FR-2: Transport Protocol

**Description:**
This section resolves ADR-005 OQ-1. The Python Solver Adapter uses **JSON over HTTP** as the transport protocol between the Daedalus Worker and the Python adapter container.

**Rationale for JSON over HTTP over the alternatives:**
- JSON over HTTP is debuggable: requests and responses are observable as plain text without special tooling
- JSON over HTTP is testable: standard HTTP testing applies without generated client stubs
- At MVP routing problem payload sizes (maximum approximately 100 stops with coordinate arrays), JSON serialization overhead is acceptable and must be measured (see Performance Considerations)
- Python's HTTP server ecosystem is mature (FastAPI, Flask, stdlib); no additional framework dependencies are required
- HTTP status codes provide a natural signal for adapter-level versus solver-level failures (FR-12)

gRPC is not selected for MVP: the strongly typed contract benefit does not outweigh the additional code generation tooling, protobuf dependency, and debugging complexity at MVP scale. stdin/stdout IPC is not selected: it would require the Worker to manage the Python process lifetime, contradicting the separate container model established by ADR-005.

**Endpoint contract:**

| Endpoint | Method | Purpose |
|---|---|---|
| `/v1/solve` | POST | Submit a SolverRequest; receive a SolverResponse |
| `/health` | GET | Adapter liveness and readiness check |

**Port:** The adapter listens on port 8080 within the Docker Compose internal network.

**Content-Type:** `application/json` for request and response bodies on `/v1/solve`.

**HTTP success semantics:** The adapter returns HTTP 200 whenever it successfully produces a SolverResponse — including `Failed`, `Timeout`, and `Cancelled` outcomes. HTTP 200 signals that a SolverResponse was produced. HTTP non-200 signals that the adapter failed to produce any SolverResponse (FR-12).

**Acceptance Criteria:**
- The Worker dispatches all Python-backend solver invocations as HTTP POST requests to `http://{python-adapter-hostname}:8080/v1/solve`
- The adapter returns HTTP 200 for every SolverResponse payload regardless of `outcome`
- HTTP non-200 from the adapter is treated by the Worker as an adapter-level failure, not a solver-level failure (FR-12)
- The adapter responds to `GET /health` with HTTP 200 when the service is operational and ready

---

### FR-3: JSON Wire Format — SolverRequest

**Description:**
The SolverRequest JSON payload is the complete wire-format representation of the SPEC-004 FR-2 SolverRequest dispatched by the Worker. All SPEC-004 FR-2 fields are present in every request.

**Top-level field encoding:**

| SPEC-004 Field | JSON Key | JSON Type | Encoding Notes |
|---|---|---|---|
| `contract_version` | `contract_version` | number | Decimal integer (uint32) |
| `routing_problem` | `routing_problem` | object | See RoutingProblem encoding below |
| `execution_timeout_ms` | `execution_timeout_ms` | number | Decimal integer (uint32); always > 0 |
| `execution_seed` | `execution_seed` | string | Decimal string encoding of uint64. JSON number precision is insufficient for the full uint64 range (JSON numbers are IEEE 754 double, 53-bit mantissa). Example: `"13835058055282163712"` |
| `job_id` | `job_id` | string | Hyphenated lowercase UUID. Example: `"550e8400-e29b-41d4-a716-446655440000"` |
| `decision_id` | `decision_id` | string | Hyphenated lowercase UUID |
| `backend_id` | `backend_id` | string | Kebab-case backend identifier matching the registered Python backend |

**RoutingProblem object encoding:**

The `routing_problem` object encodes all fields present in the SPEC-001 C++ domain representation (SPEC-001 FR-14).

| Domain Field | JSON Key | JSON Type | Encoding Notes |
|---|---|---|---|
| Problem seed (SPEC-001 FR-6) | `seed` | string | Decimal string encoding of uint64. Must not be used as PRNG entropy; `execution_seed` is the exclusive entropy source (FR-9, SPEC-004 FR-11.2) |
| Depot (SPEC-001 FR-3) | `depot` | object | `{"latitude": <float64>, "longitude": <float64>}` |
| Stops (SPEC-001 FR-4) | `stops` | array of objects | Ordered by `stop_id` ascending (reproducibility constraint; see note below). See Stop encoding |
| Vehicle count | `vehicle_count` | number | Decimal integer (uint32) |
| Capacity per vehicle | `capacity_per_vehicle` | number | Decimal integer (uint32) |
| Average vehicle speed (ODR-1) | `average_vehicle_speed_kmh` | number | IEEE 754 double |

**Stop array ordering:** The `stops` array is always ordered by `stop_id` ascending. This ordering is a reproducibility constraint: any Python backend that iterates over stops in array order produces the same iteration sequence for a given routing problem across all invocations. Changing this ordering in a future wire format revision is a reproducibility-breaking change subject to ADR-010 Decision 5 governance.

**Stop object encoding:**

| Field | JSON Key | JSON Type | Encoding Notes |
|---|---|---|---|
| stop_id (SPEC-001 FR-4) | `stop_id` | number | Non-negative decimal integer (uint32) |
| Latitude | `latitude` | number | IEEE 754 double (degrees) |
| Longitude | `longitude` | number | IEEE 754 double (degrees) |
| Demand | `demand` | number | Decimal integer (uint32) |
| Time window open (SPEC-001 FR-9) | `time_window_open` | number | Decimal integer (uint32); seconds from midnight |
| Time window close (SPEC-001 FR-9) | `time_window_close` | number | Decimal integer (uint32); seconds from midnight |
| Service duration (SPEC-001 FR-16) | `service_duration_seconds` | number | Decimal integer (uint32) |

**Acceptance Criteria:**
- The adapter deserializes `execution_seed` from its string encoding to a Python integer before passing it to the Python solver; no precision loss occurs
- The adapter returns HTTP 400 if `contract_version` is absent from the request body
- The adapter returns HTTP 400 if `routing_problem`, `execution_seed`, `execution_timeout_ms`, `job_id`, `decision_id`, or `backend_id` are absent
- `routing_problem.seed` is included in the payload but must not be used as a PRNG entropy source by the adapter or any Python backend; only `execution_seed` seeds the PRNG (SPEC-004 FR-11.2)

---

### FR-4: JSON Wire Format — SolverResponse

**Description:**
The SolverResponse JSON payload is the complete wire-format representation of the SPEC-004 FR-3 SolverResponse returned by the adapter in the HTTP 200 body.

**Top-level field encoding:**

| SPEC-004 Field | JSON Key | JSON Type | Present When |
|---|---|---|---|
| `contract_version` | `contract_version` | number | Always; must equal `contract_version` from the corresponding request |
| `outcome` | `outcome` | string | Always; one of the values in the SolverOutcome encoding table below |
| `solution` | `solution` | object | When a RoutePlan is present; absent or null otherwise |
| `statistics` | `statistics` | object | Always |
| `failure` | `failure` | object | When `outcome` ≠ `"Succeeded"`; absent or null when `outcome` = `"Succeeded"` |
| `extension_metadata` | `extension_metadata` | object | Optional; absent when no backend-specific metadata is produced |

**SolverOutcome string values:**

| SolverOutcome | JSON String Value |
|---|---|
| `Succeeded` | `"Succeeded"` |
| `Timeout` | `"Timeout"` |
| `Cancelled` | `"Cancelled"` |
| `Failed` | `"Failed"` |
| `Infeasible` | `"Infeasible"` |

**Note on `"Infeasible"` outcome:** `"Infeasible"` is listed for completeness per SPEC-004 FR-4. Heuristic Python backends are prohibited from returning this outcome per SPEC-011 FR-5.2 and Constraint 12. Only Python backends capable of formally proving infeasibility may return `"Infeasible"`.

**RoutePlan (solution) object encoding:**

```json
{
  "routes": [
    [0, 2, 5],
    [1, 3],
    []
  ]
}
```

`routes` is an array of exactly `vehicle_count` arrays. Each inner array contains `stop_id` integers in visit order. An empty inner array represents an unused vehicle. The depot is not included in any route (SPEC-004 FR-5).

**ExecutionStatistics object encoding:**

```json
{
  "execution_duration_ms": 2345,
  "solution_count": 3
}
```

`execution_duration_ms` is required (non-negative integer). `solution_count` is optional; absent when the backend does not track it.

**`execution_duration_ms` for adapter-constructed failures:** When the adapter returns a failure response for a condition detected before solver execution begins (e.g., `ContractVersionMismatch`, unrecognized `backend_id`, module import failure at request time), `execution_duration_ms` represents elapsed time from request receipt to failure detection, as measured by the adapter. It does not represent solver computation time, as none occurred.

**SolverFailureDetail object encoding:**

```json
{
  "failure_code": "InternalError",
  "failure_message": "Diagnostic description of the failure"
}
```

`failure_code` string values: `"InfeasibleProblem"`, `"ExecutionTimeout"`, `"ExecutionCancelled"`, `"ContractVersionMismatch"`, `"InternalError"`.

**extension_metadata object encoding:**

```json
{
  "key1": "value1",
  "key2": "value2"
}
```

String-to-string map. Absent when no extension metadata is produced. Semantics and key names are defined by individual Python backend specifications (SPEC-018+) per SPEC-011 FR-7.4.

**Acceptance Criteria:**
- `contract_version` in the response always equals `contract_version` in the corresponding request
- `outcome` is always one of the five defined string values
- `routes` array length equals `vehicle_count` from the routing problem in every response that includes a `solution`
- `failure` is absent (or null) when `outcome` = `"Succeeded"`; `failure` is present when `outcome` ≠ `"Succeeded"`
- `execution_duration_ms` is present in every response
- No route in `routes` contains a stop_id that appears in another route
- Every stop_id from the routing problem appears in exactly one route in every present `solution` (these constraints are validated by the Worker's post-receipt structural validation per SPEC-005 FR-11)

---

### FR-5: Timeout Behavior

**Description:**
Timeout ownership follows SPEC-004 FR-10 and SPEC-005 FR-10. The Worker remains authoritative for external timeout enforcement. The adapter is responsible for self-terminating the Python solver before the `execution_timeout_ms` deadline.

**Adapter self-termination obligation:**
1. The adapter measures elapsed time from the moment Python solver execution begins — defined as after JSON deserialization, backend routing, contract version validation, and PRNG initialization are complete.
2. The adapter MUST self-terminate the active Python solver computation before the `execution_timeout_ms` deadline.
3. Self-termination monitoring is implemented at solver-specific checkpoints defined in individual Python backend specifications. The specific interrupt mechanism (threading events, process signals, cooperative yield points) is a backend implementation concern; the observable requirement is that the adapter responds to the deadline.
4. Upon self-termination, the adapter serializes and returns HTTP 200 with a Timeout SolverResponse. If the backend implements anytime behavior and a complete, structurally valid RoutePlan exists at termination time, it is included. If not, the solution is absent.
5. A self-produced Timeout response carries `outcome = "Timeout"` and `failure_code = "ExecutionTimeout"`.

**Worker HTTP client timeout (external backstop):**
1. The Worker sets an HTTP client timeout of `execution_timeout_ms + transport_overhead_buffer_ms` milliseconds.
2. `transport_overhead_buffer_ms` is a Worker configuration value accounting for JSON serialization time, Docker Compose network latency, and adapter response finalization. Its specific value is an implementation planning concern (OQ-1).
3. If the HTTP client timeout fires before the adapter responds, the Worker treats this as a Worker-enforced timeout. The Worker constructs a Timeout SolverResponse per SPEC-004 Failure Modes (ExecutionTimeout Worker-Enforced): `outcome = Timeout`, `failure_code = ExecutionTimeout`, no `solution`, `execution_duration_ms` = Worker-measured wall-clock time from HTTP dispatch to timeout.
4. A Worker-constructed Timeout response never includes a solution, regardless of what the Python backend may have found. Self-termination before the HTTP client timeout fires is the preferred path.

**Why self-termination is preferred:** If the adapter self-terminates and returns a response, any best-so-far solution from a Python backend with anytime behavior is preserved. A Worker HTTP client timeout abort discards that solution. Adapter self-termination before `execution_timeout_ms` is a behavioral obligation, not a best-effort goal.

**Acceptance Criteria:**
- The adapter self-terminates the active Python solver before `execution_timeout_ms` expires
- A self-produced Timeout response returns HTTP 200 with `outcome = "Timeout"` and `failure_code = "ExecutionTimeout"`
- A Worker-constructed Timeout response (from HTTP client timeout) contains no `solution`
- `execution_timeout_ms = 0` is never dispatched by the Worker; the adapter may return HTTP 400 on receipt of `execution_timeout_ms = 0` as a defensive check
- `execution_duration_ms` in any adapter-produced SolverResponse with a solved outcome reflects wall-clock time from the moment Python solver computation begins (after JSON deserialization, backend routing, contract version validation, and PRNG initialization) to the moment the adapter begins constructing the SolverResponse; it excludes deserialization, routing, contract validation, PRNG initialization, and response serialization time

---

### FR-6: Cancellation Behavior

**Description:**
Cancellation follows the SPEC-005 FR-12 model. Pre-execution cancellation eliminates the HTTP dispatch entirely. In-execution cancellation is delivered by the Worker aborting the HTTP connection.

**Pre-execution cancellation (SPEC-005 FR-12, ODR-5):**
The Worker checks the `cancellation_requested` flag in the PostgreSQL job record after decision record persistence and before dispatching the HTTP POST to the adapter. If `cancellation_requested = true`, the Worker does not dispatch the HTTP request. No adapter interaction occurs. The Worker constructs a Cancelled SolverResponse per SPEC-005 FR-12.

**In-execution cancellation:**
If `cancellation_requested` is written to PostgreSQL after the HTTP POST is in flight:
1. The Worker aborts the HTTP connection. This is the cancellation signal to the adapter.
2. The adapter detects client disconnect (HTTP connection closed by the Worker before a response was returned).
3. The adapter interrupts the active Python solver computation.
4. The adapter cleans up any in-progress state.
5. No SolverResponse is returned (the connection is closed). The Worker constructs a Cancelled SolverResponse per SPEC-005 FR-12 (Worker-constructed Cancelled: no `solution`, `execution_duration_ms` = Worker-measured elapsed time from dispatch to abort).

**Adapter obligations on client disconnect:**
- The adapter must handle client disconnect without crashing or leaving the process in an inconsistent state.
- The adapter must interrupt active Python solver computation within a bounded time after detecting client disconnect. The time from disconnect detection to signaling the solver to stop is an adapter implementation planning concern (OQ-3A). The time from the adapter's stop signal to the backend actually halting is a per-backend specification concern (OQ-3B).
- After handling a disconnect, the adapter must return to its ready state and be able to accept the next request.
- No in-progress solver computation state from a disconnected request is carried into subsequent requests (statelessness, FR-7).

**Acceptance Criteria:**
- Pre-execution cancellation (SPEC-005 FR-12) produces no HTTP dispatch; adapter is not contacted
- The Worker aborts the HTTP connection to signal in-execution cancellation
- The adapter handles client disconnect without crashing
- The adapter returns to ready state after a client disconnect and accepts subsequent requests
- A Worker-constructed Cancelled response from in-execution cancellation contains no `solution`
- In-execution cancellation results in a `Completed` job with `outcome = Cancelled` in the evidence record

---

### FR-7: Adapter Statelessness

**Description:**
The Python Solver Adapter is stateless between requests.

**Statelessness obligations:**
1. No execution state from one request is accessible during or after a subsequent request.
2. No solver PRNG state, cached computation results, partial solutions, or best-so-far tracking structures persist between requests. These are initialized fresh from the SolverRequest for every execution.
3. The NumPy PCG64 generator is initialized from `execution_seed` at the start of each solver execution and is not reused across requests.
4. The adapter Python process may retain loaded modules, compiled circuit templates, or static model artifacts across requests if and only if these cached values do not depend on any SolverRequest field. Caching request-independent initialization data (e.g., a compiled Qiskit circuit template for a fixed problem structure) is permitted as an implementation optimization. All such caches must be immutable once populated and must not vary by request.

**Concurrency:** At MVP scope, the adapter processes one request at a time per container instance (consistent with SPEC-005 Assumption 6: one Worker instance, one job at a time). Behavior when a second request arrives while a request is in progress is an implementation planning concern (OQ-2).

**Acceptance Criteria:**
- Two sequential requests with identical SolverRequest content produce identical SolverResponse `solution` fields when both produce `outcome = "Succeeded"` (subject to the reproducibility invariant in FR-9)
- The NumPy PCG64 generator initialized for request N is not used during request N+1
- A cancelled or timed-out request does not affect the outcome of a subsequent request with a different `execution_seed`

---

### FR-8: Python Environment Management

**Description:**
The adapter manages its own Python environment within the container image. The Worker has no visibility into or responsibility for the Python environment.

**Build-time environment obligations:**
1. All Python dependencies required by the adapter and all hosted Python backends are installed at container image build time.
2. Dependency versions are pinned in a requirements file (`requirements.txt` or equivalent) committed to version control. Runtime `pip install` during adapter startup or during request handling is prohibited.
3. The adapter container image specifies its Python version explicitly (e.g., `FROM python:3.11-slim`).
4. The adapter validates that all required modules are importable at startup, before the HTTP service begins accepting requests.

**Module import failure — startup:**
If a required module fails to import during adapter startup:
1. The adapter logs a structured error identifying the missing module.
2. The adapter does not start the HTTP service.
3. The adapter process exits with a non-zero exit code.
4. The Docker Compose health check reports unhealthy, preventing the Worker from dispatching requests (FR-14).

**Module import failure — request-time (late-loading scenario):**
If a module that was not loaded at startup fails to import during request handling:
1. The adapter returns HTTP 200 with `outcome = "Failed"`, `failure_code = "InternalError"`, and `failure_message` identifying the import failure.
2. The job proceeds to evidence persistence via the Worker's standard Failed handling.

**Acceptance Criteria:**
- All Python dependencies are pinned in a requirements file at container build time
- The HTTP service does not accept requests until all required modules are verified importable at startup
- A startup module import failure prevents the HTTP service from starting and results in a non-zero container exit code
- A request-time module import failure returns HTTP 200 with a Failed SolverResponse carrying `failure_code = InternalError`

---

### FR-9: Reproducibility — Seed Propagation and Python PRNG Policy

**Description:**
The Python Solver Adapter must propagate `execution_seed` to Python solver backends unchanged and must establish a Python-native reproducibility contract equivalent in intent to ADR-010.

**Seed propagation:**
The `execution_seed` value received in the SolverRequest JSON (a decimal string encoding of a uint64) is parsed to a Python integer (arbitrary precision) and passed to the Python solver backend without transformation, truncation, or additional derivation. The adapter does not modify or re-derive `execution_seed`.

**Python PRNG policy — binding on all Python backends hosted by the adapter:**

| Requirement | Obligation |
|---|---|
| Exclusive entropy source | `execution_seed` from the SolverRequest is the exclusive authorized entropy source for all reproducibility-critical stochastic operations |
| Prohibited entropy sources in reproducibility-critical paths | `random.random()`, `random.seed()`, `os.urandom()`, `secrets.token_bytes()`, `time.time()`, `os.getpid()`, `numpy.random.default_rng()` without explicit seed, hardware entropy sources |
| PRNG algorithm | NumPy PCG64 initialized via SeedSequence: `rng = numpy.random.default_rng(numpy.random.PCG64(numpy.random.SeedSequence(execution_seed, spawn_key=(BACKEND_SPAWN_KEY,))))` |
| PRNG initialization | Seeded exactly once per solver execution from `execution_seed`; the PRNG is not re-seeded during a single execution |
| Backend spawn key | Each Python backend specification (SPEC-018+) must declare a unique positive integer `BACKEND_SPAWN_KEY`. The spawn key is frozen once the backend specification is Accepted; changing it is a reproducibility-breaking change governed by ADR-010 Decision 5. Spawn keys are not required to be odd. Spawn keys are not numerically comparable to C++ PCG64 stream constants; they belong to different realization mechanisms (SeedSequence hash-based initialization vs. direct PCG64 increment assignment) |
| Distribution sampling | Python backends must use the NumPy Generator interface (`rng.integers()`, `rng.uniform()`, `rng.standard_normal()`). Python's `random` module is prohibited in reproducibility-critical paths. ADR-010 Decision 3 distribution algorithm mandates apply to C++ backends; SPEC-017 FR-9 is the Python-language adaptation of the ADR-010 reproducibility policy |

**Qiskit randomness — obligation for Qiskit-based Python backends:**
Qiskit introduces multiple randomness sources. Python backend specifications for Qiskit-based solvers (SPEC-018+) must document and control all Qiskit randomness sources. At minimum, each Qiskit backend specification must address:
- Transpiler randomness (`seed_transpiler` parameter or equivalent)
- Simulator/estimator randomness (`seed_simulator` or shot-based noise seeds)
- Classical optimizer randomness, if the optimizer is stochastic (e.g., SPSA, gradient-free optimizers with random initialization)

Each Qiskit-specific seed must be derived deterministically from `execution_seed`. The specific derivation for each seed source is an individual Python backend specification concern (SPEC-018+). SPEC-017 establishes the obligation; individual backend specifications satisfy it.

**Cross-language reproducibility:**
Python backends initialize PCG64 via SeedSequence with a backend-specific spawn key. C++ backends initialize PCG64 directly with a stream constant (increment). These are distinct initialization mechanisms; the resulting PCG64 sequences are not numerically comparable across languages for the same seed value, and spawn keys are not in the same constant space as C++ stream constants. Cross-language reproducibility — the same `execution_seed` producing the same route plan from both a C++ backend and a Python backend — is not required and is architecturally unexpected, since different backends implement different algorithms. The SPEC-004 FR-11 reproducibility invariant applies within each language independently: two invocations of the same Python backend with the same `execution_seed` must produce the same solution.

**Reproducibility invariant (Python-side):**
Given two SolverRequests with identical `routing_problem` fields and identical `execution_seed`, a Python backend must produce identical `outcome` and identical `solution` on both invocations when both produce `outcome = "Succeeded"`. This invariant must hold across sequential requests to the same adapter container instance and across container restarts (given the same image and dependencies).

**Acceptance Criteria:**
- `execution_seed` is passed to the Python solver as a Python integer, without transformation or precision loss
- The adapter and all Python backends prohibit `random.random()`, `os.urandom()`, `time.time()` as entropy sources in reproducibility-critical paths
- NumPy PCG64 is initialized exactly once per solver execution from `execution_seed`
- Two identical SolverRequests produce identical `solution` fields in both responses when both produce `outcome = "Succeeded"`
- `routing_problem.seed` is never used as a PRNG entropy source by the adapter or any hosted Python backend
- Each Python backend specification (SPEC-018+) declares a unique positive integer `BACKEND_SPAWN_KEY` that is frozen on Acceptance and governs the PCG64 initialization for that backend

---

### FR-10: Contract Version Handling

**Description:**
The adapter validates `contract_version` in the SolverRequest against each registered Python backend's declared `supported_contract_version` before beginning solver execution.

**Validation sequence:**
1. The adapter reads `contract_version` from the SolverRequest after successful deserialization.
2. The adapter identifies the target Python backend from `backend_id`.
3. The adapter compares `contract_version` against the backend's declared `supported_contract_version`.
4. Mismatch: the adapter returns HTTP 200 with `outcome = "Failed"` and `failure_code = "ContractVersionMismatch"`. The `failure_message` identifies the received version and the expected version. No solver computation begins.
5. Match: the adapter proceeds with solver execution.

Contract version validation is the first check performed after request deserialization and backend identification. It occurs before any PCG64 initialization or routing problem processing (per SPEC-011 FR-8.3).

**Acceptance Criteria:**
- A SolverRequest with an unsupported `contract_version` returns HTTP 200 with `outcome = "Failed"` and `failure_code = "ContractVersionMismatch"` before any solver computation begins
- `failure_message` identifies both the received `contract_version` and the backend's `supported_contract_version`
- Contract version validation occurs before NumPy PCG64 initialization

---

### FR-11: Backend Routing

**Description:**
The adapter routes incoming SolverRequests to the appropriate Python solver backend using the `backend_id` field.

**Routing obligations:**
1. The adapter maintains a registry of Python backend identifiers it hosts. The registry is populated at adapter startup and does not change at runtime.
2. On receiving a SolverRequest, the adapter reads `backend_id` and invokes the corresponding Python solver function or class.
3. Unrecognized `backend_id`: the adapter returns HTTP 200 with `outcome = "Failed"`, `failure_code = "InternalError"`, and `failure_message` identifying the unrecognized value.
4. The `backend_id` field is used exclusively for internal routing. The adapter does not use `backend_id` in contract handling logic, timeout monitoring, serialization, or any other path beyond the initial routing decision.

**Acceptance Criteria:**
- An unrecognized `backend_id` returns HTTP 200 with `outcome = "Failed"` and `failure_code = "InternalError"`
- `backend_id` is not used in the adapter's contract handling, timeout monitoring, or serialization logic beyond routing to the Python solver function

---

### FR-12: Adapter-Level Error Handling

**Description:**
The adapter returns HTTP 200 whenever it successfully produces a SolverResponse. HTTP non-200 indicates that the adapter itself failed to produce any SolverResponse.

**HTTP 200 invariant:**
Every SolverResponse payload — including `Failed`, `Timeout`, and `Cancelled` outcomes — is returned with HTTP 200. The SolverResponse's `outcome` and `failure` fields carry the solver-level result. HTTP status codes are used only to distinguish adapter-level failures (non-200) from solver-level outcomes (always 200).

**Adapter-level failure conditions and HTTP status codes:**

| Condition | HTTP Status | Worker Action |
|---|---|---|
| Request body is not valid JSON | 400 | Worker constructs Failed SolverResponse with `InternalError`; job reaches Completed |
| Required SolverRequest field is absent | 400 | Worker constructs Failed SolverResponse with `InternalError`; job reaches Completed |
| Unhandled adapter process error (before response is written) | 500 or connection drop | Worker constructs Failed SolverResponse with `InternalError`; job reaches Completed |
| Adapter not running (connection refused) | N/A (connection error) | Worker treats as transient infrastructure failure; NACK with requeue (SPEC-005 FR-5) |
| HTTP client timeout fires (adapter did not respond in time) | N/A (timeout) | Worker constructs Timeout SolverResponse per FR-5; no solution; job reaches Completed |

**Worker handling of non-200 responses:**
- HTTP 4xx: Worker constructs Failed SolverResponse with `failure_code = InternalError`. The job proceeds to evidence persistence. The job reaches `Completed` with a Failed solver outcome.
- HTTP 5xx or connection drop mid-response: Worker constructs Failed SolverResponse with `failure_code = InternalError`. The job proceeds to evidence persistence. Job reaches `Completed`.
- Connection refused: treated as an infrastructure failure. Worker NACKs with requeue per SPEC-005 FR-5. The job does not immediately reach `Failed`; it is retried when the adapter recovers.

**Acceptance Criteria:**
- The adapter returns HTTP 200 for every SolverResponse payload including `Failed`, `Timeout`, and `Cancelled`
- The adapter returns HTTP 400 for malformed request bodies (non-JSON, missing required fields)
- Connection refused is treated by the Worker as a transient infrastructure failure and does not immediately mark the job `Failed`

---

### FR-13: Observability

**Description:**
The Worker emits the `solver.execute` OTel span for every adapter invocation (SPEC-004 FR-15). The Python adapter does not emit OTel spans (SPEC-011 FR-9.2). The adapter provides observability through structured JSON logs to stdout and by populating the SolverResponse fields that the Worker uses for span attributes.

**Required adapter structured log events:**

| Event Name | When Emitted | Required Fields |
|---|---|---|
| `adapter.request.received` | HTTP POST received and deserialized | `job_id`, `decision_id`, `backend_id`, `contract_version`, `stop_count`, `vehicle_count`, `execution_timeout_ms` |
| `adapter.solver.invoked` | Python solver function called (after routing and contract version check) | `job_id`, `decision_id`, `backend_id` |
| `adapter.solver.completed` | Python solver returned normally (Succeeded or natural termination) | `job_id`, `decision_id`, `backend_id`, `outcome`, `execution_duration_ms` |
| `adapter.solver.timeout` | Adapter self-terminated the Python solver | `job_id`, `decision_id`, `backend_id`, `execution_timeout_ms`, `elapsed_ms_at_termination` |
| `adapter.client.disconnect` | Worker aborted HTTP connection (in-execution cancellation) | `job_id`, `decision_id`, `backend_id` |
| `adapter.error` | Any adapter-level error (import failure, unrecognized backend_id, malformed request, unhandled exception) | `job_id` (if parseable), `decision_id` (if parseable), `error_type`, `error_message` |

All log events are emitted as JSON to stdout (ADR-006).

**Log safety:** Structured log events must not include routing problem raw data: geographic coordinate arrays, stop identifier lists, demand arrays, or time window arrays. The following fields are safe to log: `job_id`, `decision_id`, `backend_id`, `outcome`, `execution_duration_ms`, `stop_count`, `vehicle_count`, `contract_version`, `execution_timeout_ms`, `elapsed_ms_at_termination`, `error_type`, `error_message`.

**Request correlation:** `job_id` and `decision_id` from the SolverRequest are included in every log event where the request was successfully deserialized. This enables correlation between the Worker's `solver.execute` span (which carries `job_id` and `decision_id` as span attributes per SPEC-004 FR-15) and adapter log events, without the adapter emitting spans.

**Adapter contribution to `solver.execute` span:** The `solver.execute` span is populated by the Worker from the SolverResponse. The adapter's contribution is through the SolverResponse fields: `outcome` becomes the span's `outcome` attribute; `statistics.execution_duration_ms` becomes the span's `execution_duration_ms` attribute. All other span attributes (`backend_id`, `job_id`, `decision_id`, `problem_size_class`, `stop_count`) are available to the Worker from the SolverRequest and workload features.

**Acceptance Criteria:**
- All six log event types are emitted as JSON to stdout under their respective trigger conditions
- `job_id` and `decision_id` are present in every log event where the SolverRequest was successfully deserialized
- No log event contains geographic coordinate arrays, stop lists, or demand arrays
- The adapter does not emit OpenTelemetry spans, metrics, or traces

---

### FR-14: Health Checking

**Description:**
The adapter exposes a health endpoint for Docker Compose readiness and liveness management.

**`GET /health` semantics:**
- HTTP 200 with body `{"status": "ok"}`: adapter is fully operational and ready to accept solver requests. All required Python modules have been successfully imported and the HTTP service is ready.
- HTTP 503: adapter is starting up, initializing modules, or in a degraded state.

**Startup readiness gate:** The adapter must not return HTTP 200 from `/health` until:
1. All required Python modules have been successfully imported.
2. All backend-specific initialization (e.g., circuit template compilation, model loading) that occurs at startup is complete.
3. The HTTP server is ready to accept requests on port 8080.

**Docker Compose integration:** The `python-adapter` Docker Compose service configures a healthcheck using `GET /health`. The Worker container declares a dependency on `python-adapter` service health before the Worker begins processing jobs. The specific Docker Compose `depends_on` condition (`service_healthy`) and healthcheck interval configuration are deployment concerns.

**Acceptance Criteria:**
- `GET /health` returns HTTP 200 when all required modules are importable and the service is ready
- `GET /health` returns HTTP 503 during module initialization or startup
- The adapter does not return HTTP 200 from `/health` before completing all startup initialization

---

### FR-15: Security

**Description:**
Security posture for the Python Solver Adapter at MVP scope, consistent with ADR-005.

**Network isolation:** The adapter listens exclusively on the Docker Compose internal network. It is not exposed on the host network. No host port binding is configured for the adapter at MVP scope.

**No authentication or TLS at MVP scope:** Docker Compose internal networking is the accepted security boundary per ADR-005. Non-MVP deployments must add mTLS and authentication between Worker and adapter.

**Input validation:** The adapter validates that the SolverRequest body is well-formed JSON and contains all required fields before passing any data to a Python solver. The adapter does not re-validate routing problem domain constraints; the routing problem has been validated by Core (ADR-009) before the Worker dispatched the request.

**No external network access:** Python solver backends must not make external network calls during solver execution. IBM Quantum hardware execution is deferred per ADR-007. Unauthorized external calls from within the adapter could introduce timing non-determinism, network latency into execution budgets, and data exfiltration risk.

**Entropy isolation:** Python backends must not use OS random sources in reproducibility-critical paths. Incorrect entropy use produces non-reproducible outputs that undermine evidence integrity and violate ADR-010.

**Log safety:** Structured log events must not include geographic coordinate arrays, full stop lists, or demand arrays per SPEC-001 Security Considerations.

**Acceptance Criteria:**
- The adapter is not reachable from outside the Docker Compose internal network at MVP scope
- The adapter validates SolverRequest JSON structure before passing data to any Python solver
- Python backends make no external network calls during solver execution
- Log events contain no geographic coordinate arrays, stop lists, or demand arrays

---

# Non-Requirements

- SPEC-017 does not define any Python solver algorithm. QAOA circuit construction, Qiskit optimization passes, variational parameter optimization, and other Python solver algorithms are individual Python backend specification concerns (SPEC-018+).
- SPEC-017 does not define gRPC as a transport. JSON over HTTP is the decided transport, resolving ADR-005 OQ-1.
- SPEC-017 does not define stdin/stdout IPC.
- SPEC-017 does not define horizontal scaling of the adapter. One container instance at MVP scope.
- SPEC-017 does not define adapter authentication or TLS. Deferred beyond MVP per ADR-005.
- SPEC-017 does not define IBM Quantum hardware execution. Deferred per ADR-007.
- SPEC-017 does not define the backend spawn key for any specific Python backend. Each Python backend specification (SPEC-018+) declares its own unique positive integer spawn key per FR-9.
- SPEC-017 does not define Qiskit-specific seed derivation details for any specific backend. Individual backend specifications satisfy the obligation established in FR-9.
- SPEC-017 does not define a general Python plugin framework, CLI extension mechanism, user scripting environment, or notebook execution system. The adapter exists solely for solver execution.
- SPEC-017 does not define adapter connection pooling, keep-alive configuration, or HTTP request pipelining.
- SPEC-017 does not define the capability profile registration mechanism for Python backends. Python backends register capability profiles through the same mechanism as C++ backends (SPEC-003 OQ-2).
- SPEC-017 does not define Python adapter retry behavior. The Worker's NACK-with-requeue policy (SPEC-005 FR-13) handles retry at the job level.
- SPEC-017 does not define SPEC-018 (QAOA backend) or any subsequent individual Python backend specification. Those are separate child specifications of SPEC-017.

---

# Assumptions

1. The Python adapter runs as a separate Docker Compose container service named `python-adapter`, consistent with ADR-005 and the architecture.md container topology.
2. The Worker can reach the adapter at `http://python-adapter:8080` within the Docker Compose internal network.
3. NumPy is available as a pip-installable dependency and provides `numpy.random.PCG64` and `numpy.random.SeedSequence`. The SeedSequence-based initialization defined in FR-9 provides per-backend stream isolation equivalent in purpose to the C++ PCG64 stream constant model in ADR-010.
4. MVP routing problem payloads (maximum approximately 100 stops with coordinate arrays) are within acceptable JSON serialization overhead for HTTP. Serialization latency must be measured and reported (see Performance Considerations).
5. Python solver backends at MVP scope are CPU-bound (not GPU-bound or network-bound). IBM Quantum hardware execution is deferred (ADR-007); all solver computation is local.
6. The Worker has a non-zero `transport_overhead_buffer_ms` configuration value before HTTP client timeout (OQ-1). The adapter is expected to self-terminate before this buffer is consumed.
7. The adapter starts and remains running for the duration of the Worker's operation. The Worker does not manage the adapter process lifecycle. Adapter startup failures are infrastructure failures handled by Docker Compose restart policies.
8. All Python backends hosted by the adapter conform to SPEC-011 framework requirements as adapted for Python in SPEC-017. No Python backend is registered in the adapter without an Accepted individual Python backend specification.

---

# Constraints

1. Python is confined to the adapter role. The adapter is not the runtime, scheduler, Core, or evidence layer (ADR-005).
2. The adapter does not emit OpenTelemetry spans (SPEC-011 FR-9.2). The Worker is the exclusive span emitter for all solver invocations including Python-backed invocations.
3. `execution_seed` is derived by the Worker from `RoutingProblem.seed` (SPEC-005 FR-7, ADR-010 Decision 4, ODR-4). The adapter receives and forwards `execution_seed` unchanged; it does not re-derive it.
4. All Python backends must use NumPy PCG64 seeded from `execution_seed` for reproducibility-critical stochastic operations (FR-9). `random.random()`, `os.urandom()`, `time.time()`, and all OS and hardware entropy sources are prohibited in reproducibility-critical paths.
5. The PRNG is seeded exactly once per solver execution. Reseeding during a solver execution is prohibited.
6. The adapter returns HTTP 200 for every SolverResponse payload. HTTP non-200 indicates adapter-level failure.
7. The adapter must self-terminate active Python solver computation before `execution_timeout_ms` expires (FR-5). Relying exclusively on the Worker HTTP client timeout is a behavioral violation.
8. The adapter is stateless between requests (FR-7). PRNG state, partial solutions, and computation state from one request must not persist into a subsequent request.
9. Python backend execution must not access external networks (IBM Quantum hardware deferred per ADR-007).
10. `routing_problem.seed` must not be used as a PRNG entropy source. Only `execution_seed` seeds the PRNG (SPEC-004 FR-11.2).
11. `extension_metadata` must not contain routing problem raw data: geographic coordinate arrays, stop identifier lists, demand arrays (SPEC-004 FR-13).
12. `Infeasible` must not be returned by heuristic Python backends (SPEC-011 FR-5.2).
13. The adapter does not write to stderr for diagnostic output. Diagnostic information is returned through `SolverFailureDetail.failure_message` and `extension_metadata`, and through structured log events to stdout.
14. Python backend implementations hosted by the adapter must not write to stdout or stderr. `stdout` is exclusively reserved for the adapter's structured JSON log events (FR-13). Diagnostic information from Python backends must be communicated through `SolverFailureDetail.failure_message` and `extension_metadata` in the SolverResponse, not through stdout or stderr output.

---

# Inputs

| Input | Source | Format | Notes |
|---|---|---|---|
| SolverRequest | Worker (HTTP POST body) | JSON per FR-3 | All SPEC-004 FR-2 fields; `execution_seed` as decimal string |
| Health probe | Docker Compose / Worker | HTTP GET `/health` | Readiness and liveness check |

---

# Outputs

| Output | Consumer | Format | Notes |
|---|---|---|---|
| SolverResponse | Worker (HTTP 200 body) | JSON per FR-4 | Always produced on HTTP 200; Worker validates per SPEC-004 FR-11 post-receipt structural validation |
| Structured log events | Stdout (Docker Compose log collection) | JSON lines | Correlated by `job_id`, `decision_id`; no raw problem data |
| Health status | Docker Compose, Worker dependency check | HTTP 200 / HTTP 503 | Per FR-14 |

---

# Failure Modes

### Adapter Unavailable (Connection Refused)

**Condition:** Worker's HTTP client cannot connect to `http://python-adapter:8080/v1/solve` — adapter container not running or not yet healthy.
**Behavior:** Worker treats as transient infrastructure failure per SPEC-005 FR-5. Worker NACKs the RabbitMQ message with requeue=true.
**Fallback:** Redelivery when the adapter recovers. No job failure record until retry limit is exhausted.

---

### Request Deserialization Failure

**Condition:** Adapter receives an HTTP POST body that is not valid JSON, or is missing required SolverRequest fields.
**Behavior:** Adapter returns HTTP 400. Worker receives HTTP 400 and constructs a Failed SolverResponse with `failure_code = InternalError`. Job proceeds to evidence persistence. Job reaches `Completed` with a Failed solver outcome.
**Fallback:** Failed solver run recorded in evidence. Diagnostic information available from the `adapter.error` log event.

---

### Module Import Failure at Startup

**Condition:** A required Python module fails to import during adapter container startup.
**Behavior:** Adapter logs a structured error identifying the missing module. HTTP service does not start. Container exits with non-zero exit code. Docker Compose health check reports unhealthy. Worker cannot connect (Connection Refused path above).
**Fallback:** Infrastructure failure path. Manual investigation of container startup failure required.

---

### Module Import Failure at Request Time

**Condition:** A Python backend module that was not loaded at startup fails to import during request handling.
**Behavior:** Adapter returns HTTP 200 with `outcome = "Failed"`, `failure_code = "InternalError"`, `failure_message` identifying the import error. Job reaches `Completed`.
**Fallback:** Failed solver run recorded. Investigation of backend module availability required.

---

### Unrecognized `backend_id`

**Condition:** The `backend_id` in the SolverRequest does not match any registered Python backend.
**Behavior:** Adapter returns HTTP 200 with `outcome = "Failed"`, `failure_code = "InternalError"`, `failure_message` identifying the unrecognized `backend_id`. Job reaches `Completed`.
**Fallback:** Failed solver run recorded. Worker or Scheduler configuration error.

---

### Contract Version Mismatch

**Condition:** `contract_version` in the SolverRequest is not supported by the target Python backend.
**Behavior:** Adapter returns HTTP 200 with `outcome = "Failed"`, `failure_code = "ContractVersionMismatch"`, before any solver computation begins. Job reaches `Completed`.
**Fallback:** Failed solver run recorded. Worker configuration error; backend declared an incompatible `supported_contract_version`.

---

### Execution Timeout (Adapter Self-Termination)

**Condition:** Python solver computation reaches the `execution_timeout_ms` deadline.
**Behavior:** Adapter interrupts the Python solver. Adapter returns HTTP 200 with `outcome = "Timeout"`, `failure_code = "ExecutionTimeout"`. Best-so-far RoutePlan included if the backend produced one before the deadline; absent otherwise. Job reaches `Completed`.
**Fallback:** Timeout outcome recorded in evidence. Best-so-far solution evaluated by Core (SPEC-007) if present.

---

### Execution Timeout (Worker HTTP Client Timeout)

**Condition:** The adapter did not return a response within `execution_timeout_ms + transport_overhead_buffer_ms`. The Worker's HTTP client timeout fires.
**Behavior:** Worker aborts the HTTP connection. Worker constructs a Timeout SolverResponse per SPEC-004 (no solution, `execution_duration_ms` = Worker-measured elapsed time). Job reaches `Completed`.
**Fallback:** Timeout outcome recorded. No solution available. Indicates that the adapter failed to self-terminate before the deadline; requires investigation.

---

### In-Execution Cancellation (Client Disconnect)

**Condition:** Worker aborts the HTTP connection to signal cancellation (PostgreSQL `cancellation_requested` flag read during execution per SPEC-005 FR-12).
**Behavior:** Adapter detects client disconnect, interrupts Python solver, returns to ready state. Worker constructs a Cancelled SolverResponse (no solution) per SPEC-005 FR-12. Job reaches `Completed` with `outcome = Cancelled`.
**Fallback:** Cancelled outcome recorded in evidence.

---

### Python Exception During Solver Execution

**Condition:** Unhandled Python exception during solver execution.
**Behavior:** Adapter catches the exception, constructs a Failed SolverResponse with `failure_code = "InternalError"` and `failure_message` containing the exception type and message, returns HTTP 200. Job reaches `Completed`.
**Fallback:** Failed solver run recorded. Diagnostic failure message and `adapter.error` log event available for investigation.

---

### Invalid SolverResponse from Python Backend (Structural Violation)

**Condition:** Python backend produces a RoutePlan that fails SPEC-004 FR-5 structural validity requirements (wrong route count, missing stops, duplicate stops, capacity violation).
**Behavior:** The Worker detects the violation during post-receipt structural validation per SPEC-005 FR-11. Worker records a `ContractViolation`. Job reaches `Completed` with a contract violation record. Core quality evaluation is not invoked.
**Fallback:** Contract violation recorded. Investigation of the Python backend implementation required.

---

# Architectural Impact

| Component | Impact | Notes |
|---|---|---|
| Python Solver Adapter | Yes | New Docker Compose container; this specification defines its complete contract |
| Worker (SPEC-005) | Yes | New HTTP client dispatch path for Python-backed invocations; new configuration value `transport_overhead_buffer_ms`; HTTP error handling additions to FR-12 |
| Solver Contract (SPEC-004) | Yes | OQ-1 is resolved by this specification; SPEC-004 OQ-1 may be closed with reference to SPEC-017 |
| Scheduler (SPEC-003) | None | Python backends register capability profiles through the same mechanism as C++ backends (SPEC-003 OQ-2) |
| Evidence Log (SPEC-006) | None | `extension_metadata` from Python backends passes through the existing JSONB schema |
| Core Quality Evaluation (SPEC-007) | None | Core evaluates quality from the normalized RoutePlan; no changes required |
| Observability (ADR-006) | Yes | Python adapter structured logs to stdout; log correlation model via `job_id` and `decision_id` |
| ADR-005 | Yes | OQ-1 resolved: JSON over HTTP. ADR-005 should be advanced to Accepted status with reference to SPEC-017 |
| ADR-010 | Yes | Python-side reproducibility obligation established; NumPy PCG64 policy defined as Python-language equivalent of the C++ PCG64 policy |
| Deployment | Yes | `python-adapter` Docker Compose service with health checking; Worker `depends_on` adapter health |
| API Layer | None | No new API surface |
| Persistence (SPEC-012) | None | No schema changes |
| SPEC-011 FR-11 | Yes | The `Python Adapter` entry in the deferred backends table cites "ADR-005 OQ-1 (transport protocol) unresolved." This is now resolved. SPEC-011 FR-11 should be updated to note that SPEC-017 resolves ADR-005 OQ-1 and individual Python backend specifications may now be written. |

**ADR-005 OQ-1 Resolution:** This specification resolves ADR-005 OQ-1 by selecting JSON over HTTP as the MVP transport with the rationale documented in FR-2. ADR-005 should be advanced to Accepted status referencing SPEC-017.

**SPEC-004 OQ-1 Resolution:** SPEC-017 provides the Python adapter transport mapping. SPEC-004 OQ-1 may be closed once SPEC-017 is Accepted.

---

# Testability

These are test contracts; specific implementations are determined during implementation planning.

1. **Integration: Adapter health check** — `GET /health` returns HTTP 200 with `{"status": "ok"}` when the adapter is running and all required modules are importable.

2. **Integration: SolverRequest roundtrip** — A valid SolverRequest JSON POST to `/v1/solve` returns HTTP 200 with a SolverResponse JSON body. `contract_version` in the response equals `contract_version` in the request. `outcome` is one of the five defined string values. `execution_duration_ms` is present.

3. **Integration: ContractVersionMismatch** — A SolverRequest with a `contract_version` not supported by the target backend returns HTTP 200 with `outcome = "Failed"` and `failure_code = "ContractVersionMismatch"`. `execution_duration_ms` is near zero (no solver computation occurred).

4. **Integration: Unrecognized backend_id** — A SolverRequest with a `backend_id` not registered in the adapter returns HTTP 200 with `outcome = "Failed"` and `failure_code = "InternalError"`. `failure_message` names the unrecognized `backend_id`.

5. **Integration: Malformed request body — HTTP 400** — A POST to `/v1/solve` with a non-JSON body returns HTTP 400. A POST with a JSON body missing `execution_seed` returns HTTP 400.

6. **Integration: Timeout self-termination** — A SolverRequest with `execution_timeout_ms` set shorter than required for solver completion causes the adapter to return HTTP 200 with `outcome = "Timeout"` within a bounded time after the deadline. The response does not arrive after the Worker's HTTP client timeout would fire.

7. **Integration: Timeout with best-so-far solution** — For a Python backend implementing anytime behavior, a Timeout response includes a complete RoutePlan satisfying SPEC-004 FR-5 conditions 1–5 when the backend produced at least one valid solution before the deadline.

8. **Integration: Client disconnect handling** — The Worker (or test client) aborts the HTTP connection mid-request. The adapter handles the disconnect without crashing. A subsequent valid SolverRequest to the same adapter instance succeeds normally.

9. **Integration: Reproducibility** — Two identical SolverRequests (same routing problem, same `execution_seed`) submitted sequentially to the same adapter instance produce identical `solution` fields in both responses when both produce `outcome = "Succeeded"`.

10. **Integration: PRNG entropy isolation** — Two SolverRequests with identical routing problems but different `execution_seed` values produce different `solution` fields. (This validates that `execution_seed` is actually consumed by the NumPy PCG64 generator, not ignored.)

11. **Integration: Statelessness** — A timed-out or disconnected request does not affect the outcome of a subsequent request. Given request A (timed out) and request B (identical to A but succeeds), request B produces the same response as if request A had not occurred.

12. **Integration: extension_metadata safety** — No extension_metadata value in any SolverResponse contains geographic coordinate arrays, stop identifier lists, or demand arrays.

13. **Integration: Log event emission — correlation fields** — `adapter.request.received`, `adapter.solver.invoked`, and `adapter.solver.completed` events are emitted for every successful request. `job_id` and `decision_id` are present in each. No event contains geographic coordinates.

14. **Integration: Startup module import validation** — If a required module is made unavailable (e.g., removed from the environment), the adapter exits with a non-zero code and `GET /health` returns HTTP 503 or connection refused.

15. **Integration: solver.execute span attributes** — A `solver.execute` OTel span emitted by the Worker for a Python-backed invocation contains all required SPEC-004 FR-15 attributes. `outcome` reflects the SolverResponse `outcome`. `execution_duration_ms` reflects the SolverResponse `statistics.execution_duration_ms`. Span status is `Unset` for Timeout-with-solution, `OK` for Succeeded, `Error` for Failed.

16. **Property: Structural validity** — For all Python backend SolverResponses with a `solution`, the Worker's SPEC-004 FR-5 structural validation (routes.size() = vehicle_count; all stops assigned exactly once; capacity validity; no duplicate stops) passes without violations.

---

# Observability Requirements

Operational questions this adapter contract must enable:

1. For a given job (by `job_id`), did the adapter receive the request, invoke the solver, and what was the outcome?
2. How long did the Python solver execution take, disaggregated by `backend_id`?
3. How often does the adapter self-terminate due to timeout vs. completing naturally?
4. How often does the adapter return Failed, and for what reasons (import failure, unrecognized backend, Python exception)?
5. Is the adapter healthy and ready to accept requests?
6. Was an in-execution cancellation (client disconnect) detected?
7. Is the transport serialization overhead within the expected range?

These are answered by:
- The `solver.execute` span emitted by the Worker (`outcome`, `execution_duration_ms`, `backend_id`, `job_id`, `decision_id`)
- The adapter's structured log events (FR-13) correlated by `job_id` and `decision_id`
- Docker Compose health check status via `GET /health`
- The difference between the Worker's `job.consume` child span timing for `solver.execute` and the `statistics.execution_duration_ms` in the SolverResponse (measures serialization and transport overhead)

---

# Security Considerations

**Process isolation:** The Python adapter runs in a separate container. A Python solver crash, unhandled exception, or memory error cannot crash the Worker process.

**No authentication at MVP scope:** The adapter accepts requests from any client on the Docker Compose internal network. This is the accepted risk per ADR-005. Production deployments must add mTLS between the Worker and adapter, and restrict adapter network access to the Worker container only.

**Input validation boundary:** The adapter validates JSON structure and required fields before passing data to Python solvers. Routing problem domain validation (coordinates in range, demands non-negative) has been performed by Core (ADR-009) before the Worker dispatched the request; the adapter does not duplicate this validation.

**Entropy isolation:** Python backends must not use OS random sources in reproducibility-critical paths. Incorrect entropy use causes non-reproducible outputs that undermine evidence integrity and violate ADR-010. This must be enforced through code review of individual Python backend implementations.

**No external network access:** Python solvers must not make external network calls during execution. This must be enforced through Docker Compose network policy (no outbound internet access from the `python-adapter` container at MVP scope) and code review.

**Log safety:** Log events must not include geographic coordinate arrays, stop lists, or demand arrays per SPEC-001 Security Considerations.

**`execution_seed` confidentiality:** `execution_seed` must not appear in adapter log events. Its exposure enables deterministic reproduction of solver outputs. At MVP scope with synthetic routing problems this is accepted risk; it is included in the security model for non-MVP deployments.

---

# Performance Considerations

**JSON serialization overhead:** Serialization of the SolverRequest and deserialization of the SolverResponse add latency relative to in-process C++ invocation. For maximum-size MVP problems (approximately 100 stops), this overhead must be measured and reported. The measurement methodology: compute the difference between the Worker-measured `solver.execute` span duration and the adapter-reported `execution_duration_ms`. This difference is the end-to-end transport overhead (serialization + network + deserialization).

**`transport_overhead_buffer_ms` calibration:** The Worker HTTP client timeout buffer (OQ-1) must be empirically calibrated. If the buffer is too small, the Worker enforces timeout before the adapter's self-termination response arrives, discarding any best-so-far solution from anytime-capable Python backends. The buffer must exceed the 99th-percentile transport overhead observed in testing.

**Python solver startup cost:** Python backends that require module-level initialization (e.g., loading a Qiskit backend, compiling circuit templates) may experience higher latency on the first request after adapter startup. If this overhead is significant, adapter startup initialization (FR-8) should pre-load these artifacts before marking the service healthy. Subsequent requests after warmup should not bear this cost.

**`execution_duration_ms` scope:** `execution_duration_ms` is measured from the moment Python solver computation begins — defined as after JSON deserialization, backend routing, contract version validation, and PRNG initialization are complete — to the moment the adapter begins constructing the SolverResponse. PRNG initialization and JSON response serialization are not included. This is the authoritative measurement boundary (see also FR-5 point 1) and is consistent with SPEC-011's obligation that `execution_duration_ms` exclude request/response serialization time. For adapter-constructed failure responses where no solver execution occurred, `execution_duration_ms` represents elapsed time from request receipt to failure detection (see FR-4).

---

# Documentation Updates Required

- **ADR-005:** *(Completed — 2026-06-20)* Advanced to Accepted status. OQ-1 resolved: JSON over HTTP is the decided MVP transport. ADR-005 Transport Contract section updated to "Transport Contract (Resolved — SPEC-017)." SPEC-017 FR-2 is the authoritative endpoint and transport definition.
- **SPEC-004 OQ-1:** Close with reference to SPEC-017. The Python adapter transport realization is defined in SPEC-017. OQ-1 may be marked Resolved once SPEC-017 is Accepted.
- **SPEC-005:** Update the non-requirements section note: "Python adapter dispatch is out of scope for this specification until ADR-005 is resolved" is now resolved by SPEC-017. FR-9 (Solver Contract Invocation) dispatch mechanism note can reference SPEC-017 for the Python adapter path. A new Worker configuration value `transport_overhead_buffer_ms` is introduced (OQ-1 in SPEC-017).
  - **SPEC-005 FR-10 (timer duration):** FR-10 must be updated to document that the Python adapter dispatch path uses `execution_timeout_ms + transport_overhead_buffer_ms` as the Worker HTTP client timeout. `execution_timeout_ms` alone remains the adapter self-termination deadline enforced by the adapter internally. The dual-timer model applies to the Python adapter dispatch path only; in-process C++ backend execution is unaffected.
  - **SPEC-005 FR-12 (cancellation behavior):** FR-12 must be updated to document differentiated cancellation behavior for the Python adapter dispatch path. For Python adapter dispatch: the Worker monitors `cancellation_requested` while awaiting the HTTP response and aborts the HTTP connection upon detection. The existing statement that the Worker does not poll `cancellation_requested` continuously during solver execution applies to in-process C++ backend execution only; the Python adapter dispatch path requires connection monitoring as the cancellation signaling mechanism.
- **SPEC-011 FR-11 (MVP Backend Inventory):** The `Python Adapter` deferred backends entry cites "ADR-005 OQ-1 (transport protocol) unresolved." Update to note that SPEC-017 resolves ADR-005 OQ-1 and individual Python backend specifications may now be written, subject to Prerequisite 6 in SPEC-011 FR-10.1 (SolverContract implementation) using the SPEC-017 HTTP transport.
- **docs/architecture.md:** Python Solver Adapter responsibilities description includes "JSON or gRPC adapter contract." Update to reflect the decision: JSON over HTTP is the MVP transport.
- **ADR-010:** *(Architectural Impact annotation only — ADR-010 Decision text does not change.)* ADR-010 Architectural Impact section should note: (1) Python backends initialize PCG64 via `numpy.random.SeedSequence(execution_seed, spawn_key=(BACKEND_SPAWN_KEY,))` per SPEC-017 FR-9, which is the Python-language realization of ADR-010 Decision 1's per-component stream isolation policy. (2) Python backend spawn keys are subject to the same freeze obligation and breaking-change procedure as C++ PCG64 stream constants under ADR-010 Decision 5: a spawn key is frozen when the backend specification is Accepted, and changing it constitutes a reproducibility-breaking change requiring the full ADR-010 Decision 5 governance process. Spawn keys and C++ stream constants belong to different initialization mechanisms and are not numerically comparable.

---

# Open Questions

### OQ-1: HTTP Client Timeout Buffer Value (`transport_overhead_buffer_ms`)

**Question:** What is the value of `transport_overhead_buffer_ms` added to `execution_timeout_ms` when the Worker sets its HTTP client timeout?

**Why it matters:** Too small a buffer causes the Worker HTTP client timeout to fire before the adapter's self-termination response arrives. This discards any best-so-far solution from Python backends implementing anytime behavior and produces a Worker-constructed Timeout response with no solution. Too large a buffer provides less protection if the adapter becomes unresponsive. The value must be empirically calibrated against JSON serialization time for maximum-size routing problem payloads and Docker Compose internal network latency.

**Owner:** Worker implementation planning. Requires measurement during implementation.

**Blocking:** Blocks Worker HTTP client implementation for the Python adapter dispatch path. Does not block SPEC-017 acceptance.

---

### OQ-2: Concurrent Request Behavior

**Question:** If a second SolverRequest arrives at the adapter while a request is in progress, should the adapter queue the second request, return HTTP 503, or return HTTP 200 with a Failed SolverResponse?

**Why it matters:** At MVP scope, the Worker processes one job at a time (SPEC-005 Assumption 6), so two simultaneous requests to the adapter would indicate an unexpected deployment scenario. The adapter's behavior in this case must be defined before deployment.

**Risk:** HTTP server implementations may queue concurrent requests rather than rejecting them immediately. A queued request may wait for the running request to complete, consuming part or all of its `execution_timeout_ms` budget before solver execution begins. The adapter measures `execution_timeout_ms` from dispatch, not from queue exit; a queued request would begin solver computation with a partially exhausted deadline and no mechanism to detect this. This behavior must be defined before implementation — whether to queue, reject with HTTP 503, or return a Failed SolverResponse.

**Owner:** Python adapter implementation planning.

**Blocking:** Must be decided before Python adapter implementation begins. Not blocking for SPEC-017 acceptance.

---

### OQ-3A: Adapter Disconnect Detection Bound

**Question:** What is the maximum time between the Worker aborting the HTTP connection and the adapter signaling the active Python solver computation to stop?

**Why it matters:** This bound is the adapter's responsibility: the time from detecting a client disconnect to issuing a stop signal to the Python backend. If the adapter is slow to detect or act on a disconnect, adapter resources are consumed by orphaned computation that the backend has not yet been asked to halt. This bound is governed by the adapter's HTTP server model and Python threading or async model; it is independent of how quickly individual Python backends respond to the stop signal.

**Owner:** SPEC-017 implementation planning. Defined during Python adapter implementation.

**Blocking:** Does not block SPEC-017 acceptance. Must be decided before Python adapter implementation begins.

---

### OQ-3B: Backend Interrupt Compliance Bound

**Question:** What is the maximum time between the adapter issuing a stop signal to a Python backend and the backend actually halting computation?

**Why it matters:** This bound is per-backend: it depends on the backend's algorithm structure, checkpoint granularity, and cooperative interrupt mechanism. A backend with coarse-grained checkpoints may take longer to halt than one with fine-grained cooperative yield points. If this bound is not defined per backend, the total adapter resource consumption on a cancelled request cannot be bounded.

**Owner:** Individual Python backend specifications (SPEC-018+). Each backend specification must declare its interrupt compliance bound and the mechanism by which it honors the adapter's stop signal.

**Blocking:** Does not block SPEC-017 acceptance. Must be addressed by each individual Python backend specification before that backend can be Accepted.

---

### OQ-4: NumPy PCG64 Stream Isolation Mechanism

**Status: Resolved — Owner Decision Review (2026-06-20)**

**Original question:** Does NumPy's `PCG64` expose a stable public mechanism equivalent to the C++ PCG64 stream constant (`inc=` parameter) for per-backend stream isolation?

**Resolution:** NumPy's `PCG64` constructor does not expose an `inc=` parameter; the stream constant (increment) is not publicly settable via NumPy's public API. The C++ stream constant mechanism is not available in NumPy.

The selected Python-language stream isolation mechanism is `numpy.random.SeedSequence` with a per-backend `spawn_key` (positive integer). SeedSequence uses a hash-based initialization to derive both the PCG64 state and increment from the combination of `(execution_seed, spawn_key)`. Different `spawn_key` values produce statistically independent PCG64 initializations by design. Spawn keys and C++ stream constants are not in the same constant space; numerical comparison between them is not applicable. Statistical independence between Python backends is guaranteed by SeedSequence's design.

**Impact on FR-9:** FR-9 has been revised to use SeedSequence with `spawn_key`. Individual Python backend specifications (SPEC-018+) declare a unique positive integer `BACKEND_SPAWN_KEY` instead of a stream constant. The spawn key carries the same freeze obligation as the C++ stream constant under ADR-010 Decision 5.

---

# Acceptance Checklist

- [x] Problem is clearly defined
- [x] Domain concept is defined
- [x] Responsibility scope and component boundaries are defined (FR-1)
- [x] Transport protocol is decided and ADR-005 OQ-1 is resolved (FR-2)
- [x] JSON wire format for SolverRequest is defined (FR-3)
- [x] JSON wire format for SolverResponse is defined (FR-4)
- [x] Timeout behavior is defined, including both self-termination and Worker HTTP client timeout (FR-5)
- [x] Cancellation behavior is defined for pre-execution and in-execution paths (FR-6)
- [x] Adapter statelessness is defined (FR-7)
- [x] Python environment management is defined (FR-8)
- [x] Reproducibility and seed propagation are defined, including NumPy PCG64 policy and Qiskit obligation (FR-9)
- [x] Contract version handling is defined (FR-10)
- [x] Backend routing is defined (FR-11)
- [x] Adapter-level error handling and HTTP status code semantics are defined (FR-12)
- [x] Observability is defined including required log events, correlation fields, and log safety (FR-13)
- [x] Health checking is defined (FR-14)
- [x] Security considerations are defined (FR-15)
- [x] Non-requirements are documented
- [x] Assumptions are explicit
- [x] Failure modes are defined for all identified conditions
- [x] Observability requirements exist
- [x] Security considerations exist
- [x] Documentation updates are identified
- [x] OQ-1, OQ-2, OQ-3A, OQ-3B, and OQ-4 acknowledged with blocking status
- [x] Engineering Review complete — all blocking findings (F-001, F-002, F-003) and non-blocking findings (NB-001 through NB-008) resolved
- [x] Owner Decision Review complete — OQ-4 resolved; SeedSequence/spawn_key selected and applied in FR-9
- [x] Architecture Review complete — AR-001 tracked; AR-002, AR-003, AR-004 applied
- [x] Acceptance Review complete — no blocking issues; Ready for Acceptance verdict (2026-06-20)
- [x] ADR-005 advanced to Accepted (2026-06-20); Transport Contract section updated to Resolved — SPEC-017
- [ ] OQ-1 (`transport_overhead_buffer_ms`) resolved — blocks Worker HTTP client implementation; not blocking acceptance
- [ ] OQ-2 (concurrent request behavior) resolved — must decide before implementation; not blocking acceptance
- [ ] OQ-3A (adapter disconnect detection bound) resolved — implementation planning; not blocking acceptance
- [ ] OQ-3B (backend interrupt compliance bound) resolved — per SPEC-018+ backend specifications; not blocking acceptance

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-15) are implemented and acceptance criteria pass
- ADR-005 is at Accepted status referencing SPEC-017 *(complete — 2026-06-20)*
- SPEC-004 OQ-1 is closed referencing SPEC-017 *(deferred follow-on — see Documentation Updates Required)*
- OQ-1 (`transport_overhead_buffer_ms`) is resolved and the Worker HTTP client timeout is configured
- At least one individual Python backend specification (SPEC-018) conforming to SPEC-017 is Accepted *(satisfied — SPEC-018 Accepted 2026-06-20)*
- The `python-adapter` Docker Compose service is operational with health checking
- The Worker successfully dispatches SolverRequests to and receives SolverResponses from the adapter across all test contracts in the Testability section
- All required adapter structured log events are emitted and correlatable by `job_id` and `decision_id`
- Engineering Review, Architecture Review, and Acceptance Review complete *(complete — 2026-06-20)*
- Specification status is updated to Verified
