# SPEC-003: Scheduler Objectives and Policy Engine

## Metadata

**Feature ID:** SPEC-003

**Title:** Scheduler Objectives and Policy Engine

**Status:** Accepted

**Author:** Darkhorse286

**Created:** 2026-06-08

**Last Updated:** 2026-06-08

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-007, ADR-008, ADR-009, ADR-010

**Related Specs:** SPEC-001, SPEC-002, (pending) Evidence Log Specification

---

# Problem Statement

Project DAEDALUS routes optimization problems through multiple solver backends — classical baseline solvers and a QUBO simulated annealing proxy in MVP, with additional backends planned. No component currently defines how the most appropriate backend is selected for a given routing problem, or how that selection decision is recorded and justified.

Without a defined selection policy, backend selection is arbitrary, unjustified, and unobservable. The project's core thesis — that most routing problems should never touch a quantum computer — cannot be enforced or demonstrated without an explicit, evidence-producing policy engine.

---

# Business Value

- Enforces the core project thesis: problem characteristics drive backend selection, not assumptions
- Produces auditable evidence for every backend selection decision
- Enables configurable optimization objectives (cheapest valid, fastest valid, highest quality, etc.) without special-casing backends
- Supports future backend additions without architectural changes to the selection policy
- Demonstrates production-grade scheduling logic as a portfolio artifact

---

# Employer Signaling

- System Design
- AI Engineering
- Observability
- Reliability Engineering
- Decision Science (evidence-based policy)

---

# Requirements

### FR-1: Scheduler Responsibility Scope

**Description:**
The Scheduler is responsible for:
1. Receiving a validated routing problem and its derived workload features from Core
2. Evaluating all registered solver backends for eligibility
3. Applying hard eligibility filters to eliminate candidates that cannot handle the problem
4. Scoring all remaining eligible candidates under the active optimization objective
5. Selecting the highest-scoring eligible candidate
6. Producing a structured decision record explaining the selection and every rejection
7. Returning the decision record to the caller

The Scheduler does not:
- Execute optimization
- Generate routing problems (SPEC-002)
- Validate routing problems (ADR-009)
- Compute workload features (Core responsibility per FR-3)
- Persist the decision record (Worker responsibility per FR-11)

**Invocation:** The Scheduler is a component within Daedalus Core. The Worker invokes Core after loading the routing problem from PostgreSQL. Core performs authoritative validation (ADR-009), computes workload features, invokes the Scheduler, and returns the decision record to the Worker. The Worker does not invoke the Scheduler directly.

**Persistence boundary note:** The Daedalus Scheduler Responsibilities list in `docs/architecture.md` includes "persist explainable decisions." This specification interprets that responsibility as producing decision records in a form suitable for persistence, not performing persistence directly. The runtime flow in `docs/architecture.md` ("Scheduler decision → Worker persists decision → Worker executes selected solver") is authoritative for this component boundary. See FR-11.

**Acceptance Criteria:**
- The Scheduler produces a complete decision record for every invocation
- The Scheduler does not invoke any solver backend
- The Scheduler does not modify the routing problem
- The Scheduler does not access the PRNG or any randomness source (ADR-010)

---

### FR-2: Scheduler Inputs

**Description:**
The Scheduler accepts the following inputs per invocation:

1. **Routing Problem:** A validated SPEC-001 routing problem, uniquely identified by `problem_id`. The problem has passed Core validation per ADR-009. The Scheduler does not re-validate the problem.
2. **Workload Features:** A set of derived features computed by Core before Scheduler invocation (defined in FR-3). The Scheduler does not recompute features.
3. **Scheduler Configuration:** An active objective configuration (defined in FR-13) identifying the optimization mode and any mode-specific parameters. The Worker loads the configuration object from the configuration store using the `scheduler_config_id` from the job message before invoking Core. Core passes the resolved configuration to the Scheduler. The Scheduler does not receive a bare `scheduler_config_id` and does not perform configuration lookups.
4. **Backend Registry:** The set of registered solver backends and their declared capability profiles (defined in FR-4). The registry is accessible within Core's execution context and is fixed for the duration of a single scheduling invocation.

**Acceptance Criteria:**
- The Scheduler fails with a structured error if any required input is absent
- The Scheduler does not access any input not listed above during scoring and selection
- The Scheduler does not perform network calls, file I/O, or external lookups during the invocation

---

### FR-3: Workload Feature Computation Contract

**Description:**

**Scope of concern:** FR-3 defines the Scheduler's consumption contract: the feature names, types, valid ranges, and semantics the Scheduler requires as inputs. The computation formulas and SPEC-001 input property mappings are included here because no Core feature extraction specification currently exists. SPEC-001 FR-10 (Accepted) delegates the computation contract to SPEC-003. When a Core feature extraction specification is authored, the formula definitions should migrate to that specification, and SPEC-001 FR-10 should be updated to reference both documents. Until that migration, FR-3 is the authoritative definition of both the consumption contract and the computation formulas.

Core computes four derived workload features from the raw properties of a validated routing problem before invoking the Scheduler. The Scheduler consumes the features as provided, without modification or recomputation.

**Feature: `capacity_utilization_ratio`** (float64, range \[0.0, 1.0\])
- Semantics: The ratio of total stop demand to total available vehicle capacity for the problem
- Inputs from SPEC-001: per-stop demand values, `vehicle_count`, `capacity_per_vehicle` (SPEC-001 FR-10)
- Formula: (Σ demand per stop, over all stops) ÷ (vehicle_count × capacity_per_vehicle)
- Interpretation: 0.0 = no capacity demand; 1.0 = demand exactly equals capacity. Values above 1.0 indicate a problem that has already failed SPEC-001 FR-8 capacity feasibility; such problems must not reach the Scheduler.

**Feature: `problem_size_class`** (enumeration: `Small` | `Medium` | `Large`)
- Semantics: The routing problem's size tier, derived from `stop_count` using SPEC-001 FR-7 thresholds
- Inputs from SPEC-001: `stop_count` (SPEC-001 FR-10)
- Computation: `Small` if `stop_count` ∈ \[1, 25\]; `Medium` if `stop_count` ∈ \[26, 75\]; `Large` if `stop_count` ∈ \[76, ∞)
- Resolved by SPEC-001 FR-7. No owner confirmation required; this feature is fully specified.

**Feature: `time_window_density`** (float64, range \[0.0, 1.0\])
- Semantics: The fraction of stops in the problem that carry time window constraints
- Inputs from SPEC-001: per-stop `time_window_open` and `time_window_close` presence (SPEC-001 FR-9, FR-10)
- Formula: (count of stops where both `time_window_open` and `time_window_close` are present) ÷ stop_count
- Interpretation: 0.0 = no stops have time windows; 1.0 = all stops have time windows

**Feature: `average_time_window_width_seconds`** (float64, non-negative)
- Semantics: The average width, in seconds, of time windows across stops that carry time window constraints
- Inputs from SPEC-001: per-stop `time_window_open` and `time_window_close` (SPEC-001 FR-9, FR-10)
- Formula: (Σ (`time_window_close` − `time_window_open`) for stops where both `time_window_open` and `time_window_close` are present) ÷ (count of such stops). If no stop carries a time window, this value is 0.0.

**Acceptance Criteria:**
- Core produces all four features before the Scheduler is invoked
- The Scheduler rejects any invocation where one or more features are absent (see Failure Modes: `IncompleteFeaturesError`)
- The Scheduler rejects any invocation where a feature value is outside its declared valid range (see Failure Modes: `OutOfRangeFeaturesError`)
- `problem_size_class` is computed exactly per SPEC-001 FR-7 thresholds; no alternative definition is valid
- `capacity_utilization_ratio` is in the range \[0.0, 1.0\] on all valid problems; values outside this range are a Core validation fault, not a Scheduler concern
- Workload features are immutable for the duration of a single scheduling invocation

---

### FR-4: Backend Capability Declaration

**Description:**
Every solver backend registered with the Scheduler must declare a capability profile. The capability profile is the only mechanism by which the Scheduler determines what a backend can handle. No backend receives special-case treatment; all filtering and scoring logic is driven exclusively by declared capabilities (ADR-008 Backend Neutrality).

A capability profile must include:

| Field | Type | Description |
|---|---|---|
| `backend_id` | string | Unique, stable identifier for the backend |
| `supported_size_classes` | set\<SizeClass\> | Size classes (`Small`, `Medium`, `Large`) the backend can handle |
| `supports_time_windows` | bool | Whether the backend can handle time window constraints |
| `supports_capacity_constraints` | bool | Whether the backend can handle capacity constraints |
| `latency_profile` | map\<SizeClass, seconds\> | Estimated execution latency per supported size class. Must contain one entry for each size class in `supported_size_classes`. Values are positive numbers (seconds). For `DeadlineAware` filtering (FR-5 Phase 2), the Scheduler reads `latency_profile[problem_size_class]`. |
| `quality_profile` | enum | Relative solution quality. Valid values (ordered ascending): `Baseline` \< `Competitive` \< `Near-Optimal`. No other values are valid. A capability profile declaring an unrecognized `quality_profile` value is malformed. |
| `cost_profile` | positive number | Per-execution cost estimate. Dimensionless unit; units are defined relative to a common scale established by the backend registration contract (OQ-2). For `BudgetCapped` filtering (FR-5 Phase 2), the Scheduler reads `cost_profile` directly. |
| `is_provisional` | bool | Whether this backend is under evaluation rather than in production use. Provisional backends (`is_provisional = true`) are excluded from selection under all objective modes except `Experimental` via the `ProvisionalBackendExcluded` condition in FR-5 Phase 2. |
| `supported_contract_version` | uint32 | The SolverContract version this backend implements (SPEC-004 FR-14). The Worker reads this field before dispatch and validates that the version it intends to send matches the declared value. A pre-dispatch mismatch is a Worker configuration error; a post-dispatch mismatch detected by the backend produces `ContractVersionMismatch` (SPEC-004 FR-8). Current value: 1. |

The mechanism by which capability profiles are registered with the Scheduler is an open question (OQ-2). The OQ-2 resolution is responsible for validating capability profile internal consistency at registration time.

**Acceptance Criteria:**
- Every registered backend has a capability profile present at Scheduler invocation time
- A backend without a capability profile is not considered for selection
- The Scheduler does not infer missing capability fields from context
- No `backend_id` receives a scoring advantage or disadvantage by identity (ADR-008)
- `latency_profile` contains exactly one entry per size class in `supported_size_classes`
- `quality_profile` is one of the three enumerated values; profiles with unrecognized values are not registered

---

### FR-5: Hard Eligibility Filtering

**Description:**
Before scoring, the Scheduler applies two sequential filter phases to eliminate backends that cannot serve the current problem or are prohibited by the active objective mode. All backends that pass both phases proceed to scoring (FR-7). Rejections at either phase produce `rejection_reasons` entries in the decision record with distinct reason codes.

**Phase 1: Structural Hard Filters**

A backend fails Phase 1 if any of the following conditions hold:

1. **Size class mismatch:** The problem's `problem_size_class` is not in the backend's `supported_size_classes`. Reason code: `SizeClassMismatch`.
2. **Time window unsupported:** The problem has one or more stops with time window constraints AND the backend's `supports_time_windows` is `false`. Reason code: `TimeWindowUnsupported`.
3. **Capacity constraint unsupported:** The problem's `capacity_utilization_ratio` > 0.0 AND the backend's `supports_capacity_constraints` is `false`. Reason code: `CapacityConstraintUnsupported`.

Phase 1 conditions are derived from the routing problem's structural properties and the backend's declared structural capabilities. They are independent of the active objective mode.

**Phase 2: Objective-Mode Hard Limits**

Phase 2 is applied to backends that passed Phase 1. Phase 2 conditions depend on the active objective mode and the backend's declared profile values. A backend fails Phase 2 if any applicable condition holds.

| Condition | Active Under | Reason Code |
|---|---|---|
| `is_provisional = true` | All modes except `Experimental` | `ProvisionalBackendExcluded` |
| `latency_profile[problem_size_class] > deadline_seconds` | `DeadlineAware` mode only | `ExceedsDeadline` |
| `cost_profile > budget_limit` | `BudgetCapped` mode only | `ExceedsBudget` |

Under `Experimental` mode, Phase 2 applies no limits. All Phase 1 survivors proceed directly to scoring.

If a backend satisfies more than one Phase 2 condition, it is rejected with the reason code of the first applicable condition in the order listed above.

If no backend passes both Phase 1 and Phase 2, the Scheduler invocation fails with `NoEligibleSolver` (see FR-9 and Failure Modes).

**Acceptance Criteria:**
- Every Phase 1 rejection includes a reason code from the Phase 1 set (`SizeClassMismatch`, `TimeWindowUnsupported`, `CapacityConstraintUnsupported`)
- Every Phase 2 rejection includes a reason code from the Phase 2 set (`ProvisionalBackendExcluded`, `ExceedsDeadline`, `ExceedsBudget`)
- Phase 2 is applied only to Phase 1 survivors
- Under `Experimental` mode, no Phase 2 limit is applied; backends with `is_provisional = true` proceed to scoring
- Hard filtering logic (both phases) does not inspect `backend_id`
- A backend that passes both phases proceeds to scoring regardless of its `quality_profile` or `cost_profile`

---

### FR-6: Optimization Objective Modes

**Description:**
The Scheduler supports seven configurable optimization objectives. The active objective is specified in the Scheduler configuration (FR-13) and determines how eligible backends are scored. The objective mode also determines which Phase 2 hard limits are active (FR-5 Phase 2).

| Mode | Semantics | Required Parameters |
|---|---|---|
| `CheapestValid` | Prefer the backend with the lowest `cost_profile`. `quality_profile` is a tiebreaker. | None |
| `FastestValid` | Prefer the backend with the lowest `latency_profile[problem_size_class]`. `quality_profile` is a tiebreaker. | None |
| `Balanced` | Balance predicted cost, latency, and solution quality per explicitly configured weights. All three weight fields (`cost_weight`, `latency_weight`, `quality_weight`) are required. A `Balanced` configuration with any weight omitted is `InvalidConfiguration`. | `cost_weight`, `latency_weight`, `quality_weight` |
| `BestQuality` | Prefer the backend with the highest `quality_profile` tier. Cost and latency are tiebreakers. | None |
| `DeadlineAware` | FR-5 Phase 2 eliminates any backend where `latency_profile[problem_size_class] > deadline_seconds`. Among Phase 2 survivors, prefer highest `quality_profile`. | `deadline_seconds` |
| `BudgetCapped` | FR-5 Phase 2 eliminates any backend where `cost_profile > budget_limit`. Among Phase 2 survivors, prefer highest `quality_profile`. | `budget_limit` |
| `Experimental` | Suppresses the `ProvisionalBackendExcluded` limit in FR-5 Phase 2. Backends with `is_provisional = true` are eligible for scoring alongside non-provisional backends. Used to evaluate new backends before production promotion. Scoring follows the same rules as `BestQuality`. | None |

**Parameter Schemas:**

`Balanced` mode parameters:
- `cost_weight` (float, required): Weight for cost dimension. Range: \[0.0, 1.0\].
- `latency_weight` (float, required): Weight for latency dimension. Range: \[0.0, 1.0\].
- `quality_weight` (float, required): Weight for quality dimension. Range: \[0.0, 1.0\].
- Constraint: `cost_weight + latency_weight + quality_weight` must equal 1.0 within implementation tolerance.
- All three fields must be explicitly present. A `Balanced` configuration with any weight field absent is `InvalidConfiguration`. The Scheduler does not inherit or supply default weights for omitted fields.

`DeadlineAware` mode parameters:
- `deadline_seconds` (positive integer, required): Maximum acceptable latency for this invocation, in seconds. A backend whose `latency_profile[problem_size_class]` exceeds this value is rejected in Phase 2.

`BudgetCapped` mode parameters:
- `budget_limit` (positive number, required): Maximum acceptable cost for this invocation. Dimensionless unit; same scale as `cost_profile` in FR-4. A backend whose `cost_profile` exceeds this value is rejected in Phase 2.

Modes with no parameters: `CheapestValid`, `FastestValid`, `BestQuality`, `Experimental`. Providing unexpected parameters to these modes is `InvalidConfiguration`.

**Acceptance Criteria:**
- An unrecognized `objective_mode` causes `InvalidConfiguration` failure before any backend is evaluated
- The active objective mode is recorded verbatim in the decision record
- All eligible backends are scored under the same objective mode; mode-specific logic does not reference `backend_id`
- A `Balanced` configuration with any weight field absent causes `InvalidConfiguration` before any backend is evaluated
- Under `Experimental` mode, backends with `is_provisional = true` pass Phase 2 and reach scoring

---

### FR-7: Candidate Scoring

**Description:**
The Scheduler assigns a numeric score to each backend that passed both Phase 1 and Phase 2 eligibility filtering (FR-5). Scoring is deterministic: given the same workload features, capability profiles, and objective configuration, the same scores are produced on every invocation.

Scoring constraints:
- Scores are computed exclusively from declared capability profiles (FR-4), workload features (FR-3), and the active objective configuration (FR-6)
- No backend receives a score bonus or penalty based on its `backend_id`
- Scoring logic does not access the routing problem's raw properties directly; it operates only on the derived workload features
- The specific scoring formula for each objective mode is deferred to implementation planning (OQ-4)

**Confidence score:** The scoring step must retain the top two scores among eligible candidates in order to compute `confidence_score`. The `confidence_score` for a given invocation equals (score of the selected backend) − (score of the next-best eligible backend). When only one backend passes all filter phases, `confidence_score` is 0.0.

**Acceptance Criteria:**
- Given identical inputs, scoring produces identical results on every invocation
- Every eligible backend receives a numeric score
- Higher score = more preferred candidate
- The top two scores are retained to enable `confidence_score` computation
- The scoring formula used, including the active objective mode, is captured in the decision record

---

### FR-8: Candidate Selection

**Description:**
The Scheduler selects the highest-scoring eligible backend as the recommended solver. If two or more backends share the top score, the Scheduler selects the backend with the lexicographically smallest `backend_id`. This is a fixed behavioral specification, not a configuration option. The lexicographic tiebreaker is applied uniformly under all objective modes; it is deterministic and introduces no optimization preference.

The Scheduler does not execute the selected solver. The decision record identifies the `selected_backend_id`. Execution is the Worker's responsibility.

**Acceptance Criteria:**
- Exactly one `backend_id` is recorded in the decision record as `Selected`
- All other eligible (scored but not selected) backends are recorded with their scores
- All ineligible backends are recorded with their rejection reason codes
- Selection is deterministic given the same inputs
- When two or more backends share the top score, the one with the lexicographically smallest `backend_id` is selected

---

### FR-9: Fallback Behavior

**Description:**
The Scheduler does not apply silent fallbacks. If no backend is eligible after Phase 1 and Phase 2 filtering, the Scheduler produces a `NoEligibleSolver` decision record and returns a structured failure. The failure record includes:
- `problem_id`
- `problem_size_class`
- Rejection reasons for every registered backend, covering both Phase 1 structural rejections and Phase 2 objective-mode hard limit rejections
- Active objective mode
- Timestamp

The Scheduler does not retry with a relaxed configuration, override constraint requirements, or silently substitute a different backend. Fallback strategy — such as resubmitting with a different objective mode — is the caller's responsibility.

**Acceptance Criteria:**
- `NoEligibleSolver` failure is a structured response, not a panic or unhandled exception
- The failure record identifies every registered backend and its rejection reason, from Phase 1 or Phase 2
- No implicit fallback or silent override occurs at any point in the invocation

---

### FR-10: Decision Record

**Description:**
Every Scheduler invocation — successful or failed — produces a decision record. The decision record is the primary output of the Scheduler.

| Field | Present When | Description |
|---|---|---|
| `decision_id` | Always | Unique identifier for this decision |
| `problem_id` | Always | Identifies the routing problem |
| `objective_mode` | Always | Active objective mode at time of decision |
| `workload_features_snapshot` | Always | Snapshot of all four workload features at invocation time |
| `selected_backend_id` | `decision_status = Selected` | `backend_id` of the chosen solver |
| `candidate_scores` | Always | Numeric score for every backend that passed Phase 1 and Phase 2 filtering |
| `rejection_reasons` | Always | Per-backend reason codes for every backend that failed Phase 1 or Phase 2 filtering |
| `predicted_latency` | `decision_status = Selected` | `latency_profile[problem_size_class]` for the selected backend |
| `predicted_cost` | `decision_status = Selected` | `cost_profile` value for the selected backend |
| `predicted_quality` | `decision_status = Selected` | `quality_profile` value for the selected backend |
| `confidence_score` | `decision_status = Selected` | Score margin: (selected backend score) − (next-best eligible backend score). 0.0 when only one backend passed all filter phases. |
| `decision_status` | Always | One of: `Selected`, `NoEligibleSolver`, `InvalidConfiguration`, `IncompleteFeaturesError`, `OutOfRangeFeaturesError` |
| `timestamp` | Always | Wall-clock timestamp at decision time |
| `actual_outcome` | Post-execution | Reserved field name. Type and semantics defined by the Core specification. Absent when the Scheduler produces the record. |
| `hindsight_quality` | Post-execution | Reserved field name. Type and semantics defined by the Core specification. Absent when the Scheduler produces the record. |

**Acceptance Criteria:**
- Every invocation produces a structurally complete decision record
- The `workload_features_snapshot` in the record matches the features used for scoring
- `actual_outcome` and `hindsight_quality` are absent (null/unset) when the Scheduler produces the record
- The decision record is serializable without information loss
- `rejection_reasons` covers every backend that failed Phase 1 or Phase 2 filtering
- `confidence_score` equals the score margin between the selected and next-best eligible backend; 0.0 when only one eligible backend passed all filter phases

---

### FR-11: Decision Record Persistence

**Description:**
The Scheduler produces the decision record. The Worker persists it. The Scheduler has no persistence dependency and makes no assumptions about where the record is stored or how.

This separation ensures the Scheduler is testable in isolation, without a database, file system, or external infrastructure.

**Architecture.md consistency note:** The Daedalus Scheduler Responsibilities list in `docs/architecture.md` includes "persist explainable decisions." This specification interprets that responsibility as producing decision records in a form suitable for persistence, not performing persistence directly. The runtime flow in `docs/architecture.md` ("Scheduler decision → Worker persists decision → Worker executes selected solver") is authoritative for this component boundary. The Scheduler Responsibilities list in `docs/architecture.md` should be updated to read "produce explainable decision records" during the next architecture documentation pass.

**Acceptance Criteria:**
- The Scheduler returns the decision record to its caller; it does not write the record to any store
- The Scheduler does not fail if persistence subsequently fails (it is unaware of persistence outcomes)
- The decision record contains all information necessary for the Worker to persist it without additional Scheduler calls

---

### FR-12: Hindsight Evaluation

**Description:**
After solver execution completes, Core updates the decision record with actual outcome data by populating `actual_outcome` and `hindsight_quality`. This is the "regret calculation" referenced in the architecture document. The Scheduler is not involved in hindsight evaluation.

The Scheduler produces predicted values. Core produces the hindsight assessment. SPEC-003 reserves the field names `actual_outcome` and `hindsight_quality` in the decision record schema. Their types, formats, and population logic are defined by the Core specification. SPEC-003 does not constrain the types or semantics of these fields beyond reserving their names and specifying that they are absent at Scheduler decision time. If the Core specification uses different field names for these concepts, a SPEC-003 revision is required to align the schema.

**Acceptance Criteria:**
- `actual_outcome` and `hindsight_quality` field names are reserved in the decision record schema
- These fields are absent when the Scheduler produces the record at decision time
- Population of these fields is not a Scheduler responsibility
- A decision record without hindsight data is complete for all Scheduler purposes

---

### FR-13: Scheduler Configuration

**Description:**
The Scheduler is configured via a scheduler configuration object resolved before invocation. The Worker is responsible for loading the configuration object from the configuration store using the `scheduler_config_id` from the job message before invoking Core. Core passes the resolved configuration to the Scheduler. The Scheduler does not perform configuration lookups.

The configuration must specify:

- **`objective_mode`**: The active optimization objective (from FR-6)
- **`mode_parameters`**: Mode-specific parameters per the schemas defined in FR-6. Required parameters vary by mode; see FR-6 Parameter Schemas.

The `scheduler_config_id` referenced in SPEC-001 FR-13 is a reference to a stored configuration. The Scheduler receives a resolved configuration object; it does not receive a bare `scheduler_config_id`.

**Balanced mode constraint:** For configurations specifying `objective_mode = Balanced`, all three weight fields (`cost_weight`, `latency_weight`, `quality_weight`) must be explicitly present in `mode_parameters`. A `Balanced` configuration with any weight field absent is `InvalidConfiguration`. The Scheduler returns `InvalidConfiguration` before evaluating any backend. Implicit inheritance of default weights is not supported; every `Balanced` configuration must be self-describing.

**Acceptance Criteria:**
- The Scheduler validates that `objective_mode` is a known mode before evaluating any backend
- An unrecognized `objective_mode` causes `InvalidConfiguration` failure before any backend is evaluated
- A `Balanced` mode configuration with any weight field absent causes `InvalidConfiguration` before any backend is evaluated
- The configuration is immutable for the duration of a single invocation
- The Scheduler does not perform configuration lookups; it receives only resolved configuration objects

---

### FR-14: Default Scheduler Configuration

**Description:**
SPEC-001 FR-13 requires a default scheduler configuration to be defined in SPEC-003. The default configuration is applied when a routing problem submission does not specify an explicit `scheduler_config_id`.

The default scheduler configuration is `Balanced` mode with equal weight allocation across all three optimization dimensions: cost, latency, and quality. No optimization dimension is preferred over another in the default configuration.

The specific numeric encoding used to implement equal weights (for example, 1/3 per dimension) is an implementation planning decision. The implementation must represent equal weighting for all three dimensions without introducing implicit preference through approximation artifacts.

The default `Balanced` configuration has all three weight fields present and equal, satisfying FR-13's requirement that all weights be explicitly specified.

**Acceptance Criteria:**
- The default configuration applies `Balanced` mode
- All three weight fields (`cost_weight`, `latency_weight`, `quality_weight`) are present in the default configuration
- All three weight fields are equal in value
- No weight field contains a value that differs from the others due solely to floating-point decimal rounding without compensating for the resulting bias
- When a job is submitted without an explicit `scheduler_config_id`, the Scheduler operates under this default configuration
- The default configuration satisfies SPEC-001 FR-13

---

### FR-15: Telemetry

**Description:**
The Scheduler must produce the following OpenTelemetry span on every invocation.

**Span: `scheduler.score_solvers`**
- Scope: Begins before Phase 1 hard eligibility filtering; ends after candidate selection or failure determination
- Required span attributes:

| Attribute | Description |
|---|---|
| `problem_id` | Identifies the routing problem |
| `objective_mode` | Active objective mode |
| `problem_size_class` | Problem size class from workload features |
| `backend_count_registered` | Total backends in registry |
| `backend_count_eligible` | Backends passing both Phase 1 and Phase 2 filtering |
| `backend_count_rejected` | Backends failing Phase 1 or Phase 2 filtering |
| `selected_backend_id` | Chosen backend (absent if `NoEligibleSolver`) |
| `decision_status` | `Selected`, `NoEligibleSolver`, `InvalidConfiguration`, `IncompleteFeaturesError`, or `OutOfRangeFeaturesError` |

- Rejection reasons are recorded as span events (one event per rejected backend), not as structured log entries
- On `NoEligibleSolver`, `InvalidConfiguration`, `IncompleteFeaturesError`, or `OutOfRangeFeaturesError`, the span status is Error

**Acceptance Criteria:**
- `scheduler.score_solvers` is emitted on every Scheduler invocation, successful or not
- All required attributes are present on every emission
- Rejection reasons appear as span events
- The span is correlatable with the decision record via `problem_id`

---

# Non-Requirements

- The Scheduler does not execute any solver
- The Scheduler does not generate routing problems (SPEC-002)
- The Scheduler does not validate routing problems (ADR-009)
- The Scheduler does not compute workload features (Core, per FR-3)
- The Scheduler does not write to the evidence log or any database (Worker responsibility, per FR-11)
- The Scheduler does not make external network calls during an invocation
- The Scheduler does not implement solver-specific optimization logic
- The Scheduler does not route to quantum hardware (ADR-007 deferred)
- The Scheduler does not learn from execution history at MVP scope (historical policy learning is a future capability)

---

# Assumptions

1. Core validates the routing problem before invoking the Scheduler. The Scheduler will not receive problems that have failed SPEC-001 or ADR-009 validation.
2. Core computes all four workload features before invoking the Scheduler. The Scheduler will not receive partial feature sets.
3. The backend registry is populated before the Scheduler is invoked. An empty registry is a configuration fault, not a condition the Scheduler compensates for.
4. SPEC-001 FR-7 size class thresholds (Small 1–25, Medium 26–75, Large 76+) are stable. Changes to these thresholds require a SPEC-001 version boundary and a corresponding update to this spec.
5. MVP backends are: nearest-neighbor classical baseline, greedy insertion classical baseline, and QUBO simulated annealing proxy (ADR-007). The Scheduler makes no backend-specific assumptions about these or any future backends.
6. An evidence log specification defining the decision record persistence interface will be authored before SPEC-003 implementation can be completed. SPEC-003 defines the decision record schema (FR-10). The evidence log specification will define the Worker-facing persistence contract. FR-11 and the Definition of Done criterion "Decision records are persisted and queryable per the evidence log contract" are contingent on that specification.

---

# Constraints

1. The Scheduler must not encode special-case logic for any specific `backend_id`. All selection logic is driven by declared capabilities (ADR-008 Backend Neutrality).
2. The Scheduler must produce a decision record on every invocation, including failures (Evidence Over Hype principle, architecture.md).
3. Scoring must be deterministic given the same inputs (ADR-010 spirit applied to policy decisions).
4. The Scheduler must not access the PRNG or any randomness source (ADR-010).
5. The Scheduler must emit the `scheduler.score_solvers` OTel span on every invocation (architecture.md).
6. The Scheduler operates exclusively on normalized solver candidates (ADR-008).
7. Workload features must originate from Core. The Scheduler must not recompute features from raw problem properties.

---

# Inputs

| Input | Source | Format | Notes |
|---|---|---|---|
| Validated routing problem | Core (from PostgreSQL, after Core validation) | SPEC-001 domain model | Must have passed ADR-009 Core validation |
| Workload features | Core (computed by Core before Scheduler invocation) | Four typed fields per FR-3 | All four must be present; any absent feature is a failure condition |
| Scheduler configuration | Worker (resolved from configuration store using `scheduler_config_id` from the job message before Core invocation) | Objective mode + mode parameters per FR-13 | Not a bare `scheduler_config_id` |
| Backend registry | Infrastructure, populated at startup | Set of capability profiles per FR-4 | Fixed for duration of invocation; accessible within Core execution context |

---

# Outputs

| Output | Consumer | Format | Notes |
|---|---|---|---|
| Decision record | Worker (for persistence), caller | Structured per FR-10 | Always produced; may be a failure record |
| `scheduler.score_solvers` OTel span | Observability infrastructure | OpenTelemetry span | Always emitted per FR-15 |

---

# Failure Modes

### NoEligibleSolver

**Condition:** All registered backends fail eligibility filtering (Phase 1 or Phase 2 of FR-5), or the backend registry is empty.
**Behavior:** Scheduler produces a `NoEligibleSolver` decision record identifying every backend and its rejection reason (from Phase 1 or Phase 2). Invocation returns a structured failure.
**Fallback:** None. Fallback strategy (e.g., retry with a different objective, adjust problem constraints) is the caller's responsibility.
**Caller-visible result:** Structured failure record with per-backend rejection reasons.

---

### InvalidConfiguration

**Condition:** The scheduler configuration specifies an unrecognized `objective_mode`; a required mode parameter is absent or of the wrong type; or a `Balanced` configuration has any weight field omitted.
**Behavior:** Scheduler fails before evaluating any backend.
**Fallback:** None.
**Caller-visible result:** Structured failure identifying the invalid or missing field.

---

### IncompleteFeaturesError

**Condition:** One or more required workload features are absent from the input.
**Behavior:** Scheduler fails immediately before evaluating any backend. This is a Core contract violation.
**Fallback:** None. This is not a recoverable condition at the Scheduler level.
**Caller-visible result:** Structured failure identifying the missing feature(s).

---

### OutOfRangeFeaturesError

**Condition:** One or more workload feature values are outside their declared valid ranges (e.g., `capacity_utilization_ratio` = 1.5). This is a Core contract violation, not a routing problem constraint failure.
**Behavior:** Scheduler fails immediately before evaluating any backend. The Scheduler does not clamp, correct, or recompute out-of-range values.
**Fallback:** None. This condition represents a violation of the Core-to-Scheduler contract defined in FR-3 and is not recoverable at the Scheduler level.
**Caller-visible result:** Structured failure identifying the offending feature(s) and their received values.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer | Yes — Scheduler is a new domain component within Core |
| API Layer | Indirect — API validates `scheduler_config_id` at submission (ADR-009); no new API surface |
| Persistence | Indirect — Worker persists decision records produced by Scheduler |
| Solver Runtime | Yes — All MVP solver backends must declare capability profiles (FR-4) |
| Observability | Yes — `scheduler.score_solvers` span, decision record schema |
| Configuration | Yes — Scheduler configuration structure (FR-13, FR-14) |
| Security | None — No new trust boundaries beyond the existing ADR-009 model |

---

# Testability

1. **Unit: Hard eligibility filtering** — Given a problem with a known `problem_size_class` and constraint profile, verify correct backends are excluded with correct reason codes.
2. **Unit: Scoring determinism** — Given identical inputs, scoring produces identical numeric scores across invocations.
3. **Unit: Decision record completeness** — Every successful invocation produces a decision record containing all required fields with correct values.
4. **Unit: `NoEligibleSolver` path** — When all backends fail hard filtering, a structurally valid `NoEligibleSolver` failure record is produced (not a panic).
5. **Unit: `InvalidConfiguration` path** — An unrecognized `objective_mode` produces a structured failure before any backend is evaluated.
6. **Unit: Backend Neutrality** — No test may pass solely because a specific `backend_id` is privileged in filtering or scoring logic.
7. **Unit: Objective mode coverage** — Each of the seven objective modes produces a deterministic selection given the same eligible candidate set.
8. **Unit: Empty registry** — An empty backend registry produces a `NoEligibleSolver` failure record.
9. **Integration: Scheduler behavior on out-of-range features** — Given a feature set where one value is outside its declared valid range (e.g., `capacity_utilization_ratio` = 1.5), the Scheduler produces an `OutOfRangeFeaturesError` structured failure rather than proceeding to eligibility evaluation. The Scheduler does not clamp, correct, or recompute out-of-range values.
10. **Integration: OTel span emission** — `scheduler.score_solvers` is emitted on both successful and failed invocations with all required attributes.
11. **Unit: NoEligibleSolver rejection record completeness** — Given a backend registry where every registered backend's `supported_size_classes` excludes the problem's `problem_size_class`, the Scheduler produces a `NoEligibleSolver` failure record with exactly one `SizeClassMismatch` rejection reason entry per registered backend. The record is structurally complete.
12. **Unit: Phase 2 provisional filtering** — Under any non-`Experimental` objective mode, a backend with `is_provisional = true` is rejected in Phase 2 with reason code `ProvisionalBackendExcluded`. Under `Experimental` mode, the same backend proceeds to scoring.
13. **Unit: Balanced mode requires all three weights** — A `Balanced` configuration with any one weight field absent causes `InvalidConfiguration` failure before any backend is evaluated.

---

# Observability Requirements

Operational questions the Scheduler must answer:

1. For a given `problem_id`, which backend was selected and why?
2. Which backends were rejected and what were the rejection reasons?
3. What was the active objective mode?
4. What workload features drove the decision?
5. How many backends were eligible for a given invocation?
6. How often does `NoEligibleSolver` occur, and for which problem characteristics?

These questions are answered by the combination of:
- Decision records (per FR-10, persisted by Worker)
- `scheduler.score_solvers` OTel span (per FR-15)

No additional telemetry is required at MVP scope.

---

# Security Considerations

The Scheduler receives already-validated inputs from Core (ADR-009). No new trust boundaries are introduced. Scheduler configuration is validated by the API at submission time (ADR-009).

The Scheduler does not accept inputs from external or untrusted callers at runtime. All inputs are produced by Core components operating within the existing trust model established by ADR-009.

No new authentication or authorization requirements arise from this component.

---

# Performance Considerations

The Scheduler is a pure CPU computation with no I/O. At MVP scale (three backends, three size classes, seven objective modes), scoring and selection complete in microseconds.

No performance benchmarking is required at MVP scope. The `scheduler.score_solvers` span provides latency observability if performance degrades in future iterations with a larger backend registry.

---

# Documentation Updates Required

- `docs/architecture.md`: Add Scheduler to component descriptions; confirm runtime flow section reflects FR-1 responsibilities
- `docs/architecture.md`: Correct Scheduler Responsibilities list entry from "persist explainable decisions" to "produce explainable decision records" (per FR-11 resolution note)
- SPEC-001: FR-10 workload feature computation contract is satisfied by FR-3. No update to SPEC-001 is required.
- SPEC-001: FR-13 default scheduler configuration reference is satisfied by FR-14.
- ADR-008: No update required; SPEC-003 implements the normalized contract defined in ADR-008

---

# Open Questions

## Resolved

### OQ-1: Workload Feature Computation Formulas

**Status: Resolved — 2026-06-08 — ODR-1, ODR-2, ODR-3**

| Feature | Formula | Resolution |
|---|---|---|
| `problem_size_class` | Per SPEC-001 FR-7 thresholds (pre-existing; no owner decision required) | SPEC-001 FR-7 |
| `capacity_utilization_ratio` | (Σ demand per stop, over all stops) ÷ (vehicle_count × capacity_per_vehicle) | ODR-1 |
| `time_window_density` | (count of stops where both `time_window_open` and `time_window_close` are present) ÷ stop_count | ODR-2 |
| `average_time_window_width_seconds` | (Σ (`time_window_close` − `time_window_open`) for stops with both fields present) ÷ (count of such stops); 0.0 if no windowed stops | ODR-3 |

All formulas are incorporated in FR-3.

---

### OQ-3: Objective Mode Parameter Structures

**Status: Resolved — 2026-06-08 — ODR-4, ODR-5**

Parameter schemas are defined in FR-6. Cost units are dimensionless (ODR-5); scale is defined by the backend registration contract (OQ-2). Balanced mode requires all three weight fields explicitly (ROQ-1). See FR-6 Parameter Schemas.

---

### OQ-5: Tiebreaker Rule

**Status: Resolved — 2026-06-08 — ROQ-2**

Lexicographically smallest `backend_id`. Applied uniformly under all objective modes. Deterministic; introduces no optimization preference. Incorporated in FR-8.

---

### OQ-6: Default Scheduler Configuration

**Status: Resolved — 2026-06-08 — ODR-6, ROQ-3**

`Balanced` mode with equal weight allocation across cost, latency, and quality. Equal weighting is the architectural intent; no dimension is preferred in the default. The specific numeric encoding (e.g., 1/3 per dimension) is implementation planning; the implementation must not introduce implicit preference through rounding artifacts. Incorporated in FR-14.

---

## Open (Non-Blocking)

### OQ-2: Backend Capability Registration Mechanism

**Question:** How are backend capability profiles registered with the Scheduler? Options include: static configuration file, startup-time code registration, compile-time capability table, or runtime service registration.

**Why it matters:** Affects testability, the process for adding a new backend, and whether the registry can change at runtime.

**Blocking:** Not blocking SPEC-003 acceptance if the capability profile fields (FR-4) are accepted as specified. Blocking for implementation planning.

**Capability profile consistency note:** The OQ-2 resolution must address internal consistency validation for capability profiles at registration time. Minimum consistency rules: `latency_profile` must contain one entry per size class in `supported_size_classes`; `quality_profile` must be a value from the FR-4 enumeration; `is_provisional` must be present. The Scheduler trusts registered profiles as internally consistent at invocation time and does not re-validate them (ADR-008 Backend Neutrality).

---

### OQ-4: Scoring Formula Per Objective Mode

**Question:** What is the numeric scoring formula for each objective mode?

**Why it matters:** Scoring must be deterministic and fully specified to be testable. An underspecified formula enables inconsistent implementations.

**Note:** `confidence_score` is resolved: it equals (selected backend score) − (next-best eligible backend score); 0.0 when only one backend passes all filter phases. See FR-7 and FR-10. This question covers only the numeric scoring formula for each objective mode.

**Blocking:** Not blocking SPEC-003 acceptance if the scoring constraints (deterministic, capability-only, backend-neutral) are accepted. Blocking for implementation planning.

---

# Acceptance Checklist

- [x] Problem is clearly defined
- [x] Requirements are testable
- [x] Non-requirements are documented
- [x] Assumptions are explicit
- [x] Failure modes are defined
- [x] Observability requirements exist
- [x] Security considerations exist
- [x] Documentation updates are identified
- [x] OQ-1 resolved — workload feature computation formulas (ODR-1, ODR-2, ODR-3)
- [ ] OQ-2 resolved — backend capability registration mechanism (non-blocking for acceptance; implementation planning)
- [x] OQ-3 resolved — objective mode parameter structures (ODR-4, ODR-5)
- [ ] OQ-4 resolved — scoring formula per objective mode (non-blocking for acceptance; implementation planning)
- [x] OQ-5 resolved — tiebreaker rule (ROQ-2)
- [x] OQ-6 resolved — default scheduler configuration (ODR-6, ROQ-3)

---

# Definition of Done

This feature is complete when:

- All blocking open questions (OQ-1, OQ-3, OQ-5, OQ-6) are resolved and incorporated in this specification (complete as of 2026-06-08)
- OQ-2 (registration mechanism) and OQ-4 (scoring formula) are resolved during implementation planning
- All functional requirements (FR-1 through FR-15) are implemented and acceptance criteria pass
- Unit and integration tests covering FR-1 through FR-15 pass
- `scheduler.score_solvers` OTel span is emitted and verifiable in the test environment
- Decision records are persisted and queryable per the evidence log contract
- SPEC-001 FR-13 is satisfied (default configuration defined and applied)
- Backend capability profiles exist for all MVP backends (ADR-007)
- Engineering review passes
- Specification status is updated to Verified
