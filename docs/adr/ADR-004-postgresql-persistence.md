# ADR-004: PostgreSQL as the Persistence Layer

## Status

Accepted

**Date:** 2026-06-07

**Related Feature(s):** Daedalus API, Daedalus Worker, Evidence Log

**Related ADR(s):** ADR-002

---

# Context

Project DAEDALUS must durably persist routing problems, scheduler configurations, scheduler decisions, solver runs, solver results, telemetry events, and evidence reports. These entities are relationally linked. A scheduler decision references a job, a solver run references a scheduler decision, and a result references a solver run. The evidence log depends on the integrity of these associations.

The API and Worker both write to the persistence layer concurrently. The persistence layer must support concurrent writes without data corruption and must be queryable for evidence log reconstruction and status reporting. The MVP runs in Docker Compose without cloud dependencies.

---

# Decision

PostgreSQL is the persistence technology for all durable storage in Project DAEDALUS.

The specific data access approach (Dapper, Entity Framework Core, or raw SQL via libpqxx for the C++ Worker) and schema migration tooling have not been selected. These decisions are deferred to implementation planning.

---

# Alternatives Considered

## SQLite

### Description

Embedded relational database requiring no separate server process.

### Benefits

Zero operational overhead. No container required. Simple file-based storage.

### Drawbacks

Does not support concurrent write workloads without serialization. The API and Worker write concurrently. Write contention would degrade performance and risk data integrity under concurrent access.

### Reason Not Selected

Concurrent write access from the API and Worker is a hard requirement. SQLite's concurrency model does not satisfy it.

---

## MySQL / MariaDB

### Description

Alternative open-source relational database servers.

### Benefits

Mature and widely deployed. Docker Compose compatible.

### Drawbacks

Weaker JSONB support compared to PostgreSQL. Less ecosystem alignment for the .NET (Npgsql) and C++ (libpqxx) driver choices. Fewer advanced indexing and query operator options.

### Reason Not Selected

PostgreSQL provides a stronger feature set for the expected schema requirements, including JSONB for flexible routing problem field storage and richer indexing capabilities.

---

## MongoDB

### Description

Document database with a flexible schema model.

### Benefits

Flexible document model could accommodate variation in routing problem structure without schema migration.

### Drawbacks

Weaker referential integrity. The evidence log model requires reliable relational associations across jobs, scheduler decisions, solver runs, and results. MongoDB's document model is a liability for these relational requirements, not an advantage.

### Reason Not Selected

The evidence log is the core audit artifact of this system. Referential integrity is a hard requirement. MongoDB's document model does not provide it at the level required.

---

# Consequences

## Positive

Referential integrity enforced at the database level for evidence log associations.

JSONB support provides flexibility for routing problem field variation without forcing full schema migrations.

Strong driver ecosystems are available: Npgsql for .NET (API) and libpqxx for C++ (Worker).

Well-understood operational characteristics.

## Negative

PostgreSQL is an infrastructure service requiring container configuration, credential management, and schema migration tooling.

Schema migrations require explicit management tooling that has not yet been selected.

## Accepted Risks

Schema migration tooling is not selected. This will be resolved during API implementation planning.

No backup or recovery strategy is defined for the MVP. This is acceptable for a local development and demonstration environment.

---

# Architectural Impact

| Component      | Impact |
| -------------- | ------ |
| Domain Layer   | No     |
| API Layer      | Yes    |
| Persistence    | Yes    |
| Infrastructure | Yes    |
| Configuration  | Yes    |
| Observability  | Yes    |
| Security       | Yes    |
| Deployment     | Yes    |

The API reads and writes PostgreSQL. The Worker reads problems and writes solver results. Query latency and connection pool metrics are operational observability requirements. Database credentials must be managed in environment configuration.

---

# Supporting Evidence

- README.md: PostgreSQL listed as a core MVP component
- docs/architecture.md: PostgreSQL persistence entities enumerated
- docs/architecture.md: Open Architecture Decisions section lists "Persistence: PostgreSQL"

No benchmark evidence exists at this stage. This is a pre-implementation decision.

---

# Assumptions

Routing problem data and telemetry volume remain within PostgreSQL single-node capacity for the MVP workload.

Write throughput from the API and Worker does not require connection pool tuning beyond standard configuration defaults.

---

# Limitations

Schema migration tooling is not selected. This is a blocking dependency for API implementation.

No backup or recovery strategy exists for the MVP.

Single-node PostgreSQL provides no high availability.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)

---

# Review Triggers

Write throughput requires connection pool tuning or read replica introduction.

Evidence log query complexity requires schema or indexing changes.

Data volume exceeds practical single-node capacity.

---

# Employer Signaling

- System Design
- Reliability Engineering

---

# Decision Summary

**Decision:** PostgreSQL for all durable persistence.

**Primary Benefit:** Referential integrity and mature driver ecosystem for relational evidence log associations.

**Primary Cost:** Infrastructure service to operate; schema migration tooling selection is a pending dependency.

**Evidence Supporting the Decision:** Architecture document intent. No benchmark evidence at this stage.

**Next Review Trigger:** Write throughput or query complexity exceeds single-node PostgreSQL characteristics for the MVP workload.
