# ADR-010: Deterministic Randomness and Reproducibility Policy

## Status

Accepted

**Date:** 2026-06-08
**Amended:** 2026-06-12 — Worker owns execution seed derivation; solver specifications define seed usage policy only, not derivation policy (ODR-4, SPEC-005 FR-7)

**Related Feature(s):** SPEC-001 (Routing Problem Model), SPEC-002 (Synthetic Workload Generator), SPEC-005 (Worker Execution Lifecycle), QUBO Simulated Annealing Backend

**Related ADR(s):** ADR-001, ADR-006, ADR-007, ADR-008, ADR-009

---

# Context

The Project DAEDALUS architecture document states the following reproducibility principle:

> Every generated scenario, solver run, scheduler decision, and report must be reproducible from persisted input, configuration, and random seed.

This principle is a first-class architectural commitment. It governs at minimum two classes of MVP components:

1. **The Synthetic Workload Generator** (SPEC-002) generates routing problems from a seed using a PRNG and distribution sampling. SPEC-002 OQ-1 identified the PRNG algorithm, distribution sampling standard, cross-platform scope, and floating-point determinism as unresolved questions that exceed the generator's specification scope.

2. **The QUBO Simulated Annealing Backend** (ADR-007) uses a stochastic optimization process whose outputs must be reproducible from the same problem seed and execution configuration.

Without an architectural policy, each component that requires stochastic computation resolves these questions independently. Independent resolution creates three failure modes:

- **Algorithm inconsistency:** Component A uses `std::mt19937` with `std::uniform_int_distribution`; component B uses xoshiro256\*\*. The same seed produces different behavior per component with no stated policy.
- **Scope inconsistency:** Some components commit to cross-platform reproducibility; others commit to within-platform only. The system-level reproducibility claim is undefined.
- **Breaking change blindness:** An algorithm change in one component is not recognized as a system-level breaking change because no policy exists defining what "system-level" means.

This ADR resolves the architectural policy governing deterministic randomness and reproducibility across Project DAEDALUS. It binds the architecture.md reproducibility principle at the implementation level. It does not specify component-level draw ordering, solver seed *usage* strategies (how a solver initializes its PRNG from the `execution_seed` it receives), or manifest serialization details — those remain in their respective specifications. Execution seed *derivation* is defined here in Decision 4 and is a Worker responsibility, not a solver specification concern.

---

# Decision

Six policy decisions govern deterministic randomness in Project DAEDALUS.

## Decision 1: PRNG Algorithm — PCG64

The Permuted Congruential Generator (PCG64) is the standard PRNG for all reproducibility-critical stochastic computation in Project DAEDALUS.

**Seeding:** The PRNG is seeded using the problem seed (SPEC-001 FR-6, 64-bit unsigned integer) as the initial state. The PCG64 stream constant (increment) is a fixed value defined per component and documented in the component's specification. The stream constant must not change without treating the change as a breaking change under Decision 5.

**Rationale:** PCG64's algorithm is specified using only 64-bit and 128-bit integer arithmetic. Its output sequence is platform-independent on any conforming C++ implementation supporting standard integer types. It does not depend on the C++ standard library's `<random>` distribution implementations, which are permitted by the C++ standard to vary across implementations. PCG64 passes the TestU01 BigCrush battery, providing sufficient statistical quality for scientific simulation at MVP scale. Its seeding procedure is explicit: state is the problem seed, stream is a documented component-level constant.

## Decision 2: Cross-Platform Reproducibility Scope — Semantic Equivalence Across Conforming C++17 Toolchains

The reproducibility guarantee for Project DAEDALUS is **semantic equivalence**: given the same problem seed and configuration, the same component produces output whose field values are identical when parsed as IEEE 754 double precision, regardless of which conforming C++17 toolchain compiled the component.

Semantic equivalence means all parsed field values — coordinates, demands, time windows, fleet parameters, seeds — are identical. It does not require bit-identical byte sequences in serialized output, because JSON serializer formatting is not required to be platform-invariant.

This guarantee extends to the PCG64 integer sequence (bitwise identical across platforms given identical inputs) and to floating-point values derived from it through the approved algorithms in Decision 3. Values derived from transcendental functions (sin, cos, log, sqrt) satisfy semantic equivalence within IEEE 754 double precision; they are not required to be bitwise identical across platforms.

## Decision 3: Distribution Sampling Standard

**Uniform floating-point on (0, 1]:** A PCG64 64-bit output `v` is converted to a uniform float using:

```
u = (v >> 11) × (1.0 / (1ULL << 53))
```

This uses only a bit shift, an integer multiply, and one floating-point multiplication, and is portable under IEEE 754. When `u` must be strictly positive (for example, as input to `ln` in Box-Muller), implementations must handle the zero case; the specific method (e.g., setting the low bit before conversion, or resampling) is a component implementation decision and must be documented.

**Bounded uniform integers:** Bounded uniform integer sampling shall use a bias-free or negligibly biased algorithm using only integer arithmetic. `std::uniform_int_distribution` is prohibited in reproducibility-critical paths: its algorithm is implementation-defined and is not portable across compilers. Acceptable algorithms include Lemire's nearly-divisionless method and bounded rejection sampling using integer arithmetic. The specific algorithm must be documented in each component's specification.

**Normal distribution:** Normal distribution sampling shall use the Box-Muller transform in its standard two-output form. Given two uniform draws u1, u2 ∈ (0, 1] produced from the PCG64 output:

```
z0 = sqrt(−2 × ln(u1)) × cos(2π × u2)
z1 = sqrt(−2 × ln(u1)) × sin(2π × u2)
```

Both z0 and z1 must be consumed in each application of the transform. Implementations must not discard z1 and draw two new values for the next normal sample, as this doubles the PRNG draw consumption relative to what component specifications account for.

`std::normal_distribution` is prohibited in reproducibility-critical paths: its algorithm is implementation-defined.

**Variable-draw-count algorithms** — including the Marsaglia polar method — are prohibited in any context where the component specification defines a fixed PRNG draw sequence. A variable draw count makes the realized sequence dependent on random values rather than configured problem parameters, breaking the ability to specify exact draw counts in component specifications.

## Decision 4: Seed Authority

The problem seed (SPEC-001 FR-6, 64-bit non-negative integer) is the exclusive entropy source for all reproducibility-critical PRNG operations associated with processing that problem.

**Prohibited entropy sources:** System time, process ID, OS random sources (`/dev/urandom`, `getrandom(2)`, `std::random_device`), and hardware entropy are prohibited as inputs to any PRNG seed in reproducibility-critical paths. Their use would make output non-reproducible from the problem seed alone.

**One seeding per reproducibility unit:** The PRNG is seeded exactly once per reproducibility unit from the problem seed. A reproducibility unit is one self-contained stochastic computation whose output must be reproducible: for the generator, one generation invocation; for a solver, one solver execution.

**Execution seed derivation:** The Worker is the authoritative owner of execution seed derivation for all backends (SPEC-005 FR-7). The execution seed passed to the solver in the SolverRequest is derived as: `execution_seed = RoutingProblem.seed`. No transformation is applied (ODR-4). This policy is uniform across all registered backends; no per-backend derivation strategy exists.

**Solver seed usage policy:** Solver specifications define seed *usage* policy only — how the solver seeds its internal PRNG from the `execution_seed` value it receives in the SolverRequest. Solver specifications do not define seed derivation policy. Seed derivation is a Worker-exclusive responsibility; solvers are consumers of the execution seed, not producers. Solver internal PRNG seeding must be deterministic and must not mix in additional entropy.

## Decision 5: Backward Compatibility Requirements

The PRNG algorithm (PCG64), approved seeding procedure, approved distribution sampling algorithms, and the uniform-float conversion formula are frozen once any component specification referencing this ADR transitions to Accepted status.

**Breaking change definition:** Changing the PRNG algorithm, seeding procedure, stream constant for any component, distribution sampling algorithm, or uniform-float conversion formula is a system-level breaking change. A breaking change is any change that causes the same seed to produce different output values for an existing specification.

**Breaking change procedure:** A breaking change requires:

1. This ADR is updated to a new date with an explicit change record documenting the previous and replacement algorithm.
2. All component specifications that reference this ADR and are affected by the change are updated with an incremented version identifier.
3. Affected components explicitly document the version boundary in their specification and implementation.

Existing seeds that have produced documented outputs cannot be assumed to produce the same output after a breaking change. Evidence reports produced before and after the change are not directly comparable without knowing which algorithm version produced each.

## Decision 6: Floating-Point Determinism Obligations

Floating-point computation in reproducibility-critical paths must use IEEE 754 double precision.

**Extended precision is prohibited:** Compiler options or instruction sets that compute intermediate floating-point results at extended precision (for example, x87 80-bit extended precision) must be disabled for code subject to this policy. The specific compiler flags required to enforce this are an implementation planning concern.

**Transcendental functions:** sin, cos, log, and sqrt are permitted and are expected to produce results that are semantically equivalent within IEEE 754 double precision across platforms. Bitwise identical results for transcendental functions are not required and are not guaranteed by the C++ standard. This is consistent with the semantic equivalence scope defined in Decision 2.

**Intermediate values:** Components must not assert bitwise equality of floating-point intermediate values across compilations or platforms. Equality assertions on floating-point values must use an epsilon appropriate to the double precision range of the expected value.

---

# Alternatives Considered

## PRNG: std::mt19937_64

### Description

Use the C++11 standard Mersenne Twister engine `std::mt19937_64` seeded with the problem seed. The C++11 standard specifies the seeding LCG, making the engine output sequence reproducible across conforming implementations. Pair with explicitly implemented range conversion algorithms (not `std::uniform_int_distribution`) to achieve cross-platform reproducibility.

### Benefits

No external dependency. Part of the C++ standard library. Period of 2^19937 − 1 is more than sufficient for any foreseeable use. Seeding with a single 64-bit value is well-defined by the standard.

### Drawbacks

The practical prohibition on `std::uniform_int_distribution` and `std::normal_distribution` (both implementation-defined) must be actively enforced. Any developer who uses these APIs on top of an `mt19937_64` instance silently reintroduces platform-dependent behavior with no compilation error. PCG64 offers better statistical properties (shorter correlation length, passes more BigCrush tests) and its specification as an external artifact is less ambiguous than C++ standard prose about implementation-defined behavior at the seeding boundary.

### Reason Not Selected

No advantage over PCG64 once `<random>` distributions are prohibited. PCG64 has a more formal, independently authored specification and better multi-seed independence, which is relevant if parallel stochastic processes are added in future work.

---

## PRNG: xoshiro256\*\*

### Description

Use xoshiro256\*\* (XOR/shift/rotate), which is fast, has a period of 2^256 − 1, passes all current randomness test suites, and is specified using only 64-bit integer arithmetic.

### Benefits

Faster than PCG64 on most architectures. No implementation-defined behavior. Well-specified algorithm. The 256-bit state provides stronger multi-stream independence properties.

### Drawbacks

Seeding xoshiro256\*\* requires a 256-bit initial state. Initializing from a 64-bit problem seed requires a seed expansion function (such as SplitMix64), adding a step that must itself be specified and frozen to maintain reproducibility. PCG64's two-parameter (state + stream) seeding from a single 64-bit value is more direct for this use case.

### Reason Not Selected

PCG64 is preferred for its simpler seeding specification when the input is a single 64-bit seed. xoshiro256\*\* is a valid alternative; if PCG64 adoption proves impractical during implementation planning, xoshiro256\*\* with SplitMix64 seed expansion is the designated fallback. Selecting xoshiro256\*\* as a fallback would require an explicit change record in this ADR before adoption.

---

## Cross-Platform Scope: Within-Platform Only (Docker Compose Linux)

### Description

Restrict the reproducibility guarantee to the single Docker Compose Linux environment. All components are compiled by the same toolchain on the same base image. Within this environment, `std::uniform_int_distribution` is consistent because the same stdlib implementation runs in all containers.

### Benefits

Allows use of `std::uniform_int_distribution` and `std::normal_distribution` without explicitly specifying algorithms. Reduces implementation surface.

### Drawbacks

The architecture document's reproducibility principle has no platform qualifier. A within-platform restriction means:

1. A developer running the system on a different OS or compiler cannot reproduce results.
2. A base image update that changes the GCC or libstdc++ version may silently change `uniform_int_distribution` behavior, making previously-seeded scenarios irreproducible without a version boundary.
3. Evidence reports become environment-dependent, undermining the project's core evidence claim.

The implementation cost of avoiding stdlib distributions is low when PCG64 is already chosen. The within-platform restriction creates fragility without benefit.

### Reason Not Selected

Violates the spirit of the reproducibility principle. The additional implementation cost of specifying explicit algorithms is proportionate given the architectural commitment.

---

## Normal Distribution: Ziggurat Method

### Description

The ziggurat method samples from the normal distribution using only integer and floating-point comparisons against a precomputed table. It avoids transcendental functions entirely, producing bitwise identical output across all platforms rather than only semantically equivalent output.

### Benefits

No transcendental functions: output is bitwise identical across platforms. Typically 1 draw per sample (rare tail rejection may consume 2). Faster than Box-Muller for high-throughput generation.

### Drawbacks

Requires a precomputed table (typically 256 entries for the Doornik-Monahan variant) that must be specified in the algorithm definition to ensure cross-platform consistency. The tail rejection path is a variable-draw-count path — rare but possible — which must be explicitly handled in component draw sequence specifications. The algorithm is significantly more complex to specify and implement correctly than Box-Muller.

### Reason Not Selected

The implementation complexity of ziggurat is disproportionate to the precision benefit for this project. The additional guarantee (bitwise identical vs. semantically equivalent for transcendental-derived values) is not required by the project's evidence goals. Box-Muller's semantic equivalence guarantee meets the requirement. If high-throughput normal sampling or strict bitwise cross-platform equality becomes a requirement, this decision should be revisited.

---

## Backward Compatibility: Versioned Soft Deprecation

### Description

When changing the PRNG algorithm, allow a transition period during which both old and new algorithms are supported. Components advertise the algorithm version they used via a version tag in evidence and manifest outputs. Cross-version evidence comparisons are permitted with a documented compatibility caveat.

### Benefits

Allows incremental migration across components. Avoids a hard system-wide cutover for a single algorithm change.

### Drawbacks

Maintaining two PRNG implementations doubles the maintenance surface. Cross-version evidence comparisons with compatibility caveats undermine the core thesis: evidence must be trustworthy and directly comparable. Version tracking adds complexity to the manifest and evidence log schema. The MVP has no deployed user base requiring migration compatibility.

### Reason Not Selected

Disproportionate complexity for MVP scope. The cleaner policy — algorithm changes are breaking changes with an explicit procedure — is more honest about the system's behavior and simpler to implement. A portfolio-stage project benefits from clear policies over migration scaffolding.

---

# Consequences

## Positive

The architecture.md reproducibility principle has concrete, enforceable implementation requirements. Components cannot silently deviate from the policy without violating a stated architectural decision.

SPEC-002 OQ-1 is resolved by this ADR. The generator specification can close OQ-1 and reference this ADR for PRNG algorithm, distribution sampling standard, and semantic equivalence scope. SPEC-002 can transition from Draft to Proposed.

The QUBO simulated annealing backend specification has a clear policy to reference when defining its seed usage model and stochastic execution behavior. Algorithm inconsistency between the generator and solver is prevented by shared policy.

Breaking changes are explicitly defined. Any algorithm change that would silently invalidate existing evidence is identified as requiring a coordinated, documented response.

PCG64's platform-independent specification ensures reproducibility across Docker Compose Linux, developer workstations, and future deployment targets without additional per-platform testing.

## Negative

PCG64 is not available in the C++ standard library. A conforming implementation must be adopted as a project dependency. Dependency selection is an implementation planning decision not resolved by this ADR.

`std::uniform_int_distribution` and `std::normal_distribution` are prohibited in reproducibility-critical paths. Developers familiar with `<random>` must use explicitly specified alternatives. This restriction requires active enforcement through code review; it does not produce a compilation error.

Box-Muller's use of transcendental functions means normal distribution samples cannot be guaranteed bitwise identical across platforms. Components must not assert bitwise equality on normal-derived values.

## Accepted Risks

PCG64 is a well-established algorithm with broad adoption in scientific computing, but it is not an ISO/IEC standard. A future requirement for a standardized PRNG would require a breaking change procedure. This risk is accepted as negligible at MVP scale.

The boundary between reproducibility-critical and non-reproducibility-critical code paths must be actively maintained. Inadvertent use of `std::normal_distribution` in a reproducibility-critical path introduces silent cross-platform divergence without a compilation error. Mitigation is code review and targeted cross-platform reproducibility tests.

Box-Muller uses transcendental functions whose results may differ by at most 1 ULP across IEEE 754 platforms. This is within the semantic equivalence standard and is accepted as a limitation of the Box-Muller choice.

---

# Architectural Impact

| Component                    | Impact |
| ---------------------------- | ------ |
| Domain Layer (C++ Core)      | Yes    |
| Synthetic Workload Generator | Yes    |
| QUBO Simulated Annealing     | Yes    |
| API Layer (C# ASP.NET)       | No     |
| Persistence                  | No     |
| Infrastructure               | Yes    |
| Configuration                | No     |
| Observability                | Yes    |

**Domain Layer (C++ Core):** All stochastic operations in Core must use PCG64 and approved distribution algorithms. Components must compile with settings that disable extended floating-point precision.

**Synthetic Workload Generator:** OQ-1 is resolved. SPEC-002 FR-3 must document the PCG64 stream constant and specify the bounded integer sampling algorithm. SPEC-002 FR-9 must reference this ADR's semantic equivalence standard. SPEC-002 FR-5 must account for Box-Muller's two-output form in draw count calculations.

**QUBO Simulated Annealing:** The annealing process must seed its PRNG from the `execution_seed` it receives in the SolverRequest. The seed usage policy — how the solver initializes its PRNG from `execution_seed` — is defined in the QUBO solver specification. The solver specification does not define derivation policy; `execution_seed` is derived by the Worker per Decision 4 and ODR-4.

**Infrastructure:** PCG64 must be adopted as a C++ project dependency. The specific implementation library (the pcg-random.org header-only reference implementation is the default candidate) must be designated during implementation planning. This is not resolved by this ADR.

**Observability:** Evidence reports that include stochastic operation outputs are reproducibility-critical. The ADR-006 observability stack must ensure problem seeds are persisted alongside evidence so that outputs can be reproduced. This is a constraint on evidence schema design, not a new schema definition.

---

# Supporting Evidence

- docs/architecture.md: "Reproducibility: Every generated scenario, solver run, scheduler decision, and report must be reproducible from persisted input, configuration, and random seed."
- SPEC-001 FR-6: Problem seed is a 64-bit non-negative integer field on the routing problem. The seed is part of canonical problem identity.
- SPEC-002 OQ-1: Generator PRNG algorithm, distribution sampling algorithm, cross-platform scope, and floating-point determinism flagged as cross-cutting questions that exceed generator specification scope; elevated to ADR.
- ADR-001: C++ is the runtime language for Core and Worker. The PRNG choice must be implementable in C++17.
- ADR-007: QUBO simulated annealing is in MVP scope. The annealing process is stochastic and requires a PRNG policy.
- O'Neill, M. E. (2014). PCG: A Family of Simple Fast Space-Efficient Statistically Good Algorithms for Random Number Generation. Harvey Mudd College. pcg-random.org. Establishes the PCG family algorithm specification.
- Lemire, D. (2019). Fast Random Integer Generation in an Interval. ACM Transactions on Modeling and Computer Simulation, 29(1). Establishes the bias-free bounded integer sampling algorithm.

No benchmark evidence exists at this stage. This is a pre-implementation architectural policy decision.

---

# Assumptions

The MVP runs under a C++17-conforming toolchain on a Linux target (Docker Compose). IEEE 754 double precision is the native floating-point type on this target.

PCG64 can be adopted as a header-only C++ dependency without licensing conflicts. This is an assumption; implementation planning must confirm the dependency decision.

Box-Muller's transcendental function results are semantically equivalent within 1 ULP across C++17 toolchains on IEEE 754 platforms. This is consistent with the IEEE 754 standard's behavior for correctly-implemented transcendental functions.

Solver stochastic operations consume PRNG draws in a fixed or explicitly documented pattern, enabling reproducibility verification. This is an assumption on solver architecture that must be confirmed during solver specification.

---

# Limitations

This ADR does not specify:

- The PCG64 stream constant for any component. The stream constant is a component-level implementation decision; it must be documented in the component specification and treated as frozen once the specification is accepted.
- How any solver uses the `execution_seed` it receives to seed its internal PRNG. Seed usage policy is a solver specification concern; seed derivation policy is a Worker responsibility (SPEC-005 FR-7, Decision 4).
- The PRNG draw ordering within any component. Draw ordering is specified per component (SPEC-002 FR-3 for the generator).
- Which PCG64 implementation library to adopt. This is an implementation planning decision.
- Floating-point determinism obligations for computations outside reproducibility-critical paths (for example, scheduler scoring arithmetic). This policy applies only to computations whose outputs are included in evidence or must be exactly reproducible from a seed.

Variable-draw-count algorithms are prohibited only in contexts where the component specification defines a fixed draw sequence. In contexts without a fixed sequence specification, variable-draw-count algorithms are permitted if the component's reproducibility requirements are met by other means.

---

# Documentation Updates

- SPEC-002: Close OQ-1 with reference to this ADR. Update FR-3 to name PCG64 and document the stream constant placeholder, name Box-Muller as the normal distribution algorithm, and name the approved bounded integer sampling algorithm. Update FR-9 to reference this ADR's semantic equivalence standard. Transition SPEC-002 status from Draft to Proposed.
- SPEC-005: FR-7 is the authoritative definition of Worker execution seed derivation. Decision 4 of this ADR has been revised to reference SPEC-005 FR-7 and ODR-4 for the approved derivation policy (`execution_seed = RoutingProblem.seed`).
- docs/architecture.md: No changes required. The reproducibility principle is already stated. This ADR provides the implementation-level binding.

---

# Review Triggers

A component cannot achieve semantic equivalence using PCG64 and the approved distribution algorithms within reasonable implementation constraints.

A future requirement demands bitwise identical output across platforms for transcendental-function-derived values. This would require revisiting the Box-Muller decision in favor of ziggurat or an equivalent transcendental-free algorithm.

The PCG64 implementation library introduces licensing, dependency management, or maintenance constraints that make it unsuitable for the project.

A significant cross-platform reproducibility failure is observed during testing on a different toolchain or OS than the primary Docker Compose target.

A breaking change to the PRNG algorithm, seeding procedure, or approved distribution algorithm is required, triggering the backward compatibility procedure defined in Decision 5.

---

# Employer Signaling

- Systems Programming
- Reliability Engineering
- Performance Engineering
- Scientific Computing

---

# Decision Summary

**Decision:** PCG64 is the standard PRNG for all reproducibility-critical stochastic computation. Semantic equivalence across conforming C++17 toolchains is the reproducibility scope. Box-Muller transform and bias-free integer algorithms are the approved distribution sampling methods. The problem seed is the exclusive entropy source. Algorithm changes are system-level breaking changes with an explicit procedure.

**Primary Benefit:** The architecture.md reproducibility principle has concrete, cross-component implementation requirements. SPEC-002 OQ-1 is resolved. The QUBO annealing backend has a clear policy to reference. System-level breaking changes are explicitly defined.

**Primary Cost:** PCG64 requires an external dependency. `std::uniform_int_distribution` and `std::normal_distribution` are prohibited in reproducibility-critical paths, requiring explicitly specified alternative algorithms.

**Evidence Supporting the Decision:** Architecture.md reproducibility principle. SPEC-001 problem seed definition. SPEC-002 OQ-1 cross-cutting scope analysis. ADR-007 QUBO simulated annealing in MVP scope. O'Neill (2014) PCG family specification. Lemire (2019) bounded integer sampling algorithm.

**Next Review Trigger:** SPEC-002 closes OQ-1 and references this ADR. QUBO solver specification references this ADR for seeding policy. A cross-platform reproducibility failure is observed. A breaking algorithm change is required.
