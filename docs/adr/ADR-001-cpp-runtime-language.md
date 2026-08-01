# ADR-001: C++ as the Runtime Core and Worker Language

## Status

Accepted

**Date:** 2026-06-07

**Accepted:** 2026-08-01

**Related Feature(s):** Daedalus Core, Daedalus Worker

**Related ADR(s):** ADR-002

---

# Context

Project DAEDALUS requires a compute-intensive execution runtime that runs routing algorithms and QUBO simulated annealing directly. Daedalus Core and Daedalus Worker must execute solver workloads with predictable, low-overhead execution characteristics. The language selection determines the build toolchain, access to solver and mathematical library ecosystems, memory management model, and interoperability with the Python adapter and C# control plane.

The Worker enforces solver timeouts, executes solver algorithms, evaluates result quality, and emits telemetry. These responsibilities require deterministic execution behavior and direct access to compute resources.

---

# Decision

C++ is the implementation language for Daedalus Core, Daedalus Worker, and the CLI.

C++ provides direct memory management, deterministic execution profiles, and access to high-performance mathematical and optimization libraries without managed runtime overhead.

**C++ Standard: C++20.** C++20 provides ranges, concepts, designated initializers, `std::span`, `std::format`, `std::jthread`, and the structured bindings, `std::optional`, `std::variant`, `std::string_view`, `if constexpr`, and filesystem library features from C++17. All compiler versions targeted by the Docker Compose build environment support C++20 conformance. C++20 ranges and concepts support generic programming patterns used in the solver contract and capability profile registry.

**Build System: CMake.** CMake is the build system for all C++ components. The required solver library ecosystem and the OpenTelemetry C++ SDK provide CMake integration via FetchContent or find_package. CMake out-of-source builds are compatible with Docker multi-stage build images.

---

# Alternatives Considered

## Rust

### Description

Systems programming language with memory safety guarantees enforced at compile time without a garbage collector.

### Benefits

Memory safety without GC. Growing ecosystem for systems and mathematical programming. Excellent toolchain.

### Drawbacks

Smaller ecosystem for classical routing optimization and QUBO solver libraries compared to C++. Less tooling maturity for this specific domain at this time.

### Reason Not Selected

Ecosystem availability for the target solver domains is weaker than C++ at the time of this decision. The project prioritizes ecosystem access and established tooling for the MVP.

---

## Go

### Description

Compiled language with a simple concurrency model and fast build times.

### Benefits

Simple goroutine-based concurrency. Fast compilation. Garbage-collected but with generally low pause times.

### Drawbacks

Garbage collector introduces latency variance that is incompatible with deterministic solver benchmarking. Limited ecosystem for mathematical optimization.

### Reason Not Selected

GC-induced latency variance conflicts with the reproducibility requirement. Solver benchmark results must be deterministic.

---

## C#

### Description

Managed language already selected for the control plane. Consolidating on one language would reduce toolchain diversity.

### Benefits

Reduces language count in the repository. Strong ecosystem. .NET runtime performance has improved significantly.

### Drawbacks

Managed runtime with GC overhead. Less suitable for bare-metal solver execution. Limited access to C and C++ optimization libraries without interop overhead.

### Reason Not Selected

Managed runtime characteristics are not suitable for compute-bound solver workloads. Maintaining clear tier separation between the I/O-bound API (C#) and the compute-bound core (C++) is architecturally cleaner.

---

# Consequences

## Positive

Direct memory management enables deterministic execution profiles required for reproducible benchmarks.

Access to C++ optimization and mathematical library ecosystems for classical and QUBO solver implementations.

Clear responsibility separation between the compute-bound runtime (C++) and the I/O-bound control plane (C#).

## Negative

Two production languages increase repository complexity, toolchain complexity, and onboarding cost.

C++ build systems and dependency management are more complex than managed language equivalents.

Manual memory management introduces a class of defects that managed runtimes eliminate.

## Accepted Risks

Memory safety vulnerabilities are a known risk of C++ that must be mitigated through testing, tooling (sanitizers), and code review discipline.

---

# Architectural Impact

| Component      | Impact |
| -------------- | ------ |
| Domain Layer   | Yes    |
| API Layer      | No     |
| Persistence    | No     |
| Infrastructure | Yes    |
| Configuration  | Yes    |
| Observability  | Yes    |
| Security       | Yes    |
| Deployment     | Yes    |

C++ build toolchain must be available in the Worker container image. The OpenTelemetry C++ SDK is required for Worker instrumentation. Memory safety discipline is required to mitigate C++ defect classes.

---

# Supporting Evidence

- README.md: architectural intent for runtime components
- docs/architecture.md: Open Architecture Decisions section lists "Runtime core language: C++"

No benchmark evidence exists at this stage. This is a pre-implementation decision.

---

# Assumptions

The required optimization and solver libraries (classical routing, QUBO formulation) are available as C++ libraries or have C-compatible APIs.

The target deployment environment supports standard C++ compilation toolchains.

The development environment has access to a suitable C++ build system.

---

# Limitations

Cross-platform compilation requirements beyond Docker Compose Linux containers are not defined for the MVP. The C++20 standard and CMake build system are the committed baseline for all MVP development.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)

---

# Review Triggers

A required solver library is unavailable in C++.

Build system or toolchain complexity becomes a significant impediment to development velocity.

A superior language alternative demonstrates equivalent performance with materially lower operational risk.

---

# Employer Signaling

- Systems Programming
- Performance Engineering
- Optimization

---

# Decision Summary

**Decision:** C++ for Daedalus Core and Daedalus Worker.

**Primary Benefit:** Deterministic execution profiles and direct access to optimization library ecosystems without managed runtime overhead.

**Primary Cost:** Increased toolchain complexity and memory safety risk compared to managed alternatives. Two production languages in one repository.

**Evidence Supporting the Decision:** Architecture document intent. No benchmark evidence at this stage.

**Next Review Trigger:** Required solver library unavailable in C++, or build system complexity becomes an impediment.
