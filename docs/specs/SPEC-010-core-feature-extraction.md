# SPEC-010: Core Feature Extraction

## Metadata

**Feature ID:** SPEC-010

**Title:** Core Feature Extraction

**Status:** Draft

**Author:** Darkhorse286

**Created:** 2026-06-14

**Last Updated:** 2026-06-14

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-001, ADR-006, ADR-009, ADR-010, ADR-011

**Related Specs:** SPEC-001, SPEC-003, SPEC-005, SPEC-006, SPEC-007

---

# Problem Statement

Workload features are derived properties of a routing problem that the Scheduler uses to evaluate solver eligibility and score candidates. Four features are currently defined in SPEC-003 FR-3, under a temporary ownership arrangement. SPEC-003 explicitly notes: "The computation formulas and SPEC-001 input property mappings are included here because no Core feature extraction specification currently exists."

This produces three problems.

**No authoritative owner.** Feature computation formulas, feature definitions, feature validation rules, and feature reproducibility requirements have no single owning component. SPEC-003 owns the Scheduler's consumption contract; it does not own computation. Core computes features today without a specification that governs how.

**Incomplete feature set.** Two features with demonstrated utility for workload classification — geographic compactness and service time pressure — have no formal definition anywhere in the specification suite. Schedulers in future iterations and benchmark analysis both depend on a complete, extensible feature set.

**Fragile ownership boundary.** Without a dedicated specification, the line between feature computation (a Core responsibility) and policy evaluation (a Scheduler responsibility) is blurred. Changes to feature definitions risk uncontrolled side effects on the Scheduler policy contract.

SPEC-010 resolves these problems. It designates Core Feature Extraction as the authoritative owner of all workload feature computation, migrates the four formulas from SPEC-003, defines two additional justified features, and establishes the feature output contract, reproducibility requirements, validation requirements, execution timing, and observability obligations.

---

# Business Value

- Establishes a single, durable owner for feature computation logic. Changes to a feature formula have one place to be made and one specification to review.
- Enables the Scheduler to remain a pure policy engine. The Scheduler receives features as inputs; it does not compute them.
- Provides a clean basis for future Scheduler policy enhancements. Additional features can be added to SPEC-010 without modifying the Scheduler's eligibility or scoring logic.
- Enables workload classification and benchmark analysis across jobs. Feature values captured in decision records enable post-hoc analysis of which workload characteristics predict solver performance.
- Demonstrates production-grade separation of concerns in a complex domain system.

---

# Employer Signaling

- System Design
- Domain Modeling
- Reliability Engineering
- Observability

---

# Domain Concept

A workload feature is a derived numeric or categorical property of a routing problem that quantifies a structurally observable characteristic of the workload. Features are computed from the raw properties of a validated routing problem using deterministic, reproducible algorithms. They do not require solver execution, scheduler configuration, or backend knowledge. They do not express preferences, policies, or recommendations.

Core Feature Extraction is the component within Daedalus Core responsible for producing the complete set of workload features from a validated routing problem. It is not a separate service. It is a defined processing step within Core, invoked by Core after authoritative domain validation (ADR-009) and before Scheduler invocation.

Feature values describe the workload. The Scheduler interprets them as inputs to policy evaluation. The distinction is strict: Core Feature Extraction computes; the Scheduler decides.

---

# Requirements

### FR-1: Responsibility Scope and Component Boundaries

**Description:**
SPEC-010 defines the responsibilities of Core Feature Extraction and the adjacent responsibilities it must not perform.

| Responsibility | Owner |
|---|---|
| Feature definitions (names, semantics, formulas, valid ranges) | SPEC-010 |
| Feature computation algorithms | SPEC-010 |
| Feature output contract (WorkloadFeatures structure, schema version) | SPEC-010 |
| Feature validation before Scheduler invocation | SPEC-010 |
| Feature reproducibility requirements | SPEC-010 |
| Feature observability (features.extract span, required attributes) | SPEC-010 |
| Scheduler consumption contract (which features the Scheduler requires as inputs) | SPEC-003 FR-3 |
| Feature persistence schema | SPEC-006 (via decision record; see FR-10) |
| Scheduler eligibility filtering using feature values | SPEC-003 FR-5 |
| Scheduler scoring using feature values | SPEC-003 FR-7 |
| Solver execution | SPEC-004, SPEC-005 |
| Quality evaluation | SPEC-007 |
| Report generation | SPEC-009 |
| Routing problem validation | ADR-009, SPEC-001 |
| Workload generation | SPEC-002 |

**Core Feature Extraction does not:**
- Invoke the Scheduler
- Access the PRNG or any randomness source (feature computation is purely deterministic arithmetic on problem properties)
- Access solver backends or their capability profiles
- Perform network calls, file I/O, or external lookups
- Apply any policy, preference, or optimization objective

**Acceptance Criteria:**
- No feature computation logic appears in Scheduler code
- No Scheduler eligibility or scoring logic appears in feature extraction code
- Adding a new feature to SPEC-010 requires no modification to Scheduler eligibility filtering or scoring logic (only the Scheduler consumption contract in SPEC-003 FR-3 requires update)
- Feature computation code has no dependency on scheduler configuration, backend registry, or solver execution state

---

### FR-2: Feature Extraction Inputs

**Description:**
Core Feature Extraction receives a single input: a validated routing problem in the C++ domain representation (SPEC-001 FR-14). The problem has already passed authoritative domain validation per ADR-009 before feature extraction is invoked.

Feature extraction does not receive:
- Scheduler configuration
- Backend registry
- Solver results
- Any external data source

All features defined in FR-3 are computable exclusively from the raw properties exposed by SPEC-001 FR-10.

**Acceptance Criteria:**
- Feature extraction invocation requires only the C++ routing problem representation
- Feature extraction does not access PostgreSQL, RabbitMQ, or any I/O resource during computation
- Feature extraction does not read or write global state

---

### FR-3: Feature Definitions

**Description:**
Core Feature Extraction computes the following six workload features. For each feature, the definition includes: name, type, valid range, semantic meaning, inputs from SPEC-001, and the computation formula. All formulas are deterministic arithmetic over routing problem properties.

Features FR-3.1 through FR-3.4 are migrated from SPEC-003 FR-3. SPEC-010 is the authoritative owner of their formulas; SPEC-003 FR-3 retains the Scheduler's consumption contract for these features. Features FR-3.5 and FR-3.6 are defined here for the first time.

---

#### FR-3.1: capacity_utilization_ratio

**Type:** float64

**Valid range:** [0.0, 1.0]

**Semantics:** The ratio of total stop demand to total available fleet capacity. Measures how fully the fleet is loaded by this problem.

**Inputs from SPEC-001 FR-10:**
- `stop_demands`: individual demand values per stop
- `vehicle_count`: number of vehicles in the fleet
- `capacity_per_vehicle`: capacity per vehicle (SPEC-001 FR-2)

**Formula:**

```
capacity_utilization_ratio = (Σ demand_i for all stops i) / (vehicle_count × capacity_per_vehicle)
```

**Valid range guarantee:** On any problem that passed SPEC-001 FR-8 capacity feasibility validation, total demand does not exceed total fleet capacity; therefore capacity_utilization_ratio ∈ [0.0, 1.0]. A value above 1.0 on a problem that reached feature extraction indicates an ADR-009 validation divergence. This is caught by FR-6.

**Interpretation:** 0.0 indicates zero demand; 1.0 indicates demand exactly equals fleet capacity.

**Migrated from:** SPEC-003 FR-3 / ODR-1

---

#### FR-3.2: problem_size_class

**Type:** enumeration: `Small` | `Medium` | `Large`

**Semantics:** The routing problem's size tier, derived from stop count using the thresholds defined in SPEC-001 FR-7.

**Inputs from SPEC-001 FR-10:**
- `stop_count`: number of stops in the problem (SPEC-001 FR-7)

**Formula:**

```
Small  if stop_count ∈ [1, 25]
Medium if stop_count ∈ [26, 75]
Large  if stop_count ∈ [76, ∞)
```

**Threshold values:** The threshold values above match the initial values established in SPEC-001 FR-7. These are configurable; a configuration change to the thresholds requires only a configuration update, not a formula change. The implementation must read threshold values from configuration, not from compile-time constants.

**Migrated from:** SPEC-003 FR-3 / SPEC-001 FR-7

---

#### FR-3.3: time_window_density

**Type:** float64

**Valid range:** [0.0, 1.0]

**Semantics:** The fraction of stops in the problem that carry time window constraints.

**Inputs from SPEC-001 FR-10:**
- `stop_time_windows`: per-stop time window fields (open, close) where present (SPEC-001 FR-9)
- `stop_count`: number of stops

**Formula:**

```
time_window_density = (count of stops where both time_window_open and time_window_close are present) / stop_count
```

**Interpretation:** 0.0 = no stops have time windows; 1.0 = all stops have time windows.

**Migrated from:** SPEC-003 FR-3 / ODR-2

---

#### FR-3.4: average_time_window_width_seconds

**Type:** float64

**Valid range:** [0.0, ∞)

**Semantics:** The average width in seconds of time windows across stops that carry time window constraints.

**Inputs from SPEC-001 FR-10:**
- `stop_time_windows`: per-stop time_window_open and time_window_close values (SPEC-001 FR-9)

**Formula:**

Let W = the set of stops where both time_window_open and time_window_close are present.

```
If |W| = 0:
    average_time_window_width_seconds = 0.0

Else:
    average_time_window_width_seconds =
        (Σ (time_window_close_i − time_window_open_i) for i ∈ W) / |W|
```

**Interpretation:** 0.0 when no stop carries a time window. On valid problems (SPEC-001 FR-9 requires time_window_open < time_window_close), this value is strictly positive when |W| > 0.

**Migrated from:** SPEC-003 FR-3 / ODR-3

---

#### FR-3.5: geographic_compactness

**Type:** float64

**Valid range:** [0.0, 1.0]

**Semantics:** Measures how tightly concentrated the stops are relative to each other, expressed as the complement of the ratio of mean pairwise inter-stop distance to maximum pairwise inter-stop distance. 1.0 = all stops co-located; values approaching 0.0 = stops uniformly dispersed across the problem extent.

**Justification:** Geographic dispersion correlates with route total distance. Compact problems — where stops are clustered near each other — tend to exhibit simpler routing structure, and classical baseline solvers may perform adequately. Dispersed problems exhibit more complex routing geometry and may benefit from more sophisticated approaches. Geographic compactness supports backend differentiation at the Scheduler level and enables workload classification for benchmark analysis.

**Inputs from SPEC-001 FR-10:**
- Per-stop latitude and longitude (SPEC-001 FR-4, FR-5)

**Depot scope (ODR-6):** The depot location is excluded from the `geographic_compactness` computation. The feature measures inter-stop spatial concentration. Including the depot would conflate stop distribution with depot placement. Depot-relative routing characteristics may be represented by future workload features if evidence demonstrates scheduling value.

**Formula:**

Let P = the set of all unordered pairs (i, j) with i ≠ j across all stops.

```
If stop_count ≤ 1:
    geographic_compactness = 1.0

Else:
    p_{ij} = Haversine(stop_i, stop_j)  [in km, for all (i, j) ∈ P]
    mean_pairwise = mean(p_{ij} for (i, j) ∈ P)
    max_pairwise  = max(p_{ij} for (i, j) ∈ P)

    If max_pairwise = 0.0:
        geographic_compactness = 1.0   (all stops co-located)
    Else:
        geographic_compactness = 1.0 - (mean_pairwise / max_pairwise)
```

**Valid range guarantee:** mean_pairwise ≤ max_pairwise by definition of mean and max. Therefore mean_pairwise / max_pairwise ∈ [0.0, 1.0], and geographic_compactness ∈ [0.0, 1.0].

**Computational cost:** O(N^2) in stop count, requiring up to N(N−1)/2 Haversine evaluations. For Large-class problems (76+ stops per SPEC-001 FR-7), this is approximately 2,850 evaluations at 76 stops. The Core already computes pairwise Haversine distances for solver use (SPEC-001 FR-5, FR-14); implementations may reuse these distances rather than recomputing them.

**Determinism:** Haversine uses transcendental functions (cos, sin, sqrt, asin). These produce semantically equivalent results under IEEE 754 double precision across conforming C++17 toolchains (ADR-010 Decision 2, Decision 6). Bitwise identical results are not guaranteed and are not required.

**Defined in:** SPEC-010

---

#### FR-3.6: service_time_pressure_ratio

**Type:** float64

**Valid range:** [0.0, 1.0]

**Semantics:** The ratio of total service time at time-windowed stops to total available time window width across those stops. Measures how tightly service time fills the available time window capacity.

**Justification:** When the total service time across time-windowed stops approaches the total available window capacity, vehicles have little temporal slack. High-pressure problems require solvers that reason precisely about timing. This feature enables the Scheduler to prefer time-window-aware backends as pressure increases, and supports benchmark analysis of solver quality under varying temporal tightness.

**Inputs from SPEC-001 FR-10:**
- `stop_time_windows`: per-stop time_window_open and time_window_close (SPEC-001 FR-9)
- `stop_service_durations`: per-stop service_duration values, defaulting to 0 when absent (SPEC-001 FR-16)

**Formula:**

Let W = the set of stops where both time_window_open and time_window_close are present.

```
If |W| = 0:
    service_time_pressure_ratio = 0.0

Else:
    numerator   = Σ service_duration_i for i ∈ W
                  (using 0 when service_duration is absent, per SPEC-001 FR-16)
    denominator = Σ (time_window_close_i − time_window_open_i) for i ∈ W

    service_time_pressure_ratio = min(1.0, numerator / denominator)
```

**Denominator guarantee:** SPEC-001 FR-9 requires time_window_open < time_window_close for every time-windowed stop. Therefore each term in the denominator is strictly positive, and denominator > 0 when |W| > 0.

**Clamping rationale:** SPEC-001 does not reject problems where a stop's service_duration exceeds its time window width. Such problems may be individually infeasible but pass structural validation. Clamping to 1.0 keeps the feature in its declared range. Values near 1.0 indicate high or potential infeasibility-level pressure; the Scheduler treats them as a continuous signal, not a binary feasibility gate.

**Interpretation:** 0.0 = no time-windowed stops or trivial service time (no temporal pressure); 1.0 = service time equals or exceeds total window capacity (maximally constrained or infeasible pressure).

**Defined in:** SPEC-010

---

### FR-4: Feature Output Contract

**Description:**
Core Feature Extraction produces a typed structure: `WorkloadFeatures`. This structure is the complete, immutable output of a single feature extraction invocation. It is passed to the Scheduler as part of the Scheduler invocation (SPEC-003 FR-2).

The `WorkloadFeatures` structure must contain:

| Field | Type | Notes |
|---|---|---|
| `feature_schema_version` | uint32 | The version of the feature definition schema under which these values were computed. Current value: 1. |
| `capacity_utilization_ratio` | float64 | FR-3.1 |
| `problem_size_class` | enum (`Small`, `Medium`, `Large`) | FR-3.2 |
| `time_window_density` | float64 | FR-3.3 |
| `average_time_window_width_seconds` | float64 | FR-3.4 |
| `geographic_compactness` | float64 | FR-3.5 |
| `service_time_pressure_ratio` | float64 | FR-3.6 |

The `WorkloadFeatures` structure is immutable once produced. No component that receives a `WorkloadFeatures` value may modify it.

**Acceptance Criteria:**
- Every feature extraction invocation produces a structurally complete `WorkloadFeatures` value with all fields populated
- `feature_schema_version` is present and non-zero on every invocation
- The Scheduler receives the complete `WorkloadFeatures` structure; it does not request individual features
- No field in the structure is null, optional, or absent on a successful extraction invocation

---

### FR-5: Feature Schema Versioning

**Description:**
`feature_schema_version` is a monotonically increasing integer identifying the feature definition schema version. It is included in the `WorkloadFeatures` output structure and is captured in the `workload_features_snapshot` of the Scheduler decision record (SPEC-003 FR-10).

**Breaking change definition:** The following are breaking changes to the feature schema, requiring an increment to `feature_schema_version`:
- Adding a new feature to the `WorkloadFeatures` structure
- Removing a feature from the `WorkloadFeatures` structure
- Changing the formula for any existing feature
- Changing the valid range or type of any existing feature

**Breaking change procedure:**
1. This specification is updated with the new feature definition(s) and an incremented `feature_schema_version` value.
2. SPEC-003 FR-3 (Scheduler consumption contract) is updated if the Scheduler's input contract changes.
3. The `workload_features_snapshot` schema in SPEC-003 FR-10 is updated.
4. Evidence records produced under different schema versions are not directly comparable on added or changed fields.

**Current version:** 1

**Acceptance Criteria:**
- The `feature_schema_version` field is present in every `WorkloadFeatures` output
- Any breaking schema change results in a `feature_schema_version` increment before the change is released
- The version value is consistently 1 for the feature set defined in this specification

---

### FR-6: Feature Validation

**Description:**
After computing the `WorkloadFeatures` structure, Core Feature Extraction validates every feature value against its declared valid range before returning the structure to the caller.

**Validation rules:**

| Feature | Valid range | Out-of-range condition |
|---|---|---|
| `capacity_utilization_ratio` | [0.0, 1.0] | > 1.0 indicates a SPEC-001 FR-8 / ADR-009 validation divergence |
| `problem_size_class` | One of: `Small`, `Medium`, `Large` | Any other value is a computation error |
| `time_window_density` | [0.0, 1.0] | Any value outside [0.0, 1.0] is a computation error |
| `average_time_window_width_seconds` | [0.0, ∞) | Any negative value is a computation error |
| `geographic_compactness` | [0.0, 1.0] | Any value outside [0.0, 1.0] is a computation error |
| `service_time_pressure_ratio` | [0.0, 1.0] | Any value outside [0.0, 1.0] is a computation error |

If any feature value is outside its declared valid range, Core Feature Extraction returns a structured `FeatureExtractionError` to its caller, identifying the offending feature(s) and their computed values. Core does not return a partially valid `WorkloadFeatures` structure when any feature is out of range.

The Scheduler also performs range validation on receipt (SPEC-003 FR-3, `OutOfRangeFeaturesError`). The validation in feature extraction is the first line of defense; the Scheduler validation is the second. Both must be present.

**Acceptance Criteria:**
- Feature extraction validates all 6 features against their declared ranges before returning
- A `capacity_utilization_ratio` > 1.0 causes a structured extraction error, not a silent clamp
- No out-of-range feature value is passed to the Scheduler

---

### FR-7: Execution Timing and Invocation

**Description:**
Feature extraction occurs exactly once per job execution, at a fixed point in the Core processing sequence.

**Execution sequence within Core:**
1. Core receives the routing problem (C++ domain representation) and the resolved scheduler configuration from the Worker (SPEC-005 FR-5)
2. Core performs authoritative domain validation per ADR-009
3. If validation fails: Core returns a validation rejection to the Worker; feature extraction is not invoked
4. **Core Feature Extraction computes the WorkloadFeatures structure** (this specification)
5. If feature extraction fails: Core returns a FeatureExtractionError to the Worker; Scheduler is not invoked
6. Core invokes the Scheduler with the routing problem, WorkloadFeatures, scheduler configuration, and backend registry (SPEC-003 FR-2)
7. Core returns the Scheduler decision record to the Worker

**Who invokes feature extraction:** Core. The Worker does not invoke feature extraction directly. The Worker invokes Core as a unit; Core internally orchestrates validation, feature extraction, and Scheduler invocation.

**Invocation count:** Feature extraction is invoked exactly once per job execution. Feature values are not recomputed after the `WorkloadFeatures` structure is produced. The Scheduler does not re-invoke feature extraction.

**Caller:** Core is the exclusive caller of feature extraction. No component outside Core invokes feature extraction directly.

**Acceptance Criteria:**
- Feature extraction is never invoked on a problem that failed Core validation
- The Scheduler always receives the `WorkloadFeatures` value produced by the immediately preceding feature extraction invocation for the same job
- Feature values are not cached across job executions
- The Worker does not receive or store the `WorkloadFeatures` structure directly; it receives only the Scheduler decision record, which includes the `workload_features_snapshot` (SPEC-003 FR-10)

---

### FR-8: Reproducibility Requirements

**Description:**
Feature extraction must be deterministic. Given the same routing problem input, the same feature values must be produced on every invocation, regardless of execution environment, scheduler configuration, or selected backend.

**Determinism requirements:**
- Feature computation uses only inputs from the routing problem. No external state, configuration, time-of-day, process ID, or random source is used.
- Floating-point arithmetic in feature computation uses IEEE 754 double precision. Extended precision is prohibited per ADR-010 Decision 6.
- Haversine distance computations (used in FR-3.5 geographic_compactness) produce semantically equivalent results across conforming C++17 toolchains (ADR-010 Decision 2). Bitwise identical results for transcendental function outputs are not required and not guaranteed.

**Stochastic computation:** Feature extraction does not use any PRNG or stochastic algorithm. ADR-010's PRNG and distribution sampling requirements do not apply to feature extraction, which is purely deterministic arithmetic. The prohibitions in ADR-010 Decision 4 on prohibited entropy sources apply: no time, process ID, or OS random source may influence feature computation.

**Reproducibility verification:** Given a `job_id`, an operator can retrieve the `workload_features_snapshot` from the decision record (via SPEC-006 FR-5) and the routing problem (via SPEC-001 FR-12, referenced by `problem_id`). Re-running feature extraction on the same routing problem must produce the same 6 values and the same `feature_schema_version`.

**Acceptance Criteria:**
- Given the same routing problem, every feature extraction invocation produces identical feature values
- Feature extraction does not read global state, environment variables, or any external source during computation
- Floating-point computations that use transcendental functions (Haversine in FR-3.5) produce semantically equivalent results as defined by ADR-010 Decision 2

---

### FR-9: Scheduler Integration

**Description:**
Core Feature Extraction produces the `WorkloadFeatures` structure that the Scheduler consumes as an input. The ownership boundary is strict.

**Core Feature Extraction owns:**
- All feature computation formulas (FR-3)
- The `WorkloadFeatures` output contract (FR-4)
- Schema versioning (FR-5)
- Feature validation before Scheduler invocation (FR-6)

**SPEC-003 owns:**
- The Scheduler's consumption contract: which feature names, types, and valid ranges the Scheduler requires
- The Scheduler's eligibility filtering logic that uses feature values (SPEC-003 FR-5)
- The Scheduler's scoring logic that uses feature values (SPEC-003 FR-7)
- The `workload_features_snapshot` field in the Scheduler decision record (SPEC-003 FR-10)

**Integration flow:**
1. Core Feature Extraction produces a validated `WorkloadFeatures` structure
2. Core passes this structure to the Scheduler as part of the Scheduler invocation (SPEC-003 FR-2)
3. The Scheduler treats the received values as authoritative; it does not modify or recompute them

**Scheduler use of new features:** SPEC-003 FR-5 Phase 1 eligibility filtering references three features from the `WorkloadFeatures` input: `problem_size_class` (SizeClassMismatch condition), `time_window_density` (TimeWindowUnsupported condition), and `capacity_utilization_ratio` (CapacityConstraintUnsupported condition). The SPEC-003 FR-7 scoring formula is deferred to SPEC-003 OQ-4 and does not constitute a current accepted policy reference. All six features are available to the Scheduler through the `WorkloadFeatures` contract (FR-4). The two features introduced in SPEC-010 — `geographic_compactness` and `service_time_pressure_ratio` — are not currently referenced in any accepted SPEC-003 FR-5 eligibility condition. They are captured in the `workload_features_snapshot` of the decision record.

**Required follow-on update to SPEC-003:** The `workload_features_snapshot` field in SPEC-003 FR-10 must be updated to include `geographic_compactness` and `service_time_pressure_ratio`. The Scheduler must receive a `WorkloadFeatures` structure containing all six fields. SPEC-003 FR-3 must be updated to reference SPEC-010 as the authoritative owner of feature computation formulas. These updates are required before SPEC-010 is implemented but do not modify SPEC-003's eligibility or scoring requirements.

**Acceptance Criteria:**
- The Scheduler receives the complete `WorkloadFeatures` structure, including all six features
- The Scheduler does not recompute any feature value
- The Scheduler does not call any feature extraction function
- Core does not invoke the Scheduler before feature extraction is complete and validated

---

### FR-10: Evidence Integration

**Description:**
Feature values are not persisted as a standalone evidence artifact. They are persisted as part of the Scheduler decision record via the `workload_features_snapshot` field (SPEC-003 FR-10). The `workload_features_snapshot` is written to the Evidence Log as part of the decision record (SPEC-006 FR-5).

**Justification for no standalone persistence:**
- Feature values are derived deterministically from the routing problem. They are always recoverable by re-running feature extraction on the problem identified by `problem_id`.
- The `workload_features_snapshot` in the decision record already preserves feature values at decision time for auditability, benchmark analysis, and scheduler effectiveness analysis.
- A separate feature evidence table would duplicate data already in the decision record without providing additional auditability.
- The `feature_schema_version` in the decision record's snapshot enables detection of formula changes that might affect comparability of feature values across jobs.

**What the decision record snapshot must capture:**
The `workload_features_snapshot` must include all six features defined in FR-3, plus the `feature_schema_version`. This is a required follow-on update to SPEC-003 FR-10 (see FR-9 and Documentation Updates Required).

**Benchmark and analysis support:**
Feature values in decision records enable:
- Workload classification: grouping jobs by problem characteristics (size, density, compactness, pressure)
- Backend selection analysis: correlating feature values with which backend was selected and why
- Scheduler effectiveness analysis: correlating feature values with regret outcomes and hindsight quality

These analyses are performed on persisted decision records via Evidence Log retrieval (SPEC-006 FR-14). SPEC-010 is responsible for ensuring feature values are correctly computed and represented in the snapshot; benchmark harness design is outside SPEC-010's scope.

**SPEC-006 participation:** Feature extraction participates in SPEC-006 indirectly, through the decision record. No schema change to SPEC-006 is required by SPEC-010; the decision record schema change (adding `geographic_compactness`, `service_time_pressure_ratio`, and `feature_schema_version` to `workload_features_snapshot`) is a SPEC-003 revision. SPEC-006 FR-5 adopts the SPEC-003 FR-10 decision record schema in full.

**Acceptance Criteria:**
- Feature values for a job are retrievable from the Evidence Log via the decision record for that job
- The `workload_features_snapshot` in the decision record includes all six features and the `feature_schema_version`
- No standalone feature evidence table or artifact is created

---

### FR-11: Telemetry

**Description:**
Core Feature Extraction must emit the `features.extract` OpenTelemetry span on every invocation. This span is defined in the architecture.md required span list and is Core's telemetry obligation (SPEC-005 FR-19).

**Span: `features.extract`**
- **Owner:** Core (SPEC-005 FR-19 confirms Core ownership; the Worker does not emit this span)
- **Parent:** `job.consume` (via in-process span context propagation within the Worker process; ADR-011)
- **Scope:** Begins before any feature computation; ends after feature validation completes (whether successful or failed)

**Required span attributes:**

| Attribute | Type | Description |
|---|---|---|
| `job_id` | string | Identifies the job |
| `problem_id` | string | Identifies the routing problem |
| `feature_schema_version` | uint32 | Schema version under which features were computed |
| `problem_size_class` | string | `Small`, `Medium`, or `Large` |
| `stop_count` | int | Number of stops in the problem |
| `extraction_status` | string | `Success` or `Failure` |

**Span status rules:**
- Success: `OK`
- FeatureExtractionError (out-of-range feature value): `Error`

**Structured log events:**

| Event name | When | Required fields |
|---|---|---|
| `core.features.extracted` | Successful extraction | `job_id`, `problem_id`, `feature_schema_version`, `problem_size_class`, `stop_count` |
| `core.features.extraction.failed` | FeatureExtractionError | `job_id`, `problem_id`, `offending_features` (list of feature names with out-of-range values) |

**Acceptance Criteria:**
- `features.extract` is emitted on every feature extraction invocation, successful or failed
- All required attributes are present on every emission
- On `FeatureExtractionError`, the span status is `Error` and the `core.features.extraction.failed` event is emitted
- The span is correlatable with the scheduling decision via `job_id`

---

# Non-Requirements

- Core Feature Extraction does not invoke the Scheduler, execute solvers, or evaluate solution quality.
- Core Feature Extraction does not access PostgreSQL, RabbitMQ, or any external system during computation.
- Core Feature Extraction does not define scheduling policy or optimization objectives.
- Core Feature Extraction does not generate routing problems (SPEC-002).
- Core Feature Extraction does not perform routing problem validation (ADR-009).
- Core Feature Extraction does not define the Scheduler's consumption contract (SPEC-003 FR-3).
- Core Feature Extraction does not define the Evidence Log schema (SPEC-006).
- Core Feature Extraction does not define the benchmark harness or benchmark analysis methodology.
- Core Feature Extraction does not expose a network API or remote invocation interface; it is an in-process component within Daedalus Core (ADR-001).
- Feature caching across job executions is not required and not permitted; features are always computed fresh from the routing problem.
- No machine learning or learned feature computation is in scope. All features are deterministic functions of problem properties.

---

# Assumptions

1. The routing problem passed to feature extraction has already passed authoritative Core validation per ADR-009. Feature extraction is only invoked on valid problems; it does not re-validate domain rules.
2. The SPEC-001 FR-10 raw properties (`stop_count`, `vehicle_count`, `capacity_per_vehicle`, `stop_demands`, `stop_time_windows`, `stop_service_durations`, and per-stop coordinates) are accessible from the C++ domain representation at the time feature extraction is invoked.
3. Problem size classification threshold values (Small: 1–25, Medium: 26–75, Large: 76+) are accessible at feature extraction time without requiring a database lookup; they are resolved from configuration before Core invocation begins.
4. The Core already computes a pairwise Haversine distance matrix for solver use (SPEC-001 FR-5, FR-14). Implementations may reuse these distances in the geographic_compactness computation (FR-3.5) rather than recomputing them. This is a performance optimization, not a behavioral requirement.
5. MVP stop counts are bounded by the Large-class upper range used in practice (several hundred stops at most). The O(N^2) pairwise computation in FR-3.5 is acceptable at MVP scale.

---

# Constraints

1. Feature extraction must be implemented in C++ within Daedalus Core (ADR-001).
2. Feature extraction must not use any PRNG, OS random source, time source, or process-local state during computation (ADR-010 Decision 4, applied to non-stochastic code: prohibited entropy sources must not influence feature values).
3. Floating-point arithmetic in feature computation must use IEEE 754 double precision. Extended precision is prohibited (ADR-010 Decision 6).
4. All six features must be computed and validated before the Scheduler is invoked. The Scheduler must not be invoked if feature extraction fails.
5. Feature extraction must not depend on the scheduler configuration, the backend registry, or solver execution state.
6. `feature_schema_version` must be incremented for any breaking change to the feature schema (FR-5). A breaking change without a version increment is a specification violation.

---

# Inputs

| Input | Source | Format |
|---|---|---|
| Routing problem (C++ domain representation) | Core, from PostgreSQL via Worker (SPEC-001 FR-14, SPEC-005 FR-4) | C++ struct containing all SPEC-001 FR-10 raw properties |

No other inputs are accepted or required.

---

# Outputs

| Output | Consumer | Format |
|---|---|---|
| `WorkloadFeatures` structure | Core (passed to Scheduler; captured in decision record snapshot) | Typed C++ struct per FR-4 |
| `features.extract` OTel span | OpenTelemetry Collector | Span per FR-11 |
| `core.features.extracted` or `core.features.extraction.failed` log event | Stdout (JSON) | Structured log per FR-11 |

On failure, Core returns a `FeatureExtractionError` to the Worker (instead of a `WorkloadFeatures` value). The Worker handles this as a structured failure (see Failure Modes and SPEC-005 FR-5).

---

# Failure Modes

### FeatureExtractionError

**Condition:** One or more computed feature values are outside their declared valid range (FR-6).

**Behavior:** Core Feature Extraction returns a structured `FeatureExtractionError` to Core. Core does not invoke the Scheduler. Core propagates the failure to the Worker. Worker handling is governed by SPEC-005 FR-5.

**Fallback:** None at the Core Feature Extraction level. An out-of-range feature value is a Core internal consistency violation, not a recoverable condition at runtime. Investigation is required to identify the source of the invalid value.

**Core-observable result:** The `features.extract` span status is `Error` and the `core.features.extraction.failed` log event is emitted with the offending feature names and their computed values (FR-11).

---

### IncompleteProblemRepresentation

**Condition:** A required raw property from SPEC-001 FR-10 is inaccessible from the C++ domain representation (for example, the stop list is absent, or the property is in an unexpected state).

**Behavior:** Core Feature Extraction returns a structured `IncompleteProblemRepresentation` error before any feature computation begins. The Scheduler is not invoked. Core propagates the failure to the Worker. Worker handling is governed by SPEC-005 FR-5.

**Fallback:** None at the Core Feature Extraction level. This condition indicates a Core validation or representation construction error. Investigation is required.

**Core-observable result:** The `features.extract` span status is `Error` and the `core.features.extraction.failed` log event is emitted.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer (Daedalus Core, C++) | Yes — new authoritative component: Core Feature Extraction; feature computation moves from an implicit Core behavior to an explicitly specified component |
| API Layer | No — no new API surface; feature extraction is an internal Core concern |
| Persistence | No direct impact — features persist via the decision record snapshot (SPEC-003 FR-10, SPEC-006 FR-5); no new evidence tables |
| Scheduler (SPEC-003) | Yes — consumption contract must be updated to include `geographic_compactness` and `service_time_pressure_ratio`; formula definitions migrate to SPEC-010 |
| Solver Runtime | No |
| Observability | Yes — `features.extract` span definition is now owned by SPEC-010 (FR-11) |
| Configuration | Indirect — size class thresholds (used by `problem_size_class`) remain configurable per SPEC-001 FR-7; no new configuration is required |
| Evidence Log (SPEC-006) | No schema change required; `workload_features_snapshot` extension is a SPEC-003 FR-10 revision |

---

# Testability

### Unit Tests

1. **Capacity utilization ratio — zero demand:** A routing problem where all stop demands are 0 produces `capacity_utilization_ratio = 0.0`.

2. **Capacity utilization ratio — full capacity:** A routing problem where total demand exactly equals total fleet capacity produces `capacity_utilization_ratio = 1.0`.

3. **Problem size class — boundary values:** Problems at stop_count 25, 26, 75, and 76 produce `Small`, `Medium`, `Medium`, and `Large` respectively.

4. **Time window density — no windows:** A problem with no time-windowed stops produces `time_window_density = 0.0`.

5. **Time window density — all windows:** A problem where every stop carries a time window produces `time_window_density = 1.0`.

6. **Average time window width — no windows:** A problem with no time-windowed stops produces `average_time_window_width_seconds = 0.0`.

7. **Average time window width — single windowed stop:** A problem with one stop with `time_window_open = 100` and `time_window_close = 700` produces `average_time_window_width_seconds = 600.0`.

8. **Geographic compactness — all co-located:** A problem where all stops have the same coordinates produces `geographic_compactness = 1.0`.

9. **Geographic compactness — two stops:** Given two stops at known coordinates, verify that `geographic_compactness = 0.0` (single pair, mean equals max).

10. **Geographic compactness — single stop:** A problem with one stop produces `geographic_compactness = 1.0`.

11. **Geographic compactness — range:** For any valid routing problem, `geographic_compactness ∈ [0.0, 1.0]`.

12. **Service time pressure — no windowed stops:** A problem with no time-windowed stops produces `service_time_pressure_ratio = 0.0`.

13. **Service time pressure — zero service duration:** A problem with windowed stops and all `service_duration = 0` produces `service_time_pressure_ratio = 0.0`.

14. **Service time pressure — service fills windows exactly:** A problem where the sum of `service_duration` values across windowed stops equals the sum of `(window_close - window_open)` values produces `service_time_pressure_ratio = 1.0`.

15. **Service time pressure — clamping:** A problem where the numerator exceeds the denominator produces `service_time_pressure_ratio = 1.0` (clamped; not > 1.0).

16. **Schema version present:** Every `WorkloadFeatures` output contains `feature_schema_version = 1`.

### Reproducibility Tests

17. **Determinism — identical inputs:** Given the same routing problem, two sequential invocations of feature extraction produce bit-identical values for all integer-valued and enumeration-valued features, and semantically equivalent values (ADR-010 Decision 2) for all floating-point features.

18. **Determinism — no external state dependency:** Given the same routing problem, feature extraction produces the same values whether invoked during the first Worker execution or during idempotent re-execution (SPEC-005 FR-14).

### Validation Tests

19. **Out-of-range detection:** Inject a mock routing problem that produces a `capacity_utilization_ratio` value above 1.0 (simulating an ADR-009 divergence). Verify that feature extraction returns a structured `FeatureExtractionError` identifying the offending feature, and that the Scheduler is not invoked.

### Observability Tests

20. **Span emission — success:** A successful feature extraction produces a `features.extract` span with `extraction_status = Success` and all required attributes present.

21. **Span emission — failure:** A `FeatureExtractionError` produces a `features.extract` span with `extraction_status = Failure` and span status `Error`.

### Integration Tests

22. **Full lifecycle — features in decision record:** A complete job execution produces a decision record whose `workload_features_snapshot` contains non-null values for all six features and a non-zero `feature_schema_version`.

23. **Retrieval — features from Evidence Log:** Given a `job_id`, the `workload_features_snapshot` is retrievable from the Evidence Log via the decision record and contains all six feature values.

24. **Unit: IncompleteProblemRepresentation:** Inject a mock routing problem where a required SPEC-001 FR-10 raw property (for example, the stop list is null or absent from the C++ domain representation). Verify that Core Feature Extraction returns a structured `IncompleteProblemRepresentation` error before any feature computation begins. Verify that no `WorkloadFeatures` value is produced. Verify that the Scheduler is not invoked.

---

# Observability Requirements

Operational questions that the `features.extract` span and log events must answer:

1. For a given job, what feature values drove the Scheduler's decision?
2. Did feature extraction succeed or fail for a given job?
3. What was the problem size class for a given job?
4. What feature schema version was in effect for a given execution?
5. Are there any systematic feature extraction failures for a class of problems?

These are answered by:
- The `features.extract` span (extraction success/failure, problem size class, schema version)
- The `workload_features_snapshot` in the persisted decision record (all six feature values)
- Structured log events at extraction completion or failure

No additional metrics are required for feature extraction at MVP scope. Feature distribution analysis is best performed against persisted decision records rather than ephemeral metric counters.

---

# Security Considerations

Feature extraction operates entirely within the Daedalus Core C++ library, on data that has already been validated by both the API and Core validation layers (ADR-009). No new trust boundaries are introduced.

Feature values are derived from the routing problem's numeric and geographic data. The routing problem contains synthetic coordinates at MVP scope (SPEC-001 Assumptions). No personally identifiable information is expected in feature computations.

Feature values appear in the `workload_features_snapshot` of the decision record, which is persisted to PostgreSQL and included in evidence reports. This is consistent with the existing evidence handling model. No new data exposure risk is introduced.

The `features.extract` log event must not reproduce raw coordinate arrays or full stop lists, consistent with SPEC-001 Security Considerations.

---

# Performance Considerations

**FR-3.5 geographic_compactness (O(N^2)):** For Large-class problems, pairwise Haversine computation requires up to N(N−1)/2 evaluations. At 100 stops, this is 4,950 evaluations. Given that the Core already builds a pairwise distance matrix for solver use (SPEC-001 FR-5), implementations should reuse these distances to avoid redundant computation. The O(N^2) cost must be measured and reported as part of the `features.extract` span duration.

**Overall extraction cost:** Feature extraction is dominated by the geographic_compactness computation at large stop counts. All other features are O(N). The total extraction cost must be measured and confirmed to be small relative to solver execution time. If extraction latency becomes material, the pairwise distance reuse optimization (Assumption 4) is the primary mitigation.

**Memory:** For N stops, the pairwise distance set has N(N−1)/2 values. At 100 stops and 8 bytes per float64, this is approximately 40 KB of intermediate values. This is acceptable at MVP scale.

Do not establish numeric latency targets at this stage. Measure during implementation.

---

# Documentation Updates Required

The following updates are required before SPEC-010 can be implemented. SPEC-010 does not modify these documents; it documents the required follow-on changes.

**SPEC-003 FR-3 (Scheduler Objectives and Policy Engine):**
- Remove the introductory scope note that places formula ownership in SPEC-003 temporarily. Add a reference to SPEC-010 as the authoritative owner of all feature computation formulas and algorithms.
- Retain the Scheduler's consumption contract: the four feature names, types, valid ranges, and semantics that the Scheduler currently uses. Update this list to also include `geographic_compactness` (float64, [0.0, 1.0]) and `service_time_pressure_ratio` (float64, [0.0, 1.0]) as features present in the `WorkloadFeatures` input.
- Note that `geographic_compactness` and `service_time_pressure_ratio` are received by the Scheduler but not currently referenced in eligibility filtering (FR-5) or scoring logic (FR-7).

**SPEC-003 FR-10 (Decision Record — `workload_features_snapshot`):**
- Update the `workload_features_snapshot` field definition to include all six features and `feature_schema_version`.

**SPEC-001 FR-10 (Raw Problem Properties):**
- Add a reference to SPEC-010 as the specification that defines how the listed raw properties are used to compute workload features.

**docs/architecture.md (Daedalus Core responsibilities):**
- Update the Core responsibilities list to explicitly include "workload feature extraction per SPEC-010" alongside the existing "workload feature extraction" entry.

**SPEC-005 FR-5 (Worker Execution Lifecycle — Core invocation):**
- Must be updated to define Worker handling of two failure types returned by Core Feature Extraction: `FeatureExtractionError` (one or more feature values outside declared valid range) and `IncompleteProblemRepresentation` (a required SPEC-001 FR-10 raw property inaccessible from the C++ domain representation). For each failure type, SPEC-005 FR-5 must specify: the failure stage classification, the evidence record population behavior, and whether the job is dead-lettered. SPEC-010 owns Core Feature Extraction's contract (what Core returns); SPEC-005 owns Worker handling.
- **Terminology alignment (A-001):** SPEC-005 FR-5 currently uses `OutOfRangeFeaturesError` and `IncompleteFeaturesError` in its Core response table. These names must be replaced with `FeatureExtractionError` and `IncompleteProblemRepresentation` respectively to match the authoritative names defined in SPEC-010 FR-6 and FR-7. This is a contract alignment update only. No behavior changes to SPEC-005 FR-5 are required.
- **Failure artifact clarification (A-002):** `FeatureExtractionError` and `IncompleteProblemRepresentation` occur before Scheduler invocation (SPEC-010 FR-7, steps 4–5). These failure paths do not produce a Scheduler decision record. SPEC-005 FR-5 must be updated to reflect that the Worker persists a structured failure record — not a decision record — for these conditions. The existing SPEC-005 FR-5 language "persists the decision record" applies only to failure types where the Scheduler ran and produced a decision record (such as `NoEligibleSolver` and `InvalidConfiguration`). It does not apply to feature extraction failure paths where Scheduler invocation did not occur.

**SPEC-005 FR-13 (Worker Execution Lifecycle — Retry and Dead-Letter Behavior):**
- Must be updated to add `FeatureExtractionError` and `IncompleteProblemRepresentation` as named permanent failure conditions in the permanent failures table. For each condition, SPEC-005 FR-13 must define: the handling behavior (including dead-letter determination) and the evidence artifact the Worker persists. These are permanent failures because re-executing the same job with the same routing problem will produce the same failure. SPEC-010 owns the definitions of these failure types (what they mean and when they occur); SPEC-005 FR-13 owns the Worker's handling obligations and evidence artifact obligations for those types.

---

# Open Questions

### OQ-1: geographic_compactness — Depot Inclusion in Spatial Analysis — RESOLVED (ODR-6)

**Decision:** The depot location is excluded from the `geographic_compactness` computation. The feature uses customer stop coordinates only. The formula in FR-3.5 is correct as defined; no formula change is required.

**Rationale:** `geographic_compactness` is intended to measure workload spatial concentration — the degree to which customer stops are clustered relative to each other. Including the depot would conflate stop distribution with depot placement, changing what the feature measures without improving its utility as a workload characterization signal.

**Future consideration:** Depot-relative routing characteristics may be represented by future workload features if evidence demonstrates scheduling value. Any such addition would be a breaking change requiring a `feature_schema_version` increment (FR-5).

**Incorporated in:** FR-3.5 (depot scope made explicit via ODR-6 reference; formula and inputs unchanged).

---

### OQ-2: service_time_pressure_ratio — Scope of Pressure Measurement

**Question:** The current definition (FR-3.6) computes pressure as the ratio of total service time to total window width across all time-windowed stops. An alternative is to compute the maximum per-stop pressure ratio (max over stops of service_duration / window_width) rather than the aggregate. The per-stop maximum may be more informative for identifying individual infeasible stops.

**Why it matters:** Aggregate pressure can mask a single infeasible stop surrounded by many unconstrained stops. Per-stop maximum would identify tighter constraint interactions. However, the aggregate formulation is consistent with how other aggregate features (capacity_utilization_ratio, time_window_density) are defined.

**Classification:** Implementation planning. The aggregate definition is acceptable for MVP; this question is for future consideration.

**Blocking:** Not blocking.

---

### OQ-3: feature_schema_version in features.extract Span

**Question:** Should the `feature_schema_version` attribute on the `features.extract` span be sufficient for detection of version mismatches in monitoring, or should an additional metric counter tracking version distribution be added?

**Classification:** Implementation planning. The span attribute satisfies the observability requirement for individual jobs. Aggregate version distribution can be observed by querying persisted decision records; a dedicated metric is not necessary at MVP scope.

**Blocking:** Not blocking.

---

# Acceptance Checklist

- [x] Problem is clearly defined
- [x] Domain concept is defined
- [x] Responsibility scope and component boundaries are defined (FR-1)
- [x] Feature extraction inputs are defined (FR-2)
- [x] All six features are defined with formulas, types, valid ranges, and semantics (FR-3)
- [x] Feature output contract (WorkloadFeatures structure) is defined (FR-4)
- [x] Feature schema versioning is defined (FR-5)
- [x] Feature validation requirements are defined (FR-6)
- [x] Execution timing and invocation are defined (FR-7)
- [x] Reproducibility requirements are defined (FR-8)
- [x] Scheduler integration contract is defined (FR-9)
- [x] Evidence integration decision is made and justified (FR-10)
- [x] Telemetry requirements are defined (FR-11)
- [x] Non-requirements are documented
- [x] Assumptions are explicit
- [x] Failure modes are defined
- [x] Observability requirements exist
- [x] Security considerations exist
- [x] Performance considerations are identified
- [x] Documentation updates are identified
- [x] OQ-1 resolved — depot inclusion in geographic_compactness: depot excluded; customer stops only (ODR-6)
- [ ] OQ-2 resolved — service_time_pressure_ratio scope (implementation planning; non-blocking)
- [ ] OQ-3 resolved — feature_schema_version metric coverage (implementation planning; non-blocking)

---

# Definition of Done

This feature is complete when:

- OQ-1 (geographic_compactness depot inclusion) is resolved: depot excluded, customer stops only (ODR-6); no formula change required
- All functional requirements (FR-1 through FR-11) are implemented and acceptance criteria pass
- All six features produce correct values verified against the formulas in FR-3 on representative test problems
- All unit tests defined in the Testability section pass
- All reproducibility tests pass: identical inputs produce identical outputs across two sequential invocations
- The `features.extract` OTel span is emitted on every invocation (successful and failed) with all required attributes
- The `workload_features_snapshot` in persisted decision records contains all six feature values and `feature_schema_version`
- SPEC-003 FR-3 and FR-10 have been updated to reference SPEC-010 and include the two new features
- SPEC-001 FR-10 has been updated to reference SPEC-010
- Engineering review passes
- Specification status is updated to Verified
