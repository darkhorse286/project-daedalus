# ADR-007: Quantum Hardware Execution Deferred Beyond MVP

## Status

Proposed

**Date:** 2026-06-07

**Related Feature(s):** QUBO Simulated Annealing Backend, Daedalus Scheduler

**Related ADR(s):** ADR-008

---

# Context

Project DAEDALUS evaluates hybrid classical and quantum-adjacent optimization for fleet routing. The project thesis is that most routing problems should not use quantum hardware. Demonstrating this thesis requires a production-style scheduler that evaluates solver candidates against a configured objective and rejects expensive or experimental backends when classical methods are sufficient.

Actual quantum hardware (IBM Quantum Runtime, QAOA, and equivalent services) presents the following problems for an MVP:

- Hardware access requires cloud accounts and introduces external service dependencies that conflict with the offline Docker Compose execution model.
- Current quantum hardware produces noisy results under real-world conditions. Noisy results make reproducibility difficult and complicate evidence reporting.
- Hardware queue wait times are variable and can be significant. This makes deterministic benchmarking impractical.
- The thesis does not require hardware execution to be demonstrated. A scheduler that rejects the QUBO simulated annealing backend when classical methods satisfy the objective makes the same argument without hardware.

---

# Decision

Quantum hardware execution is explicitly deferred beyond the MVP.

The MVP includes:

- QUBO problem formulation layer
- Simulated annealing as a quantum-style execution proxy
- A scheduler capable of rejecting the QUBO simulated annealing backend when classical methods satisfy the configured objective

The MVP does not include:

- IBM Quantum Runtime execution
- QAOA hardware execution
- Any cloud quantum backend integration

The scheduler demonstrating that classical methods are sufficient is the thesis artifact. Hardware execution is not required to support the thesis in the MVP.

---

# Alternatives Considered

## Include IBM Quantum Runtime in the MVP

### Description

Integrate IBM Quantum Runtime as a real hardware backend alongside the simulated annealing backend.

### Benefits

Demonstrates actual quantum hardware integration. Stronger claim for technical publications.

### Drawbacks

External cloud dependency. Account and credential management required. Noisy hardware results complicate reproducibility. Queue wait times make deterministic benchmarking impractical. Access and cost are uncertain.

### Reason Not Selected

Hardware execution in the MVP would undermine reproducibility and make the thesis harder to support with clean, reproducible evidence. The MVP thesis does not require hardware.

---

## Include Qiskit Aer Local Simulation

### Description

Use Qiskit Aer for local quantum circuit simulation without hardware access.

### Benefits

No hardware or cloud dependency. Reproducible. Local execution compatible with Docker Compose.

### Drawbacks

Qiskit Aer simulation is a Python-layer concern. It would be implemented through the Python adapter, not the C++ core. Adding it to the MVP scope is not required to demonstrate the thesis.

### Reason Not Selected

Not required for the MVP thesis. Can be added as a Python adapter extension in a future iteration without architectural changes. Adding it now would expand MVP scope without adding thesis support.

---

# Consequences

## Positive

MVP scope is bounded to the demonstrable thesis without external dependencies.

Reproducibility is preserved. Simulated annealing with a fixed seed produces deterministic results.

The scheduler can demonstrate rejection of the QUBO simulated annealing backend when classical methods satisfy the objective. This is the thesis artifact.

## Negative

The MVP cannot make hardware execution claims. Technical writing based on MVP results must not reference actual quantum hardware behavior.

Future hardware integration may require changes to the solver contract defined in ADR-008.

## Accepted Risks

Simulated annealing does not accurately model actual quantum hardware noise or behavior. Any reporting based on MVP results must state this limitation explicitly.

---

# Architectural Impact

| Component      | Impact |
| -------------- | ------ |
| Domain Layer   | Yes    |
| API Layer      | No     |
| Persistence    | No     |
| Infrastructure | No     |
| Configuration  | Yes    |
| Observability  | Yes    |
| Security       | No     |
| Deployment     | No     |

QUBO formulation remains in scope in the domain layer. The QUBO simulated annealing backend is configurable and must be traceable when executed. No quantum cloud service infrastructure is required.

---

# Supporting Evidence

- README.md: Deferred section lists "IBM Quantum Runtime execution" and "QAOA hardware execution"
- README.md: First Article Thesis states that a production-style scheduler should treat quantum execution as an expensive backend, not a magical default
- docs/architecture.md: Open Architecture Decisions section lists "Quantum hardware execution: deferred beyond MVP"

---

# Assumptions

Simulated annealing with a QUBO formulation is an adequate proxy for demonstrating scheduler rejection of quantum-style backends in the MVP context.

IBM Quantum Runtime or equivalent hardware will become accessible at acceptable cost and latency in a future iteration if hardware demonstration becomes necessary.

---

# Limitations

Simulated annealing results do not reflect actual quantum hardware behavior. The MVP makes no claims about quantum hardware performance.

Any technical writing based on MVP results must explicitly state that hardware execution was not performed and that simulated annealing was used as a proxy.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)
- Technical reports or blog posts based on MVP results must reference this limitation

---

# Review Triggers

IBM Quantum Runtime access becomes available at acceptable cost and queue latency for a development environment.

Simulated annealing results diverge materially from hardware results in ways that affect the validity of thesis claims.

Hardware execution becomes a requirement for a target publication venue or employer demonstration.

---

# Employer Signaling

- Optimization
- AI Engineering
- Reliability Engineering (demonstrating when not to use an advanced backend is an engineering judgment)

---

# Decision Summary

**Decision:** Quantum hardware execution deferred. QUBO simulated annealing used as a proxy backend.

**Primary Benefit:** Preserves reproducibility and removes external cloud dependencies from the MVP. The thesis is demonstrable without hardware.

**Primary Cost:** The MVP cannot make hardware execution claims. Simulated annealing is not an accurate model of hardware behavior.

**Evidence Supporting the Decision:** README thesis statement and architecture document.

**Next Review Trigger:** IBM Quantum Runtime access becomes available at acceptable cost and queue latency, or a publication venue requires hardware demonstration.
