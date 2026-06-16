# SPEC-012 Persistence Schema

## Metadata

**Feature ID:** SPEC-012

**Title:** Persistence Schema

**Status:** Accepted

**Author:** Darkhorse286

**Created:** 2026-06-15

**Last Updated:** 2026-06-16

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-004, ADR-006, ADR-009, ADR-010, ADR-011

**Related Specs:** SPEC-001, SPEC-003, SPEC-004, SPEC-005, SPEC-006, SPEC-007, SPEC-008, SPEC-009, SPEC-010, SPEC-011

---

# Problem Statement

Project DAEDALUS distributes persistence requirements across eight accepted specifications and three accepted ADRs. Each specification defines the fields that must exist in persisted records for its domain, but no single specification defines the physical PostgreSQL schema, the table boundaries, the column types, the referential integrity constraints, the read/write ownership map, or the record lifecycle in terms of concrete database operations.

Without a persistence schema specification:

- Implementers must synthesize schema design from eight separate specifications, risking misinterpretation of field types, nullable semantics, and FK relationships.
- The boundary between SPEC-006 (evidence semantics) and physical schema realization is undefined, creating ambiguity about which specification is authoritative for any given schema question.
- Whether backend capability profiles require a PostgreSQL table is unresolved by any accepted specification.
- Whether workload feature snapshots (SPEC-010) belong in a standalone table or embedded in decision records has no canonical answer.
- The `uint64` storage pattern for `seed` and `execution_seed` has no defined representation in PostgreSQL.
- The two-phase write pattern for `decision_records` (pre-solver and post-quality-evaluation) has no authoritative description.
- No specification defines the initial database schema required for development environment initialization.
- The schema initialization requirement from ADR-004 (migration tooling deferred) creates an implicit dependency that SPEC-012 must bound.

SPEC-012 resolves these gaps. It defines the authoritative physical persistence model, determines open questions about persistence scope, establishes read/write ownership per table, and provides the schema definition required for implementation to begin.

---

# Business Value

- Enables database schema implementation to proceed without ad-hoc decisions made during development
- Establishes the canonical authority boundary between evidence record semantics (SPEC-006) and physical persistence realization (SPEC-012), preventing conflicting schema decisions during implementation
- Provides the complete table definitions required for the Worker, API, and Core to be implemented concurrently by different work streams
- Determines whether backend capability profiles require PostgreSQL storage, preventing a late-cycle architectural decision from blocking implementation
- Confirms the workload feature snapshot placement, closing the implicit gap between SPEC-003 FR-10 and SPEC-010 FR-10
- Establishes the `uint64` storage convention before multiple components implement it independently and produce divergent representations

---

# Employer Signaling

- System Design
- Distributed Systems
- Reliability Engineering
- Database Design
- Observability

Persistence schema design for an evidence-driven optimization system requires reasoning about multiple competing concerns simultaneously: referential integrity, idempotency under at-least-once delivery, write ownership across multiple concurrent writers, column-level security constraints (`execution_seed` must not appear in logs), and the distinction between mutable intermediate state and immutable terminal evidence. This specification demonstrates the ability to synthesize a coherent persistence model from a distributed set of behavioral specifications.

---

# Domain Concept

The Project DAEDALUS persistence layer is a single PostgreSQL instance (ADR-004) that serves as the system's durable state store. All components that require persistent state read from and write to this instance. No other durable persistence backend exists in the MVP architecture.

Eight entity types are persisted: routing problems, scheduler configurations, jobs, scheduler decision records, solver run records, quality evaluation records, failure records, and report metadata records. Each entity has a designated owner specification (defining its semantics) and a designated writer component (performing the database write). SPEC-012 is the authority for the physical realization of each entity; the owner specifications remain authoritative for semantics and field definitions.

The persistence model is evidence-centric. The primary concern is fidelity: every job execution must produce a complete, consistent, and retrievable evidence record set. Idempotency under at-least-once delivery (SPEC-005 FR-14) is achieved through `job_id`-keyed upsert on all Worker-written records. API writes are not idempotent by design (SPEC-008 FR-16): each submission creates a new job with new identifiers.

Two non-PostgreSQL persistence boundaries exist: the RabbitMQ queue (transient, not a persistence target) and the report volume (file storage, not a database table). The report volume's file path is referenced via the `file_path` column in `report_metadata_records` and is the only non-PostgreSQL durable artifact tracked in the schema.

---

# Requirements

### FR-1: Persistence Authority Boundary

**Description:**
SPEC-012 defines the boundary between evidence record semantics and physical persistence realization.

**SPEC-006 (Evidence Log) remains the authoritative owner of:**
- Evidence record definitions (which artifact types must exist, and why)
- Required field names and their semantic meaning
- Field ownership (which component writes which field)
- Retention policy (90-day minimum default per SPEC-006 ODR-3)
- Upsert semantics (all Worker-written tables must support `job_id`-keyed upsert)
- Terminal-state immutability rules

**SPEC-012 (Persistence Schema) is the authoritative owner of:**
- Physical table names
- Column types, including representation of uint64 values, JSONB payloads, and enum-as-text decisions
- NOT NULL constraints and nullable column semantics
- Primary key definitions
- Foreign key relationships and referential integrity rules
- Uniqueness constraints
- CHECK constraints
- The initial DDL required to initialize the schema
- The `uint64`-as-`bigint` storage convention (FR-4.3)
- The `stops`-as-JSONB normalization decision (FR-4.4)
- The decision record two-phase write contract (FR-7.4)

**When SPEC-006 and SPEC-012 appear to conflict:** SPEC-006 governs semantic intent; SPEC-012 governs physical realization. A field named differently in SPEC-006 and SPEC-012 is a defect requiring correction. A field typed differently (SPEC-006 says "structured type"; SPEC-012 says "JSONB") is SPEC-012's realization decision, not a conflict.

**Acceptance Criteria:**
- No table definition in SPEC-012 contradicts a field semantic defined in SPEC-006
- Any column added, removed, or renamed relative to SPEC-006 field definitions is explicitly documented as a realization decision with rationale
- SPEC-006 ODR-3 retention policy (90 days minimum default) applies to all Worker-written evidence tables

---

### FR-2: Backend Capability Profile Persistence Determination

**Description:**
SPEC-003 OQ-2 asks what mechanism backend capability profiles use to enter the Scheduler's registry at runtime. SPEC-012 determines whether any answer to that question requires a PostgreSQL persistence table.

**Determination: Backend capability profiles are not persisted in PostgreSQL.**

Backend capability profiles (SPEC-003 FR-4, SPEC-011 FR-4) describe backend algorithmic properties: supported size classes, latency profiles, quality profiles, and supported contract version. These properties are static within a deployment: they do not change at runtime and do not vary per-job. Persisting them to PostgreSQL would:

1. Duplicate information already present in individual backend solver specifications (future SPEC-013, SPEC-014, SPEC-015)
2. Require a registration synchronization mechanism (keeping the database and runtime registry in sync) that is more complex than a configuration-at-startup model
3. Provide no evidence value: no SPEC-006 record requires a join to a capability profiles table to be meaningful
4. Pre-empt SPEC-003 OQ-2 by implying one specific registration mechanism

**What is persisted instead:** The `backend_id` string appears in `decision_records.selected_backend_id`, `decision_records.candidate_scores` (JSONB), and `solver_run_records.backend_id`. These identifiers provide sufficient traceability to the backend specifications without requiring a separate capability profile table.

**This determination does not resolve SPEC-003 OQ-2.** The registration mechanism (how capability profiles enter the Scheduler's runtime registry) remains unresolved and is not defined by SPEC-012. The determination here is narrow: PostgreSQL is not the registration store for capability profiles, regardless of which mechanism SPEC-003 OQ-2 selects.

**Acceptance Criteria:**
- No `backend_capability_profiles` table appears in the SPEC-012 schema
- Every required `backend_id` reference in evidence records is satisfiable through the tables defined in FR-4 through FR-11
- SPEC-003 OQ-2 is explicitly not resolved by this determination

---

### FR-3: Workload Feature Snapshot Storage Determination

**Description:**
SPEC-010 FR-10 states that workload features are not persisted as standalone evidence artifacts; they are preserved via the `workload_features_snapshot` field in the Scheduler decision record (SPEC-003 FR-10). SPEC-012 confirms and formalizes this as the physical realization.

**Determination: Workload feature snapshots are embedded in `decision_records` as a JSONB column. No standalone feature evidence table exists.**

**Current SPEC-003 FR-10 dependency status:** SPEC-003 FR-10 (Accepted, current text) describes `workload_features_snapshot` as a "snapshot of all four workload features at invocation time." SPEC-010 FR-9 requires a future SPEC-003 FR-10 revision to enumerate all six SPEC-010 features plus `feature_schema_version`. That SPEC-003 revision has not yet occurred — it is tracked as a pending action item in SPEC-010 Documentation Updates Required (and restated in SPEC-012 Documentation Updates Required below). The JSONB structure defined below in this FR-3 is SPEC-012's anticipated physical realization of the six-feature-plus-version snapshot; it is not yet a realization of SPEC-003 FR-10's current accepted text, which still names four features. SPEC-012 does not present this as an already-completed alignment with SPEC-003; it is realizing the schema ahead of the pending SPEC-003 update, consistent with SPEC-010 FR-9's resolved direction.

Rationale (from SPEC-010 FR-10):
- Feature values are deterministically recoverable by re-running feature extraction on the routing problem identified by `problem_id`
- The `workload_features_snapshot` in the decision record already preserves feature values at decision time for auditability and scheduler effectiveness analysis
- A separate table would duplicate data already in `decision_records` without additional auditability value

**Schema representation:** The `workload_features_snapshot` column in `decision_records` is `jsonb NOT NULL`. The JSONB object contains all six features defined in SPEC-010 FR-3 plus `feature_schema_version`:

| JSON key | Value type | Source |
|---|---|---|
| `feature_schema_version` | integer | SPEC-010 FR-5 |
| `capacity_utilization_ratio` | float64 | SPEC-010 FR-3.1 |
| `problem_size_class` | string (`Small`, `Medium`, `Large`) | SPEC-010 FR-3.2 |
| `time_window_density` | float64 | SPEC-010 FR-3.3 |
| `average_time_window_width_seconds` | float64 | SPEC-010 FR-3.4 |
| `geographic_compactness` | float64 | SPEC-010 FR-3.5 |
| `service_time_pressure_ratio` | float64 | SPEC-010 FR-3.6 |

**Acceptance Criteria:**
- No standalone workload features table appears in the SPEC-012 schema
- `decision_records.workload_features_snapshot` is `jsonb NOT NULL`
- The JSONB object includes all six SPEC-010 features and `feature_schema_version`
- Feature values in the snapshot are queryable via PostgreSQL JSONB operators without schema migration
- SPEC-012 does not represent this six-feature-plus-version structure as an already-completed alignment with SPEC-003 FR-10's current accepted text; the pending SPEC-003 revision (SPEC-010 FR-9) is identified as an open upstream dependency

---

### FR-4: `routing_problems` Table

**Description:**
Stores the persisted representation of every routing problem submitted to the system. Routing problems are immutable after creation (SPEC-001). The API is the sole writer.

**Table name:** `routing_problems`

**Columns:**

| Column | Type | Nullable | Constraint | Source |
|---|---|---|---|---|
| `problem_id` | `uuid` | NOT NULL | PRIMARY KEY | SPEC-001 FR-1 |
| `seed` | `bigint` | NOT NULL | | SPEC-001 FR-6; see FR-4.3 |
| `vehicle_count` | `integer` | NOT NULL | CHECK (`vehicle_count >= 1`) | SPEC-001 FR-2 |
| `capacity_per_vehicle` | `integer` | NOT NULL | CHECK (`capacity_per_vehicle >= 1`) | SPEC-001 FR-2 |
| `average_vehicle_speed_kmh` | `double precision` | NOT NULL | CHECK (`average_vehicle_speed_kmh > 0`) | ODR-1; SPEC-008 FR-2 |
| `depot_latitude` | `double precision` | NOT NULL | | SPEC-001 FR-3, FR-5 |
| `depot_longitude` | `double precision` | NOT NULL | | SPEC-001 FR-3, FR-5 |
| `stops` | `jsonb` | NOT NULL | | SPEC-001 FR-4; see FR-4.4 |
| `created_at` | `timestamptz` | NOT NULL | | Set by API at creation |

**Primary key:** `problem_id`

**Foreign keys:** None. `routing_problems` is a root entity with no FK dependencies.

**Referencing tables:** `jobs.problem_id`, `decision_records.problem_id`, `solver_run_records.problem_id`

**FR-4.3: `seed` as `bigint` (uint64 storage convention):**

PostgreSQL has no native unsigned 64-bit integer type. `seed` (SPEC-001 FR-6) and `execution_seed` (SPEC-006 FR-6.2) are both `uint64` values. Both are stored as `bigint` (int64) using two's complement bit-identical storage. This convention applies to every `uint64` value in the schema.

**Architectural requirement:** The application layer at every read and write boundary is responsible for converting between `uint64` and the `bigint` two's complement representation. This conversion obligation applies uniformly across every component and language that reads or writes a `uint64`-backed column (the Worker and the API, regardless of implementation language). No PostgreSQL-side transformation is performed; the bit pattern stored is identical to the `uint64` bit pattern.

| Property | Behavior |
|---|---|
| Values 0 through 2^63-1 | Stored and retrieved without transformation |
| Values 2^63 through 2^64-1 | Stored as negative `bigint`; bit pattern is identical; application layer must cast to `uint64` on read |
| SQL ordering and comparison | Range queries on `seed` or `execution_seed` are not meaningful for values crossing the int64 boundary; these columns are identity values, not range-filterable |

**Implementation Planning Note (non-normative):** The conversion obligation above is an architectural requirement; the specific client library mechanism used to satisfy it is an implementation planning concern, not a SPEC-012 requirement. At the time of writing, the anticipated mechanisms are: for the C# control plane, Npgsql's `NpgsqlDbType.Bigint` with `ulong` mapping; for the C++ Worker, an explicit `uint64`-to-`int64` cast on write and `int64`-to-`uint64` cast on read via libpqxx. These are illustrative, not prescriptive — implementers may use any driver mechanism that satisfies the bit-identical conversion requirement above. This note must not be read as weakening that requirement.

**FR-4.4: `stops` as `jsonb` (normalization decision):**

Routing problem stops are stored as a JSONB array rather than normalized into a separate table.

| Factor | Rationale |
|---|---|
| Access pattern | Stops are always accessed as a unit with the routing problem (Core loads the full problem once per job) |
| No per-stop queries | No accepted specification defines a query that retrieves individual stops without the full problem |
| Avoids join on hot path | The Worker loads the routing problem on every job execution; a join to a stops table increases load latency without benefit |
| SPEC-001 FR-14 | Core's C++ domain representation treats the problem as a unit, not as row-accessed data |

Tradeoff: JSONB stops are not individually indexable without JSONB path operators. If future analytics require per-stop queries, normalization into a `routing_problem_stops` table is the migration path. This is deferred to post-MVP.

**Stop object structure within the JSONB array:**

| JSON key | Value type | Nullable | Constraint |
|---|---|---|---|
| `id` | integer | required | Non-negative; unique within the array (SPEC-001 FR-4) |
| `latitude` | float64 | required | [-90.0, 90.0] (SPEC-001 FR-5) |
| `longitude` | float64 | required | [-180.0, 180.0] (SPEC-001 FR-5) |
| `demand` | integer | required | Non-negative (SPEC-001 FR-4) |
| `time_window_open` | integer | optional | Seconds from route start; present iff `time_window_close` present (SPEC-001 FR-9) |
| `time_window_close` | integer | optional | Seconds from route start; must be greater than `time_window_open` (SPEC-001 FR-9) |
| `service_duration` | integer | optional | Seconds; application defaults to 0 when absent (SPEC-001 FR-16) |

**Immutability:** `routing_problems` rows are never updated or deleted after creation (SPEC-001). No UPDATE or DELETE is issued against this table.

**Acceptance Criteria:**
- `routing_problems` has no FK dependencies
- `seed` is `bigint`; the uint64 convention is documented in FR-4.3
- `stops` is `jsonb NOT NULL` containing the stop array structure defined above
- Routing problem rows are never updated or deleted after creation

---

### FR-5: `scheduler_configs` Table

**Description:**
Stores scheduler configuration records created via the API (SPEC-008 FR-10) and the seeded default configuration (SPEC-003 FR-14). The API is the sole writer. The default configuration is inserted at startup before the API accepts requests.

**Table name:** `scheduler_configs`

**Columns:**

| Column | Type | Nullable | Constraint | Source |
|---|---|---|---|---|
| `scheduler_config_id` | `uuid` | NOT NULL | PRIMARY KEY | SPEC-008 FR-10 |
| `objective_mode` | `text` | NOT NULL | CHECK (see FR-5.2) | SPEC-003 FR-6 |
| `mode_parameters` | `jsonb` | NOT NULL | DEFAULT `'{}'::jsonb` | SPEC-003 FR-6; SPEC-008 FR-10 |
| `created_at` | `timestamptz` | NOT NULL | | Set by API at creation or seed time |

**Primary key:** `scheduler_config_id`

**Foreign keys:** None. `scheduler_configs` is a root entity.

**Referencing tables:** `jobs.scheduler_config_id`

**FR-5.2: `objective_mode` CHECK constraint:**

```sql
CHECK (objective_mode IN (
    'CheapestValid',
    'FastestValid',
    'Balanced',
    'BestQuality',
    'DeadlineAware',
    'BudgetCapped',
    'Experimental'
))
```

**FR-5.3: Default configuration seeding:**

The default scheduler configuration (SPEC-003 FR-14: `Balanced` mode with equal weights) must be present in `scheduler_configs` before the API accepts its first request. It is inserted as part of schema initialization or API startup logic using ON CONFLICT DO NOTHING semantics to make the seed idempotent. See OQ-1 for the default config UUID stability mechanism.

**Immutability:** `scheduler_configs` rows are not updated or deleted after creation at MVP scope (SPEC-008 Non-Requirements).

**Acceptance Criteria:**
- `objective_mode` is constrained to the seven modes defined in SPEC-003 FR-6
- `mode_parameters` is `jsonb NOT NULL`; the value `null` is never stored (SPEC-008 FR-10)
- The default configuration is present and retrievable before any job submission is accepted

---

### FR-6: `jobs` Table

**Description:**
The central job lifecycle record. Created by the API at submission time. Updated by the Worker as the job progresses through lifecycle states (SPEC-005). The `cancellation_requested` flag is written by the API (SPEC-008 FR-11) and read by the Worker at the pre-execution check (SPEC-005 FR-12).

**Table name:** `jobs`

**Columns:**

| Column | Type | Nullable | Constraint | Source |
|---|---|---|---|---|
| `job_id` | `uuid` | NOT NULL | PRIMARY KEY | SPEC-006 FR-2; SPEC-008 FR-7 |
| `problem_id` | `uuid` | NOT NULL | FK → `routing_problems(problem_id)` | SPEC-001; SPEC-008 FR-4 |
| `scheduler_config_id` | `uuid` | NOT NULL | FK → `scheduler_configs(scheduler_config_id)` | SPEC-008 FR-4 |
| `status` | `text` | NOT NULL | CHECK (see FR-6.2) | SPEC-005; SPEC-006 FR-4.2 |
| `cancellation_requested` | `boolean` | NOT NULL | DEFAULT `false` | SPEC-005 FR-12; SPEC-008 FR-11 |
| `created_at` | `timestamptz` | NOT NULL | | Set by API; immutable after creation (SPEC-006 FR-4.3) |
| `updated_at` | `timestamptz` | NOT NULL | | Set by API at creation; updated by Worker on each status transition |
| `completed_at` | `timestamptz` | NULL | | Set by Worker when transitioning to `Completed`; null otherwise |
| `failed_at` | `timestamptz` | NULL | | Set by Worker when transitioning to `Failed`; null otherwise |

**Primary key:** `job_id`

**Foreign keys:**
- `problem_id` REFERENCES `routing_problems(problem_id)`
- `scheduler_config_id` REFERENCES `scheduler_configs(scheduler_config_id)`

**Referencing tables:** All evidence tables reference `jobs(job_id)`.

**FR-6.2: `status` CHECK constraint:**

```sql
CHECK (status IN ('Pending', 'Processing', 'Completed', 'Failed'))
```

**FR-6.3: Authorized status transitions:**

| From | To | Writer | Condition |
|---|---|---|---|
| (none) | `Pending` | API | Job creation |
| `Pending` | `Processing` | Worker | Message consumed |
| `Processing` | `Completed` | Worker | Worker lifecycle complete |
| `Processing` | `Failed` | Worker | Permanent failure |

No other transitions are authorized. The API never transitions job status after initial creation (SPEC-008 FR-9).

**FR-6.4: Terminal state cross-column CHECK constraint:**

```sql
CHECK (
    (status = 'Completed' AND completed_at IS NOT NULL AND failed_at IS NULL)
    OR (status = 'Failed' AND failed_at IS NOT NULL AND completed_at IS NULL)
    OR (status IN ('Pending', 'Processing') AND completed_at IS NULL AND failed_at IS NULL)
)
```

**FR-6.5: `cancellation_requested` write ownership:**

- API sets to `false` at creation (SPEC-008 FR-4)
- API sets to `true` when a cancellation request is accepted for a non-terminal job (SPEC-008 FR-11)
- Worker reads this field at the pre-execution check (SPEC-005 FR-12)
- Worker never writes `cancellation_requested` (SPEC-005 FR-12)

**Acceptance Criteria:**
- `status` is constrained to the four states in FR-6.2
- The terminal state CHECK constraint prevents contradictory `completed_at`/`failed_at` values
- `cancellation_requested` defaults to `false` at creation and is never set to null
- `created_at` is set once by the API and is never updated

---

### FR-7: `decision_records` Table

**Description:**
Stores the Scheduler's decision for each job, including backend selection rationale, predicted outcomes, workload feature snapshot, and — after quality evaluation completes — the actual outcome and hindsight quality. This record is written in two phases (FR-7.4).

**Table name:** `decision_records`

**Columns:**

| Column | Type | Nullable | Constraint | Source |
|---|---|---|---|---|
| `decision_id` | `uuid` | NOT NULL | PRIMARY KEY | SPEC-003 FR-10 |
| `job_id` | `uuid` | NOT NULL | FK → `jobs(job_id)`, UNIQUE | SPEC-006 FR-5 |
| `problem_id` | `uuid` | NOT NULL | FK → `routing_problems(problem_id)` | SPEC-003 FR-10 |
| `objective_mode` | `text` | NOT NULL | | SPEC-003 FR-10 |
| `workload_features_snapshot` | `jsonb` | NOT NULL | | SPEC-003 FR-10; SPEC-010 FR-10; see FR-3 |
| `decision_status` | `text` | NOT NULL | | SPEC-003 FR-10 |
| `selected_backend_id` | `text` | NULL | | SPEC-003 FR-10; null when no backend selected |
| `candidate_scores` | `jsonb` | NOT NULL | DEFAULT `'[]'::jsonb` | SPEC-003 FR-10 |
| `rejection_reasons` | `jsonb` | NOT NULL | DEFAULT `'{}'::jsonb` | SPEC-003 FR-10 |
| `predicted_latency` | `double precision` | NULL | | SPEC-003 FR-10; null when no backend selected |
| `predicted_cost` | `double precision` | NULL | | SPEC-003 FR-10; null when no backend selected |
| `predicted_quality` | `text` | NULL | | SPEC-003 FR-10; null when no backend selected |
| `confidence_score` | `double precision` | NULL | | SPEC-003 FR-10; null when no backend selected |
| `timestamp` | `timestamptz` | NOT NULL | | Decision creation time |
| `actual_outcome` | `jsonb` | NULL | | SPEC-003 FR-10; SPEC-007 FR-4; written in Phase 2 |
| `hindsight_quality` | `double precision` | NULL | | SPEC-003 FR-10; SPEC-007 FR-7; written in Phase 2 |

**Primary key:** `decision_id`

**Unique constraint:** `UNIQUE (job_id)` — one decision record per job.

**Foreign keys:**
- `job_id` REFERENCES `jobs(job_id)`
- `problem_id` REFERENCES `routing_problems(problem_id)`

**Referencing tables:** `solver_run_records(decision_id)`, `quality_evaluation_records(decision_id)`

**FR-7.3: `actual_outcome` JSONB structure:**

`actual_outcome` stores an `ActualOutcomeClassification` object per SPEC-007 FR-4. The expected JSON structure:

| JSON key | Value type | Nullable within object | Notes |
|---|---|---|---|
| `solver_outcome` | string | required | From SolverResponse.outcome; always non-null per SPEC-007 FR-13 |
| `solution_present` | boolean | required | Always non-null per SPEC-007 FR-13 |
| `time_window_constrained` | boolean | required | Always non-null per SPEC-007 FR-13 |
| `time_window_feasible` | boolean | nullable | Null when simulation not performed |
| `time_window_violation_count` | integer | nullable | Null when simulation not performed |

**FR-7.4: Two-phase write contract:**

The decision record is written in two distinct phases within the Worker lifecycle.

**Phase 1** (before solver dispatch, SPEC-005 FR-6): The Worker writes the complete decision record with Scheduler-produced fields. `actual_outcome` and `hindsight_quality` are NULL at this point. This write uses INSERT with ON CONFLICT (job_id) DO UPDATE to support idempotent re-execution.

**Phase 2** (after quality evaluation, SPEC-005 FR-16): The Worker issues an UPDATE to set `actual_outcome` and `hindsight_quality` on the existing row, using the `QualityEvaluationResult` from Core (SPEC-007 FR-9). The UPDATE is keyed on `decision_id`.

Phase 2 is the only authorized UPDATE to `decision_records`. All other columns are immutable after Phase 1. The record is fully immutable after Phase 2 once the job reaches terminal state.

Phase 2 is idempotent: if the Worker re-executes due to message redelivery, the same values are re-computed (SPEC-007 FR-10 determinism guarantees identical output from identical inputs) and re-applied without producing divergent records.

**Acceptance Criteria:**
- `UNIQUE (job_id)` ensures exactly one decision record per job
- `workload_features_snapshot` is `jsonb NOT NULL` and contains all six SPEC-010 features plus `feature_schema_version`
- `actual_outcome` and `hindsight_quality` are nullable; null is the initial state after Phase 1; they are set in Phase 2
- Phase 2 is the only authorized UPDATE; all other columns are immutable after Phase 1 write

---

### FR-8: `solver_run_records` Table

**Description:**
Stores the evidence record for one solver execution attempt. Captures the reproducibility tuple `(problem_id, execution_seed, backend_id, contract_version)` and the complete solver response fields. `execution_seed` is stored exclusively in this table and must never appear in structured logs, API responses, report content, or trace span attributes (SPEC-006 FR-6.4, SPEC-005 Security Considerations, ADR-010).

**Table name:** `solver_run_records`

**Columns:**

| Column | Type | Nullable | Constraint | Source |
|---|---|---|---|---|
| `solver_run_id` | `uuid` | NOT NULL | PRIMARY KEY | SPEC-006 FR-6.2 |
| `job_id` | `uuid` | NOT NULL | FK → `jobs(job_id)`, UNIQUE | SPEC-006 FR-2 |
| `decision_id` | `uuid` | NOT NULL | FK → `decision_records(decision_id)` | SPEC-006 FR-6.2 |
| `problem_id` | `uuid` | NOT NULL | FK → `routing_problems(problem_id)` | Reproducibility tuple |
| `backend_id` | `text` | NOT NULL | | SPEC-004 FR-2; reproducibility tuple |
| `contract_version` | `integer` | NOT NULL | CHECK (`contract_version >= 1`) | SPEC-004 FR-2; reproducibility tuple |
| `execution_seed` | `bigint` | NOT NULL | | SPEC-005 FR-7; ADR-010 ODR-4; see FR-4.3 and FR-8.3 |
| `solver_outcome` | `text` | NOT NULL | CHECK (see FR-8.2) | SPEC-004 FR-4 |
| `execution_duration_ms` | `bigint` | NULL | | SPEC-004 FR-6; null when statistics absent |
| `solution_count` | `integer` | NULL | | SPEC-004 FR-6; optional |
| `structural_validation_status` | `text` | NOT NULL | | SPEC-005 FR-11 |
| `route_plan` | `jsonb` | NULL | | SPEC-004 FR-5; null when no solution produced |
| `failure_detail` | `jsonb` | NULL | | SPEC-004 FR-8; null when no solver failure detail |
| `extension_metadata` | `jsonb` | NOT NULL | DEFAULT `'{}'::jsonb` | SPEC-004 FR-13 |
| `persisted_at` | `timestamptz` | NOT NULL | | SPEC-006 FR-6.2; written by Worker; see FR-8.6 |

**Primary key:** `solver_run_id`

**Unique constraint:** `UNIQUE (job_id)` — one solver run record per job per execution attempt. Idempotent re-execution replaces via upsert (SPEC-005 FR-14).

**Foreign keys:**
- `job_id` REFERENCES `jobs(job_id)`
- `decision_id` REFERENCES `decision_records(decision_id)`
- `problem_id` REFERENCES `routing_problems(problem_id)`

**FR-8.2: `solver_outcome` CHECK constraint:**

```sql
CHECK (solver_outcome IN (
    'Succeeded',
    'Infeasible',
    'Timeout',
    'Cancelled',
    'Failed',
    'ContractViolation'
))
```

**Attribution:** This column stores values from two distinct authoritative sources, not one:
- `Succeeded`, `Infeasible`, `Timeout`, `Cancelled`, `Failed` are the `SolverOutcome` enumeration values defined by SPEC-004 FR-4. These are backend-reported (Solver-level) classifications: the solver backend itself determines and reports which of these five values applies.
- `ContractViolation` is not a SPEC-004 `SolverOutcome` value and is never reported by a solver backend. It is the Worker's own (Worker-level) classification, assigned when a SolverResponse fails the Worker's post-receipt structural validation (SPEC-005 FR-11, SPEC-008 FR-9), overriding whatever outcome the backend reported.

The column is realized as a single `text` field for query simplicity, but a reader must not assume all six values share the same authoritative source. Five are Solver-authoritative (SPEC-004); one is Worker-authoritative (SPEC-005). This specification does not change the SPEC-004 `SolverOutcome` enumeration or the SPEC-005 `ContractViolation` classification; it only clarifies which source each stored value traces back to.

**FR-8.3: `execution_seed` security obligation:**

`execution_seed` is stored as `bigint NOT NULL` using the uint64 convention from FR-4.3. This column must not appear in:
- Structured log events at any level
- OTel span attributes
- API responses at any endpoint
- Report content in any section
- `extension_metadata` values passed through to any output

Access to this column is restricted to the Worker's evidence persistence path and to authorized database administrators. Because `execution_seed = RoutingProblem.seed` (ADR-010 ODR-4), the routing problem's `seed` column in `routing_problems` carries the same sensitivity and must also not appear in logs or API responses.

**FR-8.4: `route_plan` JSONB structure:**

`route_plan` stores a `RoutePlan` per SPEC-004 FR-5: an array of route entries, each an ordered sequence of stop IDs. Null when the solver produced no route plan (per SPEC-011 FR-7.1: route plan is absent under `Infeasible` and `Failed` outcomes; optional under `Timeout` and `Cancelled`).

**FR-8.5: `failure_detail` JSONB structure:**

`failure_detail` stores a `SolverFailureDetail` per SPEC-004 FR-8, with `failure_code` and `failure_message`. Null when `solver_outcome = Succeeded`. May be present under `Failed`, `Timeout`, and `Cancelled` for diagnostic purposes.

**FR-8.6: `execution_statistics` realization decision:**

SPEC-006 FR-6.2 defines `execution_statistics` as a single field whose value is the `ExecutionStatistics` object defined by SPEC-004 FR-6 (`execution_duration_ms`, `solution_count`). SPEC-012 does not realize `execution_statistics` as a nested JSONB column. Instead it decomposes the object into two individual scalar columns on `solver_run_records`: `execution_duration_ms` (`bigint`, nullable) and `solution_count` (`integer`, nullable).

Rationale:
- `ExecutionStatistics` has exactly two fields (SPEC-004 FR-6); a nested JSONB wrapper adds no structural benefit over two scalar columns.
- `execution_duration_ms` is referenced directly by observability queries (FR-17.4) and benchmarking comparisons across backends (SPEC-004 Observability Requirements); scalar columns avoid JSONB path operators on a hot analytical field.
- Flattening is consistent with how the rest of the reproducibility tuple (`problem_id`, `execution_seed`, `backend_id`, `contract_version`) is represented as individual scalar columns rather than a nested structure.

This is a SPEC-012 physical realization decision under the FR-1 authority boundary: SPEC-006 FR-6.2 and SPEC-004 FR-6 remain authoritative for the field semantics and the fact that `solution_count` is optional; SPEC-012 owns the decomposition into columns.

**`persisted_at` naming reconciliation:** SPEC-006 FR-6.2 names the Worker-written persistence timestamp `persisted_at`. An earlier draft of this table used `created_at` for the same column. Per FR-1 ("A field named differently in SPEC-006 and SPEC-012 is a defect requiring correction"), the column is named `persisted_at` to match SPEC-006 FR-6.2 exactly; this is not treated as an independent realization decision.

**Immutability:** `solver_run_records` rows are written via upsert and are immutable after the job reaches terminal state.

**Acceptance Criteria:**
- `UNIQUE (job_id)` enables `job_id`-keyed upsert idempotency
- `execution_seed` is `bigint NOT NULL`; never appears in logs, API responses, spans, or reports
- `solver_outcome` is constrained to the six values in FR-8.2
- `route_plan` and `failure_detail` are nullable JSONB columns
- The full reproducibility tuple `(problem_id, execution_seed, backend_id, contract_version)` is represented by columns in this table
- `execution_duration_ms` and `solution_count` are realized as individual scalar columns, not a nested `execution_statistics` JSONB object (FR-8.6)
- The persistence timestamp column is named `persisted_at`, matching SPEC-006 FR-6.2
- `solver_outcome = 'ContractViolation'` is documented as a Worker-level (SPEC-005) classification distinct from the five Solver-level (SPEC-004) `SolverOutcome` values sharing the same column (FR-8.2)

---

### FR-9: `quality_evaluation_records` Table

**Description:**
Stores the complete quality evaluation result for a job execution. Written by the Worker after Core quality evaluation completes (SPEC-005 FR-16, SPEC-007 FR-9). Contains both the full `QualityEvaluationResult` fields and the `actual_outcome` that is also written to `decision_records` in Phase 2. The duplication of `actual_outcome` across both tables is intentional (SPEC-007 FR-6 FR-9.3).

**Table name:** `quality_evaluation_records`

**Columns:**

| Column | Type | Nullable | Constraint | Source |
|---|---|---|---|---|
| `quality_evaluation_id` | `uuid` | NOT NULL | PRIMARY KEY | SPEC-006 FR-7 |
| `job_id` | `uuid` | NOT NULL | FK → `jobs(job_id)`, UNIQUE | SPEC-006 FR-7 |
| `decision_id` | `uuid` | NOT NULL | FK → `decision_records(decision_id)` | SPEC-007 FR-9 |
| `evaluation_status` | `text` | NOT NULL | CHECK (see FR-9.2) | SPEC-006 FR-7 |
| `evaluation_schema_version` | `integer` | NOT NULL | CHECK (`evaluation_schema_version >= 1`) | SPEC-007 FR-11 |
| `evaluated_at` | `timestamptz` | NULL | | SPEC-007 FR-12; null when evaluation not invoked |
| `route_simulation_performed` | `boolean` | NOT NULL | | SPEC-007 FR-12 |
| `actual_outcome` | `jsonb` | NOT NULL | | SPEC-007 FR-9; always present; see FR-9.3 |
| `quality_metrics` | `jsonb` | NULL | | SPEC-007 FR-6; null when simulation not performed |
| `hindsight_quality` | `double precision` | NULL | | SPEC-007 FR-7; null when simulation not performed |
| `regret_analysis` | `jsonb` | NOT NULL | | SPEC-007 FR-8; always present; fields within may be null |
| `evaluation_metadata` | `jsonb` | NOT NULL | | SPEC-007 FR-12; always present |
| `created_at` | `timestamptz` | NOT NULL | | Written by Worker |

**Primary key:** `quality_evaluation_id`

**Unique constraint:** `UNIQUE (job_id)` — one quality evaluation record per job. Idempotent re-execution replaces via upsert (SPEC-005 FR-14, SPEC-007 FR-10 determinism).

**Foreign keys:**
- `job_id` REFERENCES `jobs(job_id)`
- `decision_id` REFERENCES `decision_records(decision_id)`

**FR-9.2: `evaluation_status` CHECK constraint:**

```sql
CHECK (evaluation_status IN ('Completed', 'Skipped', 'Failed'))
```

`Completed`: evaluation ran and produced results. `Skipped`: no route plan was present; evaluation was not invoked (SPEC-007 FR-2). `Failed`: evaluation was invoked but experienced an infrastructure failure (SPEC-007 FR-13).

**FR-9.3: Intentional `actual_outcome` duplication:**

`actual_outcome` appears in both `quality_evaluation_records.actual_outcome` and `decision_records.actual_outcome`. This duplication is intentional per SPEC-007 FR-6: the decision record copy serves scheduler effectiveness analysis (which consumers compare predictions against actuals in the decision record); the quality evaluation record copy serves detailed evidence inspection and evidence log queries. Both values are identical (computed from the same in-memory `QualityEvaluationResult`). Neither copy is derived from the other; both are written by the Worker from the same source.

**FR-9.4: `quality_metrics` JSONB content and `violated_stop_ids` boundary:**

When non-null, `quality_metrics` contains a `QualityMetrics` object per SPEC-007 FR-6. This includes the `violated_stop_ids` field (a list of stop ID integers). `violated_stop_ids` is stored in `quality_metrics` within PostgreSQL. It must not appear in log events, API responses, or report content (SPEC-007 Security Considerations). Storage in the database is authorized; application-layer output is not.

**FR-9.5: `decision_id` and `route_simulation_performed` realization decisions:**

Two columns in `quality_evaluation_records` are not named in SPEC-006 FR-7.3's minimum field table and are documented here as SPEC-012 physical realization decisions, per FR-1.

- **`decision_id`** (`uuid NOT NULL`, FK → `decision_records(decision_id)`): SPEC-006 FR-7.3 does not name a `decision_id` field on the quality evaluation record. SPEC-007 FR-3 establishes that Core quality evaluation is invoked with the `scheduler_decision` (the decision record) as an input, and SPEC-007 FR-8 `RegretAnalysis` is computed from that decision record's predicted fields. SPEC-012 adds `decision_id` as an explicit foreign key so that the correlation between a quality evaluation record and the decision record it was evaluated against — already implied by SPEC-007's invocation contract — is enforceable and joinable without relying on a transitive join through `jobs`. SPEC-007 remains authoritative for why the decision record is an evaluation input; SPEC-012 owns the decision to expose that linkage as a direct FK column.
- **`route_simulation_performed`** (`boolean NOT NULL`): SPEC-007 FR-12 defines `route_simulation_performed` as a field of `EvaluationMetadata`, which SPEC-006 FR-7.3 persists as part of `evaluation_metadata`. SPEC-012 promotes this single field out of the `evaluation_metadata` JSONB object into a standalone top-level column, in addition to its presence within `evaluation_metadata`. Rationale: `route_simulation_performed` is needed to directly answer observability questions (e.g., what fraction of evaluations actually ran simulation, FR-17.4) without a JSONB path extraction on every query. SPEC-007 FR-12 remains authoritative for the field's semantics and origin within `EvaluationMetadata`; SPEC-012 owns the decision to duplicate it as a standalone column for queryability.

Both additions preserve the SPEC-006/SPEC-007 ownership boundary: neither column changes the meaning of any SPEC-006- or SPEC-007-defined field; both are additive physical realizations that make existing semantic relationships directly queryable.

**Immutability:** `quality_evaluation_records` rows are written via upsert and are immutable after the job reaches terminal state.

**Acceptance Criteria:**
- `UNIQUE (job_id)` enables `job_id`-keyed upsert idempotency
- `actual_outcome` is `jsonb NOT NULL`; always present per SPEC-007 FR-13 (input-derived fields guaranteed non-null even on infrastructure failure)
- `quality_metrics` and `hindsight_quality` are nullable; null when simulation was not performed
- `regret_analysis` and `evaluation_metadata` are `jsonb NOT NULL`
- `evaluation_schema_version` is present and `>= 1`
- `decision_id` and `route_simulation_performed` are documented as SPEC-012 realization decisions with traceability to SPEC-007 (FR-9.5)

---

### FR-10: `failure_records` Table

**Description:**
Stores the structured failure record for jobs that fail before the Worker can produce a complete evidence record (SPEC-006 FR-8). `failure_records` are written only for jobs whose terminal state is `Failed`. Jobs that reach `Completed` with a failed solver outcome do not produce a `failure_record`; their solver failure is captured in `solver_run_records.failure_detail` and `solver_run_records.solver_outcome`.

**Table name:** `failure_records`

**Columns:**

| Column | Type | Nullable | Constraint | Source |
|---|---|---|---|---|
| `failure_record_id` | `uuid` | NOT NULL | PRIMARY KEY | SPEC-006 FR-8 |
| `job_id` | `uuid` | NOT NULL | FK → `jobs(job_id)`, UNIQUE | SPEC-006 FR-8 |
| `failure_stage` | `text` | NOT NULL | CHECK (see FR-10.2) | SPEC-006 FR-8 |
| `failure_class` | `text` | NOT NULL | | SPEC-006 FR-8 |
| `failure_detail` | `text` | NULL | | SPEC-006 FR-8; human-readable diagnostic |
| `is_retryable` | `boolean` | NOT NULL | | SPEC-006 FR-8 |
| `failed_at` | `timestamptz` | NOT NULL | | SPEC-006 FR-8 |

**Primary key:** `failure_record_id`

**Unique constraint:** `UNIQUE (job_id)` — one failure record per Failed job.

**Foreign keys:**
- `job_id` REFERENCES `jobs(job_id)`

**No FK to `decision_records`:** A failure at the `Load` stage produces no decision record (scheduling never occurred). A failure at the `Schedule` stage may produce a decision record (Scheduler ran but returned a terminal error). The `failure_records` table does not enforce an FK to `decision_records` because decision record existence depends on the failure stage. The `failure_stage` value is sufficient to determine whether a decision record exists for the same `job_id`.

**FR-10.2: `failure_stage` CHECK constraint:**

```sql
CHECK (failure_stage IN ('Load', 'Schedule', 'Invocation'))
```

| Stage | Trigger examples | Decision record exists? |
|---|---|---|
| `Load` | Core validation rejection; problem load failure; feature extraction failure | No |
| `Schedule` | NoEligibleSolver; Scheduler configuration error | Yes (with terminal decision_status) |
| `Invocation` | Unrecoverable pre-solver failure not covered by Load or Schedule stages | Depends on failure point |

**FR-10.2a: Mapping to SPEC-005 Worker-internal lifecycle stages:**

`failure_stage` values (`Load`, `Schedule`, `Invocation`) are the classification vocabulary defined by SPEC-006 FR-8.2/FR-8.3; SPEC-012 realizes them as the `failure_stage` column. SPEC-006's three-value vocabulary does not use the same names as SPEC-005 FR-2's nine named Worker-internal execution stages (Loading, Scheduling, Persisting-Decision, Executing, Validating-Response, Evaluating, Persisting-Results, Reporting, Completing). The table below maps each `failure_stage` value to the specific SPEC-005 stage(s) and FR-13 permanent-failure conditions it covers, so that a reviewer can verify any persisted `failure_stage` value against the authoritative SPEC-005 Worker lifecycle model without interpretation.

| `failure_stage` value | SPEC-005 Worker-internal stage(s) (FR-2) | SPEC-005 FR-13 permanent-failure conditions covered |
|---|---|---|
| `Load` | Loading (FR-4); also covers malformed-message rejection (FR-3), which precedes Loading | Routing problem record not found; scheduler configuration record not found; corrupt routing problem record; malformed job message |
| `Schedule` | Scheduling (FR-5) | Core validation rejection (ADR-009); `NoEligibleSolver`; `InvalidConfiguration` |
| `Invocation` | Between Persisting-Decision (FR-6) and Executing (FR-9) — i.e., the pre-dispatch contract version check (FR-8), which has no dedicated named stage in SPEC-005 FR-2 | Pre-dispatch contract version mismatch (FR-8) |

This mapping is additive: it does not change the `failure_stage` vocabulary defined by SPEC-006, and it does not redefine SPEC-005's named execution stages. It only establishes the correspondence between the two vocabularies for verification purposes.

**FR-10.3: `Failed` vs `Completed` with failed solver outcome:**

| Terminal state | Where failure is captured | Table |
|---|---|---|
| `Failed` | `failure_records` | Pre-solver permanent failure; Worker could not complete orchestration |
| `Completed` with `solver_outcome = Failed` | `solver_run_records.failure_detail` | Solver execution failure; Worker completed its orchestration |

This distinction preserves the SPEC-005 lifecycle invariant: `Completed` means the Worker finished its orchestration responsibilities; `Failed` means the Worker could not complete them.

**Immutability:** `failure_records` rows are written once and never updated.

**Acceptance Criteria:**
- `UNIQUE (job_id)` ensures exactly one failure record per Failed job
- No `failure_records` row exists for a job whose `jobs.status = Completed`
- `failure_stage` is constrained to the three values in FR-10.2
- Every `failure_stage` value is traceable to a specific SPEC-005 Worker-internal lifecycle stage without interpretation, per the mapping in FR-10.2a
- No FK to `decision_records` is enforced

---

### FR-11: `report_metadata_records` Table

**Description:**
Stores the report artifact metadata for jobs that reached the Reporting stage (SPEC-009). Written by the Worker using the `report_id` and `file_path` returned by the Report Generator (SPEC-009 FR-3). The `file_path` column is the API's authoritative file location reference for report serving (SPEC-008 FR-13, SPEC-009 FR-7). Supports `job_id`-keyed upsert on message redelivery (SPEC-009 FR-8).

**Table name:** `report_metadata_records`

**Columns:**

| Column | Type | Nullable | Constraint | Source |
|---|---|---|---|---|
| `report_metadata_id` | `uuid` | NOT NULL | PRIMARY KEY | Internal PK; not exposed externally |
| `job_id` | `uuid` | NOT NULL | FK → `jobs(job_id)`, UNIQUE | SPEC-006 FR-9.2; upsert key |
| `report_id` | `uuid` | NOT NULL | UNIQUE | SPEC-009 FR-10; API retrieval key (SPEC-008 FR-13) |
| `report_format` | `text` | NOT NULL | DEFAULT `'html'` | SPEC-009 FR-10 |
| `generated_at` | `timestamptz` | NULL | | SPEC-009 FR-10; null when `generation_status = Failed` |
| `generation_status` | `text` | NOT NULL | CHECK (see FR-11.2) | SPEC-009 FR-10 |
| `file_path` | `text` | NULL | | SPEC-009 FR-7, FR-10; null when `generation_status = Failed` |
| `report_schema_version` | `integer` | NOT NULL | CHECK (`report_schema_version >= 1`) | SPEC-009 FR-11 |
| `solver_outcome` | `text` | NOT NULL | | SPEC-009 FR-10; denormalized from `solver_run_records` |
| `created_at` | `timestamptz` | NOT NULL | | Set at first write; immutable |
| `updated_at` | `timestamptz` | NOT NULL | | Updated on each upsert (redelivery) |

**Primary key:** `report_metadata_id`

**Unique constraints:**
- `UNIQUE (job_id)` — one active metadata record per job; upsert replaces on redelivery
- `UNIQUE (report_id)` — each report UUID maps to exactly one metadata record

**Foreign keys:**
- `job_id` REFERENCES `jobs(job_id)`

**FR-11.2: `generation_status` CHECK constraint:**

```sql
CHECK (generation_status IN ('Completed', 'Failed'))
```

**FR-11.3: Dual primary key rationale:**

`report_metadata_records` uses an internal `report_metadata_id` UUID as the primary key, separate from `report_id`. This is because `report_id` is reassigned on every re-execution (SPEC-009 FR-8): a redelivered message produces a new `report_id` that overwrites the old one via `job_id`-keyed upsert. Using `report_id` as the PK would require DELETE and re-INSERT on every redelivery. The internal `report_metadata_id` remains stable across redelivery upserts; only `report_id`, `file_path`, `generated_at`, and `updated_at` change on update.

**FR-11.4: `file_path` construction constraint:**

`file_path` stores the absolute path returned by the Report Generator (SPEC-009 FR-7). The API locates the physical report file by reading this column — never by constructing a path from caller-supplied request parameters. This eliminates path traversal risk in the report serving path (SPEC-008 FR-13, SPEC-009 Security Considerations).

**FR-11.5: `solver_outcome` denormalization:**

`solver_outcome` is denormalized from `solver_run_records.solver_outcome` into the report metadata record (SPEC-009 FR-10). This supports metadata queries without requiring a join to `solver_run_records`. The stored value must match `solver_run_records.solver_outcome` for the same `job_id`; both are written by the Worker from the same in-memory state.

**Acceptance Criteria:**
- `UNIQUE (job_id)` enables `job_id`-keyed upsert idempotency
- `UNIQUE (report_id)` ensures each report UUID maps to exactly one record
- `file_path` is null only when `generation_status = Failed`
- The API reads `file_path` from this table to serve report files; no file path is constructed from caller-supplied values

---

### FR-12: Upsert Semantics and Idempotency

**Description:**
All Worker-written evidence tables support `job_id`-keyed upsert (SPEC-006 FR-2.2, SPEC-005 FR-14). This enables idempotent re-execution under at-least-once delivery: if the Worker processes the same job message twice due to NACK or redelivery before ACK, the second execution produces equivalent evidence and replaces (not duplicates) the prior records.

**FR-12.1: Upsert target tables and conflict behavior:**

| Table | Upsert key | On conflict behavior |
|---|---|---|
| `decision_records` | `job_id` | Replace all Scheduler-produced columns (Phase 1). Phase 2 UPDATE uses `decision_id`. |
| `solver_run_records` | `job_id` | Replace all columns |
| `quality_evaluation_records` | `job_id` | Replace all columns |
| `failure_records` | `job_id` | Replace all columns |
| `report_metadata_records` | `job_id` | Replace `report_id`, `file_path`, `generated_at`, `updated_at`; retain `report_metadata_id`, `created_at` |

**FR-12.2: API-written tables are not upsert targets:**

`routing_problems`, `scheduler_configs`, and the initial `jobs` creation are written once by the API and are not subject to Worker upsert. Duplicate key conflicts are not expected on these tables from the submission path.

**FR-12.3: Idempotency guarantee:**

The idempotency guarantee rests on two properties:
1. Evidence computed from identical inputs is identical: solver execution is reproducible (ADR-010, SPEC-004 FR-11), quality evaluation is deterministic (SPEC-007 FR-10), and report rendering is deterministic (SPEC-009 FR-12)
2. `job_id`-keyed upserts replace prior records rather than creating duplicates

On message redelivery, the Worker re-executes the full job and re-writes all evidence records. The resulting records are semantically equivalent to and replace any prior records for the same `job_id`.

**Acceptance Criteria:**
- Every Worker-written table supports `job_id`-keyed upsert
- No Worker-written table creates a second row for the same `job_id` on re-execution
- API-written tables are exempt from Worker upsert obligations

---

### FR-13: Record Lifecycle and Mutability Rules

**Description:**
Defines which records are mutable, under what conditions, and after what events they become immutable.

**FR-13.1: Mutability summary by table:**

| Table | Mutable after creation? | Authorized mutations | Immutable after |
|---|---|---|---|
| `routing_problems` | No | None | Creation |
| `scheduler_configs` | No | None | Creation |
| `jobs` | Yes (limited) | `status`, `cancellation_requested`, `updated_at`, `completed_at`, `failed_at` | No hard cutoff; terminal state is the de facto boundary |
| `decision_records` | Yes (Phase 2 only) | `actual_outcome`, `hindsight_quality` | Phase 2 UPDATE completion; terminal job state |
| `solver_run_records` | No (after upsert settles) | Upsert replaces before terminal state | Terminal job state |
| `quality_evaluation_records` | No (after upsert settles) | Upsert replaces before terminal state | Terminal job state |
| `failure_records` | No | None | Creation |
| `report_metadata_records` | Partial (on upsert) | `report_id`, `file_path`, `generated_at`, `updated_at` | Terminal job state |

**FR-13.2: Post-terminal-state write prohibition:**

Once `jobs.status` is `Completed` or `Failed`, no application code should UPDATE or re-upsert evidence records for that job. This invariant is enforced at the application layer; no PostgreSQL trigger or row-level lock enforces it at MVP scope.

**FR-13.3: `jobs.cancellation_requested` exception:**

The API may write `cancellation_requested = true` to the `jobs` table while the job is in `Pending` or `Processing` state. This is the one case where the API modifies a field on the `jobs` row after creation.

**Acceptance Criteria:**
- `routing_problems` and `scheduler_configs` rows are never updated or deleted after creation
- `decision_records` Phase 2 UPDATE touches only `actual_outcome` and `hindsight_quality`
- No evidence table row is modified after the corresponding job reaches terminal state under normal execution
- `cancellation_requested` may be set to `true` by the API on non-terminal jobs

---

### FR-14: Schema Initialization

**Description:**
Defines the initial database schema required for MVP development environment startup. ADR-004 defers schema migration tooling selection; until that selection is made, schema initialization is a DDL-script-based procedure applied once at environment creation.

**FR-14.1: Schema components:**

The complete initial schema consists of:
1. Eight table definitions (FR-4 through FR-11 DDL)
2. Default scheduler configuration seed (FR-5.3)
3. No initial data in any other table

**FR-14.2: Required table creation order:**

Tables must be created in dependency order to satisfy FK constraints:

1. `routing_problems` (no FK dependencies)
2. `scheduler_configs` (no FK dependencies)
3. `jobs` (FKs: `routing_problems`, `scheduler_configs`)
4. `decision_records` (FKs: `jobs`, `routing_problems`)
5. `solver_run_records` (FKs: `jobs`, `decision_records`, `routing_problems`)
6. `quality_evaluation_records` (FKs: `jobs`, `decision_records`)
7. `failure_records` (FKs: `jobs`)
8. `report_metadata_records` (FKs: `jobs`)

**FR-14.3: Schema version visibility:**

SPEC-012 does not define a schema_versions table or migration tracking table. ADR-004 explicitly defers migration tooling. Schema version visibility at MVP scope is provided by this specification document as the authoritative schema source, and by the DDL script applied at initialization as an implementation planning artifact. A schema_versions table is deferred to the migration tooling selection (OQ-2).

**FR-14.4: Initialization idempotency:**

The schema initialization procedure must be safe to re-run. All CREATE TABLE statements use `IF NOT EXISTS`. The default scheduler configuration seed uses `ON CONFLICT DO NOTHING`.

**Acceptance Criteria:**
- Tables are created in the dependency order defined in FR-14.2
- The default scheduler configuration is present before the API accepts any request
- The schema initialization procedure is idempotent
- No migration tooling is selected or configured by this specification

---

### FR-15: Read/Write Ownership Map

**Description:**
Defines which component reads and writes each table. This is the authoritative source for access pattern expectations at implementation time.

| Table | API Writes | API Reads | Worker Writes | Worker Reads |
|---|---|---|---|---|
| `routing_problems` | CREATE (once at submission) | No | No | Yes (problem load at job consumption) |
| `scheduler_configs` | CREATE + seed default | Yes (configuration retrieval endpoints) | No | Yes (configuration resolution at job consumption) |
| `jobs` | CREATE initial row; UPDATE `cancellation_requested` | Yes (status polling; cancellation terminal-state check) | UPDATE `status`, `updated_at`, `completed_at`, `failed_at` | Yes (lifecycle state check; cancellation flag check before solver dispatch) |
| `decision_records` | No | No | Phase 1 UPSERT; Phase 2 UPDATE (`actual_outcome`, `hindsight_quality`) | Yes (loaded after Phase 1 to pass predicted values to quality evaluation) |
| `solver_run_records` | No | Yes (`solver_outcome` for status response; SPEC-008 FR-8) | UPSERT | No |
| `quality_evaluation_records` | No | No | UPSERT | No |
| `failure_records` | No | Yes (`failure_class` for status response; SPEC-008 FR-8) | INSERT (once) | No |
| `report_metadata_records` | No | Yes (report discovery; `file_path` for report file serving) | UPSERT | No |

**FR-15.2: Core does not access PostgreSQL:**

Core (the C++ library) does not read from or write to PostgreSQL directly. Core receives the routing problem and scheduler configuration from the Worker as in-memory C++ structs (SPEC-005 Assumption 3). All database I/O is performed by the Worker on Core's behalf. Feature extraction, scheduling, quality evaluation, and report generation all operate on in-memory data passed by the Worker.

**Acceptance Criteria:**
- No component other than the API writes to `routing_problems` or `scheduler_configs`
- No component other than the Worker writes to `decision_records`, `solver_run_records`, `quality_evaluation_records`, `failure_records`, or `report_metadata_records`
- Core issues no direct database queries

---

### FR-16: Retention Policy

**Description:**
All Worker-written evidence tables are subject to the 90-day minimum default retention policy defined in SPEC-006 ODR-3. Retention enforcement is an operational concern deferred to post-MVP, but the schema must not prevent retention-compliant deletion.

**FR-16.1: Retention-compliant deletion order:**

Rows must be deleted in reverse FK dependency order:

1. `report_metadata_records` (FK: `jobs`)
2. `failure_records` (FK: `jobs`)
3. `quality_evaluation_records` (FKs: `jobs`, `decision_records`)
4. `solver_run_records` (FKs: `jobs`, `decision_records`, `routing_problems`)
5. `decision_records` (FKs: `jobs`, `routing_problems`)
6. `jobs` (FKs: `routing_problems`, `scheduler_configs`)

This deletion order satisfies retention-compliant deletion for the six evidence tables described above. It terminates at `jobs`. `routing_problems` is excluded from this order (see FR-16.4).

**FR-16.2: `scheduler_configs` retention:**

`scheduler_configs` rows are not subject to evidence retention deletion. They are system configuration records, not evidence records, and should persist for the lifetime of the deployment.

**FR-16.3: No CASCADE DELETE:**

FK constraints must not use CASCADE DELETE. Retention deletion must be explicit and ordered. CASCADE DELETE would allow a single DELETE on `jobs` to silently remove all associated evidence records, which is incompatible with the deliberate, auditable deletion model required for a 90-day evidence store.

**FR-16.4: `routing_problems` is excluded from evidence-retention lifecycle processing:**

`routing_problems` rows are not deleted as part of the FR-16.1 retention-compliant deletion order, and the SPEC-006 ODR-3 90-day minimum retention policy does not apply to them.

SPEC-006 FR-15.4 is explicit that "the routing problem referenced by `problem_id` has independent retention semantics defined by SPEC-001. Evidence Log retention does not imply routing problem retention." SPEC-006 FR-15.5 further notes that the routing problem persists independently of evidence record deletion. The Evidence Log (SPEC-006) is not the retention authority for `routing_problems`; SPEC-012, as the physical schema owner, does not assign itself that authority either.

SPEC-001 Non-Requirements is the authoritative source for routing problem retention: "No routing problem deletion or archiving. A routing problem is retained indefinitely after submission. No deletion, expiry, or archiving mechanism exists in the MVP." There is no gap to resolve: SPEC-001 already defines the answer (indefinite retention, no deletion mechanism at MVP scope). SPEC-012 conforms to this by omitting `routing_problems` from the evidence-retention deletion order in FR-16.1.

Because `routing_problems` rows are never deleted at MVP scope, `jobs.problem_id` and `decision_records.problem_id` foreign keys to `routing_problems` never face a retention-driven deletion ordering concern. If a future SPEC-001 revision introduces routing problem deletion or archiving, SPEC-012 FR-16.1 must be revised to incorporate `routing_problems` into the deletion order at that time.

**Acceptance Criteria:**
- FK constraints do not use CASCADE DELETE
- Retention-compliant deletion is executable in the order defined in FR-16.1 without FK violations
- `scheduler_configs` is excluded from evidence retention lifecycle
- `routing_problems` is excluded from the FR-16.1 evidence-retention deletion order; its retention is governed exclusively by SPEC-001 (indefinite retention, no deletion mechanism at MVP scope)

---

### FR-17: Persistence Observability

**Description:**
Defines the observability requirements for persistence operations, consistent with ADR-006 (OpenTelemetry SDK, Prometheus, structured JSON logs) and ADR-011 (W3C TraceContext).

**FR-17.1: Structured log events for persistence failures:**

Every persistence write failure must produce a structured log event at ERROR level. Required fields:

| Field | Description |
|---|---|
| `job_id` | Present on all Worker-written record failures |
| `table_name` | The table on which the write was attempted |
| `operation` | `INSERT`, `UPDATE`, or `UPSERT` |
| `error_type` | PostgreSQL error class or connection failure category |

`execution_seed` must not appear in any persistence failure log event.

**FR-17.2: `worker.evidence.persist` span attributes:**

All Worker persistence operations occur within the `worker.evidence.persist` span (SPEC-005 FR-16). This span must carry:

| Attribute | Description |
|---|---|
| `job_id` | Correlation key |
| `artifacts_written` | List of table names written in this invocation |
| `persistence_outcome` | `success`, `partial_failure`, or `failure` |

**FR-17.3: Schema version observability:**

`evaluation_schema_version` (in `quality_evaluation_records`) and `report_schema_version` (in `report_metadata_records`) are stored as integer columns and are directly queryable. No additional instrumentation is required to observe schema version distribution across evidence records.

**FR-17.4: Operational queries answerable without instrumentation changes:**

The schema defined in FR-4 through FR-11 must directly support the following queries without schema migration:

| Question | Key columns |
|---|---|
| Job counts by status | `jobs.status` |
| Complete evidence record set for a job | All tables joined on `job_id` |
| Jobs with incomplete Phase 2 write | `decision_records WHERE actual_outcome IS NULL` joined to non-null `solver_run_records` |
| `hindsight_quality` by backend and size class | `solver_run_records.backend_id`, `quality_evaluation_records.hindsight_quality`, `decision_records.workload_features_snapshot->>'problem_size_class'` |
| Failure stage distribution | `failure_records.failure_stage`, COUNT |
| Report generation success rate | `report_metadata_records.generation_status`, COUNT |

**Acceptance Criteria:**
- Every persistence write failure produces a structured log event with required fields
- `execution_seed` never appears in any log event
- The `worker.evidence.persist` span carries `job_id`, `artifacts_written`, and `persistence_outcome`
- Schema version fields are directly queryable from evidence tables without joins

---

### FR-18: Data Integrity Constraints

**Description:**
Defines constraints that prevent schema-level integrity violations beyond what is captured in individual table definitions.

**FR-18.1: Cross-table terminal state consistency (application-enforced):**

When `jobs.status = Completed`: a `solver_run_records` row must exist for that `job_id`. When `jobs.status = Failed`: a `failure_records` row must exist for that `job_id`. These cross-table consistency rules are enforced at the application layer by the Worker lifecycle. PostgreSQL does not enforce multi-table business rules at the schema level without triggers, which are not used at MVP scope.

**FR-18.2: `actual_outcome` cross-table identity (application-enforced):**

When both `decision_records.actual_outcome` and `quality_evaluation_records.actual_outcome` are non-null for the same `job_id`, their values must be identical (SPEC-007 FR-6 duplication identity invariant). This is guaranteed by the Worker writing both from the same in-memory `QualityEvaluationResult` (SPEC-007 FR-9). No PostgreSQL constraint enforces this.

**FR-18.3: `execution_seed` single-location constraint:**

`execution_seed` appears only in `solver_run_records`. No other table, column, log event, API response, or report output may contain this value. This is an application-level constraint enforced by the Worker and API implementations.

**FR-18.4: `file_path` construction constraint:**

No application component constructs `file_path` values from caller-supplied input. File paths appear only as values returned by the Report Generator (SPEC-009 FR-9) and stored in `report_metadata_records.file_path`. The API reads `file_path` from this column exclusively (SPEC-008 FR-13).

**Acceptance Criteria:**
- Application code enforces that every `Completed` job has a `solver_run_records` row
- Application code enforces that every `Failed` job has a `failure_records` row
- No column named `execution_seed` exists outside `solver_run_records`
- No application code constructs `file_path` from request parameters

---

# Non-Requirements

- SPEC-012 does not define physical database engine tuning parameters, connection pool sizes, buffer settings, or query plan hints
- SPEC-012 does not define index definitions beyond what is implied by PRIMARY KEY and UNIQUE constraints; indexing strategy is implementation planning
- SPEC-012 does not define table partitioning, tablespace assignment, or sharding strategies
- SPEC-012 does not define replication topology or standby configuration
- SPEC-012 does not define backup or recovery procedures
- SPEC-012 does not define migration tooling selection or schema evolution governance (ADR-004 defers this)
- SPEC-012 does not define a schema versions table or migration tracking mechanism
- SPEC-012 does not define production operational runbooks or database administration procedures
- SPEC-012 does not define the RabbitMQ queue message format (owned by SPEC-008 FR-5 and ADR-003)
- SPEC-012 does not define the report volume directory structure beyond noting that `file_path` is the authoritative reference (SPEC-009 FR-7 owns that definition)
- SPEC-012 does not define evidence record semantics; those remain owned by SPEC-006
- SPEC-012 does not define retention or deletion semantics for `routing_problems`; that authority belongs exclusively to SPEC-001 (FR-16.4)
- SPEC-012 does not resolve SPEC-003 OQ-2 (capability profile registration mechanism)

---

# Assumptions

1. PostgreSQL is the sole durable persistence backend per ADR-004. No secondary databases, caches, or external stores are introduced at MVP scope.
2. RabbitMQ is transient messaging infrastructure, not a persistence store. No RabbitMQ state is part of the SPEC-012 schema.
3. The `average_vehicle_speed_kmh` field is a routing problem attribute per ODR-1 (referenced in SPEC-007 FR-3 and SPEC-008 FR-2). SPEC-001 will formally introduce this field in a future revision. SPEC-012 includes it as a `routing_problems` column based on this accepted dependency.
4. All UUID values are generated by the creating component (API or Worker) before persistence. PostgreSQL does not generate UUIDs; the application layer supplies them on every write.
5. The Worker and API connect to the same PostgreSQL instance. No connection routing or read replica differentiation is applied at MVP scope.
6. The report volume is filesystem storage mounted in the Worker and API containers. Its existence and path are infrastructure configuration. SPEC-012 references report file paths via `report_metadata_records.file_path`; it does not define the volume mount mechanism.
7. SPEC-003 OQ-2 (capability profile registration mechanism) will be resolved before any backend is registered and before Worker implementation begins. SPEC-012 FR-2's determination (no PostgreSQL table for capability profiles) does not depend on which mechanism SPEC-003 OQ-2 selects.
8. `stops` JSONB in `routing_problems` is not queried at the individual stop level at MVP scope. Future analytics requirements may require normalization into a `routing_problem_stops` table; that is post-MVP work.
9. `problem_size_class` thresholds (SPEC-001 FR-7: Small = 1-25 stops, Medium = 26-75, Large = 76+) are stable within MVP scope. Changes to these thresholds do not require a SPEC-012 schema change; they affect feature computation (SPEC-010) and JSONB values stored in `workload_features_snapshot`.
10. `routing_problems` rows are retained indefinitely with no deletion or archiving mechanism at MVP scope, per SPEC-001 Non-Requirements. SPEC-012 FR-16.1's evidence-retention deletion order does not include `routing_problems` on this basis.

---

# Constraints

1. All persistence must use PostgreSQL per ADR-004. No additional durable stores are introduced.
2. Schema must be initializable from a DDL script without migration tooling (ADR-004 migration tooling deferred).
3. `execution_seed` must be stored only in `solver_run_records` and must never appear in log events, API responses, report content, or trace span attributes (SPEC-005 Security Considerations; SPEC-006 FR-6.4; ADR-010).
4. All Worker-written tables must support `job_id`-keyed upsert for idempotency under at-least-once delivery (SPEC-006 FR-2.2, SPEC-005 FR-14).
5. FK constraints must not use CASCADE DELETE; deletion must be explicit and ordered per FR-16.1.
6. `uint64` values (`seed`, `execution_seed`) are stored as `bigint` using two's complement bit-identical representation per FR-4.3. This convention must be applied consistently by every component performing the conversion (the API and the Worker), regardless of which database client library each uses. Specific driver mechanisms are an implementation planning concern (FR-4.3 Implementation Planning Note).
7. `stops` JSONB in `routing_problems` is the only denormalized stop representation at MVP scope (FR-4.4). Individual stop normalization is deferred to post-MVP.
8. Backend capability profiles are not persisted in PostgreSQL (FR-2). If SPEC-003 OQ-2 resolves to a database-backed registry, SPEC-012 must be revised.

---

# Inputs

| Input | Source | Format | Persistence target |
|---|---|---|---|
| Routing problem submission | HTTP caller via API validation | JSON per SPEC-008 FR-2 | `routing_problems` |
| Job creation | API (derived from submission) | UUIDs, status, timestamps | `jobs` |
| Scheduler configuration creation | HTTP caller via API validation | JSON per SPEC-008 FR-10 | `scheduler_configs` |
| Cancellation request | HTTP caller via API | Job ID in path | `jobs.cancellation_requested` |
| Decision record | Core → Worker in-memory | DecisionRecord per SPEC-003 FR-10 | `decision_records` (Phase 1) |
| Quality evaluation result | Core → Worker in-memory | QualityEvaluationResult per SPEC-007 FR-9 | `quality_evaluation_records`; `decision_records` Phase 2 |
| Solver response | Solver backend → Worker in-memory | SolverResponse per SPEC-004 FR-3 | `solver_run_records` |
| Report generation result | Report Generator → Worker in-memory | Structured result per SPEC-009 FR-3 | `report_metadata_records` |
| Pre-solver failure | Worker (internal) | Failure stage, class, detail | `failure_records` |
| Job lifecycle transitions | Worker (internal) | Status, timestamps | `jobs` |

---

# Outputs

| Output | Consumer | Source table |
|---|---|---|
| Job status response | HTTP caller via API | `jobs`, `solver_run_records` (`solver_outcome`), `failure_records` (`failure_class`), `report_metadata_records` (`report_available`) |
| Scheduler config list and retrieval | HTTP caller via API | `scheduler_configs` |
| Report metadata response | HTTP caller via API | `report_metadata_records` |
| Report file path for serving | API (file I/O) | `report_metadata_records.file_path` |
| Routing problem for Worker | Worker at job consumption | `routing_problems` |
| Scheduler configuration for Worker | Worker at job consumption | `scheduler_configs` |
| Cancellation flag for Worker | Worker at pre-execution check | `jobs.cancellation_requested` |

---

# Failure Modes

### PostgreSQL Unavailable During Job Creation

**Condition:** The API cannot write to `routing_problems` or `jobs` during submission.
**Behavior:** The API rolls back any partial writes and returns HTTP 500. No queue message is published (SPEC-008 FR-4: persistence before publication). The job is never created.
**Schema impact:** No orphaned records. Both the routing problem and the job record are committed atomically or neither is (SPEC-008 FR-4).

---

### PostgreSQL Unavailable During Worker Evidence Persistence

**Condition:** The Worker cannot write evidence records after solver execution.
**Behavior:** The Worker NACKs the message; it is redelivered for re-execution (SPEC-005 FR-14). On re-execution, the Worker re-runs the solver and re-writes all evidence via upsert.
**Schema impact:** Possible partial evidence records from the failed write attempt. Upsert semantics on re-execution overwrite partial records with complete ones. The job remains in `Processing` state until persistence succeeds.

---

### Decision Record Phase 2 UPDATE Failure

**Condition:** The Worker successfully writes `solver_run_records` and `quality_evaluation_records` but cannot UPDATE `decision_records` with `actual_outcome` and `hindsight_quality`.
**Behavior:** The Worker NACKs the message; re-execution re-runs the full job. On re-execution, Phase 2 UPDATE is re-applied with identical values (SPEC-007 FR-10 determinism).
**Schema impact:** `decision_records.actual_outcome` and `decision_records.hindsight_quality` remain null until the UPDATE succeeds. A `Completed` job with a non-null `quality_evaluation_records` row but null `decision_records.actual_outcome` indicates an interrupted Phase 2 write (see OQ-3).

---

### Report Metadata Persistence Failure After Successful File Write

**Condition:** The Report Generator writes the file to the report volume but the Worker cannot write `report_metadata_records` to PostgreSQL.
**Behavior:** The report file exists on the volume but is not discoverable via the API (no metadata record). The Worker NACKs; re-execution generates a new report with a new `report_id` and re-attempts metadata persistence. The prior report file remains on the volume as an orphan (SPEC-009 OQ-2).
**Schema impact:** No metadata record for the orphaned report file. The orphaned file is an operational concern, not a schema integrity issue.

---

### FK Violation Attempt

**Condition:** A Worker write attempts to insert a record with a `job_id` that does not exist in `jobs`.
**Behavior:** PostgreSQL rejects the insert with a FK violation error. The Worker logs a structured error. This indicates a Worker implementation defect: the job record must always exist in `jobs` before any evidence record is written for that job.

---

# Architectural Impact

| Component | Impact |
|---|---|
| API Layer | Yes — API writes `routing_problems`, `scheduler_configs`, and `jobs` (initial row + cancellation update); reads `jobs`, `solver_run_records`, `failure_records`, `report_metadata_records`, `scheduler_configs` |
| Worker | Yes — Worker writes all evidence tables; reads `routing_problems`, `scheduler_configs`, `jobs` at consumption and at pre-execution cancellation check |
| Core | None — Core does not access PostgreSQL directly; receives data from Worker as in-memory C++ structs |
| Scheduler | None — Scheduler operates entirely in-memory; its output (decision record) is persisted by the Worker |
| Feature Extraction | None — Feature values are embedded in `decision_records.workload_features_snapshot`; no standalone table |
| Report Generator | None — Report Generator writes files to the report volume; does not access PostgreSQL; returns file path to Worker |
| Observability | Yes — evidence tables are directly queryable for effectiveness analysis; schema version columns are observable without instrumentation changes |
| Security | Yes — `execution_seed` storage restriction (FR-8.3); `violated_stop_ids` storage boundary (FR-9.4); `file_path` construction constraint (FR-18.4) |
| Deployment | Yes — Docker Compose PostgreSQL container must initialize the schema; API container must seed the default scheduler config at startup |

---

# Testability

1. **Schema: FK enforcement** -- Attempt to INSERT a `jobs` row with a `problem_id` that does not exist in `routing_problems`. Verify PostgreSQL rejects the insert with a FK violation error.

2. **Schema: `status` CHECK constraint** -- Attempt to INSERT a `jobs` row with `status = 'Running'`. Verify PostgreSQL rejects the insert.

3. **Schema: terminal state CHECK** -- Attempt to UPDATE `jobs` to `status = 'Completed'` without setting `completed_at`. Verify PostgreSQL rejects the update.

4. **Schema: `objective_mode` CHECK** -- Attempt to INSERT a `scheduler_configs` row with `objective_mode = 'Unknown'`. Verify PostgreSQL rejects the insert.

5. **Schema: UNIQUE on decision_records** -- INSERT two `decision_records` rows with the same `job_id`. Verify the second insert fails with a unique constraint violation.

6. **Schema: UNIQUE on report_metadata_records** -- INSERT two `report_metadata_records` rows with the same `report_id` but different `job_id`. Verify the second insert fails.

7. **Upsert idempotency: decision_records** -- Given a `decision_records` row for `job_id = X`, perform a Phase 1 upsert for the same `job_id` with updated `decision_status`. Verify exactly one row exists for `job_id = X` with the updated value.

8. **Upsert idempotency: solver_run_records** -- Given a `solver_run_records` row for `job_id = X`, perform an upsert for the same `job_id`. Verify exactly one row exists for `job_id = X`.

9. **Upsert idempotency: report_metadata_records redelivery** -- Given a `report_metadata_records` row for `job_id = X` with `report_id = R1`, perform an upsert for the same `job_id` with `report_id = R2`. Verify exactly one row exists for `job_id = X` with `report_id = R2`. Verify `report_metadata_id` is unchanged (internal PK stability).

10. **Two-phase write: decision_records** -- Write a `decision_records` row with `actual_outcome = NULL`. Perform the Phase 2 UPDATE for the same `decision_id`. Verify `actual_outcome` is populated and all other columns are unchanged.

11. **uint64 round-trip: seed** -- Insert a `routing_problems` row with `seed` equal to 2^63 (the value that maps to the minimum negative int64). Read the value back and verify the bit pattern is identical to the written value when interpreted as uint64.

12. **uint64 round-trip: execution_seed** -- Insert a `solver_run_records` row with `execution_seed` equal to 2^64 - 1 (stored as -1 in int64). Read the value back and verify the bit pattern is identical when interpreted as uint64.

13. **JSONB: workload_features_snapshot** -- Insert a `decision_records` row with all six SPEC-010 features and `feature_schema_version` in `workload_features_snapshot`. Query using the JSONB path operator (`->>'geographic_compactness'`). Verify the returned value matches the inserted value.

14. **JSONB: stops** -- Insert a `routing_problems` row with a multi-stop `stops` array. Query a specific stop's demand using JSONB path operators. Verify the returned value matches the inserted value.

15. **Retention deletion order** -- Given a complete evidence record set for `job_id = X` (rows in the six evidence tables covered by FR-16.1: `report_metadata_records`, `failure_records`, `quality_evaluation_records`, `solver_run_records`, `decision_records`, `jobs`), delete rows in the order defined in FR-16.1. Verify no FK violation occurs at any step. Verify the `routing_problems` row for the referenced `problem_id` is not deleted and remains queryable (FR-16.4).

16. **`execution_seed` isolation** -- Query the schema definition (information_schema or pg_catalog). Verify that `execution_seed` appears as a column name only in `solver_run_records` and in no other table.

17. **`file_path` not constructed from request input** -- Code review: verify no application code path constructs a file system path using the value of a `report_id` query parameter or path variable directly. Verify all file serving operations read `file_path` from `report_metadata_records`.

18. **Default config seeding** -- After schema initialization, query `scheduler_configs`. Verify exactly one row exists for the default Balanced configuration. Verify `mode_parameters` contains the three weight fields summing to 1.0 within floating-point tolerance.

19. **Integration: full lifecycle evidence** -- Execute a complete job from API submission through Worker completion with a `Succeeded` solver outcome. Query all eight tables using the `job_id`. Verify rows exist in `routing_problems`, `scheduler_configs`, `jobs`, `decision_records`, `solver_run_records`, `quality_evaluation_records`, and `report_metadata_records`. Verify `failure_records` has no row for this `job_id`.

20. **Integration: Load stage failure evidence** -- Execute a job that fails at the Load stage (Core validation rejection). Query all eight tables. Verify `jobs.status = Failed`, `failure_records` has one row with `failure_stage = Load`, and `decision_records`, `solver_run_records`, `quality_evaluation_records`, `report_metadata_records` have no rows for this `job_id`.

21. **Integration: Schedule stage failure evidence** -- Execute a job that fails at the Schedule stage (NoEligibleSolver). Verify `jobs.status = Failed`, `decision_records` has one row (Scheduler ran), `failure_records` has one row with `failure_stage = Schedule`, and `solver_run_records` has no row for this `job_id`.

22. **Observability query: incomplete Phase 2 writes** -- Given a test scenario where Phase 2 UPDATE did not complete, verify that the query `SELECT d.job_id FROM decision_records d JOIN solver_run_records s ON d.job_id = s.job_id WHERE d.actual_outcome IS NULL` returns the `job_id` of the affected job.

---

# Observability Requirements

## Operational Questions

The persistence schema and associated observability must answer the following questions without schema migration or code change:

1. How many jobs are in each status (Pending, Processing, Completed, Failed) at any point in time?
2. For a given `job_id`, what is the complete evidence record: routing problem, scheduler decision, solver outcome, quality evaluation, and report?
3. What fraction of Completed jobs have quality evaluation records? What fraction have report metadata records?
4. What is the `hindsight_quality` for a given `backend_id` across all Completed jobs with `Succeeded` solver outcome, grouped by `problem_size_class`?
5. How many jobs failed at each failure stage?
6. Which jobs have a non-null `solver_run_records` row but null `decision_records.actual_outcome`, indicating an incomplete Phase 2 write?
7. What is the distribution of `evaluation_schema_version` and `report_schema_version` across evidence records?

All seven questions are answerable through direct SQL queries on the tables defined in FR-4 through FR-11.

## Span and Log Requirements

- Every persistence write failure must produce a structured log event with `job_id`, `table_name`, `operation`, and `error_type` (FR-17.1)
- The `worker.evidence.persist` span must carry `job_id`, `artifacts_written`, and `persistence_outcome` (FR-17.2)
- No persistence log event may contain `execution_seed` (FR-8.3)

---

# Security Considerations

## `execution_seed` Isolation

`execution_seed` is a `bigint NOT NULL` column in `solver_run_records`. It must not appear in structured log events, API responses, OTel span attributes, report content, or `extension_metadata` values passed to any output channel. This restriction is established by SPEC-005 Security Considerations, SPEC-006 FR-6.4, and ADR-010. Because `execution_seed = RoutingProblem.seed` (ADR-010 ODR-4), the routing problem's `seed` column carries the same sensitivity and must also be excluded from logs and API responses.

## `violated_stop_ids` Storage Boundary

`violated_stop_ids` (a field within `quality_evaluation_records.quality_metrics` JSONB) may be stored in PostgreSQL as part of the `quality_metrics` object. It must not appear in log events, API responses, or report content (SPEC-007 Security Considerations). Storage in the database is authorized. Application-layer output is not. The violation count and `time_window_feasible` flag in `actual_outcome` provide sufficient operational signal without exposing individual stop identifiers.

## Raw Problem Data Isolation

Geographic coordinates, full stop arrays, and demand values are stored in `routing_problems.stops` as JSONB. These values must not appear in structured log events or OTel span attributes (SPEC-001 Security Considerations, SPEC-005 Security Considerations, SPEC-008 FR-17). They may be read from the database for execution purposes but must not be written to any output channel beyond the in-process SolverRequest.

## `file_path` Path Traversal Prevention

`report_metadata_records.file_path` stores the absolute path returned by the Report Generator. The API reads this value to serve report files. The API must not construct file paths from caller-supplied request parameters. All file serving reads `file_path` exclusively from this column, eliminating path traversal risk (SPEC-008 FR-13, SPEC-009 Security Considerations).

## `extension_metadata` Safety

`solver_run_records.extension_metadata` stores backend-specific metadata from `SolverResponse.extension_metadata` (SPEC-004 FR-13). This JSONB field must not contain routing problem raw data (coordinates, demands, time windows, or stop arrays). Per SPEC-011 FR-7.4: extension_metadata must not contain routing problem data. The Worker must validate this constraint before persisting.

## Database Credentials

PostgreSQL credentials must be injected via environment variables. Credentials must not be committed to source control or appear in log events. Connection strings must not appear in error responses (SPEC-008 FR-14).

---

# Performance Considerations

## Write Volume Per Job

Each job execution in the normal success path produces approximately 8 to 10 write operations: `jobs` (two to three writes: creation, Processing transition, Completed transition), `decision_records` (Phase 1 UPSERT, Phase 2 UPDATE), `solver_run_records` (one upsert), `quality_evaluation_records` (one upsert), `report_metadata_records` (one upsert). This is acceptable for a single-instance MVP workload.

## JSONB Read Performance

JSONB fields in evidence tables (`workload_features_snapshot`, `quality_metrics`, `regret_analysis`, `evaluation_metadata`, `actual_outcome`) are written and read as complete documents on the hot path. Selective key extraction for analytics queries using JSONB path operators (`->`, `->>`) is a non-critical path operation and is acceptable at MVP query volumes.

## Primary Key Lookup Dominance

All hot-path queries are primary key or unique index lookups: `jobs` by `job_id` (status polling), `report_metadata_records` by `job_id` (report discovery) and by `report_id` (API retrieval endpoint lookup), `routing_problems` by `problem_id` (Worker job load), `scheduler_configs` by `scheduler_config_id` (existence check at submission). No table scans occur on the hot path.

## `stops` JSONB Size

At MVP Large-class scale (76+ stops), the `stops` JSONB array for one routing problem is estimated at 5-15 KB. This is loaded once per job execution by the Worker. No performance concern at MVP scale.

## No Numeric Latency Targets

No specific latency targets are defined for persistence operations at this stage. Persistence latency is observable via the `worker.evidence.persist` span duration and the `api.job.submission.duration` histogram (SPEC-008 FR-17). Targets are an implementation planning concern.

---

# Documentation Updates Required

- **SPEC-006 FR-7 (Quality Evaluation Record Schema):** SPEC-006 FR-7 defines minimum quality evaluation record fields. SPEC-012 FR-9 provides the full physical schema realization as defined by SPEC-007 FR-6, FR-7, FR-8, FR-11, and FR-12. SPEC-006 FR-7 should reference SPEC-012 FR-9 as the authoritative physical schema definition.

- **SPEC-006 FR-9 (Report Metadata Schema):** SPEC-006 FR-9.3 defers schema extension to the Report Generator Specification. SPEC-009 FR-10 defines the extended schema. SPEC-012 FR-11 provides the physical realization. SPEC-006 FR-9 should reference SPEC-012 FR-11 as the authoritative physical schema.

- **SPEC-003 FR-10 (`workload_features_snapshot` field) — pending, not yet completed:** SPEC-003 FR-10's current accepted text describes `workload_features_snapshot` as a snapshot of four workload features. SPEC-010 FR-9 requires a future SPEC-003 FR-10 revision to include all six SPEC-010 features and `feature_schema_version` in the snapshot definition. That revision has not occurred as of this document. SPEC-012 FR-3 defines the physical JSONB structure anticipating this six-feature-plus-version set (see FR-3's dependency status note) and depends on the pending SPEC-003 revision landing before the schema and SPEC-003 FR-10 are fully consistent. The SPEC-003 FR-10 follow-on update remains the responsibility of the action items in SPEC-010 Documentation Updates Required; SPEC-012 does not perform that update.

- **SPEC-011 `Blocks` field:** SPEC-011 metadata lists SPEC-012, SPEC-013, and SPEC-014 as the blocked individual backend solver specifications. SPEC-012 is now the Persistence Schema specification. SPEC-011's `Blocks` field should be updated to reference the correct spec IDs for the three backend solver specifications (to be determined when those specs are assigned IDs).

- **ADR-004 (PostgreSQL Persistence):** ADR-004 defers schema migration tooling selection. SPEC-012 OQ-2 surfaces this as a blocking implementation planning item. ADR-004 should reference SPEC-012 as the specification that defines the schema requiring migration tooling.

- **docs/architecture.md:** The runtime flow description and required OTel spans list are consistent with the read/write ownership model defined in FR-15. No changes to architecture.md are required.

---

# Open Questions

### OQ-1: Default Scheduler Configuration UUID Stability

**Question:** What mechanism ensures the default scheduler configuration's `scheduler_config_id` is stable across database re-creations, so that jobs referencing the default config remain resolvable after the development database is recreated?

**Why it matters:** If the default configuration's UUID is generated randomly at seed time, recreating the database produces a different UUID. Any `jobs.scheduler_config_id` references to the prior UUID become orphaned. For a local development environment recreated frequently, this causes FK violations on re-initialization if prior job records are preserved.

**Options:**
1. Use a deterministic well-known UUID hardcoded in the seed script (simplest; safe for MVP)
2. Perform a lookup at startup for an existing Balanced configuration with matching weights and reuse its UUID if found
3. Accept that database re-creation invalidates all prior job records (appropriate if the development database is treated as ephemeral)

**Classification:** Implementation Planning Decision.

**Blocking:** Blocking for schema initialization implementation. Not blocking for SPEC-012 Draft status.

---

### OQ-2: Schema Migration Tooling

**Question:** What tooling is used to apply schema changes as SPEC-012 evolves during implementation and as additional requirements are discovered?

**Why it matters:** ADR-004 explicitly defers migration tooling selection. SPEC-012 defines the initial schema as a DDL script. If any column is added, renamed, or typed differently during MVP implementation, a migration procedure is needed. Without selected tooling, schema changes risk being applied inconsistently across development environments.

**Candidate approaches:** Flyway or Liquibase (established migration frameworks); manual versioned DDL scripts with a lightweight tracking table; EF Core migrations (if the API's C# ORM supports the schema); a minimal custom migration runner in the Docker Compose entrypoint.

**Classification:** ADR Candidate. If migration tooling selection requires a meaningful architectural tradeoff (for example, EF Core ownership of schema vs. independent tooling), an ADR is the appropriate governance artifact.

**Blocking:** Not blocking for Draft status. Blocking for implementation once the initial schema is declared stable.

---

### OQ-3: Phase 2 Incomplete Write Detection and Remediation

**Question:** Should the schema or initialization include a database view or documented query to surface jobs where the Phase 2 UPDATE to `decision_records` did not complete (i.e., `solver_run_records` row exists but `decision_records.actual_outcome IS NULL`)?

**Why it matters:** An incomplete Phase 2 write produces a detectable inconsistency: a Completed job has a quality evaluation record but null `actual_outcome` in the decision record. This condition may need operator visibility and a defined remediation path (re-triggering the UPDATE from the persisted quality evaluation record).

**Recommendation:** Define a standard SQL diagnostic query in implementation planning documentation. A database view is optional but not required at MVP scope. Remediation tooling is post-MVP.

**Classification:** Implementation Planning Decision.

**Blocking:** Not blocking for Draft status.

---

### OQ-4: Capability Profile Schema Contingency on SPEC-003 OQ-2

**Question:** If SPEC-003 OQ-2 resolves to a database-backed capability profile registry, does SPEC-012 require revision to add a `backend_capability_profiles` table?

**Current position:** SPEC-012 FR-2 determines that backend capability profiles are not persisted in PostgreSQL. This determination holds until SPEC-003 OQ-2 resolves. If the resolution selects database-backed registration, SPEC-012 must be revised to define a `backend_capability_profiles` table and its FK relationships to `decision_records` and `solver_run_records`.

**Classification:** Contingent on SPEC-003 OQ-2 resolution. No action required until OQ-2 is resolved.

**Blocking:** Not blocking. SPEC-012 FR-2 is the authoritative determination for the current scope.

---

# Acceptance Checklist

- [x] Problem is clearly defined
- [x] Domain concept is defined
- [x] Persistence authority boundary between SPEC-006 and SPEC-012 is explicit (FR-1)
- [x] Backend capability profile persistence determination is made and justified (FR-2)
- [x] Workload feature snapshot storage determination is made and justified (FR-3)
- [x] `routing_problems` table is fully defined with columns, types, constraints, and normalization rationale (FR-4)
- [x] `scheduler_configs` table is fully defined (FR-5)
- [x] `jobs` table is fully defined including status transitions, terminal state CHECK, and `cancellation_requested` write ownership (FR-6)
- [x] `decision_records` table is fully defined including two-phase write contract and `workload_features_snapshot` JSONB schema (FR-7)
- [x] `solver_run_records` table is fully defined including reproducibility tuple, uint64 convention, and `execution_seed` security obligation (FR-8)
- [x] `quality_evaluation_records` table is fully defined including intentional `actual_outcome` duplication and `violated_stop_ids` storage boundary (FR-9)
- [x] `failure_records` table is fully defined including distinction from solver-level failure and rationale for no FK to `decision_records` (FR-10)
- [x] `report_metadata_records` table is fully defined including dual-PK rationale, `file_path` safety, and upsert-on-redelivery behavior (FR-11)
- [x] Upsert semantics and idempotency are defined for all Worker-written tables (FR-12)
- [x] Record lifecycle and mutability rules are defined per table (FR-13)
- [x] Schema initialization requirements are defined including table creation order (FR-14)
- [x] Read/write ownership map is defined per table and per component (FR-15)
- [x] Retention policy and deletion order are defined with no CASCADE DELETE (FR-16)
- [x] Persistence observability requirements are defined (FR-17)
- [x] Data integrity constraints are defined (FR-18)
- [x] Non-requirements are documented
- [x] Assumptions are explicit
- [x] Failure modes are defined
- [x] Security considerations address `execution_seed`, `violated_stop_ids`, raw problem data, `file_path`, and `extension_metadata`
- [x] Performance considerations are documented
- [x] Documentation updates are identified
- [ ] OQ-1 (default config UUID stability) classified as Implementation Planning Decision, blocking for implementation
- [ ] OQ-2 (schema migration tooling) classified as ADR Candidate, not blocking for Draft
- [ ] OQ-3 (Phase 2 incomplete write detection) classified as Implementation Planning Decision, not blocking
- [ ] OQ-4 (capability profile schema contingency) classified as contingent on SPEC-003 OQ-2, not blocking

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-18) are implemented and acceptance criteria pass
- The eight-table schema is initialized in the development PostgreSQL instance from a validated DDL script applied in the dependency order defined in FR-14.2
- The default scheduler configuration is seeded and retrievable via `GET /v1/scheduler-configs` (SPEC-008 FR-10)
- OQ-1 (default config UUID stability) is resolved and the initialization script implements the chosen mechanism
- OQ-2 (schema migration tooling) is resolved via ADR, and all schema changes after initial creation are applied using the selected tooling
- All test contracts in the Testability section pass against the initialized schema
- `execution_seed` does not appear in any log event, API response, span attribute, or report output, verified by integration test
- The `worker.evidence.persist` span carries `artifacts_written` and `persistence_outcome` on every Worker execution
- A complete job execution (API submission through Worker completion with Succeeded outcome) produces retrievable rows in `routing_problems`, `scheduler_configs`, `jobs`, `decision_records`, `solver_run_records`, `quality_evaluation_records`, and `report_metadata_records` with no FK violations
- Engineering review passes
- Specification status is updated to Accepted

The feature is not complete simply because the DDL script runs without errors.
