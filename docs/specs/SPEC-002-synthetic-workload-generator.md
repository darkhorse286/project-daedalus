# Feature Specification

## Metadata

**Feature ID:** SPEC-002

**Title:** Synthetic Workload Generator

**Status:** Accepted

**Author:** Darkhorse286

**Created:** 2026-06-08

**Last Updated:** 2026-06-20

**Related ADRs:** ADR-001, ADR-004, ADR-006, ADR-009, ADR-010

**Related Specs:** SPEC-001, SPEC-003

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

The generator produces raw problem properties. It does not compute derived workload features. It does not select solvers. It does not define routing problem validation rules. It verifies its own construction invariants; it is not a validation authority under ADR-009.

---

# Requirements

## FR-1: Scenario Taxonomy

**Description:** The generator supports a defined taxonomy of scenario types. A scenario type is a named generation intent that sets default values for generation parameters and documents the structural purpose of the scenario. Scenario types exist to give developers and test harnesses a shared vocabulary for describing problem structure without requiring every parameter to be specified explicitly.

The scenario type governs the default `time_window_density`. All other parameters — including `stop_count`, which is always required — remain individually configurable. Scenario type descriptions include guidance on the intended stop count range, but this is informational only. The scenario type does not set a `stop_count` default.

The initial scenario taxonomy is:

| Scenario Type | Description | Default time_window_density |
| --- | --- | --- |
| `capacity-only` | Pure capacity VRP with no time windows on any stop. Baseline scenario for classical solver evaluation. Intended for problems in the Small size class (up to the configured Small class upper bound). | 0.0 |
| `partial-time-windowed` | Capacity VRP with time windows on roughly half the stops. Tests scheduler handling of mixed constraint problems. Intended for problems in the Medium size class. | 0.5 |
| `dense-time-windowed` | Capacity VRP with time windows on all stops. Maximum-constraint scenario for solver stress testing. Intended for problems in the Medium or Large size class. | 1.0 |
| `stress` | High stop count problem at or near the upper boundary of the Large size class, combining high stop counts with time windows to push classical solver performance. Intended for stop counts at or above the configured Large class lower bound. | 0.5 |

Scenario types are defaults, not rigid templates. A caller may override any parameter while specifying a scenario type. The override takes precedence over the scenario type default.

**Acceptance Criteria:**

- The generator recognizes each of the four scenario types by name.
- Specifying a scenario type without additional parameter overrides produces a problem whose `time_window_density` matches the scenario type default.
- A caller may override the `time_window_density` default by providing an explicit value in the generation configuration.
- An unrecognized scenario type name is rejected with a structured error before generation begins.
- A scenario type does not set a `stop_count` default. Providing a scenario type without providing `stop_count` is a validation error.

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
| average_vehicle_speed_kmh | positive float64 | Fleet-wide average vehicle speed in km/h. Passed through unchanged as the routing problem `average_vehicle_speed_kmh` field (SPEC-001 FR-17). Applied uniformly to all vehicles. Used to derive travel times from Haversine distances when evaluating time window achievability (FR-5). Must be positive and non-zero. |

**Optional Parameters (with defaults)**

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| demand_min | non-negative integer | 1 | Minimum per-stop demand value (inclusive). |
| demand_max | positive integer | max(1, floor(capacity_per_vehicle / 4)) | Maximum per-stop demand value (inclusive). Must be ≥ demand_min. With this default and the fleet sizing guidance in FR-6, expected capacity utilization falls in the 60–80% range at typical stop-to-vehicle ratios. |
| time_window_density | double [0.0, 1.0] | Set by scenario_type | Fraction of stops that receive time windows. |
| time_window_width_seconds | positive integer | 3600 | Width of each generated time window in seconds. Controls tightness relative to planning horizon. |
| planning_horizon_seconds | positive integer | 28800 | Total route time horizon in seconds (t=0 = route start, per SPEC-001 FR-9). Time window values are bounded within [0, planning_horizon_seconds]. |
| service_duration_seconds | non-negative integer | 0 | Service duration applied uniformly to all stops (SPEC-001 FR-16). |
| geographic_distribution | enum | `uniform` | Stop coordinate distribution. Valid values: `uniform`, `clustered`. |
| cluster_count | positive integer | absent | Number of geographic clusters. Required when geographic_distribution is `clustered`. |
| cluster_std_dev_degrees | positive double | 0.05 | Standard deviation of stop coordinates around cluster centers, in degrees. Used only when geographic_distribution is `clustered`. Approximately 5.5 km at mid-latitude. |
| bounding_box | object | See FR-3 | Geographic region for coordinate generation. Structure: `{ min_lat, max_lat, min_lon, max_lon }` where all four fields are double-precision values. See structural validation rules below. |

**Structural Validation Rules**

The following configuration errors must be rejected before generation begins:

- `demand_max` < `demand_min`.
- `time_window_width_seconds` ≥ `planning_horizon_seconds` (produces degenerate windows spanning the full horizon).
- `geographic_distribution` = `clustered` with `cluster_count` absent.
- `cluster_count` present with `geographic_distribution` ≠ `clustered`.
- `cluster_count` > `stop_count`.
- `vehicle_count` = 0.
- `capacity_per_vehicle` = 0.
- `stop_count` = 0.
- `average_vehicle_speed_kmh` ≤ 0 (zero or negative speed is invalid per SPEC-001 FR-17).
- `bounding_box.min_lat` ≥ `bounding_box.max_lat`.
- `bounding_box.min_lon` ≥ `bounding_box.max_lon`.
- Any `bounding_box` coordinate outside the SPEC-001 FR-5 valid ranges (latitude ∈ [-90.0, 90.0], longitude ∈ [-180.0, 180.0]).

**Acceptance Criteria:**

- A configuration satisfying all structural validation rules proceeds to generation.
- A configuration violating any structural validation rule is rejected with a structured error identifying the violated rule before any generation step runs.
- Each optional parameter applies its documented default when absent.
- A configuration with `scenario_type` absent is rejected.
- A configuration with `stop_count` absent is rejected.
- A configuration with `average_vehicle_speed_kmh` absent is rejected.
- A configuration with `average_vehicle_speed_kmh` ≤ 0 is rejected.

---

## FR-3: Geographic Generation Strategy

**Description:** The generator produces geographic coordinates for the depot and all stops. All coordinates must conform to SPEC-001 FR-5: latitude in [-90.0, 90.0], longitude in [-180.0, 180.0], represented as IEEE 754 double precision.

**Bounding Box**

All stop and depot coordinates are generated within a configurable bounding box. The bounding box is specified as `{ min_lat, max_lat, min_lon, max_lon }`.

The default bounding box covers an area of approximately 200 km × 200 km at mid-latitude. The default reference center (`center_lat`, `center_lon`) is a configurable value, not a constant in generation logic. The default reference center has no real-world geographic significance.

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

When `geographic_distribution` = `uniform`, each stop's latitude and longitude are drawn independently from uniform distributions over the bounding box extents. Latitude draws and longitude draws are independent.

**Stop Coordinate Generation: Clustered Distribution**

When `geographic_distribution` = `clustered`:

1. `cluster_count` cluster centers are drawn uniformly and independently from the bounding box interior. Cluster centers consume PRNG draws before stop coordinates.
2. Each stop is assigned to a cluster center in round-robin order: stop 0 → cluster 0, stop 1 → cluster 1, ..., stop k → cluster k mod cluster_count. Round-robin assignment is used rather than random assignment because it produces even cluster sizes, which are preferable for controlled benchmarking: even cluster sizes make solver performance comparable across seeds and scenarios. Variable cluster sizes (from random assignment) introduce variance that reduces result interpretability.
3. Each stop's latitude and longitude are drawn from independent normal distributions centered at its assigned cluster center, with standard deviation `cluster_std_dev_degrees`. Normal distribution sampling uses the Box-Muller transform per ADR-010 Decision 3. One Box-Muller application consumes 2 PRNG draws and produces 2 outputs: z0 (latitude offset) and z1 (longitude offset). Both z0 and z1 must be consumed. This accounts for the `stop_count × 2` draw count at step 2 of the PRNG draw sequence: one Box-Muller application per stop, not two independent uniform draws.
4. If a drawn coordinate falls outside the bounding box, it is clamped to the nearest bounding box boundary. Clamping does not consume additional PRNG draws.

**Clamping Limitation:** When a cluster center is placed near the bounding box boundary, stops assigned to that cluster are more likely to be clamped, introducing artificial geographic density at the bounding box edges. This effect is minimal when cluster centers are well inside the bounding box, which is the typical case. Callers requiring spatially uniform distribution should use `geographic_distribution` = `uniform`.

**PRNG Draw Sequence**

The generator uses PCG64 as its PRNG (ADR-010 Decision 1), seeded once per invocation from the `seed` value in the generation configuration. The PCG64 stream constant for this component is a fixed implementation constant, defined and frozen at implementation time per ADR-010. No other entropy sources are used (ADR-010 Decision 4).

The generator consumes PRNG draws in the following fixed order for each invocation:

1. Cluster center coordinates (only when `geographic_distribution` = `clustered`): `cluster_count` × 2 draws (lat, lon for each center, in cluster index order 0 to cluster_count − 1). Each coordinate is a uniform draw over the bounding box extent.
2. Stop coordinates: `stop_count` × 2 draws (lat, lon for each stop, in stop index order 0 to stop_count − 1). For `geographic_distribution` = `uniform`: each coordinate is a uniform draw over the bounding box extent. For `geographic_distribution` = `clustered`: each stop consumes one Box-Muller application (2 draws), producing z0 = latitude offset and z1 = longitude offset. Both z0 and z1 are consumed per stop.
3. Stop demands: `stop_count` draws (one per stop, in stop index order). Each draw is converted to a bounded uniform integer in [`demand_min`, `demand_max`] using a bias-free integer arithmetic algorithm per ADR-010 Decision 3. `std::uniform_int_distribution` is not used.
4. Time window stop selection: a partial Fisher-Yates shuffle of stop indices consuming exactly `windowed_stop_count` draws, where `windowed_stop_count` = floor(`stop_count` × `time_window_density`). The shuffle terminates after `windowed_stop_count` indices are selected. If `time_window_density` = 0.0, `windowed_stop_count` = 0 and step 4 consumes no draws.
5. Time window center values: one draw per selected stop, in selected stop index order (ascending stop index among selected stops). Each draw is converted to a bounded uniform integer in [ceil(w / 2), H − ceil(w / 2)] using a bias-free integer arithmetic algorithm per ADR-010 Decision 3. `std::uniform_int_distribution` is not used.

This sequence is fixed. Any deviation from this order, or any change to the draw count at any step, is a backward-incompatible change to the reproducibility guarantee.

**Note on distribution type and seed:** Because step 1 is conditional on `geographic_distribution`, the same seed produces different stop coordinates for `uniform` vs. `clustered` configurations. This is expected behavior: the generation configuration is the reproducibility unit. Two configurations that differ in `geographic_distribution` are not required to produce related outputs from the same seed.

**Acceptance Criteria:**

- All generated coordinates are within the specified bounding box (after clamping).
- All generated coordinates satisfy SPEC-001 FR-5 bounds.
- The depot is placed at the bounding box center and does not vary with seed.
- Given the same seed, generation configuration, and bounding box, stop coordinates are identical across invocations.
- For uniform distribution: stop coordinates are drawn independently from the bounding box extents.
- For clustered distribution: each stop is assigned to a cluster in round-robin order and its coordinates are derived from the assigned cluster center using `cluster_std_dev_degrees` as the standard deviation.
- For clustered distribution: coordinate values outside the bounding box are clamped to the bounding box boundary without consuming additional PRNG draws.

---

## FR-4: Demand Generation Strategy

**Description:** The generator assigns a non-negative integer demand to each stop. Demands are drawn uniformly from [`demand_min`, `demand_max`] (inclusive, both endpoints) using integer uniform sampling.

**Capacity Feasibility Guarantee**

The generator must not produce a routing problem where total stop demand exceeds total fleet capacity (SPEC-001 FR-8). The generator enforces this guarantee through a construction self-check after all demands are drawn.

If total generated demand > `vehicle_count` × `capacity_per_vehicle`, generation fails with a structured error. The generator does not retry with a different seed. Retry is the caller's responsibility.

**Utilization Guidance**

With the default `demand_max` = max(1, floor(`capacity_per_vehicle` / 4)) and the fleet sizing guidance in FR-6, expected capacity utilization falls in the 60–80% range at typical stop-to-vehicle ratios. Callers targeting a specific utilization level should set `demand_max` accordingly. For example, to target approximately 80% utilization with 10 vehicles and 50 stops, set `demand_max` such that `stop_count` × (`demand_min` + `demand_max`) / 2 ≈ 0.80 × `vehicle_count` × `capacity_per_vehicle`.

**Acceptance Criteria:**

- Each stop receives a demand value drawn uniformly from [`demand_min`, `demand_max`].
- All demand values are non-negative integers.
- All demand values satisfy SPEC-001 FR-4 (demand ≥ 0).
- Total stop demand ≤ total fleet capacity (`vehicle_count` × `capacity_per_vehicle`) in all emitted problems.
- If total stop demand > total fleet capacity after demand generation, generation fails before producing a routing problem document. The structured error states the total generated demand and total fleet capacity.
- Given the same seed and demand configuration, the same demand values are produced for the same stop sequence.

---

## FR-5: Time Window Generation Strategy

**Description:** The generator assigns time windows to a subset of stops determined by the `time_window_density` parameter. Time windows conform to SPEC-001 FR-9: `time_window_open` and `time_window_close` are non-negative integers in seconds from route start (t=0), with `time_window_open` strictly less than `time_window_close`.

**Stop Selection**

The number of stops receiving time windows is:

```
windowed_stop_count = floor(stop_count × time_window_density)
```

When `time_window_density` = 0.0, `windowed_stop_count` = 0 and no stops receive time windows. When `time_window_density` = 1.0, `windowed_stop_count` = `stop_count` and all stops receive time windows.

The generator selects which stops receive time windows using the partial Fisher-Yates shuffle described in FR-3's PRNG draw sequence (step 4). The first `windowed_stop_count` indices produced by the shuffle are the selected stops.

**Time Window Construction**

Let `w` = `time_window_width_seconds` and `H` = `planning_horizon_seconds`.

For each stop selected to receive a time window:

1. A window center is drawn uniformly from the integer range:

```
[ceil(w / 2),  H - ceil(w / 2)]
```

2. `time_window_open`  = center − floor(w / 2)
3. `time_window_close` = center + ceil(w / 2)

This guarantees for all valid integer values of `w` and `H` where `w` < `H`:
- `time_window_open` ≥ 0 (since center ≥ ceil(w/2) ≥ floor(w/2))
- `time_window_close` ≤ H (since center ≤ H − ceil(w/2), so center + ceil(w/2) ≤ H)
- `time_window_close` > `time_window_open` (since ceil(w/2) + floor(w/2) = w ≥ 1)

The structural validation rule in FR-2 (`time_window_width_seconds` < `planning_horizon_seconds`) ensures the center draw range is non-empty.

**Tightness**

Time window tightness is governed by `time_window_width_seconds` relative to `planning_horizon_seconds`. Narrow windows (small width relative to horizon) are tight. Wide windows are loose. The generator does not compute a tightness metric; average time window width relative to planning horizon is a derived workload feature computed by the Core (SPEC-003).

**Speed-Dependent Achievability**

Generated time windows must be achievable under the configured `average_vehicle_speed_kmh`. Travel time between two locations is derived from Haversine distance using the formula defined in SPEC-001 FR-17 (and FR-9):

```
travel_time_seconds = (haversine_distance_km / average_vehicle_speed_kmh) × 3600
```

Generated time windows must not assume a different speed. The value of `average_vehicle_speed_kmh` in the generation configuration is the sole speed parameter for achievability evaluation.

The generator's achievability obligation: for each stop assigned a time window, the stop's `time_window_close` must be reachable from the depot at t=0 under the configured speed. That is, for each time-windowed stop:

```
(haversine(depot, stop) / average_vehicle_speed_kmh) × 3600 < time_window_close
```

A time-windowed stop that violates this condition is a construction self-check failure (see Failure Modes). The generator does not produce a routing problem document when any time-windowed stop is not individually reachable from the depot under the configured speed.

This is a necessary but not sufficient condition for full route feasibility. Full route feasibility — considering inter-stop travel, service durations, vehicle count, and routing interactions — is not guaranteed by the depot-reachability check alone. Callers targeting highly feasible workloads should configure `average_vehicle_speed_kmh`, `planning_horizon_seconds`, `time_window_width_seconds`, and `bounding_box` such that inter-stop travel is achievable given the generated geography. The relationship between bounding box size, speed, and planning horizon determines the practical feasibility of generated time-windowed problems.

**Acceptance Criteria:**

- The count of stops receiving time windows equals floor(`stop_count` × `time_window_density`).
- Every generated time window pair satisfies SPEC-001 FR-9: `time_window_open` < `time_window_close`, both non-negative integers.
- Every `time_window_close` ≤ `planning_horizon_seconds`.
- Every `time_window_open` ≥ 0.
- Stops not selected for time windows carry no time window fields in the routing problem document.
- Stops selected for time windows carry both `time_window_open` and `time_window_close`.
- For `time_window_density` = 0.0: no stops carry time window fields, and the generated problem has `time_windows` constraint type flag = false (per SPEC-001 FR-9).
- For `time_window_density` = 1.0: all stops carry time window fields, and the generated problem has `time_windows` constraint type flag = true.
- For every time-windowed stop: `(haversine(depot, stop) / average_vehicle_speed_kmh) × 3600` < `time_window_close`. A violation is a construction self-check failure; no routing problem document is produced.
- Given the same seed and configuration, time window assignments and values are identical across invocations.

---

## FR-6: Fleet Generation Strategy

**Description:** The generator produces a fleet definition from the generation configuration parameters `vehicle_count` and `capacity_per_vehicle`. These are explicit inputs; the generator does not derive vehicle count or capacity automatically.

**Guidance on Fleet Sizing**

The following guidelines produce routing problems where fleet capacity is a binding constraint without making the problem trivially infeasible. These are not enforced constraints; they are documentation for scenario authors.

- Capacity utilization ratio (total demand / total fleet capacity) between 0.6 and 0.95 produces problems where capacity is an active constraint.
- Vehicle count between ceil(`stop_count` / 10) and ceil(`stop_count` / 3) produces problems where the number of routes is interesting.

The generator does not validate the utilization ratio. The only capacity constraint enforced by the generator is the construction self-check in FR-4 (total demand ≤ total fleet capacity).

**Acceptance Criteria:**

- The generated fleet contains exactly `vehicle_count` vehicles, each with capacity `capacity_per_vehicle`, matching the generation configuration.
- A generation configuration with `vehicle_count` = 0 or `capacity_per_vehicle` = 0 is rejected (FR-2).
- The fleet definition in the routing problem document is consistent with the generation configuration.

---

## FR-7: Difficulty Classification

**Description:** Each generated problem is assigned a nominal difficulty tier. The difficulty tier is a generator annotation based on the generation parameters. It describes the generator's intent — the structural properties that were requested — not the actual workload feature values computed by the Core.

**Separation from Workload Features**

The difficulty tier is not a workload feature. Workload features (capacity utilization ratio, time window density, average time window width, problem size class) are computed by Daedalus Core from the loaded problem, as defined in SPEC-001 FR-10 and the SPEC-003 computation contract. The generator's difficulty tier annotation may differ from what the Core computes if random variance produces a problem whose actual structure diverges from the generator's intent. This discrepancy is expected and acceptable.

The difficulty tier is stored in the generation manifest (FR-8). It is not a field in the SPEC-001 routing problem document.

**Difficulty Tier Definitions**

The generator reads the SPEC-001 size class threshold values from configuration at runtime. It does not hardcode numeric literals. The current configured values are: Small class upper bound = 25, Medium class upper bound = 75 (per SPEC-001 FR-7). If those thresholds are reconfigured, the generator uses the updated values without code changes.

| Tier | Stop Count | time_window_density |
| --- | --- | --- |
| Easy | ≤ configured Small class upper bound | 0.0 |
| Medium | > configured Small class upper bound AND ≤ configured Medium class upper bound | > 0.0 and ≤ 0.75 |
| Hard | > configured Medium class upper bound, OR time_window_density > 0.75 | Any |

**Assignment Rules**

1. If `stop_count` > configured Medium class upper bound: tier is Hard, regardless of `time_window_density`.
2. If `time_window_density` > 0.75: tier is Hard, regardless of `stop_count`.
3. If `stop_count` ≤ configured Small class upper bound AND `time_window_density` = 0.0: tier is Easy.
4. Otherwise: tier is Medium.

**Note on Small size class with time windows:** A problem in the Small size class with any time-windowed stops (`time_window_density` > 0.0) is classified Medium under rule 4. This is intentional: time window constraints change the problem class from CVRP to CVRPTW, materially increasing computational complexity even for small stop counts.

**Acceptance Criteria:**

- Every generated problem carries a difficulty tier in its generation manifest.
- The difficulty tier is one of: Easy, Medium, Hard.
- The generator reads the configured size class threshold values rather than hardcoded numeric literals.
- A problem with `stop_count` ≤ configured Small class upper bound and `time_window_density` = 0.0 is assigned tier Easy.
- A problem with `stop_count` > configured Medium class upper bound is assigned tier Hard.
- A problem with `time_window_density` > 0.75 is assigned tier Hard regardless of `stop_count`.
- A problem with `stop_count` in the Small size class and `time_window_density` > 0.0 is assigned tier Medium.
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
  "seed": <uint64>,
  "average_vehicle_speed_kmh": <double>
}
```

The stop `id` field is the stop's 0-based index in generation order (stop 0, stop 1, ..., stop `stop_count` − 1). Stop ids are unique within the document, satisfying SPEC-001 FR-4.

`service_duration` is included when `service_duration_seconds` > 0. When `service_duration_seconds` = 0 (the default), the `service_duration` field may be omitted from each stop, which is equivalent to 0 per SPEC-001 FR-16.

**Construction Self-Check**

The generator performs a construction self-check on its output before emission. This verifies that the generation logic produced what it was designed to produce. It is not domain validation under ADR-009 — the generator is not a validation authority. The self-check covers the invariants the generator guarantees by construction: capacity feasibility (the total demand check in FR-4), time window structural consistency (the open < close invariant and bounds from FR-5), and coordinate bounds (the bounding box constraints from FR-3). If a self-check fails, it indicates a defect in the generator's own logic, not a caller input error. No routing problem document is produced when a self-check fails.

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
| total_fleet_capacity | integer | `vehicle_count` × `capacity_per_vehicle`. |
| planning_horizon_seconds | integer | Planning horizon used. |
| generation_config | object | Generation parameters governing the generation algorithm. Excludes `seed`, which appears as a top-level field. |

The manifest does not contain per-stop coordinate arrays, per-stop demand arrays, or the full stop list. It is a structural summary.

**Submission Path**

The routing problem document is produced as output and submitted to the API via the standard SPEC-001 submission path. The mechanism by which the generator hands off the document (CLI passthrough, direct API call, file write) is an implementation decision addressed in OQ-2. This spec does not prescribe the submission mechanism.

**Acceptance Criteria:**

- The routing problem document is valid JSON conforming to the SPEC-001 input schema.
- The routing problem document passes SPEC-001 API structural and domain validation when submitted without modification.
- The generation manifest contains all required fields.
- The seed in the manifest equals the seed in the routing problem document.
- total_demand in the manifest equals the sum of all stop demands in the routing problem document.
- total_fleet_capacity in the manifest equals `vehicle_count` × `capacity_per_vehicle`.
- time_windowed_stop_count in the manifest equals the count of stops carrying `time_window_open` and `time_window_close` fields.
- The `generation_config` field in the manifest contains all generation parameters except `seed`.
- The routing problem document does not contain `difficulty_tier`, `scenario_type`, or any generation manifest field.
- The generation manifest does not contain per-stop coordinate arrays.
- The generation manifest `spec_version` field identifies the SPEC-002 version.
- The routing problem document includes `average_vehicle_speed_kmh` equal to the configured `average_vehicle_speed_kmh`.

---

## FR-9: Reproducibility Guarantees

**Description:** Given the same generation configuration (including seed), two invocations of the generator on any supported platform produce semantically equivalent routing problem documents and identical generation manifests: all parsed field values are identical. This is a hard guarantee. Any change to the PRNG algorithm, PRNG draw sequence, or generation logic that would produce a different output for the same seed and configuration is a backward-incompatible change and must be identified as such.

**PRNG Seeding**

The PRNG is seeded once per generation invocation using the `seed` value from the generation configuration. No other entropy sources (wall clock, process ID, hardware random, operating system random) contribute to generation. The seed is the sole source of randomness.

**PRNG Algorithm**

The PRNG algorithm is PCG64, per ADR-010 Decision 1. PCG64 is seeded once per generation invocation from the `seed` value in the generation configuration. The PCG64 stream constant is a fixed implementation constant, defined and frozen at implementation time per ADR-010. No other entropy sources (wall clock, process ID, hardware random, operating system random) may contribute to generation, per ADR-010 Decision 4.

**Coordinate Precision**

All coordinate values are generated and stored as IEEE 754 double precision (64-bit) values, consistent with SPEC-001 FR-5. Floating-point operations in the generation path must not introduce platform-dependent rounding differences that would break reproducibility.

**PRNG Draw Sequence**

The fixed draw sequence is defined in FR-3. Any reordering of PRNG draws, or any change to the draw count at any step, is a backward-incompatible change.

**Acceptance Criteria:**

- Given the same generation configuration (including seed), two invocations produce semantically equivalent routing problem documents: all parsed field values — coordinates, demands, time windows, fleet parameters, and seed — are identical.
- The PRNG is seeded once per invocation from the `seed` value only.
- No external entropy sources are consumed.
- Reproducibility holds across process restarts.
- Reproducibility holds across any conforming C++17 toolchain with IEEE 754 double precision support. The standard is semantic equivalence: all parsed field values — coordinates, demands, time windows, fleet parameters, and seed — are identical. This is guaranteed by PCG64's platform-independent integer arithmetic and the approved distribution sampling algorithms, per ADR-010 Decision 2.
- The PRNG algorithm is PCG64, per ADR-010 Decision 1.

---

# Non-Requirements

- No per-vehicle speed variation. All vehicles travel at the same speed defined by `average_vehicle_speed_kmh`, consistent with SPEC-001 FR-17.
- No real geographic routing. The generator does not use road networks, geographic databases, address geocoding, or map data.
- No multi-depot scenarios. Single depot only, per SPEC-001 FR-3.
- No heterogeneous fleet scenarios. All vehicles have identical capacity, per SPEC-001 FR-2.
- No multi-dimensional demand. Single-dimension demand only, per SPEC-001 FR-2.
- No dynamic or streaming generation. Each invocation produces a complete, static routing problem.
- No automatic parameter sweep or scenario variation across seeds. The caller controls all parameters per invocation.
- No per-stop variation in service duration. A single `service_duration_seconds` value is applied uniformly.
- No workload feature computation. Capacity utilization ratio, time window density, average time window width, and problem size class are computed by the Core and defined in SPEC-003.
- No solver selection or difficulty-based routing decisions. Solver selection is the Scheduler's responsibility.
- No real-time data integration. Generated scenarios are entirely synthetic.
- No batch generation API. One invocation produces one routing problem.
- No direct PostgreSQL writes. Persistence is through the standard API submission path only.
- No authentication or authorization. The generator is a developer-facing tool.

---

# Assumptions

- SPEC-001 is approved and stable. The generator's output contract depends on the SPEC-001 routing problem model and schema. Changes to SPEC-001 require reviewing this spec for compatibility.
- SPEC-001 size class thresholds are the values in effect at implementation time. FR-7 difficulty tier criteria reference these thresholds. If thresholds are revised via configuration, the generator reads the updated values without code changes.
- The generation configuration is provided by a developer or test harness. The generator does not sanitize inputs for untrusted callers beyond the structural validation rules in FR-2.
- The default bounding box is a configurable value. Changing it does not require code changes.
- A generated routing problem is submitted via the existing SPEC-001 submission path. The generator is not responsible for API availability, authentication, or submission retries.
- For the MVP, uniform service duration per stop is sufficient. Per-stop variable service duration is deferred to a future extension (OQ-4).
- `average_vehicle_speed_kmh` is an explicit generation configuration parameter provided by the caller. It is not derived from system configuration, defaults, or external sources.
- All vehicles in the generated fleet travel at the same average speed (`average_vehicle_speed_kmh`). Per-vehicle speed variation is not generated, consistent with SPEC-001 Non-Requirements.

---

# Constraints

- Generated routing problem documents must conform to SPEC-001. The generator may not produce documents that require relaxation of SPEC-001 validation rules.
- The generator must not write directly to PostgreSQL. Persistence is through the API path only (ADR-004).
- The PRNG algorithm is PCG64 (ADR-010 Decision 1). It must not depend on external entropy (ADR-010 Decision 4).
- No entropy sources other than the `seed` value may contribute to generation.
- All coordinate values must be IEEE 754 double precision (ADR-001 C++ requirement; SPEC-001 FR-5).
- The generation configuration must be self-contained. All parameters required to reproduce a problem must be present in the generation manifest (FR-8).
- The generator must read the configured SPEC-001 size class threshold values when assigning difficulty tiers (FR-7). Hardcoded numeric literals for size class boundaries are not permitted.
- The generator's construction self-checks (FR-8) are not ADR-009 validation sites. The generator verifies its own generation invariants only. The API and Core remain the authoritative validation authorities as defined in ADR-009.

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
| Routing problem document | Caller (CLI, test harness, benchmark script) | SPEC-001-conforming JSON document. |
| Generation manifest | Developer, benchmarking tools | Metadata document retained as a generation artifact. Not submitted to the API. |
| Structured log events | Observability stack | Generation events per Observability Requirements section. |
| `workload.generate` trace span | Observability stack | One span per invocation. |

---

# Failure Modes

### Invalid Generation Configuration

**Cause:** A required generation configuration field is missing, or a field value violates a structural validation rule defined in FR-2 (e.g., `demand_max` < `demand_min`, unrecognized scenario type, `cluster_count` absent when `geographic_distribution` = `clustered`, bounding box with min ≥ max).

**Expected behavior:** Generation fails before any generation step runs. A structured error identifies the invalid field and the violated rule.

**Expected fallback:** None. The caller must correct the configuration.

**User-visible result:** Structured error message with field name and rule description.

---

### Capacity Infeasibility Under Generated Demands

**Cause:** After all stop demands are drawn, total stop demand exceeds total fleet capacity (`vehicle_count` × `capacity_per_vehicle`).

**Expected behavior:** Generation fails after the demand generation step completes. No routing problem document is produced. A structured error states the total generated demand, the total fleet capacity, and the shortfall.

**Expected fallback:** None. The generator does not retry. The caller must adjust `capacity_per_vehicle`, `vehicle_count`, `demand_max`, or `stop_count`, then resubmit.

**User-visible result:** Structured error message with `total_demand` and `total_fleet_capacity` values.

---

### Coordinate Out of Bounds (Construction Self-Check)

**Cause:** A construction self-check detects that a generated coordinate falls outside the valid lat/lon range defined by SPEC-001 FR-5, even after clamping for the clustered distribution case. This self-check verifies the generator's bounding box and clamping logic produced correct output. It should not trigger under correct generation logic with a valid bounding box.

**Expected behavior:** Generation fails before the routing problem document is produced. A structured error identifies the offending stop index and coordinate value.

**Expected fallback:** None. This failure indicates a defect in the generator's construction logic and requires investigation.

**User-visible result:** Structured error identifying the offending coordinate and stop index.

---

### Invalid Time Window (Construction Self-Check)

**Cause:** A construction self-check detects that a generated time window pair violates `time_window_open` < `time_window_close`. This self-check verifies the FR-5 construction procedure produced internally consistent output. It should not trigger under correct generation logic.

**Expected behavior:** Generation fails before the routing problem document is produced. A structured error identifies the offending stop index and the generated pair.

**Expected fallback:** None. This failure indicates a defect in the generator's construction logic.

**User-visible result:** Structured error identifying the defective window and stop index.

---

### Time Window Achievability Failure (Construction Self-Check)

**Cause:** A construction self-check detects that a time-windowed stop is not individually reachable from the depot under the configured `average_vehicle_speed_kmh`: `(haversine(depot, stop) / average_vehicle_speed_kmh) × 3600` ≥ `time_window_close`.

**Expected behavior:** Generation fails after time window assignment. No routing problem document is produced. A structured error identifies the offending stop index, the computed travel time from the depot, and the `time_window_close` value that was not met.

**Expected fallback:** None. The caller must adjust `average_vehicle_speed_kmh`, `bounding_box`, `time_window_width_seconds`, or `planning_horizon_seconds` and reinvoke.

**User-visible result:** Structured error with stop index, computed travel time, and `time_window_close` value showing the unmet achievability condition.

---

### Construction Invariant Violation

**Cause:** A construction self-check detects a generation invariant violation not covered by the more specific failure modes above (e.g., a duplicate stop id, a stop demand outside the configured range). These are checks the generator runs on its own output to detect defects in generation logic. They are not implementations of SPEC-001 domain validation rules.

**Expected behavior:** Generation fails before the routing problem document is produced. A structured error identifies the violated invariant.

**Expected fallback:** None. This failure indicates a defect in the generator's construction logic.

**User-visible result:** Structured error with the violated invariant description.

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

**Configuration:** The default bounding box center, default `planning_horizon_seconds`, default `time_window_width_seconds`, the PRNG algorithm, and the SPEC-001 size class threshold values are all configuration values that must be accessible to the generator.

---

# Testability

The following behaviors must be proven before this feature is considered complete. Specific test names and implementation structure are determined during implementation planning.

## Scenario Type Behaviors

- Each of the four scenario types, when invoked without parameter overrides, produces a routing problem with the expected default `time_window_density`.
- An unrecognized scenario type name produces a structured generation error before any output is produced.
- Providing an explicit `time_window_density` override supersedes the scenario type default.
- Providing a scenario type without `stop_count` produces a validation error.

## Configuration Validation Behaviors

- A configuration with `demand_max` < `demand_min` is rejected with a structured error before generation begins.
- A configuration with `time_window_width_seconds` ≥ `planning_horizon_seconds` is rejected.
- A configuration with `geographic_distribution` = `clustered` and `cluster_count` absent is rejected.
- A configuration with `cluster_count` present and `geographic_distribution` ≠ `clustered` is rejected.
- A configuration with `cluster_count` > `stop_count` is rejected.
- A configuration with a `bounding_box` where `min_lat` ≥ `max_lat` is rejected.
- A configuration with a `bounding_box` where `min_lon` ≥ `max_lon` is rejected.
- A configuration with a `bounding_box` coordinate outside SPEC-001 FR-5 valid ranges is rejected.
- A configuration with `average_vehicle_speed_kmh` absent is rejected with a structured error before generation begins.
- A configuration with `average_vehicle_speed_kmh` = 0 is rejected.
- A configuration with a negative `average_vehicle_speed_kmh` is rejected.
- All valid configurations proceed to generation without error.

## Geographic Behaviors

- Given the same seed, generation configuration, and bounding box, stop coordinates are identical across invocations.
- All generated stop coordinates are within the specified bounding box.
- All generated coordinates satisfy SPEC-001 FR-5 bounds (latitude ∈ [-90.0, 90.0], longitude ∈ [-180.0, 180.0]).
- The depot is placed at the bounding box center, independent of seed.
- For clustered distribution with `cluster_count` = 1: at least 99% of generated stop coordinates are within 3 × `cluster_std_dev_degrees` of the single cluster center. (This follows from the three-sigma property of the normal distribution and requires no empirical calibration.)
- For clustered distribution with multiple cluster centers: the mean distance from each stop to its assigned cluster center is less than half the mean distance from that stop to any non-assigned cluster center.
- For clustered distribution where a cluster center is placed at a bounding box boundary by test construction, all generated stop coordinates assigned to that cluster are within the bounding box (coordinates are clamped, not outside the bounding box).

## Demand Behaviors

- Given the same seed and demand configuration, the same demand values are produced.
- All demands are within [`demand_min`, `demand_max`].
- All demands are non-negative integers.
- Total stop demand ≤ total fleet capacity in all emitted routing problem documents.
- A configuration that produces total_demand > total_fleet_capacity emits a structured error and no routing problem document.

## Time Window Behaviors

- The number of stops with time windows equals floor(`stop_count` × `time_window_density`) for all `time_window_density` values in [0.0, 1.0].
- All generated time window pairs satisfy `time_window_open` < `time_window_close` (SPEC-001 FR-9).
- All `time_window_open` values are ≥ 0.
- All `time_window_close` values are ≤ `planning_horizon_seconds`.
- The above hold for both even and odd values of `time_window_width_seconds`.
- For `time_window_density` = 0.0: no stops carry time window fields. The routing problem document has `time_windows` constraint type flag = false.
- For `time_window_density` = 1.0: all stops carry time window fields. The routing problem document has `time_windows` constraint type flag = true.
- For each time-windowed stop in a generated problem, `(haversine(depot, stop) / average_vehicle_speed_kmh) × 3600` < `time_window_close`.
- A configuration that would produce a time-windowed stop not individually reachable from the depot under the configured speed produces a construction self-check failure with no routing problem document emitted.

## Reproducibility Behaviors

- Two invocations with the same seed and configuration produce semantically equivalent routing problem documents: all parsed field values — coordinates, demands, time windows, fleet parameters, and seed — are identical.
- Different seeds produce different stop coordinate distributions and demand distributions.

## Output Contract Behaviors

- The routing problem document passes SPEC-001 API structural and domain validation when submitted without modification.
- The generation manifest contains all fields defined in FR-8.
- The seed in the manifest equals the seed in the routing problem document.
- total_demand in the manifest equals the sum of stop demands in the routing problem document.
- total_fleet_capacity in the manifest equals `vehicle_count` × `capacity_per_vehicle`.
- time_windowed_stop_count in the manifest equals the count of stops with time window fields in the routing problem document.
- The `generation_config` field in the manifest contains all generation parameters except `seed`.
- The routing problem document does not contain any generation manifest fields.
- The generation manifest does not contain per-stop coordinate arrays.
- The routing problem document includes `average_vehicle_speed_kmh` equal to the configured value.

## Difficulty Tier Behaviors

- The generator reads size class thresholds from configuration rather than using hardcoded literals.
- A problem with `stop_count` ≤ configured Small class upper bound and `time_window_density` = 0.0 is annotated Easy.
- A problem with `stop_count` > configured Medium class upper bound is annotated Hard.
- A problem with `time_window_density` > 0.75 is annotated Hard regardless of `stop_count`.
- A problem with `stop_count` in the Small size class and `time_window_density` > 0.0 is annotated Medium.
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

Span context from `workload.generate` should be carried as output alongside the routing problem document. Context propagation to the downstream `job.submit` span is the responsibility of the submitting entity (CLI or equivalent tool), not the generator.

---

# Security Considerations

The generator is a developer-facing tool. It does not process external user input through a network boundary. No authentication or authorization is applicable.

Generated coordinates are synthetic and carry no real-world geographic significance. No personally identifiable information is expected in generated problems, consistent with SPEC-001 security considerations.

**Log safety:** Per-stop coordinate arrays must not appear in log output, consistent with SPEC-001 observability requirements. The generation manifest must not be exposed through the API. It is an internal development artifact.

**No direct database access:** The generator must not write directly to PostgreSQL (ADR-004). All persistence is through the standard API submission path. This preserves the API as the single trust boundary for routing problem persistence.

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
- SPEC-001: SPEC-001 FR-17 formally defines `average_vehicle_speed_kmh` as a required routing problem field. SPEC-002 now accepts this field in FR-2 (generation configuration) and emits it in FR-8 (routing problem document output). This revision aligns SPEC-002 with the accepted SPEC-001 routing problem model. No further SPEC-001 changes are required.

---

# Open Questions

## Blocking

### OQ-1: PRNG Algorithm and Normal Distribution Sampling Algorithm Selection

**Status: Resolved — 2026-06-08 — ADR-010**

| Sub-question | Resolution | ADR-010 Reference |
| --- | --- | --- |
| PRNG algorithm | PCG64 | Decision 1 |
| Cross-platform reproducibility scope | Semantic equivalence across conforming C++17 toolchains with IEEE 754 support | Decision 2 |
| Normal distribution sampling algorithm | Box-Muller transform (two-output form; z0 = latitude offset, z1 = longitude offset per stop) | Decision 3 |
| Bounded integer sampling algorithm | Bias-free integer arithmetic algorithm (Lemire's method or bounded rejection sampling); `std::uniform_int_distribution` prohibited | Decision 3 |
| Variable-draw-count algorithms | Prohibited in fixed-draw-sequence contexts; Marsaglia polar method not selected | Decision 3 |
| Floating-point determinism | IEEE 754 double precision required; semantic equivalence within double precision applies to transcendental function results | Decision 6 |

---

## Non-Blocking

### OQ-2: Generator Process Location

**Question:** Where does the generator code live in the system?

**Why it matters:** Three candidates exist:

1. A library within Daedalus Core (C++) — benefits from direct access to the C++ domain types; callable by the CLI and test harness.
2. A subcommand of the Daedalus CLI — generator logic in the CLI layer; produces a JSON document without full domain type access.
3. A standalone binary — separate executable, invoked by the CLI or test scripts.

The architecture document does not assign the generator to a specific component. Option 1 (Core library) aligns best with the architecture's statement that Daedalus Core owns domain logic. This decision does not require spec revision.

---

### OQ-3: Generation Manifest Storage

**Question:** How and where is the generation manifest persisted?

**Why it matters:** Options include: local file alongside the routing problem JSON, a PostgreSQL table, or in-memory only. For benchmarking and report generation, persistent storage is valuable. A consumer interface for the manifest should be defined once the storage mechanism is decided.

Resolution deferred to implementation planning.

---

### OQ-4: Per-Stop Service Duration Variation

**Question:** Should service duration be configurable per stop, or applied uniformly?

**Why it matters:** SPEC-001 FR-16 defines service_duration as a per-stop field. The current spec treats `service_duration_seconds` as a single value applied uniformly. Per-stop variation would add realism but increases parameter complexity. For MVP, uniform service duration is sufficient.

Resolution deferred to implementation planning. If per-stop variation is needed, this spec must be revised to add `service_duration_min_seconds` and `service_duration_max_seconds` parameters.

---

### OQ-5: Named Problem Instances

**Question:** Should generated problems carry a human-readable name field?

**Why it matters:** Contingent on SPEC-001 OQ-4 resolution. If SPEC-001 adds a name field, the generator should populate it with a string derived from the scenario type, difficulty tier, and seed (e.g., `"capacity-only-easy-seed-42"`). Named problems improve evidence report readability.

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
- [x] Difficulty tier criteria reference configured thresholds, not hardcoded literals.
- [x] Construction self-checks are distinguished from ADR-009 validation authority.
- [x] Time window construction formula verified correct for all valid integer inputs.
- [x] PRNG draw sequence fully specified including draw counts.
- [x] OQ-1 (PRNG algorithm and normal distribution sampling algorithm) resolved by ADR-010.
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
- Demand generation conforms to FR-4, including the capacity feasibility self-check.
- Time window generation conforms to FR-5 for all `time_window_density` values in [0.0, 1.0], including both even and odd values of `time_window_width_seconds`.
- Fleet generation conforms to FR-6.
- Difficulty tier assignment conforms to FR-7, reading configured size class thresholds.
- All outputs conform to the output contract in FR-8 (routing problem document and generation manifest).
- Reproducibility is proven per FR-9, including semantic equivalence across conforming C++17 toolchains per ADR-010 Decision 2.
- All testability behaviors in the Testability section are proven.
- All failure modes produce structured errors before any invalid output is emitted.
- The routing problem document is submitted to the API and receives HTTP 202, confirming SPEC-001 conformance.
- The generation manifest is produced and retained as defined in OQ-3's resolution.
- All observability behaviors in the Observability Requirements section are proven.
- FR-3 and FR-9 correctly reference ADR-010 and identify PCG64 as the PRNG algorithm and Box-Muller as the normal distribution sampling algorithm.
- Security review passes.
- Engineering review passes.
- Specification status is updated to Verified.

The feature is not complete simply because the generator produces a JSON document. The reproducibility guarantee must be proven, the document must pass SPEC-001 submission, and all observability behaviors must be operational.
