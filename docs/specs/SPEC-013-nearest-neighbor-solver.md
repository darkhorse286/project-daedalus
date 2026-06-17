# SPEC-013: Nearest Neighbor Solver

## Metadata

**Feature ID:** SPEC-013

**Title:** Nearest Neighbor Solver

**Status:** Draft

**Author:** Darkhorse286

**Created:** 2026-06-16

**Last Updated:** 2026-06-16

**Supersedes:** None

**Superseded By:** None

**Parent Specification:** SPEC-011 (Backend Solver Specifications)

**Related ADRs:** ADR-005, ADR-006, ADR-008, ADR-009, ADR-010, ADR-011

**Related Specs:** SPEC-001, SPEC-003, SPEC-004, SPEC-005, SPEC-006, SPEC-007, SPEC-010, SPEC-011, SPEC-012

---

**Solver Metadata (SPEC-011 FR-3):**

| Field | Value |
|---|---|
| `backend_id` | `nearest-neighbor` |
| `display_name` | Nearest Neighbor Heuristic |
| `backend_category` | `classical_deterministic` |
| `implementation_language` | `C++` |
| `determinism_class` | Deterministic |
| `supported_contract_version` | 1 |
| `specification_version` | 1 |

---

# Problem Statement

Project DAEDALUS requires at least one classical solver baseline before the QUBO simulated annealing backend can be meaningfully compared against it. Without a classical baseline, the system cannot demonstrate its core thesis: that most routing problems should never touch a quantum-adjacent backend because cheap classical methods are sufficient.

The nearest-neighbor heuristic is the simplest established construction heuristic for the Capacitated Vehicle Routing Problem (CVRP). It is fast, deterministic, and well-understood. Its solution quality is consistently below what more sophisticated methods achieve, which makes it an honest lower bound for baseline comparison and a direct counter-argument to the premise that quantum-inspired methods always justify their cost.

SPEC-013 defines the authoritative nearest-neighbor backend implementation. It specifies the algorithm's observable behavior, its capability profile for Scheduler consumption, its failure model, and its reproducibility obligations. It does not define the backend framework, the solver contract, or the Scheduler policy. Those responsibilities belong to SPEC-004, SPEC-011, and SPEC-003, respectively.

---

# Business Value

- Provides the primary classical comparison baseline against which QUBO simulated annealing quality is measured and regret is calculated
- Demonstrates the system's ability to identify when the cheapest, fastest backend is sufficient for a given problem
- Establishes a provably correct, fast-executing baseline that validates the evidence pipeline end-to-end before more complex backends are integrated
- Implements a well-known, auditable algorithm whose output can be independently verified
- Closes one of three blocking backend implementation prerequisites in SPEC-011 FR-11

---

# Employer Signaling

- System Design
- Optimization
- Reliability Engineering

The nearest-neighbor heuristic for CVRP requires correctly handling multi-vehicle sequential construction, capacity binding, greedy candidate selection, and deterministic tie-breaking. Specifying these behaviors precisely enough that two independent implementations produce semantically equivalent route plans demonstrates the ability to write implementation-grade behavioral specifications, not just high-level descriptions.

---

# Solver Classification

**Backend category:** `classical_deterministic`

**Determinism class:** Deterministic

**Infeasibility proof capability:** No

The nearest-neighbor heuristic is a polynomial-time greedy construction algorithm with no stochastic computation. Its output is fully determined by the routing problem inputs. It cannot prove that a routing problem has no feasible solution; it can only succeed in producing a complete route plan or fail to do so.

Classification is consistent with SPEC-011 FR-2.1 and FR-2.2. This classification is a property of the algorithm, not a runtime configuration parameter.

**Timeout behavior:** The algorithm completes in O(S * N) time where S is vehicle count and N is stop count. At MVP problem sizes (Large class: 76+ stops), execution completes in sub-millisecond wall-clock time. The Worker's external timeout backstop (SPEC-005) is the operative enforcement mechanism. The backend does not implement periodic deadline polling. The Worker enforced timeout is the authoritative backstop per SPEC-004 FR-10.

**Supported SolverOutcome values:** See FR-7.

---

# Requirements

### FR-1: Algorithm Description

**Description:**

The nearest-neighbor heuristic constructs a set of vehicle routes by greedily extending each route one stop at a time, always choosing the nearest unvisited stop that fits within the vehicle's remaining capacity. The algorithm processes vehicles sequentially. This section defines externally observable algorithm behavior. It does not prescribe internal data structures or implementation techniques.

**Starting location:** Each vehicle route begins from the depot. The depot is the origin point from which the first nearest-candidate selection is made.

**Candidate selection behavior:** At each step, the set of candidates is every unassigned stop whose demand does not exceed the current vehicle's remaining capacity. From this set, the algorithm selects the stop with the minimum Haversine distance from the current position (SPEC-001 FR-5). No stop outside the candidate set may be selected.

**Distance metric:** Haversine great-circle distance in kilometers (SPEC-001 FR-5). The same formula used throughout the domain. No alternative metric is used during construction.

**Tie-breaking behavior:** When two or more candidate stops have identical Haversine distance from the current position (within the precision of IEEE 754 double arithmetic), the algorithm selects the candidate with the lowest stop_id value. This produces a fully deterministic ordering with no stochastic component.

**Stop visitation behavior:** The selected candidate is added to the current vehicle's route in the order selected. The algorithm then advances the current position to the selected stop and reduces the vehicle's remaining capacity by the stop's demand. The stop is marked assigned and removed from all future candidate sets.

**Route completion behavior:** The current vehicle's route is complete when the candidate set is empty: no unassigned stop has demand that fits within the vehicle's remaining capacity. Route completion triggers a transition to the next vehicle.

**Depot return behavior:** Vehicles return to the depot after visiting their last assigned stop. The depot is not included in the route sequence. Per SPEC-004 FR-5, routes enumerate only the intermediate stops visited between depot departure and return. This is enforced at the contract level, not the algorithm level.

**Service duration and time window behavior:** Service duration values and time window fields are not consulted during route construction. Candidate selection and ordering are based exclusively on Haversine distance. Time window feasibility is evaluated post-execution by Core Quality Evaluation (SPEC-007), not by this backend.

**Acceptance Criteria:**
- Given a routing problem with N stops, V vehicles, and capacity C per vehicle, the algorithm evaluates candidates exclusively from unassigned stops with demand <= remaining vehicle capacity
- The algorithm selects the minimum-distance candidate at each step; when distances are equal, the lowest stop_id wins
- Service duration values and time window fields are not used during construction
- The resulting route plan never places the depot in any route sequence
- The route at index i in the RoutePlan corresponds to vehicle i

---

### FR-2: Multi-Vehicle Behavior

**Description:**

SPEC-001 FR-2 supports multiple vehicles with identical capacity. The nearest-neighbor algorithm handles multiple vehicles through sequential independent route construction.

**Vehicle assignment strategy:** Vehicles are processed in ascending index order: vehicle 0, then vehicle 1, through vehicle_count - 1. A vehicle's route construction begins only after the preceding vehicle's route construction is complete.

**Route construction strategy:** Each vehicle constructs its route independently from the current set of unassigned stops. Vehicle i builds its route entirely before vehicle i+1 begins. Stop assignments are permanent: once a stop is assigned to a vehicle, it does not become available to any subsequent vehicle. Vehicle i always begins its route from the depot, regardless of the depot's position relative to any stops assigned to earlier vehicles.

**Capacity handling behavior:** A stop is a candidate for the current vehicle only if its demand does not exceed the vehicle's remaining capacity at the current step. Stops that exceed remaining capacity are not candidates for the current vehicle but remain available for subsequent vehicles that may have sufficient remaining capacity.

**Behavior when stops remain after a vehicle exhausts candidates:** When no candidate stop fits the current vehicle's remaining capacity, the vehicle's route is finalized. Construction continues with the next vehicle. Stops that could not fit in an earlier vehicle's route remain available as candidates for subsequent vehicles.

**Behavior when all vehicles are exhausted before all stops are assigned:** If unassigned stops remain after vehicle vehicle_count - 1 has finalized its route, the algorithm has failed to produce a complete RoutePlan. This is a failure condition defined in FR-5. This outcome can occur for problems where the total stop demand does not exceed total fleet capacity but the greedy assignment order prevents complete allocation. The algorithm does not backtrack or retry with alternative assignments.

**Behavior when all stops are assigned before all vehicles are used:** Remaining vehicles have empty routes. Empty routes are valid per SPEC-004 FR-5. No additional stops are assigned to vehicles after all stops are assigned.

**Acceptance Criteria:**
- Vehicles are constructed in index order 0 through vehicle_count - 1
- Each vehicle begins route construction from the depot
- Once a stop is assigned, it is unavailable to all subsequent vehicles
- Empty vehicle routes are present in the RoutePlan for unused vehicles
- Stops that exceed one vehicle's capacity can be assigned to a subsequent vehicle if that vehicle has sufficient capacity
- When stops remain unassigned after all vehicles are exhausted, the outcome is Failed (see FR-5)

---

### FR-3: Constraint Handling

**Description:**

The nearest-neighbor algorithm enforces some constraints during route construction and ignores others. This section defines which constraints are enforced, which are ignored, and which produce failure outcomes. Ignored constraints are evaluated by Core Quality Evaluation (SPEC-007) after route construction.

**Capacity constraints (actively enforced):** The algorithm enforces per-vehicle capacity during route construction. A stop is excluded from the candidate set when its demand exceeds the vehicle's remaining capacity. The final route plan satisfies the capacity validity requirements of SPEC-004 FR-5 condition 5 when the algorithm produces a Succeeded outcome.

**Service duration constraints (not enforced during construction):** Service duration values (SPEC-001 FR-16) are present in the routing problem but are not consulted during route construction. The algorithm routes by distance only. Service durations affect time-based feasibility, which is outside the scope of this backend.

**Time window constraints (not enforced during construction):** Time window fields (SPEC-001 FR-9) are not consulted during route construction. This backend does not incorporate time window awareness into candidate selection or route ordering. The backend declares `supports_time_windows = false` in its capability profile (FR-4), which makes it ineligible for time-window-constrained problems in Scheduler eligibility evaluation (SPEC-003 FR-5). This backend should not receive routing problems with time windows.

**Unreachable stops:** The algorithm does not detect or handle unreachable stops. If a stop has coordinates that produce an extreme Haversine distance from all other locations, it remains reachable in principle. The algorithm assigns it like any other stop. Haversine distances are finite for all valid coordinate pairs (SPEC-001 FR-5).

**Infeasibility detection:** This backend does not detect or prove infeasibility. It is not authorized to return `Infeasible` (SPEC-011 FR-5.2). The only failure condition resulting from problem characteristics is the degenerate capacity assignment case described in FR-2 and FR-5, which produces `Failed` with `failure_code = InternalError`, not `Infeasible`.

**Explicitly documented limitations:**

1. The algorithm does not guarantee a complete assignment even when total stop demand does not exceed total fleet capacity. Greedy sequential construction can produce capacity binding that leaves stops unassigned. The algorithm fails (see FR-5) rather than backtracking.

2. The algorithm does not account for time window feasibility. The produced route plan may violate time windows on problems where the Scheduler incorrectly selects this backend despite `supports_time_windows = false`.

3. The algorithm does not account for service durations in travel time estimation. Route quality metrics that include service duration overhead will reflect worse performance than the algorithm's distance-only construction metric suggests.

**Acceptance Criteria:**
- Every stop in a Succeeded response's RoutePlan is assigned to a vehicle whose total stop demand does not exceed capacity_per_vehicle
- No candidate stop is selected when its demand exceeds the vehicle's remaining capacity at the time of selection
- The backend never returns `Infeasible`
- Service duration and time window fields are not inspected during route construction

---

### FR-4: Capability Profile Declaration

**Description:**

The following capability profile is the registration artifact consumed by the Scheduler (SPEC-003 FR-4, SPEC-011 FR-4). All declared values must accurately reflect this backend's algorithm behavior and are the basis for Scheduler eligibility evaluation and objective scoring. Values marked as estimates require empirical measurement before `is_provisional` can be set to false.

| Field | Value | Rationale and Accuracy Basis |
|---|---|---|
| `backend_id` | `nearest-neighbor` | Stable unique identifier per SPEC-011 FR-3. See OQ-1 for naming convention clarification. |
| `supported_size_classes` | `{Small, Medium, Large}` | The algorithm is O(S * N) in vehicle count S and stop count N. For Large problems (76+ stops), execution completes in sub-millisecond time. All three size classes are supported with no practical upper bound at MVP scale. |
| `supports_time_windows` | `false` | The algorithm does not incorporate time window constraints during route construction. Candidate selection is based exclusively on Haversine distance. Declaring `true` would misrepresent the algorithm's construction behavior and cause the Scheduler to route time-windowed problems to a backend that cannot respect them. |
| `supports_capacity_constraints` | `true` | Capacity is enforced during candidate selection. A stop is excluded from the candidate set when its demand exceeds the vehicle's remaining capacity. Capacity validity is verified before returning Succeeded. |
| `latency_profile` | Small: < 1ms, Medium: < 5ms, Large: < 50ms | Algorithmic estimate based on O(S * N) construction with O(N^2) distance precomputation. Values are pre-implementation estimates. Empirical measurement on representative benchmark problems of each size class is required before `is_provisional` can be set to false. |
| `quality_profile` | `Baseline` | The nearest-neighbor heuristic produces solutions that are consistently above optimal for CVRP. The algorithm's greedy, non-backtracking construction prevents it from achieving competitive or near-optimal quality. It serves as a lower-quality reference baseline, which is its intended role in the evidence system. Basis: well-established CVRP literature; empirical measurement during benchmarking will confirm this classification. |
| `cost_profile` | `1` | The algorithm runs in-process in C++ with no external dependencies, no network calls, and no paid compute. The cost per invocation is negligible. A value of 1 represents the minimum relative cost unit. |
| `is_provisional` | `true` | Latency and quality profile values are pre-implementation estimates. This backend must declare `is_provisional = true` until empirical measurement validates the declared values per SPEC-011 FR-4.2. |
| `supported_contract_version` | `1` | Targets SPEC-004 contract version 1. |

**Accuracy basis:** Latency estimates are derived from algorithmic analysis of O(S * N) route construction and O(N^2) pairwise distance precomputation, not from empirical measurement. Quality classification as `Baseline` is consistent with established CVRP heuristic benchmarks in the literature. Both values require empirical validation against the SPEC-002 synthetic workload before `is_provisional = false` can be declared.

**Acceptance Criteria:**
- All nine fields from the SPEC-011 FR-4.1 table are present
- `backend_id` matches the FR-3 metadata value
- `supported_contract_version` equals 1
- `is_provisional = true` is declared and remains true until empirical validation is complete
- The accuracy basis for `latency_profile` and `quality_profile` is stated

---

### FR-5: Seed Usage Policy

**Description:**

The nearest-neighbor heuristic is a deterministic construction algorithm. It contains no stochastic computation. No PRNG is required.

**Determinism class:** Deterministic

**Execution seed usage:** The `execution_seed` field in the SolverRequest (SPEC-004 FR-2) is accepted without error, as required by SPEC-004 FR-11.5 for all backends regardless of classification. The field is not used in any computation. The algorithm's output is determined entirely by the routing problem's coordinates, demands, vehicle count, and capacity per vehicle.

**Unconditional determinism:** This backend produces identical route plans for identical routing problem inputs on every invocation, regardless of `execution_seed` value, invocation count, process state, or system environment. The tie-breaking rule (lowest stop_id on distance equality) and the sequential vehicle ordering are the only sources of ordering in the algorithm, both of which are fully determined by the problem's stop identifiers.

**Prohibited entropy sources:** No entropy source is used in any part of this backend's execution. System time, process ID, OS random sources, and hardware entropy are not consulted during route construction, candidate selection, tie-breaking, or any other algorithmic step.

This section satisfies the documentation obligation stated in SPEC-004 FR-1 and SPEC-011 FR-6.5.

**Acceptance Criteria:**
- Two SolverRequests with identical routing problems and different `execution_seed` values produce identical SolverResponse solutions
- Two SolverRequests with identical routing problems and identical `execution_seed` values produce identical SolverResponse solutions
- The backend accepts any `execution_seed` value without error

---

### FR-6: Supported SolverOutcome Values

**Description:**

This backend is classified `classical_deterministic`. Per SPEC-011 FR-5.1, it must support `Succeeded`, `Timeout`, `Cancelled`, and `Failed`. It must not support `Infeasible`.

| Outcome | Supported | Trigger Condition |
|---|---|---|
| `Succeeded` | Yes | All stops assigned to vehicles; RoutePlan satisfies all SPEC-004 FR-5 structural validity requirements |
| `Timeout` | Yes | Execution exceeds `execution_timeout_ms`; Worker terminates the backend (expected to be rare given O(S * N) completion) |
| `Cancelled` | Yes | Worker signals cancellation before route construction completes |
| `Failed` | Yes | Contract version mismatch; or all vehicles exhausted before all stops are assigned (degenerate capacity case); or any internal error preventing route plan production |
| `Infeasible` | **Not Supported** | This backend cannot prove infeasibility. Prohibited per SPEC-011 FR-5.2. |

**Notes on Timeout:** Given O(S * N) construction time, the algorithm is expected to complete in sub-millisecond wall-clock time for all MVP problem sizes. Timeout is listed as supported because all heuristic backends must support it per the contract (SPEC-011 FR-5.1), and because the Worker's external backstop may terminate the backend if it is invoked in a degraded environment. If the backend is terminated mid-construction with no complete route plan, it returns Timeout with no solution.

**Notes on Cancelled:** If cancellation arrives before route construction is complete, the backend returns Cancelled with no solution. If construction has already produced a complete RoutePlan satisfying SPEC-004 FR-5 at the time of cancellation, the backend returns Cancelled with the complete RoutePlan included.

This section satisfies the supported outcome declaration requirement in SPEC-011 FR-5.3.

**Acceptance Criteria:**
- The backend never returns `Infeasible`
- The backend returns `Succeeded` with a structurally valid RoutePlan when all stops are assigned
- The backend returns `Failed` with `InternalError` when stops remain unassigned after all vehicles are exhausted
- `Cancelled` and `Timeout` responses include a solution only if a complete RoutePlan satisfying SPEC-004 FR-5 was produced before the signal or deadline

---

### FR-7: RoutePlan Output Requirements

**Description:**

The nearest-neighbor algorithm produces a RoutePlan conforming to SPEC-004 FR-5. No backend-specific relaxations of the structural requirements are declared.

**RoutePlan presence per outcome:**

| Outcome | RoutePlan Present? | Requirements When Present |
|---|---|---|
| `Succeeded` | Required | All SPEC-004 FR-5 conditions 1-5 satisfied |
| `Timeout` | Optional | Present only if a complete RoutePlan satisfying all SPEC-004 FR-5 conditions existed before the deadline |
| `Cancelled` | Optional | Present only if a complete RoutePlan satisfying all SPEC-004 FR-5 conditions existed before cancellation |
| `Failed` | Absent | Never present |
| `Infeasible` | Not Applicable | Outcome not supported |

**Capacity validity guarantee on Succeeded:** The algorithm's construction rule (excluding candidates that exceed remaining capacity) ensures that the total demand of stops in each route never exceeds `capacity_per_vehicle`. This is verified by the algorithm before returning `Succeeded`. The Worker also validates this condition post-receipt (SPEC-004 FR-5).

**Stop assignment completeness guarantee on Succeeded:** A Succeeded response guarantees that every stop in the routing problem appears in exactly one route. This is verified by the algorithm before returning `Succeeded`.

**Vehicle count guarantee:** The RoutePlan always contains exactly `vehicle_count` route entries. Unused vehicles have empty route sequences. No route entry is omitted.

**Time window non-guarantee:** A Succeeded response does not guarantee time window feasibility. This backend does not enforce time window constraints during construction. Time window feasibility is evaluated by Core Quality Evaluation (SPEC-007) after the Worker receives the SolverResponse. Given that this backend declares `supports_time_windows = false`, the Scheduler should not route time-windowed problems here. A Succeeded response from this backend on a time-windowed problem (if the Scheduler incorrectly routes one) may produce time-infeasible routes; this is a solver quality defect, not a contract violation (SPEC-004 FR-7).

`execution_duration_ms` is always populated. It reflects wall-clock execution time from route construction start to RoutePlan finalization. It excludes SolverRequest deserialization and SolverResponse serialization per SPEC-011 performance obligations.

**Acceptance Criteria:**
- Every Succeeded response contains a RoutePlan with `routes.size()` = `vehicle_count`
- No stop appears in more than one route in a Succeeded response
- Every stop in the routing problem appears in exactly one route in a Succeeded response
- No route in a Succeeded response has total stop demand exceeding `capacity_per_vehicle`
- `execution_duration_ms` is present in every SolverResponse

---

### FR-8: Extension Metadata

**Description:**

The nearest-neighbor backend produces the following `extension_metadata` keys in the SolverResponse on `Succeeded` outcomes. These keys provide diagnostic information about the route construction execution. All values are string-encoded.

| Key | Type (encoded as string) | Present When | Description |
|---|---|---|---|
| `nn.total_distance_km` | float64 | Succeeded | Sum of Haversine distances across all vehicle routes, including each vehicle's return leg from its last stop to the depot. Units: kilometers. This is the solver's internal distance objective, not the normalized quality metric used by Core (SPEC-007). |
| `nn.routes_used` | uint32 | Succeeded | Number of vehicles with at least one assigned stop. Vehicles with empty routes are not counted. |
| `nn.capacity_rejections` | uint32 | Succeeded | Total number of candidate stops skipped during construction because their demand exceeded the current vehicle's remaining capacity. A higher value indicates that capacity constraints caused more vehicle transitions than a purely distance-optimal construction would require. |

Extension metadata is not produced on `Failed`, `Timeout`, or `Cancelled` outcomes.

`extension_metadata` must not contain routing problem raw data (geographic coordinate arrays, full stop lists, demands). The three keys above contain only derived summary values safe for evidence log inclusion (SPEC-004 FR-13, SPEC-001 Security Considerations).

Consumers must silently ignore any keys not listed here. Unrecognized keys are not contract-stable across `specification_version` changes.

**Acceptance Criteria:**
- All three keys are present in the `extension_metadata` of every Succeeded SolverResponse
- Key values are non-empty strings encoding valid numeric values
- No key contains routing problem raw data (coordinates, demands, stop lists)
- Extension metadata is absent on Failed, Timeout, and Cancelled responses

---

### FR-9: Failure Model

**Description:**

This section defines all failure conditions specific to the nearest-neighbor backend. General framework failure obligations are defined in SPEC-011 FR-8.

**Contract version mismatch:**

- **Trigger:** The `contract_version` in the SolverRequest does not equal 1.
- **Outcome:** `Failed`, `failure_code = ContractVersionMismatch`
- **Behavior:** The backend returns `ContractVersionMismatch` immediately before processing the routing problem. No route construction begins.
- **Metadata:** `failure_message` must identify the received version and the expected version (1).

**Degenerate capacity assignment failure:**

- **Trigger:** After all `vehicle_count` vehicles have completed sequential route construction, one or more stops remain unassigned. This occurs when the greedy sequential assignment order binds capacity in a way that prevents complete stop allocation, even though total stop demand does not exceed total fleet capacity.
- **Outcome:** `Failed`, `failure_code = InternalError`
- **Behavior:** The algorithm does not backtrack or retry. It reports failure with the number of unassigned stops in `failure_message`.
- **Metadata:** `failure_message` must identify the failure cause ("nearest-neighbor construction failed to assign all stops") and the count of unassigned stops at the time of failure.
- **Note:** This condition is rare on typical CVRP instances where capacity utilization ratio is significantly below 1.0. It becomes more likely as capacity utilization approaches 1.0 and stop demand values are heterogeneous. See Quality Expectations.

**Internal error:**

- **Trigger:** Any unexpected condition during route construction that prevents the algorithm from producing a RoutePlan (for example, invalid distance computation producing NaN or infinite values from coordinates that are structurally valid per SPEC-001 FR-5 but produce arithmetic edge cases).
- **Outcome:** `Failed`, `failure_code = InternalError`
- **Behavior:** The backend returns `Failed` immediately. No partial RoutePlan is present.
- **Metadata:** `failure_message` must provide a diagnostic description identifying the failure cause.

**Timeout:**

- **Trigger:** Worker-enforced external timeout. Given O(S * N) construction time, this is expected to be extremely rare for MVP problem sizes.
- **Outcome:** `Timeout`, `failure_code = ExecutionTimeout`
- **Behavior:** If a complete RoutePlan satisfying SPEC-004 FR-5 was produced before termination, it may be included. Otherwise, the response contains no solution.

**Cancellation:**

- **Trigger:** Worker cancellation signal received before construction completes.
- **Outcome:** `Cancelled`, `failure_code = ExecutionCancelled`
- **Behavior:** If a complete RoutePlan was produced before the cancellation signal, it may be included. Otherwise, the response contains no solution.

All `Failed` responses include a `SolverFailureDetail` with both `failure_code` and `failure_message` populated. The `failure_message` is diagnostic (developer-readable), not user-facing. The backend does not write to stdout or stderr.

**Acceptance Criteria:**
- `ContractVersionMismatch` is returned before any route construction begins on version mismatch
- `InternalError` is returned when stops remain unassigned after all vehicles are exhausted
- `failure_message` is non-empty on every Failed response
- No route plan is present in any Failed response

---

# Performance Characteristics

The nearest-neighbor heuristic executes in two phases: distance matrix precomputation and route construction.

**Distance matrix precomputation:** The backend computes pairwise Haversine distances from all geographic coordinates (depot + N stops) before route construction begins. This is O(N^2) Haversine calculations. For N = 76 (Large class lower bound), this produces approximately 2,926 distance calculations. For N = 100 (mid-Large class), approximately 5,050. Haversine is a transcendental-function-heavy computation; this phase dominates execution time at larger problem sizes.

**Route construction:** O(S * N) candidate evaluations where S is vehicle count and N is stop count. For each of the S vehicles, the algorithm examines at most N candidates per step and takes at most N steps (one per stop). Construction is bounded by O(S * N^2) in the worst case, but is typically closer to O(S * N) in practice because the candidate set shrinks with each assignment.

**Expected execution duration per size class:**

| Size Class | Stop Range | Expected Duration | Basis |
|---|---|---|---|
| Small | 1-25 stops | < 1ms | Algorithmic analysis; O(N^2) precomputation trivial |
| Medium | 26-75 stops | < 5ms | Algorithmic analysis; O(N^2) Haversine dominates |
| Large | 76+ stops | < 50ms | Algorithmic analysis; empirical validation required |

All values are pre-implementation estimates. Empirical measurement on SPEC-002 synthetic workload problems of each size class is required before `is_provisional` is set to false.

**Memory:** The dominant memory allocation is the pairwise distance matrix, O(N^2) doubles. For N = 100 stops, this is approximately 80 KB. No practical constraint at MVP scale.

**Expected solution quality:**

The nearest-neighbor heuristic is well-documented in the CVRP literature as a fast construction heuristic that consistently produces solutions above optimal. For typical benchmark CVRP instances, NN solutions are 20-30% above the best-known solution. This is consistent with the `Baseline` quality classification in the capability profile (FR-4). Route quality is sensitive to geographic stop distribution and capacity utilization:

- **Workloads where NN performs better:** Uniformly distributed stops with moderate capacity utilization (capacity_utilization_ratio 0.5-0.8). Geographic clustering naturally aligned with vehicle capacity boundaries.
- **Workloads where NN performs worse:** High-density clusters at geographic extremes, high capacity utilization (approaching 1.0), heterogeneous stop demands, or problems where the globally shortest routes require visiting stops in a non-greedy order.

---

# Quality Expectations

**Route quality expectations:** The nearest-neighbor heuristic produces baseline-quality routes. Solutions are consistently feasible (when construction succeeds) but are not optimal or near-optimal. The algorithm does not revisit or improve routes after initial construction. It is intended to serve as a comparison baseline for QUBO simulated annealing quality evaluation.

**Known weaknesses:**

1. **Degenerate capacity binding:** At high capacity utilization ratios (capacity_utilization_ratio approaching 1.0) with heterogeneous stop demands, the greedy assignment order may create capacity binding that leaves stops unassigned. This produces a Failed outcome rather than a suboptimal route plan.

2. **Locally optimal, globally suboptimal:** Each candidate selection is optimal for the current vehicle and position, but this provides no guarantee about overall tour quality. Poor early stop selections compound through subsequent steps.

3. **Route imbalance:** Vehicles processed earlier may receive more stops and longer routes than later vehicles, because earlier vehicles have access to the full unassigned stop set. This can produce unbalanced route lengths across the fleet.

4. **Geographic inefficiency near vehicle transitions:** When a vehicle exhausts its capacity in one part of the geographic space, the next vehicle must start again from the depot. This can produce route plans where vehicles cross each other's geographic territories unnecessarily.

**Workload classes where NN performs well:** Problems with uniformly distributed stops, moderate fleet capacity utilization, and no time window constraints. At Small and Medium problem sizes where a simple construction pass suffices to demonstrate evidence collection.

**Workload classes where NN performs poorly:** High-capacity-utilization problems with heterogeneous demands, geographically clustered stops at extreme distances from the depot, and any problem where backtracking from a greedy choice would improve tour quality. These workloads are where the QUBO simulated annealing backend's additional cost becomes potentially justified.

---

# Non-Requirements

- This specification does not define time window feasibility checking. Time windows are evaluated by Core Quality Evaluation (SPEC-007).
- This specification does not define route improvement heuristics (2-opt, 3-opt, or other local search). The algorithm performs one greedy construction pass without improvement.
- This specification does not define how backtracking or reassignment of stops to previous vehicles might be implemented. The algorithm does not backtrack.
- This specification does not define multi-depot behavior. Single depot only, per SPEC-001 FR-3.
- This specification does not define heterogeneous vehicle fleet handling. All vehicles have identical capacity, per SPEC-001 FR-2.
- This specification does not define how workload features are computed. That is SPEC-010's responsibility.
- This specification does not define backend selection policy. That is SPEC-003's responsibility.
- This specification does not define evidence persistence. That is SPEC-006's responsibility.
- This specification does not define quality evaluation metrics. That is SPEC-007's responsibility.
- This specification does not alter the SolverContract schema. That is SPEC-004's responsibility.
- This specification does not define parallel vehicle route construction. Vehicles are constructed sequentially.

---

# Assumptions

1. The routing problem received by this backend has been validated by Core per ADR-009. The backend does not re-validate the routing problem's structural constraints. Coordinates are within valid ranges, demands are non-negative, and total demand does not exceed total fleet capacity.

2. All vehicles have identical capacity. The algorithm applies the same capacity constraint uniformly across all vehicles. Heterogeneous fleets are not supported by this algorithm.

3. Haversine great-circle distance is an acceptable construction metric for synthetic workloads. For real-world problems, route quality would differ from road-network optimized routes, but this is a known limitation of the routing problem model (SPEC-001 FR-5), not the algorithm.

4. The pairwise distance matrix can be computed once before route construction begins and cached for the duration of the algorithm's execution. This is the expected implementation approach and is consistent with the algorithm's O(N^2) precomputation overhead.

5. The degenerate capacity assignment failure (stops remaining after all vehicles exhausted) is rare in the SPEC-002 synthetic workload at typical capacity utilization ratios. The SPEC-002 generator produces problems where total demand does not exceed total fleet capacity; whether the greedy assignment avoids degenerate binding depends on problem structure.

6. For MVP scope, the algorithm runs to completion before any timeout signal arrives. Worker-enforced timeout is the safety backstop. No periodic deadline polling is implemented in the first version.

---

# Constraints

1. This backend implements SPEC-004 contract version 1. `contract_version` mismatch must be detected before any routing problem processing begins (SPEC-011 FR-8.3).

2. `execution_seed` must be accepted without error. It must not be used in any computation (SPEC-004 FR-11.5, SPEC-011 FR-6.1).

3. No backend-specific logic may exist in the Scheduler, Worker, or Core contract-handling code. This backend is accessed exclusively through the normalized SolverContract interface (ADR-008).

4. `extension_metadata` must not contain routing problem raw data (geographic coordinate arrays, stop lists, demand arrays). Only derived summary values may appear (SPEC-004 FR-13, SPEC-011 FR-7.4).

5. The backend must not emit OpenTelemetry spans, metrics, or structured logs directly. Diagnostic output is returned through `SolverFailureDetail.failure_message` and `extension_metadata` only (SPEC-011 FR-9.2).

6. The backend must not write to stdout or stderr (SPEC-011 FR-9.2).

7. `execution_duration_ms` must reflect actual wall-clock execution time within the backend from construction start to RoutePlan finalization, excluding SolverRequest deserialization and SolverResponse serialization (SPEC-011 performance obligations).

8. The algorithm must not return `Infeasible` under any condition (SPEC-011 FR-5.2).

---

# Inputs

**SolverRequest** (SPEC-004 FR-2): The complete input to this backend. Relevant fields for the nearest-neighbor algorithm:

| Field | How Used |
|---|---|
| `routing_problem.stops` | Stop coordinates, demands, stop_id values — all used in construction |
| `routing_problem.depot` | Depot coordinates — used as the starting position for each vehicle's route |
| `routing_problem.vehicle_count` | Determines number of routes produced |
| `routing_problem.capacity_per_vehicle` | Enforced capacity limit per route |
| `execution_timeout_ms` | Deadline for execution; Worker enforces externally |
| `execution_seed` | Accepted, not used (deterministic backend) |
| `contract_version` | Validated before processing; must equal 1 |
| `job_id`, `decision_id`, `backend_id` | Correlation only; do not affect route construction |

Fields not used in construction: `routing_problem.time_window_open`, `routing_problem.time_window_close`, `routing_problem.service_duration` (see FR-3).

---

# Outputs

**SolverResponse** (SPEC-004 FR-3): The complete output of this backend.

On `Succeeded`:
- `outcome = Succeeded`
- `solution`: RoutePlan with `vehicle_count` routes, all stops assigned, capacity valid
- `statistics.execution_duration_ms`: Wall-clock construction time in milliseconds
- `extension_metadata`: Keys `nn.total_distance_km`, `nn.routes_used`, `nn.capacity_rejections`
- `failure`: Absent

On `Failed`:
- `outcome = Failed`
- `failure.failure_code`: `ContractVersionMismatch` or `InternalError`
- `failure.failure_message`: Diagnostic description
- `solution`: Absent
- `extension_metadata`: Absent

On `Timeout` or `Cancelled`:
- `outcome`: `Timeout` or `Cancelled`
- `failure`: Present with corresponding failure code
- `solution`: Present only if a complete RoutePlan satisfying SPEC-004 FR-5 was produced
- `extension_metadata`: Absent

Consumer: The Worker receives the SolverResponse, validates structural requirements (SPEC-004 FR-5), and persists it to the evidence log. Core evaluates route quality from the RoutePlan (SPEC-007). The `extension_metadata` passes through to the evidence record.

---

# Architectural Impact

| Component | Impact | Notes |
|---|---|---|
| Domain Layer (C++ Core / backends) | Yes | SPEC-013 defines a new C++ SolverContract implementation |
| Scheduler (SPEC-003) | Yes | Capability profile must be registered before backend is eligible for selection |
| Solver Contract (SPEC-004) | None | SPEC-013 satisfies SPEC-004 FR-1 obligations; no schema changes |
| Worker (SPEC-005) | None | No lifecycle changes; standard invocation, timeout, and cancellation handling applies |
| Evidence Log (SPEC-006) | None | Evidence persistence is unchanged; `extension_metadata` keys pass through |
| Quality Evaluation (SPEC-007) | None | Core evaluates quality from normalized RoutePlan; no changes required |
| Feature Extraction (SPEC-010) | None | Workload features are computed before backend selection; no changes |
| Persistence Schema (SPEC-012) | None | No new tables; `extension_metadata` stored per existing schema |
| API Layer | None | No new API surface |
| Observability | None | Worker-emitted `solver.execute` span (SPEC-004 FR-15); no additional instrumentation required |
| Security | None | No new trust boundaries; `extension_metadata` raw-data prohibition applies per FR-8 |
| Configuration | Yes | Capability profile must be registered through the mechanism resolved by SPEC-003 OQ-2 |

**SPEC-011 FR-11:** The MVP backend inventory lists `nearest_neighbor` as "Required; not yet written." SPEC-013 satisfies this prerequisite. SPEC-011 FR-11 should be updated to reflect this specification's status.

---

# Testability

The following behaviors must be verified. Specific test implementations are determined during implementation planning.

**Contract conformance (inherited from SPEC-004):**

1. **Structural validity:** A Succeeded SolverResponse contains a RoutePlan where: (a) `routes.size()` = `vehicle_count`; (b) every stop_id appears exactly once across all routes; (c) no route's total demand exceeds `capacity_per_vehicle`; (d) no route contains the depot.

2. **Determinism:** Two SolverRequests with identical routing problems and different `execution_seed` values produce identical SolverResponse solutions.

3. **Seed acceptance:** The backend accepts any `execution_seed` value without error and produces identical output regardless of the seed value.

4. **ContractVersionMismatch:** A SolverRequest with `contract_version` != 1 returns `Failed` with `ContractVersionMismatch` before any routing problem processing.

5. **Trivial case:** A routing problem with one stop and one vehicle produces a Succeeded response with a single route containing that stop.

6. **Empty vehicle routes:** A routing problem with more vehicles than stops produces a Succeeded response where some routes are empty.

**Algorithm correctness:**

7. **Nearest-neighbor selection:** On a problem with multiple unassigned stops, the stop assigned at each construction step is the nearest candidate from the current position (verified by checking all distances against the selected stop's distance).

8. **Tie-breaking:** On a problem constructed with two stops at identical distance from the current position, the stop with the lower stop_id is selected.

9. **Capacity enforcement:** No Succeeded response contains a route where total stop demand exceeds `capacity_per_vehicle`. Verified across all SPEC-002 synthetic workload sizes and constraint configurations.

10. **Sequential vehicle processing:** The route at index 0 in the RoutePlan contains only stops assigned to vehicle 0; stops that appear in route 0 do not appear in routes 1 through N-1.

11. **Degenerate capacity failure:** A routing problem designed to produce capacity binding (stops remain after all vehicles are exhausted despite total demand = total capacity) returns `Failed` with `InternalError`. The `failure_message` identifies the cause and unassigned stop count.

**Capability profile accuracy:**

12. **Latency profile validation:** Empirical execution duration on representative SPEC-002 problems of each size class is within the declared `latency_profile` bounds. Required before `is_provisional` is set to false.

13. **Quality classification:** Route quality (measured by Core's normalized metric, SPEC-007) on SPEC-002 benchmark problems is consistent with `Baseline` classification. Required before `is_provisional` is set to false.

**Extension metadata:**

14. **Key presence:** All three `extension_metadata` keys (`nn.total_distance_km`, `nn.routes_used`, `nn.capacity_rejections`) are present in every Succeeded response.

15. **nn.total_distance_km accuracy:** The reported total distance matches the sum of Haversine distances computed from the route sequences in the RoutePlan, including depot-return legs.

16. **nn.routes_used accuracy:** The reported value equals the count of routes in the RoutePlan with at least one stop.

17. **nn.capacity_rejections accuracy:** The reported value equals the total number of candidate stops excluded due to capacity constraint during construction.

**Observability:**

18. **solver.execute span:** A `solver.execute` OTel span is emitted by the Worker for every SolverRequest dispatch, successful or not. All required attributes (SPEC-004 FR-15) are present. Verified in integration test.

---

# Observability Requirements

The `solver.execute` span is emitted by the Worker for every invocation of this backend (SPEC-004 FR-15). The backend's contribution to this span is through the SolverResponse fields:

| Span Attribute | Source |
|---|---|
| `outcome` | `SolverResponse.outcome` |
| `execution_duration_ms` | `SolverResponse.statistics.execution_duration_ms` |

All other `solver.execute` span attributes are available to the Worker from the SolverRequest, routing problem, or workload features.

**Diagnostic questions this backend must enable through `extension_metadata`:**

1. What was the total route distance produced by the nearest-neighbor construction? (`nn.total_distance_km`)
2. How many vehicles were actually used? (`nn.routes_used`)
3. How many candidate stops were excluded by capacity constraints during construction? (`nn.capacity_rejections`)

These values pass through the Worker to the evidence log, where they are available for backend-specific reporting and comparison.

**No additional instrumentation:** The backend does not emit OpenTelemetry spans, metrics, or structured logs. Diagnostic output is confined to `SolverFailureDetail.failure_message` on failure and `extension_metadata` on success, per SPEC-011 FR-9.2.

The backend does not write to stdout or stderr.

---

# Security Considerations

**Seed authority:** The backend does not use any entropy source. There are no reproducibility-critical stochastic paths that could be exploited through entropy manipulation (SPEC-004 FR-11, ADR-010 Decision 4).

**Extension metadata safety:** The three `extension_metadata` keys (`nn.total_distance_km`, `nn.routes_used`, `nn.capacity_rejections`) contain only derived summary values. No key contains geographic coordinate arrays, stop identifier lists, demand arrays, or any other raw routing problem input. This complies with the raw-data prohibition in SPEC-004 FR-13 and SPEC-001 Security Considerations.

**Input trust:** The routing problem arrives at the backend pre-validated by Core (ADR-009). The backend trusts the problem's structural validity. The backend does not re-validate the routing problem's coordinates, demands, or fleet parameters.

**No external access:** The backend has no access to external systems, networks, files, or queues. It operates exclusively on the in-process SolverRequest data.

**No stdout/stderr:** Diagnostic output is confined to the SolverResponse contract fields, preventing any routing problem data from escaping through uncontrolled channels.

---

# Performance Considerations

**Distance computation overhead:** Haversine computation is trigonometric and transcendental-function-heavy. For N stops + 1 depot, the pairwise distance precomputation requires N(N+1)/2 Haversine calculations (exploiting distance symmetry). For Large-class problems (76+ stops), this number of computations should be measured empirically and its contribution to `execution_duration_ms` should be noted in benchmark evidence.

**Construction overhead:** O(S * N) candidate evaluations, where S is vehicle count and N is stop count. For MVP problem sizes (maximum 100+ stops, typical vehicle count < 20), this is negligible relative to distance precomputation.

**Memory overhead:** The pairwise distance matrix is O(N^2) doubles. For N = 100, this is approximately 80 KB. No practical constraint at MVP scale.

**Timeout practical risk:** Given sub-millisecond completion for MVP problem sizes, timeout is not a practical concern. The Worker external backstop (SPEC-005) is the theoretical safety mechanism for degenerate environments.

Do not invent specific latency targets. The areas above require measurement during implementation. See Performance Characteristics for pre-implementation estimates.

---

# Documentation Updates Required

The following follow-on updates are required as a result of this specification. No updates are performed here.

**SPEC-003:**
- The capability profile for `nearest-neighbor` must be registered through the mechanism resolved by SPEC-003 OQ-2 before this backend becomes eligible for Scheduler selection. No SPEC-003 schema change is required; the capability profile fields defined in FR-4 conform to the existing SPEC-003 FR-4 schema.

**SPEC-011:**
- FR-11.1 MVP backend inventory: Update the `nearest_neighbor` row's "Solver Specification Status" from "Required; not yet written" to reflect SPEC-013's Draft status, and ultimately to Accepted when SPEC-013 is accepted.
- OQ-3 (Classical Deterministic Backend Timeout Self-Termination Strategy): SPEC-013 resolves this question for the nearest-neighbor backend by deferring to the Worker external backstop given O(S * N) construction time. The OQ remains relevant for SPEC-014 (Greedy Insertion) if that backend has different timing characteristics.

**SPEC-012:**
- No schema changes are required. `extension_metadata` for `nearest-neighbor` passes through the existing JSONB column structure in `solver_run_records`. The three extension metadata keys defined in FR-8 are stored within the existing schema.

**docs/architecture.md:**
- No changes required. SPEC-013 conforms to the architecture principles and required observability spans. The nearest-neighbor backend is already identified in architecture.md as an MVP classical baseline solver.

---

# Open Questions

### OQ-1: backend_id Naming Convention Conflict

**Classification:** Implementation Planning Decision

**Question:** Should the backend_id be `nearest-neighbor` (kebab-case per SPEC-011 FR-3.1) or `nearest_neighbor` (snake_case per SPEC-011 FR-2.2 inventory table)?

**Context:** SPEC-011 FR-3.1 defines `backend_id` as "Kebab-case, lowercase." The SPEC-011 FR-2.2 inventory table uses `nearest_neighbor`, `greedy_insertion`, and `qubo_simulated_annealing` — all snake_case identifiers. These are in conflict. This specification uses `nearest-neighbor` per the literal FR-3.1 definition.

The `backend_id` value appears in: the `solver.execute` span attribute, the evidence log `solver_run_records`, the capability profile registration, and all trace correlation. The value must be stable once this specification is accepted (SPEC-011 FR-3.2).

**Resolution required before:** Capability profile registration (SPEC-011 FR-10.1 Prerequisite 7). The conflict must be resolved before the backend_id is registered in any persistent system.

**Blocking:** Does not block SPEC-013 draft. Blocks capability profile registration and any persistent reference to this backend_id.

---

### OQ-2: Capability Profile Empirical Latency and Quality Values

**Classification:** Implementation Planning Decision

**Question:** What are the empirically measured latency values per size class and confirmed quality classification for the nearest-neighbor backend?

**Context:** FR-4 declares `is_provisional = true` with pre-implementation latency estimates and a literature-based quality classification. Empirical measurement against the SPEC-002 synthetic workload is required before `is_provisional` can be set to false. The specific benchmark methodology, number of representative problems per size class, and statistical confidence requirements are implementation planning decisions.

**Resolution required before:** Setting `is_provisional = false` in the capability profile. This affects whether the backend is excluded from certain Scheduler eligibility phases (SPEC-003 FR-5 Phase 2).

**Blocking:** Does not block SPEC-013 acceptance or initial implementation. Blocks `is_provisional = false` declaration.

---

### OQ-3: Degenerate Capacity Failure Frequency in Synthetic Workload

**Classification:** Future Specification

**Question:** How frequently does the degenerate capacity assignment failure (stops remaining after all vehicles exhausted) occur across the SPEC-002 synthetic workload, and should the specification be updated to address it?

**Context:** FR-5 documents the failure condition and defines it as `Failed` with `InternalError`. The algorithm does not backtrack. Whether this failure condition is common or rare on SPEC-002-generated problems depends on the workload generator's capacity utilization distribution. If the failure is frequent, it may indicate that pure nearest-neighbor is insufficient as a baseline and a lightweight fallback (such as a round-robin redistribution pass for unassigned stops) should be added to the algorithm specification.

**Resolution:** Determine failure frequency empirically during initial implementation testing. If the failure rate is above a threshold acceptable for a baseline solver, escalate to specification revision (which would increment `specification_version`).

**Blocking:** Does not block SPEC-013 acceptance. Informs whether SPEC-013 requires a revision after initial implementation testing.

---

# Acceptance Checklist

**SPEC-011 Framework Obligations (FR-12):**

- [ ] Metadata: All FR-3 metadata fields present in the solver metadata table (backend_id, display_name, backend_category, implementation_language, determinism_class, supported_contract_version, specification_version)
- [ ] Problem Statement: States what optimization problem this backend solves, its algorithmic approach, and why it belongs in the MVP inventory
- [ ] Solver Classification: Backend category declared as `classical_deterministic`; determinism class declared as `Deterministic`; infeasibility proof capability declared as No
- [ ] Algorithm Description (FR-1): Starting location, candidate selection, distance metric, tie-breaking, stop visitation, route completion, and depot return defined with sufficient precision for independent implementation
- [ ] Multi-Vehicle Behavior (FR-2): Vehicle assignment strategy, route construction strategy, capacity handling, behavior on exhausted vehicles, and behavior on unused vehicles all defined
- [ ] Constraint Handling (FR-3): Actively enforced and ignored constraints documented; limitations explicitly stated
- [ ] Capability Profile Declaration (FR-4): All nine fields from SPEC-011 FR-4.1 present; accuracy basis for latency_profile and quality_profile stated; is_provisional = true declared
- [ ] Seed Usage Policy (FR-5): Deterministic classification stated; execution_seed acceptance documented; unconditional determinism stated; prohibited entropy sources confirmed absent; satisfies SPEC-004 FR-1 and SPEC-011 FR-6.5
- [ ] Supported SolverOutcome Values (FR-6): Explicit table of supported outcomes; Infeasible listed as Not Supported; satisfies SPEC-011 FR-5.3
- [ ] RoutePlan Output Requirements (FR-7): Presence per outcome; capacity validity guarantee; stop completeness guarantee; time window non-guarantee; execution_duration_ms obligation confirmed
- [ ] Extension Metadata (FR-8): All three keys documented; raw-data prohibition confirmed; absence on non-Succeeded outcomes stated; satisfies SPEC-011 FR-7.4
- [ ] Failure Model (FR-9): ContractVersionMismatch, degenerate capacity failure, internal error, timeout, and cancellation all defined with trigger conditions and required metadata
- [ ] Performance Characteristics: Expected behavior per size class; basis for estimates stated
- [ ] Testability: Correctness, determinism, outcome compliance, capability profile accuracy, and extension metadata coverage all addressed
- [ ] Open Questions: All three open questions classified; blocking status stated; no open questions with unknown classification remain
- [ ] No individual solver specification obligation contradicts a framework requirement from SPEC-011 FR-1 through FR-11

**Backend-Specific Acceptance Criteria:**

- [ ] The algorithm definition is sufficiently precise that two independent C++ implementations would produce identical route plans for identical inputs
- [ ] The capability profile accurately reflects algorithm behavior (supports_time_windows = false, supports_capacity_constraints = true)
- [ ] The degenerate capacity failure condition is explicitly defined
- [ ] The extension metadata keys are defined and their absence on non-Succeeded outcomes is stated
- [ ] The specification does not define any Scheduler, Worker, or Core behavior (owned by SPEC-003, SPEC-005, SPEC-004)

---

# Definition of Done

This backend is complete when:

- SPEC-013 is in Accepted status (this specification)
- OQ-1 (backend_id naming convention) is resolved
- The backend is implemented as a C++ SolverContract conforming to SPEC-004
- All acceptance criteria in the Testability section pass
- The `solver.execute` span is emitted for every invocation and verifiable in the test environment
- Extension metadata keys `nn.total_distance_km`, `nn.routes_used`, and `nn.capacity_rejections` are present in every Succeeded SolverResponse
- Empirical latency and quality values are measured from the SPEC-002 synthetic workload
- `is_provisional` is set to false (requires OQ-2 resolution)
- The capability profile is registered through the mechanism resolved by SPEC-003 OQ-2
- Engineering review passes
- Specification status is updated to Verified
