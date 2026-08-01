  ---
  Project DAEDALUS — Implementation Readiness Review

  Date: 2026-07-30
  Phase 0 Closure: 2026-08-01
  Scope: Full system — 14 ADRs, 22 Specifications, architecture.md, README.md

  ---
  Executive Summary

  Project DAEDALUS has completed an exceptionally thorough architecture and specification phase. All 22 specifications
  are Accepted. Thirteen of 14 ADRs are Accepted. The architecture is coherent, well-reasoned, and internally consistent
  at the subsystem level. Ownership boundaries are clear. Interface contracts are precise. The experiment harness,
  evidence model, solver contract, persistence schema, and browser client all have governing decisions and accepted
  specifications.

  The project is ready for implementation with minor prerequisites. A small set of implementation-planning decisions
  that were explicitly deferred from specification work must be resolved before coding begins. These are rapid
  decisions, not re-architecture. The remaining documentation inconsistencies are administrative and do not affect
  implementation.

  The overall assessment: the specification phase has done its job. Implementation may begin.

  Phase 0 Closure (2026-08-01): All six blocking implementation gaps identified in this review have been resolved.
  Documentation drift has been corrected. The repository structure is established. ADR-001 is now Accepted (C++20,
  CMake). All Phase 0 decisions are recorded in docs/implementation/phase-0-decisions.md. Implementation may begin
  at Phase 1.

  ---
  1. Governance Readiness Assessment

  Acceptance State

  ┌────────────────┬───────┬──────────┬──────────┬──────┐
  │ Artifact Type  │ Total │ Accepted │ Proposed │ Gap  │
  ├────────────────┼───────┼──────────┼──────────┼──────┤
  │ Specifications │ 22    │ 22       │ 0        │ None │
  ├────────────────┼───────┼──────────┼──────────┼──────┤
  │ ADRs           │ 14    │ 14       │ 0        │ None │
  └────────────────┴───────┴──────────┴──────────┴──────┘

  ADR-001 (C++ as Runtime Language): At initial review, ADR-001 was the only governance artifact not formally Accepted.

  Resolution (Phase 0): ADR-001 promoted to Accepted. C++20 is the C++ standard. CMake is the build system. See
  docs/implementation/phase-0-decisions.md Decision 1.

  Amendment Tracking

  Four ADR-mandated amendments were identified. All four have been applied at Phase 0 Closure.

  1. SPEC-001 Assumptions (required by ADR-012 Decision 3): Amended in place. The one-problem-one-job prose now
  acknowledges the experiment-context relaxation per ADR-012 Decision 3. RESOLVED.

  2. SPEC-020 OQ-1 (required by SPEC-012 Documentation Updates): OQ-1 body updated from Blocking to Resolved,
  referencing SPEC-012 FR-19 through FR-23. RESOLVED.

  3. architecture.md (required by ADR-014 Documentation Updates): Browser client node added to System Context diagram;
  web-ui container added to Container Topology; specification/ADR counts updated to 22/14; Browser Client section added
  to Major Components. RESOLVED.

  4. SPEC-008 CORS constraint (required by SPEC-021 Architectural Impact and ADR-014 Decision 5): Constraint 10 added to
  SPEC-008 Constraints section; ADR-014 entry added to SPEC-008 Documentation Updates Required. RESOLVED.

  SPEC-021 OQ-2, OQ-3, OQ-4 and SPEC-022 OQ-1, OQ-2, OQ-4 have been resolved in-place per ADR-014. RESOLVED.

  Ownership Boundaries

  Ownership is clear and internally consistent:
  - API: Writes routing_problems, scheduler_configs, jobs (initial), and all five experiment tables
  - Worker: Sole writer of all Evidence Log artifact tables (decision records, solver run records, quality evaluation
  records, failure records, report metadata records)
  - Core: Domain validation authority (ADR-009 dual-validation model)
  - CLI: Experiment orchestration executor; no direct database access
  - Browser Client: Passive consumer of SPEC-008 endpoints only

  No ownership ambiguities or contested boundaries were found.

  ---
  2. Architecture Readiness Assessment

  Implementability

  The architecture is implementable. The component model is stable. Every component has:
  - A defined responsibility boundary
  - A defined non-responsibility set
  - An identified language and framework
  - An identified deployment unit

  The multi-language architecture (C++, C#, Python, TypeScript) is intentional and governed by ADR-001 through ADR-005
  and ADR-014. Each language is confined to a specific tier with clear inter-tier boundaries.

  Subsystem Stability

  All subsystem boundaries are stable:
  - C++ Core/Worker boundary: clear (Core = domain library, Worker = execution service)
  - API/Worker boundary: asynchronous via RabbitMQ (ADR-003)
  - Worker/Python Adapter boundary: JSON over HTTP (ADR-005, SPEC-017 FR-2)
  - API/Browser Client boundary: REST over HTTP with CORS (ADR-014 Decision 5)
  - Evidence Log write authority: Worker-exclusive, no exceptions (SPEC-006 FR-1.3)

  Dependency Direction

  Dependency direction is well-governed:
  - Worker depends on Core (in-process)
  - API depends on PostgreSQL and RabbitMQ
  - Worker depends on PostgreSQL and RabbitMQ and Python Adapter
  - CLI depends on API (HTTP only)
  - Browser Client depends on API (HTTP only)
  - No circular dependencies exist

  Interface Definition

  Interfaces are sufficiently defined for implementation:
  - SolverRequest/SolverResponse (SPEC-004, SPEC-017): complete
  - Job message payload schema (SPEC-008 FR-5, SPEC-005 FR-3): complete including optional backend_id
  - AMQP metadata schema (ADR-011): W3C TraceContext in application_headers
  - API endpoint contracts (SPEC-008): complete for all job, scheduler config, experiment, and evidence endpoints
  - PostgreSQL schema (SPEC-012): complete DDL for all 13 entity types

  Remaining Architectural Risks

  ADR-011 OQ-1 (C++ OTel SDK AMQP integration): The C++ SDK does not provide a built-in AMQP carrier. The concrete
  integration approach must be confirmed during Worker implementation. The designated fallback (persist trace context in
  the job record, requires SPEC-006 revision under ODR-6) is viable and does not block the critical execution path.
  This risk is contained and managed.

  SPEC-003 OQ-2 (Capability profile registration mechanism): PostgreSQL is not the store. The mechanism — compiled-in
  registry, configuration file, or another approach — must be resolved before Scheduler implementation. This is a
  one-decision implementation planning item, not a re-architecture.

  No architectural blockers require ADR-level resolution before implementation begins.

  ---
  3. Specification Readiness Assessment

  Summary

  ┌───────────────────────────────────────┬──────────┬───────────────────────────────────────────────────┐
  │             Specification              │  Status  │                       Notes                       │
  ├────────────────────────────────────────┼──────────┼───────────────────────────────────────────────────┤
  │ SPEC-001 Routing Problem Model         │ Accepted │ Minor Assumptions amendment required (ADR-012)    │
  ├────────────────────────────────────────┼──────────┼───────────────────────────────────────────────────┤
  │ SPEC-002 Synthetic Workload Generator  │ Accepted │ OQ-1 resolved (ADR-010); OQ-2 resolved (SPEC-016) │
  ├────────────────────────────────────────┼──────────┼───────────────────────────────────────────────────┤
  │ SPEC-003 Scheduler Policy Engine       │ Accepted │ OQ-2 resolved (Phase 0 Decision 5); OQ-4 resolved (Phase 0 Decision 6) │
  ├────────────────────────────────────────┼──────────┼───────────────────────────────────────────────────┤
  │ SPEC-004 Solver Contract               │ Accepted │ Python transport OQ resolved (ADR-005/SPEC-017)   │
  ├────────────────────────────────────────┼──────────┼───────────────────────────────────────────────────┤
  │ SPEC-005 Worker Execution Lifecycle    │ Accepted │ OTel integration is implementation planning       │
  ├────────────────────────────────────────┼──────────┼───────────────────────────────────────────────────┤
  │ SPEC-006 Evidence Log                  │ Accepted │ Complete                                          │
  ├────────────────────────────────────────┼──────────┼───────────────────────────────────────────────────┤
  │ SPEC-007 Core Quality Evaluation       │ Accepted │ Complete                                          │
  ├────────────────────────────────────────┼──────────┼───────────────────────────────────────────────────┤
  │ SPEC-008 API Control Plane             │ Accepted │ OQ-7, OQ-8 non-blocking for MVP Fixed Mode        │
  ├────────────────────────────────────────┼──────────┼───────────────────────────────────────────────────┤
  │ SPEC-009 Report Generator              │ Accepted │ Complete                                               │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-010 Core Feature Extraction       │ Accepted │ Complete                                               │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-011 Backend Solver Specifications │ Accepted │ Parent spec for SPEC-013/014/015                       │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-012 Persistence Schema            │ Accepted │ OQ-1, OQ-2 are implementation planning                 │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-013 Nearest Neighbor Solver       │ Accepted │ Complete                                               │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-014 Greedy Insertion Solver       │ Accepted │ Complete                                               │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-015 QUBO Simulated Annealing      │ Accepted │ Complete                                               │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-016 Daedalus CLI                  │ Accepted │ OQ-1 blocks job list only; OQ-5 deferred               │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-017 Python Solver Adapter         │ Accepted │ Complete                                               │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-018 QAOA Qiskit Solver            │ Accepted │ Complete                                               │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-019 Quantum Hardware Solver       │ Accepted │ Behavioral contract only; execution deferred (ADR-007) │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-020 Benchmark Experiment Harness  │ Accepted │ OQ-1 text stale; blocking OQs resolved in SPEC-012     │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-021 Web UI                        │ Accepted │ ADR-014 resolves all OQs                               │
  ├────────────────────────────────────────┼──────────┼────────────────────────────────────────────────────────┤
  │ SPEC-022 Experiment Dashboard          │ Accepted │ ADR-014 resolves all OQs                               │
  └────────────────────────────────────────┴──────────┴────────────────────────────────────────────────────────┘

  API Contracts

  Complete for all job lifecycle operations, scheduler configuration management, experiment and benchmark submission,
  trial orchestration, evidence collection, and artifact retrieval. SPEC-008 includes full endpoint definitions for FR-1
  through FR-25.

  Persistence Contracts

  Complete. SPEC-012 provides full DDL for all 13 entity types including the five experiment tables (FR-19 through
  FR-23). Write ownership is mapped per component. FK ordering for initialization and deletion is defined.

  Worker Contracts

  Complete. SPEC-005 defines the full job consumption lifecycle, Core invocation, solver dispatch, evidence persistence,
  and OTel instrumentation requirements.

  Browser Contracts

  Complete. ADR-014 resolves all structural open questions for SPEC-021 and SPEC-022. Technology stack, deployment
  model, API URL discovery, and observability posture are all governed.

  Solver Contracts

  Complete. All five backends have accepted specifications. The normalized SolverContract (SPEC-004) governs all
  backends. PCG64 reproducibility requirements (ADR-010) are specified down to the distribution sampling algorithm
  level.

  ---
  4. Cross-Artifact Consistency Assessment

  ADR Consistency

  All 14 ADRs are internally consistent and mutually non-contradictory. The evolutionary chain — ADR-012 extending the
  component model, ADR-013 adding backend targeting, ADR-014 adding the browser client — follows clear dependency order.
  Later ADRs amend earlier specifications rather than contradicting them.

  Specification Consistency

  22 specifications are consistent with their governing ADRs. The ADR-013 amendment trail is fully applied: backend_id
  and selection_mode appear correctly in SPEC-003, SPEC-005, SPEC-008, SPEC-009, SPEC-012, and SPEC-016.

  Documentation Drift

  Three consistency gaps exist:

  1. SPEC-001 Assumptions: The one-problem-one-job constraint prose has not been updated per ADR-012 Decision 3.
  Implementation note: the SPEC-012 schema already supports the relaxed model; this is prose only.
  2. SPEC-020 OQ-1 status: Body text says "Blocking"; acceptance checklist says resolved; SPEC-012 confirms resolution.
  Minor administrative drift.
  3. architecture.md: Missing browser client in both diagrams; specification/ADR count is stale (shows 20/13, should be
  22/14). The Governing Specifications section timestamp is 2026-06-23, predating SPEC-021, SPEC-022, and ADR-014.

  None of these gaps affect implementation. They should be corrected before the first implementation PR is opened to
  establish a clean baseline.

  Responsibility Ownership

  No contested ownership boundaries were found. Write authority for every table is assigned to exactly one component.
  The Evidence Log write-authority restriction (SPEC-006 FR-1.3) is consistently respected across all specifications,
  including the experiment harness (ADR-012 Decision 5 API-mediated collection).

  ---
  5. Open Question Classification

  ┌────────────────────────────┬───────────────┬───────────────────────┬───────────────────────────────────────────┐
  │             OQ             │ Specification │    Classification     │                 Rationale                 │
  ├─────────────────────────────┼───────────────┼──────────────────────┼───────────────────────────────────────────┤
  │ ADR-001: C++ standard +     │ ADR-001       │ Blocking             │ Must be resolved to configure the         │
  │ build system                │               │ implementation       │ toolchain                                 │
  ├────────────────────────────────┼───────────────┼──────────────────────┼─────────────────────────────────────────┤
  │ ADR-002: .NET version + data   │ ADR-002       │ Blocking             │ Must be resolved to set up the API      │
  │ access                         │               │ implementation       │ project                                 │
  ├────────────────────────────────┼───────────────┼──────────────────────┼─────────────────────────────────────────┤
  │ ADR-011 OQ-1: C++ OTel AMQP    │ ADR-011       │ Blocking Worker      │ Must resolve before Worker OTel         │
  │ integration                    │               │ implementation       │ instrumentation; fallback defined       │
  ├────────────────────────────────┼───────────────┼──────────────────────┼─────────────────────────────────────────┤
  │ SPEC-003 OQ-2: Capability      │ SPEC-003      │ Blocking             │ Scheduler cannot be implemented without │
  │ profile registration           │               │ implementation       │  knowing how backends are registered    │
  ├────────────────────────────────┼───────────────┼──────────────────────┼─────────────────────────────────────────┤
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-003 OQ-4: Scoring formula  │ SPEC-003      │ Blocking             │ Scheduler scoring cannot be            │
  │ per mode                        │               │ implementation       │ implemented without a formula          │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-012 OQ-1: Default config   │ SPEC-012      │ Blocking             │ Schema initialization script requires  │
  │ UUID stability                  │               │ implementation       │ a stable UUID mechanism                │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-012 OQ-2: Schema migration │ SPEC-012      │ Blocking             │ Schema changes after initial creation  │
  │  tooling                        │               │ implementation       │ require a migration approach           │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-001 OQ-4/5:                │ SPEC-001      │ Implementation       │ Optional fields; deferred by design    │
  │ Stop/problem/vehicle names      │               │ discovery            │                                        │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-008 OQ-3: API versioning   │ SPEC-008      │ Non-blocking         │ MVP versioning works without this      │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-008 OQ-4: Config           │ SPEC-008      │ Non-blocking         │ Read-only after creation is already    │
  │ mutability                      │               │                      │ specified                              │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-008 OQ-7: Generated Mode   │ SPEC-008      │ Non-blocking         │ MVP scoped to Fixed Mode; CLI rejects  │
  │ creation                        │               │                      │ Generated Mode manifests               │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-008 OQ-8:                  │               │                      │                                        │
  │ SchedulerRejected CLI           │ SPEC-008      │ Non-blocking         │ Full lifecycle handling is post-MVP    │
  │ notification                    │               │                      │                                        │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-016 OQ-1: GET /v1/jobs     │ SPEC-016      │ Non-blocking         │ Blocks job list command only           │
  │ list                            │               │                      │                                        │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-016 OQ-4: --wait default   │ SPEC-016      │ Non-blocking         │ Opt-in behavior already specified      │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-016 OQ-5: Generated Mode   │ SPEC-016      │ Non-blocking         │ Depends on SPEC-008 OQ-7; deferred     │
  │ CLI                             │               │                      │                                        │
  ├─────────────────────────────────┼───────────────┼──────────────────────┼────────────────────────────────────────┤
  │ SPEC-020 OQ-3: Statistical      │ SPEC-020      │ Future enhancement   │ Post-MVP publication guidance          │
  │ threshold                       │               │                      │                                        │
  ├─────────────────────────────────┼───────────────┼─────────────────────┼────────────────────────────────────────┤
  │ SPEC-020 OQ-4: Hypothesis       │ SPEC-020      │ Future enhancement  │ Post-MVP statistical rigor             │
  │ testing                         │               │                     │                                        │
  ├─────────────────────────────────┼───────────────┼─────────────────────┼────────────────────────────────────────┤
  │ SPEC-020 OQ-5: Parallel         │ SPEC-020      │ Future enhancement  │ Post-MVP scale                         │
  │ execution                       │               │                     │                                        │
  ├─────────────────────────────────┼───────────────┼─────────────────────┼────────────────────────────────────────┤
  │ SPEC-020 OQ-6: Publication      │ SPEC-020      │ Future enhancement  │ Post-MVP                               │
  │ formats                         │               │                     │                                        │
  ├─────────────────────────────────┼───────────────┼─────────────────────┼────────────────────────────────────────┤
  │ SPEC-020 OQ-7: Hardware         │ SPEC-020      │ Future enhancement  │ Gated on ADR-007 review trigger        │
  │ qualification                   │               │                     │                                        │
  ├─────────────────────────────────┼───────────────┼─────────────────────┼────────────────────────────────────────┤
  │ SPEC-020 OQ-8: Experiment       │ SPEC-020      │ Future enhancement  │ Non-MVP; hardware experiments only     │
  │ cancellation                    │               │                     │                                        │
  ├─────────────────────────────────┼───────────────┼─────────────────────┼────────────────────────────────────────┤
  │ SPEC-020 OQ-9: Hardware cost    │ SPEC-020      │ Future enhancement  │ Non-MVP                                │
  │ controls                        │               │                     │                                        │
  ├─────────────────────────────────┼───────────────┼─────────────────────┼────────────────────────────────────────┤
  │ SPEC-012 OQ-3: Phase 2          │ SPEC-012      │ Implementation      │ Remediation strategy discovered during │
  │ incomplete write                │               │ discovery           │  implementation                        │
  ├─────────────────────────────────┼───────────────┼─────────────────────┼────────────────────────────────────────┤
  │ SPEC-012 OQ-4: Capability       │ SPEC-012      │ Implementation      │ Contingent on SPEC-003 OQ-2 selection  │
  │ profile schema contingency      │               │ discovery           │                                        │
  └─────────────────────────────────┴───────────────┴─────────────────────┴────────────────────────────────────────┘

  ---
  6. Comprehensive Gap Analysis

  Blocking Gaps — All Resolved at Phase 0 Closure

  BG-1: ADR-001 Not Accepted; C++ Standard and Build System Not Selected — RESOLVED
  Resolution: ADR-001 promoted to Accepted. C++20 standard; CMake build system. See Phase 0 Decision 1.

  (Original) Description: ADR-001 is "Proposed." C++ standard version (C++17/20/23) and build system (CMake/Meson/Bazel) are
  explicitly deferred. The PCG64 implementation library (ADR-010) is also not selected.
  - Impact: Worker and Core cannot be set up without a toolchain. PCG64 reproducibility requirements cannot be
  implemented.
  - Priority: Critical — first pre-implementation decision
  - Disposition: Implementation planning decision. Recommend C++20 for range support and concepts, CMake for ecosystem
  compatibility, the pcg-random.org header-only reference implementation for PCG64.

  BG-2: .NET Version and Data Access Approach Not Selected — RESOLVED (Phase 0 Decision 2: .NET 10 LTS; Dapper; Npgsql; DbUp)
  - Description: ADR-002 defers .NET version (8 LTS vs. 9+) and data access approach (Dapper vs. EF Core). The data
  access choice affects schema migration strategy.
  - Impact: API project cannot be initialized.
  - Priority: Critical — first pre-implementation decision for API track
  - Disposition: Implementation planning decision. Recommend .NET 10 LTS for stability. Data access choice drives the
  SPEC-012 OQ-2 migration tooling answer.

  BG-3: SPEC-003 OQ-2 — Capability Profile Registration Mechanism — RESOLVED (Phase 0 Decision 5: JSON files in config/backends/)
  - Description: How backend capability profiles enter the Scheduler's runtime registry at startup is unresolved.
  PostgreSQL is explicitly not the store (SPEC-012 FR-2). Options: compiled-in registry, configuration file (YAML/JSON),
  or another approach.
  - Impact: The Scheduler cannot be initialized without knowing where its backend registry comes from.
  - Priority: High — blocks Scheduler implementation (Phase 2)
  - Disposition: Implementation planning decision. Recommend a compiled-in registry initialized at startup — profiles
  are stable at MVP scale and a configuration file adds a deployment artifact without benefit.

  BG-4: SPEC-003 OQ-4 — Scoring Formula Per Objective Mode — RESOLVED (Phase 0 Decision 6: normalized-penalty scoring)
  - Description: The scoring formula for each objective mode (CheapestValid, FastestValid, Balanced, BestQuality,
  DeadlineAware, BudgetCapped, ExperimentalExecution) is deferred. Constraints are well-defined (deterministic,
  capability-only, backend-neutral). The specific formula is not.
  - Impact: The Scheduler cannot produce scores without a formula.
  - Priority: High — blocks Scheduler implementation
  - Disposition: Implementation planning decision. The scoring inputs (capability profiles, workload features, scheduler
  config weights) and constraints are fully specified. The formula follows from these; no ADR is required.

  BG-5: SPEC-012 OQ-1 — Default Scheduler Configuration UUID Stability — RESOLVED (Phase 0 Decision 4: 2f5ce394-c4e4-5324-b842-f1ff47aafc68)
  - Description: The default scheduler configuration must be present before the API accepts its first request. Its UUID
  must be stable across deployments for reproducibility. The mechanism for ensuring UUID stability (hardcoded UUID,
  deterministic generation, seed-based) is unresolved.
  - Impact: Schema initialization script cannot be finalized.
  - Priority: High — blocks schema initialization
  - Disposition: Implementation planning decision. Recommend a hardcoded UUID with a well-known value documented in
  SPEC-012 and embedded in the schema seed script.

  BG-6: SPEC-012 OQ-2 — Schema Migration Tooling — RESOLVED (Phase 0 Decision 3: DbUp with versioned SQL in infrastructure/postgres/migrations/)
  - Description: ADR-004 explicitly defers migration tooling. SPEC-012 requires migration tooling once the schema is
  declared stable. Options for the .NET ecosystem: Flyway, Liquibase, EF Core migrations, raw versioned SQL scripts.
  - Impact: Schema changes after initial creation have no governance mechanism.
  - Priority: High — required before any schema change after initial setup
  - Disposition: This is the joint answer with BG-2's data access choice. If EF Core is chosen, its migrations are the
  natural answer. If Dapper, a SQL-script-based tool (Flyway) is appropriate.

  Pre-Implementation Improvements — All Complete (Phase 0 Closure)

  PI-1: Amend SPEC-001 Assumptions — DONE. Applied at Phase 0 Closure.
  - Description: Remove the "exactly one job" prose and add the experiment-context relaxation per ADR-012 Decision 3.
  - Impact: Documentation accuracy; avoids confusion during experiment harness implementation.
  - Priority: Medium — complete before harness phase begins
  - Disposition: Minor in-place amendment. No functional change; schema already supports this.

  PI-2: Update architecture.md — DONE. Applied at Phase 0 Closure.
  - Description: Add browser client node to System Context and Container Topology diagrams; update counts to 22 specs/14
  ADRs.
  - Impact: Document coherence; onboarding accuracy.
  - Priority: Low — administrative
  - Disposition: Direct edit before first implementation commit.

  PI-3: Update SPEC-020 OQ-1 Status Text — DONE. Applied at Phase 0 Closure.
  - Description: Update OQ-1 body from "Blocking" to "Resolved — SPEC-012 FR-19 through FR-23."
  - Priority: Low — administrative
  - Disposition: Direct edit.

  PI-4: Promote ADR-001 to Accepted — DONE. Applied at Phase 0 Closure.
  - Description: Acceptance ceremony for an already-established decision.
  - Priority: Medium — governance hygiene before implementation
  - Disposition: Status change only; update to record selected C++ standard and build system.

  Implementation-Time Discoveries

  IT-1: ADR-011 OQ-1 — C++ OTel AMQP carrier approach. Confirm AMQP client library exposes application_headers and
  select the concrete OTel C++ SDK integration approach. Fallback to Alternative 3 (PostgreSQL job record) is
  pre-defined.

  IT-2: SPEC-012 OQ-3 — Phase 2 incomplete write detection. Define remediation for decision_records rows with null
  actual_outcome when a quality_evaluation_records row exists. Discovery during Worker implementation.

  IT-3: SPEC-012 OQ-4 — Capability profile schema contingency. If SPEC-003 OQ-2 resolves to a database-backed registry,
  SPEC-012 requires revision. Contingent on BG-3 resolution.

  IT-4: SPEC-001 OQ-4/5 — Stop/problem/vehicle names. Optional fields for reporting; resolved at implementation time.

  IT-5: SPEC-016 OQ-1 — GET /v1/jobs list endpoint. If implemented, job list command is enabled; if not, command is
  deferred. Implementation-time decision.

  Future Architecture

  FA-1: Quantum hardware execution (SPEC-019). ADR-007 explicitly defers this beyond MVP. SPEC-019 specifies the
  behavioral contract only.

  FA-2: Generated Mode experiments (SPEC-008 OQ-7, SPEC-016 OQ-5). Blocked on routing problem creation API for generated
  workloads. Deferred post-MVP.

  FA-3: Advanced statistical analysis (SPEC-020 OQ-4). Hypothesis testing, confidence intervals. Post-MVP.

  FA-4: Parallel trial execution (SPEC-020 OQ-5). Horizontal Worker scaling. Post-MVP.

  FA-5: Learned AI scheduler policy (README Deferred). Future enhancement.

  FA-6: Multi-tenant security, Kubernetes deployment (README Deferred). Future enhancement.

  ---
  7. Proposed Implementation Phasing

  Phase 0: Implementation Planning Checkpoint — COMPLETE (2026-08-01)

  All six blocking gaps (BG-1 through BG-6) resolved. All documentation drift corrected. Repository structure
  established. Decisions recorded in docs/implementation/phase-0-decisions.md.

  ---
  Phase 1: Infrastructure Foundation (parallel, 1–2 weeks)

  Objective: Stand up the Docker Compose environment, initialize the schema, and establish the C++ and C# project
  structures.

  Participating Specs: SPEC-012 (DDL)
  Participating ADRs: ADR-003, ADR-004, ADR-006

  Track A (Infrastructure): Docker Compose configuration for PostgreSQL, RabbitMQ, OTel Collector, Prometheus, Grafana.
  Health checks and credential management.

  Track B (Schema): SPEC-012 DDL initialization script. All 13 entity types. Schema seed (default scheduler config).
  Migration tooling configured.

  Track C (C++ project setup): CMake scaffolding, PCG64 dependency, OpenTelemetry C++ SDK integration, libpqxx
  dependency.

  Track D (C# project setup): .NET project with Npgsql, OpenTelemetry .NET SDK, RabbitMQ client library. CORS configured
  for http://localhost:3000.

  Expected Deliverables: Running Docker Compose environment; initialized schema; buildable C++ and C# project shells

  Completion Criteria: docker compose up starts all services cleanly; schema is queryable; both project shells build

  ---
  Phase 2: Core Domain Layer (mostly sequential, 2–3 weeks)

  Objective: Implement the C++ domain library that all other components depend on.

  Participating Specs: SPEC-001, SPEC-002, SPEC-004, SPEC-007, SPEC-010, SPEC-011
  Participating ADRs: ADR-008, ADR-009, ADR-010

  Tasks (mostly sequential — each layer depends on the previous):
  1. Routing problem model (SPEC-001) — C++ types, validation, Haversine distance
  2. Synthetic workload generator (SPEC-002) — PCG64 seeded generation per ADR-010
  3. Solver contract interface (SPEC-004) — abstract C++ types
  4. Feature extraction (SPEC-010) — fleet size, stop density, demand variance, time window pressure
  5. Quality evaluation (SPEC-007) — route distance, capacity violation detection, regret calculation
  6. Scheduler domain types and eligibility logic (SPEC-003) — Phase 1/Phase 2 filtering, scoring framework

  Parallelizable within this phase: Feature extraction and quality evaluation are independent once the problem model
  exists.

  Expected Deliverables: Linkable C++ Core library with unit test coverage

  Completion Criteria: Problem generation, validation, feature extraction, quality evaluation, and solver eligibility
  all pass unit tests; PCG64 reproducibility verified with known seeds

  ---
  Phase 3: Classical Solver Backends (parallel, 1–2 weeks)

  Objective: Implement the three C++ solver backends.

  Participating Specs: SPEC-013, SPEC-014, SPEC-015
  Participating ADRs: ADR-008, ADR-010

  Track A: Nearest Neighbor solver (SPEC-013) — deterministic, no PRNG dependency
  Track B: Greedy Insertion solver (SPEC-014) — deterministic, no PRNG dependency
  Track C: QUBO Simulated Annealing solver (SPEC-015) — stochastic, PCG64 seeded per ADR-010

  All three tracks can run in parallel after Phase 2 delivers the solver contract interface.

  Expected Deliverables: Three working C++ solvers conforming to SolverContract

  Completion Criteria: All three solvers produce valid route plans; QUBO SA output is reproducible from identical seeds;
  nearest neighbor and greedy insertion are deterministic

  ---
  Phase 4: Worker and API — Core Job Pipeline (parallel, 2–4 weeks)

  Objective: Implement the end-to-end job execution pipeline. This phase delivers the system's primary value.

  Participating Specs: SPEC-005, SPEC-006, SPEC-008, SPEC-009
  Participating ADRs: ADR-003, ADR-009, ADR-011

  Track A (Worker — C++):
  - RabbitMQ consumer (AMQP client, message parsing, ACK/NACK)
  - ADR-011 OQ-1 resolution — W3C TraceContext extraction from AMQP headers (or fallback)
  - Core invocation with problem load, feature extraction, Scheduler call
  - Solver dispatch (in-process for Phases 1–3 backends)
  - Execution timeout enforcement
  - Evidence persistence (SPEC-006 FR-1 through FR-8)
  - Report generation (SPEC-009 HTML sections 1–4)
  - OTel span emission for all nine required spans
  - Job lifecycle state management

  Track B (API — C#):
  - Job submission endpoint (SPEC-008 FR-2 through FR-7)
  - Job status polling (SPEC-008 FR-8, FR-9)
  - Report metadata and access (SPEC-008 FR-11 through FR-13)
  - Scheduler configuration endpoints (SPEC-008 FR-15, FR-16)
  - W3C TraceContext injection into AMQP message headers (ADR-011)
  - Dual validation (ADR-009) for problem model

  Track A and Track B can proceed largely in parallel. Track A depends on the schema (Phase 1) and Core library (Phase
  2) being complete. Track B depends only on the schema.

  Expected Deliverables: End-to-end job execution: submit via API → queue → Worker → solvers → evidence → report →
  API-readable result

  Completion Criteria: A routing problem submitted to POST /v1/jobs with a standard scheduler config progresses through
  all lifecycle states to Completed; evidence report is generated; all nine OTel spans appear in Grafana Tempo or
  equivalent backend; cross-process trace linkage is navigable

  ---
  Phase 5: Python Solver Adapter and QAOA Backends (1–2 weeks)

  Objective: Implement the Python execution path and QAOA local simulation.

  Participating Specs: SPEC-017, SPEC-018, SPEC-019
  Participating ADRs: ADR-005, ADR-007, ADR-010

  Tasks:
  - Python Solver Adapter container (FastAPI or Flask, POST /v1/solve, GET /health)
  - SolverRequest/SolverResponse JSON deserialization (SPEC-017)
  - PCG64 seeding via numpy SeedSequence per ADR-010 and SPEC-017 FR-9
  - QAOA Qiskit local simulation (SPEC-018) — Qiskit Aer backend
  - SPEC-019 behavioral contract implementation stub (behavioral contract only; no hardware execution)
  - Worker HTTP client integration (dispatch to Python Adapter on Python backend selection)

  Expected Deliverables: qaoa-qiskit backend producing valid solver responses via HTTP from Worker dispatch;
  qaoa-hardware backend stub returning provisional responses

  Completion Criteria: An experiment trial targeting qaoa-qiskit completes end-to-end; QAOA shots are reproducible from
  identical execution seeds; Python adapter health endpoint responds correctly

  ---
  Phase 6: CLI (parallel with Phase 5, 1–2 weeks)

  Objective: Implement the developer-facing CLI.

  Participating Specs: SPEC-016, SPEC-002
  Participating ADRs: ADR-001, ADR-010

  Tasks (most parallelizable within this phase):
  - C++ binary scaffolding
  - Workload generator command (problem generate) — SPEC-002 embedded as library
  - Problem submission (problem submit)
  - Job submission and status (job submit, job status, job report)
  - Scheduler config commands
  - HTTP client for API integration (no direct DB access)
  - Structured JSON debug logging (DAEDALUS_LOG=debug)

  Expected Deliverables: Developer can run daedalus problem generate | daedalus job submit to trigger a full job and
  inspect results

  Completion Criteria: All SPEC-016 single-invocation commands work against a running Docker Compose environment;
  workload generator produces reproducible problems from identical seeds

  ---
  Phase 7: Experiment Harness (depends on Phases 5 + 6, 2–3 weeks)

  Objective: Implement the multi-backend experiment and benchmark pipeline.

  Participating Specs: SPEC-008 (FR-18 through FR-24), SPEC-012 (FR-19 through FR-23), SPEC-016 (FR-12, FR-17 through
  FR-22), SPEC-020
  Participating ADRs: ADR-012, ADR-013

  Tasks (mostly sequential — experiment API must precede CLI harness):
  1. API experiment endpoints: manifest submission, trial record creation, trial submission linkage, evidence collection
  trigger, status retrieval, artifact retrieval (SPEC-008 FR-18 through FR-24)
  2. CLI experiment commands: benchmark submit, experiment run, experiment status, experiment summary (SPEC-016 FR-12,
  FR-17 through FR-22)
  3. Backend targeting in trial job submissions (backend_id per ADR-013)
  4. Evidence collection via API-mediated collect-evidence endpoint (ADR-012 Decision 5)
  5. Benchmark summary computation and artifact persistence (SPEC-020 FR-14)

  Expected Deliverables: daedalus experiment run <manifest.json> completes a Fixed Mode experiment with multiple
  backends and multiple repetitions; benchmark summary is queryable via API

  Completion Criteria: A two-backend, three-repetition Fixed Mode experiment completes end-to-end; each trial's evidence
  is attributed to the correct backend (selection_mode = explicitly_targeted); experiment summary artifact is persisted
  and retrievable

  ---
  Phase 8: Browser Client (can begin after Phase 4 partial, 2–3 weeks)

  Objective: Implement the Web UI and Experiment Dashboard.

  Participating Specs: SPEC-021, SPEC-022
  Participating ADRs: ADR-014

  Tasks (mostly parallelizable within this phase):
  - React/TypeScript/Vite project setup with shadcn/ui, Tailwind CSS, TanStack Table, TanStack Query, React Router
  - Multi-stage Docker Compose web-ui container (Nginx)
  - CORS configuration on API for http://localhost:3000
  - SPEC-021 views: job submission form, job status polling, report viewer
  - SPEC-022 views: experiment list, experiment detail, trial matrix, solver comparison, execution metadata

  Can begin: As soon as POST /v1/jobs and GET /v1/jobs/{id} are available from Phase 4. Dashboard views require Phase 7
  experiment endpoints.

  Expected Deliverables: Browser application accessible at http://localhost:3000; job submission, status polling, and
  report viewing functional; experiment trial matrix rendering correctly

  Completion Criteria: A developer can submit a job through the browser, observe its progress, view the evidence report,
  and browse an experiment's trial results in the dashboard without touching the CLI

  ---
  8. Coding Agent Readiness Assessment

  Implementation Clarity

  The specifications provide exceptional implementation clarity in most areas:

  High clarity (agent-ready):
  - Routing problem model (SPEC-001): all fields, validation rules, Haversine formula, PCG64 seeding — precise
  - Persistence schema (SPEC-012): complete DDL, FK constraints, nullable columns, CHECK constraints, write ownership
  map
  - Solver contract (SPEC-004): SolverRequest and SolverResponse field definitions, success/failure/timeout outcome
  semantics
  - Evidence Log (SPEC-006): table-level write authority, field definitions, idempotency patterns (FR-12)
  - Worker lifecycle (SPEC-005): phase-by-phase execution sequence, state transitions, OTel span requirements
  - API endpoints (SPEC-008): HTTP method, request/response schemas, error codes, idempotency rules, FR-2 through FR-25
  - PCG64 distribution algorithms (ADR-010): uniform float conversion formula, Box-Muller, Lemire bounded integer —
  precise to the bit level
  - Backend solvers (SPEC-013, 014, 015, 017, 018): algorithm specification, PRNG usage, draw ordering, outcome types

  Moderate clarity (requires implementation planning resolution):
  - Scheduler scoring (SPEC-003 OQ-4): resolved by Phase 0 Decision 6 (normalized-penalty scoring formula)
  - Backend registry initialization (SPEC-003 OQ-2): resolved by Phase 0 Decision 5 (JSON files in config/backends/)
  - OTel AMQP carrier (ADR-011 OQ-1): approach not specified; experimentation required

  Lower clarity (human judgment required):
  - SPEC-003 OQ-4 formula — resolved by Phase 0 Decision 6
  - SPEC-012 OQ-2 migration tooling — resolved by Phase 0 Decision 3 (DbUp)
  - SPEC-012 OQ-1 UUID mechanism — resolved by Phase 0 Decision 4 (UUIDv5)

  Ownership Ambiguity

  None identified. Every write operation has a single designated component. The Evidence Log write authority restriction
  (SPEC-006 FR-1.3) is unambiguous.

  Contract Ambiguity

  Minor: The exact Nginx configuration for web-ui beyond try_files $uri /index.html is implementation planning (ADR-014
  Limitations). The npm package manager is not specified (ADR-014 Limitations). Both are trivial implementation
  decisions.

  Sequencing Ambiguity

  Phasing dependencies are clear:
  - Core library must precede Worker and Solver implementations
  - Schema must precede both API and Worker database access
  - Phase 4 job pipeline must precede Phase 7 experiment harness
  - Phase 4 partial must precede Phase 8 Browser Client (for API endpoints)

  Mandatory Human Review Points

  The following require human review before autonomous coding agents proceed:

  1. Scheduling formula (SPEC-003 OQ-4) — resolved by Phase 0 Decision 6 (normalized-penalty scoring). Implementation
  should be reviewed for formula correctness before the Scheduler scoring implementation merges
  2. ADR-011 OQ-1 resolution — if AMQP carrier approach requires a custom C++ implementation, it should be reviewed for
  correctness before merging
  3. Phase 2 → Phase 3 boundary — the solver contract interface (SPEC-004 C++ abstract type) is the shared dependency
  for all solver implementations; its design should be reviewed before downstream work begins
  4. Evidence report HTML structure (SPEC-009) — Section 4 (Quality and Regret) mathematical correctness should be
  human-verified
  5. SPEC-020 experiment state machine transitions — the experiment lifecycle state machine (Created → Running →
  Completed/Failed) edge cases under CLI interruption deserve human review before the harness is production-ready

  ---
  9. Project Direction Summary

  Current State

  Project DAEDALUS has completed 22 accepted specifications and 14 accepted ADRs governing a complete hybrid
  optimization runtime. The architecture phase is finished. The system is fully designed: routing problem model,
  synthetic workload generation, feature extraction, scheduler policy engine, classical and quantum-adjacent solver
  backends, worker execution lifecycle, evidence log, quality evaluation, persistence schema, API control plane, report
  generator, CLI, experiment and benchmark harness, and browser client.

  The governance work has been substantive. Five ADR-level architectural decisions were required after the initial
  specification work: ADR-010 (reproducibility policy), ADR-011 (trace propagation), ADR-012 (experiment architecture),
  ADR-013 (backend targeting), and ADR-014 (browser client). Each resolved a genuine cross-specification conflict or
  architectural gap that would have caused implementation problems without explicit governance.

  Architectural Evolution

  The major architectural decisions during specification development were made for clear, traceable reasons:

  Backend Neutrality (ADR-008): All solvers access the system through a normalized contract. This decision makes the
  scheduler a pure policy engine and makes evidence comparable across backends. Without it, the system's core thesis —
  "classical methods are sufficient" — could not be demonstrated on a consistent evidentiary basis.

  API-First Architecture (ADR-002, SPEC-008): The API is the sole external interface and the durable state authority for
  experiment state (ADR-012). This decision kept the Worker and CLI free of state management concerns, preserving clean
  component boundaries throughout the evolution from simple job execution to complex multi-trial experiments.

  Python Adapter Boundary (ADR-005): Python is confined to a separate container with a defined HTTP transport. This
  decision preserved the C++ Core's execution characteristics while granting access to the Qiskit ecosystem without
  embedding CPython in the Worker process.

  Experiment Architecture (ADR-012): When SPEC-020 introduced multi-backend experiments, it created five
  cross-specification conflicts (one-problem-one-job, evidence write authority, CLI state ownership, Worker scope).
  ADR-012 resolved all five without breaking any existing accepted specification and without introducing a new
  deployment unit. The instance-sharing invariant was correctly identified as a correctness requirement, not a
  performance optimization.

  Backend Targeting (ADR-013): The optional backend_id field on POST /v1/jobs satisfies the experiment harness's need to
  produce attributable evidence without introducing experiment awareness into the Worker. The design is
  general-purpose: any caller can target a specific backend, not only the experiment harness.

  Evidence Integrity (ADR-010): The PCG64 reproducibility policy resolves a silent failure mode — different components
  using incompatible PRNG algorithms would produce results that look reproducible but aren't. Specifying the algorithm,
  seeding procedure, and distribution sampling formulas to the bit level is necessary given the system's thesis that
  evidence is trustworthy.

  Browser Architecture (ADR-014): React/TypeScript as a unified SPA with TanStack Query for polling and TanStack Table
  for the trial matrix. The Trial Matrix (SPEC-022 FR-4) required a headless table; that structural requirement drove
  the component library choice. The decision to serve from a dedicated Nginx container preserves the ADR-002 API
  boundary.

  Current Heading

  The project is now attempting to build a working Docker Compose runtime that demonstrates:
  1. A routing job submitted via API or CLI executes on a selected solver backend
  2. The Scheduler's decision — and rejection of more expensive backends when cheaper ones suffice — is captured in a
  structured evidence report
  3. A multi-backend experiment produces comparable, attributable cross-solver evidence
  4. The full stack is observable: distributed traces, structured logs, Prometheus metrics

  Implementation should prioritize the core execution path (Phases 1–4) before the experiment harness or browser client.
  The first working end-to-end job is the primary milestone. Everything else builds on that.

  Guiding principles for implementation:
  - The evidence model is load-bearing. The Evidence Log write authority (SPEC-006 FR-1.3) and the scheduler
  selection_mode attribution (ADR-013) are not implementation details — they are the system's proof mechanism.
  - PCG64 reproducibility is non-negotiable (ADR-010). Violations are silent and will undermine benchmark validity.
  - Backend Neutrality must be enforced in code review. Any special-case logic for a specific backend in the Scheduler
  violates ADR-008.
  - The Worker is experiment-unaware by design (ADR-012 Decision 4). Any attempt to add experiment context to Worker
  logic should be rejected.

  Implementation Outlook

  Readiness: High. The specification phase has been unusually thorough. The remaining pre-implementation decisions are
  rapid and unambiguous once the planning checkpoint is scheduled.

  Major remaining risks:
  1. ADR-011 OQ-1 — C++ OTel AMQP integration may require a non-trivial custom carrier. The fallback is defined and
  viable, but the primary approach must be confirmed experimentally. This is Phase 4's principal technical uncertainty.
  2. Multi-language toolchain complexity — Four languages (C++, C#, Python, TypeScript), multiple build systems, and
  Docker Compose integration. The toolchain setup (Phase 0/Phase 1) is the highest integration risk of the project.
  3. SPEC-003 OQ-4 formula — Resolved by Phase 0 Decision 6 (normalized-penalty scoring). The implementation
  must match the formula precisely; coefficient values (including risk_weight=0.1, provisional) should be reviewed
  against benchmark results during Phase 3.

  Expected complexity: High for Phase 2 (C++ Core with PCG64 reproducibility at specification level) and Phase 4 (Worker
  with cross-process OTel tracing). Moderate for API, CLI, and Python Adapter. Low for individual solver backends
  (well-specified algorithms with clear inputs and outputs).

  Confidence: High that the project can successfully transition into implementation. The architecture is sound. The
  specifications are complete. The remaining decisions are well-scoped. No re-architecture is required.

  ---
  Overall Readiness Recommendation

  Ready. Phase 0 Complete.

  At initial review (2026-07-30), the project was ready with minor prerequisites: six implementation-planning decisions
  (BG-1 through BG-6) deferred from the specification phase, and three documentation inconsistencies requiring correction.

  Phase 0 Closure (2026-08-01): All six blocking gaps are resolved. All four documentation inconsistencies are corrected.
  The repository structure is established. All Phase 0 decisions are recorded in docs/implementation/phase-0-decisions.md.

  Phase 1 infrastructure work may begin immediately. The architecture is well-governed, the contracts are precise, the
  component boundaries are unambiguous, and the toolchain is decided. The project is ready to build.