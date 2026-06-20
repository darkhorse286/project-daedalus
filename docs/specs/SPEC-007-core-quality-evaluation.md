# Feature Specification

## Metadata

**Feature ID:** SPEC-007

**Title:** Core Quality Evaluation

**Status:** Accepted

**Author:** Darkhorse286

**Created:** 2026-06-11

**Last Updated:** 2026-06-19

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-001, ADR-008, ADR-009, ADR-010

**Related Specs:** SPEC-001, SPEC-003, SPEC-004, SPEC-005, SPEC-006

---

# Problem Statement

No specification currently defines the algorithm, data contract, or behavioral obligations governing Core's post-execution quality evaluation of solver-produced route plans. Multiple accepted and proposed specifications defer to this document as the authoritative source:

- **SPEC-003 FR-10** reserves `actual_outcome` and `hindsight_quality` fields in the decision record, stating their "type and semantics defined by the Core specification."
- **SPEC-003 FR-12** reserves the `hindsight_quality` field for Core to populate and states that population logic is "defined by the Core specification."
- **SPEC-004 FR-7** states that "Core evaluates time window feasibility as part of quality evaluation" following a Succeeded solver response.
- **SPEC-005 FR-15** establishes the Worker's obligation to invoke Core quality evaluation and defers the evaluation interface and metric definitions here.
- **SPEC-005 Assumption 9** and **SPEC-006 Assumption 5** both state that Core quality evaluation must be deterministic and that this property "must be established by the Core Quality Evaluation Specification."
- **SPEC-006 FR-7** defers the quality evaluation record schema beyond its minimum fields to this specification.

Without this specification:

- Core has no defined algorithm for determining whether a solver-produced route plan satisfies time window constraints.
- The `hindsight_quality` value in the decision record has no defined type, formula, or semantics.
- The `actual_outcome` type in the decision record has no authoritative definition.
- Scheduler effectiveness analysis (actual vs. predicted outcomes) has no stable foundation.
- Solver comparison, benchmarking, and experiment evaluation have no normalized quality baseline.
- The idempotency model in SPEC-005 FR-14 has no determinism guarantee to rest on.

---

# Business Value

- Defines the authoritative quality baseline enabling solver comparison, benchmarking, and scheduler effectiveness analysis
- Establishes time window feasibility verification as an independent check decoupled from solver implementation
- Provides the `hindsight_quality` value that closes the evidence loop between predicted and actual outcomes
- Enables the project's core thesis demonstration: which backends produce what quality at what cost and latency
- Satisfies the deferred dependencies from SPEC-003, SPEC-004, SPEC-005, and SPEC-006, unblocking those specifications from advancing to Implemented status

---

# Employer Signaling

- System Design
- Optimization
- Reliability Engineering
- Observability

Quality evaluation for an optimization runtime is a domain-modeling problem: defining what "good" means, what "correct" means, and how to measure the difference between what was predicted and what was delivered. This feature demonstrates the ability to define a rigorous, deterministic evaluation contract that other system components can trust as a stable authority.

---

# Domain Concept

The Core Quality Evaluation system is a deterministic computation that accepts a solver-produced route plan and the routing problem that generated it, simulates vehicle execution of the plan, verifies constraint satisfaction, and produces a structured quality assessment.

The evaluation does not re-run optimization. It does not select backends. It does not modify the route plan. It takes what the solver returned and answers: is this plan constraint-satisfying, and how good is it by a normalized measure?

The evaluation is the system's hindsight voice. It is the component that knows, after execution, whether the Scheduler's prediction was accurate, whether the solver actually satisfied time windows, and what quality level was achieved. Every evidence record is incomplete without it.

The evaluation is owned by Core. The Worker invokes it. The Evidence Log persists its results. The Scheduler's decision record is updated with its outputs. None of those components define evaluation behavior.

---

# Requirements

### FR-1: Responsibility Scope and Component Boundaries

**Description:**
SPEC-007 defines Core Quality Evaluation's responsibilities and explicitly allocates the adjacent responsibilities it must not perform.

| Component | Responsibilities at This Boundary | Explicitly Not Responsible For |
|---|---|---|
| **Core Quality Evaluation** (SPEC-007) | Route simulation for time window feasibility; quality metric computation; actual outcome classification; hindsight quality production; regret analysis; evaluation metadata production; evaluation determinism | Solver invocation; backend selection; persisting evaluation results; populating the Evidence Log; generating reports; modifying route plans |
| **Worker** (SPEC-005) | Invoking Core quality evaluation after receiving a structurally valid SolverResponse; passing required inputs; receiving the QualityEvaluationResult; persisting results via the Evidence Log; emitting the `result.evaluate` span | Performing any quality computation; interpreting quality metrics; routing on evaluation results |
| **Scheduler / Core** (SPEC-003) | Reserving `actual_outcome` and `hindsight_quality` field names in the decision record; producing predicted values for regret comparison | Populating hindsight fields at decision time; defining evaluation behavior |
| **Evidence Log** (SPEC-006) | Defining the persistence schema for the quality evaluation record (minimum fields in SPEC-006 FR-7; full schema added in this specification — see Documentation Updates Required) | Producing or interpreting quality evaluation content |
| **Solver Backend** (SPEC-004) | Producing a structurally valid RoutePlan; reporting the honest SolverOutcome | Verifying time window feasibility; computing quality metrics |

**Acceptance Criteria:**
- No Worker code path performs quality metric computation or time window simulation
- No solver backend receives quality evaluation results as feedback during execution
- Core quality evaluation produces identical outputs given identical inputs (FR-10)
- Adding a new quality metric does not require changes to the Worker, Evidence Log schema contract, or Scheduler

---

### FR-2: Evaluation Invocation Contract

**Description:**
The Worker invokes Core quality evaluation under the conditions established in SPEC-005 FR-15. This section defines the complete invocation rules from Core's perspective.

**Invocation conditions:**
Core quality evaluation is invoked when the Worker has received a SolverResponse that passed post-receipt structural validation (SPEC-005 FR-11) and contains a route plan.

| SolverOutcome | Route Plan Present? | Core Quality Evaluation Invoked? |
|---|---|---|
| `Succeeded` | Always Yes | Yes |
| `Timeout` | Optionally Yes | Yes when route plan is present; No otherwise |
| `Cancelled` | Optionally Yes | Yes when route plan is present; No otherwise |
| `Infeasible` | Always No | No |
| `Failed` | Always No | No |

When evaluation is not invoked (no route plan present), the Worker persists null for all quality evaluation fields. This is not an evaluation failure.

**Invocation boundary:**
The Worker is the caller. Core quality evaluation is a synchronous call within the Worker process (Core is an in-process C++ library per SPEC-005 Assumption 3). The Worker passes inputs and receives a `QualityEvaluationResult` in return.

**Scheduler decision precondition:**
The Scheduler decision record is fully populated (`predicted_quality_tier`, `predicted_latency`, and `confidence_score` are non-null) at the time of Core QE invocation. This is guaranteed by SPEC-003 FR-10: these fields are required before a job enters the execution queue. Core QE is only ever invoked on jobs that reached solver execution, which requires a prior Selected Scheduler decision.

**Acceptance Criteria:**
- Core quality evaluation is invoked for every SolverResponse with a structurally valid route plan present, regardless of SolverOutcome (Succeeded, Timeout, or Cancelled with solution)
- Core quality evaluation is never invoked when no route plan is present
- Core quality evaluation is never invoked on a route plan that failed SPEC-005 FR-11 structural validation

---

### FR-3: Evaluation Inputs

**Description:**
The Worker passes the following inputs to Core quality evaluation on every invocation. All inputs are required; Core must reject invocations with missing inputs as Worker construction errors (not evaluation failures).

| Input | Type | Source | Notes |
|---|---|---|---|
| `routing_problem` | RoutingProblem | C++ domain representation (SPEC-001 FR-14) | The validated routing problem, immutable, validated by Core before solver invocation (ADR-009) |
| `route_plan` | RoutePlan | SolverResponse (SPEC-004 FR-5) | The structurally valid route plan from the solver; passed only when present per FR-2 |
| `solver_response` | SolverResponse | SolverResponse (SPEC-004 FR-3) | The complete response including `outcome`, `statistics`, `failure_detail`, `extension_metadata`; used for actual outcome classification and regret analysis. `statistics` is guaranteed non-null for all `Succeeded` responses (SPEC-004 FR-6); it may be null for `Timeout` and `Cancelled` responses. |
| `scheduler_decision` | DecisionRecord | Scheduler decision record (SPEC-003 FR-10) | The decision record produced before execution; provides predicted values for regret analysis (`predicted_latency`, `predicted_quality`, `confidence_score`). Non-null precondition guaranteed by FR-2 invocation contract. |
| `travel_speed_kmh` | float64 | Routing Problem — `RoutingProblem.average_vehicle_speed_kmh` (SPEC-001 FR-17; ODR-1) | The average vehicle speed used for route simulation (FR-5). Required for all routing problem submissions. Must be a positive non-zero value. |

**Acceptance Criteria:**
- Core quality evaluation rejects invocations with a null or absent `route_plan` (the Worker must not invoke evaluation in this case per FR-2)
- Core quality evaluation does not modify any input
- `routing_problem` contains all fields required for simulation: stop coordinates, depot coordinates, time window fields (when present), service durations, vehicle count, capacity per vehicle

---

### FR-4: Actual Outcome Classification

**Description:**
Core quality evaluation classifies the actual execution outcome and populates the `actual_outcome` field in the decision record. SPEC-003 FR-10 reserves this field with "type and semantics defined by the Core specification." SPEC-007 is that specification.

**Type: `ActualOutcomeClassification`**

| Field | Type | Present When | Description |
|---|---|---|---|
| `solver_outcome` | SolverOutcome enum | Always | The SolverOutcome value from the SolverResponse (SPEC-004 FR-4): `Succeeded`, `Timeout`, `Cancelled` |
| `solution_present` | bool | Always | True when a route plan was included in the SolverResponse |
| `time_window_constrained` | bool | Always | True when the routing problem has at least one stop with time window fields. Derived from the routing problem. |
| `time_window_feasible` | bool | When simulation was performed | True when the simulated route plan has zero time window violations (FR-5). Null when simulation was not performed (no route plan, or infrastructure failure). |
| `time_window_violation_count` | uint32 | When simulation was performed | Count of stops where simulated arrival time exceeds `time_window_close`. Null when simulation was not performed. |

**Population sequence:**
The fields in `ActualOutcomeClassification` fall into two groups by when they become available:

| Group | Fields | Availability | Source |
|---|---|---|---|
| Input-derived (always available) | `solver_outcome`, `solution_present`, `time_window_constrained` | Before simulation; derived directly from inputs | `SolverResponse.outcome`; `SolverResponse.route_plan != null`; `RoutingProblem.stops` inspection |
| Simulation-derived | `time_window_feasible`, `time_window_violation_count` | After route simulation (FR-5) completes | Simulation output |

`solver_outcome`, `solution_present`, and `time_window_constrained` are always non-null in any `QualityEvaluationResult` returned by Core, including results returned on infrastructure failure. Simulation-derived fields are null when simulation could not be performed (see FR-13).

**Semantics:**
- A route plan from a Succeeded response may be time-window infeasible. This is a solver quality defect, not a contract violation (SPEC-004 FR-7). `actual_outcome` records both that the solver Succeeded and whether the solution is time-window feasible.
- A route plan from a Timeout or Cancelled response is evaluated the same way. The `solver_outcome` distinguishes the execution circumstance from the feasibility outcome.
- `actual_outcome` is populated by Core quality evaluation and returned as part of the `QualityEvaluationResult`. The Worker writes it to the decision record via the evidence persistence path (SPEC-005 FR-16).

**Acceptance Criteria:**
- `actual_outcome` is a structured `ActualOutcomeClassification`, not a bare SolverOutcome enum value
- `solver_outcome`, `solution_present`, and `time_window_constrained` are never null in any returned `QualityEvaluationResult`, including partial results on infrastructure failure
- `time_window_feasible` is null when simulation was not performed
- `time_window_feasible = true` requires `time_window_violation_count = 0`
- `time_window_feasible = false` requires `time_window_violation_count > 0`
- If the routing problem carries no time-windowed stops (`time_window_constrained = false`), `time_window_feasible = true` and `time_window_violation_count = 0` for any complete route plan

---

### FR-5: Route Simulation Contract

**Description:**
Core quality evaluation simulates vehicle route execution to compute arrival times, check time window constraints, and produce distance-based quality metrics. Route simulation is deterministic given the same inputs.

**Simulation algorithm:**

For each vehicle route in the RoutePlan (indexed 0 through `vehicle_count − 1`):

1. Initialize:
   - `current_time_s = 0.0` (route start; t=0 per SPEC-001 FR-9)
   - `current_location = depot` (SPEC-001 FR-3)
   - `route_distance_km = 0.0`

2. For each stop in the route's ordered stop sequence:
   a. Compute `edge_distance_km = Haversine(current_location, stop.location)` per SPEC-001 FR-5
   b. Compute `travel_time_s = (edge_distance_km / travel_speed_kmh) × 3600`
   c. Compute `arrival_time_s = current_time_s + travel_time_s`
   d. If stop has time window fields (`time_window_open`, `time_window_close`):
      - Record `arrival_time_s` for this stop
      - If `arrival_time_s > time_window_close`: record as a time window violation for this stop
      - `departure_time_s = max(arrival_time_s, time_window_open) + service_duration_s`
   e. If stop has no time window fields:
      - `departure_time_s = arrival_time_s + service_duration_s`
   f. `route_distance_km += edge_distance_km`
   g. `current_time_s = departure_time_s`
   h. `current_location = stop.location`

3. Return leg (depot):
   - `return_edge_km = Haversine(current_location, depot)`
   - `route_distance_km += return_edge_km`
   - (Return leg travel time is not tracked; return is implicit per SPEC-004 FR-5)

4. Accumulate:
   - `total_route_distance_km += route_distance_km`
   - Add any violation records for this vehicle's route

**Service duration semantics:** Per SPEC-001 FR-16, `service_duration_s` defaults to 0 when absent from the stop definition. The simulation applies this default without error.

**Haversine:** Per SPEC-001 FR-5, Core computes Haversine great-circle distances. The simulation uses the same formula and the same coordinate representation (IEEE 754 double precision) as the routing problem.

**Empty routes:** A vehicle route with no stops contributes 0 km and no violations. It is not an error.

**Acceptance Criteria:**
- Route simulation starts at depot at time 0 for every vehicle
- Service duration is applied at every stop regardless of whether the stop has time window fields
- Early arrival (arrival before `time_window_open`) causes waiting; departure is at `max(arrival_time_s, time_window_open) + service_duration_s`
- Late arrival (arrival after `time_window_close`) is recorded as a violation; the vehicle does not wait; `departure_time_s = arrival_time_s + service_duration_s`
- Return leg distance is included in `total_route_distance_km` for each vehicle
- Simulation is performed for all vehicle routes including empty routes; empty routes contribute 0 km

---

### FR-6: Quality Metrics Model

**Description:**
Core quality evaluation computes the following quality metrics from the route simulation (FR-5). Metrics are present when `solution_present = true` and route simulation completed successfully. All metrics are null when no route plan was provided, or when route simulation could not be performed due to infrastructure failure (FR-13).

**Type: `QualityMetrics`**

| Field | Type | Formula / Source | Description |
|---|---|---|---|
| `total_route_distance_km` | float64 | Sum of all vehicle `route_distance_km` from FR-5 simulation | Total kilometers traveled by all vehicles, including return legs to depot. Lower is better for the same problem. |
| `max_route_distance_km` | float64 | Max of per-vehicle `route_distance_km` | Longest single vehicle route. Indicates load imbalance when far from `total_route_distance_km / routes_used`. |
| `routes_used` | uint32 | Count of vehicles with at least one stop in the RoutePlan | Number of vehicles actively deployed. |
| `routes_total` | uint32 | `vehicle_count` from the routing problem | Total available vehicles. Always equals `vehicle_count`. |
| `time_window_violation_count` | uint32 | Count of stops where `arrival_time_s > time_window_close` | Zero for a time-window-feasible plan. Mirrors the count in `ActualOutcomeClassification`. |
| `time_window_feasible` | bool | `time_window_violation_count == 0` | True when no time window violations were detected. Mirrors `ActualOutcomeClassification.time_window_feasible`. |
| `violated_stop_ids` | list\<int\> | Stop IDs where `arrival_time_s > time_window_close` | Diagnostics: which stops have time window violations. Empty when `time_window_feasible = true`. Must not appear in structured log events. See Security Considerations. |
| `per_vehicle_distance_km` | list\<float64\> | Indexed by vehicle index (0 through `vehicle_count − 1`) | Per-vehicle route distance. Enables reporting and load-balance analysis. |
| `quality_comparison_eligible` | bool | `(NOT time_window_constrained) OR (time_window_feasible = true)` | True when `hindsight_quality` is valid for cross-backend comparison. False when time windows exist and the plan violates at least one. Conservative false when `time_window_constrained = true` and simulation could not be performed (`time_window_feasible` null). See FR-7. |

**Note on duplication:** `time_window_violation_count` and `time_window_feasible` are duplicated between `QualityMetrics` and `ActualOutcomeClassification` for different consumers: `ActualOutcomeClassification` is written to the decision record (SPEC-003 FR-10) for scheduler effectiveness analysis; `QualityMetrics` is written to the quality evaluation record (SPEC-006 FR-7) for detailed evidence. Both values must be identical — they are computed from the same simulation run. See Testability TC-24.

**Note on `quality_comparison_eligible` when `quality_metrics` is null:** When `quality_metrics` is null (simulation infrastructure failure, FR-13), `quality_comparison_eligible` is contained within a null struct and is therefore null. Consumers MUST treat a null `quality_metrics` as comparison-ineligible, equivalent to `quality_comparison_eligible = false`. Null `quality_metrics` means comparison eligibility cannot be confirmed; no cross-backend comparison using `hindsight_quality` should be performed.

**Acceptance Criteria:**
- `total_route_distance_km` is the sum of all per-vehicle distances including return legs
- `routes_used` counts only vehicles with a non-empty route sequence
- `time_window_violation_count` equals `len(violated_stop_ids)`
- `per_vehicle_distance_km` has exactly `vehicle_count` entries
- `per_vehicle_distance_km[i]` is the route distance for vehicle `i` (0.0 for an empty route)
- All metric values are non-negative
- `quality_comparison_eligible = true` when `time_window_constrained = false`
- `quality_comparison_eligible = true` when `time_window_constrained = true` and `time_window_feasible = true`
- `quality_comparison_eligible = false` when `time_window_constrained = true` and `time_window_feasible = false`
- `quality_comparison_eligible = false` when `time_window_constrained = true` and `time_window_feasible` is null

---

### FR-7: Hindsight Quality

**Description:**
`hindsight_quality` is the primary normalized quality value written to the `hindsight_quality` field reserved in the decision record (SPEC-003 FR-10, FR-12). It enables cross-backend comparison on the same routing problem and supports the project thesis by making "which solver produced the best solution" a measurable, reproducible claim.

**Definition:**

```
hindsight_quality = total_route_distance_km
```

`total_route_distance_km` is the sum of all Haversine edge distances across all vehicle routes, including return legs to the depot (FR-6).

**Semantics:**
- `hindsight_quality` is a minimization metric: lower is better.
- It is defined in kilometers (float64, non-negative).
- It is present (non-null) when `solution_present = true` and route simulation completed successfully.
- It is null when `solution_present = false` (no route plan in the SolverResponse).
- It is null when route simulation could not be performed due to infrastructure failure (FR-13), even when `solution_present = true`. In this case the partial `QualityEvaluationResult` contains a non-null `actual_outcome` but a null `hindsight_quality`.
- `hindsight_quality` is dimensionally stable for comparison only within the same routing problem. Cross-problem comparison of `hindsight_quality` values is not meaningful because stop counts, depot positions, and geographic distribution vary.

**Comparison validity constraint:**
`hindsight_quality` MUST NOT be used in cross-backend comparisons unless `quality_comparison_eligible = true` (FR-6) for all plans being compared. An infeasible plan may achieve a shorter route by ignoring time window constraints, which represents a different optimality surface, not superior performance. Comparing `hindsight_quality` between a time-feasible and a time-infeasible plan produces a misleading signal that could corrupt Scheduler effectiveness analysis over time.

**Why total route distance:**
Total route distance is the canonical VRP primary objective. It is computable from a RoutePlan and a routing problem without additional inputs, is deterministic, is universally applicable across all problem configurations (with or without time windows), and produces a value directly interpretable in physical units. This makes it suitable for the normalized quality signal the Scheduler uses in effectiveness analysis.

**Relationship to `predicted_quality`:**
The Scheduler's `predicted_quality` field in the decision record (SPEC-003 FR-10) is a categorical tier (`Baseline`, `Competitive`, `Near-Optimal`). `hindsight_quality` is a numeric distance. These are different types and cannot be directly subtracted. The regret analysis (FR-8) handles this heterogeneity by capturing both values without computing a spurious numeric delta.

**Acceptance Criteria:**
- `hindsight_quality = total_route_distance_km` computed per FR-6
- `hindsight_quality` is null when no route plan was provided or when route simulation infrastructure failed (FR-13)
- `hindsight_quality` is a non-negative float64
- `hindsight_quality` MUST NOT be used in cross-backend comparisons when `quality_comparison_eligible = false`

---

### FR-8: Regret Analysis

**Description:**
Core quality evaluation produces a regret analysis comparing the Scheduler's predictions (from the decision record) against the actual execution outcome. Regret analysis is the basis for scheduler effectiveness analysis over time.

**Type: `RegretAnalysis`**

| Field | Type | Present When | Formula | Description |
|---|---|---|---|---|
| `predicted_latency_ms` | uint64 | Always | `scheduler_decision.predicted_latency × 1000` | Scheduler's predicted execution latency converted to milliseconds. Always non-null: the Scheduler decision record has `predicted_latency` populated before a job enters execution (SPEC-003 FR-10 precondition guaranteed by FR-2 invocation contract). |
| `actual_execution_duration_ms` | uint64 | When `solver_response.statistics` is non-null | `solver_response.statistics.execution_duration_ms` (SPEC-004 FR-6) | Actual wall-clock execution time reported by the solver. Non-null for all `Succeeded` responses; may be null for `Timeout` and `Cancelled` responses. |
| `latency_delta_ms` | int64 | When `actual_execution_duration_ms` is non-null | `actual_execution_duration_ms - predicted_latency_ms` | Signed latency delta: negative means faster than predicted; positive means slower. Zero is an exact prediction. |
| `predicted_quality_tier` | QualityProfile enum | Always | `scheduler_decision.predicted_quality` | The Scheduler's predicted quality tier (`Baseline`, `Competitive`, `Near-Optimal`). |
| `actual_hindsight_quality_km` | float64 | When `solution_present = true` and route simulation completed successfully (FR-7) | `hindsight_quality` (FR-7) | The actual normalized quality metric. Enables qualitative comparison against `predicted_quality_tier`. |
| `confidence_score` | float64 | Always | `scheduler_decision.confidence_score` | The score margin at decision time (SPEC-003 FR-10). Preserved for effectiveness analysis. |

**Note on cost regret:** `predicted_cost` is preserved in the Scheduler's decision record but is not included in `RegretAnalysis`. Cost regret analysis requires `actual_cost` reporting from the solver contract (SPEC-004 OQ-2), which is deferred to post-MVP. `RegretAnalysis` will be extended with a `cost_delta` field when SPEC-004 OQ-2 is resolved.

**Note on prediction success assessment:** Whether the execution "matched" the Scheduler's prediction is a reporting-layer concern, not an evidence record concern. The evidence record stores the raw facts (`solver_outcome`, `solution_present`, `time_window_feasible`, `predicted_quality_tier`, `actual_hindsight_quality_km`) from which any reporting consumer can derive prediction success by its own definition. Embedding a specific boolean interpretation in the evidence record would foreclose alternative definitions and violate the evidence-vs-interpretation boundary that underpins the project's evidence model.

**Acceptance Criteria:**
- `RegretAnalysis` is always present in the `QualityEvaluationResult`, even when no route plan was available (fields are null in that case)
- `predicted_latency_ms` is always non-null (SPEC-003 FR-10 precondition guaranteed by FR-2)
- `latency_delta_ms` is null when `actual_execution_duration_ms` is null (`predicted_latency_ms` is always non-null per the SPEC-003 FR-10 precondition)
- `latency_delta_ms = actual_execution_duration_ms - predicted_latency_ms` (signed, exact subtraction)
- `confidence_score` is preserved exactly as received from the decision record

---

### FR-9: Evaluation Result Schema

**Description:**
Core quality evaluation returns a single `QualityEvaluationResult` to the Worker for every invocation. The `QualityEvaluationResult` is the complete evaluation output.

**Type: `QualityEvaluationResult`**

| Field | Type | Present When | Description |
|---|---|---|---|
| `actual_outcome` | ActualOutcomeClassification | Always | The actual outcome classification (FR-4). Written by Worker to the decision record's `actual_outcome` field. Input-derived fields (`solver_outcome`, `solution_present`, `time_window_constrained`) are always non-null. |
| `quality_metrics` | QualityMetrics | When `solution_present = true` and simulation succeeded | The full quality metrics (FR-6). Null when no route plan or when simulation infrastructure failed. |
| `hindsight_quality` | float64 | When `solution_present = true` and simulation succeeded | The primary normalized quality metric (FR-7). Null when no route plan or when simulation infrastructure failed. |
| `regret_analysis` | RegretAnalysis | Always | Regret comparison (FR-8). Individual fields are null where inputs are absent. |
| `evaluation_metadata` | EvaluationMetadata | Always | Evaluation context and provenance (FR-12). |

The `QualityEvaluationResult` is passed from Core to the Worker. The Worker distributes its fields to the appropriate persistence targets:
- `actual_outcome` → decision record `actual_outcome` field (SPEC-003 FR-10; SPEC-006 FR-5.4)
- `hindsight_quality` → decision record `hindsight_quality` field (SPEC-003 FR-10; SPEC-006 FR-5.5)
- `quality_metrics` + `regret_analysis` + `evaluation_metadata` → quality evaluation record (SPEC-006 FR-7)

**Acceptance Criteria:**
- `QualityEvaluationResult` is always returned by Core; null is never returned
- On infrastructure failure, Core returns a partial `QualityEvaluationResult` in which `actual_outcome.solver_outcome`, `actual_outcome.solution_present`, and `actual_outcome.time_window_constrained` are always non-null (derived from inputs); simulation-derived fields (`time_window_feasible`, `time_window_violation_count`, `quality_metrics`, `hindsight_quality`) are null
- `actual_outcome` is never null; even when no route plan is present, it contains the SolverOutcome and `solution_present = false`
- The Worker does not interpret or transform `QualityEvaluationResult` fields before persisting them

---

### FR-10: Determinism Requirements

**Description:**
Core quality evaluation must be deterministic. SPEC-005 Assumption 9 and SPEC-006 Assumption 5 require this property. SPEC-007 establishes it as a hard behavioral requirement.

**Determinism invariant:**
Given identical inputs — same `routing_problem`, same `route_plan`, same `solver_response`, same `scheduler_decision`, and same `travel_speed_kmh` — Core quality evaluation must produce an identical `QualityEvaluationResult` on every invocation.

The determinism invariant applies to all computed fields in `QualityEvaluationResult`. The `evaluated_at` field in `EvaluationMetadata` (FR-12) is a wall-clock timestamp and is explicitly excluded from the determinism invariant. Two invocations at different wall-clock times may produce different `evaluated_at` values and are still considered deterministically equivalent if all computed fields are identical.

**Requirements:**
1. **No stochastic operations:** Core quality evaluation must not use any PRNG, system time, process ID, or OS random source in its computation path. The evaluation algorithm is purely deterministic arithmetic.
2. **No external state:** Core quality evaluation must not read configuration, database state, or any external resource during evaluation. All inputs are passed as parameters.
3. **Floating-point determinism:** All floating-point computations must use IEEE 754 double precision (ADR-010 Decision 6). Extended precision is prohibited in the evaluation path.
4. **Haversine consistency:** The Haversine computation in route simulation (FR-5) must use the same implementation as the routing problem model (SPEC-001 FR-5). Both computations produce semantically equivalent results across conforming C++17 toolchains (ADR-010 Decision 2).
5. **Evaluation schema version:** The evaluation result is tied to the evaluation schema version (FR-11). A given schema version produces identical outputs for identical inputs. Changing the evaluation formula increments the schema version.

**Consequence for idempotency:**
SPEC-005 FR-14 relies on Core quality evaluation being deterministic. On message redelivery, the Worker re-executes the solver and re-invokes evaluation. Because both the solver (SPEC-004 FR-11 reproducibility invariant) and Core quality evaluation are deterministic from the same inputs and seeds, the same `QualityEvaluationResult` is produced and upserted into the evidence record without creating divergent records.

**Acceptance Criteria:**
- Two invocations of Core quality evaluation with identical inputs produce byte-identical `QualityEvaluationResult` computed fields; `EvaluationMetadata.evaluated_at` is explicitly excluded from this comparison
- No `QualityEvaluationResult` computed field is derived from system time, random state, or any non-input value
- Floating-point values in `QualityEvaluationResult` are IEEE 754 double precision
- Changing any evaluation formula increments the `evaluation_schema_version` (FR-11)

---

### FR-11: Evaluation Schema Versioning

**Description:**
The quality evaluation record carries an `evaluation_schema_version` identifier that specifies which metric formulas and type definitions were used to produce the record. This enables evidence records produced at different points in the system's lifecycle to be correctly interpreted even after formula evolution.

**Current version:** `1`

**Version semantics:**
- `evaluation_schema_version` is a uint32 value present in every `EvaluationMetadata` (FR-12) and in every quality evaluation record in the Evidence Log.
- Version 1 defines the metric set, formulas, and type definitions in this specification.

**Breaking changes (MUST increment `evaluation_schema_version`):**
- Changing the formula for `hindsight_quality`
- Changing the formula for `total_route_distance_km` (e.g., changing whether return legs are included)
- Changing the route simulation algorithm in FR-5 in a way that alters computed values
- Removing a field from `QualityMetrics` or `ActualOutcomeClassification`
- Changing the type or semantics of any existing field in `QualityEvaluationResult`

**Non-breaking changes (do NOT increment `evaluation_schema_version`):**
- Adding optional fields to `QualityMetrics` or `EvaluationMetadata`
- Adding new fields to `RegretAnalysis` that are null for schema version 1 invocations
- Clarifying field descriptions without changing semantics or computed values

**Breaking change procedure:**
1. `evaluation_schema_version` is incremented
2. SPEC-007 is revised with an explicit change record documenting the previous and replacement definition
3. SPEC-006 FR-7 schema is updated to reflect the new field set and version
4. Evidence records produced under prior versions remain in the Evidence Log with their original `evaluation_schema_version`. Cross-version comparison requires version-aware consumers.

**Acceptance Criteria:**
- Every `QualityEvaluationResult` carries `evaluation_schema_version = 1` (at current version)
- Every quality evaluation record in the Evidence Log carries the `evaluation_schema_version` at which it was produced
- Consumers of quality evaluation records must not assume schema versions are identical across records without checking the `evaluation_schema_version` field

---

### FR-12: Evaluation Metadata

**Description:**
Every `QualityEvaluationResult` carries an `EvaluationMetadata` record that captures evaluation context and provenance.

**Type: `EvaluationMetadata`**

| Field | Type | Description |
|---|---|---|
| `evaluation_schema_version` | uint32 | The schema version under which this evaluation was produced (FR-11). Current value: 1. |
| `evaluated_at` | timestamp (UTC) | Wall-clock timestamp at the time evaluation was invoked. This field is metadata only; it does not participate in the determinism invariant (FR-10) and is not used in any computation. |
| `route_simulation_performed` | bool | True when the route simulation (FR-5) was performed (i.e., a route plan was present and simulation completed). False when evaluation was invoked with no route plan, or when simulation could not be performed due to infrastructure failure. |
| `travel_speed_kmh_used` | float64 | The `travel_speed_kmh` parameter value used in the route simulation (FR-5). Null when `route_simulation_performed = false`. Records the simulation parameter for reproducibility and diagnostic purposes. |
| `stop_count` | uint32 | Stop count from the routing problem. Retained for Evidence Log queries without requiring a join to the routing problem. |
| `vehicle_count` | uint32 | Vehicle count from the routing problem. Same rationale. |
| `time_window_constrained` | bool | True when the routing problem has at least one stop with time window fields. Same as `ActualOutcomeClassification.time_window_constrained`. |

**Acceptance Criteria:**
- `EvaluationMetadata` is present in every `QualityEvaluationResult`
- `evaluated_at` is a UTC timestamp
- `travel_speed_kmh_used` is null when `route_simulation_performed = false`
- `evaluation_schema_version = 1` in all results produced under this specification version

---

### FR-13: Failure Handling

**Description:**
Core quality evaluation can fail due to infrastructure faults or invalid invocation contracts. SPEC-007 distinguishes these from solver execution outcomes.

**Failure categories:**

| Failure Condition | Classification | Worker Response |
|---|---|---|
| Missing required input (null `routing_problem`, null `scheduler_decision`) | Invocation contract violation — Worker construction error | Worker logs a structured error; does not invoke Core; records quality evaluation as failed in the evidence record; job still reaches `Completed` |
| Core process unavailable or unclassified exception during evaluation | Infrastructure failure | Core returns a partial `QualityEvaluationResult` (see below); Worker writes partial result to evidence record; Worker logs `worker.quality.evaluation.failed` per SPEC-005 FR-15; job still reaches `Completed` |
| `travel_speed_kmh = 0` or negative | Invocation contract violation | Worker logs a structured error; evaluation fails; quality fields are null; job still reaches `Completed` |

**Partial `QualityEvaluationResult` on infrastructure failure:**
When Core QE experiences an infrastructure failure after receiving valid inputs, Core returns a partial `QualityEvaluationResult` rather than null. The partial result contains all input-derived fields and leaves simulation-derived fields null:

| Field | Value on Infrastructure Failure |
|---|---|
| `actual_outcome.solver_outcome` | Non-null — populated from `solver_response.outcome` (input-derived) |
| `actual_outcome.solution_present` | Non-null — populated from SolverResponse (input-derived) |
| `actual_outcome.time_window_constrained` | Non-null — populated from `routing_problem` (input-derived) |
| `actual_outcome.time_window_feasible` | Null — simulation not completed |
| `actual_outcome.time_window_violation_count` | Null — simulation not completed |
| `quality_metrics` | Null — simulation not completed |
| `hindsight_quality` | Null — simulation not completed |
| `regret_analysis` | Partially populated: `predicted_latency_ms`, `predicted_quality_tier`, `confidence_score` present; `actual_execution_duration_ms`, `latency_delta_ms`, `actual_hindsight_quality_km` null |
| `evaluation_metadata.route_simulation_performed` | False |

This ensures the Worker always has `solver_outcome` and `solution_present` to write to the decision record, eliminating the evidence gap that would occur if `actual_outcome` were entirely null on infrastructure failure.

**On evaluation failure:**
Per SPEC-005 FR-15 failure handling: if Core quality evaluation infrastructure fails, the Worker writes the partial `QualityEvaluationResult` to the evidence record. The decision record receives `actual_outcome` with `solver_outcome` and `solution_present` populated; simulation-derived quality fields are null. The job proceeds to the reporting stage and reaches `Completed`. The failure is captured in the `result.evaluate` span status as Error and in a structured log event.

A quality evaluation infrastructure failure does not cause the job to transition to `Failed`. Quality evaluation failure is not a pre-evidence failure.

**Core must not fail for these inputs:**
- Time-window-infeasible route plans (these produce quality metrics with violations, not failures)
- Routes that violate capacity constraints (structural validation already guaranteed capacity validity per SPEC-004 FR-5; Core does not re-validate)
- Problems with no stops with time windows (`time_window_constrained = false`): simulation runs normally with zero violations

**Acceptance Criteria:**
- Core quality evaluation does not panic or produce undefined behavior on any input satisfying FR-3's type constraints
- An infeasible route plan (time window violations) produces a `QualityEvaluationResult` with `time_window_feasible = false`, not a failure
- A missing required input causes a structured invocation contract violation, not a silent evaluation failure
- On infrastructure failure, `actual_outcome.solver_outcome`, `actual_outcome.solution_present`, and `actual_outcome.time_window_constrained` are always non-null in the returned partial result

---

# Non-Requirements

- Core Quality Evaluation does not execute optimization or invoke solver backends
- Core Quality Evaluation does not select or reject solver backends
- Core Quality Evaluation does not persist evaluation results — the Evidence Log (SPEC-006) and Worker (SPEC-005) own persistence
- Core Quality Evaluation does not generate evidence reports — the Report Generator component owns report content
- Core Quality Evaluation does not modify the routing problem, the route plan, or the decision record
- Core Quality Evaluation does not define the scheduler configuration or scoring weights (SPEC-003)
- Core Quality Evaluation does not define what constitutes a "good" scheduling decision — it produces the evidence; effectiveness analysis is a consumer responsibility
- Core Quality Evaluation does not compute workload features — those are computed before Scheduler invocation per SPEC-003 FR-3
- Core Quality Evaluation does not verify RoutePlan structural validity — that is Worker responsibility per SPEC-005 FR-11
- Core Quality Evaluation does not produce multi-dimensional capacity utilization metrics. Capacity was validated structurally before evaluation (SPEC-004 FR-5 conditions 1–5). Capacity utilization per route is not re-derived here.
- Core Quality Evaluation does not produce road-network-accurate travel times. Per SPEC-001 FR-5, distances are Haversine great-circle approximations. Evaluation uses the same approximation.
- Core Quality Evaluation does not produce real-time quality assessments during solver execution. Evaluation occurs post-execution, using the final solver output.

---

# Assumptions

1. The routing problem passed to Core quality evaluation has been validated by Core per ADR-009. Evaluation does not re-validate domain constraints.
2. The RoutePlan passed to Core quality evaluation has passed structural validation per SPEC-005 FR-11 and SPEC-004 FR-5. All stops appear exactly once and all capacities are satisfied.
3. The Haversine formula used in route simulation (FR-5) is the same implementation as SPEC-001 FR-5. No alternative distance formula is used.
4. `travel_speed_kmh` is a routing problem attribute (`RoutingProblem.average_vehicle_speed_kmh`), formally defined in SPEC-001 FR-17 per Project Owner Decision ODR-1. The field is required for all routing problem submissions. SPEC-007 references this field as an accepted dependency.
5. The synthetic routing problems generated by SPEC-002 are designed to be feasible under the configured `travel_speed_kmh`. That is, synthetic time windows are achievable given the stop geography and the speed setting. This is a SPEC-002 generation contract, not a SPEC-007 evaluation assumption.
6. Core is an in-process C++ library within the Worker process (SPEC-005 Assumption 3). Quality evaluation has no network latency.
7. At-most-one evaluation is invoked per job per execution attempt. The Worker does not invoke quality evaluation more than once for the same SolverResponse.

---

# Constraints

1. Core quality evaluation must be deterministic per FR-10. No stochastic operations, external entropy, or system state may affect the evaluation output.
2. All floating-point computation in the evaluation path must use IEEE 754 double precision. Extended precision is prohibited (ADR-010 Decision 6).
3. The Haversine implementation must match SPEC-001 FR-5 for consistency with the routing problem model.
4. `violated_stop_ids` must not appear in structured log events. See Security Considerations for rationale. It is for evidence record storage only.
5. Core quality evaluation must be implemented in C++ (ADR-001).
6. Adding a new quality metric must not change `evaluation_schema_version` if the metric is optional and additive. Changing a metric formula requires a version increment (FR-11).
7. The `evaluated_at` timestamp must not participate in any computed field. It is metadata only and is excluded from the determinism invariant (FR-10).

---

# Inputs

| Input | Source | Format |
|---|---|---|
| Routing problem | Worker (loaded from PostgreSQL; C++ domain representation) | SPEC-001 C++ representation (FR-14) |
| Route plan | Worker (extracted from SolverResponse) | SPEC-004 FR-5 RoutePlan |
| Solver response | Worker (from solver backend) | SPEC-004 FR-3 SolverResponse |
| Scheduler decision record | Worker (loaded from PostgreSQL after decision persistence) | SPEC-003 FR-10 DecisionRecord |
| Travel speed parameter | Worker (from routing problem: `RoutingProblem.average_vehicle_speed_kmh` per SPEC-001 FR-17; ODR-1) | float64, km/h, positive non-zero |

---

# Outputs

| Output | Consumer | Format | Notes |
|---|---|---|---|
| `QualityEvaluationResult.actual_outcome` | Worker → decision record `actual_outcome` field | `ActualOutcomeClassification` (FR-4) | Populates the SPEC-003 FR-10 reserved field |
| `QualityEvaluationResult.hindsight_quality` | Worker → decision record `hindsight_quality` field | float64 or null | Populates the SPEC-003 FR-10 reserved field |
| `QualityEvaluationResult.quality_metrics` | Worker → quality evaluation record (SPEC-006 FR-7) | `QualityMetrics` (FR-6) or null | Written to the Evidence Log |
| `QualityEvaluationResult.regret_analysis` | Worker → quality evaluation record (SPEC-006 FR-7) | `RegretAnalysis` (FR-8) | Written to the Evidence Log |
| `QualityEvaluationResult.evaluation_metadata` | Worker → quality evaluation record (SPEC-006 FR-7) | `EvaluationMetadata` (FR-12) | Written to the Evidence Log |

---

# Failure Modes

### Missing or Null Input

**Condition:** The Worker invokes Core quality evaluation with a null `routing_problem`, null `solver_response`, or null `scheduler_decision`.

**Behavior:** Core returns a structured invocation contract violation (not a QualityEvaluationResult). This is a Worker construction error. Core does not evaluate.

**Worker response:** Worker logs a structured error. Quality fields are null in the persisted evidence record. The `result.evaluate` span records Error status. Job still reaches `Completed`.

---

### Invalid `travel_speed_kmh`

**Condition:** `travel_speed_kmh` is 0, negative, or NaN.

**Behavior:** Core returns a structured invocation contract violation. Route simulation is not performed.

**Worker response:** Same as missing input.

---

### Core Process Unavailable

**Condition:** Core throws an unclassified exception or the in-process library is in an invalid state.

**Behavior:** Core returns a partial `QualityEvaluationResult` per FR-13. Input-derived fields (`actual_outcome.solver_outcome`, `actual_outcome.solution_present`, `actual_outcome.time_window_constrained`) are always populated. Simulation-derived fields are null.

**Worker response:** Worker writes the partial `QualityEvaluationResult` to the evidence record. `actual_outcome.solver_outcome` and `actual_outcome.solution_present` are persisted to the decision record. Simulation-derived quality fields are null in the evidence record. `result.evaluate` span records Error. Job still reaches `Completed`. This is a transient failure; it does not cause a NACK or job re-queuing (quality evaluation failure on completion is not retried at MVP scope per SPEC-005 FR-15).

---

### Time-Window-Infeasible Route Plan

**Condition:** The simulated routes reveal arrival times that violate one or more `time_window_close` constraints.

**Behavior:** Core quality evaluation completes successfully. `time_window_feasible = false`. `time_window_violation_count > 0`. `violated_stop_ids` lists the offending stops. `hindsight_quality` is computed normally. `quality_comparison_eligible = false`.

**This is not a failure mode.** It is the correct evaluation output for an infeasible plan.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer | Yes — defines Core quality evaluation algorithm, type definitions for `actual_outcome` and `hindsight_quality`, and route simulation contract |
| API Layer | Indirect — `actual_outcome` and `hindsight_quality` values appear in evidence records served by the API |
| Persistence | Yes — defines the full quality evaluation record schema (SPEC-006 FR-7 extension); defines `actual_outcome` as `ActualOutcomeClassification` requiring schema update |
| Solver Runtime | Yes — defines what Core does with SolverResponse after execution; no change to solver contract |
| Observability | Indirect — `result.evaluate` span attributes (owned by Worker/SPEC-005) may be extended with evaluation summary data |
| Configuration | Yes — `travel_speed_kmh` is a routing problem attribute defined in SPEC-001 FR-17 (ODR-1) |
| Security | Minimal — `violated_stop_ids` must not appear in logs; evaluation operates on pre-validated data |

**SPEC-003 FR-10:** `actual_outcome` type is now defined as `ActualOutcomeClassification` (FR-4). `hindsight_quality` type is now defined as float64 (FR-7). SPEC-003 FR-10's reserved fields are no longer untyped. The explicit named dependencies on `predicted_latency`, `predicted_quality`, and `confidence_score` fields from SPEC-003 FR-10 are consumed by FR-8 `RegretAnalysis`.

**SPEC-006 FR-7.2:** The quality evaluation record schema deferred to this specification is now defined. SPEC-006 FR-7 must be extended with the full field set from FR-6, FR-7, FR-8, FR-10, FR-11, and FR-12. (See Documentation Updates Required.)

**SPEC-006 FR-5.4:** SPEC-006 FR-5.4 states "The `actual_outcome` field in the decision record is updated by the Worker after the solver run completes, using the `SolverOutcome` value from the SolverResponse." SPEC-007 defines `actual_outcome` as the structured `ActualOutcomeClassification` type produced by Core quality evaluation, not a bare `SolverOutcome` enum. SPEC-006 FR-5.4 requires revision to reflect that: (a) `actual_outcome` is of type `ActualOutcomeClassification`; (b) it is produced by Core quality evaluation; and (c) the Worker writes it after receiving the `QualityEvaluationResult`, not directly from the SolverResponse. Even on infrastructure failure, `actual_outcome.solver_outcome` and `actual_outcome.solution_present` are always written — they are input-derived and present in the partial result (FR-13). (See Documentation Updates Required.)

---

# Testability

Every quality evaluation behavior must be proven before this feature is considered complete. These are test contracts, not test implementations.

1. **Unit: Determinism invariant** — Two invocations of Core quality evaluation with identical `routing_problem`, `route_plan`, `solver_response`, `scheduler_decision`, and `travel_speed_kmh` produce byte-identical `QualityEvaluationResult` computed fields. `EvaluationMetadata.evaluated_at` is explicitly excluded from comparison per FR-10.

2. **Unit: Time window feasibility — zero violations** — A route plan where all simulated arrival times are at or before `time_window_close` for every time-windowed stop produces `time_window_feasible = true`, `time_window_violation_count = 0`, `violated_stop_ids = []`.

3. **Unit: Time window feasibility — with violations** — A route plan where at least one simulated arrival time exceeds `time_window_close` produces `time_window_feasible = false`, `time_window_violation_count > 0`, `violated_stop_ids` containing the violating stop IDs.

4. **Unit: Time window feasibility — early arrival** — A route plan where a vehicle arrives before `time_window_open` at a time-windowed stop is not a violation. The vehicle waits. Departure is at `max(arrival_time_s, time_window_open) + service_duration_s`. No violation is recorded.

5. **Unit: No time-windowed stops** — A routing problem where no stop has time window fields produces `time_window_feasible = true`, `time_window_violation_count = 0`, `time_window_constrained = false` regardless of route order.

6. **Unit: Route distance computation** — A known routing problem with a known route plan (all coordinates given) produces `total_route_distance_km` equal to the manually computed sum of Haversine edges including return legs, within acceptable floating-point tolerance.

7. **Unit: Return leg included** — `per_vehicle_distance_km[i]` includes the distance from the last assigned stop back to the depot for each vehicle. A route with one stop: `per_vehicle_distance_km[i] = Haversine(depot, stop) + Haversine(stop, depot)`.

8. **Unit: Empty vehicle route** — A vehicle route with no stops contributes 0.0 to `total_route_distance_km` and 0.0 to `per_vehicle_distance_km[i]`. It does not count toward `routes_used`.

9. **Unit: `hindsight_quality = total_route_distance_km`** — For any route plan with `solution_present = true`, `hindsight_quality` equals `total_route_distance_km` exactly.

10. **Unit: `hindsight_quality` null when no solution** — When the Worker invokes evaluation for a SolverOutcome with no route plan (Timeout with no solution, etc.), evaluation is not invoked per FR-2. `hindsight_quality = null` in the evidence record. (This is a Worker behavior test, not a Core evaluation test, but validates the FR-2 invocation contract.)

11. **Unit: `actual_outcome.solver_outcome` matches SolverResponse** — `actual_outcome.solver_outcome` equals the SolverOutcome value from the SolverResponse passed to evaluation.

12. **Unit: Regret latency delta** — For a SolverResponse with `execution_duration_ms = 5000` and a decision record with `predicted_latency = 4.0` (seconds), `latency_delta_ms = 5000 − 4000 = 1000`. Sign is correct: positive means slower than predicted.

13. **Unit: Invocation contract violation — null routing problem** — Core quality evaluation called with a null `routing_problem` returns a structured invocation contract violation, not a `QualityEvaluationResult`.

14. **Unit: Invocation contract violation — invalid travel speed** — Core quality evaluation called with `travel_speed_kmh = 0` returns a structured invocation contract violation.

15. **Unit: Time-window-infeasible plan produces quality metrics** — A route plan with time window violations still produces `total_route_distance_km`, `hindsight_quality`, and the full `QualityMetrics`. `quality_comparison_eligible = false`. Evaluation does not fail on an infeasible plan.

16. **Property: Service duration applied** — For a stop with `service_duration = 300` (5 minutes) and no time window: departure time from that stop is `arrival_time_s + 300`. For a stop with `service_duration = 300` and `time_window_open = 600`: departure time is `max(arrival_time_s, 600) + 300`.

17. **Integration: Full quality evaluation record** — A completed job with a Succeeded solver outcome produces a quality evaluation record in the Evidence Log with all required fields: `job_id`, `evaluation_status`, `evaluated_at`, `evaluation_schema_version`, `actual_outcome`, `hindsight_quality`, `quality_metrics`, `regret_analysis`, `evaluation_metadata`.

18. **Integration: Scheduler effectiveness analysis** — For two jobs with the same `problem_id` executed with different backends (nearest-neighbor vs. QUBO SA), both producing Succeeded responses with time-feasible plans: the `hindsight_quality` values are retrievable by `problem_id` query, `quality_comparison_eligible = true` for both, and the decision records for both contain `actual_outcome` and `hindsight_quality`, enabling valid backend quality comparison for that problem.

19. **Unit: Infrastructure failure partial result** — When Core QE infrastructure fails after receiving a valid SolverResponse with `solver_outcome = Succeeded` and a route plan, the returned partial `QualityEvaluationResult` MUST have non-null `actual_outcome.solver_outcome`, `actual_outcome.solution_present`, and `actual_outcome.time_window_constrained`. Fields `actual_outcome.time_window_feasible`, `actual_outcome.time_window_violation_count`, `quality_metrics`, and `hindsight_quality` MUST be null.

20. **Unit: `quality_comparison_eligible` — no time windows** — A routing problem with no time-windowed stops produces `quality_comparison_eligible = true` regardless of route order or distances.

21. **Unit: `quality_comparison_eligible` — time-feasible plan** — A routing problem with time-windowed stops and all simulated arrival times within `time_window_close` produces `quality_comparison_eligible = true`.

22. **Unit: `quality_comparison_eligible` — time-infeasible plan** — A routing problem with time-windowed stops and at least one simulated arrival time exceeding `time_window_close` produces `quality_comparison_eligible = false`.

23. **Unit: `quality_comparison_eligible` — simulation failure** — When `time_window_constrained = true` and route simulation cannot be performed (infrastructure failure), `quality_metrics` is null (FR-9, FR-13), and `quality_comparison_eligible` is therefore null as a subfield of a null struct. Consumers MUST treat this as comparison-ineligible per the normative statement in FR-6. Absence of feasibility confirmation is not confirmation of eligibility.

24. **Unit: Duplication identity invariant** — For any `QualityEvaluationResult` where both `QualityMetrics.time_window_violation_count` and `ActualOutcomeClassification.time_window_violation_count` are non-null, their values MUST be identical. The same applies to `QualityMetrics.time_window_feasible` and `ActualOutcomeClassification.time_window_feasible`.

---

# Observability Requirements

## Operational Questions

The quality evaluation result and evidence records must answer the following questions:

1. For a given job, was time window feasibility satisfied in the solver's route plan?
2. What was the actual total route distance produced by each backend for a given routing problem?
3. How did actual execution latency compare to the Scheduler's prediction?
4. Which backends consistently produce time-window-feasible solutions?
5. How does `hindsight_quality` (actual route distance) compare across backends for the same problem?
6. How often does the Scheduler select a backend that produces a fully feasible, high-quality solution?
7. What is the distribution of `latency_delta_ms` across backends and problem size classes?

## Structured Log Events

Core quality evaluation does not emit log events directly. The Worker emits the following events that carry quality evaluation outcomes:

- `worker.quality.evaluation.skipped` (SPEC-005 FR-15): When evaluation is not invoked (no route plan). Required field: `reason = "no_route_plan"`.
- `worker.quality.evaluation.failed` (SPEC-005 FR-15): When evaluation fails due to infrastructure fault. Must include `evaluation_failure_reason`.
- `worker.evidence.persisted` (SPEC-005 FR-16): When evidence is persisted. Must include whether quality evaluation record was among `artifacts_written`.

`violated_stop_ids` must not appear in any structured log event.

## Metrics

The following quantities should be observable through Prometheus metrics. These are additions to the metric set defined in SPEC-005:

| Metric name | Type | Labels | Description |
|---|---|---|---|
| `daedalus_quality_evaluation_invoked_total` | Counter | `solver_outcome` (`Succeeded`, `Timeout`, `Cancelled`) | Total quality evaluation invocations by solver outcome |
| `daedalus_quality_evaluation_failed_total` | Counter | `failure_reason` (`invocation_contract_violation`, `core_unavailable`) | Total quality evaluation failures |
| `daedalus_hindsight_quality_km` | Histogram | `backend_id`, `problem_size_class` | Distribution of `hindsight_quality` (total route distance) by backend and problem size |
| `daedalus_time_window_feasible_total` | Counter | `feasible` (`true`, `false`), `backend_id` | Count of evaluations by time window feasibility outcome |
| `daedalus_latency_delta_ms` | Histogram | `backend_id`, `problem_size_class` | Distribution of `latency_delta_ms` (actual minus predicted latency) |

## Trace Spans

The `result.evaluate` span (owned by Worker per SPEC-005 FR-19) covers Core quality evaluation. SPEC-007 recommends the following additional span attributes beyond those defined in SPEC-005 FR-19:

| Attribute | Description |
|---|---|
| `time_window_feasible` | Boolean outcome; absent when evaluation not invoked |
| `time_window_violation_count` | Count of violations; 0 when feasible or not constrained; absent when evaluation not invoked |
| `hindsight_quality_km` | `hindsight_quality` value; absent when no route plan |
| `evaluation_schema_version` | Schema version of the evaluation result |

The final attribute set on `result.evaluate` is a Worker implementation concern. SPEC-005 FR-19 owns the span definition; these are additive recommendations.

---

# Security Considerations

**`violated_stop_ids`:** This field contains stop ID integers, not geographic coordinates. It is safe to store in the Evidence Log. It must not appear in structured log events. SPEC-001 Security Considerations restrict geographic coordinates and raw stop arrays from appearing in logs; integer stop IDs are not geographic coordinates and are not covered by that restriction. The prohibition on logging `violated_stop_ids` is a forward-looking protective measure for non-synthetic deployments: in production, stop IDs may map to customer premises, and logged stop IDs in violation traces could constitute location-identifying exposure depending on jurisdiction. The violation count and the `time_window_feasible` flag in `ActualOutcomeClassification` are sufficient for operational monitoring without exposing location-identifying identifiers.

**Input trust:** The routing problem and route plan passed to Core quality evaluation have been validated by Core (ADR-009) and by the Worker (SPEC-005 FR-11) respectively. Core quality evaluation may trust structural validity and does not re-validate.

**No external data access:** Core quality evaluation operates exclusively on the inputs passed to it. It makes no network calls, reads no configuration files during evaluation, and accesses no database state. This eliminates injection and data exfiltration risks in the evaluation path.

**`travel_speed_kmh` validation:** A zero or negative speed is an invocation contract violation (FR-13). A very large speed (approaching infinity) would produce near-zero travel times and degenerate feasibility results. The valid range for `travel_speed_kmh` should be enforced at configuration load time, not within the evaluation path.

---

# Performance Considerations

**Route simulation cost:** Route simulation is O(N) in the number of stops across all vehicles. For N stops and V vehicles, the simulation processes N stop transitions and V + 1 Haversine computations per return leg (total: N + V Haversine calls, each O(1)). At MVP Large-class scale (76+ stops), this is not a performance concern. For extreme N, this should be measured.

**Haversine computation:** The route simulation recomputes distances that solvers also computed internally. This is intentional: Core operates on the normalized route plan output without access to solver-internal distance matrices. For performance-critical scenarios, a shared precomputed distance matrix is a future optimization not in scope for MVP.

**Evaluation latency budget:** Quality evaluation is one step in the Worker lifecycle between solver execution and evidence persistence. Its contribution to total job latency must be measured during implementation. No specific latency target is established at this stage.

---

# Documentation Updates Required

- **SPEC-006 FR-7 (Proposed):** Must be extended with the full quality evaluation record schema as defined in FR-6, FR-7, FR-8, FR-11, and FR-12 of this specification. The minimum fields (`job_id`, `evaluation_status`, `evaluated_at`) are supplemented by: `evaluation_schema_version`, `actual_outcome` (as `ActualOutcomeClassification`), `hindsight_quality`, `quality_metrics` (as `QualityMetrics`), `regret_analysis` (as `RegretAnalysis`), `evaluation_metadata` (as `EvaluationMetadata`). The `OQ-1 (Blocking)` note in SPEC-006 FR-7 is resolved by this specification.

- **SPEC-006 FR-5.4 (Proposed):** Must be revised to reflect that (a) `actual_outcome` is of type `ActualOutcomeClassification` defined in SPEC-007 FR-4, not a bare `SolverOutcome` enum; (b) it is produced by Core quality evaluation and returned in the `QualityEvaluationResult`; and (c) the Worker writes it to the decision record after receiving the `QualityEvaluationResult`. Even on Core QE infrastructure failure, `actual_outcome.solver_outcome` and `actual_outcome.solution_present` are always written — they are input-derived and present in the partial result (FR-13). The phrase "using the `SolverOutcome` value from the SolverResponse" should be replaced with "using the `actual_outcome` field from the Core quality evaluation result (`QualityEvaluationResult.actual_outcome`), which is always non-null in its input-derived fields."

- **SPEC-003 FR-10 (Accepted):** The types and semantics of `actual_outcome` and `hindsight_quality` are now defined: `actual_outcome` is `ActualOutcomeClassification` (SPEC-007 FR-4); `hindsight_quality` is float64 representing total route distance in km (SPEC-007 FR-7). The parenthetical "Type and semantics defined by the Core specification" is satisfied by SPEC-007. The explicit named dependencies on SPEC-003 FR-10 fields `predicted_latency`, `predicted_quality`, and `confidence_score` are consumed by SPEC-007 FR-8 `RegretAnalysis`.

- **SPEC-005 Assumption 9 (Proposed):** The assumption that "Core quality evaluation is deterministic... this property must be established by the Core Quality Evaluation Specification" is now satisfied by SPEC-007 FR-10. SPEC-005 Assumption 9 can reference SPEC-007 FR-10 as the authority.

- **SPEC-005 FR-15 (Proposed):** Must be revised to reflect the partial `QualityEvaluationResult` model defined in SPEC-007 FR-13. The current language "the Worker persists the solver run record with `actual_outcome` and `hindsight_quality` null" is inconsistent with SPEC-007 FR-13: on Core QE infrastructure failure, Core returns a partial result in which `actual_outcome.solver_outcome`, `actual_outcome.solution_present`, and `actual_outcome.time_window_constrained` are always non-null (input-derived). The Worker writes this partial `actual_outcome` to the decision record. Only simulation-derived quality fields (`hindsight_quality`, `quality_metrics`) are null on infrastructure failure. This is the intended behavior of Amendment A (M-1 resolution).

- **SPEC-001 (Accepted — revision complete):** `average_vehicle_speed_kmh` is formally defined in SPEC-001 FR-17 (per Project Owner Decision ODR-1) as a required float64 field for all routing problem submissions. SPEC-007 FR-3 and FR-5 now reference SPEC-001 FR-17 as the authoritative definition. No further SPEC-001 revision is required.

- **docs/architecture.md:** The runtime execution flow mentions "Runtime evaluates quality, cost, runtime, and regret." This language is now concretely realized by SPEC-007. No change to architecture.md is required; the existing language is satisfied. The `result.evaluate` span already appears in the required spans list.

- **SPEC-002 (Synthetic Workload Generator, Draft):** If SPEC-002 generates time-windowed routing problems, it must generate time windows that are achievable under the configured `travel_speed_kmh`. SPEC-002 should document the speed assumption used when generating synthetic time windows. This is a SPEC-002 generation contract requirement, not a quality evaluation requirement.

---

# Open Questions

### OQ-1: Travel Speed Parameter Source (Resolved — Project Owner Decision ODR-1)

**Question (resolved):** Is `travel_speed_kmh` a field on the routing problem (SPEC-001 extension), a system configuration parameter, or a scheduler configuration field?

**Owner Decision (ODR-1):** The Project Owner has accepted the architectural recommendation. `travel_speed_kmh` (`RoutingProblem.average_vehicle_speed_kmh`) belongs to the routing problem domain. It is a property of the vehicles in the fleet, which is a routing problem concern. SPEC-007 references this field as an accepted dependency.

**Required action:** Complete. SPEC-001 FR-17 formally defines `average_vehicle_speed_kmh` as a required routing problem field for all submissions.

**Status:** Fully resolved. SPEC-001 FR-17 is the authoritative definition. No longer blocking for FR-5 implementation.

---

### OQ-2: Evaluation Schema Versioning (Resolved — Project Owner Decision ODR-2)

**Question (resolved):** Should the quality evaluation record carry an `evaluation_schema_version` field that enables consumers to distinguish records produced under different metric formulas?

**Owner Decision (ODR-2):** The Project Owner has confirmed that `evaluation_schema_version` shall remain part of SPEC-007. Evaluation outputs require schema versioning for historical comparability. All versioning requirements (FR-11, FR-12) are preserved as specified.

**Status:** Resolved. FR-11 and FR-12 are retained. No further action required.

---

### OQ-3: Regret Analysis Location in Evidence (Non-Blocking, Deferred)

**Question:** Should `RegretAnalysis` be stored in the quality evaluation record, or should it be computable on demand from the decision record and solver run record?

**Architecture Review adjudication:** Deferred to implementation planning. The preliminary architectural recommendation is to store `RegretAnalysis` in the quality evaluation record. `RegretAnalysis` fields are computed from transient state (the Scheduler's prediction fields at the time of execution). Storing ensures a consistent snapshot and avoids recomputation divergence if decision record fields are later updated or if Scheduler model updates change how predictions are computed. Consumers that prefer on-demand computation may choose not to use the stored `RegretAnalysis` fields without affecting correctness.

**Blocking:** Not blocking for SPEC-007 Draft or Proposed status.

---

# Acceptance Checklist

- [x] Problem is clearly defined.
- [x] Domain concept is defined.
- [x] Responsibility scope and component boundaries are defined (FR-1).
- [x] Evaluation invocation contract is defined (FR-2).
- [x] Evaluation inputs are defined (FR-3).
- [x] Actual outcome classification model is defined (FR-4).
- [x] Route simulation contract is defined (FR-5).
- [x] Quality metrics model is defined (FR-6).
- [x] Hindsight quality is defined (FR-7).
- [x] Regret analysis is defined (FR-8).
- [x] Evaluation result schema is defined (FR-9).
- [x] Determinism requirements are defined (FR-10).
- [x] Evaluation schema versioning is defined (FR-11).
- [x] Evaluation metadata is defined (FR-12).
- [x] Failure handling is defined (FR-13).
- [x] Non-requirements are documented.
- [x] Assumptions are explicit.
- [x] Failure modes are defined.
- [x] Observability requirements exist.
- [x] Security considerations exist.
- [x] Documentation updates are identified.
- [x] OQ-1 resolved — Project Owner Decision ODR-1 accepted: `travel_speed_kmh` belongs to routing problem domain as `RoutingProblem.average_vehicle_speed_kmh`. SPEC-001 FR-17 is the authoritative definition. No longer blocking for implementation.
- [x] OQ-2 resolved — Project Owner Decision ODR-2: `evaluation_schema_version` confirmed; all versioning requirements (FR-11, FR-12) preserved.
- [ ] OQ-3 resolved — Deferred to implementation planning; preliminary recommendation to store `RegretAnalysis`. Non-blocking.

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-13) are implemented and acceptance criteria pass.
- OQ-1 (travel speed parameter source) is resolved by Project Owner Decision ODR-1: `travel_speed_kmh` is `RoutingProblem.average_vehicle_speed_kmh`. FR-3 and FR-5 reference this field. SPEC-001 revision is complete and the field is formally defined.
- OQ-2 (evaluation schema versioning) is resolved per Project Owner Decision ODR-2: `evaluation_schema_version` is included and all versioning requirements are preserved.
- All test contracts defined in the Testability section pass.
- SPEC-006 FR-7 is extended with the full quality evaluation record schema per this specification.
- SPEC-006 FR-5.4 is revised to reflect `actual_outcome` as `ActualOutcomeClassification` with partial population semantics on infrastructure failure.
- SPEC-005 Assumption 9 references SPEC-007 FR-10 as the determinism authority.
- SPEC-005 FR-15 is revised to reflect the partial `QualityEvaluationResult` model defined in SPEC-007 FR-13: input-derived `actual_outcome` fields are always non-null on infrastructure failure; only simulation-derived quality fields (`hindsight_quality`, `quality_metrics`) are null.
- The `result.evaluate` span emits quality evaluation outcomes and is verifiable in the test environment.
- Core quality evaluation passes the determinism property test (test contract 1) across two execution attempts for the same inputs.
- Engineering review passes.
- Specification status is updated to Verified.

The feature is not complete simply because the evaluation function runs without errors.
