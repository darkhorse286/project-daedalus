# ADR-013: Experiment Backend Targeting

## Status

Accepted

**Date:** 2026-06-23

**Related Feature(s):** Benchmark and Experiment Harness (SPEC-020), API / Control Plane (SPEC-008), Daedalus CLI (SPEC-016)

**Related ADR(s):** ADR-008, ADR-009, ADR-012

---

# Context

SPEC-020 (Benchmark and Experiment Harness, Accepted) defines a multi-backend experiment model in which each trial targets a specific backend from the `solver_set`. ADR-012 Decision 1 establishes the CLI as the orchestration executor. ADR-012 Decision 4 establishes that the Worker remains experiment-unaware.

A structural gap exists in the current job submission contract. `POST /v1/jobs` (SPEC-008 FR-2) accepts `problem_id` and `scheduler_config_id` but no mechanism for directing execution to a specific backend. With a single experiment-wide `scheduler_config_id`, the Scheduler selects the backend by policy for every trial job. There is no current mechanism to guarantee that a trial job submitted for `backend_id = "qubo-simulated-annealing"` is processed by that backend rather than an alternative eligible backend.

This produces two critical failure modes:

**Per-backend attribution failure.** Experiment artifacts record evidence attributed to a specific backend. Without guaranteed backend targeting, the evidence may have been produced by a different backend. A benchmark that claims "QUBO SA produced lower total route distance than nearest-neighbor" is invalid if the QUBO SA trial ran on nearest-neighbor because the Scheduler selected it by policy. The project's POC||GTFO standard — evidence must be produced by the backend claimed — makes this a correctness failure, not a quality issue.

**Cross-solver comparison invalidity.** SPEC-007 FR-7 establishes that `hindsight_quality` is dimensionally comparable only within the same routing problem. ADR-012 Decision 3 established the instance-sharing invariant to satisfy this requirement. A parallel correctness requirement applies to backend attribution: `hindsight_quality` values are attributable to a specific solver only when the evidence was produced by that solver. Backend attribution uncertainty undermines cross-solver comparison validity regardless of problem instance sharing.

This gap is identified as a blocking implementation question in architecture.md (SPEC-016 OQ-6: Per-backend job targeting) and in SPEC-016 OQ-6. The candidate resolution approaches from SPEC-016 OQ-6 are:

- (a) Extend `POST /v1/jobs` with an optional `backend_id` field that directs the Scheduler to a specific backend.
- (b) Require each trial to use a per-backend `scheduler_config_id` (one config per backend in the `solver_set`).
- (c) Add backend-identity validation to SPEC-008 FR-25 (Trial Submission Linkage), rejecting the link if the job's actual backend does not match the trial's `backend_id`.
- (d) Accept that backend targeting is best-effort at MVP; document this as an accepted risk.

ADR-013 selects approach (a) and establishes the behavioral contract for backend targeting.

---

# Decision

`POST /v1/jobs` gains an optional `backend_id` field. Two execution paths are defined:

**Standard path (`backend_id` absent):** Normal job submission. The Scheduler evaluates all eligible backends and selects one by policy. This path is unchanged from the pre-ADR-013 model.

**Targeted path (`backend_id` present):** Targeted job submission. The Scheduler validates that the specified backend is eligible for the submitted routing problem, then directs execution to that backend. No policy-based substitution occurs: if the specified backend is ineligible, the job fails at the Scheduler stage. The Scheduler records the decision with `selection_mode = explicitly_targeted`, enabling evidence traceability from trial to backend.

**Behavioral contract:**

1. `backend_id` is an optional string field on `POST /v1/jobs`. When absent, the standard path is taken. When present, the targeted path is taken. The API validates `backend_id` as a non-empty string; no existence check against the Scheduler capability profile registry is performed at the API layer because capability profiles are not persisted in PostgreSQL (SPEC-012 FR-2 scope).

2. The experiment harness (CLI) supplies `backend_id` on every trial job submission. `backend_id` is set to the backend identifier from the trial record (`experiment_trials.backend_id`). This makes backend targeting the harness's submission contract, not a server-enforced requirement. The API does not distinguish experiment-context submissions from standard submissions.

3. When `backend_id` is present, the Scheduler validates the specified backend's eligibility for the routing problem using the same capability-checking rules applied in the standard path. Eligibility failure on a targeted backend is not recoverable by policy: the Scheduler does not fall back to another backend. The job transitions to a terminal state with a Scheduler rejection outcome.

4. The Scheduler persists a decision record for both paths. Decision records include a `selection_mode` field distinguishing `policy_selected` (standard path) from `explicitly_targeted` (targeted path). This field is required for evidence traceability: it allows post-hoc verification that each trial's evidence was produced by the claimed backend.

5. The Worker passes `backend_id` (when present in the job message) to the Scheduler for eligibility evaluation and backend dispatch. The Worker has no knowledge of experiments. A targeted job is a standard job that contains an additional routing field in the queue message. From the Worker's perspective, this is an additive field, not an experiment concept.

6. The `jobs` table does not receive an `experiment_id` column. Experiment-to-job linkage remains in `experiment_trials.job_id` (ADR-012 Decision 4). The `backend_id` field, when present, is stored in the job record and included in the queue message payload.

7. ADR-012 Decision 4 (Worker experiment-unawareness) is fully preserved. `backend_id` is a job-level routing directive, not an experiment reference.

---

# Alternatives Considered

## Alternative 1: Per-Backend Scheduler Configurations

### Description

Each backend in the `solver_set` receives a distinct `scheduler_config_id` configured to restrict Scheduler selection to that specific backend. The CLI constructs per-backend scheduler configurations before experiment submission and assigns each trial's job submission to the corresponding configuration. The experiment manifest carries a mapping from `backend_id` to `scheduler_config_id`, replacing or extending the single experiment-wide `scheduler_config_id`.

### Benefits

No changes to `POST /v1/jobs`. Backend targeting is expressed through existing scheduler configuration infrastructure. The Scheduler selects the backend by policy, and the configuration permits only one backend.

### Drawbacks

Requires creating N distinct scheduler configurations per experiment, where N = `|solver_set|`. Configuration proliferation: each experiment repetition with a different solver set requires new configurations, since configurations are immutable after creation (SPEC-008 FR-10, OQ-4 recommendation). More critically, a scheduler configuration is intended to express optimization policy — objective mode, weights, thresholds — not backend identity. Encoding backend identity in configuration conflates two distinct concerns. Evidence citing `scheduler_config_id` would no longer describe the optimization objective alone; it would describe a (objective, backend-restriction) pairing embedded in configuration, creating hidden experiment semantics. Backend eligibility restriction via configuration is not a supported semantic in SPEC-003 FR-6; implementing it would require new configuration fields or objective mode extensions not currently in scope.

### Reason Not Selected

Per-backend scheduler configurations introduce hidden experiment semantics into a component whose purpose is optimization policy. The Scheduler configuration model exists to express how backends are evaluated and selected, not which backend must be selected. Configuration proliferation — N configurations per experiment — adds governance and operational overhead with no benefit to the objective-setting purpose of configurations. Backend targeting requires an explicit routing directive on the submission, not a policy workaround.

---

## Alternative 2: Experiment-Only `forced_backend_id` Field

### Description

`POST /v1/jobs` gains a distinct `forced_backend_id` field (semantically separate from a general-purpose `backend_id`) that is valid only in experiment context. The API conditionally enforces its presence or absence based on detected experiment context.

### Benefits

Explicit field naming distinguishes experiment-context targeting from other potential backend specification use cases. The word "forced" communicates the no-fallback intent directly.

### Drawbacks

The Worker is experiment-unaware (ADR-012 Decision 4). Validating `forced_backend_id` against experiment context at the API layer requires the API to know whether a submission is in experiment context — knowledge that should belong only to the CLI and the experiment trial records. A conditional field that is valid in some submission contexts and invalid in others creates an asymmetric API contract that is difficult to document and test. The distinction between `forced_backend_id` and a general-purpose `backend_id` is artificial: both direct the Scheduler to a specific backend. A general-purpose optional `backend_id` is cleaner and more production-grade. The "forced" framing also incorrectly implies that eligibility validation is bypassed, which is not the case: capability validation remains required regardless of selection mode.

### Reason Not Selected

Experiment-only fields introduce hidden experiment semantics at the API boundary. Whether a submission is in experiment context is a concern of the CLI and the experiment trial records, not the API contract. A general-purpose optional `backend_id` is explicit, production-grade, and usable beyond experiment context without requiring the API to detect submission context. The API should not know or care why a `backend_id` was specified.

---

## Alternative 3: Post-Execution Backend Inference (Scheduler-Chooses, Verify After)

### Description

Trial job submissions include no `backend_id`. After each trial job reaches a terminal state, the CLI or API reads the Scheduler decision record to determine which backend was actually selected. SPEC-008 FR-25 (Trial Submission Linkage) is extended with backend-identity validation: the trial linkage is rejected if the job's Scheduler-selected backend does not match the trial's `backend_id`.

### Benefits

No changes to `POST /v1/jobs`. Trial submission remains unchanged. Backend attribution is verified post-hoc from the Scheduler decision record, using an evidence record the Worker already produces.

### Drawbacks

Post-hoc verification without pre-execution targeting does not guarantee correct attribution — it only detects misattribution after resources have been consumed. In a multi-backend deployment, the Scheduler may select a different backend than intended, producing evidence attributed to the wrong backend. FR-25 backend-identity rejection on mismatch leaves the trial in an unlinked state: the job ran, produced evidence, and reached terminal status, but the trial linkage is rejected because the backend was wrong. The experiment must re-submit the trial against the correct backend, wasting execution resources and creating an orphaned job with unreferenced evidence. Worse, in a deployment where Scheduler policy consistently selects the same backend (e.g., a deployment with only one eligible backend per problem class), this alternative appears to work correctly in testing but fails silently in production when backend availability changes. The POC||GTFO standard requires pre-execution targeting, not post-execution detection.

### Reason Not Selected

Post-hoc detection is insufficient for a system whose correctness requirement is evidence traceability. The project's evidentiary standard requires that evidence be produced by the claimed backend before it is attributed. Re-executing misattributed trials wastes resources and creates orphaned evidence records without resolving the architectural gap. Backend targeting must be established before execution, not verified after.

---

## Alternative 4: Add `experiment_id` to Jobs

### Description

The `jobs` table gains an `experiment_id` column, populated by the API when a job is submitted in experiment context. The Worker reads `experiment_id` from the job record and looks up the associated trial record to determine the target `backend_id`. The Worker uses the retrieved `backend_id` to constrain Scheduler selection.

### Benefits

Backend targeting information is derivable from experiment state co-located with the job record. No new field is required on the job submission endpoint.

### Drawbacks

This approach directly violates ADR-012 Decision 4 (Worker experiment-unawareness). The Worker would need to read `experiment_trials` to determine `backend_id` from `experiment_id` + job lookup, making the Worker aware of experiments, trial records, and experiment state — the exact coupling Decision 4 prohibits. Additionally, `experiment_id` on the `jobs` table requires the API to associate each job with an experiment trial at submission time, which inverts the current two-step architecture (job created first, then linked to trial via FR-25). Populating `experiment_id` at job creation time would require the CLI to provide trial linkage during job submission rather than after it. ADR-012 Decision 4 explicitly prohibits this: "SPEC-012-R1 must not add an `experiment_id` column to the `jobs` table; doing so would require the Worker to populate it, violating this decision."

### Reason Not Selected

This alternative directly violates ADR-012 Decision 4. Worker experiment-unawareness is a foundational architectural decision that preserves the Worker's clean per-job boundary and decouples execution from experiment coordination. No alternative that requires the Worker to access experiment or trial records is acceptable within the current architecture. ADR-012 Decision 4 is not a candidate for relaxation to solve the backend targeting problem.

---

## Alternative 5: Dedicated Experiment Execution Service

### Description

A new container — an experiment executor service — manages backend-targeted job submission. The CLI submits the experiment manifest to the executor. The executor submits backend-targeted trial jobs, monitors execution, and collects evidence. The CLI delegates orchestration to the executor rather than managing the trial loop directly.

### Benefits

Backend targeting and experiment orchestration are isolated in a dedicated service. The service can maintain state across CLI exits, enabling experiment resumption without a long-lived CLI process.

### Drawbacks

Violates ADR-012 Decision 6 (no additional deployment unit for MVP). A new service introduces container lifecycle management, inter-service coordination, new API surface between CLI and executor, and new operational failure modes (executor crash mid-experiment, state recovery, executor-to-API authentication). The executor's only novel capability over the current architecture is backend-targeted job submission — a function the CLI can perform directly with a minimal change to the job submission endpoint.

### Reason Not Selected

The backend targeting problem does not require a new deployment unit. An optional `backend_id` field on `POST /v1/jobs` — consumed by the existing CLI orchestrator — achieves the same targeting capability without new infrastructure. ADR-012 Decision 6 explicitly prohibits new deployment units at MVP scope. The per-unit infrastructure cost is disproportionate to the incremental capability gained.

---

# Consequences

## Positive

- Multi-backend experiments produce evidence with verifiable backend attribution. The POC||GTFO evidentiary standard is satisfiable: the trial's evidence is produced by the claimed backend, not by whichever backend the Scheduler happened to select.
- Cross-solver quality comparisons (SPEC-007 FR-7) are valid at the attribution level when combined with ADR-012 Decision 3 (instance-sharing invariant): the same routing problem is executed by the correct backend for each trial.
- Scheduler decision records with `selection_mode` support evidence traceability for both execution paths: `policy_selected` records confirm policy-driven decisions; `explicitly_targeted` records confirm direction-driven decisions. Both are auditable from evidence artifacts.
- Normal job submission behavior is unchanged. The optional `backend_id` field is additive; callers who do not supply it receive standard-path behavior with no change.
- No new deployment unit. ADR-012 Decisions 4 (Worker experiment-unawareness) and 6 (no additional deployment unit) remain intact.

## Negative

- SPEC-008 must be amended to add `backend_id` to `POST /v1/jobs` (FR-2, FR-3, FR-4, FR-5) and to the queue message payload.
- SPEC-016 must be amended to supply `backend_id` in experiment trial job submissions (FR-18 orchestration loop).
- SPEC-003 must be amended to define `selection_mode` in the Scheduler decision record and to specify the no-fallback eligibility rule for `explicitly_targeted` decisions.
- SPEC-012 must be amended to add nullable `backend_id` to the `jobs` table and `selection_mode` to the `decision_records` table.
- The Scheduler's no-fallback behavior on targeted jobs diverges from standard-path behavior. This divergence must be documented clearly in SPEC-003 and SPEC-005 to prevent implementors from accidentally introducing fallback behavior in the targeted path.

## Accepted Risks

- API-level validation of `backend_id` is limited to non-empty string format checking. The API cannot verify whether the specified `backend_id` is registered in the Scheduler's capability profile registry because capability profiles are not persisted in PostgreSQL (SPEC-012 FR-2 scope). An unregistered `backend_id` is only detected at Scheduler execution time in the Worker, causing the trial's job to transition to a Scheduler rejection terminal state. This is consistent with the existing capability-profile validation gap acknowledged in SPEC-008 FR-18 and with the general principle that Scheduler eligibility is a runtime concern, not a submission-time concern.
- A targeted job whose `backend_id` is ineligible for the submitted routing problem produces a `SchedulerRejected` outcome without fallback. This is a correctness invariant, not a failure mode: if the designated backend is ineligible, the trial must not produce evidence attributed to a different backend. The absence of fallback is load-bearing for benchmark validity.

---

# Architectural Impact

| Component | Impact | Nature |
|---|---|---|
| API Layer (ASP.NET Core, SPEC-008) | Yes | Add optional `backend_id` to `POST /v1/jobs` FR-2, FR-3, FR-4, FR-5; add `backend_id` to queue message payload; add `backend_id` to job record persistence |
| Persistence (PostgreSQL, SPEC-012) | Yes | Add nullable `backend_id` column to `jobs` table; add `selection_mode` column to `decision_records` table |
| Daedalus Scheduler (SPEC-003) | Yes | Add `selection_mode` (`policy_selected`, `explicitly_targeted`) to decision records; implement no-fallback eligibility path for targeted jobs; apply capability validation to specified backend |
| Daedalus Worker (SPEC-005) | Yes (minimal) | Forward `backend_id` from job message to Scheduler when present; no experiment awareness is introduced |
| Daedalus CLI (SPEC-016) | Yes | Supply `backend_id` in every experiment trial job submission (FR-18 orchestration loop); retrieve `backend_id` per trial from GET /v1/experiments/{id}/trials response |
| Daedalus Core (C++ domain, architecture.md) | No | Capability checks are unchanged; the selection constraint is applied by the Scheduler, not Core |
| Evidence Log (SPEC-006) | No | Evidence Log write authority and schema are unchanged |
| Evidence Reports (SPEC-009) | Yes (minor) | Scheduler decision section of the evidence report should reflect `selection_mode`; the field is read from the decision record and requires no new Worker logic |
| Python Solver Adapter (SPEC-017) | No | The adapter is backend-specific; backend dispatch is a Scheduler and Worker concern |
| Docker Compose Topology | No | No new containers; topology is unchanged |
| Experiment Harness (SPEC-020) | No | The harness model is unchanged; `backend_id` is a per-trial job submission field supplied by the CLI per ADR-012 Decision 1 |
| Observability | Yes (minor) | `job.submit` span attributes should include `backend_id` when present; `scheduler.score_solvers` span attributes should include `selection_mode`; these additions follow existing ADR-006 span requirements |

---

# Assumptions

1. The Scheduler's capability profile registry is accessible at job execution time within the Worker process. Backend eligibility validation for `explicitly_targeted` jobs uses the same registry and eligibility rules applied in the standard path.
2. `backend_id` values in experiment trial records are populated by the API at experiment manifest submission time (SPEC-008 FR-18, SPEC-012 FR-21). The CLI retrieves these values from `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22) before beginning the trial submission loop (SPEC-016 FR-18).
3. A nullable `backend_id` column on the `jobs` table is backward-compatible. Existing job records receive NULL (standard-path job); the column addition requires no data migration.
4. A targeted job that fails backend eligibility validation produces the same `SchedulerRejected` observable outcome as a policy-rejected trial in the experiment harness. No new trial status or solver outcome is required to represent targeted eligibility rejection.
5. The `selection_mode` field in decision records is sufficient to distinguish targeted from policy-selected executions for all evidence traceability purposes at MVP scope. No additional per-decision metadata is required.

---

# Limitations

- This ADR does not define the exact `backend_id` format validation rules beyond non-empty string. Whether the API validates `backend_id` against an enumeration of known backend identifiers is an implementation planning concern for the SPEC-008 amendment.
- This ADR does not define the `selection_mode` values beyond `policy_selected` and `explicitly_targeted`. Additional values are not needed at MVP scope.
- This ADR does not resolve SPEC-008 OQ-7 (Generated Mode routing problem creation). When Generated Mode is implemented, the targeted path applies to Generated Mode trial jobs with the same behavioral contract; no re-evaluation of this decision is required.
- This ADR does not resolve SPEC-020 OQ-8 (experiment cancellation). Backend targeting does not affect the cancellation model.
- Automatic retry behavior for `SchedulerRejected` targeted trials is not defined by this ADR. When a targeted trial is rejected due to backend ineligibility, the trial is terminal in `SchedulerRejected` state. Retry logic, if any, is an experiment harness implementation concern governed by SPEC-020 FR-10 and SPEC-016 FR-18 error handling.
- This ADR does not add a new public-facing error response code for Scheduler-stage backend eligibility rejection at the API submission endpoint. The rejection is observable through job status polling, not through the submission response.

---

# Documentation Updates Required

The following amendments are required as a consequence of this decision. Each amendment requires its own governance cycle (Engineering Review → Architecture Review → Acceptance Review).

**SPEC-008 (Required — Blocked by this ADR):**
- FR-2 (Job Submission Request Model): Add optional `backend_id` string field to the `POST /v1/jobs` request body. Document both submission paths: standard (`backend_id` absent) and targeted (`backend_id` present). Document that `backend_id` is optional for all callers and that the experiment harness always provides it for trial submissions.
- FR-3 (Submission Request Validation): Add validation rule for `backend_id` when present: must be a non-empty string. No existence check against the capability profile registry is performed (registry not in PostgreSQL). Absence of `backend_id` is not a validation failure on any submission path.
- FR-4 (Job Creation and Persistence): Store `backend_id` (nullable) in the job record at persistence time when present in the request.
- FR-5 (Queue Publication): Include `backend_id` (when present) in the queue message payload alongside `job_id`, `problem_id`, and `scheduler_config_id`. Absence of `backend_id` in the message means standard-path execution.
- FR-7 (Job Identifier Model): Note that `backend_id`, when present, is a routing directive stored in the job record and queue message; it is not an experiment reference and does not link the job to any experiment or trial.
- FR-17 (Observability): Add `backend_id` to `job.submit` span attributes (when present); add `selection_mode` to `scheduler.score_solvers` span attributes.

**SPEC-016 (Required — Blocked by this ADR):**
- FR-18 (Experiment Trial Orchestration Loop): Amend per-trial job submission to include the trial's `backend_id` in the `POST /v1/jobs` request body. The `backend_id` for each trial is read from the trial record retrieved via `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22). The `backend_id` is set to `experiment_trials.backend_id`, which is populated by the API at experiment creation time from the manifest's `solver_set`.
- OQ-6: Mark as resolved. Resolution: optional `backend_id` on `POST /v1/jobs`, supplied by the CLI for every experiment trial job submission. ADR-013 is the authoritative resolution.

**SPEC-003 (Required — Blocked by this ADR):**
- Add `selection_mode` to the Scheduler decision record. Valid values: `policy_selected` (standard path, Scheduler selected by objective function and eligibility), `explicitly_targeted` (targeted path, caller specified the backend). The `selection_mode` field must be persisted in the decision record and must appear in all decision record outputs consumed by the evidence report (SPEC-009) and the Evidence Log (SPEC-006).
- Add the no-fallback rule for `explicitly_targeted` decisions: if the specified backend fails the eligibility check, the Scheduler records a decision with `selection_mode = explicitly_targeted` and a rejection outcome. No alternative backend is evaluated or selected. The job transitions to the Scheduler rejection terminal state. This rule is load-bearing for benchmark validity and must not be treated as optional.

**SPEC-012 (Required — Blocked by this ADR):**
- Add nullable `backend_id` column to the `jobs` table. NULL represents standard-path submissions. The column is populated by the API at job creation time when `backend_id` is present in the submission request.
- Add `selection_mode` column to the `decision_records` table. Values: `policy_selected`, `explicitly_targeted`. The column is populated by the Worker's Scheduler at decision record persistence time (SPEC-005 FR-16, SPEC-006 FR-6).
- Update FR-15 (Read/Write Ownership Map) to reflect that `jobs.backend_id` is written by the API at job creation time (extending the existing API write ownership on `jobs`).

**docs/architecture.md (Required):**
- Remove "SPEC-016 OQ-6: Per-backend job targeting" from the Implementation Blockers section under Open Architecture Questions. This question is resolved by ADR-013.
- Update the Experiment Execution Flow sequence diagram: the `POST /v1/jobs` call in the Submit phase should reflect `(problem_id, backend_id)` to show that the experiment harness supplies a backend targeting directive per trial.

---

# Review Triggers

- SPEC-008 OQ-7 resolution (Generated Mode implementation begins): verify that `backend_id` targeting applies correctly to Generated Mode trial submissions with the same behavioral contract; the mechanism is identical, only the submission source (generated vs. pre-existing `problem_id`) differs.
- ADR-007 review trigger satisfied (IBM Quantum Runtime integration begins): verify that `backend_id = "qaoa-hardware"` targeting satisfies the hardware evidence traceability requirements for hardware experiments at the time of hardware integration; specifically, confirm that the Scheduler's `explicitly_targeted` no-fallback rule is compatible with hardware provider availability constraints.
- SPEC-003 revision (new eligibility rules or objective mode additions): assess whether `selection_mode` semantics remain accurate under revised eligibility evaluation; the no-fallback rule for targeted jobs must be re-evaluated if eligibility criteria become context-dependent.

---

# Employer Signaling

- System Design
- Distributed Systems
- API Contract Design
- Reliability Engineering
- Scientific Computing

Designing backend targeting for a multi-backend experiment harness requires identifying the exact correctness invariant that motivates the design (POC||GTFO: evidence must be produced by the claimed backend), tracing that invariant to its specification sources (SPEC-007 FR-7 cross-solver comparability, SPEC-020 FR-5 solver set attribution), and selecting the minimal mechanism that satisfies it without violating existing architectural boundaries (ADR-012 Decision 4: Worker experiment-unawareness; ADR-012 Decision 6: no new deployment unit; ADR-008: backend neutrality). Five alternatives were evaluated and rejected. The selected mechanism — an optional job-level routing field — is additive, explicit, observable, and production-grade. It requires no new component awareness of experiments, no configuration proliferation, and no post-execution attribution inference. The decision to reject experiment-only field naming in favor of a general-purpose `backend_id` reflects understanding that API contracts should express what a caller wants (execute on backend X) rather than why the caller wants it (because this is an experiment trial).

---

# Decision Summary

**Decision:** `POST /v1/jobs` gains an optional `backend_id` field. When absent, the Scheduler selects by policy (standard path, behavior unchanged). When present, the Scheduler validates eligibility for the specified backend and directs execution to it without fallback (targeted path). The Worker forwards `backend_id` to the Scheduler but has no knowledge of experiments. Scheduler decision records include `selection_mode` distinguishing `policy_selected` from `explicitly_targeted`. The `jobs` table does not receive `experiment_id`. Experiment-to-job linkage remains in `experiment_trials.job_id`. ADR-012 Decision 4 is fully preserved.

**Primary Benefit:** Multi-backend experiments produce evidence with verifiable backend attribution, satisfying the POC||GTFO evidentiary standard and making cross-solver comparisons valid under SPEC-007 FR-7.

**Primary Cost:** SPEC-008, SPEC-003, SPEC-012, and SPEC-016 amendments are required before the multi-backend experiment harness can be implemented. The Scheduler must implement a no-fallback eligibility path for targeted jobs.

**Open Questions Resolved:** architecture.md Implementation Blocker "SPEC-016 OQ-6: Per-backend job targeting"; SPEC-016 OQ-6 (Per-Backend Job Targeting Mechanism).

**Next Review Trigger:** SPEC-008 OQ-7 (Generated Mode implementation); ADR-007 review trigger (hardware integration begins).
