# ADR-005: Python Solver Adapter Role and Contract Boundary

## Status

Proposed

**Date:** 2026-06-07

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

## Transport Contract (Unresolved)

The transport mechanism between the C++ Worker and the Python adapter has not been decided.

The following candidates remain under consideration:

- JSON over HTTP: Simple, debuggable, testable. Adds per-request serialization overhead. Suitable if routing problem payloads remain small.
- gRPC: Lower serialization overhead. Strongly typed contract. Better suited for large payloads. Higher implementation complexity.
- stdin/stdout IPC: Minimal overhead. Simple process lifecycle management. Less structured. Harder to observe and test.

The transport decision is deferred until implementation planning establishes evidence on routing problem payload sizes and acceptable solver execution latency budgets. This ADR will be updated when the transport decision is made.

Implementation of the Python adapter cannot begin until the transport contract is resolved.

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

The transport contract is unresolved. The Python adapter cannot be implemented until it is decided.

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

The transport contract is unresolved. Python adapter implementation is blocked until ADR-005 is updated with the transport decision.

The security of the adapter-to-worker communication channel is not defined for the MVP.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)
- This ADR must be updated when the transport contract decision is made

---

# Review Triggers

IPC transport overhead materially affects solver benchmark results.

A transport candidate proves unsuitable during implementation planning.

The transport contract decision is made. This ADR must be updated at that point.

---

# Employer Signaling

- System Design
- Distributed Systems
- AI Engineering

---

# Decision Summary

**Decision:** Python confined to a separate adapter container. Transport contract unresolved and deferred.

**Primary Benefit:** Clean separation of Python ecosystem access from C++ runtime execution. Python remains replaceable.

**Primary Cost:** IPC overhead on the solver execution path. Transport contract is a blocking dependency for adapter implementation.

**Evidence Supporting the Decision:** Architecture document intent. No benchmark evidence at this stage.

**Next Review Trigger:** Transport contract resolved during implementation planning. This ADR must be updated at that point.
