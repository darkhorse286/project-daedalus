# SPEC-015: QUBO Simulated Annealing Solver

## Metadata

**Feature ID:** SPEC-015

**Title:** QUBO Simulated Annealing Solver

**Status:** Draft

**Author:** Darkhorse286

**Created:** 2026-06-19

**Last Updated:** 2026-06-19

**Supersedes:** None

**Superseded By:** None

**Parent Specification:** SPEC-011 (Backend Solver Specifications)

**Related ADRs:** ADR-001, ADR-005, ADR-006, ADR-007, ADR-008, ADR-009, ADR-010, ADR-011

**Related Specs:** SPEC-001, SPEC-002, SPEC-003, SPEC-004, SPEC-005, SPEC-006, SPEC-007, SPEC-010, SPEC-011, SPEC-012

---

**Solver Metadata (SPEC-011 FR-3):**

| Field | Value |
|---|---|
| `backend_id` | `qubo-simulated-annealing` |
| `display_name` | QUBO Simulated Annealing |
| `backend_category` | `quantum_inspired_stochastic` |
| `implementation_language` | `C++` |
| `determinism_class` | Stochastic (reproducible) |
| `supported_contract_version` | 1 |
| `specification_version` | 1 |

---

# Problem Statement

Project DAEDALUS requires a stochastic optimization backend to complete the three-backend comparison necessary for its core thesis: that most routing problems should never touch a quantum-adjacent backend because cheaper deterministic methods are sufficient. Without a stochastic optimization backend, the system can only demonstrate that two classical construction heuristics produce different quality levels — it cannot answer the question that motivates the entire architecture: *when, if ever, does a higher-cost stochastic optimizer justify its additional computation?*

The nearest-neighbor (SPEC-013) and greedy insertion (SPEC-014) backends provide the classical baselines. Their solutions are deterministic and fast but structurally limited: each makes locally optimal decisions at every step without global search. The evidence system can report that one classical heuristic outperforms another, but it cannot produce evidence that a stochastic optimizer achieves meaningfully better routes under the same execution constraints.

QUBO simulated annealing is the designated stochastic optimization backend for Project DAEDALUS. Simulated annealing is a classical stochastic search algorithm with well-understood convergence properties. Framing it as a QUBO (Quadratic Unconstrained Binary Optimization) algorithm positions it within the quantum-inspired optimization taxonomy without requiring quantum hardware, satisfying the project's quantum-adjacent framing while remaining entirely local and reproducible.

SPEC-015 defines the authoritative QUBO simulated annealing backend implementation. It specifies observable algorithm behavior, the capability profile for Scheduler consumption, reproducibility obligations under ADR-010, anytime timeout behavior, extension metadata for annealing evidence, and the failure model. It does not define the solver framework, the solver contract, the Scheduler policy, or the internal QUBO formulation. Those responsibilities belong to SPEC-004, SPEC-011, SPEC-003, and implementation planning, respectively.

---

# Business Value

- Completes the three-backend MVP inventory required by SPEC-011 FR-11, enabling the system to execute its full comparison cycle
- Provides the stochastic optimizer against which the Scheduler's regret analysis measures the cost of choosing construction heuristics over optimization
- Demonstrates the system's thesis empirically: on workload classes where SA produces meaningfully better routes, the evidence report documents the quality improvement and its cost; on workload classes where classical heuristics are sufficient, the evidence report demonstrates that SA's additional cost was unjustified
- Introduces the first `execution_seed`-consuming backend, validating the ADR-010 reproducibility pipeline end-to-end
- Delivers the system's first "quantum-inspired" component for portfolio signaling purposes, backed by a traceable evidence chain that supports the claim

---

# Employer Signaling

- System Design
- Optimization
- AI Engineering
- Reliability Engineering

QUBO simulated annealing is a classical stochastic optimization technique that maps a combinatorial optimization problem onto a quadratic binary form, then uses SA to minimize the binary objective. Specifying a reproducible stochastic backend requires correctly handling seed authority, PRNG draw ordering, anytime behavior, and the evidence chain for stochastic execution — concerns that do not arise in deterministic construction heuristics. This specification demonstrates the ability to write reproducible stochastic system specifications against a formal PRNG policy, and to define the evidence artifacts that make stochastic execution comparable across invocations.

---

# Solver Classification

**Backend category:** `quantum_inspired_stochastic`

**Determinism class:** Stochastic (reproducible)

**Infeasibility proof capability:** No

The QUBO simulated annealing backend uses a PRNG-dependent stochastic search process. Its output is not identical for identical routing problem inputs in general: two invocations with different `execution_seed` values may produce different route plans. However, given the same `execution_seed`, the backend produces an identical `solution` on every invocation (ADR-010 reproducibility invariant, SPEC-004 FR-11).

Classification is `quantum_inspired_stochastic` per SPEC-011 FR-2.1, which applies to "stochastic optimization heuristics using PRNG-dependent search over a solution space." This classification is a property of the algorithm's stochastic execution, not a runtime configuration parameter.

The backend is not an exact solver. It cannot prove that a routing problem has no feasible solution. The only failure conditions resulting from problem characteristics produce `Failed` with `InternalError`, not `Infeasible`.

**Timeout behavior:** This backend implements anytime behavior per SPEC-004 FR-10. It maintains a running best-so-far complete solution — a RoutePlan satisfying all SPEC-004 FR-5 structural validity requirements — that is returnable at any point during annealing. When the `execution_timeout_ms` deadline is reached, the backend self-terminates and returns the best-so-far solution if one has been found. This is a critical behavioral distinction from classical construction heuristics (SPEC-013, SPEC-014), which can only return Timeout with no solution. See FR-6 and FR-10.

**Supported SolverOutcome values:** See FR-10.

---

# Requirements

### FR-1: Algorithm Description

**Description:**

The QUBO simulated annealing backend solves the Capacitated Vehicle Routing Problem (CVRP) by encoding the routing problem as a quadratic binary optimization instance, then applying simulated annealing to search the binary solution space for low-cost route assignments. Solutions in binary space are decoded into route plans; decoded plans are evaluated for structural validity and route distance quality. The algorithm maintains the best complete, structurally valid solution found at any point during annealing.

This section defines externally observable algorithm behavior. It does not define the internal QUBO energy function, penalty weights, binary encoding scheme, decoding algorithm, or repair heuristic. Those are implementation planning decisions governed by FR-2.

**High-level execution phases:**

1. **Initial solution generation:** The backend generates an initial starting state for the annealing process from the routing problem and `execution_seed`. All PCG64 draws consumed during this phase occur before any annealing iteration begins.

2. **Annealing iterations:** The backend repeatedly proposes candidate state transitions, evaluates them against the current state, and accepts or rejects them according to the Metropolis acceptance criterion (FR-5). The temperature parameter decreases monotonically across iterations (FR-3). The backend tracks the best complete, structurally valid solution found during all iterations (FR-6).

3. **Termination:** The annealing process ends when the temperature schedule is exhausted, the maximum iteration budget is reached, or the `execution_timeout_ms` deadline arrives — whichever occurs first. At termination, the backend returns the appropriate outcome with the best-so-far solution if one exists.

**Distance metric:** Haversine great-circle distance in kilometers (SPEC-001 FR-5). The same formula used throughout the domain. No alternative metric is used during route distance evaluation.

**Service duration and time window behavior:** Service duration values and time window fields are not consulted during the optimization process. Route quality optimization is based exclusively on Haversine route distance. Time window feasibility is evaluated post-execution by Core Quality Evaluation (SPEC-007).

**Completion condition:** The algorithm produces a Succeeded response when the annealing temperature schedule or iteration budget is exhausted naturally — before the `execution_timeout_ms` deadline — AND at least one complete, structurally valid RoutePlan was found during annealing. The returned solution is the best-so-far complete solution, not necessarily the last state evaluated.

**Timeout condition:** The algorithm produces a Timeout response when the `execution_timeout_ms` deadline is reached before the natural annealing termination. The best-so-far complete solution is included if one was found before the deadline. See FR-6 and FR-10.

**Acceptance Criteria:**
- The algorithm executes in the three phases described above: initial solution generation, annealing iterations, termination
- Service duration and time window fields are not consulted during the optimization process
- Succeeded is returned only when annealing terminates naturally (schedule or iteration budget exhausted before deadline) AND a complete solution was found
- Timeout is returned when the deadline is reached; best-so-far is included if available

---

### FR-2: QUBO Formulation Ownership Boundaries

**Description:**

The QUBO formulation maps the CVRP routing problem to a binary optimization problem. This section defines which aspects of that formulation are within SPEC-015's scope, which belong to implementation planning, and which are owned by adjacent specifications.

**SPEC-015 owns:**
- The observable behavioral contract: given a SolverRequest, the backend must produce a SolverResponse conforming to SPEC-004
- The reproducibility invariant: identical `execution_seed` values produce identical solutions across invocations (FR-8, ADR-010)
- The PRNG policy: PCG64 seeded from `execution_seed` with stream constant `0xcbbb9d5dc90c2383` is the exclusive entropy source (FR-8, ADR-010 Decision 1)
- The anytime contract: best-so-far tracking and its implications for Timeout and Cancelled responses (FR-6)
- The extension metadata key definitions: what annealing evidence is persisted and how (FR-12)
- The failure model: what conditions produce each outcome (FR-13)

**Implementation planning owns:**
- The binary encoding scheme: how stops, vehicles, and assignments are represented as binary variables
- The QUBO objective function: the specific form of the quadratic binary objective, including penalty terms for capacity constraints, assignment completeness, and any other feasibility requirements
- The penalty weights: the relative magnitudes of penalty and objective terms in the QUBO energy function
- The initial solution construction algorithm: how the starting binary state is generated from the PCG64 stream (subject to the draw ordering documented in FR-4)
- The SA neighborhood structure: the specific move operator(s) used to propose candidate transitions from the current state (subject to the draw ordering documented in FR-4)
- The decoding algorithm: how a binary state is mapped to a RoutePlan candidate
- The repair heuristic: how a decoded state that violates SPEC-004 FR-5 structural validity requirements is corrected or discarded
- The annealing schedule parameters: initial temperature, cooling rate, minimum temperature, iteration budget (subject to the constraints in FR-3)

**Adjacent specification ownership:**
- Core quality evaluation of the returned RoutePlan: SPEC-007
- Evidence persistence schema for solver run records: SPEC-006, SPEC-012
- Capability profile consumption by the Scheduler: SPEC-003 FR-4
- Workload feature computation: SPEC-010
- External timeout enforcement: SPEC-005

**Acceptance Criteria:**
- No implementation planning decision invalidates the reproducibility invariant (FR-8): the PCG64 stream constant, draw ordering, and seeding procedure are frozen once this specification is Accepted
- No implementation planning decision causes the backend to return `Infeasible` (prohibited per SPEC-011 FR-5.2)
- The decoding and repair strategy must ensure that any RoutePlan included in a SolverResponse satisfies all SPEC-004 FR-5 structural validity requirements

---

### FR-3: Annealing Schedule Behavior

**Description:**

The annealing schedule controls the temperature parameter across iterations. SPEC-015 defines the observable properties of the annealing schedule; the specific parameter values are implementation planning decisions.

**Monotonic decrease:** The temperature parameter must be monotonically non-increasing across annealing iterations. The temperature must never increase during normal annealing execution. This constraint is observable through `qsa.initial_temperature` and `qsa.final_temperature` in extension_metadata: `final_temperature` must be less than or equal to `initial_temperature`.

**Temperature schedule class:** The specific annealing schedule class (geometric, linear, adaptive, or other) is an implementation planning decision. The observable constraint is monotonic non-increase: the temperature must not increase at any iteration boundary during normal execution. The three extension metadata values — `qsa.initial_temperature`, `qsa.final_temperature`, and `qsa.iteration_count` — are each independently observable; their interpretation does not require knowledge of the schedule class.

**Temperature termination condition:** Annealing terminates naturally (Succeeded path) when the temperature drops below the implementation-defined minimum temperature threshold, or when the maximum iteration budget is reached, or when both conditions are satisfied simultaneously.

**Iteration budget behavior:** The backend has a maximum iteration count. This count may be derived from the problem size, a static configuration value, or the `execution_timeout_ms` budget. The specific derivation is an implementation planning decision. If timeout arrives before the iteration budget is exhausted, the timeout takes precedence (Timeout path).

**Acceptance Criteria:**
- `qsa.final_temperature` ≤ `qsa.initial_temperature` in all SolverResponses that include extension_metadata
- A Succeeded response implies natural termination (schedule or iteration budget exhausted) before the `execution_timeout_ms` deadline

---

### FR-4: Initial Solution Generation and PRNG Draw Ordering

**Description:**

This section documents the PRNG draw ordering for all PCG64 draws consumed during a single solver execution (ADR-010 Limitations, SPEC-011 FR-6.2). The ordering is divided into two phases: initial solution generation and annealing iterations.

**Phase 1 — Initial solution generation:**

All PCG64 draws consumed to construct the starting state for the annealing process occur in Phase 1. All Phase 1 draws are consumed before the first annealing iteration begins.

The exact ordering of Phase 1 draws depends on the initial solution construction algorithm selected during implementation planning. The construction strategy may be stop-centric (processing stops in a deterministic order), binary-variable-centric (initializing binary state variables in index order), or another approach determined by the QUBO encoding choice (FR-2). Because the QUBO binary encoding is an implementation planning decision, the Phase 1 draw ordering cannot be fixed in this Draft specification. The specific Phase 1 draw ordering must be documented as part of OQ-2 resolution; FR-4 must be updated at that time with the complete Phase 1 draw sequence.

The observable constraint is: all Phase 1 draws must be consumed before the first annealing iteration begins, regardless of which construction strategy is chosen.

**Phase 2 — Annealing iterations:**

All PCG64 draws consumed during the annealing process occur in Phase 2, after Phase 1 is complete. Within Phase 2, draws are consumed in iteration order: all draws for iteration n are consumed before any draws for iteration n+1. Within each iteration, draws are consumed in a fixed sequence determined by the SA neighborhood structure: first the draws for neighbor proposal, then the draw for Metropolis acceptance (if needed; see FR-5). The exact number of draws per iteration is an implementation planning decision determined by the SA neighborhood structure (FR-2). See OQ-2.

**Cross-phase ordering:**

Phase 1 draws are always consumed before Phase 2 draws, regardless of implementation ordering. This ordering is fixed and must not be altered. The boundary between phases is observable: the first annealing iteration begins after the last Phase 1 draw is consumed.

**Acceptance Criteria:**
- Phase 1 draws are consumed before Phase 2 begins; no Phase 2 draw occurs before Phase 1 is complete
- Phase 1 draw ordering within the construction strategy is documented in the OQ-2 resolution update to this section
- Phase 2 draws are consumed in iteration order; within each iteration, neighbor proposal draws precede acceptance draws
- Two executions with identical `execution_seed` values and identical routing problems consume draws in the same sequence and produce the same result (ADR-010 reproducibility invariant)

---

### FR-5: Transition Proposal and Metropolis Acceptance

**Description:**

At each annealing iteration, the backend proposes a candidate transition from the current state to a neighboring state, then applies the Metropolis acceptance criterion to decide whether to adopt the new state.

**Neighbor proposal:** The SA neighborhood structure (the specific move operator) is an implementation planning decision (FR-2). The proposal process consumes PCG64 draws as described in FR-4 (Phase 2, neighbor proposal draws first within each iteration).

**Metropolis acceptance criterion:** The decision to accept or reject a proposed transition follows the Metropolis criterion:

- If the proposed state has lower or equal energy (ΔE ≤ 0): accept unconditionally. No PCG64 draw is consumed for this decision.
- If the proposed state has higher energy (ΔE > 0): accept with probability `e^(-ΔE/T)`, where T is the current temperature. One PCG64 uniform floating-point draw in (0, 1] is consumed for this comparison (ADR-010 Decision 3, uniform floating-point conversion formula). If the draw value is less than `e^(-ΔE/T)`, the transition is accepted; otherwise it is rejected.

The Metropolis acceptance draw, when consumed, always follows all neighbor proposal draws for that iteration. The acceptance draw is consumed only when ΔE > 0; it is never consumed for improving transitions.

**State update:** If a transition is accepted, the current state is updated to the proposed state. The best-so-far solution tracking (FR-6) evaluates the new state after the state update.

**Energy function:** The specific energy function (QUBO objective) is an implementation planning decision (FR-2). The energy function determines what constitutes an "improvement" (ΔE ≤ 0) versus a "worsening" (ΔE > 0). The energy function must be consistent with Haversine route distance minimization and capacity constraint encoding.

**Acceptance Criteria:**
- Worsening transitions (ΔE > 0) consume exactly one PCG64 uniform float draw for the Metropolis acceptance decision
- Improving transitions (ΔE ≤ 0) consume no PCG64 draw for the acceptance decision
- The acceptance draw is the last draw consumed in each iteration where it applies
- `qsa.accepted_worse_transitions` in extension_metadata equals the count of accepted worsening transitions across all iterations

---

### FR-6: Best-So-Far Solution Tracking

**Description:**

The backend implements anytime behavior (SPEC-004 FR-10). It maintains a best-so-far RoutePlan — the best complete, structurally valid solution found at any point during Phase 1 or Phase 2 — that can be returned on Timeout or Cancelled without waiting for natural termination.

**What qualifies as a best-so-far candidate:** A RoutePlan qualifies as a best-so-far candidate if and only if it satisfies all SPEC-004 FR-5 structural validity requirements:
1. `routes.size()` equals `vehicle_count`
2. Every stop ID is a valid stop ID from the routing problem
3. No stop ID appears in more than one route
4. Every stop ID appears in exactly one route
5. For each route, total stop demand ≤ `capacity_per_vehicle`

A decoded binary state that violates any of these requirements does not qualify as a best-so-far candidate, regardless of its energy value. Partial assignments — where some stops are unassigned — are never valid candidates (SPEC-004 FR-5, SPEC-011 OQ-5).

**Quality comparison for best-so-far updates:** When a new complete, valid RoutePlan is produced, it replaces the current best-so-far if and only if its total Haversine route distance (sum of all vehicle route distances including depot-to-first-stop and last-stop-to-depot legs) is strictly less than the best-so-far total distance. On the first valid solution produced, it becomes the best-so-far unconditionally.

**When best-so-far is evaluated:** The best-so-far is evaluated after each state that produces a complete, valid decoded RoutePlan, whether during initial solution generation (Phase 1) or during annealing iterations (Phase 2).

**Absence of best-so-far:** If no complete, valid RoutePlan has been produced by the time annealing terminates (whether via natural termination, timeout, or cancellation), the best-so-far is absent. The response includes no `solution` field.

**Anytime contract:** Because the backend maintains a best-so-far, it satisfies the anytime behavior requirement in SPEC-004 FR-10. Unlike construction heuristics (SPEC-013, SPEC-014) that can only produce a complete solution at algorithm completion, the SA backend may have a complete solution available at any point after the first valid decoded state is found. This is the primary architectural distinction between this backend and the classical deterministic backends.

**solution_count in ExecutionStatistics:** The backend populates `solution_count` (SPEC-004 FR-6) with the number of times the best-so-far was updated during execution. A value of 1 means the first valid solution found was never improved upon. A value of 0 means no valid solution was ever found. A value of N means the best-so-far was updated N times (the initial and N-1 subsequent improvements).

**Acceptance Criteria:**
- A RoutePlan is included in any SolverResponse if and only if a complete, structurally valid solution satisfying all SPEC-004 FR-5 requirements was found before the corresponding termination event
- The returned RoutePlan is the one with the minimum total Haversine route distance among all complete, valid solutions found during execution
- `solution_count` equals the number of best-so-far updates; it equals 0 when no complete solution was found
- On Timeout and Cancelled responses, the best-so-far solution is present if it exists; absent otherwise

---

### FR-7: Constraint Handling

**Description:**

The QUBO SA backend enforces some constraints during its optimization process and ignores others. This section defines which constraints are enforced, which are delegated, and which produce failure outcomes.

**Capacity constraints (enforced through formulation):** Capacity constraints are encoded into the QUBO objective as penalty terms. A decoded state that violates capacity constraints does not qualify as a best-so-far candidate (FR-6). The final returned RoutePlan always satisfies capacity validity when present (SPEC-004 FR-5 condition 5).

**Service duration constraints (not enforced):** Service duration values (SPEC-001 FR-16) are present in the routing problem but are not consulted during the optimization process. The objective minimizes Haversine route distance only. Service durations affect time-based feasibility, which is outside this backend's scope.

**Time window constraints (not enforced):** Time window fields (SPEC-001 FR-9) are not consulted during the optimization process. This backend declares `supports_time_windows = false` in its capability profile (FR-8), making it ineligible for time-window-constrained problems in Scheduler eligibility evaluation (SPEC-003 FR-5).

**Infeasibility detection:** This backend does not detect or prove infeasibility. It is not authorized to return `Infeasible` (SPEC-011 FR-5.2). Failure to find any valid solution after completing the annealing process produces `Failed` with `failure_code = InternalError`.

**Acceptance Criteria:**
- Every RoutePlan present in any SolverResponse satisfies SPEC-004 FR-5 capacity validity (condition 5)
- Time window and service duration fields are not consulted during the optimization process
- The backend never returns `Infeasible`

---

### FR-8: Capability Profile Declaration

**Description:**

The following capability profile is the registration artifact consumed by the Scheduler (SPEC-003 FR-4, SPEC-011 FR-4). All declared values must accurately reflect this backend's algorithm behavior and are the basis for Scheduler eligibility evaluation and objective scoring. Values marked as estimates require empirical measurement before `is_provisional` can be set to false.

| Field | Value | Rationale and Accuracy Basis |
|---|---|---|
| `backend_id` | `qubo-simulated-annealing` | Stable unique identifier per SPEC-011 FR-3. Kebab-case per the normative definition in SPEC-011 FR-3.1. |
| `supported_size_classes` | `{Small, Medium, Large}` | SA is applicable to all MVP problem sizes. At Small sizes (1–25 stops), SA incurs more overhead than construction heuristics but produces the same or better quality. All three size classes are supported. |
| `supports_time_windows` | `false` | The optimization process does not incorporate time window constraints. Candidate evaluation is based exclusively on Haversine distance and capacity. Declaring `true` would misrepresent the backend's construction behavior. |
| `supports_capacity_constraints` | `true` | Capacity constraints are encoded as penalty terms in the QUBO objective. Decoded solutions that violate capacity are not treated as best-so-far candidates. The returned RoutePlan always satisfies capacity validity when present. |
| `latency_profile` | Small: < 1.0s, Medium: < 10.0s, Large: < 60.0s | Pre-implementation estimates based on algorithmic analysis: SA's per-iteration cost scales with the binary state dimension (which scales with stop count and vehicle count). Values expressed in seconds per SPEC-003 FR-4. Empirical measurement on SPEC-002 synthetic workload problems of each size class is required before `is_provisional` can be set to false. |
| `quality_profile` | `Competitive` | SA's global stochastic search consistently outperforms greedy construction heuristics on CVRP for problems with sufficient iteration budgets. `Competitive` reflects expected performance at or above the `Competitive` tier established by greedy insertion (SPEC-014) and above the `Baseline` tier of nearest-neighbor (SPEC-013). Note: both `greedy-insertion` (SPEC-014) and `qubo-simulated-annealing` declare `Competitive`; the Scheduler cannot differentiate between them on quality alone under this classification. If empirical validation demonstrates SA consistently achieves quality above GI's tier, this classification should be revised to `Near-Optimal` as part of the `is_provisional = false` transition. See OQ-4. |
| `cost_profile` | `1` | In-process C++ execution with no external dependencies, no network calls, and no paid compute. Cost per invocation is negligible. Value of `1` represents the minimum relative cost unit, consistent with the nearest-neighbor (SPEC-013) and greedy insertion (SPEC-014) backends. SA's higher computational cost relative to construction heuristics is captured in `latency_profile`, not `cost_profile`. |
| `is_provisional` | `true` | Latency and quality profile values are pre-implementation estimates. This backend must declare `is_provisional = true` until empirical measurement validates the declared values per SPEC-011 FR-4.2. Additionally, the stream constant (OQ-1) must be confirmed before `is_provisional` can be set to false. |
| `supported_contract_version` | `1` | Targets SPEC-004 contract version 1. |

**Accuracy basis:** Latency estimates are derived from algorithmic analysis of SA's per-iteration cost over the binary solution space. For N stops and V vehicles, the QUBO state dimension scales as O(N × V), and each SA iteration evaluates one neighbor proposal at that scale. Latency values are registered in seconds per SPEC-003 FR-4. Quality classification as `Competitive` is consistent with SA-for-VRP benchmark results in the literature demonstrating significant improvement over greedy construction. Both values require empirical validation against the SPEC-002 synthetic workload before `is_provisional = false` can be declared.

**Acceptance Criteria:**
- All nine fields from the SPEC-011 FR-4.1 table are present
- `backend_id` matches the FR-3 metadata value
- `supported_contract_version` equals 1
- `is_provisional = true` is declared and remains true until empirical validation and OQ-1 resolution are complete
- The accuracy basis for `latency_profile` and `quality_profile` is stated
- `latency_profile` values are expressed in seconds per SPEC-003 FR-4

---

### FR-9: Seed Usage Policy

**Description:**

The QUBO simulated annealing backend is a stochastic backend. Its reproducibility depends entirely on the `execution_seed` provided in the SolverRequest.

**Determinism class:** Stochastic (reproducible)

**PRNG algorithm:** PCG64 (ADR-010 Decision 1). No other PRNG is permitted in reproducibility-critical paths.

**PCG64 stream constant:** `0xcbbb9d5dc90c2383`

This value is the PCG64 increment (stream) parameter. It is a 64-bit odd integer. It is frozen once this specification is Accepted; changing it requires the full ADR-010 Decision 5 breaking change procedure, which includes updating ADR-010 with an explicit change record documenting the previous and replacement value. A `specification_version` increment alone is insufficient. See OQ-1.

**Seeding procedure:** The PCG64 PRNG is seeded exactly once per solver execution using `execution_seed` from the SolverRequest (SPEC-004 FR-2) as the initial state, and `0xcbbb9d5dc90c2383` as the stream constant. The PRNG is not reseeded at any point during execution — not at phase boundaries, not when a new best-so-far is found, and not on timeout or cancellation. Reseeding during execution is prohibited.

**Seed authority:** `execution_seed` is the exclusive authorized entropy source for all reproducibility-critical stochastic operations. The PRNG must be seeded from `execution_seed` only. System time, process ID, OS random sources (`/dev/urandom`, `getrandom(2)`, `std::random_device`), and hardware entropy are prohibited as inputs to any PRNG seed in reproducibility-critical paths (ADR-010 Decision 4).

**Seed derivation:** `execution_seed` is derived by the Worker from `RoutingProblem.seed` per SPEC-005 FR-7 and ADR-010 Decision 4. The derivation policy is: `execution_seed = RoutingProblem.seed` (no transformation). This backend consumes the provided `execution_seed`; it does not re-derive from the problem seed. Backends must not use `routing_problem.seed` as a PRNG entropy source (SPEC-004 FR-11.2).

**Reproducibility invariant (SPEC-004 FR-11.1):** Given two SolverRequests with identical `routing_problem` fields and identical `execution_seed` values, this backend produces identical `outcome` and identical `solution` fields in both responses, when both produce `outcome = Succeeded`. The reproducibility invariant applies when both invocations produce Succeeded. `execution_duration_ms` is excluded from the invariant (SPEC-004 FR-11.1). Timeout and Cancelled outcomes are timing- and signal-dependent and carry no reproducibility obligation (SPEC-004 FR-11.6).

**Distribution sampling:** This backend uses the following ADR-010 Decision 3 approved distribution algorithms in reproducibility-critical paths:
- **Uniform floating-point on (0, 1]:** For Metropolis acceptance probability comparisons (FR-5), using the conversion formula: `u = (v >> 11) × (1.0 / (1ULL << 53))` where `v` is a 64-bit PCG64 output. When `u` must be strictly positive (as input to any transcendental function), the zero case is handled by resampling (consuming an additional PCG64 draw until a non-zero value is produced).
- **Bounded uniform integers:** For any integer-range random selections in neighbor proposal (FR-5, Phase 2 draws in FR-4), a bias-free bounded integer sampling algorithm is required. `std::uniform_int_distribution` is prohibited in reproducibility-critical paths (ADR-010 Decision 3). ADR-010 Decision 3 requires the specific algorithm to be named in this specification. The specific algorithm is an implementation planning decision dependent on the SA neighborhood structure (FR-2) and must be identified by name as part of OQ-2 resolution; FR-9 must be updated at that time.
- `std::normal_distribution` is not used by this backend.

**Floating-point determinism:** All floating-point computation in reproducibility-critical paths uses IEEE 754 double precision (ADR-010 Decision 6). Extended precision (x87 80-bit) must be disabled in reproducibility-critical code. The specific compiler flags are an implementation planning concern.

**PRNG draw ordering:** See FR-4 and FR-5 for the complete documentation of PCG64 draw ordering across Phase 1 (initial solution generation) and Phase 2 (annealing iterations). FR-4 documents the phase structure and inter-phase ordering; FR-5 documents the within-iteration ordering (neighbor proposal draws first, acceptance draw when ΔE > 0). The draw ordering is frozen once this specification is Accepted.

This section satisfies the documentation obligation stated in SPEC-004 FR-1 and SPEC-011 FR-6.5.

**Acceptance Criteria:**
- Two SolverRequests with identical routing problems and identical `execution_seed` values produce identical SolverResponse `solution` fields when both invocations produce `Succeeded`
- Two SolverRequests with identical routing problems and different `execution_seed` values may produce different `solution` fields
- The backend accepts any `execution_seed` value without error
- No entropy source other than `execution_seed` is used in reproducibility-critical paths
- PCG64 is seeded exactly once per execution from `execution_seed` with stream constant `0xcbbb9d5dc90c2383`

---

### FR-10: Supported SolverOutcome Values

**Description:**

This backend is classified `quantum_inspired_stochastic`. Per SPEC-011 FR-5.1, it must support `Succeeded`, `Timeout`, `Cancelled`, and `Failed`. It must not support `Infeasible`.

| Outcome | Supported | Trigger Condition |
|---|---|---|
| `Succeeded` | Yes | Annealing terminates naturally (temperature schedule or iteration budget exhausted before deadline) AND at least one complete, structurally valid RoutePlan was found during execution |
| `Timeout` | Yes | `execution_timeout_ms` deadline reached before natural annealing termination; backend self-terminates; best-so-far RoutePlan included if found before deadline |
| `Cancelled` | Yes | Worker cancellation signal received during annealing; backend terminates; best-so-far RoutePlan included if found before cancellation signal |
| `Failed` | Yes | Contract version mismatch; natural annealing termination with no valid solution found; internal error preventing execution completion |
| `Infeasible` | **Not Supported** | This backend cannot prove infeasibility. Prohibited per SPEC-011 FR-5.2. |

**Notes on Succeeded:** Succeeded is returned when annealing terminates naturally — the cooling schedule runs to its minimum temperature threshold, or the iteration budget is exhausted — before the `execution_timeout_ms` deadline. Natural termination before the deadline and the existence of a best-so-far solution are both required for Succeeded. If the deadline is reached first, Timeout is returned regardless of the annealing state.

**Notes on Timeout:** Timeout is the expected outcome for large problems where the annealing schedule does not complete within the `execution_timeout_ms` budget. The backend self-terminates when the deadline is approached (monitored at iteration boundaries) and returns the best-so-far solution. Unlike the construction heuristics (SPEC-013, SPEC-014), which return Timeout without a solution, this backend's anytime behavior means Timeout responses routinely include a complete solution. This is a core distinction of this backend.

**Notes on Cancelled:** Cancellation behavior mirrors Timeout: the best-so-far solution is included if one exists. The backend terminates upon receiving the Worker's cancellation signal. If cancellation arrives before any valid solution is produced, the response contains no solution.

**Notes on Failed:** Failed is returned for contract version mismatch (detected before processing begins), for natural annealing termination where no complete, structurally valid solution was ever found, and for any internal error preventing execution completion. A Failed response resulting from annealing completion with no valid solution is documented in FR-13.

This section satisfies the supported outcome declaration requirement in SPEC-011 FR-5.3.

**Acceptance Criteria:**
- The backend never returns `Infeasible`
- Timeout and Cancelled responses include a complete RoutePlan when and only when a best-so-far solution existed at the time of termination
- Succeeded is returned only when annealing terminates naturally before the deadline AND a best-so-far solution exists
- Failed with InternalError is returned when natural annealing termination produces no valid solution

---

### FR-11: RoutePlan Output Requirements

**Description:**

The QUBO SA backend produces a RoutePlan conforming to SPEC-004 FR-5. This section declares this backend's specific behavior relative to the framework RoutePlan obligations in SPEC-011 FR-7.1.

**RoutePlan presence per outcome:**

| Outcome | RoutePlan Present? | Requirements When Present |
|---|---|---|
| `Succeeded` | Required | All SPEC-004 FR-5 conditions 1–5 satisfied |
| `Timeout` | Present if best-so-far exists; Absent otherwise | All SPEC-004 FR-5 conditions 1–5 satisfied |
| `Cancelled` | Present if best-so-far exists; Absent otherwise | All SPEC-004 FR-5 conditions 1–5 satisfied |
| `Failed` | Absent | Never present |
| `Infeasible` | Not Applicable | Outcome not supported |

**SPEC-011 FR-7.1 narrowing for Timeout and Cancelled:** SPEC-011 FR-7.1 declares the RoutePlan as Optional under Timeout and Cancelled for all backends. This backend narrows that permission to "present when best-so-far exists, absent when it does not." This narrowing is the opposite direction from SPEC-013 (FR-6) and SPEC-014 (FR-10), which both narrow Timeout Optional→Absent: construction heuristics have no intermediate complete solutions and therefore always produce Timeout without a solution. This backend's anytime design means Timeout-with-solution is the expected outcome rather than an edge case. The framework's Optional permission deliberately accommodates both narrowing directions.

**Capacity validity guarantee when solution present:** The returned RoutePlan always satisfies SPEC-004 FR-5 condition 5 (total stop demand per route ≤ `capacity_per_vehicle`). Decoded states that violate capacity are not eligible as best-so-far candidates (FR-6). No post-return capacity validation is required; the selection criterion ensures this.

**Stop assignment completeness guarantee when solution present:** A RoutePlan present in any SolverResponse assigns every stop in the routing problem to exactly one route (SPEC-004 FR-5 conditions 3–4). Partial assignments — where some stops are unassigned — are never valid best-so-far candidates (SPEC-011 OQ-5, SPEC-004 FR-5). If the annealing process has only produced partially-assigned states at timeout, no solution is returned.

**Vehicle count guarantee when solution present:** When a RoutePlan is present, it contains exactly `vehicle_count` route entries. Unused vehicles have empty route sequences.

**Time window non-guarantee:** A RoutePlan present in any SolverResponse does not guarantee time window feasibility. This backend declares `supports_time_windows = false` (FR-8). Time window feasibility is evaluated by Core Quality Evaluation (SPEC-007).

`execution_duration_ms` is always populated. It reflects wall-clock execution time from backend execution start — including initial solution generation and all annealing iterations — to the point at which the SolverResponse is finalized. It excludes SolverRequest deserialization and SolverResponse serialization (SPEC-011 performance obligations).

**Acceptance Criteria:**
- Every RoutePlan present in any SolverResponse satisfies all SPEC-004 FR-5 conditions 1–5
- No SolverResponse contains a RoutePlan with unassigned stops
- `execution_duration_ms` is present in every SolverResponse
- No RoutePlan is present under Failed

---

### FR-12: Extension Metadata

**Description:**

The QUBO SA backend produces the following `extension_metadata` keys in the SolverResponse when a solution is present (Succeeded, or Timeout/Cancelled with best-so-far). When no solution is present (Failed, or Timeout/Cancelled without solution), extension_metadata is absent. All values are string-encoded.

| Key | Type (encoded as string) | Present When | Description |
|---|---|---|---|
| `qsa.iteration_count` | uint64 | Solution present | Total number of SA annealing iterations executed before termination. Includes all iterations regardless of whether they produced valid solutions. Does not count Phase 1 (initial solution generation). |
| `qsa.accepted_worse_transitions` | uint64 | Solution present | Total number of state transitions accepted during annealing where the proposed state had higher energy than the current state (ΔE > 0) and the Metropolis criterion draw was below `e^(-ΔE/T)`. Each accepted worsening transition increments this count by one. Rejected worsening transitions and accepted improving transitions are not counted. |
| `qsa.best_solution_iteration` | uint64 | Solution present | The iteration number (1-indexed) at which the returned best-so-far solution was last updated. A value of 0 indicates the best-so-far was set during Phase 1 (initial solution generation, before any annealing iteration). |
| `qsa.initial_temperature` | float64 | Solution present | The temperature parameter at the start of the annealing process (the temperature used in iteration 1). Encoded as a decimal string in IEEE 754 double precision. |
| `qsa.final_temperature` | float64 | Solution present | The temperature parameter at the last annealing iteration completed before termination. Encoded as a decimal string in IEEE 754 double precision. Must satisfy `final_temperature ≤ initial_temperature` (FR-3). |
| `qsa.solution_route_distance_km` | float64 | Solution present | Total Haversine route distance in kilometers of the returned best-so-far RoutePlan. Computed as the sum of Haversine distances for all vehicle routes, including each vehicle's depot-to-first-stop leg and last-stop-to-depot leg. This is the solver's internal quality metric, not the normalized quality metric used by Core (SPEC-007). |

Extension metadata is absent when no solution is present (Failed outcomes, and Timeout/Cancelled outcomes without a best-so-far).

**Departure from construction heuristic precedent:** SPEC-013 (nearest-neighbor) and SPEC-014 (greedy insertion) produce `extension_metadata` only on Succeeded responses. This specification produces `extension_metadata` on Timeout-with-solution and Cancelled-with-solution responses as well. This departure is deliberate. For SA, Timeout-with-solution is an expected outcome path on large problems, not an edge case. Restricting `extension_metadata` to Succeeded would suppress the annealing diagnostic evidence (temperature values, acceptance rates, iteration counts) precisely on the outcome paths where it is most diagnostically valuable.

`extension_metadata` must not contain routing problem raw data (geographic coordinate arrays, stop identifier lists, demand arrays, or other raw problem input fields). The six keys above contain only derived summary values safe for evidence log inclusion (SPEC-004 FR-13, SPEC-001 Security Considerations).

Consumers must silently ignore any keys not listed here. Unrecognized keys are not contract-stable across `specification_version` changes.

**Acceptance Criteria:**
- All six keys are present in the `extension_metadata` of every SolverResponse where a solution is present
- Key values are non-empty strings encoding valid numeric values in the specified types
- `qsa.final_temperature` ≤ `qsa.initial_temperature` in all responses
- No key contains routing problem raw data (coordinates, demands, stop lists)
- Extension metadata is absent on responses without a solution

---

### FR-13: Failure Model

**Description:**

This section defines all failure conditions specific to the QUBO SA backend. General framework failure obligations are defined in SPEC-011 FR-8.

**Contract version mismatch:**

- **Trigger:** The `contract_version` in the SolverRequest does not equal 1.
- **Outcome:** `Failed`, `failure_code = ContractVersionMismatch`
- **Behavior:** The backend returns `ContractVersionMismatch` immediately before any processing begins — before initial solution generation, before any PCG64 seeding. No stochastic computation occurs.
- **Metadata:** `failure_message` must identify the received version and the expected version (1).

**Natural termination with no valid solution:**

- **Trigger:** The annealing schedule or iteration budget is exhausted before the `execution_timeout_ms` deadline, AND no complete, structurally valid RoutePlan was found during Phase 1 or Phase 2.
- **Outcome:** `Failed`, `failure_code = InternalError`
- **Behavior:** The backend returns Failed with a diagnostic failure message. No RoutePlan is present. Extension metadata is absent.
- **Metadata:** `failure_message` must identify the failure cause ("qubo-simulated-annealing: annealing completed without producing a valid route plan") and the problem characteristics that may have contributed (stop count, vehicle count, capacity utilization if computable without additional PRNG draws).
- **Note:** This condition is expected to be rare for well-parameterized SA on typical CVRP instances. It may occur for extreme problem parameters (very high capacity utilization, very tight vehicle constraints) where the penalty terms in the QUBO formulation do not guide the search toward feasible binary states. This is a solver quality defect, not a contract violation. It should trigger investigation of the QUBO formulation and penalty parameters.

**Internal error:**

- **Trigger:** Any unexpected condition preventing algorithm execution or RoutePlan production (for example, arithmetic errors in distance computation producing NaN or infinite values, failed PRNG initialization, or memory allocation failure).
- **Outcome:** `Failed`, `failure_code = InternalError`
- **Behavior:** The backend returns `Failed` immediately. No RoutePlan is present. Extension metadata is absent.
- **Metadata:** `failure_message` must provide a diagnostic description identifying the failure cause.

**Timeout:**

- **Trigger:** `execution_timeout_ms` deadline reached.
- **Outcome:** `Timeout`, `failure_code = ExecutionTimeout`
- **Behavior:** The backend MUST monitor the deadline at iteration boundaries and self-terminate when the deadline is approached. Self-termination is a stronger obligation for this backend than the advisory in SPEC-004 FR-10.1: if the Worker enforces an external timeout because the backend did not self-terminate, the Worker-constructed Timeout response contains no solution and no extension metadata, forfeiting the anytime behavior value of this backend. On self-termination, the backend returns Timeout with the best-so-far solution (if found) and extension metadata.
- **Self-termination monitoring:** The backend MUST check elapsed time at the boundary between annealing iterations. When the remaining time before the deadline falls below a threshold sufficient to finalize the response, the backend terminates the current iteration and begins response construction. The specific threshold is an implementation planning decision.

**Cancellation:**

- **Trigger:** Worker cancellation signal received during execution (Phase 1 or Phase 2).
- **Outcome:** `Cancelled`, `failure_code = ExecutionCancelled`
- **Behavior:** The backend terminates upon receiving the cancellation signal and returns the best-so-far solution (if found) and extension metadata. If cancellation arrives before any valid solution is produced — including during Phase 1 before the initial solution construction completes — the response contains no solution and no extension metadata. The best-so-far tracking rule (FR-6) applies at all phases: if a complete, structurally valid RoutePlan was produced during Phase 1 before the cancellation signal arrived, it is included as the best-so-far.

All `Failed` responses include a `SolverFailureDetail` with both `failure_code` and `failure_message` populated. The `failure_message` is diagnostic (developer-readable), not user-facing. The backend does not write to stdout or stderr.

**Acceptance Criteria:**
- `ContractVersionMismatch` is returned before any processing begins (including PCG64 seeding) on version mismatch
- `InternalError` with the prescribed message is returned when natural annealing termination produces no valid solution
- `failure_message` is non-empty on every Failed response
- No RoutePlan is present in any Failed response
- Timeout self-termination is monitored at iteration boundaries

---

# Performance Characteristics

The QUBO SA backend's execution cost scales with the problem size (number of stops and vehicles) and the iteration budget. The following estimates are derived from algorithmic analysis, not empirical measurement.

**Initial solution generation (Phase 1):** O(N × V) cost where N is stop count and V is vehicle count, reflecting the binary state initialization. PCG64 draws are consumed in O(N) bounded integer samples for stop placement.

**Per-iteration cost (Phase 2):** O(N × V) cost per iteration for evaluating the QUBO energy delta of a proposed binary state change. For large problems (76+ stops, 10+ vehicles), per-iteration cost may be significant. The total execution cost is O(K × N × V) where K is the iteration budget.

**Expected execution duration per size class:**

| Size Class | Stop Range | Expected Duration | Basis |
|---|---|---|---|
| Small | 1–25 stops | < 1.0s | Algorithmic analysis; small state dimension limits per-iteration cost |
| Medium | 26–75 stops | < 10.0s | Algorithmic analysis; O(N × V) per-iteration cost with moderate iteration budget |
| Large | 76+ stops | < 60.0s | Algorithmic analysis; empirical validation required |

These durations correspond to the `latency_profile` values in the capability profile (FR-8). All values are pre-implementation estimates. Empirical measurement on SPEC-002 synthetic workload problems of each size class is required before `is_provisional` is set to false. Measurements must use `execution_duration_ms` as defined (including Phase 1).

**Memory:** The dominant memory allocation is the binary state vector and QUBO matrix, both O((N × V)²) in the general case. For Large problems (76+ stops, 10+ vehicles), the QUBO matrix dimension requires attention during implementation planning.

**Expected solution quality:**

SA's optimization capability scales with the iteration budget relative to the solution space size. For Small problems, SA may underperform or match construction heuristics due to the overhead of global search. For Medium and Large problems, SA's ability to escape local optima produces consistently better solutions than greedy construction. The expected quality improvement over construction heuristics is problem-structure-dependent and is the primary subject of the evidence thesis.

---

# Quality Expectations

**Route quality expectations:** The QUBO SA backend is expected to produce Competitive-quality routes — at or above the quality of the greedy insertion backend (SPEC-014) on most problem instances, with larger improvements observed at larger problem sizes and higher capacity utilization ratios where construction heuristics are most constrained.

**Workload classes where SA performs better than construction heuristics:**
- Larger problem sizes (Medium, Large) where the iteration budget can explore more of the solution space
- High-density geographic clusters where construction heuristics make poor early choices
- High capacity utilization problems where route interleaving and global assignment yield better solutions than sequential greedy construction

**Workload classes where SA may not significantly outperform construction heuristics:**
- Small problems (1–25 stops) where the construction heuristics are near-optimal by chance and the search space is small
- Problems with uniform stop distribution and moderate capacity utilization where greedy construction already produces near-optimal results
- Very short execution budgets where the annealing schedule cannot explore sufficient iterations to escape local optima

**Evidence thesis implications:** The evidence system is designed to capture exactly this variation. The quality comparison (SPEC-007) and regret analysis (SPEC-003) across the three MVP backends on identical SPEC-002 synthetic workloads will produce the empirical evidence base for the system's thesis: "most routing problems should never touch a quantum computer."

---

# Non-Requirements

- This specification does not define the QUBO energy function, penalty weights, or binary encoding scheme. Those are implementation planning decisions (FR-2).
- This specification does not define the SA neighborhood structure (specific move operator). That is an implementation planning decision (FR-2, FR-5).
- This specification does not define the annealing schedule parameters (initial temperature, cooling rate, minimum temperature, iteration budget). Those are implementation planning decisions (FR-3).
- This specification does not define the binary solution decoding algorithm. That is an implementation planning decision (FR-2).
- This specification does not define the repair heuristic for infeasible decoded states. That is an implementation planning decision (FR-2).
- This specification does not define time window feasibility checking. Time windows are evaluated by Core Quality Evaluation (SPEC-007).
- This specification does not define multi-depot behavior. Single depot only, per SPEC-001 FR-3.
- This specification does not define heterogeneous vehicle fleet handling. All vehicles have identical capacity, per SPEC-001 FR-2.
- This specification does not define how workload features are computed. That is SPEC-010's responsibility.
- This specification does not define backend selection policy. That is SPEC-003's responsibility.
- This specification does not define evidence persistence. That is SPEC-006's responsibility.
- This specification does not define quality evaluation metrics. That is SPEC-007's responsibility.
- This specification does not alter the SolverContract schema. That is SPEC-004's responsibility.
- This specification does not define quantum hardware execution. Quantum hardware is deferred per ADR-007.

---

# Assumptions

1. The routing problem received by this backend has been validated by Core per ADR-009. The backend does not re-validate the routing problem's structural constraints. Coordinates are within valid ranges, demands are non-negative, and total demand does not exceed total fleet capacity.

2. The QUBO formulation can encode the CVRP capacity constraints as penalty terms such that low-energy binary states correspond to capacity-valid route assignments. If the penalty weighting is incorrect, the SA process may converge to high-energy feasible states or low-energy infeasible states. This is a formulation quality risk addressed during implementation planning.

3. The SA process can produce at least one complete, capacity-valid RoutePlan for typical SPEC-002 synthetic workload problems within the allocated `execution_timeout_ms` budget. If no valid solution is ever produced, the backend returns Failed (FR-13). The risk that the annealing process never reaches a valid feasible state on typical problem instances is low for well-parameterized SA, but must be confirmed during implementation and benchmarking.

4. Anytime behavior is achievable: at the `execution_timeout_ms` deadline, the backend can finalize and return its best-so-far RoutePlan without additional significant computation. The response finalization overhead (encoding the RoutePlan from the best-so-far state and populating extension_metadata) is negligible relative to the iteration budget.

5. The PCG64 stream constant `0xcbbb9d5dc90c2383` is statistically independent from the stream constant used by SPEC-002 (the Synthetic Workload Generator). Independence prevents correlation between generated problem instances and solver PRNG sequences when the same problem seed drives both. This assumption must be confirmed during implementation planning (OQ-1).

6. All MVP backends are executed within the same C++17 toolchain and Docker Compose Linux environment established by ADR-001. The IEEE 754 double precision and PCG64 cross-platform reproducibility guarantees of ADR-010 apply in this environment.

7. The PCG64 reference implementation (pcg-random.org or equivalent header-only library) is adoptable as a project dependency without licensing conflict, consistent with ADR-010 Assumptions.

---

# Constraints

1. This backend implements SPEC-004 contract version 1. `contract_version` mismatch must be detected before any processing begins, including before PCG64 seeding (SPEC-011 FR-8.3).

2. PCG64 is the exclusive PRNG for all reproducibility-critical stochastic operations. No other PRNG may be used in reproducibility-critical paths (ADR-010 Decision 1, SPEC-011 FR-6.2).

3. `execution_seed` is the exclusive authorized entropy source. System time, process ID, OS random sources, and hardware entropy are prohibited in reproducibility-critical paths (ADR-010 Decision 4).

4. The PRNG must be seeded exactly once per execution from `execution_seed`. Reseeding during execution is prohibited (ADR-010 Decision 4).

5. PCG64 stream constant `0xcbbb9d5dc90c2383` is frozen once this specification is Accepted. Changing it requires the full ADR-010 Decision 5 breaking change procedure. A `specification_version` increment alone is insufficient.

6. `std::uniform_int_distribution` and `std::normal_distribution` are prohibited in reproducibility-critical paths (ADR-010 Decision 3).

7. No backend-specific logic may exist in the Scheduler, Worker, or Core contract-handling code. This backend is accessed exclusively through the normalized SolverContract interface (ADR-008).

8. `extension_metadata` must not contain routing problem raw data (geographic coordinate arrays, stop lists, demand arrays). Only derived summary values may appear (SPEC-004 FR-13, SPEC-011 FR-7.4).

9. The backend must not emit OpenTelemetry spans, metrics, or structured logs directly. Diagnostic output is returned through `SolverFailureDetail.failure_message` and `extension_metadata` only (SPEC-011 FR-9.2).

10. The backend must not write to stdout or stderr (SPEC-011 FR-9.2).

11. The backend must not return `Infeasible` under any condition (SPEC-011 FR-5.2).

12. Any RoutePlan included in a SolverResponse — under any outcome — must satisfy all SPEC-004 FR-5 structural validity requirements. Partial assignments are never valid.

---

# Inputs

**SolverRequest** (SPEC-004 FR-2): The complete input to this backend. Relevant fields:

| Field | How Used |
|---|---|
| `routing_problem.stops` | Stop coordinates, demands, stop_id values — all used in QUBO formulation and solution decoding |
| `routing_problem.depot` | Depot coordinates — used in route distance computation for best-so-far quality comparison |
| `routing_problem.vehicle_count` | Determines number of routes in the decoded RoutePlan; part of QUBO binary state dimension |
| `routing_problem.capacity_per_vehicle` | Enforced as a constraint via penalty encoding; decoded solution must satisfy SPEC-004 FR-5 condition 5 |
| `execution_timeout_ms` | Deadline for execution; monitored at iteration boundaries for self-termination |
| `execution_seed` | Seeded into PCG64 with stream constant `0xcbbb9d5dc90c2383` before Phase 1 begins |
| `contract_version` | Validated before any processing; must equal 1 |
| `job_id`, `decision_id`, `backend_id` | Correlation only; do not affect algorithm execution or route construction |

Fields not used in optimization: `routing_problem.time_window_open`, `routing_problem.time_window_close`, `routing_problem.service_duration` (see FR-7).

---

# Outputs

**SolverResponse** (SPEC-004 FR-3): The complete output of this backend.

On `Succeeded`:
- `outcome = Succeeded`
- `solution`: Best-so-far RoutePlan with `vehicle_count` routes, all stops assigned, capacity valid
- `statistics.execution_duration_ms`: Wall-clock backend execution time in milliseconds, including Phase 1 and all Phase 2 iterations
- `statistics.solution_count`: Number of times the best-so-far was updated during execution
- `extension_metadata`: All six keys (FR-12)
- `failure`: Absent

On `Timeout` with best-so-far:
- `outcome = Timeout`
- `failure.failure_code = ExecutionTimeout`; `failure.failure_message`: Diagnostic
- `solution`: Best-so-far RoutePlan satisfying all SPEC-004 FR-5 requirements
- `statistics.execution_duration_ms`: Wall-clock time from execution start to termination
- `statistics.solution_count`: Number of best-so-far updates before termination
- `extension_metadata`: All six keys (FR-12)

On `Timeout` without best-so-far:
- `outcome = Timeout`
- `failure.failure_code = ExecutionTimeout`; `failure.failure_message`: Diagnostic
- `solution`: Absent
- `extension_metadata`: Absent

On `Cancelled` (mirrors Timeout structure; solution and extension_metadata present if best-so-far exists):
- `outcome = Cancelled`
- `failure.failure_code = ExecutionCancelled`

On `Failed`:
- `outcome = Failed`
- `failure.failure_code`: `ContractVersionMismatch` or `InternalError`
- `failure.failure_message`: Diagnostic description
- `solution`: Absent
- `extension_metadata`: Absent

Consumer: The Worker receives the SolverResponse, validates structural requirements (SPEC-004 FR-5), and persists it to the evidence log. Core evaluates route quality from the RoutePlan (SPEC-007). The `extension_metadata` passes through to the evidence record.

---

# Architectural Impact

| Component | Impact | Notes |
|---|---|---|
| Domain Layer (C++ Core / backends) | Yes | SPEC-015 defines a new C++ SolverContract implementation; first `quantum_inspired_stochastic` backend |
| Scheduler (SPEC-003) | Yes | Capability profile must be registered before backend is eligible for selection |
| Solver Contract (SPEC-004) | None | SPEC-015 satisfies SPEC-004 FR-1 obligations; no schema changes |
| Worker (SPEC-005) | None | No lifecycle changes; standard invocation, timeout, and cancellation handling applies; anytime solution inclusion is a backend behavior, not a Worker behavior change |
| Evidence Log (SPEC-006) | None | Evidence persistence is unchanged; new `extension_metadata` keys pass through existing JSONB column |
| Quality Evaluation (SPEC-007) | None | Core evaluates quality from normalized RoutePlan; no changes required; Timeout-with-solution responses are evaluated the same as Succeeded responses |
| Feature Extraction (SPEC-010) | None | Workload features are computed before backend selection; no changes |
| Persistence Schema (SPEC-012) | None | No new tables; `extension_metadata` stored per existing schema |
| Reproducibility Pipeline (ADR-010) | Yes | First stochastic backend validates the ADR-010 pipeline end-to-end: `execution_seed` derivation (SPEC-005 FR-7), PCG64 seeding, draw ordering, semantic equivalence claim |
| API Layer | None | No new API surface |
| Observability | None | Worker-emitted `solver.execute` span (SPEC-004 FR-15); `outcome = Timeout` with solution present sets span status to Unset (not Error) per SPEC-004 FR-15 span status rules |
| Security | None | No new trust boundaries; `extension_metadata` raw-data prohibition applies per FR-12 |
| Configuration | Yes | Capability profile must be registered through the mechanism resolved by SPEC-003 OQ-2 |

**SPEC-011 FR-11:** The MVP backend inventory lists `qubo-simulated-annealing` as "Required; not yet written." SPEC-015 satisfies this prerequisite. SPEC-011 FR-11 should be updated to reflect this specification's status.

**SPEC-004 FR-15 span status note:** When the Worker emits a `solver.execute` span for a Timeout response that includes a solution, the span status is `Unset` (not Error) per SPEC-004 FR-15: "outcome = Timeout or Cancelled with solution present: span status Unset — the solver produced a usable result within the execution boundary." This is the correct behavior and requires no change to SPEC-004 or the Worker; it is noted here because Timeout-with-solution is the expected outcome path for this backend on large problems.

---

# Testability

The following behaviors must be verified. Specific test implementations are determined during implementation planning.

**Contract conformance (inherited from SPEC-004):**

1. **Structural validity:** A Succeeded SolverResponse contains a RoutePlan where: (a) `routes.size()` = `vehicle_count`; (b) every stop_id appears exactly once across all routes; (c) no route's total demand exceeds `capacity_per_vehicle`; (d) no route contains the depot.

2. **Reproducibility:** Two SolverRequests with identical routing problems and identical `execution_seed` values produce identical `outcome` and `solution` fields when both produce Succeeded.

3. **Seed sensitivity:** Two SolverRequests with identical routing problems and different `execution_seed` values may produce different `solution` fields. (This validates that the PRNG is actually consuming `execution_seed`, not ignoring it.)

4. **Seed acceptance:** The backend accepts any `execution_seed` value without error.

5. **ContractVersionMismatch:** A SolverRequest with `contract_version` != 1 returns `Failed` with `ContractVersionMismatch` before any processing, including before PCG64 seeding.

6. **Trivial case:** A routing problem with one stop and one vehicle produces a Succeeded response with a single route containing that stop.

7. **Empty vehicle routes:** A routing problem with more vehicles than stops produces a Succeeded response where some routes are empty.

**Failure model:**

8. **Natural termination with no valid solution:** Given a routing problem and execution parameters calibrated to exhaust the iteration budget before the `execution_timeout_ms` deadline without producing any complete, structurally valid RoutePlan, the backend returns `Failed` with `failure_code = InternalError` and a `failure_message` containing "qubo-simulated-annealing: annealing completed without producing a valid route plan". Verifies the FR-13 natural-termination-with-no-valid-solution failure path.

**Stochastic algorithm correctness:**

9. **Anytime behavior — Timeout with solution:** Given a SolverRequest with a `execution_timeout_ms` set to a value that allows at least one complete valid solution to be found but not the full annealing schedule, the backend returns `Timeout` with a complete, structurally valid RoutePlan.

10. **Anytime behavior — solution quality improves with time:** Given the same problem and seed with increasing `execution_timeout_ms` values, the `qsa.solution_route_distance_km` in the response does not increase. A longer execution budget should not produce a worse solution than a shorter one (monotone non-worsening).

11. **Best-so-far update tracking:** `solution_count` equals the number of times the best-so-far solution was updated. Verified by comparing `qsa.best_solution_iteration` against total `qsa.iteration_count` and checking monotone improvement.

12. **Metropolis acceptance:** `qsa.accepted_worse_transitions` is non-negative and no greater than `qsa.iteration_count`. At sufficiently high initial temperature (early iterations), `qsa.accepted_worse_transitions / qsa.iteration_count` should be close to the theoretical acceptance rate.

13. **Temperature monotonicity:** `qsa.final_temperature` ≤ `qsa.initial_temperature` in all responses that include extension_metadata.

14. **No partial assignments:** No Succeeded, Timeout, or Cancelled response contains a RoutePlan with unassigned stops. Every stop in the routing problem appears in exactly one route in every RoutePlan present.

15. **Capacity validity across workload:** For all SolverResponses with a RoutePlan across the full SPEC-002 synthetic workload, no route has total demand exceeding `capacity_per_vehicle`.

**Reproducibility pipeline:**

16. **PCG64 stream constant:** Two executions with the same problem and seed produce the same `qsa.iteration_count`, `qsa.accepted_worse_transitions`, and `qsa.solution_route_distance_km` values. This verifies that the PRNG stream is determined by `execution_seed` and the frozen stream constant.

17. **Draw ordering phase boundary:** Phase 1 draws are completed before Phase 2 begins. Verified by demonstrating that changing the initial solution construction (Phase 1) while holding Phase 2 constant changes the output, and vice versa, when draw ordering is validated against the specification.

**Capability profile accuracy:**

18. **Latency profile validation:** Empirical execution duration on representative SPEC-002 problems of each size class — measured as `execution_duration_ms` — is within the declared `latency_profile` bounds (< 1.0s, < 10.0s, < 60.0s per size class). Required before `is_provisional` is set to false.

19. **Quality classification:** Route quality (measured by Core's normalized metric, SPEC-007) on SPEC-002 benchmark problems is consistent with `Competitive` classification relative to the classical construction heuristic baselines. Required before `is_provisional` is set to false.

**Extension metadata:**

20. **Key presence on solution:** All six `extension_metadata` keys are present when a solution is present; extension_metadata is absent when no solution is present.

21. **qsa.solution_route_distance_km accuracy:** The reported distance matches the sum of Haversine distances computed from the route sequences in the RoutePlan, including depot-return legs.

22. **Temperature relation:** `qsa.final_temperature ≤ qsa.initial_temperature` in all responses.

**Observability:**

23. **solver.execute span:** A `solver.execute` OTel span is emitted by the Worker for every SolverRequest dispatch, successful or not. All required attributes (SPEC-004 FR-15) are present. Span status is `Unset` for Timeout-with-solution (not Error). Verified in integration test.

---

# Observability Requirements

The `solver.execute` span is emitted by the Worker for every invocation of this backend (SPEC-004 FR-15). The backend's contribution is through SolverResponse fields:

| Span Attribute | Source |
|---|---|
| `outcome` | `SolverResponse.outcome` |
| `execution_duration_ms` | `SolverResponse.statistics.execution_duration_ms` |

All other `solver.execute` span attributes are available to the Worker from the SolverRequest, routing problem, or workload features.

**Diagnostic questions this backend must enable through `extension_metadata`:**

1. Did the annealing process run for a sufficient number of iterations? (`qsa.iteration_count`)
2. Was the annealing temperature high enough to escape local optima? (`qsa.accepted_worse_transitions` relative to `qsa.iteration_count`)
3. When was the best solution found — early or late in the annealing process? (`qsa.best_solution_iteration` relative to `qsa.iteration_count`)
4. How much did the temperature drop? (`qsa.initial_temperature`, `qsa.final_temperature`)
5. What is the solver's internal quality measure for the returned solution? (`qsa.solution_route_distance_km`)

These values pass through the Worker to the evidence log, where they are available for backend-specific reporting. The SPEC-009 evidence report's Solver Execution section surfaces these values for the annealing evidence narrative.

**No additional instrumentation:** The backend does not emit OpenTelemetry spans, metrics, or structured logs. Diagnostic output is confined to `SolverFailureDetail.failure_message` on failure and `extension_metadata` on success (or Timeout/Cancelled with solution), per SPEC-011 FR-9.2.

The backend does not write to stdout or stderr.

---

# Security Considerations

**Seed authority (ADR-010 Decision 4):** All stochastic operations use PCG64 seeded from `execution_seed`. No system entropy source is consulted. Using `std::random_device`, `/dev/urandom`, system time, or process ID in reproducibility-critical paths would cause non-reproducible outputs that undermine evidence integrity.

**Extension metadata safety:** The six `extension_metadata` keys contain only derived scalar summary values. No key contains geographic coordinate arrays, stop identifier lists, demand arrays, or any other raw routing problem input. This complies with the raw-data prohibition in SPEC-004 FR-13 and SPEC-001 Security Considerations. The annealing diagnostic values (iteration counts, temperatures, distances) are summary statistics, not raw data.

**Input trust:** The routing problem arrives at the backend pre-validated by Core (ADR-009). The backend trusts the problem's structural validity. The backend does not re-validate the routing problem's coordinates, demands, or fleet parameters.

**No external access:** The backend has no access to external systems, networks, files, or queues. It operates exclusively on the in-process SolverRequest data.

**No stdout/stderr:** Diagnostic output is confined to the SolverResponse contract fields, preventing routing problem data from escaping through uncontrolled channels.

---

# Performance Considerations

**QUBO matrix dimension:** The binary state space dimension scales with stop count and vehicle count. For Large problems (76+ stops, 10+ vehicles), the QUBO matrix dimension can become significant. The specific state representation and its dimension are implementation planning decisions (FR-2). The memory profile of the QUBO matrix must be measured during implementation and confirmed acceptable before `is_provisional` is set to false.

**Per-iteration cost:** Each iteration evaluates the energy delta of a proposed neighbor. For dense QUBO matrices, this is O(D) where D is the binary state dimension. Total execution cost is O(K × D) for K iterations. Implementation planning should balance K (iteration budget) and D (state dimension) to fit within the `execution_timeout_ms` budget at each size class.

**Timeout self-termination overhead:** The backend monitors the deadline at iteration boundaries. The overhead of this check (a clock read per iteration) must be negligible relative to the iteration cost. For long iterations (large D), per-iteration monitoring is not expensive; for very short iterations, it may add overhead that should be measured.

**Distance computation:** Route distance computation for best-so-far updates uses Haversine (SPEC-001 FR-5). For N stops and V vehicles, computing the route distance of a decoded RoutePlan is O(N). This is called at most once per iteration that produces a valid decoded solution. For typical iteration counts and problem sizes, this is not a performance bottleneck.

Do not invent specific latency targets. The areas above require measurement during implementation. See Performance Characteristics for pre-implementation estimates.

---

# Documentation Updates Required

The following follow-on updates are required as a result of this specification. No updates are performed here.

**SPEC-003:**
- The capability profile for `qubo-simulated-annealing` must be registered through the mechanism resolved by SPEC-003 OQ-2 before this backend becomes eligible for Scheduler selection. No SPEC-003 schema change is required; the capability profile fields defined in FR-8 conform to the existing SPEC-003 FR-4 schema.

**SPEC-011:**
- FR-11.1 MVP backend inventory: Update the `qubo-simulated-annealing` row's "Solver Specification Status" from "Required; not yet written" to reflect SPEC-015's current status.
- OQ-1 (PCG64 Stream Constants): SPEC-015 proposes `0xcbbb9d5dc90c2383` as the stream constant. When OQ-1 is resolved and the stream constant is confirmed, SPEC-011 OQ-1 may be updated to reference SPEC-015's resolution. No SPEC-011 content change is required; SPEC-011 OQ-1 defers the specific value to the individual solver specification.

**SPEC-009:**
- No structural changes required. The Solver Execution section of the evidence report surfaces `extension_metadata` values for solver runs. For SA invocations, the qsa.* extension_metadata keys provide annealing diagnostic evidence (temperature values, acceptance rate proxy via `qsa.accepted_worse_transitions`, best-solution iteration) that is specifically valuable for the evidence narrative comparing stochastic and deterministic backends. Report generation logic should surface these keys for SA-specific evidence reporting.

**SPEC-012:**
- No schema changes are required. `extension_metadata` for `qubo-simulated-annealing` passes through the existing JSONB column structure in `solver_run_records`. The six extension metadata keys defined in FR-12 are stored within the existing schema.

**ADR-010:**
- No changes required. ADR-010 already governs `quantum_inspired_stochastic` backends. SPEC-015 satisfies ADR-010 obligations by documenting the stream constant, draw ordering phases, approved distribution algorithms, and seeding procedure. The ADR-010 Architectural Impact section notes that the QUBO SA backend specification references this ADR; SPEC-015's acceptance constitutes the first concrete realization of ADR-010 for a stochastic solver backend.

**docs/architecture.md:**
- No changes required. SPEC-015 conforms to the architecture principles and required observability spans. The QUBO simulated annealing backend is already identified in architecture.md as an MVP stochastic backend.

---

# Open Questions

### OQ-1: PCG64 Stream Constant Confirmation

**Classification:** Implementation Planning Decision (blocks Proposed status)

**Question:** Is `0xcbbb9d5dc90c2383` an appropriate PCG64 stream constant for the `qubo-simulated-annealing` backend, and is it confirmed to be statistically independent from the stream constant used by SPEC-002 (Synthetic Workload Generator)?

**Context:** ADR-010 Decision 1 requires each stochastic component to document a unique stream constant, frozen once the specification is Accepted. Statistical independence from other components' stream constants prevents correlation in PRNG output sequences when the same problem seed is used across multiple components (e.g., the workload generator and the solver both processing a problem derived from the same root seed). SPEC-011 OQ-1 identifies this as a per-backend implementation planning decision. The value `0xcbbb9d5dc90c2383` is proposed in this Draft specification. It must be confirmed to differ from SPEC-002's stream constant and from any other registered stochastic component's stream constant before this specification advances to Proposed.

**Resolution required before:** This specification advances from Draft to Proposed. The stream constant is frozen at Accepted; it cannot be changed after acceptance without the full ADR-010 Decision 5 breaking change procedure.

**Blocking:** Blocks advancement to Proposed. Does not block engineering review of this Draft.

---

### OQ-2: PRNG Draw Ordering — Per-Iteration Count, Phase 1 Ordering, and Bounded Integer Algorithm

**Classification:** Implementation Planning Decision (blocks Proposed status)

**Question:** Three draw ordering items require resolution once the SA neighborhood structure and initial solution construction strategy are selected during implementation planning:

1. What is the exact number of PCG64 draws consumed per annealing iteration for the selected neighborhood structure?
2. What is the specific draw ordering within Phase 1 (initial solution generation): stop-centric (processing stops in a deterministic order), binary-variable-centric (initializing binary state variables in index order), or another approach?
3. What is the specific bias-free bounded integer sampling algorithm used for neighbor proposal draws, as required by ADR-010 Decision 3?

**Context:** SPEC-011 FR-6.2 requires each stochastic solver specification to document its PRNG draw ordering — "the fixed sequence in which PCG64 draws are consumed during a solver execution." FR-4 of this specification documents the two-phase structure (Phase 1: initial solution generation; Phase 2: annealing iterations) and the ordering within Phase 2 (neighbor proposal draws first, acceptance draw second when ΔE > 0). Three items remain deferred pending implementation planning decisions:

- **Per-iteration draw count:** The exact number of draws consumed per annealing iteration depends on the specific SA neighborhood structure (the move operator). A move that selects two stops for a swap consumes two bounded integer draws; a move that selects a stop and a destination vehicle consumes two draws of different types. Until the neighborhood structure is chosen, the per-iteration draw count cannot be specified with precision.

- **Phase 1 draw ordering:** The initial solution construction strategy determines the Phase 1 draw ordering. Because the QUBO binary encoding is an implementation planning decision (FR-2), the Phase 1 draw ordering cannot be fixed until the encoding and construction strategy are selected. A stop-centric strategy processes stops in some deterministic order; a binary-variable-centric strategy initializes binary state variables in index order. The choice between strategies must be explicitly specified before the full PRNG draw sequence is frozen.

- **Bounded integer sampling algorithm:** ADR-010 Decision 3 requires the specific bias-free bounded integer algorithm to be named in this specification. The algorithm choice may depend on the neighborhood structure. Common choices include Lemire's nearly-divisionless method (Lemire 2019) and rejection-sampling with power-of-two masking.

**Resolution required before:** This specification advances from Draft to Proposed. OQ-2 resolution must include: (1) an explicit per-iteration draw enumeration (e.g., "two bounded integer draws for neighbor proposal, then one uniform float draw for acceptance when ΔE > 0"); (2) a complete description of Phase 1 draw ordering; and (3) the specific named bounded integer sampling algorithm. FR-4 and FR-9 must be updated with this information at resolution time.

**Blocking:** Blocks advancement to Proposed. Does not block engineering review of this Draft.

---

### OQ-3: Anytime Behavior Implementation Risk

**Classification:** Implementation Planning Decision (must be validated during implementation)

**Question:** Can the SA backend reliably produce at least one complete, structurally valid RoutePlan for typical SPEC-002 synthetic workload problems within a reasonable portion of the `execution_timeout_ms` budget?

**Context:** SPEC-004 Assumption 6 notes: "a partial annealing state may not decode into a complete, feasible route plan, in which case the backend returns Timeout or Cancelled with no solution. This risk must be validated and explicitly addressed during QUBO backend specification." If the QUBO formulation's penalty terms do not guide the search toward feasible binary states quickly, the backend may frequently reach Timeout without ever producing a valid best-so-far. A backend that routinely returns Timeout-without-solution offers no advantage over the construction heuristics in the evidence comparison. The expected ratio of Timeout-with-solution to Timeout-without-solution on SPEC-002 problems is unknown until empirical testing.

**Resolution required before:** `is_provisional` can be set to false. If empirical testing reveals that the backend frequently fails to produce valid solutions, the QUBO formulation (penalty weights, encoding strategy) must be revised. This may require a `specification_version` increment if it changes observable behavior (FR-2 formulation ownership) or the extension metadata semantics.

**Blocking:** Does not block SPEC-015 acceptance. Informs whether `is_provisional` can be set to false and whether the `quality_profile` declaration of `Competitive` is accurate.

---

### OQ-4: Capability Profile Empirical Latency and Quality Values

**Classification:** Implementation Planning Decision

**Question:** What are the empirically measured latency values per size class and confirmed quality classification for the `qubo-simulated-annealing` backend?

**Context:** FR-8 declares `is_provisional = true` with pre-implementation latency estimates and a literature-based quality classification. Empirical measurement against the SPEC-002 synthetic workload is required before `is_provisional` can be set to false. The specific benchmark methodology, number of representative problems per size class, and statistical confidence requirements are implementation planning decisions. Latency measurements must use `execution_duration_ms` as defined (including Phase 1).

**Resolution required before:** Setting `is_provisional = false` in the capability profile. This affects whether the backend is excluded from certain Scheduler eligibility phases (SPEC-003 FR-5 Phase 2).

**Blocking:** Does not block SPEC-015 acceptance or initial implementation. Blocks `is_provisional = false` declaration.

---

### OQ-5: Failed Outcome on No Valid Solution — Frequency Threshold

**Classification:** Project Owner Decision Required

**Question:** If the annealing process terminates naturally without ever producing a valid RoutePlan, the backend returns `Failed` with `InternalError` (FR-13). At what frequency of this failure on the SPEC-002 synthetic workload does it indicate a QUBO formulation defect requiring revision, versus an acceptable edge case?

**Context:** FR-13 documents this failure condition. Whether it is rare (an edge case for extreme problem parameters) or common (a systemic defect in the QUBO formulation) depends on the formulation quality. No threshold is defined in this specification. A Project Owner decision is required to set the acceptable failure rate, against which empirical measurement during implementation testing can be compared.

**Resolution required before:** `is_provisional` can be set to false. A failure rate above the accepted threshold requires QUBO formulation revision and a `specification_version` increment.

**Blocking:** Does not block SPEC-015 acceptance or initial implementation.

---

# Acceptance Checklist

**SPEC-011 Framework Obligations (FR-12):**

- [ ] Metadata: All FR-3 metadata fields present in the solver metadata table (backend_id, display_name, backend_category, implementation_language, determinism_class, supported_contract_version, specification_version)
- [ ] Problem Statement: States what optimization problem this backend solves, its algorithmic approach, and why it belongs in the MVP inventory
- [ ] Solver Classification: Backend category declared as `quantum_inspired_stochastic`; determinism class declared as `Stochastic (reproducible)`; infeasibility proof capability declared as No
- [ ] Algorithm Description (FR-1): Execution phases, distance metric, service duration non-use, completion condition, timeout condition defined
- [ ] QUBO Formulation Ownership Boundaries (FR-2): Spec-owned vs. implementation planning vs. adjacent specification ownership clearly delineated
- [ ] Annealing Schedule Behavior (FR-3): Monotonic non-increase constraint defined; temperature schedule class is an implementation planning decision; termination conditions defined
- [ ] Initial Solution and PRNG Draw Ordering (FR-4): Phase 1 and Phase 2 draw ordering described; Phase 1 ordering deferred to OQ-2 resolution; iteration ordering within Phase 2; phase boundary defined
- [ ] Transition Proposal and Metropolis Acceptance (FR-5): Acceptance criterion defined; draw consumption for improving vs. worsening transitions stated
- [ ] Best-So-Far Solution Tracking (FR-6): Candidate qualification requirements; quality comparison criterion; anytime contract; solution_count semantics defined
- [ ] Constraint Handling (FR-7): Capacity enforced through formulation; time windows and service durations not enforced; infeasibility detection not supported
- [ ] Capability Profile Declaration (FR-8): All nine fields from SPEC-011 FR-4.1 present; accuracy basis for latency_profile and quality_profile stated; is_provisional = true declared; latency_profile values expressed in seconds per SPEC-003 FR-4
- [ ] Seed Usage Policy (FR-9): Stochastic class stated; PCG64 named; stream constant documented; seeding procedure (once per execution, before Phase 1); seed authority (execution_seed exclusive); draw ordering cross-reference (FR-4 and FR-5); approved distribution algorithms identified; specific bounded integer algorithm deferred to OQ-2 resolution per ADR-010 Decision 3; floating-point determinism stated; prohibited entropy sources confirmed absent; satisfies SPEC-004 FR-1 and SPEC-011 FR-6.5
- [ ] Supported SolverOutcome Values (FR-10): Explicit table; Infeasible listed as Not Supported; Timeout-with-solution and Cancelled-with-solution behavior stated; satisfies SPEC-011 FR-5.3
- [ ] RoutePlan Output Requirements (FR-11): Presence per outcome; SPEC-011 FR-7.1 narrowing for Timeout and Cancelled declared; capacity validity guarantee; stop completeness guarantee; time window non-guarantee; execution_duration_ms obligation confirmed
- [ ] Extension Metadata (FR-12): All six keys documented with types, presence conditions, and descriptions; raw-data prohibition confirmed; absence on no-solution responses stated; satisfies SPEC-011 FR-7.4
- [ ] Failure Model (FR-13): ContractVersionMismatch, natural termination with no valid solution, internal error, timeout, and cancellation all defined with trigger conditions and required metadata
- [ ] Performance Characteristics: Expected behavior per size class; basis for estimates stated; relationship to seconds-based capability profile values noted
- [ ] Testability: Reproducibility, anytime behavior, Metropolis acceptance, temperature monotonicity, capability profile accuracy, extension metadata coverage, and observability all addressed
- [ ] Open Questions: OQ-1 through OQ-5 classified; blocking status stated; no open questions with unknown classification remain
- [ ] No individual solver specification obligation contradicts a framework requirement from SPEC-011 FR-1 through FR-11

**Blocking Open Questions Before Proposed:**

- [ ] OQ-1: PCG64 stream constant `0xcbbb9d5dc90c2383` confirmed independent from SPEC-002 stream constant
- [ ] OQ-2: Per-iteration PCG64 draw count documented in FR-4 after SA neighborhood structure is selected

**Backend-Specific Acceptance Criteria:**

- [ ] The anytime behavioral contract is clearly defined: Timeout and Cancelled responses include the best-so-far solution when it exists
- [ ] The QUBO formulation ownership boundary is explicitly stated, distinguishing specification-owned observable behavior from implementation planning decisions
- [ ] The two-phase PRNG draw ordering is documented (Phase 1 before Phase 2; per-iteration structure within Phase 2)
- [ ] The Metropolis acceptance draw consumption rule is unambiguous (one uniform float draw for ΔE > 0; no draw for ΔE ≤ 0)
- [ ] Extension metadata presence conditions are unambiguous (present when solution present; absent otherwise)
- [ ] The specification does not define any Scheduler, Worker, or Core behavior (owned by SPEC-003, SPEC-005, SPEC-004)
- [ ] The specification does not define the QUBO energy function, penalty weights, encoding, or decoding algorithm (implementation planning decisions per FR-2)

---

# Definition of Done

This backend is complete when:

- SPEC-015 is in Accepted status (this specification)
- OQ-1 (PCG64 stream constant) is confirmed and FR-4 and FR-9 reflect the confirmed value
- OQ-2 (PRNG draw ordering: per-iteration draw count, Phase 1 ordering, and bounded integer algorithm) is resolved and FR-4 and FR-9 are updated with the exact draw sequence and algorithm name
- The backend is implemented as a C++ SolverContract conforming to SPEC-004
- All acceptance criteria in the Testability section pass
- Reproducibility is verified: two identical SolverRequests (same problem, same execution_seed) produce identical SolverResponse solution fields when both produce Succeeded
- Anytime behavior is validated: the backend returns Timeout with a complete RoutePlan on test problems designed to trigger deadline self-termination
- The `solver.execute` span is emitted for every invocation and verifiable in the test environment
- Extension metadata keys (all six from FR-12) are present in every SolverResponse where a solution is present
- Empirical latency and quality values are measured from the SPEC-002 synthetic workload; `is_provisional = false` is declared after OQ-4 is resolved
- The capability profile is registered through the mechanism resolved by SPEC-003 OQ-2
- OQ-3 (anytime behavior risk) is validated empirically: the backend produces valid solutions on typical SPEC-002 problems
- OQ-5 (no-valid-solution failure rate) is evaluated; rate is within Project Owner approved threshold
- Engineering review passes
- Specification status is updated to Verified
