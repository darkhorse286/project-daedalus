# ADR-009: Domain Validation Authority

## Status

Accepted

**Date:** 2026-06-07

**Related Feature(s):** SPEC-001, Daedalus API, Daedalus Core

**Related ADR(s):** ADR-001, ADR-002, ADR-008

---

# Context

Project DAEDALUS has two runtime components that process routing problems:

- The API (C# ASP.NET Core) receives routing problems from callers at submission time.
- The Worker calls the Core (C++) to load the routing problem from PostgreSQL and process it before solver execution.

The architecture document states that the C++ Core owns domain logic. SPEC-001 defines domain validation rules — coordinate bounds, time window consistency, capacity feasibility, and service duration bounds — that are semantic constraints on the routing problem domain, not merely API contract constraints.

Two questions arise from this structure:

1. Which component is the authoritative validator for domain rules?
2. What happens if the two components disagree on whether a routing problem is valid?

Three options exist:

**API-only validation:** The API validates at submission time. The Core trusts the API's decision and proceeds without re-validation.

**Core-only validation:** The API performs structural validation (JSON schema and type checking). Domain validation runs exclusively in the Core at execution time. Invalid problems are discovered only when a job begins executing.

**Dual validation:** The API validates for fast caller feedback. The Core re-validates as the canonical authority before solver execution. If they disagree, the disagreement is an observable fault.

---

# Decision

Dual validation is adopted for Project DAEDALUS.

**API role:** The API performs structural validation (JSON schema conformance, field type checking, required field presence) and fast domain validation (coordinate bounds, time window consistency, capacity feasibility, service duration non-negativity, scheduler_config_id existence). API validation provides immediate, field-level feedback to callers and prevents invalid problems from entering the queue.

**Core role:** The Core performs authoritative domain validation when loading the routing problem from PostgreSQL, before feature extraction and solver execution begin. If the Core determines the routing problem is invalid, the job fails. The failure is recorded as a structured validation error in the evidence log. The job does not proceed to solver execution.

**Divergence:** If the Core rejects a problem that the API accepted, this is an observable fault condition. The Core's decision is final. The job status is updated to failed with a validation error code. Investigation of the divergence is required to identify the inconsistency between API and Core validation logic.

---

# Alternatives Considered

## API-Only Validation

### Description

The API validates all domain rules at submission time. The Core trusts that any problem loaded from PostgreSQL is valid and proceeds directly to feature extraction.

### Benefits

Validation logic exists in one place. No duplicate maintenance. No divergence fault mode.

### Drawbacks

The Core must trust the API unconditionally. Any path that bypasses the API — direct queue injection for testing, migration scripts, future tooling — sends unvalidated problems to the Core. Solver execution failures from invalid problems surface without a structured validation error, making them harder to diagnose. The architecture principle "Core owns domain logic" is violated.

### Reason Not Selected

Creates an unconditional trust dependency from the Core to the API. Any bypass of the API path removes all domain validation from the system. Architecturally unsound.

---

## Core-Only Validation

### Description

The API performs structural validation only (JSON schema conformance, field types, required field presence). Domain validation runs exclusively in the Core at execution time.

### Benefits

Authoritative domain validation in the correct component. Single implementation of validation logic in C++.

### Drawbacks

Invalid problems enter the queue and are discovered only when a job begins executing, which may be seconds to minutes after submission. HTTP 202 Accepted is semantically misleading: the caller has no indication that their problem may fail domain validation. Caller iteration time for correcting invalid submissions is significantly worse.

### Reason Not Selected

HTTP 202 should indicate a successfully accepted submission. Returning 202 for a submission that will subsequently fail domain validation misrepresents the outcome to callers. Fast rejection is a required caller experience for a developer-facing API.

---

## Shared Validation Library

### Description

Extract validation logic into a shared library consumed by both the C# API and C++ Core.

### Benefits

Single implementation of validation rules. No divergence risk.

### Drawbacks

C# and C++ cannot share a native library without a significant cross-language binding layer. A JSON-serialized rule set or a separate validation service would add complexity disproportionate to the MVP scope.

### Reason Not Selected

Implementation cost is disproportionate to the MVP scope. The Architectural Maturity Rule applies.

---

# Consequences

## Positive

The Core maintains independent protection against invalid routing problems reaching solver execution, regardless of how the problem entered the system.

Callers receive immediate, field-level validation feedback at submission time.

The divergence fault mode (API accepts, Core rejects) is detectable, observable, and identifiable through the evidence log.

## Negative

Validation rules must be maintained in two places: C# in the API and C++ in the Core. This creates a risk of divergence as rules change.

The divergence fault mode is observable but not automatically correctable. It requires manual investigation.

## Accepted Risks

API and Core validation rules may diverge over time without explicit synchronization discipline. This must be mitigated by treating Core validation failures as high-priority defects and maintaining test coverage for the divergence case.

The divergence fault mode (HTTP 202 followed by job failure due to Core validation) may be confusing for callers who do not poll job status. This is an accepted trade-off given the MVP scope and developer-demonstration context.

---

# Architectural Impact

| Component      | Impact |
| -------------- | ------ |
| Domain Layer   | Yes    |
| API Layer      | Yes    |
| Persistence    | No     |
| Infrastructure | No     |
| Configuration  | No     |
| Observability  | Yes    |
| Security       | Yes    |
| Deployment     | No     |

**Domain Layer:** The Core must re-validate the routing problem on load. The C++ validation logic must cover all domain rules defined in SPEC-001.

**API Layer:** The C# API must implement fast domain validation covering all rules defined in SPEC-001. Validation must run before persistence.

**Observability:** Core validation rejections must produce observable failures in the evidence log and on the problem.load trace span. The failure must be correlatable with the originating submission.

**Security:** Dual validation prevents unvalidated problems from reaching solver execution regardless of how they entered the system.

---

# Supporting Evidence

- docs/architecture.md: "the runtime core owns domain logic"
- SPEC-001: defines domain validation rules and references this ADR in FR-8, FR-9, and Constraints
- ADR-002: establishes C# as the API layer language

No benchmark evidence exists at this stage. This is a pre-implementation decision.

---

# Assumptions

The domain validation rules are fully defined by SPEC-001 and can be implemented independently in C# and C++ without a shared library.

The divergence fault mode (API accepts, Core rejects) will occur infrequently enough that manual investigation is a proportionate response at MVP scale.

---

# Limitations

Validation rules must be kept consistent between the C# API and C++ Core implementations. Consistency is a maintenance discipline, not a technical guarantee at this stage.

No automatic mechanism to detect API-Core validation divergence exists at this stage. Detection occurs only when the fault mode surfaces at runtime.

---

# Documentation Updates

- SPEC-001: references this ADR in FR-8, FR-9, Constraints, and the Core Validation Rejection failure mode
- SPEC-001 Testability: Core validation behaviors are required test targets

---

# Review Triggers

A significant divergence between API and Core validation logic is discovered during implementation.

A new domain validation rule is added to SPEC-001 requiring implementation in both components.

The MVP scope is extended to include paths that bypass the API (for example, direct queue injection tooling), requiring the Core to serve as the first and only validation layer for those paths.

---

# Employer Signaling

- System Design
- Reliability Engineering
- API Contract Design

---

# Decision Summary

**Decision:** Dual validation. API validates for fast caller feedback. Core validates as the canonical authority before solver execution.

**Primary Benefit:** Callers receive immediate rejection feedback. The Core is protected against invalid problems regardless of how they entered the system.

**Primary Cost:** Validation logic must be maintained in two places (C# and C++). Divergence is a maintenance risk.

**Evidence Supporting the Decision:** Architecture principle "Core owns domain logic." SPEC-001 domain validation requirements.

**Next Review Trigger:** A divergence between API and Core validation is discovered, or a new SPEC-001 validation rule is added that requires updates to both layers.
