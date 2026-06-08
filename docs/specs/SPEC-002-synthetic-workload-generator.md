# Feature Specification

## Metadata

**Feature ID:** SPEC-002

**Title:** Synthetic Workload Generator

**Status:** Draft

**Author:** Darkhorse286

**Created:** 2026-06-08

**Last Updated:** 2026-06-08

**Related ADRs:** ADR-001, ADR-004, ADR-006, ADR-009

**Related Specs:** SPEC-001, SPEC-003 (pending)

---

# Problem Statement

Project DAEDALUS must evaluate its scheduler and solver backends against reproducible, parameterized routing scenarios. Without a synthetic workload generator:

- Scheduler evaluation requires manually authored routing problems, which are time-consuming and cannot be varied systematically across size classes, constraint structures, and difficulty levels.
- Solver benchmarking cannot produce comparative results across a consistent, reproducible problem distribution.
- The project thesis — that most routing problems should not use expensive backends — cannot be demonstrated across a controlled range of problem structures without a reproducible problem corpus.

The system currently has no automated mechanism for producing routing problems that conform to the SPEC-001 routing problem model. Every generated scenario must be traceable to its seed and parameters so that evidence reports can describe exactly what was solved and how it was produced.

---

# Business Value

The generator enables:

- Systematic scheduler evaluation across parameterized routing scenarios spanning small, medium, and large size classes with and without time windows.
- Reproducible solver benchmarking with measurable, comparable inputs that can be regenerated at any time.
- Demonstration of the project thesis using a controlled and varied problem corpus rather than ad hoc inputs.
- Developer workflow acceleration: a single command produces a ready-to-submit routing problem with known structural properties.

---

# Employer Signaling

- System Design
- Simulation Engineering
- Performance Engineering
- Optimization

---

# Scope and Responsibility Boundary

SPEC-002 defines the synthetic workload generator: its inputs, outputs, generation strategies, and reproducibility guarantees. It does not define solver behavior, scheduler behavior, or workload feature computation.

## Responsibility Boundary

| Responsibility | Owner |
| --- | --- |
| Stop model, depot model, fleet model, time window model, service duration model, coordinate representation | SPEC-001 |
| Validation rules for routing problem documents | SPEC-001 |
| Validation authority (who validates and when) | ADR-009 |
| Workload feature computation (capacity utilization ratio, time window density, average time window width, problem size class) | Daedalus Core — computation contract defined in SPEC-003 |
| Solver eligibility and backend selection | Daedalus Scheduler — defined in SPEC-003 |
| Producing conforming routing problem instances from parameterized generation configurations | **SPEC-002 (this spec)** |
| Assigning nominal difficulty tiers to generated problems based on generation parameters | **SPEC-002 (this spec)** |
| Producing a generation manifest that describes what was generated and why | **SPEC-002 (this spec)** |

The generator produces raw problem properties. It does not compute derived workload features. It does not select solvers. It does not define routing problem validation rules.

---

# Requirements

## FR-1: Scenario Taxonomy

**Description:** The generator supports a defined taxonomy of scenario types. A scenario type is a named generation intent that sets default values for generation parameters and documents the structural purpose of the scenario. Scenario types exist to give developers and test harnesses a shared vocabulary for describing problem structure without requiring every parameter to be specified explicitly.

The scenario type governs the default time_window_density and the default size class guidance. All other parameters remain individually configurable regardless of scenario type.

The initial scenario taxonomy is:

| Scenario Type | Description | Default time_window_density | Default Size Guidance |
| --- | --- | --- | --- |
| `capacity-only` | Pure capacity VRP. No time windows on any stop. Baseline scenario for classical solver evaluation. | 0.0 | Small |
| `partial-time-windowed` | Capacity VRP with time windows on roughly half the stops. Tests scheduler handling of mixed constraint problems. | 0.5 | Medium |
| `dense-time-windowed` | Capacity VRP with time windows on nearly all stops. High constraint density for solver stress testing. | 1.0 | Medium |
| `stress` | Large problem at or near the upper boundary of the Large size class. May combine time windows with high stop counts to stress classical solver performance. | 0.5 | Large |

Scenario types are defaults, not rigid templates. A caller may override any parameter while specifying a scenario type. The override takes precedence over the scenario type default.

**Acceptance Criteria:**

- The generator recognizes each of the four scenario types by name.
- Specifying a scenario type without additional parameter overrides produces a problem whose structure matches the scenario type defaults.
- A caller may override the time_window_density default by providing an explicit value in the generation configuration.
- An unrecognized scenario type name is rejected with a structured error before generation begins.

---

## FR-2: Generation Configuration Contract

**Description:** Generator behavior is fully determined by a generation configuration. A generation configuration is the complete set of input parameters required to produce a routing problem. Given the same generation configuration (including seed), the generator always produces the same routing problem document.

The generation configuration is the caller's contract with the generator.

**Required Parameters**

| Parameter | Type | Description |
| --- | --- | --- |
| seed | uint64 | Random seed. Passed through unchanged as the routing problem seed (SPEC-001 FR-6). |
| scenario_type | enum | One of the four scenario types defined in FR-1. |
| stop_count | positive integer | Number of stops in the generated problem. |
| vehicle_count | positive integer | Number of vehicles in the generated fleet. |
| capacity_per_vehicle | positive integer | Capacity per vehicle in demand units. |

**Optional Parameters (with defaults)**

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| demand_min | non-negative integer | 1 | Minimum per-stop demand value (inclusive). |
| demand_max | positive integer | capacity_per_vehicle | Maximum per-stop demand value (inclusive). Must be ≥ demand_min. |
| time_window_density | double [0.0, 1.0] | Set by scenario_type | Fraction of stops that receive time windows. |
| time_window_width_seconds | positive integer | 3600 | Width of each generated time window in seconds. Controls tightness relative to planning horizon. |
| planning_horizon_seconds | positive integer | 28800 | Total route time horizon in seconds (t=0 = route start, per SPEC-001 FR-9). Time window values are bounded within [0, planning_horizon_seconds]. |
| service_duration_seconds | non-negative integer | 0 | Service duration applied uniformly to all stops (SPEC-001 FR-16). |
| geographic_distribution | enum | `uniform` | Stop coordinate distribution. Valid values: `uniform`, `clustered`. |
| cluster_count | positive integer | absent | Number of geographic clusters. Required when geographic_distribution is `clustered`. |
| cluster_std_dev_degrees | positive double | 0.05 | Standard deviation of stop coordinates around cluster centers, in degrees. Used only when geographic_distribution is `clustered`. |
| bounding_box | lat/lon bounds | See FR-3 | Geographic region for coordinate generation. |

**Structural Validation Rules**

The following configuration errors must be rejected before generation begins:

- demand_max < demand_min.
- time_window_width_seconds ≥ planning_horizon_seconds (produces degenerate windows spanning the full horizon).
- geographic_distribution = `clustered` with cluster_count absent.
- cluster_count present with geographic_distribution ≠ `clustered`.
- cluster_count > stop_count.
- vehicle_count = 0.
- capacity_per_vehicle = 0.
- stop_count = 0.

**Acceptance Criteria:**

- A configuration satisfying all structural validation rules proceeds to generation.
- A configuration violating any structural validation rule is rejected with a structured error identifying the violated rule before any generation step runs.
- Each optional parameter applies its documented default when absent.
- A configuration with scenario_type absent is rejected.

---

## FR-3: Geographic Generation Strategy

**Description:** The generator produces geographic coordinates for the depot and all stops. All coordinates must conform to SPEC-001 FR-5: latitude in [-90.0, 90.0], longitude in [-180.0, 180.0], represented as IEEE 754 double precision.

**Bounding Box**

All stop and depot coordinates are generated within a configurable bounding box. The bounding box is specified as (min_lat, max_lat, min_lon, max_lon).

The default bounding box covers an area of approximately 200 km × 200 km at mid-latitude. The default reference center (center_lat, center_lon) is a configurable value, not a constant in generation logic. The default reference center has no real-world geographic significance.

Default bounding box (derived from configurable center):

```
min_lat = center_lat - 0.9
max_lat = center_lat + 0.9
min_lon = center_lon - 1.1
max_lon = center_lon + 1.1
```

The degree offsets above approximate 200 km at a mid-latitude reference. The specific degree values are configuration constants.

**Depot Placement**

The depot is placed at the center of the bounding box:

```
depot_lat = (min_lat + max_lat) / 2
depot_lon = (min_lon + max_lon) / 2
```

The depot location is deterministic given the bounding box. It is not generated from the PRNG. The depot location does not consume any PRNG draws.

**Stop Coordinate Generation: Uniform Distribution**

When geographic_distribution = `uniform`, each stop's latitude and longitude are drawn independently from uniform distributions over the bounding box extents. Latitude draws and longitude draws are independent.

**Stop Coordinate Generation: Clustered Distribution**

When geographic_distribution = `clustered`:

1. cluster_count cluster centers are drawn uniformly and independently from the bounding box interior. Cluster centers consume PRNG draws before stop coordinates.
2. Each stop is assigned to a cluster center in round-robin order (stop 0 → cluster 0, stop 1 → cluster 1, ..., stop k → cluster k mod cluster_count). Assignment order is deterministic given stop index.
3. Each stop's latitude and longitude are drawn from independent normal distributions centered at its assigned cluster center, with standard deviation cluster_std_dev_degrees.
4. If a drawn coordinate falls outside the bounding box, it is clamped to the nearest bounding box boundary. Clamping does not consume additional PRNG draws.

**PRNG Draw Sequence**

The generator consumes PRNG draws in the following fixed order for each invocation:

1. Cluster center coordinates (if geographic_distribution = `clustered`): cluster_count × 2 draws (lat, lon for each center).
2. Stop coordinates: stop_count × 2 draws (lat, lon for each stop, in stop index order from 0 to stop_count - 1).
3. Stop demands: stop_count draws (one per stop, in stop index order).
4. Time window stop selection: drawn by shuffling stop indices (see FR-5).
5. Time window center values: one draw per selected stop, in selected stop index order.

This sequence is fixed. Any deviation from this order is a backward-incompatible change to the reproducibility guarantee.

**Acceptance Criteria:**

- All generated coordinates are within the specified bounding box (after clamping).
- All generated coordinates satisfy SPEC-001 FR-5 bounds.
- The depot is placed at the bounding box center and does not vary with seed.
- Given the same seed, generation configuration, and bounding box, stop coordinates are identical across invocations.
- For uniform distribution: stop coordinates are drawn independently from the bounding box extents.
- For clustered distribution: each stop is assigned to a cluster in round-robin order and its coordinates are derived from the assigned cluster center.
- For clustered distribution: cluster_std_dev_degrees controls the spread of stops around their cluster centers.

---

## FR-4: Demand Generation Strategy

**Description:** The generator assigns a non-negative integer demand to each stop. Demands are drawn uniformly from [demand_min, demand_max] (inclusive, both endpoints) using integer uniform sampling.

**Capacity Feasibility Guarantee**

The generator must not produce a routing problem where total stop demand exceeds total fleet capacity (SPEC-001 FR-8). The generator enforces this guarantee through a pre-emission check after all demands are drawn.

If total generated demand > vehicle_count × capacity_per_vehicle, generation fails with a structured error. The generator does not retry with a different seed. Retry is the caller's responsibility.

To reduce the likelihood of this failure, the generator performs a feasibility forecast before beginning generation. If the expected total demand (stop_count × (demand_min + demand_max) / 2) exceeds total fleet capacity, the generator emits a warning in structured log output before proceeding. The generation is not rejected at the forecast stage; the warning informs the caller that infeasibility is likely.

**Acceptance Criteria:**

- Each stop receives a demand value drawn uniformly from [demand_min, demand_max].
- All demand values are non-negative integers.
- All demand values satisfy SPEC-001 FR-4 (demand ≥ 0).
- Total stop demand ≤ total fleet capacity (vehicle_count × capacity_per_vehicle) in all emitted problems.
- If total stop demand > total fleet capacity after demand generation, generation fails before producing a routing problem document. The structured error states the total generated demand and total fleet capacity.
- Given the same seed and demand configuration, the same demand values are produced for the same stop sequence.
- The generator emits a structured warning log event when expected total demand exceeds total fleet capacity, before generation begins.

---

## FR-5: Time Window Generation Strategy

**Description:** The generator assigns time windows to a subset of stops determined by the time_window_density parameter. Time windows conform to SPEC-001 FR-9: time_window_open and time_window_close are non-negative integers in seconds from route start (t=0), with time_window_open strictly less than time_window_close.

**Stop Selection**

The number of stops receiving time windows is:

```
windowed_stop_count = floor(stop_count × time_window_density)
```

When time_window_density = 0.0, windowed_stop_count = 0 and no stops receive time windows.
When time_window_density = 1.0, windowed_stop_count = stop_count and all stops receive time windows.

The generator selects which stops receive time windows by performing a PRNG-driven shuffle of stop indices (using the Fisher-Yates algorithm) and taking the first windowed_stop_count indices from the shuffled order. This is the step labeled "time window stop selection" in the FR-3 PRNG draw sequence.

**Time Window Construction**

For each stop selected to receive a time window:

1. A window center is drawn uniformly from the integer range:

```
[ceil(time_window_width_seconds / 2),
 planning_horizon_seconds - floor(time_window_width_seconds / 2)]
```

2. time_window_open = center - floor(time_window_width_seconds / 2)
3. time_window_close = center + ceil(time_window_width_seconds / 2)

This ensures:
- time_window_open ≥ 0
- time_window_close ≤ planning_horizon_seconds
- time_window_close > time_window_open (as required by SPEC-001 FR-9)

The generator does not compute a tightness metric for the generated windows. Average time window width relative to planning horizon is a derived workload feature computed by the Core (SPEC-003).

**Acceptance Criteria:**

- The count of stops receiving time windows equals floor(stop_count × time_window_density).
- Every generated time window pair satisfies SPEC-001 FR-9: time_window_open < time_window_close, both non-negative integers.
- Every time_window_close ≤ planning_horizon_seconds.
- Every time_window_open ≥ 0.
- Stops not selected for time windows carry no time window fields in the routing problem document.
- Stops selected for time windows carry both time_window_open and time_window_close.
- For time_window_density = 0.0: no stops carry time window fields, and the generated problem has time_windows constraint type flag = false (per SPEC-001 FR-9).
- For time_window_density = 1.0: all stops carry time window fields, and the generated problem has time_windows constraint type flag = true.
- Given the same seed and configuration, time window assignments and values are identical across invocations.

---

## FR-6: Fleet Generation Strategy

**Description:** The generator produces a fleet definition from the generation configuration parameters vehicle_count and capacity_per_vehicle. These are explicit inputs; the generator does not derive vehicle count or capacity automatically.

**Guidance on Fleet Sizing**

The following guidelines produce routing problems where fleet capacity is a binding constraint without making the problem trivially infeasible. These are not enforced constraints; they are documentation for scenario authors.

- Capacity utilization ratio (total demand / total fleet capacity) between 0.6 and 0.95 produces problems where capacity is an active constraint.
- Vehicle count between ceil(stop_count / 10) and ceil(stop_count / 3) produces problems where the number of routes is interesting.

The generator does not validate the utilization ratio. The only capacity constraint enforced by the generator is the feasibility check in FR-4 (total demand ≤ total fleet capacity).

**Acceptance Criteria:**

- The generated fleet contains exactly vehicle_count vehicles, each with capacity capacity_per_vehicle, matching the generation configuration.
- A generation configuration with vehicle_count = 0 or capacity_per_vehicle = 0 is rejected (FR-2).
- The fleet definition in the routing problem document is consistent with the generation configuration.

---

## FR-7: Difficulty Classification

**Description:** Each generated problem is assigned a nominal difficulty tier. The difficulty tier is a generator annotation based on the generation parameters. It describes the generator's intent — the structural properties that were requested — not the actual workload feature values computed by the Core.

**Separation from Workload Features**

The difficulty tier is not a workload feature. Workload features (capacity utilization ratio, time window density, average time window width, problem size class) are computed by Daedalus Core from the loaded problem, as defined in SPEC-001 FR-10 and the SPEC-003 computation contract. The generator's difficulty tier annotation may differ from what the Core computes if random variance produces a problem whose actual structure diverges from the generator's intent. This discrepancy is expected and acceptable.

The difficulty tier is stored in the generation manifest (FR-8). It is not a field in the SPEC-001 routing problem document.

**Difficulty Tier Definitions**

| Tier | Stop Count (from SPEC-001 FR-7) | time_window_density |
| --- | --- | --- |
| Easy | ≤ 25 (Small size class) | 0.0 |
| Medium | 26–75 (Medium size class) | > 0.0 and ≤ 0.75 |
| Hard | ≥ 76 (Large size class), OR any size class with time_window_density > 0.75 | Any |

Assignment rules:

1. If stop_count ≥ 76: tier is Hard, regardless of time_window_density.
2. If stop_count ≤ 25 and time_window_density = 0.0: tier is Easy.
3. If time_window_density > 0.75: tier is Hard.
4. Otherwise: tier is Medium.

The difficulty tier criteria reference SPEC-001 FR-7 size class thresholds (Small: 1–25, Medium: 26–75, Large: 76+). If those thresholds are revised via configuration, the difficulty tier criteria above should be reviewed for consistency.

**Acceptance Criteria:**

- Every generated problem carries a difficulty tier in its generation manifest.
- The difficulty tier is one of: Easy, Medium, Hard.
- A problem with stop_count ≤ 25 and time_window_density = 0.0 is assigned tier Easy.
- A problem with stop_count ≥ 76 is assigned tier Hard.
- A problem with time_window_density > 0.75 is assigned tier Hard regardless of stop count.
- The difficulty tier does not appear in the SPEC-001 routing problem document.

---

## FR-8: Output Contract

**Description:** The generator produces two distinct outputs per invocation: a routing problem document and a generation manifest.

**Routing Problem Document**

The routing problem document is a JSON document conforming to the SPEC-001 input schema. It is the artifact submitted to the API.

The routing problem document contains:

```json
{
  "fleet": {
    "vehicle_count": <integer>,
    "capacity_per_vehicle": <integer>
  },
  "depot": {
    "latitude": <double>,
    "longitude": <double>
  },
  "stops": [
    {
      "id": <integer, 0-indexed>,
      "latitude": <double>,
      "longitude": <double>,
      "demand": <integer>,
      "time_window_open": <integer, seconds from route start, optional>,
      "time_window_close": <integer, seconds from route start, optional>,
      "service_duration": <integer, seconds, optional>
    }
  ],
  "seed": <uint64>
}
```

The stop id field is the stop's 0-based index in generation order (stop 0, stop 1, ..., stop stop_count - 1). Stop ids are unique within the document, satisfying SPEC-001 FR-4.

service_duration is included when service_duration_seconds > 0. When service_duration_seconds = 0 (the default), the service_duration field may be omitted from each stop, which is equivalent to 0 per SPEC-001 FR-16.

The routing problem document must pass the SPEC-001 API structural and domain validation. The generator performs a pre-emission validation pass covering the capacity feasibility check (FR-4) and time window structural checks (FR-5) before producing the document. If any pre-emission check fails, no document is produced.

**Generation Manifest**

The generation manifest is a separate JSON document associated with the routing problem document. It is a development and benchmarking artifact. It is not submitted to the API and is not persisted through the standard submission path.

The manifest contains:

| Field | Type | Description |
| --- | --- | --- |
| spec_version | string | The SPEC-002 version that produced this manifest (e.g., "SPEC-002-draft-1"). |
| scenario_type | string | The scenario type used (FR-1). |
| difficulty_tier | string | The nominal difficulty tier assigned (FR-7). |
| seed | uint64 | The seed used. Must match the seed in the routing problem document. |
| stop_count | integer | Number of stops. |
| time_windowed_stop_count | integer | Number of stops with time windows. |
| total_demand | integer | Sum of all stop demands. |
| total_fleet_capacity | integer | vehicle_count × capacity_per_vehicle. |
| planning_horizon_seconds | integer | Planning horizon used. |
| generation_config | object | Full generation configuration (all resolved parameters, including defaults). |

The manifest does not contain per-stop coordinate arrays, per-stop demand arrays, or the full stop list. It is a structural summary.

**Submission Path**

The routing problem document is produced as output and submitted to the API via the standard SPEC-001 submission path. The mechanism by which the generator hands off the document (CLI passthrough, direct API call, file write) is an implementation decision addressed in OQ-2. This spec does not prescribe the submission mechanism.

**Acceptance Criteria:**

- The routing problem document is valid JSON conforming to the SPEC-001 input schema.
- The routing problem document passes SPEC-001 API structural and domain validation without modification when submitted.
- The generation manifest contains all required fields.
- The seed in the manifest equals the seed in the routing problem document.
- total_demand in the manifest equals the sum of all stop demands in the routing problem document.
- total_fleet_capacity in the manifest equals vehicle_count × capacity_per_vehicle.
- time_windowed_stop_count in the manifest equals the count of stops carrying time_window_open and time_window_close fields.
- The routing problem document does not contain difficulty_tier, scenario_type, or any generation manifest field.
- The generation manifest does not contain per-stop coordinate arrays.
- The generation manifest spec_version field identifies the SPEC-002 version.

---

## FR-9: Reproducibility Guarantees

**Description:** Given the same generation configuration (including seed), two invocations of the generator on any supported platform produce identical routing problem documents and identical generation manifests. This is a hard guarantee. Any change to the PRNG algorithm, PRNG draw sequence, or generation logic that would produce a different output for the same seed and configuration is a backward-incompatible change and must be identified as such.

**PRNG Seeding**

The PRNG is seeded once per generation invocation using the seed value from the generation configuration. No other entropy sources (wall clock, process ID, hardware random, operating system random) contribute to generation. The seed is the sole source of randomness.

**PRNG Algorithm**

The PRNG algorithm must be fixed and documented. It must produce identical output for a given seed across all supported platforms and compiler configurations.

Open Question OQ-1 (Blocking): The specific PRNG algorithm has not been selected. This is a blocking open question. See the Open Questions section.

**Coordinate Precision**

All coordinate values are generated and stored as IEEE 754 double precision (64-bit) values, consistent with SPEC-001 FR-5. Floating-point operations in the generation path must not introduce platform-dependent rounding differences that would break reproducibility.

**PRNG Draw Sequence**

The fixed draw sequence is defined in FR-3. Any reordering of PRNG draws is a backward-incompatible change.

**Acceptance Criteria:**

- Given the same generation configuration (including seed), two invocations produce byte-for-byte identical routing problem documents after JSON serialization.
- The PRNG is seeded once per invocation from the seed value only.
- No external entropy sources are consumed.
- Reproducibility holds across process restarts.
- Reproducibility holds across the same platform and compiler configuration. Cross-platform reproducibility is contingent on OQ-1 resolution.
- The PRNG algorithm is identified in this spec (pending OQ-1 resolution).

---

# Non-Requirements

- No real geographic routing. The generator does not use road networks, geographic databases, address geocoding, or map data.
- No multi-depot scenarios. Single depot only, per SPEC-001 FR-3.
- No heterogeneous fleet scenarios. All vehicles have identical capacity, per SPEC-001 FR-2.
- No multi-dimensional demand. Single-dimension demand only, per SPEC-001 FR-2.
- No dynamic or streaming generation. Each invocation produces a complete, static routing problem.
- No automatic parameter sweep or scenario variation across seeds. The caller controls all parameters per invocation.
- No per-stop variation in service duration. A single service_duration_seconds value is applied uniformly.
- No workload feature computation. Capacity utilization ratio, time window density, average time window width, and problem size class are computed by the Core and defined in SPEC-003.
- No solver selection or difficulty-based routing decisions. Solver selection is the Scheduler's responsibility.
- No real-time data integration. Generated scenarios are entirely synthetic.
- No batch generation API. One invocation produces one routing problem.
- No submission to the API. The generator produces a document; the submission mechanism is outside this spec.
- No direct PostgreSQL writes. Persistence is through the standard API submission path only.
- No authentication or authorization. The generator is a developer-facing tool.

---

# Assumptions

- SPEC-001 is approved and stable. The generator's output contract depends on the SPEC-001 routing problem model and schema. Changes to SPEC-001 require reviewing this spec for compatibility.
- SPEC-001 size class thresholds (Small: 1–25, Medium: 26–75, Large: 76+) are the values in effect at implementation time. FR-7 difficulty tier criteria reference these thresholds. If thresholds are revised via configuration, FR-7 must be reviewed.
- The generation configuration is provided by a developer or test harness. The generator does not sanitize inputs for untrusted callers beyond the structural validation rules in FR-2.
- The default bounding box is a configurable value. Changing it does not require code changes.
- A generated routing problem is submitted via the existing SPEC-001 submission path. The generator is not responsible for API availability, authentication, or submission retries.
- For the MVP, uniform service duration per stop is sufficient. Per-stop variable service duration is deferred to a future extension (OQ-4).

---

# Constraints

- Generated routing problem documents must conform to SPEC-001. The generator may not produce documents that require relaxation of SPEC-001 validation rules.
- The generator must not write directly to PostgreSQL. Persistence is through the API path only (ADR-004).
- The PRNG algorithm must be deterministic and must not depend on external entropy (pending OQ-1 resolution).
- No entropy sources other than the seed value may contribute to generation.
- All coordinate values must be IEEE 754 double precision (ADR-001 C++ requirement; SPEC-001 FR-5).
- The generation configuration must be self-contained. All parameters required to reproduce a problem must be present in the generation manifest (FR-8).

---

# Inputs

| Input | Source | Description |
| --- | --- | --- |
| Generation configuration | Caller (CLI, test harness, benchmark script) | Complete set of parameters described in FR-2. Includes the seed. |

The generation configuration is the only external input to the generator. No database reads, API calls, or file reads are required during generation.

---

# Outputs

| Output | Consumer | Description |
| --- | --- | --- |
| Routing problem document | API submission path | SPEC-001-conforming JSON document. |
| Generation manifest | Developer, benchmarking tools | Metadata document retained as a generation artifact. Not submitted to the API. |
| Structured log events | Observability stack | Generation events per Observability Requirements section. |
| `workload.generate` trace span | Observability stack | One span per invocation. |

---

# Failure Modes

### Invalid Generation Configuration

**Cause:** A required generation configuration field is missing, or a field value violates a structural validation rule defined in FR-2 (e.g., demand_max < demand_min, unrecognized scenario type, cluster_count absent when geographic_distribution = `clustered`).

**Expected behavior:** Generation fails before any generation step runs. A structured error identifies the invalid field and the violated rule.

**Expected fallback:** None. The caller must correct the configuration.

**User-visible result:** Structured error message with field name and rule description.

---

### Capacity Infeasibility Under Generated Demands

**Cause:** After all stop demands are drawn, total stop demand exceeds total fleet capacity (vehicle_count × capacity_per_vehicle).

**Expected behavior:** Generation fails after the demand generation step completes. No routing problem document is produced. A structured error states the total generated demand, the total fleet capacity, and the shortfall.

**Expected fallback:** None. The generator does not retry. The caller must adjust capacity_per_vehicle, vehicle_count, demand_max, or stop_count, then resubmit.

**User-visible result:** Structured error message with total_demand and total_fleet_capacity values.

---

### Coordinate Out of Bounds

**Cause:** A generated coordinate (after clamping for clustered distribution) falls outside the valid lat/lon ranges defined in SPEC-001 FR-5. This is a defensive check; it should not occur under correct generation logic with a valid bounding box.

**Expected behavior:** Generation fails before the routing problem document is produced. A structured error identifies the offending stop index and coordinate value.

**Expected fallback:** None. This failure indicates a generator defect and requires investigation.

**User-visible result:** Structured error identifying the defective coordinate and stop index.

---

### Invalid Time Window Generated

**Cause:** A generated time window pair violates time_window_open < time_window_close after the construction procedure in FR-5. This is a defensive check; it should not occur under correct generation logic.

**Expected behavior:** Generation fails before the routing problem document is produced. A structured error identifies the offending stop index and the generated pair.

**Expected fallback:** None. This failure indicates a generator defect.

**User-visible result:** Structured error identifying the defective window and stop index.

---

### Pre-Emission Validation Failure

**Cause:** The pre-emission validation pass detects a SPEC-001 violation not covered by the more specific failure modes above (e.g., a duplicate stop id, a negative service_duration). This is a defensive check.

**Expected behavior:** Generation fails before the routing problem document is produced. A structured error identifies the violated SPEC-001 rule.

**Expected fallback:** None. This failure indicates a generator defect.

**User-visible result:** Structured error with the SPEC-001 rule reference.

---

# Architectural Impact

| Component | Impact |
| --- | --- |
| Domain Layer | Yes |
| API Layer | Indirect — generator output is submitted via the existing SPEC-001 submission path |
| Persistence | Indirect — manifest storage mechanism is an open question (OQ-3) |
| Solver Runtime | No |
| Simulation Framework | Yes — this spec defines the simulation framework entry point |
| Observability | Yes |
| Security | Yes |
| Configuration | Yes |

**Domain Layer:** The generator produces SPEC-001 routing problem representations. If the generator is implemented as a C++ library (OQ-2), it operates within Daedalus Core's domain layer. If implemented in the CLI, it produces a JSON document without needing the full C++ domain type.

**API Layer:** No changes to the API are required. The generator's output enters the system through the existing SPEC-001 submission path.

**Persistence:** The routing problem document is persisted by the API per SPEC-001 FR-12. The generation manifest requires a separate storage decision (OQ-3). No schema changes are required until OQ-3 is resolved.

**Simulation Framework:** The generator is the simulation framework's entry point. Scenario reproduction, solver benchmarking, and scheduler evaluation all begin with a generation configuration and seed.

**Observability:** The `workload.generate` trace span is not in the architecture document's required span list. It must be added during implementation.

**Configuration:** The default bounding box center, default planning_horizon_seconds, default time_window_width_seconds, and the PRNG algorithm selection are all configuration values or implementation constants that must be configurable.

---

# Testability

The following behaviors must be proven before this feature is considered complete. Specific test names and implementation structure are determined during implementation planning.

## Scenario Type Behaviors

- Each of the four scenario types, when invoked without parameter overrides, produces a routing problem with the expected default time_window_density.
- An unrecognized scenario type name produces a structured generation error before any output is produced.
- Providing an explicit time_window_density override supersedes the scenario type default.

## Configuration Validation Behaviors

- A configuration with demand_max < demand_min is rejected with a structured error before generation begins.
- A configuration with time_window_width_seconds ≥ planning_horizon_seconds is rejected.
- A configuration with geographic_distribution = `clustered` and cluster_count absent is rejected.
- A configuration with cluster_count present and geographic_distribution ≠ `clustered` is rejected.
- A configuration with cluster_count > stop_count is rejected.
- All valid configurations proceed to generation without error.

## Geographic Behaviors

- Given the same seed, generation configuration, and bounding box, stop coordinates are identical across invocations.
- All generated stop coordinates are within the specified bounding box.
- All generated coordinates satisfy SPEC-001 FR-5 bounds (latitude ∈ [-90.0, 90.0], longitude ∈ [-180.0, 180.0]).
- The depot is placed at the bounding box center, independent of seed.
- For clustered distribution: stops are concentrated around cluster centers rather than uniformly distributed. This is verifiable by computing the mean distance of stops to their assigned cluster center and comparing it to the mean distance to a random non-assigned center.

## Demand Behaviors

- Given the same seed and demand configuration, the same demand values are produced.
- All demands are within [demand_min, demand_max].
- All demands are non-negative integers.
- Total stop demand ≤ total fleet capacity in all emitted routing problem documents.
- A configuration that produces total_demand > total_fleet_capacity emits a structured error and no routing problem document.

## Time Window Behaviors

- The number of stops with time windows equals floor(stop_count × time_window_density) for all time_window_density values in [0.0, 1.0].
- All generated time window pairs satisfy time_window_open < time_window_close (SPEC-001 FR-9).
- All time_window_open values are ≥ 0.
- All time_window_close values are ≤ planning_horizon_seconds.
- For time_window_density = 0.0: no stops carry time window fields. The routing problem document has time_windows constraint type flag = false.
- For time_window_density = 1.0: all stops carry time window fields. The routing problem document has time_windows constraint type flag = true.

## Reproducibility Behaviors

- Two invocations with the same seed and configuration produce identical routing problem documents (byte-for-byte equivalent after JSON serialization with a deterministic serializer).
- Different seeds produce different stop coordinate distributions and demand distributions.

## Output Contract Behaviors

- The routing problem document passes SPEC-001 API structural and domain validation when submitted without modification.
- The generation manifest contains all fields defined in FR-8.
- The seed in the manifest matches the seed in the routing problem document.
- total_demand in the manifest equals the sum of stop demands in the routing problem document.
- total_fleet_capacity in the manifest equals vehicle_count × capacity_per_vehicle.
- time_windowed_stop_count in the manifest equals the count of stops with time window fields in the routing problem document.
- The routing problem document does not contain any generation manifest fields.
- The generation manifest does not contain per-stop coordinate arrays.

## Difficulty Tier Behaviors

- A problem with stop_count ≤ 25 and time_window_density = 0.0 is annotated Easy.
- A problem with stop_count ≥ 76 is annotated Hard.
- A problem with time_window_density > 0.75 is annotated Hard regardless of stop count.
- Difficulty tier appears in the generation manifest.
- Difficulty tier does not appear in the routing problem document.

## Telemetry Behaviors

- A successful generation emits a structured log event with all fields required by the Observability Requirements section.
- A failed generation emits a structured log event identifying the failure reason.
- Per-stop coordinate arrays do not appear in any log event.
- A `workload.generate` trace span is emitted for each invocation with the required attributes.

---

# Observability Requirements

Observability must be designed alongside this feature.

## Structured Log Events

**On successful generation**

The generator emits a structured log event when a routing problem document is successfully produced. The event must carry sufficient information to reproduce or describe the generation without accessing the document itself.

Required fields:
- seed
- scenario_type
- difficulty_tier
- stop_count
- time_windowed_stop_count
- total_demand
- total_fleet_capacity
- geographic_distribution
- generation_duration_ms (wall time of the full generation step in milliseconds)

Per-stop coordinate arrays must not appear in log output.

**On generation failure**

The generator emits a structured log event when generation fails at any stage. The event must identify the failure mode and carry diagnostic context.

Required fields:
- seed (if available from the configuration)
- scenario_type (if available)
- failure_reason (a bounded, structured value from the defined failure modes)
- Relevant numeric context (e.g., total_demand and total_fleet_capacity for capacity infeasibility failure)

**On feasibility forecast warning**

The generator emits a structured warning log event when expected total demand exceeds total fleet capacity at the configuration validation stage (FR-4). This event precedes generation and may precede a capacity infeasibility failure.

Required fields:
- seed
- scenario_type
- expected_total_demand
- total_fleet_capacity

## Metrics

The following quantities must be observable through metrics:

- Total routing problems generated successfully (counter).
- Total generation failures, disaggregated by failure_reason (counter). The set of failure_reason values must be bounded and match the failure modes in the Failure Modes section.
- Distribution of stop_count values across generated problems (histogram).
- Distribution of time_window_density values across generated problems (histogram).
- Distribution of generation_duration_ms values (histogram).

## Trace Spans

**`workload.generate`** (proposed, in the generator component)

This span is not in the architecture document's required span list. It is proposed here as a generator-specific span. Adding it to the required spans list in the architecture document is a documentation update identified in the Documentation Updates section.

The span must carry:
- seed
- scenario_type
- stop_count
- difficulty_tier
- outcome (success or failure)
- failure_reason (when outcome = failure)

The `workload.generate` span should be propagated to the downstream `job.submit` span when the generated problem is submitted to the API, enabling correlation between generation and execution in evidence reports.

---

# Security Considerations

The generator is a developer-facing tool. It does not process external user input through a network boundary. No authentication or authorization is applicable.

Generated coordinates are synthetic and carry no real-world geographic significance. No personally identifiable information is expected in generated problems, consistent with SPEC-001 security considerations.

**Log safety**: Per-stop coordinate arrays must not appear in log output, consistent with SPEC-001 observability requirements. The generation manifest must not be exposed through the API. It is an internal development artifact.

**No direct database access**: The generator must not write directly to PostgreSQL (ADR-004). All persistence is through the standard API submission path. This preserves the API as the single trust boundary for routing problem persistence.

---

# Performance Considerations

Generation of a single routing problem is expected to complete in milliseconds at all MVP stop counts. Specific latency targets should be measured during implementation.

For the Large size class (76+ stops), coordinate generation (FR-3) and demand generation (FR-4) are O(N) in stop count. This is not expected to be a performance bottleneck at MVP scale. The generation_duration_ms metric defined in the Observability Requirements section will provide empirical data.

The PRNG draw sequence (FR-3) is fixed and bounded at O(N) draws total. No generation step is superlinear in stop_count.

The generation manifest is small by design (no per-stop arrays). Its production time is negligible.

Do not invent specific latency targets. These require measurement.

---

# Documentation Updates Required

- Architecture documentation: Add `workload.generate` to the required spans list. Add the generation manifest as a development artifact in the system context.
- SPEC-003 (pending): Must define workload feature computation contracts for the features that the Core computes from generator-produced problems.
- SPEC-001: No changes required. The generator conforms to SPEC-001 as published.
- This spec (SPEC-002): Must be updated when OQ-1 (PRNG algorithm selection) is resolved to record the selected algorithm.

---

# Open Questions

## Blocking

### OQ-1: PRNG Algorithm Selection

**Question:** Which PRNG algorithm should the generator use?

**Why it matters:** The reproducibility guarantee (FR-9) requires that the PRNG algorithm be fixed and produce identical output for a given seed across all supported platforms and compilers. This choice must be made before implementation begins. The algorithm selection is a backward-incompatible commitment: changing the algorithm in a later version breaks reproducibility for all previously generated scenarios.

**Candidates:**

| Algorithm | Notes |
| --- | --- |
| std::mt19937 (Mersenne Twister, C++11) | Standard library availability. Well-known. Some seeding behavior differs between libstdc++ and libc++, which may break cross-compiler reproducibility if the full seed state is not handled carefully. |
| PCG64 (Permuted Congruential Generator) | Well-specified. Deterministic across platforms. Requires a dependency or a small implementation. No standard library coupling. |
| xoshiro256** | Fast. Easily portable. Well-tested. Small state. Requires a small implementation. |

**Cross-platform scope:** Does the reproducibility guarantee need to hold across different C++ compilers (GCC, Clang) and platforms (Linux, macOS, Windows), or only within a single compiler/platform combination for the Docker Compose environment?

**Clarification needed from author before implementation.** FR-9 cannot be finalized until this question is resolved.

---

## Non-Blocking

### OQ-2: Generator Process Location

**Question:** Where does the generator code live in the system?

**Why it matters:** Three candidates exist:

1. A library within Daedalus Core (C++) — benefits from direct access to the C++ domain types; callable by the CLI and test harness.
2. A subcommand of the Daedalus CLI — generator logic in the CLI layer; produces a JSON document without full domain type access.
3. A standalone binary — separate executable, invoked by the CLI or test scripts.

The architecture document does not assign the generator to a specific component. Option 1 (Core library) aligns best with the architecture's statement that Daedalus Core owns domain logic. Option 2 (CLI subcommand) keeps the CLI simple but separates generation logic from the C++ runtime.

This is an implementation planning decision. It does not require spec revision.

---

### OQ-3: Generation Manifest Storage

**Question:** How and where is the generation manifest persisted?

**Why it matters:** The manifest is a development and benchmarking artifact. Options:

1. Written to a local file alongside the routing problem JSON (no schema change required).
2. Written to PostgreSQL in an additional table (requires schema addition, makes manifests queryable).
3. Retained only in memory; not persisted beyond the invocation (simplest, loses traceability).

For benchmarking and report generation, persistent storage is valuable. The mechanism is an implementation decision. Resolution deferred to implementation planning.

---

### OQ-4: Per-Stop Service Duration Variation

**Question:** Should service duration be configurable per stop, or applied uniformly?

**Why it matters:** SPEC-001 FR-16 defines service_duration as a per-stop field. The current spec (FR-2) treats service_duration_seconds as a single value applied uniformly to all stops. Per-stop variation would add realism to benchmarking scenarios but increases parameter complexity. For the MVP, uniform service duration is likely sufficient to demonstrate time window behavior without adding a distribution parameter.

Resolution deferred to implementation planning. If per-stop variation is needed, this spec must be revised to add service_duration_min_seconds and service_duration_max_seconds parameters.

---

### OQ-5: Named Problem Instances

**Question:** Should generated problems carry a human-readable name field?

**Why it matters:** SPEC-001 OQ-4 (deferred) asks whether routing problems should carry an optional name field. If SPEC-001 resolves OQ-4 in the affirmative, the generator should populate the name field with a string derived from the scenario type, difficulty tier, and seed (e.g., `"capacity-only-easy-seed-42"`). Named problems improve evidence report readability.

Resolution contingent on SPEC-001 OQ-4 resolution.

---

# Acceptance Checklist

- [x] Problem is clearly defined.
- [x] Domain concept is defined.
- [x] Requirements are testable.
- [x] Non-requirements are documented.
- [x] Assumptions are explicit.
- [x] Failure modes are defined.
- [x] Observability requirements exist.
- [x] Security considerations exist.
- [x] Documentation updates are identified.
- [x] Responsibility boundary table distinguishes generator responsibilities from SPEC-001 and SPEC-003 responsibilities.
- [x] Difficulty tier is distinguished from Core-computed workload features.
- [ ] OQ-1 (PRNG algorithm) is blocking and unresolved. Clarification required before implementation planning begins.
- [ ] OQ-2 (generator process location) deferred to implementation planning.
- [ ] OQ-3 (manifest storage) deferred to implementation planning.
- [ ] OQ-4 (per-stop service duration variation) deferred to implementation planning.
- [ ] OQ-5 (named problem instances) contingent on SPEC-001 OQ-4 resolution.

---

# Definition of Done

This feature is complete when:

- All four scenario types defined in FR-1 produce structurally valid routing problems with their expected defaults.
- All generation configuration parameters defined in FR-2 are accepted, validated, and applied correctly.
- Geographic generation conforms to FR-3 for both uniform and clustered distributions.
- Demand generation conforms to FR-4, including the capacity feasibility guarantee and the feasibility forecast warning.
- Time window generation conforms to FR-5 for all time_window_density values in [0.0, 1.0].
- Fleet generation conforms to FR-6.
- Difficulty tier assignment conforms to FR-7.
- All outputs conform to the output contract in FR-8 (routing problem document and generation manifest).
- Reproducibility is proven per FR-9 (contingent on OQ-1 resolution).
- All testability behaviors in the Testability section are proven.
- All failure modes produce structured errors before any invalid output is emitted.
- The routing problem document is submitted to the API and receives HTTP 202, confirming SPEC-001 conformance.
- The generation manifest is produced and retained as defined in OQ-3's resolution.
- All observability behaviors in the Observability Requirements section are proven.
- OQ-1 (PRNG algorithm selection) is resolved and FR-9 is updated.
- Security review passes.
- Engineering review passes.
- Specification status is updated to Verified.

The feature is not complete simply because the generator produces a JSON document. The reproducibility guarantee must be proven, the document must pass SPEC-001 submission, and all observability behaviors must be operational.
