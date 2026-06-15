# ADR-008: Solver Contract and Backend Neutrality

## Status

Accepted

**Date:** 2026-06-07

**Related Feature(s):** Daedalus Scheduler, Daedalus Core, all solver backends

**Related ADR(s):** ADR-001, ADR-005, ADR-007

---

# Context

Project DAEDALUS supports multiple solver backends: classical baseline solvers (nearest-neighbor, greedy insertion), a QUBO simulated annealing backend, and optionally a Python adapter for quantum or AI-based experiments. The scheduler must select from these backends using a configurable objective and produce evidence explaining why a backend was chosen and why alternatives were rejected.

If solver backends are not accessed through a normalized contract, the scheduler must contain backend-specific code paths. Backend-specific scheduler logic creates two structural problems:

First, adding a new solver backend requires changes to the scheduler. The scheduler becomes responsible for backend implementation details rather than selection policy.

Second, result comparison becomes inconsistent. If each backend returns a different result structure, quality evaluation and regret calculation cannot operate on a shared basis. The evidence log cannot produce consistent comparisons across backends.

The Backend Neutrality architectural principle requires that no solver backend is privileged by the system. Privileging occurs when special-case logic exists for a specific backend anywhere outside the backend's own implementation.

---

# Decision

All solver backends in Project DAEDALUS are accessed through a normalized solver contract.

The contract principles are:

1. A solver accepts a routing problem and an execution configuration as inputs.
2. A solver returns a route plan, execution duration, and execution statistics. Quality metrics are computed by Core from the normalized route plan output and are not returned directly by solver backends.
3. A solver produces evidence of its execution suitable for persistence to the evidence log.
4. No solver backend bypasses the contract. The scheduler operates exclusively on normalized candidates.
5. The scheduler evaluates eligibility, scores candidates, and selects a backend using only information available through the contract. It does not inspect backend internals.

The concrete interface definition (C++ abstract type, struct layout, or equivalent) is not defined in this ADR. The concrete contract depends on the routing problem model, which has not yet been specified. The concrete interface will be defined during SPEC-001 and the subsequent solver implementation planning.

This ADR records the principle and constraint. The concrete definition follows from the specification process.

---

# Alternatives Considered

## Per-Solver Scheduler Logic

### Description

The scheduler contains backend-specific code paths for each solver type. Classical solvers, the QUBO backend, and the Python adapter are handled by separate branches in the scheduler.

### Benefits

Simpler initial implementation when only one solver exists.

### Drawbacks

Every new solver backend requires scheduler changes. Backend implementation details pollute scheduler policy logic. Evidence comparisons become inconsistent if backends return different result structures. Violates the Backend Neutrality architectural principle.

### Reason Not Selected

Architectural principle violation. Unacceptable coupling between scheduler policy and backend implementation. Does not scale beyond a single solver without increasing scheduler complexity.

---

## Heterogeneous Result Types Per Solver

### Description

Each solver returns a backend-specific result structure. The evidence log accepts multiple formats.

### Benefits

Allows each solver to expose backend-specific outputs without mapping to a shared format.

### Drawbacks

Destroys comparability. Quality evaluation and regret calculation require a shared quality metric. If result structures differ, comparison is impossible without backend-specific handling in the evaluator, which is the same problem at a different location.

### Reason Not Selected

Comparability is a hard requirement for quality evaluation and regret analysis. The core thesis depends on comparing solver outcomes on a consistent basis.

---

# Consequences

## Positive

Backend neutrality: adding a new solver backend requires only implementing the contract. The scheduler requires no changes.

Comparable results: all solver outputs share a quality metric that enables regret calculation and evidence comparison.

Explainable decisions: the scheduler evaluates all candidates on the same basis and produces consistent evidence regardless of which backend is selected.

## Negative

The normalized contract may not expose backend-specific data that is useful for extended reporting. A solver evidence extension mechanism may be required for backend-specific metadata such as QUBO energy landscape information.

The concrete interface definition is a blocking dependency for all solver implementations. No solver can be implemented until the contract is defined.

## Accepted Risks

The concrete interface may require revision after SPEC-001 defines the routing problem model. The routing problem structure determines the contract input structure.

The Python adapter transport contract (ADR-005) is unresolved. The way the solver contract is surfaced through the Python adapter depends on the transport decision. ADR-005 and ADR-008 have a dependency that must be resolved together during Python adapter implementation planning.

---

# Architectural Impact

| Component      | Impact |
| -------------- | ------ |
| Domain Layer   | Yes    |
| API Layer      | No     |
| Persistence    | Yes    |
| Infrastructure | No     |
| Configuration  | Yes    |
| Observability  | Yes    |
| Security       | No     |
| Deployment     | No     |

The solver contract is a core domain boundary. The persistence schema for solver runs must align with the contract result structure. Solver configuration and eligibility parameters are contract inputs. All solver executions must produce evidence through the contract for tracing and evidence log persistence.

---

# Supporting Evidence

- docs/architecture.md: Architectural Principles section defines Backend Neutrality
- docs/architecture.md: Daedalus Scheduler responsibilities include "evaluate solver eligibility" and "persist explainable decisions"
- docs/architecture.md: Daedalus Core responsibilities include "solver eligibility checks" and "result quality evaluation"

No benchmark evidence exists at this stage. This is a pre-implementation decision.

---

# Assumptions

All MVP solver backends (nearest-neighbor, greedy insertion, QUBO simulated annealing) can produce a route plan and quality metrics in a normalized format without requiring backend-specific evaluation logic.

A shared quality metric meaningful for comparison across classical and QUBO backends can be defined. The definition of this metric is an open question for SPEC-001 and solver implementation planning.

---

# Limitations

The concrete solver contract interface is not defined here. It is a blocking dependency for all solver implementations and will be defined during SPEC-001 and solver specification.

Backend-specific metadata (for example, QUBO annealing energy data) may require an evidence extension mechanism that is not yet defined.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)
- SPEC-001 will define the routing problem model that determines the contract input structure

---

# Review Triggers

A required solver backend cannot conform to the normalized contract without significant loss of useful information.

Evidence quality requirements demand backend-specific contract extensions that cannot be accommodated without breaking backend neutrality.

SPEC-001 defines a routing problem model structure that is incompatible with the contract inputs assumed here.

---

# Employer Signaling

- System Design
- Optimization
- AI Engineering

---

# Decision Summary

**Decision:** All solver backends accessed through a normalized solver contract enforcing backend neutrality.

**Primary Benefit:** Scheduler remains policy-only. All backends are interchangeable. Evidence is comparable across solver selections.

**Primary Cost:** Concrete contract definition is a blocking dependency for all solver implementations. Backend-specific metadata may require an extension mechanism.

**Evidence Supporting the Decision:** Backend Neutrality architectural principle in the architecture document.

**Next Review Trigger:** Concrete solver contract defined during SPEC-001 planning. A required backend cannot conform to the normalized contract. ADR-005 transport contract resolved, enabling Python adapter contract definition.
