# SPEC-018: QAOA Solver Backend

## Metadata

**Feature ID:** SPEC-018

**Title:** QAOA Solver Backend

**Status:** Proposed

**Author:** Darkhorse286

**Created:** 2026-06-20

**Last Updated:** 2026-06-20 (Revised — Engineering Review complete; F-001 through F-005 and NB-003, NB-004 applied; status advanced to Proposed)

**Supersedes:** None

**Superseded By:** None

**Parent Specification:** SPEC-011 (Backend Solver Specifications)

**Adapter Specification:** SPEC-017 (Python Solver Adapter)

**Review History:**
- Engineering Review: Completed 2026-06-20

**Related ADRs:** ADR-005, ADR-006, ADR-007, ADR-008, ADR-009, ADR-010, ADR-011

**Related Specs:** SPEC-001, SPEC-002, SPEC-003, SPEC-004, SPEC-005, SPEC-006, SPEC-007, SPEC-010, SPEC-011, SPEC-012, SPEC-015, SPEC-017

---

**Solver Metadata (SPEC-011 FR-3):**

| Field | Value |
|---|---|
| `backend_id` | `qaoa-qiskit` |
| `display_name` | QAOA (Qiskit, Local Simulator) |
| `backend_category` | `quantum_inspired_stochastic` |
| `implementation_language` | `Python` |
| `determinism_class` | Stochastic (reproducible) |
| `supported_contract_version` | 1 |
| `specification_version` | 1 |

---

# Problem Statement

Project DAEDALUS documents the evidence thesis: most routing problems should never touch a quantum computer. Demonstrating this claim with credibility requires the system to actually execute a quantum-adjacent solver so that the Scheduler's rejection decisions are grounded in real evidence, not hypothetical comparison. Without a solver that executes a quantum-style algorithm, the system can compare classical heuristics against each other but cannot produce evidence about the cost and quality tradeoffs of promoting a quantum-inspired backend.

The QUBO Simulated Annealing backend (SPEC-015) provides the first quantum-inspired backend using a classical C++ process. SPEC-015 establishes the evidence baseline for quantum-adjacent execution cost. However, simulated annealing does not use a quantum circuit and cannot produce evidence about the characteristics of actual variational quantum execution — the initialization overhead, shot-based sampling noise, classical-quantum feedback loop, and parameterized circuit depth behavior that distinguish QAOA from purely classical approaches.

SPEC-018 defines the QAOA solver backend: a variational quantum algorithm implemented in Python using Qiskit, executed on a local Aer simulator through the SPEC-017 Python Solver Adapter. This backend produces evidence about the actual behavior of a QAOA execution path — including transpilation cost, circuit evaluation overhead, optimizer iteration count, and the frequency with which shot-based measurements yield feasible solutions — without requiring quantum hardware. ADR-007 explicitly identifies Qiskit Aer local simulation as the appropriate vehicle for this purpose and defers hardware execution.

SPEC-018 defines observable solver behavior: what inputs the backend consumes, what outputs it produces, how it handles timeouts and cancellations, what reproducibility guarantees it provides, and what diagnostic evidence it emits. It does not define Qiskit circuit construction details, QUBO encoding schemes, QAOA parameter initialization strategies beyond their reproducibility obligations, or the classical optimizer algorithm.

---

# Business Value

- Introduces a true quantum-circuit-executing backend into the Project DAEDALUS solver inventory, enabling the Scheduler to produce evidence comparing QAOA execution characteristics against classical heuristics and QUBO simulated annealing
- Validates the SPEC-017 Python Solver Adapter end-to-end on a real Qiskit workload, confirming that timeout self-termination, seed propagation, and the JSON over HTTP transport function correctly under a circuit-based execution model
- Demonstrates that QAOA circuit execution on a local simulator is significantly more expensive than simulated annealing at equivalent problem sizes, providing the evidence basis for the Scheduler's rejection of this backend for most routing workloads
- Produces QAOA-specific diagnostic evidence — ansatz depth, shot count, optimizer evaluations, bitstring energy, feasible solution frequency — that the evidence report can surface to explain the cost-quality tradeoff of variational quantum approaches
- Extends the project's employer signaling from quantum-inspired classical optimization (SPEC-015) to actual variational quantum algorithm execution, demonstrating familiarity with the QAOA algorithm, Qiskit, and the engineering concerns unique to quantum-circuit-based backends

---

# Employer Signaling

- System Design
- Optimization
- AI Engineering
- Reliability Engineering

QAOA is a variational quantum algorithm that requires controlling multiple randomness sources across a classical-quantum feedback loop: parameter initialization, circuit transpilation, shot-based measurement sampling, and classical optimizer convergence. Specifying a reproducible QAOA backend requires correctly identifying all Qiskit randomness sources, defining deterministic seed derivation for each, and establishing the phase boundaries within which each source is consumed. This demonstrates the ability to specify stochastic Python backend behavior against a formal reproducibility policy (ADR-010, SPEC-017 FR-9) in a setting where the randomness model is more complex than a single seeded PRNG — it involves multiple interacting sub-systems that must each be deterministically seeded from a single root execution seed.

---

# Solver Classification

**Backend category:** `quantum_inspired_stochastic`

**Determinism class:** Stochastic (reproducible)

**Infeasibility proof capability:** No

The QAOA backend uses a PRNG-dependent stochastic optimization process. Shot-based circuit measurements produce outcomes that depend on the quantum state probability distribution; on a local simulator these outcomes are reproducible given the same circuit and measurement seed. The backend's solution output is not identical for identical routing problem inputs in general: two invocations with different `execution_seed` values may produce different route plans. However, given the same `execution_seed`, the backend produces an identical `solution` on every invocation (SPEC-004 FR-11 reproducibility invariant, SPEC-017 FR-9 reproducibility invariant).

Classification is `quantum_inspired_stochastic` per SPEC-011 FR-2.1: "stochastic optimization heuristics using PRNG-dependent search over a solution space." QAOA's shot-based measurement sampling is PRNG-dependent on a local simulator, and its variational parameter optimization depends on reproducibly initialized random parameters and optimizer state. Classification is a property of the algorithm's stochastic execution, not a runtime configuration parameter.

The backend is not an exact solver. It cannot prove that a routing problem has no feasible solution. Conditions where no feasible bitstring is found after complete optimization produce `Failed` with `InternalError`, not `Infeasible`.

**Timeout behavior:** This backend implements anytime behavior per SPEC-004 FR-10. It maintains a running best-so-far complete solution — a RoutePlan satisfying all SPEC-004 FR-5 structural validity requirements — that is returnable on Timeout or Cancelled. The best-so-far is updated after each optimizer evaluation in which a feasible bitstring is found. When the `execution_timeout_ms` deadline is reached, the backend self-terminates at the next optimizer evaluation boundary and returns the best-so-far solution if one has been found. See FR-7 and FR-11.

**Supported SolverOutcome values:** See FR-11.

---

# Requirements

### FR-1: Algorithm Description

**Description:**

The QAOA backend solves the Capacitated Vehicle Routing Problem (CVRP) by encoding the routing problem as a Quadratic Unconstrained Binary Optimization (QUBO) instance and then using the Quantum Approximate Optimization Algorithm (QAOA) to search the binary solution space for low-energy, feasible route assignments. QAOA prepares a parameterized quantum state over binary assignments, samples measurement outcomes (bitstrings) from that state, evaluates the corresponding QUBO energy classically, and adjusts the quantum circuit parameters using a classical outer optimizer to drive the measurement distribution toward lower-energy (better-quality) assignments.

This section defines externally observable algorithm behavior. It does not define the internal QUBO energy function, penalty weight strategy, binary variable encoding scheme, circuit structure, Qiskit primitive selection, or decoding algorithm. Those are implementation planning decisions governed by FR-2.

**High-level execution phases:**

1. **QUBO construction and circuit initialization:** The backend constructs the QUBO representation of the routing problem and initializes the QAOA ansatz circuit of depth p (p layers of cost and mixer unitaries, parameterized by vectors γ and β of length p). The QUBO construction and circuit structure are implementation planning decisions (FR-2). All PCG64 draws and rng-derived values consumed for seed derivation and initial parameter generation occur in this phase (FR-10, Phases 1 and 2), before the first optimizer evaluation begins.

2. **Variational optimization loop:** The classical outer optimizer iteratively adjusts the QAOA parameters (γ, β) to minimize the expected QUBO energy measured from the circuit. Each optimizer evaluation consists of:
   - Preparing the QAOA circuit with the current parameters
   - Transpiling the circuit for the local simulator (transpilation may be performed once before the loop or per evaluation; this is an implementation planning decision; the transpiler seed is frozen per FR-10)
   - Executing the circuit for `shots_per_evaluation` measurements on the local simulator
   - Computing the expected QUBO energy from measurement outcomes
   - Decoding each measured bitstring into a RoutePlan candidate (FR-6)
   - Updating the best-so-far solution if a feasible RoutePlan with lower total Haversine distance is found (FR-7)
   - Returning the expected energy to the classical optimizer

3. **Termination:** The optimization loop terminates when the classical optimizer declares convergence, when the maximum optimizer evaluation budget is exhausted, or when the `execution_timeout_ms` deadline arrives — whichever occurs first. At termination, the backend returns the appropriate outcome with the best-so-far solution if one exists.

**Distance metric:** Haversine great-circle distance in kilometers (SPEC-001 FR-5). No alternative metric is used during route distance evaluation or best-so-far comparison.

**Service duration and time window behavior:** Service duration values and time window fields are not consulted during the optimization process. Route quality optimization is based exclusively on Haversine route distance and QUBO constraint satisfaction. Time window feasibility is evaluated post-execution by Core Quality Evaluation (SPEC-007).

**Completion condition (Succeeded path):** The backend returns `Succeeded` when the optimization loop terminates naturally — the optimizer converges or the evaluation budget is exhausted before the `execution_timeout_ms` deadline — AND at least one complete, structurally valid RoutePlan was found during the optimization loop.

**Timeout condition:** The backend returns `Timeout` when the `execution_timeout_ms` deadline arrives before natural optimizer termination. The best-so-far solution is included if one was found before the deadline. See FR-7 and FR-11.

**Acceptance Criteria:**
- The algorithm executes in the three phases above: initialization (including all rng draws), optimization loop, termination
- Service duration and time window fields are not consulted during the optimization process
- `Succeeded` is returned only when the optimizer terminates naturally before the deadline AND a feasible solution was found
- `Timeout` is returned when the deadline arrives; best-so-far is included if available

---

### FR-2: QUBO Formulation and Qiskit Ownership Boundaries

**Description:**

This section defines which aspects of the QAOA execution are within SPEC-018's scope and which belong to implementation planning or adjacent specifications.

**SPEC-018 owns:**
- The observable behavioral contract: given a SolverRequest, the backend must produce a SolverResponse conforming to SPEC-004, delivered via the SPEC-017 adapter
- The reproducibility invariant: identical `execution_seed` values produce identical solutions across invocations (FR-10, SPEC-017 FR-9)
- The PRNG policy: NumPy PCG64 initialized via `SeedSequence(execution_seed, spawn_key=(BACKEND_SPAWN_KEY,))` with `BACKEND_SPAWN_KEY = 20260620` is the primary entropy source (FR-10); Qiskit sub-system seeds derived from it per FR-10 Phase 1
- The anytime contract: best-so-far tracking and its implications for Timeout and Cancelled responses (FR-7)
- The Qiskit randomness obligation: what Qiskit randomness sources exist and that each is seeded from a value derived from `execution_seed` (FR-10); the specific Qiskit API parameters used to set those seeds are implementation planning details
- The extension metadata key definitions: what QAOA diagnostic evidence is produced and when (FR-13)
- The failure model: what conditions produce each outcome (FR-14)
- The interrupt compliance bound: the maximum time from receiving a stop signal to halting computation (FR-14)

**Implementation planning owns:**
- The binary encoding scheme: how stops, vehicles, and assignments are represented as binary variables in the QUBO
- The QUBO objective function: the specific quadratic binary objective, including penalty terms for capacity constraints, assignment completeness, and route distance encoding
- The penalty weights: the relative magnitudes of penalty and objective terms
- The QAOA circuit construction: the specific gate sequence for cost and mixer unitaries
- The Qiskit primitive selection: which Qiskit execution primitive (Sampler, Estimator, or equivalent) is used to evaluate the circuit
- The transpilation strategy: when transpilation occurs (once before the loop, or per evaluation), which optimization level, and how the transpiler seed is passed to Qiskit
- The classical optimizer algorithm: which optimizer (COBYLA, SPSA, BFGS, or equivalent) is selected, and how its seed is used
- The ansatz depth p: how p is determined (fixed, problem-size-dependent, or configurable); p ≥ 1 is required
- The shot count: how `shots_per_evaluation` is determined (fixed, problem-size-dependent, or configurable); at least 1 shot per evaluation is required
- The initial parameter range: the value ranges for γ and β initialization (must use the rng draws defined in FR-10 Phase 2)
- The decoding algorithm: how a measured bitstring is mapped to a RoutePlan candidate (must satisfy FR-6)
- The repair or discard strategy for infeasible decoded states: how bitstrings that decode to invalid RoutePlans are handled (discard, repair, or penalty assignment)
- The optimizer evaluation budget: the maximum number of classical optimizer function evaluations before natural termination is declared

**Adjacent specification ownership:**
- Core quality evaluation of the returned RoutePlan: SPEC-007
- Evidence persistence schema for solver run records: SPEC-006, SPEC-012
- Capability profile consumption by the Scheduler: SPEC-003 FR-4
- Workload feature computation: SPEC-010
- External timeout enforcement and HTTP client backstop: SPEC-005, SPEC-017 FR-5
- JSON serialization and adapter transport: SPEC-017

**Acceptance Criteria:**
- No implementation planning decision invalidates the reproducibility invariant (FR-10): the BACKEND_SPAWN_KEY, Phase 1 draw ordering, and Phase 2 draw ordering are frozen once this specification is Accepted
- No implementation planning decision causes the backend to return `Infeasible` (prohibited per SPEC-011 FR-5.2)
- Every RoutePlan included in a SolverResponse satisfies all SPEC-004 FR-5 structural validity requirements; decoding and filtering logic must ensure this

---

### FR-3: Ansatz Depth Handling

**Description:**

The QAOA ansatz depth p is the number of cost-mixer unitary layer pairs in the parameterized quantum circuit. SPEC-018 defines the observable properties and constraints of ansatz depth; the specific value of p and its derivation strategy are implementation planning decisions (FR-2).

**Observable constraints on p:**
- p must be a positive integer: p ≥ 1. A depth-0 circuit is not a valid QAOA circuit.
- The actual p used in a given execution is reported in `extension_metadata` as `qaoa.ansatz_depth` (FR-13).
- For a given execution, p is fixed before the optimization loop begins. The depth does not change between optimizer evaluations.
- p governs the 2p variational parameters (p gamma values and p beta values) initialized in FR-10 Phase 2. The parameter count is deterministically derived from p.

**Depth-quality tradeoff:** Higher p values allow the circuit to represent more complex probability distributions over the binary solution space, increasing the theoretical approximation ratio achievable by QAOA. However, higher p values also increase circuit depth (more gate layers), which on a local simulator increases execution time per evaluation and overall latency. The choice of p is a key implementation planning decision that must balance achievable quality against latency budget.

**Simulator constraint:** Local Aer simulator performance degrades with circuit depth. For routing problems at Small scale (1–25 stops), the QUBO binary variable count and circuit depth must remain within the simulator's tractable range. The specific qubit count and depth limits are implementation planning concerns; they must be validated empirically before `is_provisional` can be set to false.

**Acceptance Criteria:**
- p ≥ 1 in every execution
- `qaoa.ansatz_depth` in extension_metadata equals the actual p used
- p is fixed before the first optimizer evaluation

---

### FR-4: Shot Execution

**Description:**

At each optimizer evaluation, the QAOA circuit is executed for `shots_per_evaluation` measurements. Each measurement produces a bitstring sampled from the probability distribution encoded in the quantum circuit state.

**Observable constraints on shot execution:**
- `shots_per_evaluation` must be a positive integer: shots_per_evaluation ≥ 1. An evaluation with zero shots is not valid.
- The actual shot count used is reported in `extension_metadata` as `qaoa.shots_per_evaluation` (FR-13).
- `shots_per_evaluation` is fixed before the optimization loop begins. The shot count does not change between optimizer evaluations.
- Shot-based measurement outcomes on a local simulator are reproducible given the simulator's seeded state (derived from `execution_seed` per FR-10 Phase 1).

**Bitstring pool:** The set of `shots_per_evaluation` bitstrings from each optimizer evaluation constitutes the pool from which feasible RoutePlan candidates are decoded (FR-6). All bitstrings in each evaluation are decoded; those producing valid RoutePlans are compared to the current best-so-far.

**Shot reproducibility:** The local Aer simulator produces identical shot outcomes for identical circuits, identical parameters, and an identical measurement seed (derived from `execution_seed` per FR-10). This ensures that the same execution_seed always produces the same pool of bitstrings at each optimizer evaluation, which in turn ensures the reproducibility invariant for the best-so-far solution.

**Acceptance Criteria:**
- shots_per_evaluation ≥ 1 in every execution
- `qaoa.shots_per_evaluation` in extension_metadata equals the actual shot count per evaluation
- Shot outcomes are reproducible given the same execution_seed (via the seeded simulator)

---

### FR-5: Classical Optimizer Behavior

**Description:**

The QAOA backend includes a classical outer optimizer that adjusts the QAOA parameters (γ, β) between circuit evaluations to minimize the expected QUBO energy. SPEC-018 defines observable optimizer behaviors; the optimizer algorithm and its parameters are implementation planning decisions (FR-2).

**Observable optimizer behaviors:**
- The optimizer receives the expected QUBO energy from each circuit evaluation and produces updated parameter values.
- The optimizer terminates when it declares convergence (the parameter update falls below a convergence threshold) or when it reaches its maximum evaluation count. Both termination conditions result in `Succeeded` if a feasible solution was found.
- The optimizer evaluation count at termination is reported in `extension_metadata` as `qaoa.optimizer_evaluations` (FR-13).
- If the `execution_timeout_ms` deadline arrives during an optimizer evaluation, the backend terminates the current evaluation and self-terminates, returning `Timeout` with the best-so-far solution (FR-7, FR-14).

**Optimizer reproducibility:**
- If the classical optimizer is stochastic (for example, SPSA, which uses random gradient estimates), it must be seeded from `optimizer_seed` derived deterministically from `execution_seed` in FR-10 Phase 1.
- If the classical optimizer is deterministic (for example, COBYLA or L-BFGS-B), no additional seeding is required beyond the initial parameter values derived from the rng in FR-10 Phase 2.
- In either case, the choice of optimizer does not alter the reproducibility invariant: identical executions with identical `execution_seed` values produce identical parameter trajectories and identical solution outcomes.

**Timeout during optimizer iteration:** The backend checks elapsed time at the boundary between optimizer evaluations — after one evaluation completes and before the next begins. When the remaining time before the deadline falls below a threshold sufficient to finalize the response, the backend does not begin the next optimizer evaluation and instead proceeds to response construction. The specific threshold and check mechanism are implementation planning decisions. See FR-14.

**Acceptance Criteria:**
- The optimizer terminates due to convergence, evaluation budget exhaustion, or deadline — no other termination conditions are defined
- `qaoa.optimizer_evaluations` equals the number of circuit evaluations completed before termination
- If the optimizer is stochastic, its behavior is deterministic given the same `execution_seed` (via `optimizer_seed` from FR-10 Phase 1)

---

### FR-6: Bitstring Decoding and Route Construction

**Description:**

At each optimizer evaluation, the `shots_per_evaluation` measured bitstrings are decoded into RoutePlan candidates. Only bitstrings that produce structurally valid RoutePlans are eligible as best-so-far candidates.

**Decoding obligations:**
- Each bitstring from the shot pool is decoded into a RoutePlan candidate according to the decoding algorithm (implementation planning decision per FR-2).
- The decoded RoutePlan must be evaluated against all SPEC-004 FR-5 structural validity requirements:
  1. `routes.size()` equals `vehicle_count`
  2. Every stop ID is a valid stop ID from the routing problem
  3. No stop ID appears in more than one route
  4. Every stop ID appears in exactly one route
  5. For each route, total stop demand ≤ `capacity_per_vehicle`

**Feasibility gate:** A decoded RoutePlan is eligible as a best-so-far candidate if and only if it satisfies all five SPEC-004 FR-5 conditions above. Bitstrings that produce structurally invalid RoutePlans — including partial assignments, duplicate assignments, or capacity violations — are ineligible. They contribute to the shot pool size and optimizer energy estimate but do not update the best-so-far.

**Capacity constraint handling:** Capacity constraints are encoded in the QUBO objective via penalty terms (FR-2). A decoded state that violates capacity does not qualify as a best-so-far candidate regardless of its QUBO energy. The final returned RoutePlan always satisfies capacity validity when present.

**Distance metric for ranking:** Among all feasible RoutePlans found at a given optimizer evaluation, the one with the minimum total Haversine route distance (sum of all vehicle route distances including depot-to-first-stop and last-stop-to-depot legs) is compared to the current best-so-far (FR-7).

**Time window and service duration non-enforcement:** Time window and service duration fields are not consulted during bitstring decoding or route construction. Decoded feasibility is evaluated exclusively against SPEC-004 FR-5 structural requirements and capacity. Time window feasibility is evaluated by Core Quality Evaluation (SPEC-007) after the response is received by the Worker.

**Acceptance Criteria:**
- Every RoutePlan included in any SolverResponse satisfies all SPEC-004 FR-5 conditions 1–5
- No partial assignment is ever included in a SolverResponse
- Bitstrings that fail any SPEC-004 FR-5 condition are not included as best-so-far candidates, regardless of their QUBO energy

---

### FR-7: Best-So-Far Solution Tracking

**Description:**

The backend implements anytime behavior per SPEC-004 FR-10 and SPEC-017 FR-5. It maintains a best-so-far RoutePlan — the best complete, structurally valid solution found at any point during the optimization loop — that can be returned on Timeout or Cancelled without waiting for natural optimizer termination.

**What qualifies as a best-so-far candidate:** A RoutePlan qualifies if and only if it satisfies all SPEC-004 FR-5 structural validity requirements (FR-6). Partial assignments and capacity-violating assignments are never valid candidates.

**Update trigger:** The best-so-far is evaluated after each optimizer evaluation by inspecting the pool of `shots_per_evaluation` decoded bitstrings. For each bitstring that produces a valid RoutePlan, the total Haversine route distance is computed. If any valid RoutePlan has a lower total distance than the current best-so-far, it replaces the best-so-far. On the first optimizer evaluation that produces at least one valid RoutePlan, the best valid solution from that evaluation becomes the best-so-far unconditionally.

**Quality comparison criterion:** Total Haversine route distance (sum over all vehicles of the Haversine distance traversed from depot, through assigned stops in visit order, back to depot). Lower total distance is better. The comparison is strict: a new solution replaces the best-so-far only if its total distance is strictly less than the current best-so-far total distance.

**Absence of best-so-far:** If no optimizer evaluation has produced a feasible RoutePlan by the time the optimization loop terminates (for any reason), the best-so-far is absent. The response includes no `solution` field.

**Anytime contract:** The best-so-far is maintained across all optimizer evaluations and is returnable at any time after the first feasible evaluation. Unlike classical construction heuristics (SPEC-013, SPEC-014), which can only return a complete solution after algorithm completion, and unlike the QUBO SA backend (SPEC-015), which updates its best-so-far continuously during annealing, the QAOA backend updates its best-so-far discretely at each optimizer evaluation boundary. The granularity is coarser: the best-so-far can only change between evaluations, not within them.

**solution_count in ExecutionStatistics:** The backend populates `solution_count` (SPEC-004 FR-6) with the number of optimizer evaluations in which the best-so-far was updated (i.e., the number of evaluations that improved upon the current best). A value of 0 means no feasible solution was ever found. A value of 1 means a feasible solution was found in exactly one evaluation and was never improved. A value of N means the best-so-far improved N times across all evaluations.

**Acceptance Criteria:**
- A RoutePlan is included in any SolverResponse if and only if at least one feasible RoutePlan was found before the corresponding termination event
- The returned RoutePlan has the minimum total Haversine route distance among all feasible RoutePlans produced across all optimizer evaluations before termination
- `solution_count` equals the number of optimizer evaluations that improved upon the previous best-so-far; it equals 0 when no feasible solution was found
- On Timeout and Cancelled responses, the best-so-far solution is present if it exists; absent otherwise

---

### FR-8: Constraint Handling

**Description:**

The QAOA backend enforces some constraints during optimization and ignores others. This section defines which constraints are enforced, which are delegated, and which produce failure outcomes.

**Capacity constraints (enforced through QUBO penalty):** Capacity constraints are encoded in the QUBO objective as penalty terms. Decoded solutions that violate capacity do not qualify as best-so-far candidates (FR-6, FR-7). The final returned RoutePlan always satisfies capacity validity when present (SPEC-004 FR-5 condition 5).

**Service duration constraints (not enforced):** Service duration values (SPEC-001 FR-16) are present in the routing problem but are not consulted during the optimization process. The objective minimizes Haversine route distance and penalizes constraint violations.

**Time window constraints (not enforced):** Time window fields (SPEC-001 FR-9) are not consulted during the optimization process. This backend declares `supports_time_windows = false` in its capability profile (FR-9), making it ineligible for time-window-constrained problems in Scheduler eligibility evaluation (SPEC-003 FR-5).

**Infeasibility detection:** This backend does not detect or prove infeasibility. It is not authorized to return `Infeasible` (SPEC-011 FR-5.2). Failure to find any feasible solution after completing the optimization loop produces `Failed` with `failure_code = InternalError`.

**Acceptance Criteria:**
- Every RoutePlan present in any SolverResponse satisfies SPEC-004 FR-5 capacity validity (condition 5)
- Time window and service duration fields are not consulted during optimization or bitstring decoding
- The backend never returns `Infeasible`

---

### FR-9: Capability Profile Declaration

**Description:**

The following capability profile is the registration artifact consumed by the Scheduler (SPEC-003 FR-4, SPEC-011 FR-4). All declared values must accurately reflect this backend's algorithm behavior. Values marked as estimates require empirical measurement before `is_provisional` can be set to false.

| Field | Value | Rationale and Accuracy Basis |
|---|---|---|
| `backend_id` | `qaoa-qiskit` | Stable unique identifier per SPEC-011 FR-3. Kebab-case per the normative definition in SPEC-011 FR-3.1. |
| `supported_size_classes` | `{Small}` | QAOA circuit qubit count scales with the QUBO binary variable count, which scales with stop count and vehicle count. Local Aer simulator tractability degrades steeply with qubit count. For Small problems (1–25 stops), the QUBO dimension may be within the simulator's tractable range at shallow ansatz depth. Medium and Large problems (26+ stops) are not supported at MVP scope: the expected QUBO dimension exceeds practical simulator capacity for any useful ansatz depth. Declaring Medium or Large as supported would overstate the backend's capability profile. |
| `supports_time_windows` | `false` | The optimization process does not incorporate time window constraints. Bitstring decoding and feasibility evaluation are based exclusively on SPEC-004 FR-5 structural validity and Haversine distance. Declaring `true` would misrepresent the backend's construction behavior. |
| `supports_capacity_constraints` | `true` | Capacity constraints are encoded as penalty terms in the QUBO objective. Decoded solutions that violate capacity are not treated as best-so-far candidates (FR-6). The returned RoutePlan always satisfies capacity validity when present. |
| `latency_profile` | Small: < 300.0s | Pre-implementation estimate based on algorithmic analysis. QAOA latency on a local simulator has multiple components: circuit transpilation (one-time or per-evaluation), shot-based circuit simulation (exponential in qubit count), and classical optimizer iterations. For Small problems at the lower end of the size class (< 10 stops), latency may be well under 60s at shallow depth. For Small problems near the upper bound (20–25 stops), the QUBO binary variable count may require deep circuits or many shots, pushing latency to 300s or more — which may consume the full `execution_timeout_ms` budget and produce Timeout-with-solution rather than Succeeded. Empirical measurement on SPEC-002 synthetic workload problems across the Small size class is required before `is_provisional` can be set to false. Latency values are expressed in seconds per SPEC-003 FR-4. |
| `quality_profile` | `Baseline` | Shallow QAOA circuits at realistic shot counts and optimizer iteration budgets do not guarantee solution quality superior to classical construction heuristics on CVRP at Small problem sizes. The QAOA approximation ratio improves with circuit depth and optimizer convergence, but local simulator tractability limits both. `Baseline` is the honest classification for MVP scope. If empirical validation demonstrates quality consistently at or above the Competitive tier, this classification should be revised as part of the `is_provisional = false` transition. |
| `cost_profile` | `1` | In-process Python execution within the Docker Compose python-adapter container. No external network calls, no paid compute, no cloud API charges (ADR-007). Value of `1` represents the minimum relative cost unit, consistent with other in-process MVP backends. Qiskit simulation computational cost is captured in `latency_profile`, not `cost_profile`. |
| `is_provisional` | `true` | Latency and quality profile values are pre-implementation estimates. This backend must declare `is_provisional = true` until empirical measurement validates the declared values per SPEC-011 FR-4.2. |
| `supported_contract_version` | `1` | Targets SPEC-004 contract version 1. |

**Accuracy basis:** The `supported_size_classes: {Small}` declaration is based on the known exponential scaling of state-vector simulation with qubit count: a QUBO with Q binary variables requires a 2^Q-dimensional state vector. For Small problems (1–25 stops) with 2–4 vehicles, Q may range from approximately 2 to 100 binary variables depending on the encoding strategy. Tractable simulation is achievable for small Q values; larger Q values rapidly become intractable. The exact feasible Q range must be empirically determined at implementation time and reported as part of `is_provisional = false` qualification. `quality_profile = Baseline` reflects QAOA's known performance characteristics at shallow depth on local simulators without noise models; it does not claim quantum advantage.

**Acceptance Criteria:**
- All nine fields from the SPEC-011 FR-4.1 table are present
- `backend_id` matches the FR-3 metadata value
- `supported_contract_version` equals 1
- `is_provisional = true` is declared and remains true until empirical validation of latency and quality values is complete
- The accuracy basis for `latency_profile` and `quality_profile` is stated
- `latency_profile` values are expressed in seconds per SPEC-003 FR-4

---

### FR-10: Seed Usage Policy

**Description:**

The QAOA backend is a stochastic Python backend executed through the SPEC-017 adapter. Its reproducibility depends on `execution_seed` from the SolverRequest and on the frozen `BACKEND_SPAWN_KEY`. This section satisfies the documentation obligation stated in SPEC-004 FR-1, SPEC-011 FR-6.5, and SPEC-017 FR-9.

**Determinism class:** Stochastic (reproducible)

**PRNG algorithm:** NumPy PCG64, initialized via `numpy.random.SeedSequence` per SPEC-017 FR-9. No other PRNG is permitted in reproducibility-critical paths.

**BACKEND_SPAWN_KEY:** `20260620`

This value is the spawn key parameter passed to `numpy.random.SeedSequence`. It is a positive integer. It is frozen once this specification is Accepted; changing it requires the full ADR-010 Decision 5 breaking change procedure, which includes updating ADR-010 with an explicit change record. A `specification_version` increment alone is insufficient. Future Python backend specifications (SPEC-019 and beyond) must declare a spawn key distinct from `20260620`.

Spawn keys and C++ PCG64 stream constants (such as SPEC-015's `0xcbbb9d5dc90c2383`) belong to different initialization mechanisms (SeedSequence hash-based initialization vs. direct PCG64 increment assignment) and are not numerically comparable. Statistical independence between this backend and C++ backends or other Python backends is guaranteed by SeedSequence's design property when distinct spawn keys are used.

**PRNG initialization:** The NumPy PCG64 rng is initialized exactly once per solver execution using the SeedSequence formula defined in SPEC-017 FR-9:

`rng = numpy.random.default_rng(numpy.random.PCG64(numpy.random.SeedSequence(execution_seed, spawn_key=(20260620,))))`

The rng is not re-seeded at any point during execution — not at phase boundaries, not between optimizer evaluations, and not on timeout or cancellation.

**Seed authority:** `execution_seed` is the exclusive authorized entropy source for all reproducibility-critical stochastic operations. The rng must be initialized from `execution_seed` only. Prohibited entropy sources in reproducibility-critical paths: `random.random()`, `random.seed()`, `os.urandom()`, `secrets.token_bytes()`, `time.time()`, `os.getpid()`, `numpy.random.default_rng()` without explicit seed, and any hardware entropy source (SPEC-017 FR-9, ADR-010 Decision 4).

**Distribution algorithms (SPEC-017 FR-9):**
- `rng.integers(low, high)` — for bounded integer seed derivation draws (Phase 1 below)
- `rng.uniform(low, high)` — for QAOA parameter initialization draws (Phase 2 below)
- `rng.standard_normal()` — not used in this backend

**PRNG draw ordering — Phase 1: Qiskit sub-system seed derivation**

All Phase 1 draws occur before Phase 2 begins. Phase 1 consumes exactly three draws from the rng, in this fixed order:

| Draw | Name | Formula | Purpose |
|---|---|---|---|
| 1 | `transpiler_seed` | `int(rng.integers(0, 2**31))` | Seed for the Qiskit transpiler's internal randomness. Passed to the Qiskit transpiler's seed parameter. |
| 2 | `simulator_seed` | `int(rng.integers(0, 2**31))` | Seed for the Aer simulator's shot measurement outcomes. Passed to the simulator's shot seed parameter. |
| 3 | `optimizer_seed` | `int(rng.integers(0, 2**31))` | Seed for the classical optimizer's internal randomness, if the optimizer is stochastic. Consumed only if the optimizer requires a seed; still drawn from rng in this position regardless, to maintain draw-ordering stability across optimizer choices. |

These three draws are consumed in Phase 1 regardless of whether the optimizer is deterministic or stochastic. The `optimizer_seed` draw is always consumed at position 3; for deterministic optimizers, the drawn value is generated but not used. This ensures that Phase 2 always begins at the same rng state regardless of optimizer algorithm choice.

**Phase 1 satisfies the SPEC-017 FR-9 Qiskit randomness obligation:** Each Qiskit randomness source (transpiler, simulator/estimator, classical optimizer) is addressed and seeded from a value derived deterministically from `execution_seed`. The specific Qiskit API parameter names used to pass these seed values are implementation planning details.

**PRNG draw ordering — Phase 2: QAOA parameter initialization**

Phase 2 begins immediately after Phase 1's three draws are complete. Phase 2 initializes the 2p QAOA variational parameters (p gamma values and p beta values) in the following fixed order:

For i = 0, 1, ..., p−1:
- Draw γ[i]: `rng.uniform(0.0, 2π)` — gamma parameter for QAOA layer i
- Draw β[i]: `rng.uniform(0.0, π)` — beta parameter for QAOA layer i

Total Phase 2 draws: 2p, consumed in the interleaved order γ[0], β[0], γ[1], β[1], ..., γ[p-1], β[p-1].

The specific range values (0.0 to 2π for γ, 0.0 to π for β) are the observable bounds. The exact formula for the rng call uses `rng.uniform()` per SPEC-017 FR-9.

**Phase 3: Optimization loop (via Qiskit sub-system seeds)**

All randomness consumed during the optimization loop — circuit transpilation, shot-based measurement outcomes, and stochastic optimizer steps — flows through the Qiskit sub-systems initialized from the Phase 1 seeds, not directly from the rng. The main rng is not consumed during Phase 3.

**Aer simulator seed model (part of the reproducibility contract):** The Aer simulator is initialized once per solver execution using `simulator_seed`, before the optimization loop begins. The simulator maintains its internal PRNG state across all optimizer evaluations within a single execution; it is not re-seeded between evaluations. Shot outcomes at evaluation N depend on the PRNG state accumulated through evaluations 1..N−1 in addition to the current circuit parameters. This stateful model ensures that given identical `execution_seed` values, the simulator's state trajectory is identical across invocations, producing identical shot sequences at every evaluation. Changing to a per-evaluation seed reset — passing `simulator_seed` to each individual `run()` call rather than to the simulator constructor — would produce different shot sequences at evaluations 2..N and constitutes a reproducibility-breaking change governed by ADR-010 Decision 5.

This phase boundary means that:
- The rng's total contribution is exactly 3 + 2p draws per solver execution
- Phase 3 randomness is fully determined by `transpiler_seed`, `simulator_seed`, and `optimizer_seed`, which are in turn determined by `execution_seed`
- The reproducibility invariant is satisfied: identical `execution_seed` and identical routing problem produce identical Phase 1 seeds, identical Phase 2 parameters, identical Phase 3 circuit execution outcomes, and therefore identical best-so-far solutions

**Reproducibility invariant (SPEC-004 FR-11.1, SPEC-017 FR-9):** Given two SolverRequests with identical `routing_problem` fields and identical `execution_seed` values, this backend produces identical `outcome` and identical `solution` fields in both responses, when both produce `outcome = "Succeeded"`. The invariant applies when both invocations produce Succeeded. `execution_duration_ms` is excluded from the invariant. Timeout and Cancelled outcomes are timing- and signal-dependent and carry no reproducibility obligation (SPEC-004 FR-11.6).

**Acceptable nondeterminism:**
- `execution_duration_ms` — excluded from the reproducibility invariant in all cases (SPEC-004 FR-11.1)
- `Timeout` and `Cancelled` outcomes — timing- and signal-dependent; no reproducibility obligation
- Shot outcomes on real quantum hardware — not applicable; this backend uses a local simulator only (ADR-007)

The reproducibility invariant applies when comparing two Succeeded responses from this backend with identical SolverRequests. It does not apply across Succeeded and Timeout responses, across different execution_seed values, or across different routing problems.

**Floating-point determinism:** All floating-point computation in reproducibility-critical paths must use IEEE 754 double precision (ADR-010 Decision 6). Extended precision (x87 80-bit) must be disabled in reproducibility-critical code. The specific NumPy and Python environment configuration required is an implementation planning concern.

**Prohibited entropy sources (confirmed absent):**
- `random.random()` or `random.seed()`: prohibited in reproducibility-critical paths
- `os.urandom()`, `secrets.token_bytes()`: prohibited
- `time.time()`, `os.getpid()`: prohibited
- `numpy.random.default_rng()` without explicit seed: prohibited
- `routing_problem.seed` as PRNG entropy: prohibited (SPEC-004 FR-11.2); the problem seed field is received in the SolverRequest JSON per SPEC-017 FR-3 and must not be used as a PRNG entropy source
- Hardware entropy sources: prohibited

**Reproducibility Decision Summary**

The following decisions are frozen once this specification is Accepted. Changing any value constitutes a reproducibility-breaking change governed by ADR-010 Decision 5: update the ADR-010 change record, increment `specification_version`, and update all affected specifications.

| Decision | Value | Notes |
|---|---|---|
| `BACKEND_SPAWN_KEY` | `20260620` | Frozen on Acceptance. Future Python backend specifications must use distinct spawn keys. |
| Phase 1 Draw 1 | `transpiler_seed = int(rng.integers(0, 2**31))` | Always consumed at position 1 before any parameter initialization. |
| Phase 1 Draw 2 | `simulator_seed = int(rng.integers(0, 2**31))` | Always consumed at position 2. |
| Phase 1 Draw 3 | `optimizer_seed = int(rng.integers(0, 2**31))` | Always consumed at position 3 regardless of optimizer type. For deterministic optimizers the drawn value is generated but not applied. |
| Phase 2 Draw order | Interleaved: γ[0], β[0], γ[1], β[1], ..., γ[p−1], β[p−1] via `rng.uniform()` | 2p draws total. Draw ordering is fixed by layer index i, gamma before beta within each layer. |
| Aer simulator seed model | Initialized once with `simulator_seed` before the optimization loop; internal state maintained across all evaluations; not re-seeded between evaluations. | A change to per-evaluation seed reset (passing `simulator_seed` to each `run()` call) changes shot sequences at evaluations 2..N and is a breaking change. |

**Acceptance Criteria:**
- Two SolverRequests with identical routing problems and identical `execution_seed` values produce identical `solution` fields when both produce `outcome = "Succeeded"`
- The rng is initialized exactly once per execution from `execution_seed` with `BACKEND_SPAWN_KEY = 20260620`
- Phase 1 draws (3 total: transpiler_seed, simulator_seed, optimizer_seed) are consumed before Phase 2 begins
- Phase 2 draws (2p total: γ[0], β[0], ..., γ[p-1], β[p-1]) are consumed after Phase 1 completes and before the first optimizer evaluation
- Phase 3 randomness flows through Qiskit sub-system seeds derived in Phase 1; the main rng is not consumed in Phase 3
- The `optimizer_seed` draw at Phase 1 position 3 is always consumed from the rng regardless of whether the optimizer uses it
- No prohibited entropy source is used in reproducibility-critical paths

---

### FR-11: Supported SolverOutcome Values

**Description:**

This backend is classified `quantum_inspired_stochastic`. Per SPEC-011 FR-5.1, it must support `Succeeded`, `Timeout`, `Cancelled`, and `Failed`. It must not support `Infeasible`.

| Outcome | Supported | Trigger Condition |
|---|---|---|
| `Succeeded` | Yes | The optimization loop terminates naturally (optimizer converges or evaluation budget exhausted) before the `execution_timeout_ms` deadline AND at least one feasible RoutePlan was found during the optimization loop |
| `Timeout` | Yes | `execution_timeout_ms` deadline reached before natural optimizer termination; backend self-terminates at an optimizer evaluation boundary; best-so-far RoutePlan included if found before deadline |
| `Cancelled` | Yes | Worker cancellation signal (HTTP connection abort per SPEC-017 FR-6) received during the optimization loop; backend terminates at the next optimizer evaluation boundary; best-so-far RoutePlan included if found before cancellation |
| `Failed` | Yes | Contract version mismatch; natural optimizer termination with no feasible solution found; Qiskit import failure; QUBO construction failure; transpilation failure; internal error preventing execution completion |
| `Infeasible` | **Not Supported** | This backend cannot prove infeasibility. Prohibited per SPEC-011 FR-5.2. |

**Notes on Succeeded:** Succeeded is returned when the optimizer terminates naturally before the deadline. Both conditions are required: natural termination (not deadline-driven) AND existence of a best-so-far solution. If the optimizer converges without ever producing a feasible decoded bitstring, `Failed` is returned instead.

**Notes on Timeout:** Timeout is the expected outcome on Small problems at the upper range of the size class, where circuit depth and optimizer iterations may not complete within a typical `execution_timeout_ms` budget. Unlike the construction heuristics (SPEC-013, SPEC-014), which return Timeout without a solution because they produce no intermediate complete solutions, this backend may return Timeout with a solution when feasible bitstrings were found in at least one optimizer evaluation before the deadline.

**Notes on Cancelled:** Cancellation behavior mirrors Timeout with respect to the best-so-far solution. The backend terminates at the next optimizer evaluation boundary after receiving the cancellation signal via client disconnect (SPEC-017 FR-6). If the cancellation signal arrives before the first optimizer evaluation completes, no solution is present in the response.

**Notes on Failed:** Failed is returned for: contract version mismatch (detected before processing begins); optimizer convergence with no feasible solution ever found; any Qiskit library failure (transpilation error, simulator error, import error); any internal error preventing the optimization loop from producing a valid response. A Failed response from an optimizer that converged without feasible solutions is documented in FR-14. No RoutePlan is present in any Failed response.

This section satisfies the supported outcome declaration requirement in SPEC-011 FR-5.3.

**Acceptance Criteria:**
- The backend never returns `Infeasible`
- Timeout and Cancelled responses include a complete RoutePlan when and only when a feasible solution existed before the termination event
- Succeeded is returned only when the optimizer terminates naturally before the deadline AND a best-so-far solution exists
- Failed with InternalError is returned when natural optimizer termination produces no feasible solution

---

### FR-12: RoutePlan Output Requirements

**Description:**

The QAOA backend produces a RoutePlan conforming to SPEC-004 FR-5. This section declares this backend's specific behavior relative to the framework RoutePlan obligations in SPEC-011 FR-7.1, delivered through the SPEC-017 JSON wire format.

**RoutePlan presence per outcome:**

| Outcome | RoutePlan Present? | Requirements When Present |
|---|---|---|
| `Succeeded` | Required | All SPEC-004 FR-5 conditions 1–5 satisfied |
| `Timeout` | Present if best-so-far exists; absent otherwise | All SPEC-004 FR-5 conditions 1–5 satisfied |
| `Cancelled` | Present if best-so-far exists; absent otherwise | All SPEC-004 FR-5 conditions 1–5 satisfied |
| `Failed` | Absent | Never present |
| `Infeasible` | Not Applicable | Outcome not supported |

**SPEC-011 FR-7.1 narrowing for Timeout and Cancelled:** SPEC-011 FR-7.1 declares the RoutePlan as optional under Timeout and Cancelled for all backends. This backend narrows that permission to "present when best-so-far exists, absent when it does not." This is the same direction as SPEC-015 (FR-11): anytime backends return solutions on Timeout when available. Construction heuristics (SPEC-013, SPEC-014) narrow in the opposite direction (always absent). The framework's Optional permission deliberately accommodates both.

**Capacity validity guarantee:** The returned RoutePlan always satisfies SPEC-004 FR-5 condition 5 (total stop demand per route ≤ `capacity_per_vehicle`) when present. Decoded bitstrings that violate capacity are excluded from best-so-far candidacy (FR-6). No post-return capacity re-validation is needed.

**Stop assignment completeness guarantee:** A RoutePlan present in any SolverResponse assigns every stop in the routing problem to exactly one route (SPEC-004 FR-5 conditions 3–4). Partial assignments are never valid best-so-far candidates (FR-6). If all optimizer evaluations before termination produced only partially-assigned decoded states, no solution is returned.

**Vehicle count guarantee:** When a RoutePlan is present, it contains exactly `vehicle_count` route entries. Unused vehicles have empty route sequences. The depot is not included in any route (SPEC-004 FR-5).

**Time window non-guarantee:** A RoutePlan present in any SolverResponse does not guarantee time window feasibility. This backend declares `supports_time_windows = false` (FR-9). Time window feasibility is evaluated by Core Quality Evaluation (SPEC-007) after the response is received.

`execution_duration_ms` is always populated in every SolverResponse. It reflects wall-clock backend execution time from the moment Python solver computation begins — defined as after JSON deserialization, backend routing, contract version validation, and rng initialization — to the moment the adapter begins constructing the SolverResponse, per SPEC-017 FR-5. This includes Phase 1 seed derivation, Phase 2 parameter initialization, transpilation time, and all optimizer evaluation time.

**Acceptance Criteria:**
- Every RoutePlan present in any SolverResponse satisfies all SPEC-004 FR-5 conditions 1–5
- No SolverResponse contains a RoutePlan with unassigned stops
- `execution_duration_ms` is present in every SolverResponse
- No RoutePlan is present under Failed

---

### FR-13: Extension Metadata

**Description:**

The QAOA backend produces the following `extension_metadata` keys in the SolverResponse when a solution is present (`Succeeded`, or `Timeout`/`Cancelled` with best-so-far). When no solution is present (`Failed`, or `Timeout`/`Cancelled` without solution), extension_metadata is absent. All values are encoded as non-empty strings.

| Key | Type (encoded as string) | Present When | Description |
|---|---|---|---|
| `qaoa.ansatz_depth` | uint32 | Solution present | The QAOA circuit depth p used in this execution. The number of cost-mixer layer pairs in the parameterized ansatz. |
| `qaoa.shots_per_evaluation` | uint32 | Solution present | The number of circuit measurement shots per classical optimizer function evaluation. Determines the size of the bitstring pool decoded at each evaluation step. |
| `qaoa.optimizer_evaluations` | uint32 | Solution present | The number of classical optimizer function evaluations (circuit executions) that completed before the boundary at which termination was detected — whether natural, timeout, or cancellation. Timeout and cancellation are detected only at evaluation boundaries; no partial evaluations are counted. |
| `qaoa.best_bitstring_energy` | float64 | Solution present | The QUBO energy of the bitstring corresponding to the returned best-so-far RoutePlan. Computed from the QUBO objective function (implementation planning decision) at the bitstring that produced the returned RoutePlan. Encoded as a decimal string in IEEE 754 double precision. Lower energy indicates a better QUBO solution within this execution. This value is meaningful only within a single QUBO formulation and implementation; it is not comparable across different encoding schemes or penalty weight configurations. |
| `qaoa.feasible_bitstrings_found` | uint32 | Solution present | The total count of bitstrings across all optimizer evaluations that decoded to a RoutePlan satisfying all SPEC-004 FR-5 structural validity requirements. Includes all valid bitstrings, not only those that became the best-so-far. A value of 1 means exactly one feasible bitstring was found across the entire optimization run. |
| `qaoa.solution_route_distance_km` | float64 | Solution present | Total Haversine route distance in kilometers of the returned best-so-far RoutePlan. Computed as the sum of Haversine distances for all vehicle routes, including each vehicle's depot-to-first-stop leg and last-stop-to-depot leg. This is the solver's internal quality metric; the normalized quality metric used by Core (SPEC-007) is computed independently. Encoded as a decimal string in IEEE 754 double precision. |

**Presence condition:** Extension metadata is present when and only when a solution is present. The same condition that governs RoutePlan presence (FR-12) governs extension metadata presence. All six keys are always present when a solution is present; no key is conditionally omitted.

**Rationale for producing extension metadata on Timeout and Cancelled with solution:** Like SPEC-015, this backend extends extension metadata presence to Timeout-with-solution and Cancelled-with-solution responses. For QAOA, the diagnostic value of knowing how many optimizer evaluations completed, what circuit depth was used, and what the best bitstring's energy was is significant on Timeout paths, which are the expected outcome for larger Small problems. Restricting extension metadata to Succeeded would suppress QAOA-specific diagnostic evidence on the outcome path most relevant to evidence analysis.

`extension_metadata` must not contain routing problem raw data: geographic coordinate arrays, stop identifier lists, demand arrays, time window arrays, or other raw problem input fields. The six keys above contain only derived summary scalar values safe for evidence log inclusion (SPEC-004 FR-13, SPEC-001 Security Considerations).

Consumers must silently ignore any keys not listed here. Unrecognized keys are not contract-stable across `specification_version` changes.

**Acceptance Criteria:**
- All six keys are present in the `extension_metadata` of every SolverResponse where a solution is present
- All six keys are absent when no solution is present
- Key values are non-empty strings encoding valid numeric values of the specified types
- No key contains routing problem raw data (coordinates, demands, stop lists, time windows)
- `qaoa.feasible_bitstrings_found` is a non-negative integer not exceeding `qaoa.optimizer_evaluations × qaoa.shots_per_evaluation`

---

### FR-14: Failure Model

**Description:**

This section defines all failure conditions specific to the QAOA backend and its obligations for the SPEC-017 OQ-3B interrupt compliance bound. General framework failure obligations are defined in SPEC-011 FR-8.

**Contract version mismatch:**
- **Trigger:** `contract_version` in the SolverRequest JSON does not equal the backend's declared `supported_contract_version` (1).
- **Outcome:** `Failed`, `failure_code = "ContractVersionMismatch"`
- **Behavior:** The adapter performs this check before invoking the Python solver function (SPEC-017 FR-10). No Python backend code executes. No rng initialization occurs. No QUBO construction occurs. No Qiskit import is triggered by this path. The adapter returns the ContractVersionMismatch response directly.
- **Metadata:** `failure_message` must identify the received version and the expected version (1).

**Qiskit import failure:**
- **Trigger:** A required Qiskit module (qiskit, qiskit-aer, or a required qiskit submodule) fails to import during backend initialization.
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** The backend returns Failed immediately. This condition should be caught at adapter startup (SPEC-017 FR-8 module import validation) before any request is dispatched; if it occurs at request time, the adapter's request-time import failure handling applies (SPEC-017 FR-8).
- **Metadata:** `failure_message` identifies the failing module and the import error.

**QUBO construction failure:**
- **Trigger:** An error occurs during QUBO formulation of the routing problem (e.g., arithmetic overflow in penalty weight computation, invalid problem parameters).
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** The backend returns Failed immediately, before Phase 1 rng initialization. No solution is present. Extension metadata is absent.
- **Metadata:** `failure_message` identifies the failure cause.

**QUBO dimension exceeds simulator tractability limit:**
- **Trigger:** After QUBO formulation, the computed binary variable count Q exceeds the maximum tractable qubit count for the local Aer simulator (empirically determined at implementation time per OQ-1 and OQ-6).
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** The backend returns Failed immediately after QUBO construction determines Q, before Phase 1 rng initialization and before any QAOA circuit is constructed or submitted to the simulator. No stochastic computation occurs. No solution is present. Extension metadata is absent.
- **Metadata:** `failure_message` must identify the computed Q value, the maximum supported Q value, the stop count, and the vehicle count. Example: `"qaoa-qiskit: QUBO dimension Q=52 exceeds maximum supported qubit count Q_max=30 for stop_count=18 vehicle_count=4."` This condition indicates a capability boundary, not a solver implementation fault.

**Transpilation failure:**
- **Trigger:** The Qiskit transpiler raises an error when compiling the QAOA circuit.
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** The backend returns Failed. No solution is present. Extension metadata is absent.
- **Metadata:** `failure_message` identifies the transpilation error.

**Natural optimizer termination with no feasible solution:**
- **Trigger:** The classical optimizer converges or the evaluation budget is exhausted before the `execution_timeout_ms` deadline, AND no bitstring across all optimizer evaluations decoded to a RoutePlan satisfying all SPEC-004 FR-5 structural validity requirements.
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** The backend returns Failed. No RoutePlan is present. Extension metadata is absent.
- **Metadata:** `failure_message` must identify the failure cause ("qaoa-qiskit: optimization completed without producing a feasible route plan") and the relevant problem characteristics (stop count, vehicle count, ansatz depth, evaluations completed).
- **Note:** This condition is expected to occur on some problem instances, particularly those where the QUBO penalty structure does not guide the search toward feasible assignments within the evaluation budget. It should trigger investigation of the QUBO formulation, penalty weights, and ansatz depth.

**Internal error:**
- **Trigger:** Any unexpected condition preventing algorithm execution or RoutePlan production (e.g., Qiskit simulator exception, NaN or infinite QUBO energy values, NumPy arithmetic errors, memory allocation failure).
- **Outcome:** `Failed`, `failure_code = "InternalError"`
- **Behavior:** The backend returns Failed immediately. No RoutePlan is present. Extension metadata is absent.
- **Metadata:** `failure_message` provides a diagnostic description identifying the failure cause, exception type, and traceback summary.

**Timeout:**
- **Trigger:** `execution_timeout_ms` deadline reached during an optimizer evaluation.
- **Outcome:** `Timeout`, `failure_code = "ExecutionTimeout"`
- **Behavior:** The backend MUST monitor the deadline at optimizer evaluation boundaries. When the remaining time before the deadline falls below a threshold sufficient to finalize the response (including response serialization and return), the backend terminates the current evaluation and begins response construction. The specific threshold is an implementation planning decision. On timeout, the backend returns the best-so-far solution if found; absent otherwise. Extension metadata is present if a solution is present.
- **Self-termination obligation:** Self-termination is a stronger obligation for this backend than the advisory in SPEC-004 FR-10.1. If the Worker's HTTP client timeout fires because the adapter did not self-terminate, the Worker-constructed Timeout response contains no solution and no extension metadata, discarding the best-so-far and all QAOA diagnostic evidence. Self-termination before the HTTP client timeout fires is a behavioral obligation.
- **Monitoring mechanism:** The backend checks elapsed time at each optimizer evaluation boundary — after one evaluation completes and before the next begins. Time checks within a single circuit execution are not required and not defined. If the deadline is detected at a boundary, the backend does not begin the next evaluation; the evaluation that just completed is the last counted evaluation in `qaoa.optimizer_evaluations`. Termination is never declared mid-evaluation.

**Cancellation:**
- **Trigger:** Worker cancellation signal received via HTTP connection abort (SPEC-017 FR-6) during an optimizer evaluation.
- **Outcome:** `Cancelled`, `failure_code = "ExecutionCancelled"`
- **Behavior:** The adapter detects client disconnect and signals the Python backend to stop (SPEC-017 FR-6). The backend terminates at the next optimizer evaluation boundary after receiving the stop signal. If the signal arrives during a Qiskit circuit execution, the backend allows that execution to complete and terminates before beginning the next evaluation. The best-so-far solution is returned if found; absent otherwise. Extension metadata is present if a solution is present.

**Interrupt compliance bound (SPEC-017 OQ-3B):** The maximum time from the adapter's stop signal to the backend halting computation is bounded by the duration of one Qiskit circuit evaluation (transpilation if per-evaluation, plus shot simulation). For Small problems at the latency profile declared in FR-9, one circuit evaluation takes at most as long as the full `execution_timeout_ms` budget in the worst case. If circuit evaluation time significantly exceeds the `execution_timeout_ms` budget, the backend's interrupt compliance bound is too coarse to be useful; this situation indicates that the implementation planning choices for ansatz depth and shots are miscalibrated for the given problem size. The acceptable single-evaluation time bound must be validated during implementation and reported in the `is_provisional = false` qualification review.

All `Failed` responses include a `SolverFailureDetail` with both `failure_code` and `failure_message` populated. `failure_message` is diagnostic (developer-readable), not user-facing. The backend must not write to stdout or stderr (SPEC-017 Constraint 14); all diagnostic output is returned through `SolverFailureDetail.failure_message` and `extension_metadata`.

**Acceptance Criteria:**
- `ContractVersionMismatch` is returned before any processing begins (including rng initialization) on version mismatch
- `InternalError` with the prescribed message pattern is returned when natural optimizer termination produces no feasible solution
- `failure_message` is non-empty on every Failed response
- No RoutePlan is present in any Failed response
- Timeout self-termination is monitored at optimizer evaluation boundaries
- The interrupt compliance bound (one circuit evaluation time) is documented during implementation planning

---

# Performance Characteristics

The QAOA backend's execution cost is dominated by circuit simulation on the local Aer simulator, which scales exponentially with qubit count. All estimates are derived from algorithmic analysis; empirical measurement is required before `is_provisional` is set to false.

**Phase 1 (seed derivation):** Three `rng.integers()` draws. Negligible cost.

**Phase 2 (parameter initialization):** 2p `rng.uniform()` draws. Negligible cost.

**Transpilation:** One-time (or per-evaluation) circuit compilation using the seeded Qiskit transpiler. Cost depends on circuit depth p and qubit count Q. For shallow circuits (p ≤ 3) at small qubit counts (Q ≤ 20), transpilation is expected to take less than a few seconds. For deeper circuits or larger Q, transpilation cost must be measured.

**Per-evaluation cost:** Shot-based state-vector simulation at Q qubits takes O(shots × 2^Q) compute. For Q = 10 qubits and 1000 shots, this is roughly 10^6 amplitude evaluations per evaluation. For Q = 20 qubits, this is roughly 10^9. The tractable Q range for the latency profile declared in FR-9 (< 300.0s for Small) must be empirically determined.

**Classical optimizer overhead:** Per-evaluation gradient or function evaluation by the classical optimizer adds overhead proportional to the optimizer algorithm's complexity. For derivative-free methods (COBYLA), this is O(1) per evaluation. For finite-difference gradient methods, this may multiply the effective evaluation count.

**Expected execution duration per size class:**

| Size Class | Stop Range | Expected Duration | Basis |
|---|---|---|---|
| Small (lower range) | 1–10 stops | < 60.0s | Algorithmic analysis; small QUBO dimension; tractable qubit count |
| Small (upper range) | 11–25 stops | 60.0s–300.0s | Algorithmic analysis; QUBO dimension may approach simulator tractability limit |

These estimates assume shallow ansatz depth (p ≤ 3) and modest shot counts (100–1000 shots per evaluation). Empirical measurement on SPEC-002 synthetic workload problems is required before `is_provisional` is set to false.

**Memory:** The dominant memory allocation is the Aer simulator state vector (2^Q complex amplitudes). For Q = 20 qubits, this is approximately 16 MB per state-vector copy. For Q = 30 qubits, approximately 16 GB — well beyond MVP constraints. The implementation must ensure Q remains within simulator-tractable bounds; if Q exceeds the tractable limit for a given problem, the backend should return Failed with InternalError before attempting simulation.

---

# Quality Expectations

**Route quality expectations:** The QAOA backend at shallow ansatz depth and typical shot counts is expected to produce Baseline-quality routes. QAOA's approximation ratio improves with circuit depth, but local simulator tractability limits achievable depth. For the Small size class at MVP scope, the evidence system is expected to demonstrate that QAOA on a local simulator does not outperform the classical construction heuristics in route quality while incurring significantly higher latency.

**Workload classes where QAOA performance may improve:**
- Problems at the lower range of the Small size class (< 10 stops) where the QUBO dimension is small enough to allow deeper circuits and more optimizer evaluations within the latency budget
- Problems with high capacity utilization where the QUBO penalty structure effectively guides the optimizer toward feasible assignments

**Workload classes where QAOA is expected to underperform:**
- Small problems near the upper range (20–25 stops) where QUBO dimension requires shallow circuits and few optimizer evaluations
- Any problem where the QUBO penalty structure does not consistently guide shots toward feasible assignments (resulting in low `qaoa.feasible_bitstrings_found` counts)

**Evidence thesis implications:** The Scheduler's decision to reject the QAOA backend in favor of classical heuristics for most routing workloads is supported by evidence showing that QAOA incurs dramatically higher latency and lower solution quality relative to the classical backends. The `qaoa.feasible_bitstrings_found` count and `qaoa.optimizer_evaluations` values in the evidence report quantify the QAOA execution effort; comparing these against the route quality achieved (SPEC-007 normalized metric) supports the thesis that quantum-adjacent execution cost is rarely justified.

---

# Non-Requirements

- This specification does not define the QUBO energy function, penalty weights, or binary variable encoding. Those are implementation planning decisions (FR-2).
- This specification does not define the QAOA circuit gate sequence or the specific Qiskit primitives used (Sampler, Estimator, etc.). Those are implementation planning decisions (FR-2).
- This specification does not define the classical optimizer algorithm (COBYLA, SPSA, L-BFGS-B, etc.) or its convergence parameters. Those are implementation planning decisions (FR-2, FR-5).
- This specification does not define the ansatz depth p or the shot count per evaluation beyond the minimum constraint (p ≥ 1, shots_per_evaluation ≥ 1). Those are implementation planning decisions (FR-3, FR-4).
- This specification does not define the transpilation strategy (once vs. per-evaluation), optimization level, or specific transpiler backend. Those are implementation planning decisions (FR-2).
- This specification does not define the QUBO decoding algorithm (how a bitstring maps to a RoutePlan) or repair heuristics for infeasible decoded states. Those are implementation planning decisions (FR-2, FR-6).
- This specification does not define time window feasibility checking. Time windows are evaluated by Core Quality Evaluation (SPEC-007).
- This specification does not define IBM Quantum hardware execution. Hardware execution is deferred per ADR-007.
- This specification does not define multi-depot behavior. Single depot only, per SPEC-001 FR-3.
- This specification does not define heterogeneous vehicle fleet handling. All vehicles have identical capacity, per SPEC-001 FR-2.
- This specification does not define workload feature computation. That is SPEC-010's responsibility.
- This specification does not define backend selection policy. That is SPEC-003's responsibility.
- This specification does not define evidence persistence. That is SPEC-006's responsibility.
- This specification does not define quality evaluation metrics. That is SPEC-007's responsibility.
- This specification does not define the Python adapter HTTP transport. That is SPEC-017's responsibility.
- This specification does not define the Python adapter environment management, health checking, or startup behavior. That is SPEC-017's responsibility.
- This specification does not define any subsequent Python backend specification.

---

# Assumptions

1. The routing problem received by this backend has been validated by Core per ADR-009 and delivered by the Worker through the SPEC-017 adapter per SPEC-017 FR-3. The backend does not re-validate the routing problem's structural constraints.

2. The QUBO formulation can encode the CVRP capacity constraints as penalty terms such that some bitstrings in the shot pool, after sufficient optimizer evaluations, correspond to capacity-valid route assignments. If the penalty weighting is incorrect, the optimizer may converge to parameters that never sample feasible bitstrings. This is a formulation quality risk addressed during implementation planning.

3. The local Aer simulator produces deterministic, reproducible shot outcomes given a fixed circuit, fixed parameters, and a fixed measurement seed (derived from `execution_seed` per FR-10). This assumption holds for the `statevector` and `qasm_simulator` simulation methods in Qiskit Aer; it must be confirmed for the specific simulator mode selected during implementation planning.

4. Anytime behavior is achievable: the best-so-far RoutePlan, once found, can be returned on timeout without additional computation beyond response finalization. Phase 1 seed derivation and Phase 2 parameter initialization are fast (< 1ms) relative to circuit evaluation time and do not materially affect the `execution_timeout_ms` budget available for optimization.

5. The BACKEND_SPAWN_KEY value `20260620` produces a PCG64 initialization that is statistically independent from all other NumPy PCG64 instances in the project and from the C++ PCG64 instances used by SPEC-015 (stream constant `0xcbbb9d5dc90c2383`) and SPEC-002. Statistical independence between Python backends is guaranteed by SeedSequence's design for distinct spawn keys; independence from C++ instances is guaranteed by the different initialization mechanisms (SeedSequence vs. direct PCG64 increment).

6. Qiskit and Qiskit Aer are available as pip-installable dependencies within the Python adapter container image (SPEC-017 FR-8) without licensing conflicts. The specific version constraints are implementation planning decisions.

7. All Python backend execution occurs within the `python-adapter` Docker Compose container per SPEC-017. The adapter is running and healthy per SPEC-017 FR-14 before the Worker dispatches any SolverRequest to this backend.

8. The Worker's `transport_overhead_buffer_ms` configuration (SPEC-017 OQ-1) is calibrated to account for the JSON serialization time of routing problem payloads at Small problem scale. The QAOA backend's self-termination before the HTTP client timeout fires is achievable if termination monitoring occurs at optimizer evaluation boundaries (FR-14).

---

# Constraints

1. This backend implements SPEC-004 contract version 1. `contract_version` mismatch must be detected before any processing begins (SPEC-011 FR-8.3, SPEC-017 FR-10).

2. NumPy PCG64 initialized via `SeedSequence(execution_seed, spawn_key=(20260620,))` is the exclusive PRNG for reproducibility-critical stochastic operations. No other PRNG is permitted in reproducibility-critical paths (ADR-010 Decision 1, SPEC-017 FR-9).

3. `execution_seed` is the exclusive authorized entropy source. Prohibited: `random.random()`, `os.urandom()`, `time.time()`, `os.getpid()`, any unseeded NumPy generator, hardware entropy (SPEC-017 FR-9, ADR-010 Decision 4).

4. The rng must be initialized exactly once per execution. Reseeding during execution is prohibited (ADR-010 Decision 4).

5. BACKEND_SPAWN_KEY `20260620` is frozen once this specification is Accepted. Changing it requires the full ADR-010 Decision 5 breaking change procedure. Future Python backend specifications must use distinct spawn keys.

6. `routing_problem.seed` must not be used as a PRNG entropy source. Only `execution_seed` seeds the rng (SPEC-004 FR-11.2, SPEC-017 FR-9).

7. All Qiskit randomness sources (transpiler, simulator/estimator, classical optimizer) must be seeded from values derived deterministically from `execution_seed` via Phase 1 draws (FR-10).

8. The backend must not make external network calls during solver execution. IBM Quantum hardware execution is deferred per ADR-007. No cloud API is accessed.

9. The backend must not emit OpenTelemetry spans, metrics, or structured logs (SPEC-011 FR-9.2). Diagnostic output is returned exclusively through `SolverFailureDetail.failure_message` and `extension_metadata`.

10. The backend must not write to stdout or stderr (SPEC-017 Constraint 14). The Python adapter's stdout channel is reserved for the adapter's structured JSON log events.

11. The backend must not return `Infeasible` under any condition (SPEC-011 FR-5.2).

12. Any RoutePlan included in a SolverResponse must satisfy all SPEC-004 FR-5 structural validity requirements. Partial assignments and capacity violations are never valid.

13. No backend-specific logic may exist in the Scheduler, Worker, or Core contract-handling code. This backend is accessed exclusively through the normalized SolverContract interface via the SPEC-017 adapter (ADR-008).

14. `extension_metadata` must not contain routing problem raw data: geographic coordinate arrays, stop identifier lists, demand arrays, time window arrays (SPEC-004 FR-13, SPEC-011 FR-7.4).

---

# Inputs

**SolverRequest** (SPEC-004 FR-2, SPEC-017 FR-3): The complete input to this backend, received by the adapter as JSON and deserialized before the Python solver function is invoked. Relevant fields:

| Field | How Used |
|---|---|
| `routing_problem.stops` | Stop coordinates, demands, stop_id values — all used in QUBO formulation and bitstring decoding |
| `routing_problem.depot` | Depot coordinates — used in route distance computation for best-so-far quality comparison |
| `routing_problem.vehicle_count` | Determines number of routes in the decoded RoutePlan; part of QUBO binary state dimension |
| `routing_problem.capacity_per_vehicle` | Enforced as a constraint via QUBO penalty encoding; decoded solutions must satisfy SPEC-004 FR-5 condition 5 |
| `execution_timeout_ms` | Deadline for execution; monitored at optimizer evaluation boundaries for self-termination |
| `execution_seed` | Received as a decimal string in JSON; parsed to Python integer; used to initialize the NumPy PCG64 rng with BACKEND_SPAWN_KEY = 20260620 |
| `contract_version` | Validated before any processing; must equal 1 (checked by adapter per SPEC-017 FR-10 before Python solver is invoked) |
| `job_id`, `decision_id`, `backend_id` | Correlation only; do not affect algorithm execution or route construction |
| `routing_problem.seed` | Present in JSON per SPEC-017 FR-3; must not be used as PRNG entropy (SPEC-004 FR-11.2) |

Fields not used in optimization: `routing_problem.time_window_open`, `routing_problem.time_window_close`, `routing_problem.service_duration_seconds` (see FR-8).

---

# Outputs

**SolverResponse** (SPEC-004 FR-3, SPEC-017 FR-4): The complete output, serialized to JSON by the adapter and returned as HTTP 200.

On `Succeeded`:
- `outcome = "Succeeded"`
- `solution`: Best-so-far RoutePlan with `vehicle_count` routes, all stops assigned, capacity valid
- `statistics.execution_duration_ms`: Wall-clock backend execution time in milliseconds (per SPEC-017 FR-5)
- `statistics.solution_count`: Number of optimizer evaluations in which the best-so-far improved
- `extension_metadata`: All six keys (FR-13)
- `failure`: Absent

On `Timeout` with best-so-far:
- `outcome = "Timeout"`
- `failure.failure_code = "ExecutionTimeout"`; `failure.failure_message`: Diagnostic
- `solution`: Best-so-far RoutePlan satisfying all SPEC-004 FR-5 requirements
- `statistics.execution_duration_ms`: Wall-clock time from execution start to self-termination
- `statistics.solution_count`: Number of best-so-far updates before self-termination
- `extension_metadata`: All six keys (FR-13)

On `Timeout` without best-so-far:
- `outcome = "Timeout"`
- `failure.failure_code = "ExecutionTimeout"`; `failure.failure_message`: Diagnostic
- `solution`: Absent
- `extension_metadata`: Absent

On `Cancelled` (mirrors Timeout structure; solution and extension_metadata present if best-so-far exists):
- `outcome = "Cancelled"`
- `failure.failure_code = "ExecutionCancelled"`

On `Failed`:
- `outcome = "Failed"`
- `failure.failure_code`: `"ContractVersionMismatch"` or `"InternalError"`
- `failure.failure_message`: Diagnostic description
- `solution`: Absent
- `extension_metadata`: Absent

Consumer: The SPEC-017 adapter receives the Python solver's output, serializes it to JSON, and returns it to the Worker as HTTP 200. The Worker validates structural requirements (SPEC-004 FR-5), emits the `solver.execute` span (SPEC-004 FR-15), and persists the result to the evidence log. Core evaluates route quality from the RoutePlan (SPEC-007). The `extension_metadata` passes through to the evidence record.

---

# Architectural Impact

| Component | Impact | Notes |
|---|---|---|
| Python Solver Adapter (SPEC-017) | Yes | First Python backend execution; validates the SPEC-017 adapter end-to-end on a Qiskit workload |
| Scheduler (SPEC-003) | Yes | Capability profile for `qaoa-qiskit` must be registered before this backend is eligible for selection |
| Solver Contract (SPEC-004) | None | SPEC-018 satisfies SPEC-004 FR-1 obligations through the SPEC-017 adapter; no schema changes |
| Worker (SPEC-005) | None | Standard Python adapter dispatch path per SPEC-017; no Worker changes required |
| Evidence Log (SPEC-006) | None | `extension_metadata` from this backend passes through the existing JSONB column; no schema changes |
| Quality Evaluation (SPEC-007) | None | Core evaluates quality from the normalized RoutePlan; Timeout-with-solution responses evaluated same as Succeeded |
| Feature Extraction (SPEC-010) | None | Workload features computed before backend selection; no changes |
| Persistence Schema (SPEC-012) | None | No new tables; `extension_metadata` stored per existing schema |
| Reproducibility Pipeline (ADR-010) | Yes | First Qiskit-based backend; validates the SPEC-017 FR-9 Python PRNG policy (SeedSequence spawn_key mechanism) and Qiskit seed propagation obligations end-to-end |
| API Layer | None | No new API surface |
| Observability | None | Worker-emitted `solver.execute` span (SPEC-004 FR-15); span status `Unset` for Timeout-with-solution per SPEC-004 FR-15 span status rules |
| Security | None | No new trust boundaries; `extension_metadata` raw-data prohibition applies per FR-13 |
| Configuration | Yes | Capability profile must be registered through the mechanism resolved by SPEC-003 OQ-2 |
| Deployment | Yes | Qiskit and Qiskit Aer must be added to the python-adapter container image's requirements file (SPEC-017 FR-8) |

**SPEC-011 FR-11:** The SPEC-011 MVP backend inventory does not list this backend. SPEC-011 FR-11.2 identifies "Python Adapter" as a deferred backend with note "Individual Python backend specifications (SPEC-018+) may now be written." SPEC-018 satisfies the stated condition. SPEC-011 FR-11 should be updated to list `qaoa-qiskit` as a Python adapter backend with this specification's status once SPEC-018 is Accepted.

**ADR-007:** ADR-007 is Proposed status and explicitly identifies Qiskit Aer local simulation as a permissible future extension. This specification is consistent with ADR-007: no cloud quantum hardware is used, no IBM Quantum Runtime dependency is introduced, and all execution is local. ADR-007 should be advanced to Accepted following this specification's review.

---

# Testability

The following behaviors must be verified. Specific test implementations are determined during implementation planning.

**Contract conformance (inherited from SPEC-004 via SPEC-017):**

1. **Structural validity:** A Succeeded SolverResponse contains a RoutePlan where: (a) `routes.size()` = `vehicle_count`; (b) every stop_id appears exactly once across all routes; (c) no route's total demand exceeds `capacity_per_vehicle`; (d) no route contains the depot.

2. **Reproducibility:** Two SolverRequests with identical routing problems and identical `execution_seed` values produce identical `outcome` and `solution` fields when both produce `Succeeded`. This validates that all QAOA randomness (parameters, transpiler, shot outcomes) is fully determined by `execution_seed`.

3. **Seed sensitivity:** Two SolverRequests with identical routing problems and different `execution_seed` values produce different `solution` fields (with high probability). This validates that `execution_seed` is actually consumed by the rng, not ignored.

4. **Seed acceptance:** The backend accepts any `execution_seed` value without error (within uint64 range).

5. **ContractVersionMismatch:** A SolverRequest with `contract_version` != 1 returns `Failed` with `ContractVersionMismatch` before any processing, before rng initialization. Verified by the adapter per SPEC-017 FR-10.

6. **Trivial case:** A routing problem with one stop and one vehicle produces a Succeeded response with a single route containing that stop (assuming sufficient optimizer evaluations and shots to find the trivially feasible single-stop assignment).

7. **Empty vehicle routes:** A routing problem with more vehicles than stops produces a Succeeded response where some routes are empty.

**Failure model:**

8. **Natural termination with no feasible solution:** Given an execution configured so the optimizer evaluates its full budget without any bitstring decoding to a valid RoutePlan, the backend returns `Failed` with `failure_code = "InternalError"` and `failure_message` containing "qaoa-qiskit: optimization completed without producing a feasible route plan."

9. **QUBO construction error injection:** Given an artificially malformed routing problem that causes QUBO formulation to fail, the backend returns `Failed` with `failure_code = "InternalError"` before rng initialization.

**Stochastic algorithm correctness:**

10. **Anytime behavior — Timeout with solution:** Given a SolverRequest with `execution_timeout_ms` set to allow at least one optimizer evaluation to complete (producing a feasible bitstring) but not the full optimization schedule, the backend returns `Timeout` with a complete, structurally valid RoutePlan.

11. **Anytime behavior — no solution before deadline:** Given a SolverRequest with `execution_timeout_ms` set so short that no optimizer evaluation completes before the deadline, the backend returns `Timeout` without a solution.

12. **Optimizer evaluations count:** `qaoa.optimizer_evaluations` equals the number of circuit evaluations completed before termination. Verifiable by instrumenting evaluation count during implementation testing.

13. **Feasible bitstrings count:** `qaoa.feasible_bitstrings_found` is non-negative and does not exceed `qaoa.optimizer_evaluations × qaoa.shots_per_evaluation`.

14. **No partial assignments:** No Succeeded, Timeout, or Cancelled response contains a RoutePlan with unassigned stops. Every stop in the routing problem appears in exactly one route in every RoutePlan present.

15. **Capacity validity across workload:** For all SolverResponses with a RoutePlan across the SPEC-002 synthetic Small-class workload, no route has total demand exceeding `capacity_per_vehicle`.

**Reproducibility pipeline:**

16. **Phase 1 seed derivation:** Two executions with the same `execution_seed` produce the same `transpiler_seed`, `simulator_seed`, and `optimizer_seed` values. Verifiable by logging derived seeds during testing.

17. **Phase 2 parameter initialization:** Two executions with the same `execution_seed` and same ansatz depth p produce the same initial γ and β parameter vectors.

18. **Qiskit randomness isolation:** Changing `execution_seed` while holding the routing problem constant changes the initial parameters (Phase 2) and the shot outcomes (Phase 3), producing different `solution` fields and `qaoa.best_bitstring_energy` values.

19. **Prohibited entropy source absence:** Code review confirms that `random.random()`, `os.urandom()`, `time.time()`, and other prohibited sources are absent from all reproducibility-critical paths.

**Capability profile accuracy:**

20. **Latency profile validation:** Empirical execution duration on representative SPEC-002 Small-class problems is within the declared `latency_profile` bound (< 300.0s). Measured as `execution_duration_ms`. Required before `is_provisional` is set to false.

21. **Quality classification:** Route quality (Core normalized metric, SPEC-007) on SPEC-002 benchmark problems is consistent with `Baseline` classification relative to the classical heuristic baselines. Required before `is_provisional` is set to false.

**Extension metadata:**

22. **Key presence on solution:** All six `extension_metadata` keys are present when a solution is present; extension_metadata is absent when no solution is present.

23. **qaoa.solution_route_distance_km accuracy:** The reported distance matches the sum of Haversine distances computed from the route sequences in the RoutePlan, including depot-return legs.

24. **qaoa.ansatz_depth and qaoa.shots_per_evaluation consistency:** The ansatz depth declared in extension_metadata matches the QAOA circuit depth used; the shot count declared matches the actual shots executed per evaluation.

**Observability:**

25. **solver.execute span:** A `solver.execute` OTel span is emitted by the Worker for every SolverRequest dispatch through the SPEC-017 adapter, successful or not. All required attributes (SPEC-004 FR-15) are present. Span status is `Unset` for Timeout-with-solution (not Error), `OK` for Succeeded, `Error` for Failed. Verified in integration test against the SPEC-017 adapter.

**Adapter integration:**

26. **Self-termination before HTTP client timeout:** The backend returns its Timeout response before the Worker's HTTP client timeout fires (SPEC-017 FR-5). Confirmed by verifying that `execution_duration_ms` in the response is less than `execution_timeout_ms + transport_overhead_buffer_ms` minus a margin sufficient for response serialization.

27. **Client disconnect handling:** The adapter handles HTTP connection abort during a QAOA optimizer evaluation without crashing. The adapter returns to ready state and accepts a subsequent request normally.

---

# Observability Requirements

The `solver.execute` span is emitted by the Worker for every invocation of this backend through the SPEC-017 adapter (SPEC-004 FR-15). The backend's contribution is through SolverResponse fields:

| Span Attribute | Source |
|---|---|
| `outcome` | `SolverResponse.outcome` (JSON string) |
| `execution_duration_ms` | `SolverResponse.statistics.execution_duration_ms` |

All other `solver.execute` span attributes are available to the Worker from the SolverRequest, routing problem, or workload features.

**Diagnostic questions this backend must enable through `extension_metadata`:**

1. How many optimizer evaluations completed before termination? (`qaoa.optimizer_evaluations`) — determines whether the optimizer had sufficient budget to converge
2. What circuit depth was used? (`qaoa.ansatz_depth`) — correlates with expected approximation quality
3. How many shots were taken per evaluation? (`qaoa.shots_per_evaluation`) — affects the probability of sampling feasible bitstrings
4. How many total feasible bitstrings were found? (`qaoa.feasible_bitstrings_found`) — indicates whether the QUBO formulation effectively guided the optimizer toward feasible solutions
5. What was the QUBO energy of the returned solution? (`qaoa.best_bitstring_energy`) — the solver's internal quality measure
6. What was the Haversine route distance of the returned solution? (`qaoa.solution_route_distance_km`) — the solver's route quality estimate

These values pass through the Worker to the evidence log, where they are available for QAOA-specific evidence reporting. The SPEC-009 evidence report's Solver Execution section may surface these values for the QAOA evidence narrative.

**Adapter observability (SPEC-017 FR-13):** The SPEC-017 adapter emits structured JSON log events for every request lifecycle event: `adapter.request.received`, `adapter.solver.invoked`, `adapter.solver.completed` or `adapter.solver.timeout`, `adapter.client.disconnect` (if applicable), and `adapter.error` (if applicable). These events are correlated with the `solver.execute` span via `job_id` and `decision_id`.

**No additional backend instrumentation:** The backend does not emit OpenTelemetry spans, metrics, or structured logs. Diagnostic output is confined to `SolverFailureDetail.failure_message` on failure and `extension_metadata` on solution-present responses, per SPEC-011 FR-9.2.

---

# Security Considerations

**Seed authority (ADR-010 Decision 4, SPEC-017 FR-9):** All reproducibility-critical stochastic operations use the NumPy PCG64 rng seeded from `execution_seed`. No OS entropy source is consulted in reproducibility-critical paths. Using prohibited sources produces non-reproducible outputs that undermine evidence integrity.

**Extension metadata safety:** The six `extension_metadata` keys contain only derived scalar summary values. No key contains geographic coordinate arrays, stop identifier lists, demand arrays, time window arrays, or any other raw routing problem input. This complies with the raw-data prohibition in SPEC-004 FR-13 and SPEC-001 Security Considerations.

**Input trust:** The routing problem arrives at the backend pre-validated by Core (ADR-009) and pre-deserialized by the adapter (SPEC-017 FR-3). The backend trusts the problem's structural validity and does not re-validate coordinates, demands, or fleet parameters.

**No external access:** The backend has no access to external networks, cloud APIs, IBM Quantum Runtime, or any services outside the Docker Compose internal network. ADR-007 explicitly defers hardware execution.

**No stdout/stderr:** Diagnostic output is confined to the SolverResponse contract fields, preventing routing problem data from escaping through uncontrolled channels. The adapter's stdout channel is reserved for its own structured JSON log events (SPEC-017 Constraint 14).

**`execution_seed` confidentiality:** `execution_seed` must not appear in adapter log events (SPEC-017 Security Considerations). The derived sub-seeds (`transpiler_seed`, `simulator_seed`, `optimizer_seed`) must also be treated as sensitive and must not appear in adapter log events. At MVP scope with synthetic routing problems, this is accepted risk; it is included in the security model for non-MVP deployments.

**Qiskit dependency security:** The Qiskit and Qiskit Aer packages must be version-pinned in the adapter's requirements file (SPEC-017 FR-8). Unpinned dependencies risk behavioral changes across versions that may violate the reproducibility invariant.

---

# Performance Considerations

**Simulator tractability boundary:** The exponential scaling of state-vector simulation with qubit count is the primary performance constraint. The implementation must determine the maximum tractable QUBO binary variable count Q before the backend is registered as eligible. If a routing problem at Small scale produces Q values exceeding the tractable limit, the backend should return Failed with InternalError at QUBO construction time rather than attempting simulation.

**Transpilation cost:** Transpilation must be measured separately from circuit simulation time. If transpilation occurs once before the optimization loop, it is a fixed overhead per invocation; if per-evaluation, it multiplies with optimizer evaluation count. The implementation must report transpilation overhead to inform the `latency_profile` empirical measurement.

**`transport_overhead_buffer_ms` interaction:** The SPEC-017 OQ-1 buffer must account for QAOA's potentially large JSON response serialization time (when `extension_metadata` values are large) and for the possibility that the backend is mid-evaluation when it detects the timeout. The buffer must be calibrated with QAOA responses in the test set.

**`execution_duration_ms` scope (SPEC-017 FR-5):** `execution_duration_ms` measures from the moment Python solver computation begins (after JSON deserialization, backend routing, contract version validation, and rng initialization) to the moment the adapter begins constructing the SolverResponse. Phase 1 rng draws and Phase 2 parameter initialization are included. Transpilation time is included if transpilation occurs after rng initialization. JSON response serialization is excluded.

---

# Documentation Updates Required

**SPEC-011 FR-11 (MVP Backend Inventory):**
- The `Python Adapter` deferred backends table entry notes "individual Python backend specifications (SPEC-018+) may now be written." SPEC-018 satisfies this. Update SPEC-011 FR-11 to list `qaoa-qiskit` under Python adapter backends with this specification's status once SPEC-018 is Accepted.

**SPEC-003:**
- The capability profile for `qaoa-qiskit` must be registered through the mechanism resolved by SPEC-003 OQ-2 before this backend becomes eligible for Scheduler selection. No SPEC-003 schema change is required; the capability profile fields defined in FR-9 conform to the existing SPEC-003 FR-4 schema.

**ADR-007:**
- ADR-007 is currently Proposed status. It explicitly identifies Qiskit Aer local simulation as a permissible future extension ("Can be added as a Python adapter extension in a future iteration without architectural changes"). SPEC-018 is that extension. ADR-007 should be advanced to Accepted following this specification's acceptance review, referencing SPEC-018 as the concrete realization of the Qiskit Aer local simulation alternative it identified.

**SPEC-017:**
- SPEC-017 Definition of Done states: "At least one individual Python backend specification (SPEC-018) conforming to SPEC-017 is Accepted." SPEC-018 satisfies this prerequisite once Accepted. SPEC-017 Definition of Done may be updated accordingly.

**Python adapter container image (SPEC-017 FR-8):**
- `qiskit`, `qiskit-aer`, and required Qiskit subpackages must be added to the python-adapter container image's pinned requirements file before this backend can be deployed.

**ADR-010:**
- No changes required. ADR-010 Architectural Impact section already annotates Python backends: "Python backends initialize PCG64 via `numpy.random.SeedSequence(execution_seed, spawn_key=(BACKEND_SPAWN_KEY,))` per SPEC-017 FR-9." SPEC-018 declares `BACKEND_SPAWN_KEY = 20260620`. Future Python backends must choose spawn keys distinct from `20260620`.

**docs/architecture.md:**
- No changes required. The Python Solver Adapter is already identified in architecture.md. The `qaoa-qiskit` backend is one of the adapter's hosted backends and requires no architecture-level change.

---

# Open Questions

### OQ-1: QUBO Binary Variable Count Tractability Limit

**Classification:** Implementation Planning Decision

**Question:** What is the maximum QUBO binary variable count Q that is tractable on the local Aer simulator within the Small size class `latency_profile` bound (< 300.0s)?

**Context:** QAOA circuit simulation scales exponentially with Q. For a CVRP with N stops and V vehicles, Q depends on the binary encoding scheme (implementation planning decision per FR-2). Encoding strategies range from a dense O(N × V) representation to sparser encodings. The tractable Q boundary determines which Small problems can actually be solved and whether the backend should declare supported_size_classes as a subset of Small (e.g., only problems with N ≤ 10 or N ≤ 15).

**Resolution required before:** Implementation begins. If the tractable Q limit excludes a significant portion of the Small size class, the capability profile declaration in FR-9 must be revised to reflect the actual supported problem range before `is_provisional` is set to false.

**Blocking:** Does not block SPEC-018 acceptance. Blocks `is_provisional = false` declaration and must be resolved before implementation to avoid implementing a backend that cannot satisfy its declared capability profile.

---

### OQ-2: Transpilation Strategy — Once vs. Per-Evaluation

**Classification:** Implementation Planning Decision

**Question:** Should transpilation occur once before the optimization loop (with fixed circuit structure) or per optimizer evaluation (allowing parameter-dependent compilation)?

**Context:** Transpiling once before the loop minimizes transpilation overhead but requires a circuit structure compatible with all possible parameter values (a parameterized circuit). Transpiling per evaluation allows potential compilation optimizations for each parameter set but multiplies transpilation overhead by the optimizer evaluation count. The choice affects `execution_duration_ms` composition and the `qaoa.optimizer_evaluations` to wall-clock time ratio.

**Resolution required before:** Implementation planning begins. The transpilation strategy affects how `transpiler_seed` is used (once vs. repeatedly) and whether the transpiler produces identical output per evaluation (observable in reproducibility testing).

**Blocking:** Does not block SPEC-018 acceptance. Must be decided before implementation begins.

---

### OQ-3: Classical Optimizer Algorithm Selection

**Classification:** Implementation Planning Decision

**Question:** Which classical optimizer algorithm should be used for variational parameter optimization?

**Context:** The classical optimizer drives the QAOA feedback loop. Deterministic optimizers (COBYLA, L-BFGS-B with finite differences) do not consume `optimizer_seed` from FR-10 Phase 1 but are available as implementations in SciPy and do not require additional seeding beyond the initial parameters from Phase 2. Stochastic optimizers (SPSA) use random gradient estimates that must be seeded from `optimizer_seed`. COBYLA is widely used for QAOA in the literature due to its no-gradient requirement. The specific optimizer affects convergence behavior, evaluation count, and whether `optimizer_seed` is actively consumed.

**Resolution required before:** Implementation planning. If a stochastic optimizer is selected, its seeding from `optimizer_seed` must be verified to produce reproducible parameter trajectories.

**Blocking:** Does not block SPEC-018 acceptance. Must be decided before implementation begins.

---

### OQ-4: Capability Profile Empirical Latency and Quality Values

**Classification:** Implementation Planning Decision

**Question:** What are the empirically measured latency values across the Small size class and the confirmed quality classification for the `qaoa-qiskit` backend?

**Context:** FR-9 declares `is_provisional = true` with pre-implementation latency estimates and `Baseline` quality classification. Empirical measurement against the SPEC-002 synthetic workload is required. The feasibility of achieving < 300.0s for all Small problems depends on the tractable Q limit (OQ-1) and the chosen ansatz depth and shot count. If the empirical latency exceeds 300.0s for Small problems at any meaningful qubit count, the `latency_profile` declaration may need revision (reducing supported_size_classes or increasing the bound).

**Resolution required before:** Setting `is_provisional = false` in the capability profile.

**Blocking:** Does not block SPEC-018 acceptance or initial implementation. Blocks `is_provisional = false` declaration.

---

### OQ-5: Feasible Solution Frequency Threshold

**Classification:** Project Owner Decision Required

**Question:** What minimum frequency of feasible bitstring discovery (`qaoa.feasible_bitstrings_found > 0`) across SPEC-002 Small workload executions is acceptable before concluding that the QUBO formulation is effective?

**Context:** If the QUBO penalty structure does not guide the optimizer toward feasible assignments, `qaoa.feasible_bitstrings_found` will be 0 across most executions, and the backend will routinely return `Failed` with no solution (FR-14, natural termination with no feasible solution). This constitutes a formulation quality defect, not a contract violation. A Project Owner threshold is required to determine when this failure rate triggers a required QUBO formulation revision.

**Resolution required before:** `is_provisional` can be set to false and the backend can be considered qualified. A failure rate above the accepted threshold requires QUBO formulation revision and a `specification_version` increment.

**Blocking:** Does not block SPEC-018 acceptance or initial implementation. Blocks `is_provisional = false` declaration.

---

### OQ-6: Maximum Single Evaluation Duration

**Classification:** Implementation Qualification Requirement

**Question:** What is the maximum acceptable wall-clock duration of a single Qiskit circuit evaluation (transpilation time, if per-evaluation, plus shot simulation time) for the QAOA backend across the supported problem range within the Small size class?

**Context:** The interrupt compliance bound declared in FR-14 is "the duration of one Qiskit circuit evaluation." Until the maximum single-evaluation duration is empirically established, this bound provides no operational assurance for the adapter's resource planning after a cancellation signal. A single-evaluation duration that approaches the full `execution_timeout_ms` budget renders the interrupt compliance bound effectively unbounded. The empirically established maximum also informs the `transport_overhead_buffer_ms` calibration (SPEC-017 OQ-1): if one evaluation can take close to the full execution budget, the buffer must account for the adapter still completing that evaluation after the HTTP client timeout window opens.

**Resolution required before:** `is_provisional = false` qualification review. The implementation must measure the 99th-percentile single-evaluation wall-clock duration across the supported range of QUBO dimensions and ansatz depths within the Small size class and report the measured bound as part of the qualification evidence.

**Blocking:** Does not block SPEC-018 acceptance or initial implementation. Blocks `is_provisional = false` declaration.

---

# Acceptance Checklist

**SPEC-011 Framework Obligations (FR-12):**

- [ ] Metadata: All FR-3 metadata fields present in the solver metadata table (backend_id, display_name, backend_category, implementation_language, determinism_class, supported_contract_version, specification_version)
- [ ] Problem Statement: States what optimization problem this backend solves, its algorithmic approach (QAOA via Qiskit local simulator), and why it belongs in the project
- [ ] Solver Classification: Backend category declared as `quantum_inspired_stochastic`; determinism class declared as `Stochastic (reproducible)`; infeasibility proof capability declared as No
- [ ] Algorithm Description (FR-1): Three execution phases defined; distance metric stated; service duration and time window non-use stated; Succeeded and Timeout conditions defined
- [ ] QUBO Formulation and Qiskit Ownership Boundaries (FR-2): SPEC-018-owned vs. implementation planning vs. adjacent specification ownership clearly delineated; Qiskit implementation details correctly categorized as implementation planning decisions
- [ ] Ansatz Depth Handling (FR-3): Observable constraints (p ≥ 1, fixed before loop, reported in extension_metadata) defined; specific p value is implementation planning decision
- [ ] Shot Execution (FR-4): Observable constraints (shots_per_evaluation ≥ 1, fixed before loop, reported in extension_metadata) defined; shot reproducibility stated
- [ ] Classical Optimizer Behavior (FR-5): Observable optimizer behaviors defined; stochastic optimizer seeding obligation stated; timeout monitoring at evaluation boundaries stated
- [ ] Bitstring Decoding and Route Construction (FR-6): Feasibility gate defined; capacity constraint handling stated; distance metric for ranking defined; time window non-enforcement stated
- [ ] Best-So-Far Solution Tracking (FR-7): Update trigger (per optimizer evaluation); quality comparison criterion (minimum Haversine distance); anytime contract; solution_count semantics; coarser granularity vs. SPEC-015 acknowledged
- [ ] Constraint Handling (FR-8): Capacity enforced; time windows and service durations not enforced; infeasibility detection not supported
- [ ] Capability Profile Declaration (FR-9): All nine fields from SPEC-011 FR-4.1 present; supported_size_classes = {Small} with rationale; quality_profile = Baseline with rationale; is_provisional = true declared; latency_profile in seconds; accuracy basis stated
- [ ] Seed Usage Policy (FR-10): Stochastic class stated; NumPy PCG64 named; BACKEND_SPAWN_KEY declared as 20260620 and frozen; rng initialization formula per SPEC-017 FR-9 stated; Phase 1 draw ordering (3 draws: transpiler_seed, simulator_seed, optimizer_seed) documented; Phase 2 draw ordering (2p draws: interleaved γ, β) documented; Phase 3 via sub-system seeds stated; Qiskit randomness obligation (SPEC-017 FR-9) satisfied; prohibited entropy sources confirmed absent; satisfies SPEC-004 FR-1 and SPEC-011 FR-6.5 and SPEC-017 FR-9
- [ ] Supported SolverOutcome Values (FR-11): Explicit table; Infeasible listed as Not Supported; Timeout-with-solution and Cancelled-with-solution behavior stated; satisfies SPEC-011 FR-5.3
- [ ] RoutePlan Output Requirements (FR-12): Presence per outcome; SPEC-011 FR-7.1 narrowing declared; capacity validity guarantee; stop completeness guarantee; time window non-guarantee; execution_duration_ms obligation per SPEC-017 FR-5
- [ ] Extension Metadata (FR-13): All six keys documented with types, presence conditions, and descriptions; raw-data prohibition confirmed; absence on no-solution responses stated; rationale for Timeout-with-solution inclusion stated; satisfies SPEC-011 FR-7.4
- [ ] Failure Model (FR-14): ContractVersionMismatch, Qiskit import failure, QUBO construction failure, transpilation failure, natural termination with no feasible solution, internal error, timeout, and cancellation all defined; interrupt compliance bound declared (SPEC-017 OQ-3B satisfied)
- [ ] Performance Characteristics: Expected behavior by Small range; basis for estimates stated; simulator tractability constraint noted
- [ ] Testability: 27 test contracts covering contract conformance, failure model, stochastic algorithm correctness, reproducibility pipeline, capability profile accuracy, extension metadata, observability, and adapter integration
- [ ] Open Questions: OQ-1 through OQ-6 classified with blocking status; no open questions with unknown classification remain
- [ ] No individual solver specification obligation contradicts a framework requirement from SPEC-011 FR-1 through FR-11

**SPEC-017 Compliance Obligations:**

- [ ] BACKEND_SPAWN_KEY declared as a unique positive integer (20260620), frozen on Acceptance, distinct from any existing spawn key
- [ ] rng initialization formula follows SPEC-017 FR-9 exactly
- [ ] All three Qiskit randomness sources addressed (transpiler, simulator, optimizer per SPEC-017 FR-9)
- [ ] Seed derivation produces Python integers in [0, 2^31) compatible with Qiskit seed parameters
- [ ] Prohibited entropy sources per SPEC-017 FR-9 confirmed absent
- [ ] SPEC-017 OQ-3B interrupt compliance bound declared (one circuit evaluation time)
- [ ] No external network calls during solver execution confirmed (ADR-007)
- [ ] No stdout/stderr usage confirmed (SPEC-017 Constraint 14)

**Backend-Specific Acceptance Criteria:**

- [ ] The anytime behavioral contract is clearly defined: Timeout and Cancelled responses include the best-so-far solution when it exists, with discrete update granularity (per optimizer evaluation)
- [ ] The Qiskit ownership boundary is explicitly stated: Qiskit implementation details (primitives, transpilation strategy, optimizer algorithm) are correctly deferred to implementation planning; SPEC-018 defines only observable behaviors and seed derivation obligations
- [ ] The three-phase PRNG structure is unambiguous: Phase 1 (3 draws), Phase 2 (2p draws), Phase 3 (via sub-system seeds, no main rng consumption)
- [ ] The optimizer_seed draw at Phase 1 position 3 is documented as always consumed regardless of optimizer type
- [ ] Extension metadata presence conditions are unambiguous (all six keys present when solution present; all absent otherwise)
- [ ] The specification does not define Scheduler, Worker, or Core behavior
- [ ] The specification does not define any Qiskit circuit, gate, primitive, or implementation detail beyond observable behavioral contracts and seed derivation obligations
- [ ] supported_size_classes = {Small} only is justified by simulator tractability constraints

---

# Definition of Done

This backend is complete when:

- SPEC-018 is in Accepted status
- OQ-1 (QUBO binary variable count tractability limit) is determined empirically during implementation and reflected in a revised capability profile if the tractable range is narrower than the full Small size class
- OQ-2 (transpilation strategy) is decided before implementation begins
- OQ-3 (classical optimizer selection) is decided before implementation begins; if a stochastic optimizer is selected, its seeding from `optimizer_seed` is verified to produce reproducible parameter trajectories
- The backend is implemented as a Python solver function hosted by the SPEC-017 python-adapter container
- All SPEC-004 contract conformance tests pass via the SPEC-017 adapter
- Reproducibility is verified: two identical SolverRequests (same problem, same execution_seed) produce identical SolverResponse solution fields when both produce Succeeded; verification must confirm that transpiler, simulator, and optimizer seeding all flow deterministically from execution_seed
- Anytime behavior is validated: the backend returns Timeout with a complete RoutePlan on test problems designed to trigger deadline self-termination
- The `solver.execute` span is emitted by the Worker for every adapter invocation and verifiable in the test environment
- Extension metadata keys (all six from FR-13) are present in every SolverResponse where a solution is present
- The adapter structured log events (SPEC-017 FR-13) are emitted and correlated via job_id and decision_id
- Self-termination before the Worker's HTTP client timeout is verified on test cases that trigger the timeout path
- Empirical latency and quality values are measured from the SPEC-002 synthetic workload; `is_provisional = false` is declared after OQ-4 and OQ-5 are resolved by Project Owner decision and OQ-6 is reported
- The capability profile is registered through the mechanism resolved by SPEC-003 OQ-2
- OQ-5 (feasible solution frequency threshold) is reviewed against empirical measurements; threshold is within the Project Owner-approved range
- OQ-6 (maximum single-evaluation duration) is empirically measured across the supported range of QUBO dimensions and ansatz depths; the measured bound is documented in the `is_provisional = false` qualification review
- Qiskit and Qiskit Aer are added to the python-adapter container image's pinned requirements file
- ADR-007 is advanced to Accepted following this specification's acceptance
- Engineering review passes
- Specification status is updated to Verified
