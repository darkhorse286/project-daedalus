# ADR-012: Experiment Execution Architecture

## Status

Proposed

**Date:** 2026-06-21

**Related Feature(s):** Benchmark and Experiment Harness (SPEC-020)

**Related ADR(s):** ADR-004, ADR-008, ADR-009, ADR-010

---

# Context

SPEC-020 (Benchmark and Experiment Harness, Accepted) defines a systematic experiment execution model for Project DAEDALUS. The harness enables reproducible multi-solver, multi-problem comparison experiments with repeated trials, statistical aggregation, and artifact generation. SPEC-020 introduces six system-level questions that require architectural decisions not answerable by inference from existing ADRs:

**1. Experiment orchestration ownership.** The existing component model assigns the API to "submit and observe" and the Worker to "execute." Neither role encompasses coordinating multi-job trial sequences, managing experiment lifecycle state across many jobs, or producing experiment-level artifacts. Who owns orchestration?

**2. Experiment state persistence.** SPEC-016 OQ-3 (Accepted) stated that experiment results are local manifest only and require no API persistence. SPEC-020 defines manifests, trial records, aggregate statistics, and artifact payloads that require durable persistence across a potentially long-running experiment. What is the state authority?

**3. Problem-to-job cardinality in experiment context.** SPEC-008 FR-7 (Accepted) and SPEC-001 Assumptions (Accepted) both state that each routing problem is associated with exactly one job. SPEC-020 FR-4 requires that in Generated Mode, for each (problem_config_index, repetition_index) pair, a single routing problem instance is submitted to all solvers in the solver set — meaning multiple jobs reference the same problem_id. These are in direct conflict. Must the one-problem-one-job constraint hold in experiment context?

**4. Worker experiment awareness.** The Worker executes individual jobs. Experiments are a coordination concept above the job level. Should the Worker be extended to understand experiments, or should experiment-awareness be excluded from the Worker boundary?

**5. Evidence collection mechanism.** SPEC-006 FR-1.3 (Accepted) restricts writing to Evidence Log artifact tables to the Worker. The experiment harness must collect per-trial evidence from completed jobs. What mechanism allows evidence collection without violating SPEC-006 FR-1.3?

**6. Deployment topology.** An experiment coordination function could be implemented as a new deployment unit — a dedicated experiment service or background task runner. Is a new unit warranted for MVP?

These six questions are architectural in scope: they establish component boundaries, define write authority, and govern cross-specification interactions. They are not implementation details. No existing ADR answers them, and they cannot be left to per-specification interpretation without creating irreconcilable conflicts between SPEC-001, SPEC-006, SPEC-008, SPEC-016, and SPEC-020.

**Cross-specification conflicts requiring resolution:**

| Conflict | Existing Authority | SPEC-020 Requirement |
|---|---|---|
| One-problem-one-job | SPEC-001 Assumptions; SPEC-008 FR-7 | Instance-sharing invariant (FR-4): one `problem_id` per (problem_config_index, repetition_index), shared across all backends |
| Local manifest only | SPEC-016 OQ-3 | API persists all experiment state: manifest, trial records, collected evidence, statistics, artifact payloads |
| Distinct problem_ids per backend | SPEC-016 FR-12 Assumptions | All backends in a solver set submit trials against the same `problem_id` within a repetition |
| Single-invocation CLI | SPEC-016 Assumptions | Long-running orchestration loop for `daedalus benchmark run` |
| Worker-only evidence writes | SPEC-006 FR-1.3 | Harness collects evidence from completed trial jobs |

The cross-solver quality comparability constraint is the technical forcing function for the instance-sharing conflict: SPEC-007 FR-7 states that `hindsight_quality` (total route distance in km) is dimensionally comparable only within the same routing problem. Cross-solver comparison requires that all backends evaluated the same geometry with the same problem_id. If backends solve independently created problems with distinct problem_ids, cross-solver quality comparison is not valid. The instance-sharing invariant is not an optimization; it is the prerequisite for the experiment to produce valid comparative evidence.

---

# Decision

Six architectural decisions govern experiment execution in Project DAEDALUS.

## Decision 1: CLI Is the Orchestration Executor

The CLI process (SPEC-016) is the experiment orchestration executor for all experiment and benchmark commands. The harness orchestration loop runs in the CLI process. The CLI:

- Submits the experiment manifest to the API before any trials are submitted
- Iterates through the trial set in defined submission order (problem_config_index ascending, repetition_index ascending, solver_set_index ascending)
- Submits each trial as a job via the SPEC-008 job submission endpoint
- Polls the API job status endpoints for trial completion
- Notifies the API when each trial reaches a terminal state, triggering API-side evidence collection and trial record update

This decision scopes the CLI's orchestration role precisely: the CLI is the submission and polling agent, not the state store. Long-running execution — hardware experiments with queue wait times, large trial counts — is a CLI runtime concern, not a new service concern.

**Scoped exception to SPEC-016 Assumptions:** SPEC-016 Assumptions state that "The CLI is single-invocation per command. It does not run as a daemon or background process." The `daedalus benchmark run` command is a scoped exception to this assumption. For the duration of a benchmark run, the CLI runs as a long-lived orchestration process managing trial submission and polling. This exception applies only to benchmark and experiment commands; all other CLI commands retain single-invocation behavior.

## Decision 2: API Is the Durable State Authority for Experiment State

The API is the sole durable state authority for all experiment and benchmark state. All experiment state that must survive the CLI process lifetime is persisted through the API:

- Experiment and benchmark manifests
- Trial records (status, job_id, evidence collected)
- Per-(problem, solver) aggregate evidence
- Experiment summary artifacts
- Benchmark summary artifacts

This decision supersedes SPEC-016 OQ-3 ("Local manifest only. Experiment results are captured at run time and saved to `<name>.result.json`. No API persistence of experiment records is required at MVP scope."). OQ-3 was written before SPEC-020 existed. SPEC-020's domain model — reproducible experiments, durable trial records, statistical aggregation computed at completion, benchmark summary updates triggered by experiment completion — requires persistent state that survives the CLI session. A local result file cannot support multi-session replay, benchmark-level aggregation, or API-served artifact retrieval.

The API's persistence responsibility for experiment state is not a violation of its documented non-responsibilities (solver execution, heavy optimization logic, scheduler scoring, workload feature extraction). Experiment state persistence is analogous to job state persistence, which is already an API responsibility.

## Decision 3: One Routing Problem May Be Referenced by Multiple Jobs in Experiment Context

In Generated Mode experiments, for each (problem_config_index, repetition_index) combination, the harness generates exactly one routing problem instance with one problem_id. All jobs submitted for backends in the solver set for that (problem_config_index, repetition_index) reference that same problem_id. This is the instance-sharing invariant (SPEC-020 FR-4).

This decision relaxes the one-problem-one-job constraint in experiment context only:

- **SPEC-008 FR-7** ("At MVP scope, each problem is associated with exactly one job") is relaxed from a universal constraint to a default for non-experiment job submissions. In experiment context, the constraint does not apply.
- **SPEC-001 Assumptions** ("A routing problem is created once and associated with exactly one job. No routing problem is shared between jobs.") is relaxed for experiment context. The routing problem model itself (SPEC-001 FR-1 through FR-16) is not changed; only the cardinality assumption is scoped.
- **SPEC-016 FR-12 Assumptions** ("Because each `POST /v1/jobs` produces a distinct `problem_id`, the problems are not shared at the persistence layer — they are identical in content but independently persisted") is superseded for experiment context. In SPEC-020 Generated Mode, backends within the same (problem_config_index, repetition_index) share one problem_id, not independently persisted copies.

**Schema status:** SPEC-012 FR-6 (jobs table) does not impose a UNIQUE constraint on `jobs.problem_id`. The existing schema already supports multiple jobs referencing the same routing problem. The one-problem-one-job restriction exists only in prose (SPEC-008 FR-7, SPEC-001 Assumptions); it is not a physical schema constraint. No schema migration is required to implement instance-sharing.

**Why the invariant is required:** SPEC-007 FR-7 establishes that `hindsight_quality = total_route_distance_km` is dimensionally comparable only within the same routing problem. Two jobs submitting identical problem content with distinct problem_ids produce geometrically identical routing problems but independent solver executions; cross-solver comparison of their `hindsight_quality` values is technically valid. However, this approach requires the harness to generate and persist N copies of each problem geometry per backend — creating N-fold redundancy with no evidence benefit and making "same problem" an application-layer assertion rather than a schema-enforced guarantee. The instance-sharing invariant eliminates the redundancy while preserving the comparability requirement: one problem instance, shared across all backends in a repetition, is both correct and efficient.

## Decision 4: Worker Remains Experiment-Unaware

The Worker executes individual jobs. The Worker has no knowledge of experiments, trial sets, repetition counts, benchmark_ids, or experiment lifecycle state. From the Worker's perspective, a job submitted by the experiment harness is indistinguishable from any other job submission.

This decision maintains the Worker's existing operational boundary (SPEC-005): consume a job from the queue, load the problem and scheduler configuration, invoke Core, execute the selected solver backend, persist evidence, generate a report, and mark the job complete. No new Worker responsibilities are introduced for experiment execution.

**Consequence for harness architecture:** Because the Worker is experiment-unaware, the harness cannot rely on Worker-side coordination for trial sequencing or experiment progress. The CLI polling model (Decision 1) is the necessary consequence of Worker experiment-unawareness: the CLI monitors job status endpoints to detect trial completion and trigger evidence collection.

## Decision 5: Evidence Collection Occurs Through API-Mediated Collection

The experiment harness collects per-trial evidence by triggering the API to read from Evidence Log tables after a trial's job reaches a terminal state. The API reads from the Evidence Log and writes the collected evidence to experiment-specific trial records. The CLI does not read the Evidence Log directly.

**SPEC-006 FR-1.3 compliance:** SPEC-006 FR-1.3 states "No component other than the Worker (SPEC-005) may write to Evidence Log artifact tables after job creation." The API-mediated collection does not write to Evidence Log tables. The API reads from Evidence Log tables (reading is unrestricted per SPEC-006 FR-1.3, which restricts writing only) and writes the collected evidence to experiment trial records — a separate table category governed by SPEC-012-R1, not SPEC-006.

**SPEC-016 Constraints compliance:** SPEC-016 Constraints state "The CLI must not access PostgreSQL directly. All data access is through the Daedalus API." The CLI triggers evidence collection through the API; it does not read PostgreSQL directly.

The mechanism: when the CLI determines (via job status polling) that a trial job has reached a terminal state, the CLI calls a designated API endpoint that reads the relevant Evidence Log records for that `job_id` and persists the collected evidence in the experiment trial record. Experiment trial records are experiment persistence tables (to be defined in SPEC-012-R1), not Evidence Log tables.

## Decision 6: No Additional Deployment Unit for MVP

No new deployment unit is introduced for experiment execution at MVP scope. The experiment harness is implemented within the existing Docker Compose topology: the CLI process, the API container, the Worker container(s), and the PostgreSQL instance. No dedicated experiment service, background task runner, or experiment coordinator container is introduced.

This decision bounds MVP infrastructure complexity. The CLI as orchestration executor (Decision 1) and API as durable state authority (Decision 2) together provide a sufficient execution model without a new service. A new deployment unit would introduce container lifecycle management, inter-service coordination, and operational overhead not warranted at MVP scale.

---

# Alternatives Considered

## Alternative 1A: Server-Side Orchestration (API Background Tasks)

### Description

The experiment orchestration loop runs as a background task in the API process. When the CLI submits an experiment manifest, the API spawns a background task that manages trial submission, job polling, and evidence collection for the experiment lifecycle.

### Benefits

The CLI exits after submission; the experiment runs server-side without requiring a long-lived CLI process. Multiple experiments can run concurrently without multiple CLI sessions. Experiment progress is observable through the API without CLI coordination.

### Drawbacks

Introduces background task lifecycle management within the API container: task supervision, restart on API container restart, persistence of in-flight state during container failure. Background tasks require careful error isolation to prevent a single failing experiment from affecting API health. The API's existing non-responsibilities (SPEC-008 FR-1) do not include execution coordination; adding it blurs the API's boundary with the Worker.

### Reason Not Selected

The CLI-as-executor model (Decision 1) is simpler at MVP scale. The CLI already owns experiment orchestration scope (SPEC-016 scope table: "Experiment orchestration: multi-job submission and result collection"). Background task orchestration adds infrastructure complexity — task supervision, restart semantics, concurrent task isolation — that a CLI polling loop avoids. At MVP scale, experiment workloads (tens to hundreds of trials) are tractable within a CLI session. The exception to the single-invocation assumption is bounded and scoped.

---

## Alternative 1B: Dedicated Experiment Coordinator Service

### Description

A new container — `experiment-coordinator` — manages experiment execution. The CLI submits manifests to the coordinator. The coordinator submits trials to the API, monitors job status, and updates experiment state.

### Benefits

Clean separation of experiment orchestration from CLI session lifetime. The coordinator can be replicated for parallel experiment execution. Long-running hardware experiments (with multi-hour queue waits) do not require an open CLI session.

### Drawbacks

Adds a new deployment unit: a new container, a new service in Docker Compose, a new API surface between CLI and coordinator, and new operational failure modes (coordinator crash mid-experiment, state recovery, coordinator-to-API authentication). Violates Decision 6 (no additional deployment unit for MVP). Requires a new specification governing the coordinator.

### Reason Not Selected

The coordinator's responsibilities are fully satisfiable by CLI + API at MVP scale. The new unit adds infrastructure overhead without enabling functionality the CLI cannot provide. Post-MVP parallel execution or hardware-scale experiments may justify a coordinator; MVP does not.

---

## Alternative 3: Submit Identical Problem Content with Distinct Problem_IDs per Backend

### Description

Instead of sharing a problem_id across backends in a repetition, the harness submits identical problem content as a separate `POST /v1/jobs` for each backend, producing distinct problem_ids. This preserves the SPEC-001 one-problem-one-job invariant without modification.

### Benefits

No conflict with SPEC-001 Assumptions or SPEC-008 FR-7. No prose amendment required for existing specifications.

### Drawbacks

N distinct problem submissions per repetition for N backends. For an experiment with 5 backends, 5 identical routing problems are persisted per repetition — consuming storage, generating audit records, and requiring problem load at job execution time without evidence benefit. Cross-solver quality comparison is technically valid (identical geometry), but the problems are distinct database records with distinct problem_ids, making evidence traceability more complex. If any backend's problem submission fails, that backend's trial references a different problem record than others in the same repetition. The harness must reconcile "same content, different problem_id" as equivalent for comparison purposes, which is an application-layer assertion rather than a schema-enforced guarantee.

### Reason Not Selected

The instance-sharing invariant is architecturally cleaner and less error-prone. One problem_id per (problem_config_index, repetition_index) makes the "same problem" relationship explicit in the schema. The redundant-persistence approach trades schema clarity for prose compatibility — a poor tradeoff when SPEC-012 already supports multiple jobs per problem without constraint.

---

## Alternative 5A: Worker Writes Experiment Evidence Directly

### Description

The Worker is extended to detect that a job was submitted by the experiment harness (e.g., via an `experiment_id` field in the job message) and writes experiment trial evidence directly to experiment tables after job completion.

### Benefits

Eliminates the CLI polling / API collection coordination. The Worker's evidence-persistence path (SPEC-005 FR-16) is already the most complete evidence access point; extending it to write experiment records would be architecturally compact.

### Drawbacks

Violates Decision 4 (Worker remains experiment-unaware). Adds experiment schema knowledge to the Worker, coupling the C++ Worker implementation to experiment table definitions. The Worker's evidence-persistence path is designed for job-scoped artifacts; experiment-scoped aggregation (per-experiment, per-solver summary) is outside the Worker's per-job model. The Worker would need to coordinate with other Workers processing other trials in the same experiment, which is a distributed coordination problem the Worker has no mechanism for.

### Reason Not Selected

Worker experiment-unawareness (Decision 4) is essential for maintaining the Worker's clean per-job boundary. The Worker's responsibilities (SPEC-005) are fully job-scoped; experiment-level coordination is above that boundary. API-mediated collection (Decision 5) preserves the Worker boundary while enabling evidence aggregation.

---

## Alternative 5B: CLI Reads Evidence Log Directly

### Description

The CLI reads Evidence Log tables directly from PostgreSQL to collect per-trial evidence after each job completes.

### Benefits

Eliminates the need for a new API endpoint for evidence collection. The CLI can read any table it needs.

### Drawbacks

Directly violates SPEC-016 Constraints: "The CLI must not access PostgreSQL directly. All data access is through the Daedalus API." This constraint is not an incidental preference; it enforces the API as the sole access layer for all data, enabling validation, authorization, and observability at the API boundary. Direct CLI-to-database access bypasses all three.

### Reason Not Selected

The constraint is authoritative and accepted. API-mediated collection (Decision 5) satisfies the same evidence-access requirement without violating it.

---

# Consequences

## Positive

- The CLI's orchestration scope (already established in SPEC-016's scope table) is formally confirmed as the execution executor, with a bounded, scoped exception to the single-invocation assumption.
- The API's persistence responsibility for experiment state follows naturally from its existing role as the system's durable state authority (ADR-004). No new architectural patterns are introduced.
- The instance-sharing invariant (Decision 3) eliminates N-fold problem record redundancy while satisfying the SPEC-007 FR-7 cross-solver comparability requirement. The SPEC-012 schema already supports it without constraint change.
- Worker experiment-unawareness (Decision 4) preserves the Worker's clean per-job boundary and avoids coupling C++ Worker implementation to experiment schema evolution.
- API-mediated evidence collection (Decision 5) satisfies SPEC-016's direct-database-access constraint and SPEC-006's Evidence Log write-authority restriction simultaneously.
- No new deployment unit (Decision 6) keeps the MVP nine-container Docker Compose topology stable.

## Negative

- The long-running CLI process (Decision 1 scoped exception) introduces an availability dependency: if the CLI process exits mid-experiment, in-flight trials continue executing but the harness cannot submit new trials, poll for completion, or trigger evidence collection until the CLI is restarted. Experiment recovery after CLI interruption is not defined by this ADR; it is deferred to SPEC-020 implementation.
- The one-problem-one-job constraint (SPEC-001 Assumptions, SPEC-008 FR-7) must be relaxed in two accepted specifications. This requires SPEC-008-R1 and SPEC-001 Assumptions amendment. Until those amendments are Accepted, the constraint conflict is documented here but unresolved in the source specifications.
- API-mediated evidence collection (Decision 5) requires a new API endpoint category not defined in SPEC-008's current Accepted text. SPEC-008-R1 must add trial evidence collection endpoints. Until SPEC-008-R1 is Accepted, the experiment harness cannot be fully implemented against a governed API contract.
- The experiment persistence schema (SPEC-020 OQ-1, Blocking) is not resolved by this ADR. Experiment tables are referenced here as "experiment-specific trial records" and "experiment persistence tables," but their DDL is deferred to SPEC-012-R1.

## Accepted Risks

- The scoped exception to SPEC-016's single-invocation assumption applies to `daedalus benchmark run` only. Accidental misapplication to other CLI commands is a governance risk, not a technical risk; the exception is bounded by this ADR.
- Evidence collection latency — the delay between job terminal state and API evidence collection completion — creates a window in which experiment state and the Evidence Log are temporarily inconsistent. This is an expected and acceptable operational condition at MVP scale.

---

# Architectural Impact

| Component | Impact | Nature |
|---|---|---|
| CLI (SPEC-016) | Yes | New benchmark orchestration command; long-running process exception; trial submission and polling loop |
| API Layer (ASP.NET Core) | Yes | New endpoints: experiment and benchmark manifest submission, experiment status, trial evidence collection trigger, artifact retrieval |
| Persistence (PostgreSQL) | Yes | New experiment and benchmark tables (defined in SPEC-012-R1): experiment manifests, trial records, artifact payloads |
| Worker (C++ Runtime) | No | Worker remains job-scoped and experiment-unaware |
| Core (C++ Domain) | No | No changes |
| Evidence Log (SPEC-006) | No | API reads from Evidence Log; write authority is not changed |
| Observability | Yes | Experiment progress events, trial progress events, evidence completeness metrics (SPEC-020 FR-15) |
| Docker Compose Topology | No | No new containers; existing nine-container topology is unchanged |
| Solver Contract (ADR-008) | No | Solver backends remain experiment-unaware; the solver contract is not changed |
| Randomness (ADR-010) | No | SPEC-020 FR-8 trial seed derivation is ADR-010 compliant; ADR-010 is not changed |

---

# Supporting Evidence

**SPEC-020 FR-4 (Instance-Sharing Invariant):** "For each (problem_config_index, repetition_index) combination, the harness generates exactly one routing problem instance and assigns it a single `problem_id`. All backends in the solver set for that (problem_config_index, repetition_index) submit trials against the same `problem_id`."

**SPEC-007 FR-7 (Cross-Solver Comparability Constraint):** "`hindsight_quality` is dimensionally stable for comparison only within the same routing problem. Cross-problem comparison of hindsight_quality values is not meaningful." This is the technical forcing function for Decision 3.

**SPEC-016 Scope Table:** "Experiment orchestration (multi-job submission and result collection)" is owned by SPEC-016. The CLI's role as orchestration executor (Decision 1) is already designated in the Accepted specification.

**SPEC-016 Constraints:** "The CLI must not access PostgreSQL directly. All data access is through the Daedalus API." This constraint is the primary driver of Alternative 5B's rejection and Decision 5's structure.

**SPEC-006 FR-1.3:** "No component other than the Worker (SPEC-005) may write to Evidence Log artifact tables after job creation." Reading is unrestricted. This establishes that API-mediated evidence collection (Decision 5) complies: the API reads from Evidence Log tables and writes to experiment tables, both of which are permitted.

**SPEC-012 FR-6 (`jobs` table):** No UNIQUE constraint on `jobs.problem_id`. The schema permits multiple jobs to reference the same routing problem. The one-problem-one-job restriction is prose-only (SPEC-008 FR-7, SPEC-001 Assumptions), not a physical schema constraint.

**SPEC-020 Domain Concept:** "The API persists all experiment state: the manifest, trial records, collected evidence, computed statistics, and artifact payloads. All durable state is API-backed; the CLI process is the orchestration executor but not the state authority." This passage in the Accepted SPEC-020 directly states Decisions 1 and 2.

**SPEC-020 Non-Requirements:** "The Worker executes individual jobs; it is unaware of experiments." This passage in the Accepted SPEC-020 directly states Decision 4.

**docs/architecture.md (API Non-responsibilities):** "solver execution, heavy optimization logic, scheduler scoring, workload feature extraction." Experiment state persistence is absent from this list. Decision 2 is consistent with the API's defined responsibilities.

---

# Assumptions

1. The Docker Compose environment (nine containers) is stable and sufficient for MVP experiment workloads. No horizontal scaling, container replication, or distributed execution is required.
2. The CLI can maintain an open process for the duration of a `daedalus benchmark run` command without operating system or infrastructure intervention terminating it. For hardware experiments with multi-hour queue waits (SPEC-019), this assumption may not hold; SPEC-020 OQ-8 identifies experiment cancellation as a non-blocking open question.
3. Evidence Log tables (SPEC-006, SPEC-012) are populated by the Worker before the CLI triggers API evidence collection for a terminal-state job. The API reads Evidence Log records that the Worker has already written; no race between Worker write and API read is anticipated at MVP scale given job terminal state as the collection trigger.
4. SPEC-012 `jobs.problem_id` (FK → `routing_problems(problem_id)`) supports multiple rows with the same `problem_id` value. SPEC-012 FR-6 confirms no UNIQUE constraint exists on this column.
5. SPEC-008-R1 and SPEC-001 Assumptions amendment will be authored and accepted before the experiment harness is implemented. This ADR establishes the architectural authority for those amendments.
6. SPEC-012-R1 will define the experiment and benchmark table schema before the experiment harness is implemented. This ADR establishes the architectural authority for that amendment.

---

# Limitations

- This ADR does not define the experiment or benchmark table schema. Schema definition is deferred to SPEC-012-R1, which is blocked on this ADR's acceptance.
- This ADR does not define the API endpoint contracts for experiment submission, status retrieval, evidence collection trigger, or artifact retrieval. Those contracts are defined in SPEC-008-R1, which is blocked on this ADR's acceptance.
- This ADR does not resolve SPEC-020 OQ-8 (experiment cancellation). A mid-experiment CLI interruption leaves in-flight trials executing without a harness to collect evidence or submit remaining trials. Recovery behavior is an implementation concern deferred to SPEC-020 implementation planning.
- This ADR does not define the trial seed derivation algorithm. Seed derivation is governed by SPEC-020 FR-8 (ADR-010 compliant, uniqueness and determinism required) and is an implementation planning decision.
- The scope of the SPEC-001 Assumptions amendment is narrow: the one-problem-one-job cardinality assumption is relaxed for experiment context. SPEC-001's routing problem model (FR-1 through FR-16), solver contract (ADR-008), and Evidence Log integration (SPEC-006) are not changed.

---

# Documentation Updates

The following specification amendments are required before experiment harness implementation begins. This ADR establishes the architectural authority for each amendment. Each amendment requires its own governance cycle (Engineering Review → Architecture Review → Acceptance Review).

**SPEC-008-R1 (Required — Blocked by this ADR):**
- FR-7: Remove "each problem is associated with exactly one job" as a universal constraint. Replace with the scoped constraint: in standard job submissions, each routing problem is associated with exactly one job; in experiment context (ADR-012, SPEC-020 FR-4), a routing problem may be referenced by multiple jobs
- New functional requirements for: experiment manifest submission endpoint, benchmark manifest submission endpoint, experiment status retrieval endpoint, trial evidence collection trigger endpoint, trial results retrieval endpoint, experiment summary retrieval endpoint, benchmark summary retrieval endpoint
- Remove or scope any language reflecting the SPEC-016 OQ-3 "no API persistence of experiment records" position

**SPEC-001 Assumptions Amendment (Required — Blocked by this ADR):**
- Remove or scope the assumption: "A routing problem is created once and associated with exactly one job. No routing problem is shared between jobs."
- Replace with: in standard usage, each routing problem is associated with exactly one job; in experiment context (ADR-012, SPEC-020 FR-4), a routing problem may be referenced by multiple jobs
- SPEC-001's routing problem model (FR-1 through FR-16) is not changed

**SPEC-012-R1 (Required — Blocked by this ADR):**
- Define experiment persistence tables: experiments, benchmark manifests, trial records, artifact storage
- These tables are not Evidence Log tables (SPEC-006) and are not governed by SPEC-006 write-authority restrictions

**SPEC-016-R1 (Required — Blocked by this ADR):**
- Add `daedalus benchmark run <manifest.json>` command (or extend `daedalus experiment run` to accept the SPEC-020 manifest format)
- Document the long-running CLI process exception for benchmark and experiment commands
- Remove or scope SPEC-016 OQ-3 ("Local manifest only") — superseded by Decision 2
- Update SPEC-016 FR-12 Assumptions to reflect that in Generated Mode, backends within the same (problem_config_index, repetition_index) use the same `problem_id`, not independently persisted copies

**docs/architecture.md:**
- Add experiment and benchmark harness to the Major Components section
- Add experiment state persistence to the API responsibilities list

---

# Review Triggers

- SPEC-020 OQ-8 resolution (experiment cancellation): may require revision to the scope of Decision 1's long-running CLI exception or introduction of a server-side alternative for hardware experiments
- SPEC-020 OQ-5 resolution (parallel trial execution): may require reconsideration of Decision 6 (no additional deployment unit) if parallel execution requires a coordinator service
- ADR-007 review trigger satisfied (IBM Quantum Runtime integration begins): multi-hour hardware queue waits may make Assumption 2 (CLI process availability) untenable; server-side orchestration becomes a stronger candidate at that time

---

# Employer Signaling

- System Design
- Distributed Systems
- Reliability Engineering
- Scientific Computing

Designing the execution architecture for a multi-backend experiment harness requires reasoning simultaneously about component boundary preservation (Worker experiment-unawareness), cross-specification conflict resolution (cardinality constraints across five accepted specifications, evidence write authority), and the tradeoff between orchestration simplicity (CLI polling) and reliability (server-side durability). The instance-sharing invariant decision demonstrates the ability to identify a subtle correctness requirement (cross-solver quality comparability under SPEC-007 FR-7), trace it to its specification source, and resolve a multi-specification prose conflict in a schema-compatible way without requiring physical database changes.

---

# Decision Summary

**Decision 1:** The CLI process is the experiment orchestration executor. The orchestration loop runs in the CLI, with a scoped exception to SPEC-016's single-invocation assumption for `daedalus benchmark run` only.

**Decision 2:** The API is the durable state authority for all experiment state. SPEC-016 OQ-3 (local manifest only) is superseded. All experiment manifests, trial records, aggregate evidence, and artifacts are persisted through the API.

**Decision 3:** In experiment context, a routing problem may be referenced by multiple jobs. The SPEC-001 one-problem-one-job cardinality assumption and SPEC-008 FR-7 policy are relaxed in experiment context only. The SPEC-012 schema already supports this without constraint change.

**Decision 4:** The Worker is experiment-unaware. From the Worker's perspective, experiment-submitted jobs are indistinguishable from standard submissions. Worker responsibilities are not extended for experiment execution.

**Decision 5:** Evidence collection occurs through API-mediated collection. The CLI triggers the API to read Evidence Log records for completed trial jobs and write the collected evidence to experiment trial records. This satisfies SPEC-006 FR-1.3 (no additional writers to Evidence Log tables) and SPEC-016 Constraints (no direct CLI database access).

**Decision 6:** No additional deployment unit is introduced for MVP. The nine-container Docker Compose topology is unchanged.

**Primary Benefit:** A coherent, minimally-invasive experiment architecture that resolves five cross-specification conflicts without schema migration, new deployment units, or Worker boundary violations.

**Primary Cost:** Three specification amendments (SPEC-008-R1, SPEC-012-R1, SPEC-016-R1) and one assumptions amendment (SPEC-001) are required before implementation can begin. Each amendment requires its own governance cycle.

**Evidence Supporting the Decisions:** SPEC-020 FR-4 (instance-sharing invariant), SPEC-007 FR-7 (cross-solver comparability constraint), SPEC-016 scope table (CLI orchestration owner), SPEC-006 FR-1.3 (Evidence Log write authority), SPEC-012 FR-6 (no UNIQUE constraint on `jobs.problem_id`), SPEC-020 Domain Concept (Decisions 1 and 2 explicitly stated), SPEC-020 Non-Requirements (Decision 4 explicitly stated).

**Next Review Trigger:** SPEC-020 OQ-8 (experiment cancellation) resolution; SPEC-020 OQ-5 (parallel execution) resolution; ADR-007 review trigger satisfied (hardware integration begins).
