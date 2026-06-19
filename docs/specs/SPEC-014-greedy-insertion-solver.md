# SPEC-014: Greedy Insertion Solver

## Metadata

**Feature ID:** SPEC-014

**Title:** Greedy Insertion Solver

**Status:** Proposed

**Author:** Darkhorse286

**Created:** 2026-06-19

**Last Updated:** 2026-06-19

**Supersedes:** None

**Superseded By:** None

**Parent Specification:** SPEC-011 (Backend Solver Specifications)

**Related ADRs:** ADR-005, ADR-006, ADR-008, ADR-009, ADR-010, ADR-011

**Related Specs:** SPEC-001, SPEC-003, SPEC-004, SPEC-005, SPEC-006, SPEC-007, SPEC-010, SPEC-011, SPEC-012

---

**Solver Metadata (SPEC-011 FR-3):**

| Field | Value |
|---|---|
| `backend_id` | `greedy-insertion` |
| `display_name` | Greedy Insertion Heuristic |
| `backend_category` | `classical_deterministic` |
| `implementation_language` | `C++` |
| `determinism_class` | Deterministic |
| `supported_contract_version` | 1 |
| `specification_version` | 1 |

---

# Problem Statement

Project DAEDALUS requires a second classical solver baseline to support meaningful backend comparison before invoking the QUBO simulated annealing backend. The nearest-neighbor heuristic (SPEC-013) provides the cheapest and fastest classical baseline. However, it makes purely local decisions — selecting the nearest unvisited stop from the current vehicle's position — and cannot correct poor early choices. This limits its solution quality to a predictably weak Baseline tier.

A stronger classical construction heuristic is needed to answer the question: *how much solution quality can a deterministic, in-process C++ algorithm achieve before a more expensive stochastic backend is justified?* Without a stronger classical baseline, the system cannot distinguish between "the routing problem requires a stochastic solver" and "any modestly better classical algorithm would have been sufficient."

The greedy insertion heuristic is the second established construction heuristic for the Capacitated Vehicle Routing Problem (CVRP). Rather than greedily extending a route from a current position, it considers every possible insertion of every unassigned stop into every existing route position across all vehicles at each step, selecting the globally cheapest feasible insertion. This produces higher-quality solutions than nearest-neighbor while remaining deterministic, in-process, and polynomial-time.

SPEC-014 defines the authoritative greedy insertion backend implementation. It specifies observable algorithm behavior, the capability profile for Scheduler consumption, the failure model, and reproducibility obligations. It does not define the solver framework, the solver contract, or the Scheduler policy. Those responsibilities belong to SPEC-004, SPEC-011, and SPEC-003, respectively.

---

# Business Value

- Provides a stronger classical baseline than nearest-neighbor, enabling the system to determine whether a modestly better algorithm is sufficient before invoking QUBO simulated annealing
- Quantifies the marginal quality improvement achievable by a better-but-still-cheap deterministic algorithm, directly supporting the evidence thesis that quantum-adjacent backends are not always necessary
- Demonstrates the system's ability to compare multiple classical baselines against a stochastic backend on identical problem instances
- Implements a well-known, auditable algorithm whose output can be independently verified
- Closes the second of three blocking backend implementation prerequisites in SPEC-011 FR-11

---

# Employer Signaling

- System Design
- Optimization
- Reliability Engineering

The greedy insertion heuristic for CVRP requires correctly handling global candidate evaluation across all routes and positions simultaneously, incremental insertion cost computation using a segment-split formula with depot boundary handling, per-vehicle capacity tracking across routes at different lengths, and a four-level deterministic selection rule (lowest cost, then lowest stop_id, then lowest vehicle index, then lowest position index) that eliminates implementation divergence. Specifying these behaviors precisely enough that two independent implementations produce semantically equivalent route plans demonstrates the ability to write implementation-grade behavioral specifications for non-trivial combinatorial algorithms.

---

# Solver Classification

**Backend category:** `classical_deterministic`

**Determinism class:** Deterministic

**Infeasibility proof capability:** No

The greedy insertion heuristic is a polynomial-time construction algorithm with no stochastic computation. Its output is fully determined by the routing problem inputs and the specified tie-breaking rule. It cannot prove that a routing problem has no feasible solution; it can only succeed in producing a complete route plan or fail when no feasible insertion candidate exists.

Classification is consistent with SPEC-011 FR-2.1 and FR-2.2. This classification is a property of the algorithm, not a runtime configuration parameter.

**Timeout behavior:** The algorithm executes insertion steps in sequence. At each step it evaluates all remaining (stop, vehicle, position) candidates. Unlike nearest-neighbor, greedy insertion's per-step evaluation cost grows with problem size, and Large class problems may approach timeout budgets depending on implementation approach. The algorithm may monitor the `execution_timeout_ms` deadline at insertion step boundaries and self-terminate if the deadline is reached before a complete assignment is produced. Whether deadline polling is implemented is an implementation planning decision. The Worker's external timeout backstop (SPEC-005) is the authoritative enforcement mechanism regardless.

**Supported SolverOutcome values:** See FR-9.

---

# Requirements

### FR-1: Algorithm Description

**Description:**

The greedy insertion heuristic constructs a set of vehicle routes by repeatedly evaluating all feasible insertions of all unassigned stops into all existing route positions across all vehicles, then committing the single insertion with the lowest incremental route distance increase. The algorithm begins with all routes empty. This section defines externally observable algorithm behavior. It does not prescribe internal data structures or implementation techniques.

**Initial route state:** All `vehicle_count` vehicle routes begin empty. An empty route represents a vehicle that departs the depot, visits no stops, and returns to the depot immediately. The depot is not included in route sequences (SPEC-004 FR-5). Depot coordinates are used in insertion cost computation (FR-3).

**Insertion step:** At each step, the algorithm evaluates the complete feasible insertion candidate set (FR-2), selects the candidate with the lowest insertion cost (FR-3), breaks ties deterministically (FR-4), and commits the insertion. One stop is inserted per step. The algorithm repeats until all stops are assigned (Succeeded) or the candidate set is empty with stops remaining (Failed).

**Distance metric:** Haversine great-circle distance in kilometers (SPEC-001 FR-5). No alternative metric is used during construction.

**Service duration and time window behavior:** Service duration values and time window fields are not consulted during route construction. Candidate evaluation and insertion cost computation are based exclusively on Haversine distance. Time window feasibility is evaluated post-execution by Core Quality Evaluation (SPEC-007).

**Completion condition:** The algorithm produces a Succeeded response when all stops in the routing problem have been assigned to exactly one vehicle route, satisfying all SPEC-004 FR-5 structural validity requirements.

**Failure condition:** The algorithm produces a Failed response when the feasible insertion candidate set is empty and at least one stop remains unassigned. This condition occurs when every remaining unassigned stop's demand exceeds every vehicle's remaining capacity. The algorithm does not backtrack. See FR-12.

**Acceptance Criteria:**
- Given a routing problem with N stops, V vehicles, and capacity C per vehicle, the algorithm evaluates all feasible (stop, vehicle, position) insertion candidates at each step
- One stop is inserted per step; the algorithm requires exactly N insertion steps to produce a Succeeded response
- Service duration values and time window fields are not consulted during construction
- The resulting route plan never places the depot in any route sequence
- The algorithm terminates with Failed when the candidate set is empty and stops remain unassigned

---

### FR-2: Insertion Candidate Evaluation

**Description:**

The feasible insertion candidate set at each step is the set of all (stop, vehicle, position) triples satisfying:

1. The stop is currently unassigned.
2. Adding the stop's demand to vehicle v's current total assigned demand does not cause the vehicle's total assigned demand to exceed `capacity_per_vehicle`.
3. The position is a valid insertion position within vehicle v's current route (as defined below).

**Capacity eligibility:** Stop s is eligible for insertion into vehicle v at this step if and only if:

```
sum(demand[r] for r in route[v]) + demand[s] ≤ capacity_per_vehicle
```

A stop that exceeds remaining capacity for vehicle v is not a candidate for vehicle v at this step but may be a candidate for other vehicles with sufficient remaining capacity. Capacity infeasibility is evaluated at the (stop, vehicle) level before position-level evaluation begins.

**Valid insertion positions:** For an empty route (route length = 0), there is exactly one valid insertion position: position index 0, which places the stop as the sole route stop.

For a non-empty route with K stops in sequence [s₁, s₂, …, sK], there are K + 1 valid insertion positions:

- Position 0: before s₁
- Position 1: between s₁ and s₂
- …
- Position K − 1: between s_{K-1} and sK
- Position K: after sK

**Candidate set exhaustion:** If the feasible insertion candidate set is empty — no unassigned stop can be inserted into any vehicle route without exceeding capacity — the algorithm encounters the degenerate failure condition defined in FR-12.

**Acceptance Criteria:**
- Only capacity-feasible (stop, vehicle) pairs participate in position-level evaluation
- For a route with K stops, exactly K + 1 insertion positions are defined
- For an empty route, exactly one insertion position (index 0) is evaluated
- A stop exceeding one vehicle's remaining capacity may still be evaluated against other vehicles with sufficient remaining capacity

---

### FR-3: Insertion Cost Calculation

**Description:**

The insertion cost of inserting stop X at position p in vehicle v's route is the incremental increase in that vehicle's total route distance caused by the insertion. All distance computations use Haversine distance in kilometers (SPEC-001 FR-5). The depot is treated as a route node for boundary position calculations.

**Route distance convention:** The total distance of a vehicle's route is the sum of Haversine distances from the depot to the first stop, between consecutive stops in sequence, and from the last stop back to the depot.

**Insertion cost formula:** Let A be the node immediately before position p and B be the node immediately after position p, where A is the depot when p = 0 and B is the depot when p = K (the last position in a route with K stops).

```
insertion_cost(X, p) = distance(A, X) + distance(X, B) − distance(A, B)
```

Where:
- `distance(A, X)` is the Haversine distance in kilometers from node A to stop X
- `distance(X, B)` is the Haversine distance in kilometers from stop X to node B
- `distance(A, B)` is the Haversine distance in kilometers from node A to node B (the segment being split by the insertion)

**Empty route insertion cost:** For an empty route, the only valid insertion position is position 0, where A = depot and B = depot:

```
insertion_cost(X, 0) = distance(depot, X) + distance(X, depot)
```

This equals twice the Haversine distance between the depot and stop X, since Haversine distance is symmetric.

**Non-negativity:** By the triangle inequality for Haversine distances, the insertion cost is always non-negative for valid input coordinate pairs.

**Acceptance Criteria:**
- Insertion cost for position p in a non-empty route equals `d(A,X) + d(X,B) − d(A,B)`, where A and B are the adjacent nodes at that position (depot for boundary positions)
- Insertion cost for position 0 in an empty route equals `d(depot,X) + d(X,depot)`
- All distance computations use Haversine distance in kilometers per SPEC-001 FR-5
- Insertion cost is non-negative for all valid input coordinate pairs

---

### FR-4: Deterministic Selection and Tie-Breaking

**Description:**

At each insertion step, the algorithm applies the four-level deterministic selection rule below to identify a unique insertion candidate. Level 1 (lowest cost) determines the selection when costs differ; Levels 2–4 (tie-breaking) are applied in sequence only when earlier levels produce equal results.

**Level 1 — Lowest insertion cost (primary selection):** Select the feasible insertion candidate with the lowest insertion cost as an IEEE 754 double value.

**Levels 2–4 — Tie-breaking (applied in order when Level 1 produces equal results):**

2. **Lowest stop_id:** Among candidates with equal insertion cost, select the candidate inserting the stop with the lowest `stop_id` value. `stop_id` is the unique identifier assigned to each stop in the routing problem (SPEC-001 FR-4).

3. **Lowest vehicle index:** Among candidates with equal insertion cost and equal stop_id, select the candidate inserting into the vehicle with the lowest index. Vehicle index is 0-based, consistent with the RoutePlan route array ordering (SPEC-004 FR-5).

4. **Lowest insertion position index:** Among candidates with equal insertion cost, equal stop_id, and equal vehicle index, select the candidate with the lowest insertion position index within that vehicle's route. Position indices are 0-based as defined in FR-2.

**Fully determined ordering:** The four-level selection rule produces a unique selection for any finite feasible candidate set. No additional disambiguation is required. This ordering avoids implementation divergence across independently developed implementations.

**Acceptance Criteria:**
- Given two candidates with equal insertion cost, the candidate inserting the stop with the lower stop_id is selected
- Given equal insertion cost and equal stop_id, the candidate inserting into the lower-indexed vehicle is selected
- Given equal insertion cost, equal stop_id, and equal vehicle index, the candidate at the lower position index is selected
- Output is fully determined by problem inputs and selection rule; no stochastic component exists

---

### FR-5: Multi-Vehicle Behavior

**Description:**

The greedy insertion algorithm assigns stops to vehicles cooperatively — at each step, the globally cheapest feasible insertion across all vehicles is selected. This differs from nearest-neighbor, which constructs vehicle routes sequentially and independently. All vehicle routes are built concurrently through the global selection process.

**Route initialization:** All vehicle routes begin empty at the start of execution. No vehicle is privileged; all participate in candidate evaluation from the first insertion step.

**Global assignment strategy:** At each step, any unassigned stop may be inserted into any vehicle's route where doing so is capacity-feasible. The algorithm does not pre-assign stops to vehicles. Vehicle assignment emerges from the global greedy selection over all (stop, vehicle, position) candidates at each step.

**Behavior when all stops are assigned before all vehicles are used:** If all stops are assigned with some vehicles still having empty routes, the algorithm terminates with Succeeded. The RoutePlan contains exactly `vehicle_count` route entries; unused vehicles have empty route sequences. Empty routes are valid per SPEC-004 FR-5. No additional insertions occur after all stops are assigned.

**Behavior when no feasible insertion exists with stops remaining:** If, at any step, all remaining unassigned stops are excluded from all vehicles due to capacity constraints, no feasible insertion candidate exists. This is the degenerate failure condition defined in FR-12. The algorithm does not backtrack.

**Vehicle count guarantee:** The RoutePlan always contains exactly `vehicle_count` route entries, including empty entries for unused vehicles.

**Acceptance Criteria:**
- All vehicle routes are empty at algorithm start
- Any vehicle may receive any stop at any step, subject to capacity feasibility
- A Succeeded response contains exactly `vehicle_count` route entries
- Unused vehicles have empty route sequences (valid per SPEC-004 FR-5)
- The algorithm terminates with Failed when no feasible insertion candidate exists and stops remain unassigned

---

### FR-6: Constraint Handling

**Description:**

The greedy insertion algorithm enforces capacity constraints during candidate evaluation and ignores time window and service duration constraints during route construction.

**Capacity constraints (actively enforced):** The algorithm enforces per-vehicle capacity at every candidate evaluation. A (stop, vehicle) pair is excluded from the candidate set when adding the stop's demand to the vehicle's current total assigned demand would exceed `capacity_per_vehicle`. The final route plan satisfies the capacity validity requirements of SPEC-004 FR-5 condition 5 when the algorithm produces a Succeeded outcome. This guarantee holds by construction.

**Service duration constraints (not enforced):** Service duration values (SPEC-001 FR-16) are present in the routing problem but are not consulted during route construction. The algorithm routes by Haversine distance only.

**Time window constraints (not enforced):** Time window fields (SPEC-001 FR-9) are not consulted during route construction. The backend declares `supports_time_windows = false` in its capability profile (FR-7), making it ineligible for time-window-constrained problems in Scheduler eligibility evaluation (SPEC-003 FR-5). This backend should not receive routing problems with time windows.

**Infeasibility detection:** This backend does not detect or prove infeasibility. It is not authorized to return `Infeasible` (SPEC-011 FR-5.2). The only failure condition resulting from problem characteristics is the degenerate no-feasible-insertion case defined in FR-12, which produces `Failed` with `failure_code = InternalError`.

**Explicitly documented limitations:**

1. The algorithm does not guarantee a complete assignment even when total stop demand does not exceed total fleet capacity. Global greedy insertion can produce capacity binding in earlier routes that prevents remaining stops from being assigned. Unlike nearest-neighbor, this degenerate case can occur mid-construction (not only after all vehicles are exhausted), because the cost-optimal insertion sequence may pack a single vehicle route to near capacity before other vehicles receive stops.

2. The algorithm does not account for time window feasibility. A Succeeded response may violate time windows on problems where the Scheduler incorrectly routes a time-windowed problem to this backend.

3. The algorithm does not account for service durations in travel time estimation.

**Acceptance Criteria:**
- Every stop in a Succeeded RoutePlan is assigned to a vehicle whose total stop demand does not exceed `capacity_per_vehicle`
- No (stop, vehicle) insertion is committed when it would cause the vehicle's total demand to exceed `capacity_per_vehicle`
- The backend never returns `Infeasible`
- Service duration and time window fields are not inspected during route construction

---

### FR-7: Capability Profile Declaration

**Description:**

The following capability profile is the registration artifact consumed by the Scheduler (SPEC-003 FR-4, SPEC-011 FR-4). All declared values must accurately reflect this backend's algorithm behavior. Values marked as estimates require empirical measurement before `is_provisional` can be set to false.

| Field | Value | Rationale and Accuracy Basis |
|---|---|---|
| `backend_id` | `greedy-insertion` | Stable unique identifier per SPEC-011 FR-3. Kebab-case per SPEC-011 FR-3.1. |
| `supported_size_classes` | `{Small, Medium, Large}` | The algorithm is polynomial-time in stop count and vehicle count. Expected sub-second execution for all MVP problem sizes based on O(N³) total evaluation count analysis. See Performance Characteristics. |
| `supports_time_windows` | `false` | The algorithm does not incorporate time window constraints during route construction. Candidate evaluation and insertion cost computation are based exclusively on Haversine distance. |
| `supports_capacity_constraints` | `true` | Capacity is enforced at every candidate evaluation step. A (stop, vehicle) pair is excluded when adding the stop's demand to the vehicle's current total assigned demand would exceed `capacity_per_vehicle`. |
| `latency_profile` | Small: < 0.010s, Medium: < 0.100s, Large: < 1.000s | Values expressed in seconds per SPEC-003 FR-4. Conservative algorithmic estimates based on O(N³/6 + V·N²/2) total insertion candidate evaluations. Not derived from empirical measurement. Empirical measurement on SPEC-002 synthetic workload problems is required before `is_provisional = false`. See OQ-1. |
| `quality_profile` | `Competitive` | The greedy insertion heuristic evaluates all feasible insertions globally at each step, consistently producing better solutions than the nearest-neighbor baseline. CVRP literature reports greedy insertion solutions typically 10–20% above optimal, compared to 20–30% for nearest-neighbor. `Competitive` reflects this meaningful improvement over `Baseline`. Empirical measurement on SPEC-002 workloads is required to confirm this classification. See OQ-2. |
| `cost_profile` | `1` | In-process C++ execution with no external dependencies, no network calls, and no paid compute. Cost per invocation is negligible. Value of 1 represents the minimum relative cost unit, consistent with the nearest-neighbor backend. Greedy insertion's higher computational cost relative to nearest-neighbor is captured in `latency_profile`, not `cost_profile`. |
| `is_provisional` | `true` | Latency and quality profile values are pre-implementation estimates. This backend must declare `is_provisional = true` until empirical measurement validates the declared values per SPEC-011 FR-4.2. |
| `supported_contract_version` | `1` | Targets SPEC-004 contract version 1. |

**Accuracy basis:** Latency estimates are derived from algorithmic analysis of insertion evaluation count (approximately O(N³/6 + V·N²/2) total candidate evaluations) and have not been validated against a running implementation. The actual execution duration depends significantly on whether pairwise distances are precomputed, which is an implementation planning decision. Quality classification as `Competitive` is consistent with established CVRP construction heuristic benchmarks. Both values require empirical validation against the SPEC-002 synthetic workload before `is_provisional = false` can be declared.

**Acceptance Criteria:**
- All nine fields from the SPEC-011 FR-4.1 table are present
- `backend_id` matches the solver metadata value
- `supported_contract_version` equals 1
- `is_provisional = true` is declared and remains true until empirical validation is complete
- The accuracy basis for `latency_profile` and `quality_profile` is stated
- `latency_profile` values are expressed in seconds per SPEC-003 FR-4

---

### FR-8: Seed Usage Policy

**Description:**

The greedy insertion heuristic is a deterministic construction algorithm. It contains no stochastic computation. No PRNG is required or used.

**Determinism class:** Deterministic

**Execution seed usage:** The `execution_seed` field in the SolverRequest (SPEC-004 FR-2) is accepted without error, as required by SPEC-004 FR-11.5 for all backends regardless of classification. The field is not used in any computation. The algorithm's output is determined entirely by the routing problem's coordinates, demands, vehicle count, capacity per vehicle, and the deterministic tie-breaking rule defined in FR-4.

**Determinism within a conforming execution environment:** This backend produces identical route plans for identical routing problem inputs on every invocation within a given conforming C++17 execution environment, regardless of `execution_seed` value, invocation count, or process state.

**Cross-platform reproducibility:** Route plans produced from identical inputs are semantically equivalent across conforming C++17 toolchains per ADR-010 Decision 2. Insertion cost computation uses transcendental functions (sin, cos, asin, sqrt) through Haversine distance. Per ADR-010 Decision 2, values derived from transcendental functions satisfy semantic equivalence within IEEE 754 double precision but are not required to be bitwise identical across platforms. The reproducibility guarantee for this backend follows ADR-010's semantic equivalence scope.

**Prohibited entropy sources:** No entropy source is used in any part of this backend's execution. System time, process ID, OS random sources (`/dev/urandom`, `getrandom(2)`, `std::random_device`), and hardware entropy are not consulted during route construction, candidate evaluation, insertion cost computation, tie-breaking, or any other algorithmic step.

This section satisfies the documentation obligation stated in SPEC-004 FR-1 and SPEC-011 FR-6.5.

**Acceptance Criteria:**
- Two SolverRequests with identical routing problems and different `execution_seed` values produce identical SolverResponse solutions
- Two SolverRequests with identical routing problems and identical `execution_seed` values produce identical SolverResponse solutions
- The backend accepts any `execution_seed` value without error

---

### FR-9: Supported SolverOutcome Values

**Description:**

This backend is classified `classical_deterministic`. Per SPEC-011 FR-5.1, it must support `Succeeded`, `Timeout`, `Cancelled`, and `Failed`. It must not support `Infeasible`.

| Outcome | Supported | Trigger Condition |
|---|---|---|
| `Succeeded` | Yes | All stops assigned to vehicles; RoutePlan satisfies all SPEC-004 FR-5 structural validity requirements |
| `Timeout` | Yes | Execution deadline exceeded; the backend may self-terminate at insertion step boundaries, or the Worker may enforce the timeout externally per SPEC-004 FR-10 |
| `Cancelled` | Yes | Worker signals cancellation before route construction completes |
| `Failed` | Yes | Contract version mismatch; or no feasible insertion candidate exists and stops remain unassigned; or any internal error preventing route plan production |
| `Infeasible` | **Not Supported** | This backend cannot prove infeasibility. Prohibited per SPEC-011 FR-5.2. |

**Notes on Timeout:** Unlike nearest-neighbor, greedy insertion's O(N³) evaluation growth means Large class problems may approach timeout budgets depending on implementation approach. Self-termination at insertion step boundaries is recommended but is an implementation planning decision. A self-terminating backend returns `Timeout` with no solution if no complete RoutePlan exists at the time of deadline detection. Construction heuristics produce no intermediate complete solutions during construction; a mid-construction timeout always produces a Timeout response with no route plan. The Worker's external backstop remains authoritative.

**Notes on Cancelled:** If cancellation arrives before construction is complete, the backend returns `Cancelled` with no solution. If construction has already produced a complete RoutePlan satisfying SPEC-004 FR-5 at the time of cancellation, the backend returns `Cancelled` with the complete RoutePlan included.

This section satisfies the supported outcome declaration requirement in SPEC-011 FR-5.3.

**Acceptance Criteria:**
- The backend never returns `Infeasible`
- The backend returns `Succeeded` with a structurally valid RoutePlan when all stops are assigned
- The backend returns `Failed` with `InternalError` when no feasible insertion candidate exists and stops remain unassigned
- `Timeout` responses never include a RoutePlan
- `Cancelled` responses include a RoutePlan only if a complete RoutePlan satisfying SPEC-004 FR-5 existed before the cancellation signal

---

### FR-10: RoutePlan Output Requirements

**Description:**

The greedy insertion algorithm produces a RoutePlan conforming to SPEC-004 FR-5. No backend-specific relaxations of the structural requirements are declared.

**RoutePlan presence per outcome:**

| Outcome | RoutePlan Present? | Requirements When Present |
|---|---|---|
| `Succeeded` | Required | All SPEC-004 FR-5 conditions 1–5 satisfied |
| `Timeout` | Absent | Deliberate narrowing of SPEC-004 FR-10 Optional allowance; see note below |
| `Cancelled` | Optional | Present only if a complete RoutePlan satisfying all SPEC-004 FR-5 conditions existed before cancellation |
| `Failed` | Absent | Never present |
| `Infeasible` | Not Applicable | Outcome not supported |

**Notes on Timeout RoutePlan narrowing:** SPEC-004 FR-10 and SPEC-011 FR-7.1 permit a Timeout response to include a complete solution (declared Optional), enabling iterative improvement backends to return their best-found solution when time expires. SPEC-014 narrows this to Absent. This narrowing is deliberate and algorithmically justified: the greedy insertion algorithm has no intermediate complete solutions. Each insertion step assigns exactly one stop; until all N stops are inserted, no state within the algorithm satisfies SPEC-004 FR-5 condition 2 (all stops assigned). A construction that completes all N insertions before the deadline exits as Succeeded, not Timeout. A mid-construction timeout produces a Timeout response with no route plan. The SPEC-004 FR-10 Optional permission is vacuously empty for this backend class. When SPEC-013 is next revised, its Timeout RoutePlan declaration (currently Optional) should be reviewed for consistency with this reasoning, as the nearest-neighbor heuristic is subject to the same algorithmic property.

**Capacity validity guarantee on Succeeded:** The algorithm's candidate evaluation rule guarantees that no (stop, vehicle) insertion is committed when it would cause the vehicle's total demand to exceed `capacity_per_vehicle`. This invariant holds by construction; the RoutePlan in every Succeeded response is capacity-valid. The Worker validates this condition post-receipt per SPEC-004 FR-5.

**Stop assignment completeness guarantee on Succeeded:** A Succeeded response guarantees that every stop in the routing problem appears in exactly one route. The algorithm transitions to Succeeded only when all stops have been inserted into routes.

**Vehicle count guarantee:** The RoutePlan always contains exactly `vehicle_count` route entries. Unused vehicles have empty route sequences. No route entry is omitted.

**Route sequence semantics:** Each vehicle's route sequence represents the travel order — the sequence of stops visited between depot departure and depot return. Because greedy insertion may insert stops at any valid position within a route (not exclusively appending to the end), the stop ordering within each route reflects accumulated position insertions, not insertion step order.

**Time window non-guarantee:** A Succeeded response does not guarantee time window feasibility. Time window feasibility is evaluated by Core Quality Evaluation (SPEC-007).

`execution_duration_ms` is always populated. It reflects wall-clock execution time from backend execution start to RoutePlan finalization, including any distance precomputation performed by this backend if precomputation is implemented. SolverRequest deserialization and SolverResponse serialization are excluded.

**Acceptance Criteria:**
- Every Succeeded response contains a RoutePlan with `routes.size()` = `vehicle_count`
- No stop appears in more than one route in a Succeeded response
- Every stop in the routing problem appears in exactly one route in a Succeeded response
- No route in a Succeeded response has total stop demand exceeding `capacity_per_vehicle`
- `execution_duration_ms` is present in every SolverResponse regardless of outcome

---

### FR-11: Extension Metadata

**Description:**

The greedy insertion backend produces the following `extension_metadata` keys in the SolverResponse on `Succeeded` outcomes. All values are string-encoded.

| Key | Type (encoded as string) | Present When | Description |
|---|---|---|---|
| `gi.total_distance_km` | float64 | Succeeded | Sum of Haversine distances across all vehicle routes, including each vehicle's departure leg from the depot to its first assigned stop and return leg from its last assigned stop to the depot. Vehicles with no stops (empty routes) contribute 0.0 to this sum. Units: kilometers. This is the solver's internal distance objective, not the normalized quality metric used by Core (SPEC-007). |
| `gi.routes_used` | uint32 | Succeeded | Number of vehicles with at least one assigned stop. Vehicles with empty routes are not counted. |
| `gi.insertion_evaluations` | uint32 | Succeeded | Total number of insertion cost evaluations performed across the complete construction run. Counting semantics: one evaluation is counted each time the insertion cost formula (FR-3) is applied for a specific (stop, vehicle, position) triple. Capacity-infeasible (stop, vehicle) pairs are excluded before position-level evaluation begins and do not contribute to this count. Only triples whose insertion cost is computed — whether or not that insertion is ultimately selected — are counted. |
| `gi.capacity_rejections` | uint32 | Succeeded | Total number of (stop, vehicle) pair rejection events where a stop was excluded from a vehicle's candidate positions because adding the stop's demand to the vehicle's current total assigned demand would exceed `capacity_per_vehicle`. Counting semantics: at each insertion step, for each unassigned stop s and each vehicle v, if `total_demand[v] + demand[s] > capacity_per_vehicle`, this count is incremented by one. The same (stop, vehicle) pair may be rejected at multiple steps; each rejection at each step is counted independently. |

Extension metadata is not produced on `Failed`, `Timeout`, or `Cancelled` outcomes.

`extension_metadata` must not contain routing problem raw data (geographic coordinate arrays, stop identifier lists, demand arrays, or other problem input fields). The four keys above contain only derived summary values safe for evidence log inclusion (SPEC-004 FR-13, SPEC-001 Security Considerations).

Consumers must silently ignore any keys not listed here. Unrecognized keys are not contract-stable across `specification_version` changes.

**Acceptance Criteria:**
- All four keys are present in the `extension_metadata` of every Succeeded SolverResponse
- Key values are non-empty strings encoding valid numeric values
- No key contains routing problem raw data (coordinates, demands, stop lists)
- `gi.insertion_evaluations` counts only position-level cost evaluations; capacity-infeasible (stop, vehicle) pairs are excluded from this count
- `gi.capacity_rejections` counts one per (stop, vehicle) pair per step at which capacity infeasibility is detected, regardless of position-level evaluation
- Extension metadata is absent on Failed, Timeout, and Cancelled responses

---

### FR-12: Failure Model

**Description:**

This section defines all failure conditions specific to the greedy insertion backend. General framework failure obligations are defined in SPEC-011 FR-8.

**Contract version mismatch:**
- **Trigger:** The `contract_version` in the SolverRequest does not equal 1.
- **Outcome:** `Failed`, `failure_code = ContractVersionMismatch`
- **Behavior:** The backend returns `ContractVersionMismatch` immediately before processing the routing problem. No route construction begins.
- **Metadata:** `failure_message` must identify the received version and the expected version (1).

**Degenerate no-feasible-insertion failure:**
- **Trigger:** At any insertion step, the feasible insertion candidate set (FR-2) is empty and at least one stop remains unassigned. This occurs when every remaining unassigned stop's demand exceeds every vehicle's remaining capacity across all routes. The algorithm does not backtrack or retry with alternative previous insertions.
- **Outcome:** `Failed`, `failure_code = InternalError`
- **Behavior:** The algorithm terminates immediately. No partial RoutePlan is produced.
- **Metadata:** `failure_message` must identify the failure cause (`"greedy-insertion construction exhausted feasible insertions"`) and the count of unassigned stops at the time of failure.
- **Note:** Unlike the nearest-neighbor degenerate case (which can only occur after all vehicles are sequentially exhausted), this failure can occur mid-construction because the global cost-optimal insertion sequence may pack a single vehicle route tightly before other vehicles receive stops. The condition is rare at moderate capacity utilization on SPEC-002 workloads. See Quality Expectations and OQ-3.

**Internal error:**
- **Trigger:** Any unexpected condition during route construction that prevents the algorithm from producing a RoutePlan (for example, an insertion cost computation producing NaN or infinite values from coordinates that are structurally valid per SPEC-001 FR-5 but produce arithmetic edge cases).
- **Outcome:** `Failed`, `failure_code = InternalError`
- **Behavior:** The backend returns `Failed` immediately. No partial RoutePlan is present.
- **Metadata:** `failure_message` must provide a diagnostic description identifying the failure cause.

**Timeout:**
- **Trigger:** Execution deadline reached. The backend may detect the deadline at insertion step boundaries and self-terminate (see FR-1 timeout behavior note and FR-9 timeout notes). When the Worker externally enforces the timeout, the Worker constructs the Timeout response on the backend's behalf per SPEC-004 FR-10.
- **Outcome:** `Timeout`, `failure_code = ExecutionTimeout`
- **Behavior:** No complete solution is present in the Timeout response. Construction heuristics produce no intermediate complete solutions; a mid-construction timeout cannot produce a partial RoutePlan that satisfies SPEC-004 FR-5 structural validity.

**Cancellation:**
- **Trigger:** Worker cancellation signal received before construction completes.
- **Outcome:** `Cancelled`, `failure_code = ExecutionCancelled`
- **Behavior:** If a complete RoutePlan satisfying SPEC-004 FR-5 existed before the cancellation signal, it may be included. Otherwise, the response contains no solution.

All `Failed` responses include a `SolverFailureDetail` with both `failure_code` and `failure_message` populated. The `failure_message` is diagnostic (developer-readable), not user-facing. The backend does not write to stdout or stderr.

**Acceptance Criteria:**
- `ContractVersionMismatch` is returned before any route construction begins on version mismatch
- `InternalError` is returned when the feasible insertion candidate set is empty and stops remain unassigned
- `failure_message` is non-empty on every Failed response
- No route plan is present in any Failed response

---

# Performance Characteristics

The greedy insertion heuristic evaluates insertion candidates iteratively. At each of N insertion steps, the algorithm evaluates all (remaining stop, eligible vehicle, valid position) triples.

**Evaluation count analysis:** The total number of insertion cost evaluations is bounded by approximately O(N³/6 + V·N²/2), where N is stop count and V is vehicle count. At step k (k stops already inserted), the algorithm evaluates up to (N − k) remaining stops, each against eligible vehicles with up to (k + V) total insertion positions across all routes (k existing positions plus one per empty vehicle). For N = 76 stops and V = 10 vehicles, this yields approximately 100,000 insertion cost evaluations across the complete construction run.

**Per-evaluation cost:** Each evaluation applies the insertion cost formula from FR-3, requiring three Haversine distance computations (d(A,X), d(X,B), d(A,B)) or three lookups from a precomputed distance matrix. Whether pairwise distances are precomputed is an implementation planning decision that significantly affects execution duration.

**Expected execution duration per size class:**

| Size Class | Stop Range | Expected Duration | Basis |
|---|---|---|---|
| Small | 1–25 stops | < 0.010s (< 10ms) | Algorithmic estimate; O(N³) total evaluations trivial at N ≤ 25 |
| Medium | 26–75 stops | < 0.100s (< 100ms) | Algorithmic estimate; O(N³) evaluation count significant; estimate assumes efficient distance computation |
| Large | 76+ stops | < 1.000s (< 1,000ms) | Algorithmic estimate; empirical validation required; actual duration depends heavily on implementation approach |

These durations correspond to the `latency_profile` values registered in the capability profile (FR-7), expressed in seconds per SPEC-003 FR-4. All values are conservative pre-implementation estimates. Empirical measurement on SPEC-002 synthetic workload problems of each size class is required before `is_provisional` is set to false.

**Comparison with nearest-neighbor:** Greedy insertion performs O(N³/6 + V·N²/2) total insertion evaluations versus nearest-neighbor's O(S·N²) candidate evaluations. Greedy insertion's asymptotically higher evaluation count is the cost of its globally-aware selection policy, which produces better solution quality. The evaluation count difference is bounded and manageable at MVP problem sizes.

**Memory:** The dominant memory allocation depends on implementation approach. A full pairwise distance matrix for N stops + 1 depot requires O((N+1)²) doubles. For N = 100 stops, this is approximately 88 KB. No practical constraint at MVP scale.

---

# Quality Expectations

**Route quality expectations:** The greedy insertion heuristic consistently produces better routes than the nearest-neighbor baseline by selecting globally among all insertion positions at each step rather than greedily extending from a current position. CVRP literature consistently reports greedy insertion solutions within 10–20% of optimal, compared to 20–30% for nearest-neighbor. The `Competitive` quality profile classification reflects this expected improvement. Empirical confirmation on SPEC-002 workloads is required before `is_provisional = false`.

**Known weaknesses:**

1. **Degenerate capacity binding:** At high capacity utilization with heterogeneous stop demands, the globally cost-optimal insertion sequence can pack earlier routes tightly, exhausting feasible insertion positions before all stops are assigned. This produces a Failed outcome. Unlike nearest-neighbor, this failure can occur mid-construction because the algorithm is not bound to sequential per-vehicle construction.

2. **No improvement phase:** The algorithm commits to each insertion permanently. No local search improvement (2-opt, 3-opt, or Or-opt) is performed after construction. Reaching near-optimal quality would require an improvement phase beyond this specification's scope.

3. **Greedy global optimality is not plan-level optimality:** Each insertion is globally cheapest at the time it is made. Earlier insertions constrain the solution space for later steps in ways the algorithm does not anticipate or correct.

**Workload classes where greedy insertion performs well:** Problems with moderate capacity utilization (0.5–0.8), where the globally cheapest insertion consistently avoids the route-crossing inefficiencies that nearest-neighbor produces. Problems with geographic clustering where the choice of insertion position significantly affects total route length.

**Workload classes where greedy insertion performs less well:** High-capacity-utilization problems with heterogeneous demands (increased degenerate failure risk), problems requiring backtracking to achieve optimality, and very large problem sizes where O(N³) evaluation cost strains the latency budget.

---

# Non-Requirements

- This specification does not define time window feasibility checking. Time windows are evaluated by Core Quality Evaluation (SPEC-007).
- This specification does not define route improvement heuristics. The algorithm performs one greedy insertion pass without post-construction improvement.
- This specification does not define backtracking or reassignment of committed insertions. Once a stop is inserted, the insertion is permanent.
- This specification does not define multi-depot behavior. Single depot only, per SPEC-001 FR-3.
- This specification does not define heterogeneous vehicle fleet handling. All vehicles have identical capacity, per SPEC-001 FR-2.
- This specification does not define the distance precomputation strategy. Whether pairwise distances are precomputed in a matrix or computed on demand is an implementation planning decision.
- This specification does not define how workload features are computed. That is SPEC-010's responsibility.
- This specification does not define backend selection policy. That is SPEC-003's responsibility.
- This specification does not define evidence persistence. That is SPEC-006's responsibility.
- This specification does not define quality evaluation metrics. That is SPEC-007's responsibility.
- This specification does not alter the SolverContract schema. That is SPEC-004's responsibility.

---

# Assumptions

1. The routing problem received by this backend has been validated by Core per ADR-009. The backend does not re-validate the routing problem's structural constraints.

2. All vehicles have identical capacity. The algorithm applies the same `capacity_per_vehicle` constraint uniformly across all vehicles.

3. Haversine great-circle distance is an acceptable construction metric for synthetic workloads. This is a known limitation of the routing problem model (SPEC-001 FR-5), not the algorithm.

4. The degenerate no-feasible-insertion failure is rare in the SPEC-002 synthetic workload at typical capacity utilization ratios. The SPEC-002 generator produces problems where total stop demand does not exceed total fleet capacity; whether greedy insertion avoids degenerate binding depends on problem structure.

5. For MVP scope, the algorithm is expected to complete within typical timeout budgets at all three supported size classes. Empirical measurement may revise this assumption for Large class problems depending on implementation approach and the declared `execution_timeout_ms` budget.

---

# Constraints

1. This backend implements SPEC-004 contract version 1. `contract_version` mismatch must be detected before any routing problem processing begins (SPEC-011 FR-8.3).

2. `execution_seed` must be accepted without error and must not be used in any computation (SPEC-004 FR-11.5, SPEC-011 FR-6.1).

3. No backend-specific logic may exist in the Scheduler, Worker, or Core contract-handling code. This backend is accessed exclusively through the normalized SolverContract interface (ADR-008).

4. `extension_metadata` must not contain routing problem raw data. Only derived summary values may appear (SPEC-004 FR-13, SPEC-011 FR-7.4).

5. The backend must not emit OpenTelemetry spans, metrics, or structured logs directly. Diagnostic output is returned through `SolverFailureDetail.failure_message` and `extension_metadata` only (SPEC-011 FR-9.2).

6. The backend must not write to stdout or stderr (SPEC-011 FR-9.2).

7. `execution_duration_ms` must reflect actual wall-clock execution time from backend execution start to RoutePlan finalization, including any distance precomputation performed by this backend if precomputation is implemented. SolverRequest deserialization and SolverResponse serialization are excluded.

8. The algorithm must not return `Infeasible` under any condition (SPEC-011 FR-5.2).

---

# Inputs

**SolverRequest** (SPEC-004 FR-2):

| Field | How Used |
|---|---|
| `routing_problem.stops` | Stop coordinates, demands, stop_id values — used in candidate evaluation and insertion cost computation |
| `routing_problem.depot` | Depot coordinates — used in insertion cost computation for boundary positions (A = depot at position 0; B = depot at final position) |
| `routing_problem.vehicle_count` | Determines number of routes produced |
| `routing_problem.capacity_per_vehicle` | Enforced during candidate evaluation |
| `execution_timeout_ms` | Execution deadline; backend may self-terminate at insertion step boundaries; Worker enforces externally |
| `execution_seed` | Accepted, not used (deterministic backend) |
| `contract_version` | Validated before processing; must equal 1 |
| `job_id`, `decision_id`, `backend_id` | Correlation only; do not affect route construction |

Fields not used in construction: `routing_problem.time_window_open`, `routing_problem.time_window_close`, `routing_problem.service_duration` (see FR-6).

---

# Outputs

**SolverResponse** (SPEC-004 FR-3):

On `Succeeded`:
- `outcome = Succeeded`
- `solution`: RoutePlan with `vehicle_count` routes, all stops assigned, capacity valid
- `statistics.execution_duration_ms`: Wall-clock backend execution time in milliseconds
- `extension_metadata`: Keys `gi.total_distance_km`, `gi.routes_used`, `gi.insertion_evaluations`, `gi.capacity_rejections`
- `failure`: Absent

On `Failed`:
- `outcome = Failed`
- `failure.failure_code`: `ContractVersionMismatch` or `InternalError`
- `failure.failure_message`: Diagnostic description
- `solution`: Absent
- `extension_metadata`: Absent

On `Timeout`:
- `outcome = Timeout`
- `failure`: Present with `ExecutionTimeout` failure code
- `solution`: Absent (construction heuristics produce no intermediate complete solutions)
- `extension_metadata`: Absent

On `Cancelled`:
- `outcome = Cancelled`
- `failure`: Present with `ExecutionCancelled` failure code
- `solution`: Optionally present if a complete RoutePlan satisfying SPEC-004 FR-5 existed before the cancellation signal
- `extension_metadata`: Absent

---

# Architectural Impact

| Component | Impact | Notes |
|---|---|---|
| Domain Layer (C++ Core / backends) | Yes | SPEC-014 defines a new C++ SolverContract implementation |
| Scheduler (SPEC-003) | Yes | Capability profile must be registered before backend is eligible for selection |
| Solver Contract (SPEC-004) | None | SPEC-014 satisfies SPEC-004 FR-1 obligations; no schema changes |
| Worker (SPEC-005) | None | No lifecycle changes; standard invocation, timeout, and cancellation handling applies |
| Evidence Log (SPEC-006) | None | Evidence persistence is unchanged; `extension_metadata` keys pass through |
| Quality Evaluation (SPEC-007) | None | Core evaluates quality from normalized RoutePlan; no changes required |
| Feature Extraction (SPEC-010) | None | Workload features are computed before backend selection; no changes |
| Persistence Schema (SPEC-012) | None | No new tables; `extension_metadata` stored in existing JSONB column |
| API Layer | None | No new API surface |
| Observability | None | Worker-emitted `solver.execute` span (SPEC-004 FR-15); no additional instrumentation required |
| Security | None | No new trust boundaries; `extension_metadata` raw-data prohibition applies per FR-11 |
| Configuration | Yes | Capability profile must be registered through the mechanism resolved by SPEC-003 OQ-2 |

**SPEC-011 FR-11:** The MVP backend inventory lists `greedy-insertion` as "Required; not yet written." SPEC-014 satisfies this prerequisite. SPEC-011 FR-11 should be updated to reflect this specification's status.

---

# Testability

**SolverContract conformance:**

1. **Structural validity:** A Succeeded SolverResponse contains a RoutePlan where: (a) `routes.size()` = `vehicle_count`; (b) every stop_id appears exactly once across all routes; (c) no route's total demand exceeds `capacity_per_vehicle`; (d) no route contains the depot.

2. **Determinism:** Two SolverRequests with identical routing problems and different `execution_seed` values produce identical SolverResponse solutions.

3. **Seed acceptance:** The backend accepts any `execution_seed` value without error and produces identical output regardless of seed value.

4. **ContractVersionMismatch:** A SolverRequest with `contract_version` != 1 returns `Failed` with `ContractVersionMismatch` before any routing problem processing.

5. **Trivial case:** A routing problem with one stop and one vehicle produces a Succeeded response with a single route containing that stop. Insertion cost equals `d(depot, stop) + d(stop, depot)`.

6. **Empty vehicle routes:** A routing problem with more vehicles than stops produces a Succeeded response where some routes are empty.

**Algorithm correctness:**

7. **Insertion cost correctness:** For a known route configuration, the insertion cost computed for a (stop, vehicle, position) candidate equals `d(A, stop) + d(stop, B) − d(A, B)` per FR-3, verified by independent calculation. Depot boundary cases (position 0 and position K) must be included in verification.

8. **Empty route insertion cost:** For an empty route, the insertion cost equals `d(depot, stop) + d(stop, depot)`.

9. **Global minimum selection:** Given a routing problem with multiple vehicles where an unassigned stop has different insertion costs across vehicles (for example, cost X km in vehicle 0 and cost Y km in vehicle 1, with Y < X), the algorithm selects the candidate in vehicle 1 regardless of vehicle iteration order. The algorithm evaluates all (stop, vehicle, position) candidates globally and selects the minimum-cost candidate across all vehicles, not the minimum-cost candidate within any single vehicle's routes.

10. **Tie-breaking — stop_id:** On a problem constructed so that two stops produce equal insertion costs at the same position in the same vehicle route, the stop with the lower stop_id is selected.

11. **Tie-breaking — vehicle index:** On a problem constructed so that the same stop has equal insertion cost in two different vehicles, the stop is inserted into the lower-indexed vehicle.

12. **Tie-breaking — position index:** On a problem constructed so that the same stop has equal insertion cost at two positions within the same vehicle's route, the stop is inserted at the lower position index.

13. **Capacity enforcement:** No Succeeded response contains a route where total stop demand exceeds `capacity_per_vehicle`. Verified across all SPEC-002 synthetic workload sizes and constraint configurations.

14. **No-feasible-insertion failure:** A routing problem constructed to exhaust all feasible insertions before all stops are assigned returns `Failed` with `InternalError`. The `failure_message` identifies the cause and the count of unassigned stops.

**Capability profile accuracy (required before `is_provisional = false`):**

15. **Latency profile validation:** Empirical execution duration on representative SPEC-002 problems of each size class is within the declared `latency_profile` bounds (< 0.010s, < 0.100s, < 1.000s per size class). Duration measured from backend execution start, including any distance precomputation performed, to RoutePlan finalization.

16. **Quality classification:** Route quality on SPEC-002 benchmark problems is consistent with `Competitive` classification relative to the nearest-neighbor baseline and relative to known-optimal or best-known solutions on standard CVRP benchmark instances.

**Extension metadata:**

17. **Key presence:** All four `extension_metadata` keys are present in every Succeeded SolverResponse.

18. **gi.total_distance_km accuracy:** The reported value equals the sum of Haversine distances computed from route sequences in the RoutePlan, including depot-departure and depot-return legs for each vehicle with at least one stop. Empty vehicle routes contribute 0.0.

19. **gi.routes_used accuracy:** The reported value equals the count of routes with at least one assigned stop.

20. **gi.insertion_evaluations accuracy:** The reported count equals the total number of (stop, vehicle, position) triples whose insertion cost was computed during construction. Capacity-infeasible (stop, vehicle) pairs must not be counted.

21. **gi.capacity_rejections accuracy:** The reported count equals the total number of (stop, vehicle) pair rejection events due to capacity infeasibility across all insertion steps, counted once per (stop, vehicle) pair per step at which rejection occurs.

**Observability:**

22. **solver.execute span:** A `solver.execute` OTel span is emitted by the Worker for every SolverRequest dispatch. All required attributes (SPEC-004 FR-15) are present. Verified in integration test.

---

# Observability Requirements

The `solver.execute` span is emitted by the Worker for every invocation of this backend (SPEC-004 FR-15). The backend's contribution to this span is:

| Span Attribute | Source |
|---|---|
| `outcome` | `SolverResponse.outcome` |
| `execution_duration_ms` | `SolverResponse.statistics.execution_duration_ms` |

**Diagnostic questions this backend enables through `extension_metadata`:**

1. What total route distance did the greedy insertion construction produce? (`gi.total_distance_km`) — enables direct comparison with `nn.total_distance_km` on the same problem.
2. How many vehicles were actually used? (`gi.routes_used`)
3. How much evaluation work did the algorithm perform? (`gi.insertion_evaluations`) — supports latency analysis and complexity validation.
4. How heavily did capacity constraints constrain the search? (`gi.capacity_rejections`)

The backend does not emit OpenTelemetry spans, metrics, or structured logs. Diagnostic output is confined to `SolverFailureDetail.failure_message` on failure and `extension_metadata` on success. The backend does not write to stdout or stderr.

---

# Security Considerations

**Seed authority:** The backend uses no entropy source. There are no reproducibility-critical stochastic paths (SPEC-004 FR-11, ADR-010 Decision 4).

**Extension metadata safety:** The four `extension_metadata` keys contain only derived summary values. No key contains geographic coordinate arrays, stop identifier lists, demand arrays, or any other raw routing problem input. This complies with the raw-data prohibition in SPEC-004 FR-13 and SPEC-001 Security Considerations.

**Input trust:** The routing problem arrives pre-validated by Core (ADR-009). The backend trusts the problem's structural validity and does not re-validate coordinates, demands, or fleet parameters.

**No external access:** The backend has no access to external systems, networks, files, or queues. It operates exclusively on the in-process SolverRequest data.

**No stdout/stderr:** Diagnostic output is confined to SolverResponse contract fields, preventing routing problem data from escaping through uncontrolled channels.

---

# Performance Considerations

**Evaluation count growth:** Total insertion evaluation count grows as O(N³/6 + V·N²/2). For Large class problems (N = 76+), this represents hundreds of thousands of evaluations. Whether these are cheap (precomputed distance lookups) or expensive (on-demand Haversine computation) significantly impacts `execution_duration_ms`. Implementation planning should consider distance precomputation for Medium and Large class problems.

**Distance computation cost:** Each insertion cost evaluation requires up to three Haversine distance computations. For N + 1 nodes, a full pairwise distance matrix requires (N+1)(N+2)/2 Haversine calculations upfront. Implementation planning should weigh precomputation cost against the per-evaluation on-demand computation cost.

**Timeout risk:** Unlike nearest-neighbor, greedy insertion's O(N³) evaluation growth means Large class problems may approach timeout budgets depending on implementation approach. Empirical measurement should establish actual execution duration against declared `latency_profile` bounds before `is_provisional = false`.

Do not invent specific latency targets. The areas above require measurement during implementation. See Performance Characteristics for pre-implementation estimates.

---

# Documentation Updates Required

**SPEC-003:**
- The capability profile for `greedy-insertion` must be registered through the mechanism resolved by SPEC-003 OQ-2 before this backend becomes eligible for Scheduler selection. No SPEC-003 schema change is required.

**SPEC-011:**
- FR-11.1 MVP backend inventory: Update the `greedy-insertion` row from "Required; not yet written" to reflect SPEC-014 Draft status, and ultimately Accepted when SPEC-014 is accepted.
- FR-2.2 inventory table: The backend_id naming convention conflict identified in SPEC-013 OQ-1 (snake_case vs. kebab-case) also affects the `greedy_insertion` entry. SPEC-014 follows the normative FR-3.1 definition (`greedy-insertion`, kebab-case). SPEC-011 FR-2.2 should be updated to use kebab-case identifiers when SPEC-013 OQ-1 is resolved.
- OQ-3 (Classical Deterministic Backend Timeout Self-Termination Strategy): SPEC-014 partially informs this question by noting that GI may implement deadline polling at insertion step boundaries due to its O(N³) evaluation growth. However, the obligation level for self-termination (whether the backend MUST, SHOULD, or MAY poll) is a specification-level behavioral decision that has not been established in SPEC-014; it remains deferred to implementation planning. SPEC-011 OQ-3 should be updated to reflect SPEC-014's partial contribution but remains open as an Implementation Planning Decision pending obligation-level determination for both classical deterministic backends.

**SPEC-012:**
- No schema changes required. `extension_metadata` for `greedy-insertion` passes through the existing JSONB column structure in `solver_run_records`.

**docs/architecture.md:**
- No changes required.

---

# Open Questions

### OQ-1: Empirical Latency Profile Values

**Classification:** Implementation Planning Decision

**Question:** What are the empirically measured latency values per size class for the greedy insertion backend?

**Context:** FR-7 declares `is_provisional = true` with conservative pre-implementation estimates (Small < 0.010s, Medium < 0.100s, Large < 1.000s). These are based on O(N³) evaluation count analysis and have not been validated. Actual execution duration depends significantly on whether pairwise distances are precomputed. For Large class problems, the actual latency could vary widely depending on implementation choices. Empirical measurement against the SPEC-002 synthetic workload is required before `is_provisional = false`.

**Blocking:** Does not block SPEC-014 acceptance or initial implementation. Blocks `is_provisional = false` declaration.

---

### OQ-2: Quality Profile — Competitive or Baseline

**Classification:** Implementation Planning Decision

**Question:** Should `quality_profile` be `Competitive` or `Baseline` for the greedy insertion backend before empirical validation?

**Context:** FR-7 declares `quality_profile = Competitive` based on CVRP literature reporting that greedy insertion solutions are typically 10–20% above optimal, compared to 20–30% for nearest-neighbor. However, literature benchmarks may not reflect SPEC-002 synthetic workload characteristics (stop distribution, capacity utilization, problem structure). If empirical measurement shows greedy insertion quality is indistinguishable from nearest-neighbor on SPEC-002 workloads, `Baseline` may be the accurate classification. The distinction matters for Scheduler objective scoring under quality-sensitive modes (SPEC-003 FR-7). If measurement requires changing `quality_profile` from `Competitive` to `Baseline`, this constitutes a behavioral change requiring a `specification_version` increment.

**Blocking:** Does not block SPEC-014 acceptance. Blocks `is_provisional = false` declaration. May require `specification_version` increment if quality_profile is revised after empirical measurement.

---

### OQ-3: Degenerate No-Feasible-Insertion Frequency

**Classification:** Implementation Planning Decision (workload frequency measurement); Project Owner Decision (corrective threshold determination)

**Question:** How frequently does the degenerate no-feasible-insertion failure occur on the SPEC-002 synthetic workload, and should the specification be revised to address it?

**Context:** FR-12 documents this failure as `InternalError`. Unlike the nearest-neighbor degenerate case (which only occurs after all vehicles are sequentially exhausted), greedy insertion can exhaust feasible insertions mid-construction because the global cost-optimal sequence may pack a single vehicle route before others receive stops. Whether this occurs with significant frequency on SPEC-002 workloads depends on the generator's capacity utilization distribution. This question has two components: (1) measuring how frequently the failure occurs on the SPEC-002 synthetic workload — this is an implementation planning concern addressed during initial implementation testing; and (2) determining whether a measured failure rate warrants specification revision (incrementing `specification_version`) — this is a Project Owner concern requiring explicit Project Owner approval based on benchmark evidence. The two components are reported together but resolved separately.

**Blocking:** Does not block SPEC-014 acceptance. Informs whether SPEC-014 requires revision after initial implementation testing.

---

# Acceptance Checklist

**SPEC-011 Framework Obligations (FR-12):**

- [ ] Metadata: All FR-3 fields present (backend_id, display_name, backend_category, implementation_language, determinism_class, supported_contract_version, specification_version)
- [ ] Problem Statement: States the optimization problem, algorithmic approach, and rationale for MVP inclusion
- [ ] Solver Classification: `classical_deterministic`; Deterministic; infeasibility proof capability No
- [ ] Algorithm Description (FR-1): Initial state, insertion step, completion condition, failure condition defined with precision for independent implementation
- [ ] Insertion Candidate Evaluation (FR-2): Capacity eligibility rule, valid insertion positions per route length, candidate set exhaustion condition defined
- [ ] Insertion Cost Calculation (FR-3): Non-empty route formula defined; empty route formula defined; depot boundary handling defined; non-negativity noted
- [ ] Deterministic Selection and Tie-Breaking (FR-4): Four-level rule fully specified; produces unique selection for any finite candidate set
- [ ] Multi-Vehicle Behavior (FR-5): Global assignment strategy, initial state, unused vehicle handling, failure condition defined
- [ ] Constraint Handling (FR-6): Enforced and ignored constraints documented; limitations stated; `Infeasible` prohibition stated
- [ ] Capability Profile Declaration (FR-7): All nine SPEC-011 FR-4.1 fields present; accuracy basis stated; `is_provisional = true`; latency values in seconds per SPEC-003 FR-4
- [ ] Seed Usage Policy (FR-8): Deterministic class stated; `execution_seed` acceptance documented; semantic equivalence scope per ADR-010 Decision 2 stated; prohibited entropy sources confirmed absent; satisfies SPEC-004 FR-1 and SPEC-011 FR-6.5
- [ ] Supported SolverOutcome Values (FR-9): Explicit table; `Infeasible` listed as Not Supported; satisfies SPEC-011 FR-5.3
- [ ] RoutePlan Output Requirements (FR-10): Presence per outcome; capacity validity guarantee; stop completeness guarantee; vehicle count guarantee; `execution_duration_ms` obligation confirmed
- [ ] Extension Metadata (FR-11): All four keys documented; counting semantics for `gi.insertion_evaluations` and `gi.capacity_rejections` explicit; raw-data prohibition confirmed; absent on non-Succeeded outcomes; satisfies SPEC-011 FR-7.4
- [ ] Failure Model (FR-12): ContractVersionMismatch, degenerate no-feasible-insertion, internal error, timeout, and cancellation defined with trigger conditions and required metadata
- [ ] Performance Characteristics: Expected behavior per size class; basis for estimates stated; comparison with nearest-neighbor noted
- [ ] Testability: Insertion cost correctness, global minimum selection across vehicles, tie-breaking (all three levels), capacity enforcement, degenerate failure, capability profile accuracy, and extension metadata counting accuracy all addressed
- [ ] Open Questions: All three open questions classified; blocking status stated; no open questions with unknown classification remain
- [ ] No obligation contradicts a framework requirement from SPEC-011 FR-1 through FR-11

**Backend-Specific Acceptance Criteria:**

- [ ] The insertion cost formula is precisely defined for both empty and non-empty routes, including depot boundary handling at position 0 and position K
- [ ] The four-level selection rule is sufficiently precise that two independent C++17 implementations produce identical selections for any candidate set
- [ ] The algorithm definition is sufficiently precise that two independent implementations produce semantically equivalent route plans for identical inputs within a conforming C++17 execution environment
- [ ] The degenerate no-feasible-insertion failure is distinguished from the nearest-neighbor degenerate case and its mid-construction occurrence is documented
- [ ] `gi.insertion_evaluations` and `gi.capacity_rejections` counting rules are precisely distinguished from each other
- [ ] The specification does not define any Scheduler, Worker, or Core behavior

---

# Definition of Done

This backend is complete when:

- SPEC-014 is in Accepted status
- The backend is implemented as a C++ SolverContract conforming to SPEC-004
- All testability acceptance criteria pass
- The `solver.execute` span is emitted for every invocation and verifiable in the test environment
- All four extension metadata keys are present in every Succeeded SolverResponse
- Empirical latency and quality values are measured from the SPEC-002 synthetic workload
- `is_provisional` is set to false (requires OQ-1 and OQ-2 resolution)
- The capability profile is registered through the mechanism resolved by SPEC-003 OQ-2
- Engineering review passes
- Specification status is updated to Verified
