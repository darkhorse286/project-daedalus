# ADR-007: Quantum Hardware Execution Deferred Beyond MVP

## Status

Accepted

**Date:** 2026-06-07

**Amended:** 2026-06-20

**Related Feature(s):** QUBO Simulated Annealing Backend (SPEC-015), QAOA Solver Backend (SPEC-018), Daedalus Scheduler

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
- Simulated annealing as a quantum-style execution proxy (SPEC-015)
- QAOA local circuit simulation via Qiskit Aer, implemented through the Python Solver Adapter (SPEC-017, SPEC-018)
- A scheduler capable of rejecting the QUBO simulated annealing backend when classical methods satisfy the configured objective

The MVP does not include:

- IBM Quantum Runtime execution
- QAOA hardware execution
- Any cloud quantum backend integration

The scheduler demonstrating that classical methods are sufficient is the thesis artifact. Hardware execution is not required to support the thesis in the MVP.

**Clarification — Qiskit Aer local simulation:** Qiskit Aer local simulation is accepted as the MVP quantum-circuit execution path. SPEC-018 (QAOA Solver Backend) is the concrete realization of this path, implemented through the Python Solver Adapter (SPEC-017). Local simulation executes QAOA circuits on the local Aer simulator within the Docker Compose environment, without hardware access, without IBM Quantum Runtime, and without cloud dependencies. Local simulation does not violate the hardware deferral decision: it is reproducible, offline, and makes no quantum hardware performance claims.

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

### Selected for SPEC-018

This alternative was deferred at initial decision and has since been adopted. SPEC-018 (QAOA Solver Backend) implements QAOA on the local Aer simulator through the Python Solver Adapter (SPEC-017). The selection satisfies all conditions identified at deferral:

- **No cloud dependency:** The Aer simulator executes within the `python-adapter` Docker Compose container; no external service is contacted.
- **Reproducible:** Shot outcomes are seeded from `execution_seed` via PCG64/SeedSequence per ADR-010 and SPEC-017 FR-9; results are reproducible given identical inputs.
- **Docker Compose compatible:** Qiskit and Qiskit Aer are installed in the python-adapter container image per SPEC-017 FR-8; no infrastructure changes are required.
- **Python adapter boundary:** SPEC-018 is implemented as a Python solver function hosted by SPEC-017; no C++ core changes are required.
- **No hardware claims:** SPEC-018 is explicitly scoped to local simulation. IBM Quantum Runtime and cloud quantum execution remain out of scope.

Adding local simulation at this point extends the evidence portfolio without adding external dependencies and is consistent with the original assessment that this alternative could be added "without architectural changes."

---

# Consequences

## Positive

MVP scope is bounded to the demonstrable thesis without external dependencies.

Reproducibility is preserved. Simulated annealing with a fixed seed produces deterministic results. QAOA local simulation (SPEC-018) is reproducible by the same mechanism — PCG64 seeded from `execution_seed` via SeedSequence per ADR-010.

The scheduler can demonstrate rejection of the QUBO simulated annealing backend when classical methods satisfy the objective. This is the thesis artifact.

SPEC-018 enables QAOA circuit execution on the local Aer simulator, extending the scheduler's backend portfolio with a genuine quantum-adjacent algorithm backed by real evidence rather than a proxy.

## Negative

The MVP cannot make quantum hardware execution claims. Technical writing based on MVP results must not reference actual quantum hardware behavior. SPEC-018 uses local simulation only; its results do not reflect hardware noise, gate fidelity, or decoherence.

Future hardware integration may require changes to the solver contract defined in ADR-008.

Hardware execution remains deferred. SPEC-019 or later may introduce IBM Quantum Runtime or equivalent hardware backends if access conditions improve and the thesis requires hardware demonstration.

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

Simulated annealing results (SPEC-015) do not reflect actual quantum hardware behavior. QAOA local simulation results (SPEC-018) reflect the behavior of classical state-vector simulation, not physical quantum gate execution, hardware noise, or decoherence. The MVP makes no claims about quantum hardware performance.

Any technical writing based on MVP results must explicitly state that hardware execution was not performed and that local simulation was used as a proxy.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)
- SPEC-017 (Python Solver Adapter) — defines the Python adapter boundary through which SPEC-018 executes
- SPEC-018 (QAOA Solver Backend) — the concrete realization of Qiskit Aer local simulation accepted by this ADR
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

**Decision:** Quantum hardware execution deferred. QUBO simulated annealing (SPEC-015) and QAOA local simulation via Qiskit Aer (SPEC-018) are the MVP quantum-adjacent execution paths. IBM Quantum Runtime and cloud quantum execution remain out of scope.

**Primary Benefit:** Preserves reproducibility and removes external cloud dependencies from the MVP. The thesis is demonstrable without hardware. Local simulation provides genuine QAOA evidence within the offline Docker Compose model.

**Primary Cost:** The MVP cannot make quantum hardware execution claims. Local simulation does not reflect hardware noise or gate fidelity.

**Evidence Supporting the Decision:** README thesis statement, architecture document, SPEC-018 (QAOA Solver Backend).

**Next Review Trigger:** IBM Quantum Runtime access becomes available at acceptable cost and queue latency, or a publication venue requires hardware demonstration.
