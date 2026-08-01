# Phase 0 Implementation Decisions

**Date:** 2026-08-01

**Owner:** Darkhorse286

**Status:** Approved

These decisions were approved by the Project Owner as part of Phase 0 Closure. They close the six blocking implementation gaps identified in the Implementation Readiness Review and establish the concrete technical baseline required before implementation can begin.

---

## Decision 1: C++ Toolchain

**C++20 is the C++ standard for Daedalus Core, Daedalus Worker, and the CLI. CMake is the build system.**

**Context:** ADR-001 selected C++ as the implementation language for Daedalus Core and Daedalus Worker but explicitly deferred the C++ standard version and build system to implementation planning. SPEC-016 (CLI) also uses C++. The IRR identified the deferred toolchain choice as BG-1 (Blocking Gap 1).

**Decisions:**

- **C++ Standard: C++20.** C++20 provides ranges, concepts, designated initializers, `std::span`, `std::format`, `std::jthread`, and the structured bindings, `std::optional`, `std::variant`, `std::string_view`, `if constexpr`, and filesystem library features from C++17. All compiler versions targeted by the Docker Compose build environment support C++20 conformance. C++20 ranges and concepts support generic programming patterns used in the solver contract and capability profile registry.

- **Build System: CMake.** CMake is the de facto standard for cross-platform C++ builds. The required solver library ecosystem (QUBO formulation, mathematical optimization) has established CMake support. The OpenTelemetry C++ SDK provides CMake integration via FetchContent or find_package. CMake's out-of-source build model is compatible with Docker multi-stage builds. Meson and Bazel are viable alternatives but have weaker ecosystem coverage for the specific library dependencies in scope.

**Governance update:** ADR-001 promoted to Accepted. C++20 and CMake replace the deferred status recorded in ADR-001 Limitations.

---

## Decision 2: C# and .NET Stack

**.NET 10 LTS is the runtime version for the Daedalus API. ASP.NET Core is the web framework. Npgsql is the PostgreSQL driver.**

**Context:** ADR-002 selected C# with ASP.NET Core as the API layer language. The .NET version was not specified. The IRR identified the unspecified .NET version as BG-2 (Blocking Gap 2).

**Decisions:**

- **Runtime: .NET 10 LTS.** .NET 10 is the current Long-Term Support release. It provides a stable support window appropriate for the MVP development lifecycle. .NET 9 is a Standard Term Support release with a shorter support window. .NET 10 is the appropriate MVP-stable choice.

- **Framework: ASP.NET Core (included in .NET 10).** No additional framework selection is required; ASP.NET Core is bundled with the .NET SDK.

- **PostgreSQL driver: Npgsql.** Npgsql is the .NET PostgreSQL driver. It is required for both Dapper query execution and DbUp migration runner operation.

- **ORM / database access: Dapper.** Dapper is a lightweight micro-ORM that operates on top of Npgsql. It provides typed query results without a full ORM migration infrastructure. Entity Framework Core is not required; the schema is defined by SPEC-012 and managed by DbUp (Decision 3).

---

## Decision 3: Schema Migration Tooling

**DbUp manages PostgreSQL schema migrations. Versioned SQL files are stored in `infrastructure/postgres/migrations/`. A dedicated migration runner container applies migrations at deployment time.**

**Context:** The IRR identified the absence of a database migration tooling decision as BG-6 (Blocking Gap 6). SPEC-012 defines the schema; no tooling was specified.

**Decisions:**

- **Migration tool: DbUp.** DbUp is a .NET-native migration runner that applies versioned SQL scripts from a directory. It maintains an applied-migrations journal table in PostgreSQL. Scripts are applied in filename-sorted order; the standard naming convention is `{sequence}_{description}.sql` (e.g., `001_create_jobs.sql`). DbUp does not generate migration SQL; schema authors write SQL directly, which is appropriate for a hand-specified schema defined by SPEC-012.

- **Migration directory: `infrastructure/postgres/migrations/`.** All versioned SQL migration files are stored in this directory. Files are committed to the repository and tracked by git history.

- **Seed data directory: `infrastructure/postgres/seed/`.** Seed scripts (e.g., default scheduler configuration seed per SPEC-003 FR-14) are stored separately from migration scripts. Seed scripts are idempotent (`INSERT ... ON CONFLICT DO NOTHING` or equivalent).

- **Execution model: separate migration runner.** Migrations are not applied at API startup. A dedicated migration runner container (or a one-shot `docker compose run` command) applies pending migrations before the API starts. This prevents startup-time migration races and keeps the API container's responsibility boundary clean.

- **Startup migration prohibition:** The API container must not apply DbUp migrations at startup. Any API startup routine that detects unapplied migrations and applies them is prohibited. Migrations are an explicit operational step.

---

## Decision 4: Default Scheduler Configuration UUID

**The default scheduler configuration uses the deterministic UUIDv5 identifier `2f5ce394-c4e4-5324-b842-f1ff47aafc68`.**

**Context:** SPEC-003 FR-14 specifies that a default scheduler configuration must be seeded into PostgreSQL at API startup. The IRR identified the absence of a stable, deterministic identifier for this configuration as BG-5 (Blocking Gap 5). A deterministic UUID prevents migration idempotency failures when the seed script is applied multiple times.

**Decisions:**

- **UUID generation algorithm: UUIDv5.** UUIDv5 uses SHA-1 to generate a deterministic UUID from a namespace UUID and a name string. Given the same namespace and name, the output is always identical, making the seed script idempotent without a database-side lookup.

- **Namespace: DNS (`6ba7b810-9dad-11d1-80b4-00c04fd430c8`).** The DNS namespace is the appropriate choice for system-assigned logical names.

- **Logical name: `daedalus/scheduler-config/default/v1`.** The name encodes the project, entity type, semantic role, and version. The version suffix (`v1`) allows a future default configuration with different semantics to use `daedalus/scheduler-config/default/v2` with a distinct UUID.

- **Computed UUID: `2f5ce394-c4e4-5324-b842-f1ff47aafc68`.** This value is the deterministic result of `uuid5(DNS_NAMESPACE, "daedalus/scheduler-config/default/v1")` and must be used verbatim in the seed script and any reference to the default scheduler configuration.

---

## Decision 5: Backend Capability Profile Registry

**Backend capability profiles are stored as JSON files in `config/backends/`. The Worker loads and validates profiles at startup into an immutable in-memory registry.**

**Context:** SPEC-003 FR-4 requires a capability profile per registered backend. The IRR identified the absence of a profile registration mechanism as BG-3 (Blocking Gap 3) and noted that SPEC-003 OQ-2 (registration mechanism) was unresolved. This decision resolves SPEC-003 OQ-2.

**Decisions:**

- **Storage format: JSON files.** Each backend has one JSON file in `config/backends/` named `{backend_id}.json`. The JSON structure is validated against `config/backend-profile.schema.json` at Worker startup.

- **Schema governance: `config/backend-profile.schema.json`.** The JSON Schema document in `config/backend-profile.schema.json` is the authoritative contract for profile files. It enforces all fields required by SPEC-011 FR-4.1 and the SPEC-011 FR-3 metadata fields.

- **Loading policy: startup-only.** The Worker loads all files in `config/backends/` once at startup, validates each against the schema, and constructs an immutable in-memory registry. Profiles are not reloaded without a Worker restart.

- **Validation on load:** A profile file that fails schema validation causes Worker startup failure (not a runtime error). No backend may enter the registry in an invalid state.

- **Immutability:** Once loaded, the registry is read-only for the lifetime of the Worker process. Runtime profile mutation is prohibited. A profile change requires a Worker restart.

- **Initial profiles:** Five backend profile files are committed to the repository at Phase 0 Closure: `nearest-neighbor.json`, `greedy-insertion.json`, `qubo-simulated-annealing.json`, `qaoa-qiskit.json`, `qaoa-hardware.json`. All five declare `is_provisional = true` pending empirical validation.

---

## Decision 6: Normalized-Penalty Scoring Formula

**The Scheduler scoring formula is normalized penalty scoring: lower total penalty wins. Penalty dimensions are cost, latency, quality, and risk. All dimensions are normalized to `[0, 1]` before weighting.**

**Context:** SPEC-003 FR-7 defers the specific scoring formula for each objective mode to implementation planning (OQ-4). The IRR identified the unresolved formula as BG-4 (Blocking Gap 4). This decision resolves SPEC-003 OQ-4.

**Decisions:**

- **Scoring direction: lower penalty wins.** All objective modes use a penalty-based score where lower total penalty indicates a more preferred backend. This is consistent with the BudgetCapped and DeadlineAware modes, which eliminate backends exceeding hard limits, and allows `CheapestValid`, `FastestValid`, and `BestQuality` to operate as single-dimension penalty modes.

- **Normalization: [0, 1] range.** Each penalty dimension is normalized to the `[0, 1]` range across the set of eligible backends before combining. Normalization base is the range across eligible candidates for the current invocation. A backend with the worst value in a dimension receives penalty 1.0; the best receives 0.0. Single-eligible-backend invocations produce 0.0 on all dimensions.

- **Penalty dimensions and weights by objective mode:**

| Mode | cost_penalty | latency_penalty | quality_penalty | risk_penalty |
|---|---|---|---|---|
| `CheapestValid` | 1.0 | 0.0 | tiebreaker | 0.0 |
| `FastestValid` | 0.0 | 1.0 | tiebreaker | 0.0 |
| `Balanced` | `cost_weight` | `latency_weight` | `quality_weight` | 0.0 |
| `BestQuality` | 0.0 | 0.0 | 1.0 | 0.0 |
| `DeadlineAware` | 0.0 | 0.0 | 1.0 | 0.0 |
| `BudgetCapped` | 0.0 | 0.0 | 1.0 | 0.0 |
| `Experimental` | 0.0 | 0.0 | 1.0 | risk_weight |

- **Quality penalty direction:** `quality_profile` is an ordinal enum (`Baseline < Competitive < Near-Optimal`). Higher quality receives lower penalty. The quality penalty is `1 - quality_rank / max_rank`, where ranks are 0 (Baseline), 1 (Competitive), 2 (Near-Optimal).

- **Risk dimension:** The `risk_penalty` dimension applies under `Experimental` mode and assigns higher penalty to backends with `is_provisional = false`. This is the inverse of the normal quality preference: under `Experimental` mode, provisional backends are preferred for evaluation, so non-provisional backends receive a mild risk penalty. The risk weight under `Experimental` mode is 0.1 (provisional — not empirically derived; subject to calibration during Phase 3 benchmarking). This dimension is 0.0 in all other modes.

- **Tiebreaker integration:** After penalty scoring, ties are broken by `backend_id` lexicographic order per SPEC-003 FR-8. The tiebreaker is not a penalty component.

- **No quantum-specific scoring:** No penalty bonus or malus is applied based on `backend_category` or `backend_id`. The evidence thesis requires that quantum and classical backends compete on declared capability profile values only.
