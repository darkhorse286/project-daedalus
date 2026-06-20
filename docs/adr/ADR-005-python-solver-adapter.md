# ADR-005: Python Solver Adapter Role and Contract Boundary

## Status

Accepted

**Date:** 2026-06-07

**Amended:** 2026-06-20 — OQ-1 (transport contract) resolved. JSON over HTTP accepted as MVP transport. Transport Contract section updated from Unresolved to Resolved. SPEC-017 FR-2 is the authoritative transport and endpoint definition.

**Related Feature(s):** Python Solver Adapter

**Related ADR(s):** ADR-001, ADR-008

---

# Context

Python-native optimization ecosystems are relevant to Project DAEDALUS. Qiskit provides quantum circuit tooling for QUBO experiments. OR-Tools provides classical optimization research libraries. AI-based policy experiments require Python-native ML tooling. These ecosystems cannot be replicated in C++ without significant implementation cost.

The runtime core and worker are implemented in C++. Embedding CPython in a C++ process to call Python directly introduces significant coupling, process lifecycle complexity, and Python GIL constraints that conflict with C++ concurrency requirements.

Python must not become the center of the runtime. The architecture principles require that the API submits jobs, the Worker executes them, and the Core owns domain logic. Python's role must be confined so that it cannot become a dependency of the critical execution path.

---

# Decision

This ADR records two separate concerns at different resolution states.

## Role Boundary (Decided)

Python is confined to an adapter role in Project DAEDALUS.

The Python adapter runs as a separate container process. The Daedalus Worker communicates with the Python adapter through a defined transport contract when a Python-based solver backend is selected.

Python is not the runtime. Python is not the scheduler. Python is an adapter.

## Transport Contract (Resolved — SPEC-017)

**OQ-1 status: Resolved (2026-06-20)**

The transport mechanism between the C++ Worker and the Python adapter is **JSON over HTTP**. This decision is recorded in SPEC-017 FR-2, which is the authoritative endpoint and transport definition.

**Decided transport:**
- **JSON over HTTP** — Selected for MVP. Simple, debuggable, testable. Requests and responses are observable as plain text without special tooling. Standard HTTP testing applies without generated client stubs. Python's HTTP server ecosystem is mature (FastAPI, Flask, stdlib). At MVP routing problem payload sizes (maximum approximately 100 stops with coordinate arrays), JSON serialization overhead is acceptable and must be measured. Full rationale documented in SPEC-017 FR-2.

**Rejected transports:**
- **gRPC** — Not selected for MVP. The strongly typed contract benefit does not outweigh the additional code generation tooling, protobuf dependency, and debugging complexity at MVP scale.
- **stdin/stdout IPC** — Not selected. Would require the Worker to manage the Python process lifetime, contradicting the separate container model established by this ADR.

**Endpoint contract (defined by SPEC-017 FR-2):**
- POST `/v1/solve` — Submit a SolverRequest; receive a SolverResponse
- GET `/health` — Adapter liveness and readiness check
- Port 8080 within the Docker Compose internal network

Python remains confined to the adapter container role per the Role Boundary decision above. The transport decision does not alter Python's role.

---

# Alternatives Considered

## Embed CPython in the C++ Worker

### Description

Use the CPython C API to call Python code from within the C++ worker process.

### Benefits

Eliminates inter-process communication overhead. Direct function call semantics.

### Drawbacks

Tight coupling between the C++ worker and Python lifecycle. CPython embedding is complex. The Python GIL conflicts with C++ concurrency. Python version upgrades become difficult. Debugging embedded Python failures is significantly harder than debugging a separate process.

### Reason Not Selected

Coupling and operational complexity are unacceptable. The adapter process boundary is intentional architecture, not a workaround.

---

## Python as the Primary Runtime

### Description

Move optimization logic to Python. Use C++ only for performance-critical paths.

### Benefits

Python ecosystem access without C++ complexity for the core logic.

### Drawbacks

Contradicts the Backend Neutrality and Production-Style Separation of Concerns principles. Python's execution characteristics are unsuitable for the compute-bound solver execution and deterministic benchmarking requirements.

### Reason Not Selected

Rejected by the core architectural principles of Project DAEDALUS. Python cannot own the runtime execution path.

---

## Rewrite Python Libraries in C++

### Description

Port Qiskit, OR-Tools, or equivalent Python solver libraries to C++.

### Benefits

Eliminates the Python dependency entirely.

### Drawbacks

Massive implementation cost. Duplicates established open-source work. Disproportionate to the MVP objective.

### Reason Not Selected

Disproportionate cost. The Python adapter is the correct abstraction.

---

# Consequences

## Positive

The adapter process boundary prevents Python lifecycle management from coupling to the C++ worker.

Python remains replaceable. A different adapter implementation can satisfy the transport contract.

Access to Python-native solver ecosystems without polluting the core runtime execution path.

## Negative

Inter-process communication adds latency on the solver execution path. This must be measured and reported in solver benchmarks.

Two process lifetimes to manage in Docker Compose.

## Accepted Risks

The transport contract decision may require interface changes when resolved. Any interface sketched before the decision is provisional.

IPC overhead may affect solver timing benchmarks. This must be measured, disclosed, and accounted for in evidence reports.

---

# Architectural Impact

| Component      | Impact |
| -------------- | ------ |
| Domain Layer   | No     |
| API Layer      | No     |
| Persistence    | No     |
| Infrastructure | Yes    |
| Configuration  | Yes    |
| Observability  | Yes    |
| Security       | Yes    |
| Deployment     | Yes    |

Adapter execution results persist through the Worker to PostgreSQL. Adapter execution latency must be traced as a separate span from Worker execution. Internal Docker Compose networking is assumed sufficient for MVP security. The adapter-to-worker channel must be secured in any future production deployment.

---

# Supporting Evidence

- README.md: Python Solver Adapter listed as a system component
- docs/architecture.md: Python Solver Adapter responsibilities defined
- docs/architecture.md: Open Architecture Decisions section lists "Python role: adapter only"

No benchmark evidence exists at this stage. This is a pre-implementation decision.

---

# Assumptions

Python solver libraries (Qiskit, OR-Tools) will remain accessible as standard pip-installable packages.

The IPC transport overhead will be acceptable relative to solver execution time. This assumption must be validated during implementation.

---

# Limitations

The security of the adapter-to-worker communication channel is not defined for the MVP.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)
- **SPEC-017 (Accepted):** Provides the full transport and behavioral contract for the Python Solver Adapter. SPEC-017 FR-2 is the authoritative endpoint definition, resolving OQ-1.
- **docs/architecture.md:** Python Solver Adapter responsibilities description should be updated from "JSON or gRPC adapter contract" to "JSON over HTTP adapter contract."

---

# Review Triggers

IPC transport overhead materially affects solver benchmark results and requires a revision to the transport selection.

mTLS or authentication requirements arise for the adapter-to-worker channel in a non-MVP deployment context.

---

# Employer Signaling

- System Design
- Distributed Systems
- AI Engineering

---

# Decision Summary

**Decision:** Python confined to a separate adapter container. JSON over HTTP is the accepted MVP transport (resolved from OQ-1 by SPEC-017 FR-2).

**Primary Benefit:** Clean separation of Python ecosystem access from C++ runtime execution. Python remains replaceable. JSON over HTTP provides a debuggable, testable interface without code generation overhead.

**Primary Cost:** IPC overhead on the solver execution path. JSON serialization latency must be measured and reported in solver benchmarks.

**Evidence Supporting the Decision:** Architecture document intent. SPEC-017 FR-2 transport rationale. No IPC benchmark evidence at this stage; measurement required during implementation.

**Next Review Trigger:** IPC transport overhead materially affects solver benchmark results and requires a revision to the transport selection.
