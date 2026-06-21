# SPEC-019: Quantum Hardware Solver Backend

## Metadata

**Feature ID:** SPEC-019

**Title:** Quantum Hardware Solver Backend

**Status:** Draft

**Author:** Darkhorse286

**Created:** 2026-06-20

**Last Updated:** 2026-06-20

**Supersedes:** None

**Superseded By:** None

**Parent Specification:** SPEC-011 (Backend Solver Specifications)

**Adapter Specification:** SPEC-017 (Python Solver Adapter)

**Related ADRs:** ADR-005, ADR-006, ADR-007, ADR-008, ADR-009, ADR-010, ADR-011

**Related Specs:** SPEC-001, SPEC-003, SPEC-004, SPEC-005, SPEC-006, SPEC-007, SPEC-010, SPEC-011, SPEC-012, SPEC-017, SPEC-018

---

**Solver Metadata (SPEC-011 FR-3):**

| Field | Value |
|---|---|
| `backend_id` | `qaoa-hardware` |
| `display_name` | QAOA (Quantum Hardware) |
| `backend_category` | `quantum_hardware` |
| `implementation_language` | `Python` |
| `determinism_class` | Stochastic (non-reproducible) |
| `supported_contract_version` | 1 |
| `specification_version` | 1 |

---

# Problem Statement

SPEC-018 establishes a QAOA execution path using a local Aer simulator. The local simulator produces reproducible, noise-free results and operates entirely within the Docker Compose environment. It allows the Scheduler to produce evidence about QAOA's classical-quantum feedback loop, circuit evaluation overhead, and optimizer behavior without requiring cloud access. However, local simulation does not reflect the behavior of physical quantum hardware: it produces no gate fidelity errors, no measurement noise, no decoherence effects, and no calibration drift. Results from local simulation cannot support claims about hardware quantum execution.

SPEC-019 defines the hardware-backed QAOA solver: a QAOA execution path that submits circuits to external cloud quantum execution services and receives results from physical quantum processing units. This backend consumes the same QUBO problem representation as SPEC-018 and produces the same RoutePlan contract output, but the shot measurement outcomes originate from hardware rather than classical state-vector simulation. Hardware execution introduces behavioral properties absent from local simulation: queue waiting, hardware noise, calibration drift, monetary cost, provider dependencies, and non-reproducible shot outcomes.

This specification defines observable hardware execution behavior — what the backend receives, what it produces, how it behaves during queue waiting, what evidence it emits, and how it fails. It does not define IBM Runtime APIs, provider-specific SDKs, transpiler implementations, error mitigation algorithms, circuit compilation algorithms, or cloud authentication mechanisms.

---

# Business Value

- Extends the Project DAEDALUS evidence portfolio with hardware quantum execution evidence, enabling technical writing that references real QPU results rather than classical simulation
- Allows the Scheduler to produce evidence comparing hardware execution cost, latency, and quality against local simulation and classical heuristics — strengthening the thesis that quantum execution is rarely cost-effective for routing workloads at current hardware scale
- Enables the Evidence Report to surface hardware-specific diagnostic evidence: queue duration, calibration state, shot noise behavior, and hardware error rates
- Demonstrates integration of external cloud quantum execution services within a production-style runtime architecture that maintains evidence integrity, reproducibility controls for classical components, and cost reporting
- Provides a differentiated employer-signaling artifact: a system that actually executes on quantum hardware, with evidence distinguishing hardware behavior from simulation

---

# Employer Signaling

- System Design
- Distributed Systems
- Reliability Engineering
- Observability
- Cloud Engineering

SPEC-019 demonstrates the ability to integrate an external, non-deterministic, usage-billed, queue-backed execution service into a formal solver contract (SPEC-004) that was designed for local, reproducible, zero-cost backends. The non-trivial engineering challenge is defining the behavioral contract for a backend that violates two of the contract's baseline assumptions — reproducibility and zero monetary cost — without breaking the contract's structural guarantees (RoutePlan validity, outcome reporting, evidence completeness). Specifying how timeout budget accounting works across queue and execution phases, how cancellation is delivered to a provider API rather than a cooperative process, and how evidence captures the difference between a calibration-induced failure and a circuit design failure demonstrates engineering judgment about distributed system observability.

---

# Solver Classification

**Backend category:** `quantum_hardware`

**Determinism class:** Stochastic (non-reproducible)

**Infeasibility proof capability:** No

The `quantum_hardware` category is defined in SPEC-011 FR-2.1 as: "Backends executing on real quantum hardware. Non-reproducible by nature; hardware entropy precludes reproducibility invariant." This backend implements that category.

Hardware quantum execution produces shot measurement outcomes that depend on the physical quantum state of the QPU, gate fidelity errors, measurement noise, and environmental decoherence. These outcomes are not reproducible across executions even when the same circuit, the same parameters, and the same classical seed are used. The SPEC-004 FR-11 reproducibility invariant does not apply to this backend. See FR-13 for the full reproducibility expectations.

Classical pre-processing — variational parameter initialization and classical optimizer updates — remains seeded and deterministic per the PCG64 policy established in SPEC-017 FR-9. Hardware shot outcomes are not governed by the classical seed.

This backend does not detect or prove infeasibility. Conditions where no feasible bitstring is found produce `Failed` with `failure_code = InternalError`.

---

# Requirements

### FR-1: Responsibility Scope and Component Boundaries

**Description:**
SPEC-019 defines only the behavioral contract for hardware-backed QAOA execution. It explicitly does not redefine the QAOA algorithm, Python adapter behavior, local simulation behavior, or provider-specific implementation details. The following table allocates responsibilities between this specification and adjacent owners.

**SPEC-019 owns:**
- Observable hardware execution behavior: what the backend receives, what it produces, and how it behaves across queue, execution, cancellation, and failure phases
- Queue behavior and timeout accounting across queue and execution phases
- Hardware job lifecycle: submission, status monitoring, result retrieval, cancellation
- Reproducibility expectations for hardware-backed execution — what is and is not reproducible, and acceptable hardware nondeterminism
- Shot execution obligations unique to hardware: shot count reporting, hardware noise acknowledgment
- Failure behavior unique to hardware: hardware errors, calibration failures, provider outages
- Evidence requirements: what hardware diagnostic data must be captured and in which evidence fields
- Capability profile declaration for this backend
- Cost reporting obligations

**SPEC-019 does not own:**
- The QAOA algorithm: circuit construction, QUBO formulation, decoding, best-so-far tracking, optimizer structure — all inherited from SPEC-018 by reference; changes between local simulation and hardware execution are called out explicitly in SPEC-019
- Python Solver Adapter transport, JSON wire format, health checking, environment management — owned by SPEC-017
- IBM Runtime APIs, Qiskit Runtime specifics, provider SDK invocation patterns — implementation planning decisions
- Transpiler implementation, error mitigation algorithms, circuit compilation strategy — implementation planning decisions
- Cloud authentication mechanisms — implementation planning decisions
- Solver Contract schema (SolverRequest, SolverResponse, SolverOutcome) — SPEC-004
- Scheduler policy and eligibility rules — SPEC-003
- Worker execution lifecycle, external timeout enforcement, RabbitMQ acknowledgment — SPEC-005
- Evidence persistence schema — SPEC-006
- Route quality evaluation — SPEC-007

**SPEC-018 vs. SPEC-019 boundary:**

| Behavior | Owner |
|---|---|
| QUBO problem representation | SPEC-018 (reused by SPEC-019) |
| RoutePlan contract | SPEC-004 (unchanged) |
| Bitstring decoding model | SPEC-018 (reused by SPEC-019) |
| QAOA circuit structure and Qiskit implementation | SPEC-018 for local simulation; SPEC-019 inherits the same algorithm, executed on hardware |
| Best-so-far tracking logic | SPEC-018 (applies to SPEC-019 with queue-phase constraint noted in FR-10) |
| Shot outcomes on local simulator | SPEC-018 |
| Shot outcomes on hardware QPU | SPEC-019 |
| Queue waiting behavior | SPEC-019 |
| Timeout budget including queue time | SPEC-019 |
| Hardware job cancellation | SPEC-019 |
| Hardware failure modes | SPEC-019 |
| Hardware evidence fields | SPEC-019 |
| Calibration identifiers | SPEC-019 |
| Monetary cost reporting | SPEC-019 |

**Acceptance Criteria:**
- No part of SPEC-019 defines provider-specific API calls, IBM Runtime SDK methods, or authentication mechanisms
- No part of SPEC-019 redefines the QAOA algorithm or QUBO formulation (these are inherited from SPEC-018)
- No part of SPEC-019 alters the SolverRequest or SolverResponse schema (SPEC-004)

---

### FR-2: Hardware Provider Abstraction

**Description:**
This backend executes QAOA circuits on an external cloud quantum execution service. The backend treats the cloud service as a provider: a named execution target that accepts circuits, queues them, executes them on hardware, and returns measurement results. SPEC-019 does not specify which provider or providers are acceptable.

**Provider abstraction obligations:**
1. The backend accepts a configured provider connection from the deployment environment. The mechanism by which provider credentials, endpoints, and selection parameters are supplied to the backend is an implementation planning decision (OQ-1).
2. The backend interacts with the provider through a Python SDK layer. The specific SDK and SDK version are implementation planning decisions.
3. The backend must not hard-code provider API behavior. Provider API calls are isolated behind an adapter layer within the backend implementation, enabling provider substitution without contract changes.
4. If no configured provider is available at invocation time, the backend returns `Failed` with `failure_code = InternalError` and a `failure_message` identifying the configuration failure.

**Provider neutrality obligation:** The observable contract — SolverRequest input, SolverResponse output, extension_metadata keys — must not vary based on which provider is used. Provider-specific data belongs in extension_metadata values (keyed generically), not in new contract fields.

**Acceptance Criteria:**
- A valid SolverResponse conforming to SPEC-004 is produced regardless of which configured provider is used
- Extension metadata keys are provider-neutral; provider identity appears as a value, not as a key

---

### FR-3: Backend Selection and Hardware Device Declaration

**Description:**
A cloud quantum execution provider may offer multiple hardware devices (backends) with different qubit counts, connectivity topologies, error rates, and calibration states. The backend must select a target hardware device for each execution.

**Device selection obligations:**
1. The backend selects a hardware device from among the configured eligible devices before submitting the circuit. The selection criteria — qubit count compatibility with the QUBO dimension, gate fidelity thresholds, availability — are implementation planning decisions (OQ-2).
2. The selected hardware device identifier is recorded before job submission and is available for evidence reporting regardless of execution outcome.
3. The backend must not submit a circuit to a device whose qubit count is insufficient for the QUBO binary variable count Q of the routing problem. If no eligible device has sufficient qubit count, the backend returns `Failed` with `failure_code = InternalError` before job submission.
4. If no eligible device is available (all devices are offline, at capacity, or misconfigured), the backend returns `Failed` with `failure_code = InternalError`.
5. The selected device identifier and its hardware version at the time of selection are included in extension_metadata (FR-15.1) regardless of whether the execution produces a RoutePlan.

**Acceptance Criteria:**
- The selected hardware device identifier appears in extension_metadata on every response except pre-selection failures (FR-3.3, FR-3.4 conditions that preclude device selection)
- No circuit is submitted to a device with insufficient qubit count for the QUBO dimension
- Device unavailability produces `Failed` with `InternalError`, not `Timeout`

---

### FR-4: Hardware Capability Declaration

**Description:**
The capability profile for this backend reflects hardware quantum execution constraints. See FR-15 for the full capability profile table. This section states the behavioral constraints that govern the declared values.

**Size class constraint:** The backend is eligible only for routing problems whose QUBO binary variable count Q, derived from the encoding strategy (implementation planning decision per SPEC-018 FR-2), does not exceed the available hardware qubit count on any configured eligible device. This constraint maps to the `supported_size_classes` declaration (OQ-2). Until empirical QUBO dimension measurements are available for specific hardware devices, `supported_size_classes` is provisionally declared as `{Small}`, matching SPEC-018.

**Time window constraint:** This backend does not consult time window fields during optimization. It inherits the same constraint as SPEC-018: `supports_time_windows = false`.

**Capacity constraint:** This backend inherits SPEC-018's capacity enforcement: capacity constraints are encoded as QUBO penalty terms; decoded bitstrings violating capacity are ineligible as best-so-far candidates. `supports_capacity_constraints = true`.

**Provisional status:** This backend must declare `is_provisional = true` until all qualification criteria are satisfied (OQ-5). The qualification criteria include empirical latency and quality measurement, provider access confirmation, and Project Owner approval of cost thresholds.

---

### FR-5: Shot Execution on Hardware

**Description:**
This backend executes QAOA circuits for `shots_per_evaluation` measurements on a physical QPU. Shot execution on hardware differs from local simulation in the following observable ways.

**Hardware shot model:**
1. Shot measurement outcomes are drawn from the physical quantum probability distribution, subject to gate fidelity errors, measurement noise, and environmental decoherence. These errors cause the measured probability distribution to deviate from the ideal theoretical distribution.
2. Shot outcomes are not reproducible. Given the same circuit, the same parameters, and the same classical seed, two hardware executions produce different shot outcome distributions. This is expected and is not a failure.
3. The expected QUBO energy computed from hardware shots will differ across executions with identical circuit and parameters. This affects the optimizer's parameter update trajectory.
4. The feasible bitstring count (`qaoa.feasible_bitstrings_found`, FR-15) may differ across identical invocations due to hardware noise.
5. Shot reproducibility is not required, not expected, and is not a failure condition for this backend.

**Shot count obligations:**
- `shots_per_evaluation` must be a positive integer: `shots_per_evaluation ≥ 1`.
- The actual shot count submitted to hardware must equal `shots_per_evaluation`. Provider-side shot count rounding or reduction is a provider concern; if the provider returns fewer shots than submitted, the backend reports the actual shot count in extension_metadata.
- `shots_per_evaluation` is fixed before the optimization loop begins. It does not change between optimizer evaluations.
- The actual shot count is reported in extension_metadata as `qaoa.shots_per_evaluation` (FR-15).

**Calibration state:** Hardware shot outcomes depend on the calibration state of the device at execution time. The calibration identifier current at execution time is recorded in extension_metadata (FR-15). Calibration changes between two executions of identical circuits may produce different shot distributions.

**Acceptance Criteria:**
- `shots_per_evaluation ≥ 1` in every execution
- Actual shot count is reported in extension_metadata
- Hardware shot variation across identical executions is not treated as a failure
- Calibration identifier is recorded in extension_metadata

---

### FR-6: Queue Behavior

**Description:**
Cloud quantum execution services queue submitted jobs before hardware execution. Queue wait time is the elapsed time from circuit submission to the start of hardware execution. Queue wait time is variable and depends on provider load, device availability, and job priority.

**Queue model:**
1. The backend submits a hardware job after circuit transpilation. From the moment of submission, the job enters the provider's queue.
2. The backend monitors job status while the job is queued. Monitoring mechanism (polling interval, callback) is an implementation planning decision.
3. Queue wait time is counted against the `execution_timeout_ms` budget. The budget is not paused during queue waiting. See FR-8 for timeout accounting.
4. The backend records the elapsed queue wait time from job submission to execution start. This value is reported in extension_metadata as `qaoa.hardware.queue_wait_ms` (FR-15.1).
5. If the queue wait time alone exhausts the `execution_timeout_ms` budget before hardware execution begins, the backend cancels the queued job and returns `Timeout` with no solution. See FR-8.4.
6. Job priority and queue position are provider-specific concerns not defined by this specification.

**Queue status monitoring:**
- The backend must not block without monitoring. It must check job status at intervals and check whether the `execution_timeout_ms` deadline has been reached.
- If the backend detects a Worker cancellation signal (HTTP connection abort per SPEC-017 FR-6) while the job is queued, the backend cancels the queued job and returns `Cancelled` with no solution. See FR-9.

**Acceptance Criteria:**
- Queue wait time is recorded and reported in extension_metadata
- A job that exhausts its `execution_timeout_ms` budget while still queued is cancelled and produces `Timeout` with no solution
- Queue cancellation on Worker cancellation signal produces `Cancelled` with no solution
- The backend monitors the deadline during queue waiting

---

### FR-7: Hardware Job Lifecycle

**Description:**
A hardware execution job proceeds through a defined lifecycle: preparation, submission, queue waiting, execution, result retrieval, and response construction. This section defines the observable behavior at each phase boundary.

**Lifecycle phases:**

| Phase | Description | Timeout Consumed |
|---|---|---|
| 1. Circuit preparation | QUBO construction, QAOA circuit generation, transpilation (may include PCG64 draws per FR-13 PRNG Phase 1) | Yes |
| 2. Job submission | Circuit and parameters submitted to provider API | Yes |
| 3. Queue wait | Job waits in provider queue before hardware execution begins | Yes |
| 4. Hardware execution | Circuit executes on QPU; shots are measured | Yes |
| 5. Result retrieval | Measurement results fetched from provider | Yes |
| 6. Decoding and best-so-far update | Bitstrings decoded to RoutePlan candidates; best-so-far updated | Yes |
| 7. Response construction | SolverResponse serialized for adapter | Yes (small) |

All phases consume elapsed time from the `execution_timeout_ms` budget. The budget begins at the moment Python solver computation begins (per SPEC-017 FR-5: after JSON deserialization, backend routing, contract version validation, and rng initialization).

**Phase 3 result availability:** During queue waiting (Phase 3), no partial results are available. The best-so-far remains null until at least one result is retrieved from hardware. See FR-10 for best-so-far behavior.

**Phase 4 result streaming:** Some providers support streaming result retrieval during hardware execution. If the provider supports streaming, the backend may update the best-so-far after receiving partial shot results. If the provider does not support streaming and delivers results only after the full job completes, the best-so-far is updated only during Phase 6.

**Atomic execution model:** If the provider executes all optimizer evaluations as a single hardware job batch, the best-so-far can only be updated after Phase 5 completes. In this case, the anytime contract (FR-10) applies at job-batch granularity, not per-optimizer-evaluation granularity.

**Acceptance Criteria:**
- All seven lifecycle phases consume time against the `execution_timeout_ms` budget
- Phase 3 produces no partial results and does not update best-so-far
- If streaming is available, Phase 5 updates best-so-far as results arrive
- If not streaming, Phase 6 updates best-so-far after full result retrieval

---

### FR-8: Timeout Behavior and Budget Accounting

**Description:**
`execution_timeout_ms` in the SolverRequest is the total execution budget covering all lifecycle phases (FR-7). Timeout accounting differs from SPEC-018 because queue wait time can constitute a significant fraction of the total budget.

**Budget decomposition:**
The total budget covers:
- Circuit preparation time (Phase 1–2)
- Queue wait time (Phase 3)
- Hardware execution time (Phase 4)
- Result retrieval and post-processing time (Phase 5–6)

The backend does not have separate queue and execution timeouts. One budget governs all phases.

**Timeout during queue wait (Phase 3):**
1. The backend checks the `execution_timeout_ms` deadline at each status monitoring interval during queue waiting.
2. If the deadline is reached while the job is still queued, the backend submits a cancellation request to the provider API.
3. After confirming the job is cancelled (or after a bounded cancellation confirmation time per OQ-4), the backend returns HTTP 200 with `outcome = "Timeout"`, `failure_code = "ExecutionTimeout"`, and no `solution`. FR-15.2 shared keys (solution fields) are absent. Hardware evidence keys (FR-15.1) are present for each phase reached: `qaoa.hardware.provider_backend`, `qaoa.hardware.hardware_version`, `qaoa.hardware.provider_job_id`, `qaoa.hardware.transpiled_circuit_depth`, and `qaoa.hardware.transpiled_cx_count` are present (device was selected and job was submitted); `qaoa.hardware.queue_wait_ms` is present and populated with the measured queue wait up to cancellation. Hardware execution keys (`qaoa.hardware.circuit_execution_ms`, `qaoa.hardware.calibration_id`, `qaoa.hardware.actual_shot_count`, `qaoa.hardware.estimated_cost`, `qaoa.hardware.error_mitigation_applied`, `qaoa.hardware.error_mitigation_method`) are absent because hardware execution did not begin.
4. A response produced under this condition reports `execution_duration_ms` as the total elapsed time from execution start to response construction.

**Timeout during hardware execution (Phase 4–5):**
1. If the deadline is reached while hardware execution is in progress, the backend submits a cancellation request to the provider API.
2. Result retrieval may continue for a bounded time to collect any partial results already computed before the provider cancels. Whether partial results are available depends on provider behavior (OQ-4).
3. If at least one complete, structurally valid RoutePlan was decoded from partial results, it is included in the Timeout response.
4. The backend returns HTTP 200 with `outcome = "Timeout"`, `failure_code = "ExecutionTimeout"`. Solution is present if best-so-far exists; absent otherwise.

**Worker HTTP client timeout (external backstop):**
The SPEC-017 FR-5 external backstop applies identically to this backend. The Worker's HTTP client timeout fires at `execution_timeout_ms + transport_overhead_buffer_ms`. The adapter's self-termination obligation applies. For hardware execution, self-termination requires submitting a cancellation request to the provider before the HTTP client timeout fires. This is a tighter deadline than for local computation.

**Quantum-specific note:** Hardware queue wait times can be long (minutes to hours under high provider load). An `execution_timeout_ms` value that is short relative to the expected queue time will produce frequent Timeout outcomes with no solution. The capability profile `latency_profile` declaration must account for expected queue time in addition to circuit execution time.

**Acceptance Criteria:**
- Queue wait time counts against the `execution_timeout_ms` budget
- Deadline detection occurs at each status monitoring interval during queue waiting
- Deadline during queue wait produces Timeout with no solution after job cancellation
- Deadline during hardware execution produces Timeout with best-so-far if available
- The adapter self-terminates before the Worker's HTTP client timeout fires

---

### FR-9: Cancellation Behavior

**Description:**
Cancellation for hardware-backed execution requires cancelling a job that may be queued or actively executing on a remote provider. The cancellation mechanism differs from SPEC-018's cooperative Python thread interrupt.

**Pre-execution cancellation:** Unchanged from SPEC-017 FR-6. If `cancellation_requested` is set before the Worker dispatches the HTTP POST, no dispatch occurs. No adapter or provider interaction occurs.

**In-execution cancellation — queued job:**
1. The Worker aborts the HTTP connection (SPEC-017 FR-6 in-execution cancellation signal).
2. The adapter detects client disconnect and signals the backend to stop.
3. The backend submits a cancellation request to the provider API for the queued job.
4. After confirming the job is cancelled, the backend terminates.
5. The Worker constructs a Cancelled SolverResponse with no solution (SPEC-005 FR-12 Worker-constructed Cancelled).
6. Extension metadata is absent.

**In-execution cancellation — executing job:**
1. The Worker aborts the HTTP connection.
2. The backend submits a cancellation request to the provider API for the executing job.
3. If the provider supports partial result retrieval from a cancelled job, the backend may attempt to retrieve partial results before terminating.
4. The Worker constructs a Cancelled SolverResponse with no solution (SPEC-005 FR-12 Worker-constructed Cancelled). Because the HTTP connection was aborted before the adapter returned a response, no partial results decoded by the backend are accessible to the Worker.
5. The adapter returns to ready state after the cancellation completes.

**Interrupt compliance bound (SPEC-017 OQ-3B):**
The maximum time from the adapter's stop signal to the backend halting computation is bounded by the time to submit a cancellation request to the provider API plus a bounded provider acknowledgment wait (OQ-4). The backend does not wait indefinitely for provider cancellation confirmation. If the provider does not acknowledge cancellation within the configured cancellation timeout, the backend terminates its own monitoring loop and returns to ready state. The provider job may continue executing after the adapter has terminated; this is a known limitation of cloud-based execution.

**Acceptance Criteria:**
- Cancellation of a queued job submits a provider API cancellation request
- Cancellation of an executing job submits a provider API cancellation request
- The backend returns to ready state after cancellation regardless of provider confirmation timing
- Pre-execution cancellation produces no provider interaction

---

### FR-10: Best-So-Far Solution Behavior

**Description:**
The anytime contract from SPEC-018 FR-7 applies to this backend, with the constraint that best-so-far updates are only possible when shot results are available. Shot results are only available after hardware execution completes (or is streaming, per FR-7).

**Best-so-far update trigger:**
- If the provider supports per-evaluation streaming: best-so-far is updated after each optimizer evaluation that produces decoded feasible bitstrings, as in SPEC-018 FR-7.
- If the provider executes all optimizer evaluations in a single hardware batch: best-so-far is updated only after the batch completes and results are decoded. There is no per-evaluation best-so-far update during the job's execution phase.

**Queue phase:** During queue wait (Phase 3), best-so-far is null. No solution can be returned from a Timeout or Cancelled event that occurs while the job is still queued.

**Timeout with partial results:**
- If the deadline arrives after hardware execution has completed but before full result decoding, the backend completes decoding the results already retrieved.
- If partial shot results are available when the deadline is reached during hardware execution (provider-dependent), the backend decodes available bitstrings and includes the best-so-far in the Timeout response if a feasible RoutePlan was found.

**Cancellation: no partial results available:**
- When the Worker aborts the HTTP connection (FR-9), the adapter cannot return data through the closed connection. The Worker constructs a Cancelled SolverResponse with no solution (SPEC-005 FR-12). No partial results are included in a Worker-constructed Cancelled response regardless of what results the backend decoded before the connection was closed.

**Feasibility gate:** Unchanged from SPEC-018 FR-6. Decoded bitstrings are eligible as best-so-far candidates if and only if they satisfy all SPEC-004 FR-5 structural validity requirements. Hardware noise increases the likelihood of infeasible bitstrings (constraint-violating assignments) relative to noise-free simulation.

**solution_count in ExecutionStatistics:** Number of times the best-so-far solution improved during execution. For streaming providers: identical semantics to SPEC-018 FR-7 — the count of optimizer evaluations in which a new feasible best solution was decoded. For batch providers where all optimizer evaluations are submitted as a single hardware job: the count of distinct improvements to the best-so-far observed while processing the complete result set in order, scanning evaluation results sequentially and counting each step at which the decoded route distance improved. These definitions are equivalent for streaming providers; they diverge for batch providers because no per-evaluation boundary exists during job execution.

**Acceptance Criteria:**
- No solution is returned from a Timeout or Cancellation that occurred during queue waiting
- Best-so-far is updated from hardware results using the same feasibility gate as SPEC-018
- Streaming-capable provider execution supports per-evaluation best-so-far updates
- Batch-only provider execution supports best-so-far update after batch completion

---

### FR-11: Hardware Failure Behavior

**Description:**
Hardware execution introduces failure modes that do not exist for local simulation. These are in addition to the failure modes inherited from SPEC-018 (contract version mismatch, QUBO construction failure, natural termination with no feasible solution, internal error).

**Hardware-specific failure conditions:**

**Circuit incompatibility with target hardware:**
- **Trigger:** After QUBO construction, the transpiled circuit depth or qubit count exceeds the capacity of the selected hardware device.
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** Before job submission. `failure_message` identifies the circuit depth, qubit count, and device limits.
- **Metadata:** `qaoa.hardware.provider_backend` included if device selection occurred before the failure. Other hardware evidence fields absent.

**Job submission failure:**
- **Trigger:** The provider API returns an error when the job is submitted (transient network error, API error, quota exceeded).
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** The backend returns Failed after a bounded number of submission retries (OQ-4). `failure_message` identifies the provider error.
- **Hardware evidence:** `qaoa.hardware.provider_backend` is included if device selection occurred.

**Hardware execution error:**
- **Trigger:** The provider reports that the hardware job failed during execution (device error, decoherence-induced failure that the provider classifies as an error, hardware malfunction).
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** `failure_message` includes the provider's error code or classification. `qaoa.hardware.failure_classification` is populated (FR-15.1).
- **Hardware evidence:** All hardware evidence fields that are available at the time of failure are included in extension_metadata.

**Calibration-induced quality degradation (not a failure):**
- **Description:** Hardware calibration drift can cause gate error rates to increase over time between recalibration cycles. This manifests as fewer feasible bitstrings in shot results, not as a job failure. Calibration-induced quality degradation is observed in the evidence (lower `qaoa.feasible_bitstrings_found`, lower solution quality) but does not produce a `Failed` outcome.
- **Evidence:** The calibration identifier at execution time (FR-15.1) enables post-hoc correlation of result quality with calibration state.

**Natural optimizer termination with no feasible solution:**
- Inherited from SPEC-018 FR-14. Behavior unchanged. On hardware, this condition is more likely due to shot noise interfering with the optimizer's energy landscape.
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Metadata:** `failure_message` must include "qaoa-hardware: optimization completed without producing a feasible route plan" plus problem characteristics and hardware evidence available at the time.

**Acceptance Criteria:**
- Circuit incompatibility produces `Failed` with diagnostic failure_message before job submission
- Job submission failure produces `Failed` after bounded retries
- Hardware execution error produces `Failed` with `qaoa.hardware.failure_classification` in extension_metadata
- Hardware calibration drift does not produce `Failed`; it is observable through evidence fields

---

### FR-12: Provider Outage Behavior

**Description:**
A provider outage occurs when the cloud quantum execution service is unavailable: the provider API is unreachable, returning persistent errors, or reporting no available hardware devices.

**Provider outage at job submission:**
- **Detection:** The provider API returns connection errors or service unavailability responses for all submission attempts within the bounded retry window (OQ-4).
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** The backend returns Failed. `failure_message` identifies the outage as the cause. The Worker receives HTTP 200 with a Failed SolverResponse (not a connection-refused error) because the adapter itself is running.
- **Worker handling:** The Worker processes this as a solver-level failure per SPEC-017 FR-12. The job reaches `Completed` with a Failed solver outcome. Unlike adapter unavailability (connection refused, which triggers NACK-with-requeue), a provider outage produces a completed failed solver run that is recorded in evidence.

**Provider outage during queue wait:**
- **Detection:** Provider API status polling returns persistent errors indicating the service is unavailable.
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** The backend terminates queue monitoring and returns Failed. `failure_message` identifies the outage. Hardware evidence fields available before the outage are included.

**Provider outage during hardware execution:**
- **Detection:** Provider API becomes unreachable while the job is executing.
- **Outcome:** `Timeout`, `failure_code = "ExecutionTimeout"`, `qaoa.hardware.failure_classification = "provider_outage"` (FR-15.1)
- **Behavior:** Result retrieval may have produced partial results. If any complete RoutePlan was decoded before the outage, it is included in the Timeout response. Given the investment of hardware execution time, this situation is classified as `Timeout` rather than `Failed`, allowing the best-so-far to be returned if available.
- **Note:** This is the only case where `Timeout` is returned due to an infrastructure failure rather than a time budget event. The `qaoa.hardware.failure_classification` field in extension_metadata (FR-15.1) is the distinguishing marker that separates this infrastructure-failure Timeout from a time-budget Timeout. This usage is authorized by SPEC-004 FR-10 (quantum hardware anytime exception).

**Outage classification in evidence:** Provider outage during execution produces `Timeout` with `qaoa.hardware.failure_classification = "provider_outage"` (FR-15.1). This makes provider-outage Timeouts distinguishable from time-budget Timeouts in evidence and supports post-incident analysis.

**Acceptance Criteria:**
- Provider outage at submission or queue wait produces `Failed` with `InternalError` (not connection-refused to the Worker)
- Provider outage during execution produces `Timeout` with best-so-far if available, `Timeout` with no solution otherwise
- `qaoa.hardware.failure_classification = "provider_outage"` is present in provider-outage Timeout responses, distinguishing them from time-budget Timeouts in evidence

---

### FR-13: Reproducibility Expectations

**Description:**
SPEC-011 FR-2.1 explicitly classifies `quantum_hardware` backends as "Stochastic (non-reproducible)." The SPEC-004 FR-11 reproducibility invariant does not apply to this backend. This section defines what IS expected, what IS NOT expected, and what constitutes acceptable hardware nondeterminism.

**What is reproducible:**
- Classical pre-processing: variational parameter initialization (PRNG Phase 2 draws) and Qiskit sub-system seeds (PRNG Phase 1 draws) are reproducible given the same `execution_seed` and the same frozen `BACKEND_SPAWN_KEY`. Two invocations with the same `execution_seed` begin the optimization loop with identical initial parameters.
- QUBO construction: the QUBO representation of the routing problem is deterministic given the same routing problem fields.
- Classical optimizer updates: for deterministic optimizers, parameter update trajectories are reproducible given the same initial parameters (which are reproducible from the seed). For stochastic optimizers, the `optimizer_seed` derived in Phase 1 governs their behavior.

**What is NOT reproducible:**
- Hardware shot outcomes: given the same circuit, the same parameters, and the same classical seed, two hardware executions produce different shot outcome distributions.
- Best-so-far solution: because shot outcomes differ, the feasible bitstrings found and the best-so-far RoutePlan may differ across identical invocations.
- `qaoa.best_bitstring_energy`: hardware noise causes the QUBO energy of observed bitstrings to vary.
- `qaoa.feasible_bitstrings_found`: the count of bitstrings decoding to feasible RoutePlans varies across executions.

**Acceptable hardware nondeterminism:**
- Shot outcome variation across invocations is expected and not a failure.
- Variation in feasible solution count and best-so-far quality between invocations with identical inputs is expected.
- The returned RoutePlan may differ across identical invocations. This does not constitute a reproducibility failure.
- Calibration drift: the device calibration state changes between recalibration cycles. Executions separated by a recalibration event may produce systematically different shot distributions. The calibration identifier in evidence (FR-15.1) enables correlation of results with calibration state.
- Shot variance expectations: shot outcome variance is governed by quantum mechanical probability and hardware noise models, not by the classical PRNG. No deterministic bound on per-execution shot outcome variance is defined.

**Reproducibility guarantee scope:**
- The same `execution_seed` guarantees identical initial QAOA parameters across invocations.
- The same `execution_seed` does not guarantee identical shot outcomes, identical best-so-far solutions, or identical SolverResponse content.
- Evidence records for this backend are not directly comparable across invocations in the same way as local simulation evidence. Comparison requires acknowledging hardware shot variance.

**Cross-hardware reproducibility:** Not required and not expected. Results from one hardware device are not expected to match results from a different hardware device, even with the same classical seed.

**BACKEND_SPAWN_KEY:** A unique positive integer, distinct from `20260620` (SPEC-018). Must be assigned and declared in this specification before it transitions to Accepted. Governs PCG64 initialization for classical pre-processing phases only. Changing the spawn key after Acceptance requires the full ADR-010 Decision 5 breaking change procedure.

**PRNG draw ordering (classical phases only):**

PRNG Phase 1 (Qiskit sub-system seed derivation): Same draw sequence as SPEC-018 FR-10 Phase 1. Three draws from the PCG64 rng, in this order:
1. `transpiler_seed`: `int(rng.integers(0, 2**31))`
2. `simulator_seed`: Not applicable (no simulator). This draw is still consumed at position 2 to maintain draw-ordering stability consistent with SPEC-018. The drawn value is generated but not used.
3. `optimizer_seed`: `int(rng.integers(0, 2**31))` — consumed if the optimizer is stochastic; generated but not used if deterministic.

PRNG Phase 2 (QAOA parameter initialization): Same draw sequence as SPEC-018 FR-10 Phase 2. 2p draws in interleaved order: γ[0], β[0], γ[1], β[1], ..., γ[p-1], β[p-1].

PRNG Phase 3 (Hardware execution): Randomness during hardware execution originates from the QPU and is not governed by the classical rng. The transpiler seed (PRNG Phase 1 Draw 1) governs classical transpilation only.

**Reproducibility invariant status:** The SPEC-004 FR-11.1 reproducibility invariant — identical inputs produce identical solutions — does not apply to this backend. This exemption is established by SPEC-011 FR-2.1. Evidence records must not assert reproducibility of hardware-sourced results.

**Acceptance Criteria:**
- Evidence records do not assert or imply reproducibility of hardware shot outcomes
- Classical pre-processing (parameter initialization) is deterministic given the same `execution_seed` and `BACKEND_SPAWN_KEY`
- Hardware shot variation across identical invocations is not treated as a failure
- The calibration identifier is recorded in evidence to support post-hoc result correlation

---

### FR-14: Evidence Requirements

**Description:**
Hardware execution generates evidence that is not produced by local simulation. SPEC-019 requires comprehensive hardware evidence in extension_metadata to support future technical publications and post-hoc analysis.

**Evidence categories:**

**Queue and execution timing:**
- Queue duration: elapsed time from job submission to hardware execution start
- Execution duration: elapsed time for hardware circuit execution only (excluding queue wait and result retrieval)
- Total `execution_duration_ms` in ExecutionStatistics: total solver wall-clock time per SPEC-017 FR-5

**Hardware identity:**
- Selected backend (device) identifier: the provider's name for the hardware device
- Hardware version: the device's firmware or version string at the time of execution, if available from the provider
- Calibration identifier: the calibration run identifier current at execution time, if provided by the provider

**Execution parameters:**
- Actual shot count: shots measured by hardware (may differ from submitted shot count on some providers)
- Ansatz depth p: same as SPEC-018
- Optimizer evaluations: count of classical optimizer function evaluations completed (same as SPEC-018)

**Transpilation metrics:**
- Transpiled circuit depth: gate depth of the transpiled circuit submitted to hardware
- Transpiled two-qubit gate count: count of two-qubit gates (typically CX/CNOT) in the transpiled circuit; primary indicator of circuit fidelity burden on hardware

**Provider metadata:**
- Provider job identifier: the identifier assigned to the hardware job by the provider; enables post-execution investigation via the provider's own tooling

**Error mitigation:**
- Error mitigation applied: whether any error mitigation technique was applied to the raw measurement results (boolean, reported as string "true" or "false")
- Error mitigation method: name of the mitigation method applied, if any (e.g., "readout_mitigation", "zne"); provider-agnostic identifier

**Failure classification:**
- Failure classification: a structured string identifying the failure class: one of `"circuit_incompatible"`, `"submission_error"`, `"hardware_error"`, `"calibration_degraded"`, `"provider_outage"`, `"no_feasible_solution"`, `"internal_error"`. Present on all Failed responses. Also present on Timeout responses when the Timeout was caused by a provider outage during hardware execution (FR-12), with value `"provider_outage"`.

**Evidence purpose:** These fields support the evidence thesis of Project DAEDALUS. A technical publication based on hardware results must disclose the hardware device, calibration state, shot count, and transpilation metrics to allow readers to assess the evidence's validity. An evidence record lacking these fields cannot support a credible hardware execution claim.

**Acceptance Criteria:**
- All evidence fields listed in FR-15 are populated in every response where the relevant phase was reached
- Failure classification is present in every Failed response
- Provider job identifier is present in every response where a job was submitted
- Evidence fields contain no routing problem raw data

---

### FR-15: Extension Metadata

**Description:**
The following extension_metadata keys are defined for this backend. Keys prefixed `qaoa.` are shared with SPEC-018 and carry the same semantics. Keys prefixed `qaoa.hardware.` are specific to this backend.

**FR-15.1: Hardware-specific extension metadata:**

| Key | Type (as string) | Present When | Description |
|---|---|---|---|
| `qaoa.hardware.queue_wait_ms` | uint64 | Job submitted | Elapsed time in milliseconds from hardware job submission to hardware execution start, as measured by the backend. Zero if the job executed immediately without queuing. |
| `qaoa.hardware.circuit_execution_ms` | uint64 | Job executed | Elapsed time in milliseconds for hardware circuit execution only, excluding queue wait time and result retrieval time. Provider-reported if available; adapter-measured otherwise. |
| `qaoa.hardware.provider_backend` | string | Device selected | The provider's identifier for the selected hardware device. Example: `"ibm_brisbane"` or equivalent. Provider-neutral description; actual value is provider-assigned. |
| `qaoa.hardware.hardware_version` | string | Device selected | Hardware device version or firmware version string as provided by the provider. `"unknown"` if not available from the provider. |
| `qaoa.hardware.calibration_id` | string | Job executed | The calibration run identifier current at hardware execution time, as reported by the provider. `"unknown"` if the provider does not expose calibration identifiers. |
| `qaoa.hardware.provider_job_id` | string | Job submitted | The job identifier assigned by the provider. Enables investigation via provider tooling. |
| `qaoa.hardware.transpiled_circuit_depth` | uint32 | Job submitted | Gate depth of the transpiled circuit as submitted to hardware. Includes all gate layers including barriers if counted by the transpiler. |
| `qaoa.hardware.transpiled_cx_count` | uint32 | Job submitted | Count of two-qubit gate operations (CX, CNOT, or provider-equivalent) in the transpiled circuit. Primary indicator of hardware fidelity burden. |
| `qaoa.hardware.error_mitigation_applied` | string | Job executed | `"true"` if any error mitigation technique was applied to raw results; `"false"` otherwise. |
| `qaoa.hardware.error_mitigation_method` | string | `error_mitigation_applied = "true"` | Name of the error mitigation method applied. Absent if no mitigation applied. |
| `qaoa.hardware.actual_shot_count` | uint32 | Job executed | Number of shot measurement results actually returned by the hardware provider. May differ from `shots_per_evaluation` if the provider truncated or modified the shot count. |
| `qaoa.hardware.estimated_cost` | string | Job executed | Estimated monetary cost of this hardware job execution in units defined by the provider's pricing model. Encoded as a decimal string. Example: `"0.00023"`. See FR-17 for cost reporting obligations. `"unknown"` if cost cannot be determined from the provider. |
| `qaoa.hardware.failure_classification` | string | `outcome = "Failed"`; or `outcome = "Timeout"` caused by provider outage during execution (FR-12) | Structured failure class string. One of: `"circuit_incompatible"`, `"submission_error"`, `"hardware_error"`, `"calibration_degraded"`, `"provider_outage"`, `"no_feasible_solution"`, `"internal_error"`. On provider-outage Timeout responses, value is `"provider_outage"`. |

**FR-15.2: Shared extension metadata (inherited from SPEC-018):**

The following keys carry identical semantics to SPEC-018 FR-13. All are present when a solution is present; all are absent when no solution is present.

| Key | Semantics |
|---|---|
| `qaoa.ansatz_depth` | QAOA circuit depth p. Same semantics as SPEC-018. |
| `qaoa.shots_per_evaluation` | Configured shot count per optimizer evaluation. Same semantics as SPEC-018. Note: `qaoa.hardware.actual_shot_count` captures hardware-returned count. |
| `qaoa.optimizer_evaluations` | Optimizer evaluations completed before termination. Same semantics as SPEC-018. |
| `qaoa.best_bitstring_energy` | QUBO energy of the returned best-so-far bitstring. Same semantics as SPEC-018. Subject to hardware noise effects. |
| `qaoa.feasible_bitstrings_found` | Total feasible RoutePlan candidates across all evaluations. Same semantics as SPEC-018. Expected to be lower on hardware than on simulator due to noise. |
| `qaoa.solution_route_distance_km` | Total Haversine route distance of the returned RoutePlan. Same semantics as SPEC-018. |

**Presence rules:**
- Hardware evidence keys (FR-15.1): each key is present when the phase described in the "Present When" column was reached, regardless of `outcome`. `qaoa.hardware.provider_backend` is present whenever device selection occurred. `qaoa.hardware.provider_job_id` is present whenever a job was submitted to the provider.
- Shared keys (FR-15.2): present when and only when a solution is present in the response. Absent otherwise.
- `qaoa.hardware.failure_classification`: present on all `Failed` responses; also present on `Timeout` responses caused by provider outage during hardware execution (FR-12), with value `"provider_outage"`. Absent on all other Timeout and Cancelled responses.

**Acceptance Criteria:**
- All FR-15.1 hardware keys are present as specified by their "Present When" conditions
- All FR-15.2 shared keys are present when and only when a solution is present
- No key contains routing problem raw data (coordinates, demands, stop lists)
- `qaoa.hardware.failure_classification` is present in every Failed response and in every provider-outage Timeout response (FR-12); contains one of the seven defined values
- `qaoa.hardware.estimated_cost` is present in every response where hardware execution occurred; `"unknown"` is permitted if provider cost data is unavailable

---

### FR-16: Capability Profile Declaration

**Description:**
The following capability profile is the registration artifact consumed by the Scheduler (SPEC-003 FR-4, SPEC-011 FR-4). All declared values reflect hardware quantum execution constraints.

| Field | Value | Rationale and Accuracy Basis |
|---|---|---|
| `backend_id` | `qaoa-hardware` | Stable unique identifier. Provider-neutral. |
| `supported_size_classes` | `{Small}` | Provisionally matches SPEC-018. Hardware qubit count and connectivity may permit Small problems (1–25 stops) at the QUBO binary variable counts produced by available devices. Medium and Large problems (26+ stops) are not supported at MVP hardware scope: QUBO dimension for these sizes exceeds practical circuit depth on near-term devices. The actual supported subset of Small is governed by the tractable QUBO dimension for available hardware (OQ-2). |
| `supports_time_windows` | `false` | Time window fields are not consulted during optimization. Inherited from SPEC-018. |
| `supports_capacity_constraints` | `true` | Capacity constraints are encoded as QUBO penalty terms. Decoded bitstrings violating capacity are ineligible best-so-far candidates. Inherited from SPEC-018. |
| `latency_profile` | Small: TBD | Hardware execution latency includes queue wait time, which varies with provider load and can range from seconds to hours. Pre-implementation latency estimates are not meaningful. Empirical measurement is required (OQ-3). Declared as TBD until OQ-3 is resolved. |
| `quality_profile` | `Baseline` | Shallow QAOA circuits on current near-term hardware with gate errors and shot noise are not expected to produce solution quality superior to classical construction heuristics at Small problem sizes. Hardware noise degrades the optimizer's energy landscape relative to noise-free simulation. `Baseline` is the honest pre-empirical classification. Empirical validation required before `is_provisional = false` (OQ-5). |
| `cost_profile` | TBD | Hardware execution incurs monetary cost per shot executed on provider hardware. Cost depends on provider pricing model, shot count, and circuit complexity. Cannot be represented as a static cost profile without provider selection and pricing model (OQ-6). |
| `is_provisional` | `true` | Latency and quality profile values are pre-implementation estimates. Cost profile is undefined. Provider access is not confirmed. `is_provisional = true` must remain until all qualification criteria in OQ-5 are satisfied. |
| `supported_contract_version` | `1` | Targets SPEC-004 contract version 1. |

**Accuracy basis:** `supported_size_classes: {Small}` is a provisional declaration matching SPEC-018 pending hardware-specific QUBO dimension validation (OQ-2). `quality_profile: Baseline` reflects the known degradation of QAOA approximation quality under hardware noise and shallow circuit depth. `latency_profile` and `cost_profile` are not quantifiable without provider access and empirical measurement; these are flagged as open questions (OQ-3, OQ-6).

**Acceptance Criteria:**
- All nine fields from SPEC-011 FR-4.1 are present
- `is_provisional = true` is declared
- `latency_profile` and `cost_profile` values marked TBD are replaced with empirically measured values before `is_provisional = false` is declared
- The accuracy basis for all declared values is stated

---

### FR-17: Cost Reporting

**Description:**
Hardware quantum execution incurs monetary cost. This backend must report estimated execution cost in extension_metadata. Full cost reporting through the SolverResponse requires resolution of SPEC-004 OQ-2.

**Current obligation (pre-SPEC-004 OQ-2 resolution):**
- The backend reports estimated monetary cost in `qaoa.hardware.estimated_cost` (FR-15.1) as a decimal string.
- Cost is reported in the units defined by the configured provider's pricing model (provider-specific).
- If the provider does not expose per-job cost estimates, the value is `"unknown"`.
- The cost field is present in every response where hardware execution occurred (job was submitted and results were retrieved or attempted).

**Future obligation (post-SPEC-004 OQ-2 resolution):**
- If SPEC-004 OQ-2 resolves to add a `reported_cost` field to ExecutionStatistics, this backend must populate it.
- The `qaoa.hardware.estimated_cost` extension_metadata key remains for evidence granularity even after the formal field is added.

**SPEC-004 OQ-2 dependency:** SPEC-004 OQ-2 explicitly identifies cloud-based backends with usage-based pricing as the motivating scenario for formal cost reporting in the SolverResponse. SPEC-019 is the first backend that makes SPEC-004 OQ-2 non-hypothetical. Resolving SPEC-004 OQ-2 is recommended before SPEC-019 transitions out of provisional status.

**Scheduler cost_profile:** The Scheduler uses `cost_profile` for BudgetCapped filtering (SPEC-003 FR-5). For this backend, `cost_profile` cannot be a static integer without knowing the provider's pricing. The resolution of OQ-6 must produce a `cost_profile` value that the Scheduler can compare against job cost budgets. If provider pricing is per-shot, a representative shot count must be assumed.

**Acceptance Criteria:**
- `qaoa.hardware.estimated_cost` is present in extension_metadata for every response where a hardware job was submitted
- `"unknown"` is the value when the provider does not expose cost data
- SPEC-004 OQ-2 is flagged as a dependency for full cost reporting integration

---

### FR-18: Supported SolverOutcome Values

**Description:**
This backend is classified `quantum_hardware`. The supported outcome set for `quantum_hardware` category backends is defined in SPEC-011 FR-5.1 (see Documentation Updates for the required SPEC-011 amendment). This backend does not support `Infeasible` (SPEC-011 FR-5.2) and cannot satisfy the SPEC-004 FR-11.1 reproducibility invariant (SPEC-011 FR-2.1).

| Outcome | Supported | Trigger Condition |
|---|---|---|
| `Succeeded` | Yes | The optimization loop terminates naturally before the `execution_timeout_ms` deadline AND at least one feasible RoutePlan was found AND hardware execution completed without error |
| `Timeout` | Yes | `execution_timeout_ms` budget exhausted during queue wait, hardware execution, or result retrieval; also used for provider outage during execution (FR-12); best-so-far RoutePlan included if available |
| `Cancelled` | Yes | Worker cancellation signal received during queue wait or hardware execution; hardware job cancellation submitted; no solution (Worker-constructed per SPEC-005 FR-12; HTTP connection abort precludes adapter-returned data) |
| `Failed` | Yes | Contract version mismatch; device selection failure; circuit incompatibility; job submission failure; hardware execution error; natural optimizer termination with no feasible solution; internal error |
| `Infeasible` | **Not Supported** | This backend cannot prove infeasibility. Prohibited per SPEC-011 FR-5.2. |

This section satisfies the supported outcome declaration requirement in SPEC-011 FR-5.3.

**Acceptance Criteria:**
- The backend never returns `Infeasible`
- Timeout and Cancelled responses include a RoutePlan when and only when a feasible solution was decoded from hardware results before termination
- `Failed` with `InternalError` is returned when natural optimizer termination produces no feasible decoded bitstring
- Provider outage during execution produces `Timeout` with best-so-far (FR-12)

---

# Non-Requirements

- SPEC-019 does not define IBM Quantum Runtime APIs, Qiskit Runtime primitives, or IBM-specific SDK methods.
- SPEC-019 does not define Qiskit Runtime Session management.
- SPEC-019 does not define cloud authentication, API key management, or token refresh mechanisms.
- SPEC-019 does not define transpilation strategy, optimization level, or circuit compilation algorithms.
- SPEC-019 does not define error mitigation algorithms (ZNE, readout mitigation, probabilistic error cancellation, etc.).
- SPEC-019 does not define the QAOA algorithm, QUBO formulation, or bitstring decoding algorithm. These are owned by SPEC-018 and inherited by reference.
- SPEC-019 does not define the Python adapter transport. That is SPEC-017.
- SPEC-019 does not define local simulation behavior. That is SPEC-018.
- SPEC-019 does not define any solver other than this hardware-backed QAOA backend.
- SPEC-019 does not define multi-provider routing or load balancing across providers.
- SPEC-019 does not define provider credential rotation or certificate management.
- SPEC-019 does not define retry policies within a single invocation beyond the constraints in FR-11.
- SPEC-019 does not define Scheduler backend selection logic.
- SPEC-019 does not define quality evaluation metrics. That is SPEC-007.
- SPEC-019 does not define evidence persistence schema. That is SPEC-006.

---

# Assumptions

1. A cloud quantum execution provider offering gate-based quantum hardware accessible via a Python SDK is available to the deployment environment. Provider access is not confirmed at specification time (OQ-1).

2. The Python SDK for cloud quantum execution is pip-installable and can be added to the python-adapter container image's pinned requirements file (SPEC-017 FR-8) without licensing conflicts. Version pinning is required.

3. The selected hardware device has sufficient qubits and connectivity to accept transpiled QAOA circuits for Small routing problems at the qubit counts produced by the QUBO encoding strategy inherited from SPEC-018. This assumption is unvalidated (OQ-2).

4. The provider's Python SDK accepts deterministic seed parameters for transpilation (equivalent to SPEC-018's `transpiler_seed`). If the transpiler does not accept a seed parameter, transpilation nondeterminism is accepted as an additional source of hardware nondeterminism.

5. Provider queue times for the development and demonstration environment are within the `execution_timeout_ms` budget for a reasonable job budget (not necessarily minutes, but at most a few hours). Long queue times require correspondingly large `execution_timeout_ms` values, which affects the Scheduler's latency filtering.

6. The Docker Compose `python-adapter` container can reach the provider API endpoint from within the host network environment. This requires external network access from the adapter container, which is prohibited for local simulation per SPEC-017 FR-15. A Docker Compose network policy change is required for this backend (OQ-7).

7. The provider exposes a cancellation API that can cancel queued or executing jobs. This is required for FR-9. If the provider does not support cancellation, the backend cannot honor cancellation signals for in-flight jobs, and the interrupt compliance bound (SPEC-017 OQ-3B) becomes unbounded.

8. Hardware execution cost per invocation is within a development budget acceptable to the Project Owner (OQ-6). The backend must not be included in Scheduler eligibility for automated test runs without explicit cost approval.

9. ADR-007 review trigger has been satisfied: "IBM Quantum Runtime access becomes available at acceptable cost and queue latency for a development environment." This specification may be written in anticipation of that trigger; implementation is gated on the trigger being satisfied.

10. The same QUBO problem formulation and bitstring decoding algorithm from SPEC-018 applies to hardware execution. No hardware-specific problem reformulation is required.

---

# Constraints

1. This backend implements SPEC-004 contract version 1. `contract_version` mismatch must be detected before any processing begins (SPEC-011 FR-8.3, SPEC-017 FR-10).

2. The SPEC-004 FR-11.1 reproducibility invariant does not apply to this backend. The `quantum_hardware` category exemption established in SPEC-011 FR-2.1 is the governing authority.

3. Classical pre-processing uses NumPy PCG64 initialized via `SeedSequence(execution_seed, spawn_key=(BACKEND_SPAWN_KEY,))` per SPEC-017 FR-9. `BACKEND_SPAWN_KEY` must be a unique positive integer distinct from `20260620` (SPEC-018). Must be assigned before Acceptance. **Blocking: `BACKEND_SPAWN_KEY = TBD` is not valid in an Accepted specification.**

4. `execution_seed` is the exclusive authorized entropy source for classical pre-processing. Prohibited in reproducibility-critical classical paths: `random.random()`, `os.urandom()`, `time.time()`, `os.getpid()`, any unseeded NumPy generator. Hardware shot outcomes are not governed by this constraint.

5. `routing_problem.seed` must not be used as a PRNG entropy source. Only `execution_seed` seeds the classical rng.

6. The rng is initialized exactly once per solver execution. Reseeding is prohibited.

7. The backend must not return `Infeasible` (SPEC-011 FR-5.2).

8. Any RoutePlan included in a SolverResponse must satisfy all SPEC-004 FR-5 structural validity requirements. Partial assignments are never valid.

9. `extension_metadata` must not contain routing problem raw data: geographic coordinate arrays, stop identifier lists, demand arrays, time window arrays.

10. The backend must not emit OpenTelemetry spans (SPEC-011 FR-9.2).

11. The backend must not write to stdout or stderr (SPEC-017 Constraint 14).

12. No backend-specific logic may exist in the Scheduler, Worker, or Core contract-handling code (ADR-008).

13. External network access from the python-adapter container is required for this backend and must be explicitly permitted in Docker Compose network configuration. This is a constraint on deployment configuration, not on this backend's observable behavior. Provider endpoints contacted during execution must be documented (OQ-1).

14. Hardware execution must not occur without Project Owner approval of a cost budget per invocation (OQ-6). This backend must declare `is_provisional = true` until a cost budget is approved and the cost_profile is defined.

15. Ansatz depth p must be determined before Phase 1 rng draws begin, consistent with SPEC-018 Constraint 15. p may not be derived from rng draws.

---

# Inputs

| Field | How Used |
|---|---|
| `routing_problem.stops` | Stop coordinates, demands, stop_id values — QUBO formulation (inherited from SPEC-018) and bitstring decoding |
| `routing_problem.depot` | Depot coordinates — route distance computation for best-so-far quality comparison |
| `routing_problem.vehicle_count` | Determines route count; part of QUBO binary state dimension |
| `routing_problem.capacity_per_vehicle` | Enforced via QUBO penalty; capacity validity required for best-so-far candidacy |
| `execution_timeout_ms` | Total budget for all phases including queue wait; monitored continuously |
| `execution_seed` | Received as decimal string; parsed to Python integer; initializes NumPy PCG64 with BACKEND_SPAWN_KEY |
| `contract_version` | Validated by adapter before Python solver invoked; must equal 1 |
| `job_id`, `decision_id`, `backend_id` | Correlation only; do not affect execution |
| `routing_problem.seed` | Present in JSON per SPEC-017 FR-3; must not be used as PRNG entropy |

Fields not used in optimization: `routing_problem.time_window_open`, `routing_problem.time_window_close`, `routing_problem.service_duration_seconds`.

---

# Failure Modes

### Pre-Selection Failures

**No provider configured / no device available:**
- **Condition:** Provider connection is absent or no eligible device exists.
- **Outcome:** `Failed`, `InternalError`
- **Fallback:** Failed solver run recorded. Configuration investigation required.

**QUBO dimension exceeds hardware capacity:**
- **Condition:** Computed Q exceeds all available device qubit counts.
- **Outcome:** `Failed`, `InternalError`
- **Fallback:** Failed solver run with circuit_incompatible classification. Problem is ineligible for this backend.

---

### Queue Phase Failures

**Budget exhausted during queue wait:**
- **Condition:** `execution_timeout_ms` expires before hardware execution begins.
- **Outcome:** `Timeout`, no solution
- **Fallback:** Timeout recorded in evidence. `qaoa.hardware.queue_wait_ms` captures wait duration. Indicates budget is too small relative to queue latency.

**Worker cancellation during queue wait:**
- **Condition:** HTTP connection abort received while job is queued.
- **Outcome:** `Cancelled` (Worker-constructed), no solution
- **Fallback:** Cancellation recorded. Hardware job is cancelled via provider API.

---

### Hardware Execution Failures

**Hardware execution error:**
- **Condition:** Provider reports hardware error during circuit execution.
- **Outcome:** `Failed`, `InternalError`, `qaoa.hardware.failure_classification = "hardware_error"`
- **Fallback:** Failed solver run with hardware error classification. Investigation of device error logs via provider tooling recommended.

**Provider outage during execution:**
- **Condition:** Provider API becomes unreachable while job is executing.
- **Outcome:** `Timeout` with best-so-far (if available); `Timeout` with no solution otherwise.
- **Fallback:** Outage-classified Timeout recorded. `qaoa.hardware.failure_classification = "provider_outage"` is present in the Timeout response, distinguishing this infrastructure failure from time-budget Timeouts in evidence.

**Natural optimizer termination with no feasible solution:**
- **Condition:** Optimization completes before deadline; no feasible bitstring was decoded.
- **Outcome:** `Failed`, `InternalError`, `qaoa.hardware.failure_classification = "no_feasible_solution"`
- **Fallback:** Failed solver run. Indicates QUBO formulation or penalty structure does not guide hardware-noisy shots toward feasible solutions.

---

# Architectural Impact

| Component | Impact | Notes |
|---|---|---|
| Python Solver Adapter (SPEC-017) | Yes | New Python backend hosted by adapter; requires provider SDK in container image |
| Scheduler (SPEC-003) | Yes | Capability profile for `qaoa-hardware` must be registered; `cost_profile` requires OQ-6 resolution |
| Solver Contract (SPEC-004) | Yes | SPEC-004 OQ-2 (actual cost reporting) becomes non-hypothetical; resolution recommended before SPEC-019 moves out of Draft |
| Worker (SPEC-005) | None | Standard Python adapter dispatch path; no Worker changes |
| Evidence Log (SPEC-006) | None | Additional extension_metadata keys pass through existing JSONB column |
| Quality Evaluation (SPEC-007) | None | Core evaluates quality from normalized RoutePlan; no changes |
| Deployment | Yes | Provider SDK added to python-adapter requirements; external network access required from adapter container; Docker Compose network policy change required |
| Observability | None | Worker-emitted `solver.execute` span unchanged; hardware evidence flows through extension_metadata |
| Security | Yes | External network access from adapter container introduces new attack surface; provider credentials must be managed securely (OQ-1) |
| ADR-007 | Yes | Review trigger for this specification: "IBM Quantum Runtime access becomes available at acceptable cost and queue latency." SPEC-019 is the formal realization of the hardware deferral end state. |

---

# Testability

**Note on hardware testing:** Full integration testing of this backend requires provider access. Unit testing of the adapter integration and classical pre-processing phases does not require provider access. Hardware-specific behaviors (shot outcomes, queue wait, calibration) cannot be unit tested without a mock provider or hardware access.

**Contract conformance:**

1. **Structural validity:** A Succeeded SolverResponse contains a RoutePlan satisfying all SPEC-004 FR-5 conditions 1–5. Identical to SPEC-018 Testability item 1.

2. **ContractVersionMismatch:** Version mismatch returns `Failed` with `ContractVersionMismatch` before any processing. Verified by adapter per SPEC-017 FR-10.

3. **No infeasible outcome:** The backend never returns `Infeasible` under any condition.

**Hardware-specific behaviors:**

4. **Non-reproducibility:** Two SolverRequests with identical routing problems and identical `execution_seed` values submitted to hardware produce different `solution` fields. This confirms that hardware shot outcomes vary. Note: this test requires hardware access and may not produce different results on every run.

5. **Classical pre-processing reproducibility:** Two SolverRequests with identical `execution_seed` produce identical initial QAOA parameters (Phase 2 values) and identical Phase 1 seeds. Verifiable without hardware access by logging derived seeds during testing.

6. **Queue wait recording:** `qaoa.hardware.queue_wait_ms` is present and non-negative in every response where a job was submitted. Value is zero only if the job executed immediately without queuing.

7. **Budget-exhausted timeout during queue:** Given `execution_timeout_ms` set shorter than the expected queue time, the backend produces `Timeout` with no solution and `qaoa.hardware.queue_wait_ms` reflecting the wait before cancellation.

8. **Hardware evidence field completeness:** All FR-15.1 keys are present under their specified conditions. `qaoa.hardware.provider_job_id` is present in every response where a job was submitted. `qaoa.hardware.failure_classification` is present in every Failed response.

9. **Cost field presence:** `qaoa.hardware.estimated_cost` is present in every response where hardware execution occurred; the value is a decimal string or `"unknown"`.

10. **Cancellation during queue:** An in-flight job in the queue is cancelled when the Worker aborts the HTTP connection. The adapter returns to ready state. A subsequent request succeeds normally.

**Capability profile accuracy (blocked on provider access):**

11. **Latency profile validation:** Empirical execution duration (including queue wait) on SPEC-002 Small-class problems is within the declared `latency_profile` bound. Required before `is_provisional = false`. Requires hardware access.

12. **Quality classification:** Route quality on SPEC-002 benchmark problems is consistent with `Baseline` classification. Required before `is_provisional = false`. Requires hardware access.

**Extension metadata:**

13. **Shared key presence on solution:** All six SPEC-018 shared keys (`qaoa.ansatz_depth`, etc.) are present when a solution is present; absent otherwise.

14. **No raw data in extension_metadata:** No extension_metadata value contains geographic coordinate arrays, stop lists, or demand arrays. Verified by code review and automated scan.

15. **failure_classification values:** `qaoa.hardware.failure_classification` contains one of the seven defined values on every Failed response.

**Observability:**

16. **solver.execute span:** A `solver.execute` OTel span is emitted by the Worker for every dispatch through the SPEC-017 adapter. Span status is `Unset` for Timeout-with-solution, `OK` for Succeeded, `Error` for Failed. Verifiable without hardware access via mock provider.

17. **Adapter log correlation:** `adapter.request.received`, `adapter.solver.invoked`, and `adapter.solver.completed` (or `adapter.solver.timeout`) events are emitted by the adapter with `job_id` and `decision_id` present.

---

# Observability Requirements

The `solver.execute` span is emitted by the Worker (SPEC-004 FR-15). The backend's contribution is through SolverResponse fields: `outcome` and `statistics.execution_duration_ms`.

**Hardware-specific diagnostic questions enabled by extension_metadata:**

1. How long did the job wait in the hardware queue? (`qaoa.hardware.queue_wait_ms`) — distinguishes queue latency from circuit execution latency
2. Which hardware device was selected? (`qaoa.hardware.provider_backend`) — required for hardware-specific result analysis
3. What calibration state was active? (`qaoa.hardware.calibration_id`) — enables correlation of result quality with calibration events
4. What was the transpiled circuit's depth and two-qubit gate count? (`qaoa.hardware.transpiled_circuit_depth`, `qaoa.hardware.transpiled_cx_count`) — quantifies hardware fidelity burden
5. Was error mitigation applied? (`qaoa.hardware.error_mitigation_applied`, `qaoa.hardware.error_mitigation_method`) — required for result interpretation
6. What is the provider's job identifier? (`qaoa.hardware.provider_job_id`) — enables investigation via provider tooling
7. What did hardware execution cost? (`qaoa.hardware.estimated_cost`) — required for cost-benefit evidence
8. How were feasible bitstrings affected by hardware noise? (compare `qaoa.feasible_bitstrings_found` across SPEC-018 and SPEC-019 results) — quantifies noise impact on solution quality
9. How was the failure classified? (`qaoa.hardware.failure_classification`) — distinguishes algorithm failures from infrastructure failures

---

# Security Considerations

**External network access:** This backend requires the python-adapter container to contact cloud provider endpoints. This is a new trust boundary absent from all other MVP backends. The adapter container's network access must be restricted to permitted provider endpoints only. Docker Compose network policy must enforce this (OQ-7).

**Credential management:** Provider API credentials must not appear in adapter log events, extension_metadata, or SolverResponse fields. Credential injection mechanism is an implementation planning decision (OQ-1); it must use Docker Compose secrets or equivalent, not environment variables that appear in container inspection.

**Log safety:** Extension metadata must not contain routing problem raw data. `execution_seed`, derived Phase 1 seeds, and provider credentials must not appear in adapter structured log events.

**No authentication at MVP scope:** The adapter-to-Worker channel remains unauthenticated per ADR-005. This is accepted risk for the Docker Compose MVP environment. Non-MVP deployments must add mTLS.

**Provider-side data exposure:** Routing problem data (stop coordinates, demands) is transmitted to the cloud provider as circuit parameters or as input to the QUBO construction step. The data protection implications of transmitting routing problem data to a third-party cloud provider must be addressed in the Project Owner decision (OQ-1) before this backend is enabled.

---

# Performance Considerations

**Queue dominates latency:** For most hardware executions, queue wait time will dominate total `execution_duration_ms`. A latency profile based on circuit execution time alone severely understates actual response time. The `latency_profile` in FR-16 must reflect median end-to-end time including expected queue wait.

**QUBO dimension and circuit scaling:** The exponential scaling of circuit simulation (SPEC-018 Performance Considerations) does not apply to hardware execution: hardware executes circuits of any supported depth at the same wall-clock time per shot (governed by the device's gate time and decoherence time). However, deeper circuits have higher error rates and longer execution times per shot. The tractable depth range for achieving useful results on hardware is empirically determined (OQ-2).

**Cost vs. quality tradeoff:** Higher shot counts improve the probability of sampling feasible bitstrings at the cost of higher monetary cost per evaluation. The relationship between shot count, feasible bitstring probability, and total cost is a key empirical question (OQ-5, OQ-6).

**`execution_duration_ms` scope:** Per SPEC-017 FR-5, `execution_duration_ms` is measured from the start of Python solver computation (after adapter overhead) to the start of response construction. This includes Phase 1 (rng draws, transpilation), Phase 2 (parameter initialization), Phase 3 (queue wait + hardware execution + result retrieval), and decoding. It excludes JSON serialization.

---

# Documentation Updates Required

**SPEC-004:**
- OQ-2 (actual execution cost reporting) is the first concrete scenario requiring resolution. SPEC-019's monetary cost reporting obligation makes OQ-2 non-hypothetical. A `reported_cost` field in ExecutionStatistics should be considered before SPEC-019 moves to Proposed.
- FR-10 (Timeout Behavior): SPEC-019 FR-12 uses `Timeout` for provider outage during hardware execution — the only case in this contract where `Timeout` is produced by an infrastructure failure rather than a time budget event. The purpose is to preserve best-so-far results when partial hardware execution has occurred. The outcome is distinguished from time-budget Timeouts by `qaoa.hardware.failure_classification = "provider_outage"` in extension_metadata (FR-15.1). SPEC-004 FR-10 must explicitly acknowledge this Timeout usage for `quantum_hardware` anytime backends before SPEC-019 advances to Proposed. This amendment does not add new SolverOutcome values, does not change the SolverResponse schema, and does not alter Timeout semantics for non-hardware backends.

**SPEC-011 FR-3.1 (DeterminismClass controlled vocabulary):**
- The `determinism_class` metadata field's controlled vocabulary (SPEC-011 FR-3.1) currently defines two valid values: `Deterministic` and `Stochastic (reproducible)`. This specification declares `determinism_class = "Stochastic (non-reproducible)"`, which is described as the `quantum_hardware` category's determinism class in SPEC-011 FR-2.1 but is not listed in FR-3.1's controlled vocabulary. SPEC-011 FR-3.1 must be amended to add `Stochastic (non-reproducible)` as a third valid DeterminismClass value with definition: "Non-reproducible by nature; hardware entropy precludes reproducibility from `execution_seed` alone. Applies to `quantum_hardware` category backends (SPEC-011 FR-2.1)." This amendment is required before SPEC-019 transitions to Proposed.

**SPEC-011 FR-5.1 (Supported SolverOutcome Values matrix):**
- SPEC-011 FR-5.1 does not include a `quantum_hardware` column in its supported SolverOutcome Values matrix. SPEC-011 FR-5.1 must be amended to add a `quantum_hardware` column with: `Succeeded` — Required; `Timeout` — Required; `Cancelled` — Required; `Failed` — Required; `Infeasible` — Not Supported. This amendment is required before SPEC-019 transitions to Proposed.

**SPEC-011 FR-11 (deferred backends):**
- The "Quantum Hardware" deferred backend entry cites ADR-007. SPEC-019 should be noted as the planned individual solver specification for this backend.

**SPEC-017 FR-15 (External network access prohibition):**
- SPEC-017 FR-15 prohibits external network access from the python-adapter container for all Python solver backends, citing ADR-007 hardware deferral as justification. SPEC-019 is the realization of that deferral end state and requires external network access. SPEC-017 FR-15 must be amended to add an exception for `quantum_hardware`-category backends: "Exception: Python backends in the `quantum_hardware` category (SPEC-011 FR-2.1) are permitted to make external network calls to configured, authorized provider endpoints. This exception applies only to backends for which the Docker Compose network policy change has been resolved and the adapter container's outbound access is restricted to permitted provider endpoints." The FR-15 acceptance criterion ("Python backends make no external network calls during solver execution") must be updated to apply only to non-`quantum_hardware` backends. This amendment is required before SPEC-019 transitions to Proposed.

**SPEC-017 OQ-1 (`transport_overhead_buffer_ms`):**
- The `transport_overhead_buffer_ms` value in SPEC-017 OQ-1 was calibrated for local computation serialization overhead. Hardware backends require additional time for provider API cancellation before self-terminating; this latency may exceed the current calibrated buffer. If OQ-4 resolution identifies provider cancellation round-trip times that exceed the current buffer, SPEC-017 OQ-1 must be reopened to assess whether a hardware-specific `transport_overhead_buffer_ms` value is required.

**ADR-007:**
- SPEC-019 is the formal hardware execution specification anticipated by ADR-007. ADR-007 must be updated to name SPEC-019 as the formal planned realization of the quantum hardware execution path. This update is required before SPEC-019 transitions to Proposed. The update does not change ADR-007's core hardware deferral decision; implementation remains gated on the ADR-007 review trigger ("IBM Quantum Runtime access becomes available at acceptable cost and queue latency") being satisfied, and SPEC-019 does not imply hardware execution is part of the current MVP until those conditions are met.

**SPEC-018:**
- No content changes required. SPEC-019 defines a separate backend that reuses SPEC-018's QUBO representation and decoding model. SPEC-018 may note in its Documentation Updates section that SPEC-019 is the hardware execution counterpart.

**docs/architecture.md:**
- No changes required at Draft status.

---

# Open Questions

### OQ-1: Provider Selection and Credential Management

**Classification:** Project Owner Decision Required

**Question:** Which cloud quantum execution provider(s) are authorized for use with this backend, and through what mechanism are provider credentials injected into the Python adapter container?

**Why it matters:** Provider selection determines the available hardware devices, qubit counts, pricing model, SDK compatibility, and data protection terms. Routing problem data is transmitted to the provider as part of circuit execution; this has data protection implications that require Project Owner decision. Credential management determines the security posture of the deployment.

**Blocking:** Blocks implementation and any hardware test execution. Does not block SPEC-019 Draft or Proposed status.

---

### OQ-2: Supported Hardware Families and QUBO Dimension Tractability

**Classification:** Implementation Planning Decision (gated on OQ-1 resolution)

**Question:** What hardware device families are eligible for use, and what is the maximum QUBO binary variable count Q that produces useful results (feasible bitstring probability above a threshold) on available devices within a reasonable shot budget?

**Why it matters:** Q determines which problem sizes can be addressed. If available devices support only Q ≤ 20 (typical for near-term devices with useful fidelity), the supported_size_classes may need to be narrowed to a subset of Small. The relationship between Q, circuit depth, error rate, and feasible bitstring probability must be established empirically.

**Blocking:** Blocks capability profile finalization. Does not block SPEC-019 Draft status.

---

### OQ-3: Runtime Budget Limits and Latency Profile

**Classification:** Project Owner Decision Required (budget limit) + Implementation Planning Decision (empirical measurement)

**Question:** What maximum `execution_timeout_ms` value is reasonable for hardware execution jobs, and what is the empirically observed end-to-end latency (including queue wait) for Small routing problems on target hardware?

**Why it matters:** Hardware queue times can be long and variable. A latency profile that does not account for queue time will cause the Scheduler to underestimate hardware execution duration. The Project Owner must decide whether to authorize hardware jobs with long queue budgets. If queue times are unpredictable, the latency profile may need to be expressed as a distribution rather than a point estimate.

**Blocking:** Blocks `latency_profile` declaration and `is_provisional = false` transition. Does not block Draft status.

---

### OQ-4: Provider Cancellation Semantics and Partial Result Availability

**Classification:** Implementation Planning Decision (gated on OQ-1 resolution)

**Question:** Does the selected provider support: (a) cancellation of queued jobs; (b) cancellation of executing jobs; (c) partial result retrieval from cancelled or timed-out jobs?

**Why it matters:** The cancellation behavior defined in FR-9 and timeout behavior in FR-8 assume that provider cancellation is possible. If the provider does not support cancellation, in-flight job cancellation is impossible, and the interrupt compliance bound (SPEC-017 OQ-3B) cannot be satisfied. Partial result availability determines whether Timeout responses can include a best-so-far solution.

**transport_overhead_buffer_ms dependency:** The OQ-4 resolution must include an estimate of provider API cancellation round-trip time (call submission latency plus provider acknowledgment latency). This estimate must be provided to the SPEC-017 OQ-1 resolution process: the `transport_overhead_buffer_ms` calibration was performed for local computation and may be insufficient for hardware-backend self-termination, which requires issuing a provider API cancellation request before the Worker's HTTP client timeout fires. See Documentation Updates (SPEC-017 OQ-1).

**Blocking:** Blocks implementation of FR-8 and FR-9. Does not block Draft status.

---

### OQ-5: Empirical Qualification Criteria

**Classification:** Project Owner Decision Required

**Question:** What conditions must this backend satisfy to declare `is_provisional = false` in its capability profile?

**Context:** For local simulation backends (SPEC-018), qualification requires empirical latency and quality measurements on SPEC-002 workloads. For hardware execution, additional criteria apply: confirmed provider access, approved cost budget per invocation, minimum feasible bitstring frequency across a benchmark workload, and hardware latency measurements including queue time. The specific thresholds and criteria are a Project Owner decision.

**Blocking:** Blocks `is_provisional = false` declaration. Does not block Draft status or initial implementation.

---

### OQ-6: Cost Thresholds and cost_profile Definition

**Classification:** Project Owner Decision Required

**Question:** What is the maximum acceptable monetary cost per hardware execution invocation for development and demonstration use, and what value should `cost_profile` declare to allow the Scheduler to filter this backend under BudgetCapped objectives?

**Why it matters:** The Scheduler's BudgetCapped filter (SPEC-003 FR-5) uses `cost_profile` to exclude backends that exceed a configured cost budget. For hardware backends, cost is a real monetary value that must not be left as an unbounded unknown. A Project Owner decision is required to define acceptable per-invocation cost and the translation to the `cost_profile` units used by the Scheduler.

**Blocking:** Blocks `cost_profile` declaration and prevents meaningful Scheduler eligibility evaluation. Does not block Draft status.

---

### OQ-7: Docker Compose Network Policy for External Provider Access

**Classification:** Implementation Planning Decision

**Question:** What outbound network access must be permitted from the `python-adapter` Docker Compose container to reach the cloud provider API, and how is this access restricted to permitted endpoints?

**Why it matters:** SPEC-017 FR-15 prohibits external network access from the adapter container (for security and local-execution guarantees). SPEC-019 requires it. The Docker Compose network configuration must be changed, and the change must be scoped to permit access only to authorized provider endpoints, preventing unauthorized external calls from Python backend code.

**Blocking:** Blocks deployment of this backend. Does not block Draft status.

---

# Acceptance Checklist

**SPEC-011 Framework Obligations (FR-12):**

- [ ] Metadata: All FR-3 metadata fields present (backend_id, display_name, backend_category, implementation_language, determinism_class, supported_contract_version, specification_version)
- [ ] Problem Statement: States what problem hardware execution solves; algorithmic approach; why this backend belongs in the project
- [ ] Solver Classification: Backend category declared as `quantum_hardware`; determinism class declared as Stochastic (non-reproducible); infeasibility proof capability declared as No
- [ ] Algorithm Description (FR-1): Responsibility boundary between SPEC-018 and SPEC-019 defined; inherited vs. changed behaviors distinguished
- [ ] Hardware Provider Abstraction (FR-2): Provider neutrality obligation stated; observable contract provider-independence confirmed
- [ ] Backend Selection (FR-3): Device selection obligations defined; no-device failure condition defined; device identifier evidence requirement stated
- [ ] Hardware Capability Declaration (FR-4): Behavioral constraints governing declared values stated; provisional rationale stated
- [ ] Shot Execution (FR-5): Hardware shot non-reproducibility explicitly stated; shot count obligations defined; calibration state recording required
- [ ] Queue Behavior (FR-6): Queue wait definition; queue time counted against budget; deadline monitoring during queue required; cancellation during queue defined
- [ ] Hardware Job Lifecycle (FR-7): Seven phases defined; timeout consumption by all phases stated; streaming vs. batch result availability addressed
- [ ] Timeout Behavior (FR-8): Budget decomposition defined; queue-phase timeout behavior defined; execution-phase timeout behavior defined; Worker HTTP client backstop applies
- [ ] Cancellation Behavior (FR-9): Queue-phase cancellation; execution-phase cancellation; interrupt compliance bound declared; pre-execution cancellation inherited from SPEC-017
- [ ] Best-So-Far Behavior (FR-10): Queue phase has no solution; streaming vs. batch provider constraint; feasibility gate inherited from SPEC-018
- [ ] Hardware Failure Behavior (FR-11): Circuit incompatibility; job submission failure; hardware execution error; calibration degradation (not a failure); no-feasible-solution failure
- [ ] Provider Outage Behavior (FR-12): Outage at submission/queue produces Failed; outage during execution produces Timeout with best-so-far
- [ ] Reproducibility Expectations (FR-13): SPEC-011 FR-2.1 quantum_hardware exemption cited; what IS reproducible (classical pre-processing) stated; what IS NOT reproducible (hardware shots) stated; BACKEND_SPAWN_KEY assigned to a unique positive integer; draw ordering consistent with SPEC-018
- [ ] Evidence Requirements (FR-14): All evidence categories stated; publication support rationale provided
- [ ] Extension Metadata (FR-15): All hardware-specific keys documented; shared SPEC-018 keys referenced; presence rules defined; raw-data prohibition confirmed
- [ ] Capability Profile (FR-16): All nine SPEC-011 FR-4.1 fields present; is_provisional = true; TBD fields flagged; accuracy basis stated
- [ ] Cost Reporting (FR-17): SPEC-004 OQ-2 dependency noted; estimated_cost in extension_metadata defined; future obligation stated
- [ ] Supported SolverOutcome Values (FR-18): Explicit table; Infeasible listed as Not Supported; satisfies SPEC-011 FR-5.3
- [ ] Seed Usage Policy (FR-13): Non-reproducible class stated; PCG64 for classical phases; BACKEND_SPAWN_KEY TBD; draw ordering defined; hardware shot exemption from seed governance stated; satisfies SPEC-004 FR-1 and SPEC-011 FR-6.5 and SPEC-017 FR-9
- [ ] RoutePlan Output Requirements: Same as SPEC-018; presence rules per outcome; capacity validity guarantee; no partial assignments
- [ ] Failure Model: Hardware-specific failure conditions defined; inherited failure conditions from SPEC-018 referenced; failure_classification values enumerated
- [ ] Performance Characteristics: Queue dominance noted; QUBO dimension and hardware scaling addressed; cost vs. quality tradeoff identified
- [ ] Testability: Test contracts covering contract conformance, hardware-specific behaviors, capability profile accuracy, extension metadata, observability; hardware access requirements noted
- [ ] Open Questions: OQ-1 through OQ-7 classified with blocking status; no open question with unknown classification
- [ ] No constraint contradicts SPEC-011 FR-1 through FR-11 framework requirements

**SPEC-017 Compliance Obligations:**

- [ ] `BACKEND_SPAWN_KEY` assigned to a unique positive integer distinct from `20260620` (SPEC-018) and declared in this specification. **Blocking: This specification cannot transition to Accepted with `BACKEND_SPAWN_KEY = TBD`.**
- [ ] rng initialization formula follows SPEC-017 FR-9 (SeedSequence with spawn_key)
- [ ] PRNG Phase 1 draw ordering documented and consistent with SPEC-018 FR-10 Phase 1; `simulator_seed` draw consumed but not applied
- [ ] PRNG Phase 2 draw ordering documented and consistent with SPEC-018 FR-10 Phase 2
- [ ] SPEC-017 OQ-3B interrupt compliance bound declared (bounded by cancellation API round-trip time plus provider acknowledgment wait, OQ-4)
- [ ] No stdout/stderr usage confirmed (SPEC-017 Constraint 14)

**Backend-Specific Acceptance Criteria:**

- [ ] The non-reproducibility of hardware shot outcomes is explicitly and unambiguously stated
- [ ] The classical pre-processing reproducibility is explicitly stated and the boundary with hardware randomness is clear
- [ ] Queue wait time is explicitly counted against the execution_timeout_ms budget
- [ ] Provider outage during execution produces Timeout (not Failed) to preserve best-so-far
- [ ] All OQ classifications are Project Owner Decision Required or Implementation Planning Decision — no unknown classification remains
- [ ] `BACKEND_SPAWN_KEY = TBD` is flagged as requiring assignment before Acceptance

---

# Definition of Done

This backend is complete when:

- SPEC-019 is in Accepted status
- ADR-007 review trigger is satisfied: provider access is available at acceptable cost and queue latency
- OQ-1 (provider selection and credential management) is resolved by Project Owner decision
- OQ-2 (hardware device families and Q tractability) is determined empirically during implementation and reflected in a revised capability profile
- OQ-3 (runtime budget limits and latency profile) is resolved; `latency_profile` is populated with empirically measured values
- OQ-4 (provider cancellation and partial result availability) is resolved; cancellation and timeout behaviors are implemented per the resolved semantics
- OQ-6 (cost thresholds and cost_profile) is resolved by Project Owner decision; `cost_profile` is declared
- OQ-7 (Docker Compose network policy) is resolved and the adapter container is configured for provider access
- SPEC-004 OQ-2 (actual execution cost reporting) is resolved and integrated if a formal `reported_cost` field is added
- BACKEND_SPAWN_KEY is assigned (unique positive integer distinct from `20260620`) and declared in this specification
- The backend is implemented as a Python solver function hosted by the SPEC-017 python-adapter container
- All SPEC-004 contract conformance tests pass via the SPEC-017 adapter (to the extent hardware access permits)
- Hardware evidence fields (FR-15.1) are populated in integration tests against the target provider
- `solver.execute` span is emitted by the Worker for every adapter invocation and verifiable in the test environment
- Extension metadata keys are validated: all FR-15.1 keys present under specified conditions; all FR-15.2 shared keys present when solution is present
- Adapter structured log events are emitted and correlated via job_id and decision_id
- Self-termination before the Worker's HTTP client timeout is verified on test cases that trigger the timeout path
- Empirical latency and quality values are measured; `is_provisional = false` declared after OQ-5 qualification criteria are satisfied
- OQ-5 (empirical qualification criteria) is reviewed against measurements; thresholds are within Project Owner-approved bounds
- The provider SDK and credentials are documented and secured; Docker Compose network policy is configured
- Engineering review passes
- Specification status is updated to Verified
