# ADR-002: C# ASP.NET Core as the Control Plane API Language

## Status

Proposed

**Date:** 2026-06-07

**Related Feature(s):** Daedalus API

**Related ADR(s):** ADR-001, ADR-003, ADR-004

---

# Context

Project DAEDALUS requires a control plane API that handles routing job submission, job status polling, scheduler configuration management, and evidence report serving. The API is the integration boundary for the developer CLI and future end-users.

The API is I/O-bound, not compute-bound. It must integrate with PostgreSQL for persistence and RabbitMQ for job publishing. It must produce structured responses, validate routing job requests, and serve as the primary external interface of the system.

---

# Decision

C# with ASP.NET Core is the implementation language and framework for Daedalus API.

ASP.NET Core provides a mature, strongly-typed web framework with built-in support for dependency injection, structured serialization, middleware pipelines, and OpenAPI generation. These capabilities are well-matched to the I/O-bound control plane requirements.

The specific .NET version (8 LTS or 9/10) and data access approach (Dapper or Entity Framework Core) are not selected in this ADR. These decisions are deferred to implementation planning.

---

# Alternatives Considered

## Go (with chi, gin, or echo)

### Description

Lightweight HTTP frameworks for a statically compiled, garbage-collected language.

### Benefits

Small binary size. Fast startup. Simple concurrency model. No managed runtime overhead beyond GC.

### Drawbacks

Less mature ecosystem for structured request validation, OpenAPI generation, and database integration compared to ASP.NET Core. Would require more implementation work for capabilities the framework provides natively.

### Reason Not Selected

The productivity cost of assembling validation, serialization, and dependency injection from components outweighs the performance benefits for a control plane API that does not have high throughput requirements.

---

## Python (FastAPI)

### Description

Modern async Python web framework with automatic OpenAPI generation.

### Benefits

Rapid development. Automatic OpenAPI generation. Strong ecosystem for data-adjacent systems.

### Drawbacks

Contradicts the architectural constraint that Python is confined to an adapter role (ADR-005). Weaker static typing discipline than C# for production API contracts.

### Reason Not Selected

A Python control plane would blur the responsibility boundary between the control plane and the Python solver adapter. Contradicts ADR-005.

---

## Rust (Axum or Actix-web)

### Description

High-performance Rust web frameworks.

### Benefits

Excellent raw throughput. Memory safety without GC.

### Drawbacks

High development complexity for a control plane with modest throughput requirements. Premature performance optimization.

### Reason Not Selected

Complexity is disproportionate to the control plane requirements. The API is I/O-bound and not a throughput bottleneck in this architecture.

---

# Consequences

## Positive

Mature framework reduces implementation cost for request validation, serialization, dependency injection, and OpenAPI generation.

Strongly typed API contracts reduce the risk of contract drift.

Clear tier separation: C# for the I/O-bound control plane, C++ for the compute-bound core.

## Negative

.NET runtime adds container image size and startup overhead compared to natively compiled alternatives.

Two languages in the repository (C# and C++) increase toolchain diversity and onboarding cost.

## Accepted Risks

.NET version upgrades require periodic API layer maintenance.

Data access library selection (Dapper vs. EF Core) has downstream consequences for schema migration strategy that are not resolved in this ADR.

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

The API layer does not own domain logic. Domain logic lives in Daedalus Core. The .NET PostgreSQL driver (Npgsql) and OpenTelemetry .NET SDK are required.

---

# Supporting Evidence

- README.md: architectural intent for the control plane
- docs/architecture.md: Open Architecture Decisions section lists "Control plane language: C# ASP.NET Core"

No benchmark evidence exists at this stage. This is a pre-implementation decision.

---

# Assumptions

The API layer will not become a compute bottleneck requiring native performance optimization.

.NET 8 LTS or later is available in the target container runtime environment.

---

# Limitations

The .NET version has not been selected. .NET 8 LTS and .NET 9 are candidates.

The data access approach (Dapper vs. Entity Framework Core) has not been selected. This decision affects schema migration strategy.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)

---

# Review Triggers

API throughput becomes a measurable bottleneck under the expected workload.

A required integration is unavailable or poorly supported in the .NET ecosystem.

---

# Employer Signaling

- System Design
- Distributed Systems
- Reliability Engineering

---

# Decision Summary

**Decision:** C# ASP.NET Core for Daedalus API.

**Primary Benefit:** Mature production-grade framework reduces implementation cost for the I/O-bound control plane.

**Primary Cost:** .NET runtime overhead; increased language diversity in the repository.

**Evidence Supporting the Decision:** Architecture document intent. No benchmark evidence at this stage.

**Next Review Trigger:** API throughput becomes a measurable bottleneck, or a required integration is unavailable in the .NET ecosystem.
