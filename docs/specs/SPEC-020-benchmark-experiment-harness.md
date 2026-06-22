# Feature Specification

## Metadata

**Feature ID:** SPEC-020

**Title:** Benchmark and Experiment Harness

**Status:** Proposed

**Author:** Darkhorse286

**Created:** 2026-06-21

**Last Updated:** 2026-06-22

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-001, ADR-004, ADR-006, ADR-008, ADR-010

**Related Specs:** SPEC-001, SPEC-002, SPEC-003, SPEC-004, SPEC-005, SPEC-006, SPEC-007, SPEC-011, SPEC-013, SPEC-014, SPEC-015, SPEC-017, SPEC-018, SPEC-019

---

# Problem Statement

Project DAEDALUS produces individual evidence records for each solver execution through its job pipeline. These records — quality metrics, regret analysis, scheduler decisions — are well-defined by SPEC-006 and SPEC-007. However, no mechanism exists to systematically execute comparison experiments across solver populations and routing problem distributions, aggregate results across repeated trials, compute summary statistics, and produce experiment artifacts that support the project's core thesis and technical writing objectives.

Without an experiment harness:

- Solver comparison requires manually submitting individual jobs and aggregating results by hand. There is no shared vocabulary for grouping related comparisons under a research question.
- Reproducibility of multi-solver comparisons depends on manual bookkeeping rather than a defined protocol. An engineer cannot replay a comparison from a stored configuration.
- Statistical claims ("Solver A produces lower total route distance than Solver B across a workload distribution") have no systematic evidence basis. Individual job executions prove individual outcomes; they do not produce distribution-level evidence.
- The `quantum_hardware` backend (SPEC-019) cannot be fairly evaluated against classical baselines without a mechanism to account for queue variance, hardware noise, and repeated trials.
- Technical writing that references solver comparison results has no defined artifact chain from experiment configuration to reported conclusion.

SPEC-020 defines the experiment and benchmarking framework that addresses these gaps. The harness orchestrates trials over the existing job execution pipeline, collects per-trial evidence from the Evidence Log, computes summary statistics, and generates comparison artifacts. It does not optimize routes, implement solvers, or define scheduler policy.

---

# Business Value

- Produces the systematic comparative evidence required to demonstrate the project thesis: that most routing problems should not use expensive quantum-adjacent backends, because classical heuristics are sufficient for the evaluated workload distributions
- Enables reproducible experiments whose results can be cited in technical writing and presented as portfolio artifacts with a traceable chain from configuration to conclusion
- Provides the statistical foundation for comparing solver backends fairly: across controlled problem distributions, repeated trials, and documented variance
- Validates the end-to-end system by exercising the full pipeline — experiment manifest, job submission, solver execution, quality evaluation, evidence persistence, and artifact generation — as an integrated workflow
- Demonstrates production-grade experiment engineering: reproducibility controls, hardware failure handling, evidence aggregation, and statistical rigor consistent with the project's benchmarking standard

---

# Employer Signaling

- System Design
- Performance Engineering
- Reliability Engineering
- Scientific Computing
- Observability

Designing an experiment harness for a multi-backend optimization system requires reasoning about reproducibility under different stochastic models (deterministic classical, reproducible stochastic, non-reproducible hardware), statistical validity of aggregated outcomes across heterogeneous backend behaviors, and evidence traceability from raw trial outputs to summary conclusions. The harness must handle all three backend categories within a unified trial model while preserving semantic distinctions between them in all generated artifacts.

---

# Domain Concept

An **experiment** is a planned, reproducible comparison of one or more solver backends across one or more routing problems with a defined repetition count. The experiment is fully described by its manifest: the workload set, solver set, repetition count, seed, and execution configuration. Every experiment produces trial-level evidence linked to the Evidence Log, aggregate statistics per solver and per problem, and a summary artifact.

A **benchmark** is a named research initiative that groups one or more experiments under a shared identifier and research question. A benchmark manifest records the research question, hypothesis, null hypothesis, and the set of participating experiments. Benchmark summaries aggregate evidence across experiments to support a conclusion.

A **trial** is a single execution of one solver against one routing problem instance. Each trial maps to exactly one job submitted to the existing execution pipeline. The experiment defines the set of trials through the cartesian product of its workload set, solver set, and repetition count.

A **repetition** is one execution of a specific (problem configuration, solver) pairing. The repetition count governs how many times each pairing is executed. Multiple repetitions of a stochastic solver on distinct problem instances provide the sample distribution from which quality and runtime statistics are drawn.

The experiment harness does not execute optimization. It does not select solvers. It does not define scheduler policy. It does not generate routing problems directly. It orchestrates existing system components — job submission, the worker execution pipeline, the Evidence Log — to produce structured comparative evidence.

The harness supports all three trial cardinality combinations:
- One problem, many solvers: workload set with one problem, solver set with multiple backends
- Many problems, one solver: workload set with multiple problems, solver set with one backend
- Many problems, many solvers: workload set with multiple problems, solver set with multiple backends (the canonical benchmarking configuration)

**Harness execution model:** The harness orchestration loop runs in the CLI process. SPEC-016 owns "Experiment orchestration (multi-job submission and result collection)" and is the designated execution owner. The CLI submits the experiment manifest to the API, iterates through the trial set in defined submission order, submits each trial as a job via the SPEC-008 job submission endpoint, and polls the API job status endpoints for trial completion. When a trial job reaches a terminal state, the CLI instructs the API to read the associated Evidence Log records and persist collected evidence in the trial record. The API persists all experiment state: the manifest, trial records, collected evidence, computed statistics, and artifact payloads. All durable state is API-backed; the CLI process is the orchestration executor but not the state authority.

---

# Scope and Responsibility Boundary

| Responsibility | Owner |
|---|---|
| Routing problem model, validation, and persistence | SPEC-001 |
| Synthetic routing problem generation | SPEC-002 |
| Scheduler policy and solver eligibility | SPEC-003 |
| Solver contract (SolverRequest, SolverResponse, SolverOutcome) | SPEC-004 |
| Worker execution lifecycle | SPEC-005 |
| Evidence Log persistence semantics | SPEC-006 |
| Quality evaluation metrics and route simulation | SPEC-007 |
| Individual evidence report generation per job | SPEC-009 |
| Backend solver framework and capability profiles | SPEC-011 |
| Benchmark definitions, research questions, and hypotheses | **SPEC-020 (this spec)** |
| Experiment definitions and manifests | **SPEC-020 (this spec)** |
| Experiment execution and trial orchestration | **SPEC-020 (this spec)** |
| Repeated trial management and seed strategy | **SPEC-020 (this spec)** |
| Solver comparison across workload distributions | **SPEC-020 (this spec)** |
| Evidence collection from the Evidence Log per trial | **SPEC-020 (this spec)** |
| Result aggregation and statistical summaries | **SPEC-020 (this spec)** |
| Artifact generation (manifest, trial results, summaries) | **SPEC-020 (this spec)** |
| Reproducibility classification for experiments and trials | **SPEC-020 (this spec)** |
| Hardware failure policy at the experiment level | **SPEC-020 (this spec)** |
| Web UI presentation of experiment results and artifacts | SPEC-021 (pending specification) |
| Dashboard visualization of benchmark and experiment summaries | SPEC-022 (pending specification) |

The harness does not own:
- Individual solver behavior: how a solver solves a problem
- Individual job evidence records: owned and written by the Evidence Log (SPEC-006)
- Quality evaluation computation: owned by Core, defined in SPEC-007
- Report rendering or visualization: owned by SPEC-009, SPEC-021, SPEC-022

---

# Requirements

---

### FR-1: Experiment Identity

**Description:**
Each experiment has a unique system-generated identifier and a set of identity fields established at submission time.

**Experiment identity fields:**

| Field | Type | Source | Description |
|---|---|---|---|
| `experiment_id` | UUID | System-generated | Unique identifier for this experiment |
| `benchmark_id` | string | Required | User-supplied grouping identifier. Associates this experiment with a benchmark. Multiple experiments may share the same `benchmark_id`. |
| `experiment_name` | string | Required | Human-readable name. Does not affect execution. |
| `created_at` | timestamp | System-generated | Creation timestamp |
| `status` | ExperimentStatus | System-maintained | Lifecycle state. Defined in FR-11. |

**Acceptance Criteria:**
- `experiment_id` is a UUID generated before the experiment manifest is persisted
- `benchmark_id` is a non-empty string
- `experiment_name` is a non-empty string
- Two experiments with different `experiment_id` values may share the same `benchmark_id`
- The same `experiment_id` cannot appear in two distinct experiment records

---

### FR-2: Benchmark Definition and Manifest

**Description:**
A benchmark is a named research initiative identified by `benchmark_id`. A benchmark manifest is a separately submitted document that records the benchmark's research context. Experiments are associated with a benchmark at experiment submission time by supplying the `benchmark_id` field.

**Benchmark manifest fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `benchmark_id` | string | Yes | Identifier shared with all member experiments |
| `benchmark_name` | string | Yes | Human-readable benchmark name |
| `research_question` | string | Yes | The question the benchmark attempts to answer. Example: "Does QUBO simulated annealing produce lower total route distance than nearest-neighbor on small routing problems?" |
| `hypothesis` | string | Yes | What is expected to happen |
| `null_hypothesis` | string | Yes | What would be true if the proposed improvement does not exist |
| `controls` | list[string] | Yes | What is held constant across experiments. Examples: problem size class, time window presence, fleet configuration |
| `independent_variables` | list[string] | Yes | What changes between experiments. Example: backend_id, repetition_count |
| `dependent_variables` | list[string] | Yes | What is measured. Example: hindsight_quality, execution_duration_ms |
| `created_at` | timestamp | System-generated | |

The benchmark manifest is immutable after submission.

A benchmark summary is produced from all experiments associated with the `benchmark_id`. The benchmark summary is updated each time a member experiment reaches terminal state. Benchmark summary contents are defined in FR-14 (Artifact 4).

**Acceptance Criteria:**
- A benchmark manifest is submitted before or at the time of the first experiment submission under that `benchmark_id`
- A benchmark manifest is identifiable by `benchmark_id` and retrievable after submission
- A benchmark summary references all `experiment_id` values that share its `benchmark_id`
- The benchmark manifest is not required to enumerate expected `experiment_id` values at submission time; experiments join the benchmark by declaring the `benchmark_id`
- A `benchmark_id` that has no submitted benchmark manifest but has associated experiments is valid; in this case, the benchmark summary contains only the experiment-level evidence without research context fields

---

### FR-3: Experiment Manifest

**Description:**
The experiment manifest is the authoritative, immutable record of an experiment's configuration. It is persisted when the experiment is submitted and is not modified after submission.

**Experiment manifest fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `experiment_id` | UUID | System-generated | Experiment identifier (FR-1) |
| `benchmark_id` | string | Yes | Benchmark grouping identifier (FR-2) |
| `experiment_name` | string | Yes | Human-readable name |
| `experiment_seed` | uint64 | Yes | Top-level entropy source for all trial seed derivation in this experiment (FR-8) |
| `workload_set` | WorkloadSetDefinition | Yes | Defines the routing problems participating (FR-4) |
| `workload_mode` | "fixed" \| "generated" | System-derived | Derived from the WorkloadSetDefinition type |
| `solver_set` | list[backend_id] | Yes | Ordered list of backend identifiers participating (FR-5). Minimum 1. |
| `repetition_count` | uint32 | Yes | Number of times each (problem_config, solver) pairing is executed. Minimum 1. |
| `planned_trial_count` | uint32 | System-derived | Total trial executions: `problem_count × |solver_set| × repetition_count`. Computed at manifest creation. |
| `execution_timeout_ms_per_trial` | uint32 | Yes | Execution timeout applied to every trial's job submission (SPEC-004 FR-2). Must be positive. |
| `scheduler_config_id` | UUID | Optional | Scheduler configuration applied to every trial. If absent, the default scheduler configuration is used (SPEC-003). |
| `hardware_policy` | HardwarePolicy | Optional | Retry and failure handling for `quantum_hardware` backend trials (FR-10). If absent, default hardware policy applies. |
| `reproducibility_class` | "fully_reproducible" \| "partially_reproducible" \| "non_reproducible" | System-derived | Derived from the union of backend determinism classes in the solver set (FR-9). |
| `created_at` | timestamp | System-generated | |

The experiment manifest is immutable after creation. No field may be modified after the experiment is submitted. The `planned_trial_count` field represents the originally intended number of trials at manifest creation time. In cases of partial workload generation failure (see Failure Modes), the actual number of trials submitted may be less than `planned_trial_count`; the experiment's runtime state tracks this as an `effective_trial_count` distinct from the immutable manifest field.

**Acceptance Criteria:**
- The experiment manifest is persisted before any trials are submitted
- `planned_trial_count = problem_count × |solver_set| × repetition_count`, where `problem_count` is the number of problem instances in the resolved workload set
- The experiment manifest is retrievable by `experiment_id` at any point after submission
- A submitted experiment manifest cannot be modified

---

### FR-4: Workload Set Model

**Description:**
The workload set defines the routing problems used in the experiment. The harness supports two workload modes.

**Mode A — Fixed Workload:**
The workload set is a list of pre-existing routing problem identifiers. Each `problem_id` must reference a routing problem already persisted in PostgreSQL (SPEC-001, SPEC-012).

| Field | Type | Required | Description |
|---|---|---|---|
| `mode` | "fixed" | Yes | |
| `problem_ids` | list[UUID] | Yes | One or more existing problem identifiers. Minimum 1 element. |

In fixed workload mode, all repetitions of a given (problem, solver) pair use the same routing problem instance. The problem seed is fixed and was established at problem submission time.

**Mode B — Generated Workload:**
The workload set is a SPEC-002 generation configuration. The harness submits routing problem generation requests before trial execution begins.

| Field | Type | Required | Description |
|---|---|---|---|
| `mode` | "generated" | Yes | |
| `generation_config` | GenerationConfig | Yes | SPEC-002 generation configuration: scenario type, stop count range, fleet parameters, time window parameters, and all parameters required by SPEC-002 FR-1 through FR-5 |
| `instance_count` | uint32 | Yes | Number of distinct routing problem instances to generate per repetition. Minimum 1. |

In generated workload mode, the harness generates `instance_count × repetition_count` routing problem instances total before trial execution begins. Instance seeds are derived per (problem_config_index, repetition_index) pair from the experiment_seed (FR-8). Different repetitions of the same problem configuration use different seeds, producing different routing problem instances with the same structural parameters.

For each (problem_config_index, repetition_index) combination, the harness generates exactly one routing problem instance and assigns it a single `problem_id`. All backends in the solver set for that (problem_config_index, repetition_index) submit trials against the same `problem_id`. This instance-sharing invariant is required for cross-solver quality comparisons to be valid within-problem comparisons (SPEC-007 FR-7): within each repetition index, all backends evaluated the same problem geometry.

Trials are submitted in (problem_config_index ascending, repetition_index ascending, solver_set_index ascending) order. For a given (problem_config_index, repetition_index), all backends are submitted before advancing to the next repetition index. In Generated Mode, this ensures that each problem instance is fully submitted to all backends before the next instance is generated. Trial submission order is an observable harness behavior and must be consistent across experiment runs.

**Problem count by mode:**

| Mode | problem_count in planned_trial_count |
|---|---|
| Fixed | `\|problem_ids\|` |
| Generated | `instance_count` |

In generated mode, `repetition_count` repetitions each generate fresh instances. The harness generates `instance_count × repetition_count` total instances. Each repetition uses `instance_count` distinct problem instances.

**Acceptance Criteria:**
- In fixed workload mode: all `problem_ids` must be validated against PostgreSQL before trials begin; an unresolvable `problem_id` is an experiment configuration error that transitions the experiment to `Failed`
- In generated workload mode: the harness generates all problem instances before trial submission begins; generation failure for any instance is an experiment configuration error
- The workload mode is recorded in the experiment manifest and does not change after submission
- `problem_count` for `planned_trial_count` computation equals `|problem_ids|` (fixed) or `instance_count` (generated)

---

### FR-5: Solver Set

**Description:**
The solver set is the ordered list of backend identifiers participating in the experiment.

**Acceptance Criteria:**
- The solver set contains at least one `backend_id`
- Each `backend_id` must be registered in the Scheduler's capability profile at the time the experiment is submitted (SPEC-003 FR-4, SPEC-011 FR-7)
- The solver set order is preserved in the experiment manifest and in all artifact outputs; solvers appear in the same order in all comparison tables
- Solver eligibility for specific routing problems is determined by the Scheduler at trial execution time. The harness does not pre-validate per-problem eligibility.
- When the Scheduler rejects a solver as ineligible for a specific problem at trial execution time, the trial outcome is `SchedulerRejected` (FR-11). This is not a harness failure; it is recorded as evidence of Scheduler behavior.

---

### FR-6: Trial Model

**Description:**
A trial is a single execution of one solver against one routing problem instance. Each trial corresponds to exactly one job submitted to the existing job execution pipeline.

**Trial identity:**

| Field | Type | Description |
|---|---|---|
| `trial_id` | System-assigned UUID | Unique identifier for this trial within the experiment |
| `experiment_id` | UUID | Parent experiment |
| `problem_id` | UUID | The routing problem instance used for this trial |
| `backend_id` | string | The solver backend targeted |
| `repetition_index` | uint32 | Zero-based repetition index (0 through `repetition_count − 1`) |
| `job_id` | UUID | Job identifier assigned by the API at trial submission. Null before submission. |
| `trial_seed` | uint64 | The seed supplied as the routing problem's `seed` field for this trial (FR-8) |
| `trial_type` | "reproducible" \| "non_reproducible" | Reproducibility class for this trial, derived from the backend's `determinism_class` (FR-9) |
| `status` | TrialStatus | Current lifecycle state |

**Trial status lifecycle:**

| TrialStatus | Meaning |
|---|---|
| `Pending` | Trial is planned but not yet submitted |
| `Submitted` | Job has been submitted to the API; awaiting Worker consumption |
| `Executing` | Worker has consumed the job; solver execution is in progress |
| `Completed` | Job has reached a terminal state; a `solver_outcome` is recorded |
| `SchedulerRejected` | The Scheduler determined the backend is ineligible for this problem; no job was submitted |
| `HarnessError` | The harness could not submit or track the trial; error reason recorded |

**Acceptance Criteria:**
- Each trial maps to exactly one `job_id` after submission
- A trial transitions to `Completed` when the associated job reaches a terminal state in the Evidence Log (SPEC-006 FR-4)
- A `SchedulerRejected` trial does not submit a job; the Scheduler's rejection reason is recorded in the trial record
- A `HarnessError` trial records the failure reason; the harness continues with remaining trials
- The total number of trials in an experiment equals `planned_trial_count` from the experiment manifest

---

### FR-7: Repetition Model

**Description:**
The repetition count governs how many independent trial executions are performed for each (problem_config, solver) pairing. The term "trial" means one individual execution; the term "repetition" means the repetition number within a set of executions for a pairing.

**Repetition semantics by workload mode:**

| Workload Mode | Repetition Behavior |
|---|---|
| Fixed (Mode A) | All `repetition_count` repetitions for a (problem, solver) pairing submit the same `problem_id` with the same problem seed. The execution_seed received by the solver equals the problem seed (ADR-010 Decision 4). For deterministic backends (nearest-neighbor, greedy insertion), all repetitions produce identical outputs. For stochastic reproducible backends (SA, QAOA simulator), all repetitions produce identical outputs because the problem seed is identical. Repetitions in fixed mode verify execution consistency and serve as reliability checks. |
| Generated (Mode B) | Each repetition uses a distinct routing problem instance generated with a different trial seed (FR-8). The different seed produces a different routing problem geometry while holding structural parameters (stop count, demand ranges, fleet, time window density) constant. For stochastic backends, different seeds produce different execution outcomes. This mode supports genuine statistical distribution of quality outcomes across the repetition set. |

**Relationship between repetition_count and planned_trial_count:**
- `planned_trial_count = problem_count × |solver_set| × repetition_count`
- Example: 5 problems, 3 solvers, 10 repetitions = 150 total trial executions

**Acceptance Criteria:**
- `repetition_count` is a positive integer, minimum 1
- `repetition_index` runs from 0 to `repetition_count − 1` for each (problem_config, solver) pairing
- In fixed workload mode: all repetitions for a given (problem, solver) pair use the same `problem_id`
- In generated workload mode: each repetition uses a distinct trial seed (FR-8) and generates a distinct routing problem instance
- A `repetition_count` of 1 is valid; it produces a single trial per (problem_config, solver) pairing with a single-element sample

---

### FR-8: Seed Strategy

**Description:**
The experiment seed is the top-level entropy source for the experiment. The seed strategy governs how per-trial seeds are derived and how deterministic experiment replay is established.

**Experiment seed:**
The `experiment_seed` is a non-negative 64-bit unsigned integer supplied at experiment submission time. It is recorded in the experiment manifest. Providing the same `experiment_seed` with the same experiment manifest is the necessary and sufficient condition for replaying a generated-workload experiment with identical problem instances.

**Per-trial seed (repetition seeds) in generated workload mode:**
In generated workload mode, the harness derives a distinct trial seed for each (problem_config_index, repetition_index) pair. The trial seed identifies one routing problem instance; it is not derived per backend. All backends in the solver set for the same (problem_config_index, repetition_index) pair use the same derived trial seed and therefore the same `problem_id`, consistent with the instance-sharing invariant established in FR-4. Trial seeds must satisfy the following properties:
- Determinism: the same (experiment_seed, problem_config_index, repetition_index) triple always produces the same trial seed
- Uniqueness: no two trials within the same experiment share the same trial seed
- ADR-010 compliance: the trial seed is submitted as the routing problem's `seed` field, becoming the `execution_seed` passed to the solver by the Worker (ADR-010 Decision 4, SPEC-005 FR-7)

The specific derivation function — the algorithm used to derive trial seeds from the experiment seed and trial coordinates — is an implementation planning decision. It must be documented as part of the implementation and treated as a breaking change if modified (consistent with ADR-010 Decision 5 principles).

**Fixed workload mode seed behavior:**
In fixed workload mode, the problem seeds are fixed (established at routing problem submission time and immutable per SPEC-001). The `experiment_seed` is recorded in the manifest for traceability but does not influence trial execution. Reproducibility of a fixed-workload experiment requires preserving the referenced `problem_ids` and their associated seeds. If a problem is deleted or its record is lost, the experiment is no longer reproducible.

**Hardware backend seed scope:**
For `quantum_hardware` backends (SPEC-019), the trial seed seeds the classical pre-processing phase (parameter initialization and classical optimizer state). Hardware shot outcomes are governed by quantum physical processes and are not reproducible regardless of seed. Trial seeds for hardware experiments are derived using the same strategy as for simulator and classical backends.

**Deterministic experiment replay:**
A generated-workload experiment is fully replayable given:
1. The experiment manifest (including `experiment_seed`, `generation_config`, `solver_set`, `repetition_count`)
2. The same SPEC-002 generator implementation
3. The same seed derivation algorithm

The seed derivation algorithm version must be recorded in the experiment state (in the API's experiment record) so that replay instructions are complete. Without the algorithm version, a researcher cannot confirm which derivation function was used, and replay fidelity cannot be guaranteed.

A fixed-workload experiment is fully replayable given the `problem_ids` remain queryable and the solver backends have not changed their algorithms.

**Acceptance Criteria:**
- `experiment_seed` is a non-negative 64-bit integer
- In generated workload mode: each trial is assigned a distinct trial seed derived from the experiment_seed
- No two trials in the same experiment share the same derived trial seed in generated mode
- In fixed workload mode: `experiment_seed` is recorded in the manifest without modifying problem seeds
- The trial seed appears in the trial record and in the trial results artifact

---

### FR-9: Reproducibility Classification

**Description:**
Each trial and each experiment receives a reproducibility classification derived from the participating backends' `determinism_class` values (SPEC-011 FR-3).

**Trial reproducibility classes:**

| Class | Condition | Meaning |
|---|---|---|
| `reproducible` | Backend `determinism_class` is "Deterministic" or "Stochastic (reproducible)" | Given the same trial inputs (problem, backend, trial_seed), the trial produces identical outputs. SPEC-013, SPEC-014, SPEC-015, SPEC-018 backends are reproducible. |
| `non_reproducible` | Backend `determinism_class` is "Stochastic (non-reproducible)" | Given the same trial inputs, the trial may produce different outputs due to hardware entropy. SPEC-019 (`qaoa-hardware`) is non-reproducible. |

**Experiment reproducibility classes:**

| Class | Condition | Meaning |
|---|---|---|
| `fully_reproducible` | All backends in the solver set are `reproducible` | The entire experiment is reproducible from the manifest and experiment_seed |
| `partially_reproducible` | At least one backend is `reproducible` and at least one is `non_reproducible` | Classical and simulator trial outcomes are reproducible; hardware trial outcomes are not |
| `non_reproducible` | All backends are `non_reproducible` | No trial is expected to produce reproducible results |

**Acceptable hardware variance:**
For `non_reproducible` trials, the following behaviors are expected and are not treated as failures:
- Different shot measurement outcomes across identical circuit executions on hardware (SPEC-019 FR-5)
- Different `hindsight_quality` values across repetitions of the same trial configuration
- Different outcome distributions across two runs of the same `non_reproducible` experiment

**Reproducibility annotation obligations:**
- Each trial record carries its `trial_type` field
- The experiment manifest carries its `reproducibility_class`
- All generated artifacts (experiment summary, benchmark summary) annotate experiments and trials with their reproducibility class
- Aggregate statistics for `non_reproducible` or `partially_reproducible` experiments include an explicit annotation stating that hardware trial outcomes may vary across experiment runs

**Acceptance Criteria:**
- Every trial's `trial_type` is derived from the backend's `determinism_class` at experiment submission time
- The experiment manifest's `reproducibility_class` is derived from the union of all backend determinism classes in the solver set
- An experiment containing only SPEC-013, SPEC-014, SPEC-015, and SPEC-018 backends receives `fully_reproducible`
- An experiment containing SPEC-019 and any other backend receives `partially_reproducible`
- An experiment containing only SPEC-019 receives `non_reproducible`
- Artifact summaries for `partially_reproducible` and `non_reproducible` experiments include the reproducibility annotation

---

### FR-10: Hardware Execution Policy

**Description:**
The hardware execution policy defines how the experiment harness handles the behavioral characteristics unique to `quantum_hardware` backends (SPEC-019): queue waiting, provider failures, transient errors, and non-reproducible results.

**HardwarePolicy fields:**

| Field | Type | Default | Description |
|---|---|---|---|
| `retry_limit` | uint32 | 2 | Maximum retry attempts per trial for retriable failure codes |
| `retry_delay_ms` | uint32 | 5000 | Delay between retry attempts in milliseconds |
| `retry_on_failure_codes` | list[SolverFailureCode] | [InternalError] | Failure codes that trigger a retry. `ExecutionTimeout` and `ExecutionCancelled` are never eligible for retry regardless of this field. |

**Retry policy:**
1. A hardware trial that returns outcome `Failed` with a `failure_code` in `retry_on_failure_codes` is retried up to `retry_limit` times.
2. Retries re-submit the same trial inputs: the same problem instance and the same `backend_id`. The trial seed is unchanged across retry attempts.
3. If the trial fails after all retry attempts are exhausted, the trial is recorded as `Completed` with `solver_outcome = Failed`.
4. The retry history is recorded in the trial record: the number of attempts and the `failure_code` from each attempt.
5. A trial that returns `Timeout` is not retried. A queue timeout is evidence of hardware execution cost and is a data point, not a transient error.
6. `Cancelled` trials are not retried.

**Provider outage treatment:**
If all retry attempts for a trial fail with provider connectivity errors (a subset of `InternalError`), the trial is recorded as `Completed` with `solver_outcome = Failed`. The harness records the provider error context in the trial's `harness_failure_reason`. The experiment continues with remaining trials.

**Queue timeout treatment:**
A trial whose `execution_timeout_ms` budget is exhausted during queue waiting (SPEC-019 FR-8) returns `outcome = Timeout` with no solution. This is a valid solver outcome. The queue wait time is recorded in the trial evidence from `qaoa.hardware.queue_wait_ms` (FR-12). No retry is performed.

**Failed trial behavior:**
Trials that complete with `solver_outcome = Failed` contribute to the `trials_failed` outcome count. They do not contribute to quality statistics (FR-13). They are part of the experiment evidence and are included in the experiment summary.

**Missing result handling:**
If the harness cannot retrieve the Evidence Log record for a completed job, the harness retries the read up to 3 times with exponential backoff starting at 500 ms before classifying the trial. If the record remains unavailable after retry exhaustion, the trial is marked with `status = HarnessError` and a `harness_failure_reason` identifying the read failure. The trial does not contribute to quality statistics. The experiment continues. HarnessError trials are reported in the experiment summary as a count distinct from `trials_failed`.

**Effect on experiment validity:**
Hardware failures at any rate do not invalidate the experiment at the harness level. An experiment where all hardware trials fail still reaches `Completed` status and produces summary artifacts documenting the full outcome distribution. The experiment summary reports the evidence as it is, including the failure distribution, for the researcher to evaluate.

**Acceptance Criteria:**
- Retries apply only to failure codes in `retry_on_failure_codes`; `Timeout` and `Cancelled` outcomes are never retried regardless of `retry_on_failure_codes`
- After `retry_limit` exhaustion, the trial terminates with `solver_outcome = Failed`
- Retry history (attempt count and per-attempt failure code) is recorded in the trial record
- Hardware failures do not halt or invalidate the experiment
- A Timeout trial produced by queue exhaustion is recorded as `Completed` with `solver_outcome = Timeout`; it is not retried

---

### FR-11: Experiment Lifecycle

**Description:**
The experiment harness manages the experiment through a defined lifecycle. The lifecycle governs when trials are submitted, when evidence is collected, and when summary artifacts are produced.

**ExperimentStatus lifecycle:**

| Status | Meaning | Terminal |
|---|---|---|
| `Created` | Manifest persisted; workload validation in progress; no trials submitted | No |
| `Running` | One or more trials are in a non-terminal state | No |
| `Completed` | All planned trials have reached a terminal trial status | Yes |
| `Failed` | The experiment could not be started due to a configuration or infrastructure error | Yes |

**Transition rules:**
- `Created` → `Running`: when the first trial is submitted to the job API
- `Running` → `Completed`: when the last trial reaches a terminal trial status (`Completed`, `SchedulerRejected`, or `HarnessError`)
- `Created` → `Failed`: when the experiment manifest is invalid, workload validation fails for all instances, or no trials can be submitted
- The `Failed` experiment status is distinct from trials with `solver_outcome = Failed`; a `Completed` experiment may contain many trials with `solver_outcome = Failed`

**Successful experiment completion:**
An experiment reaches `Completed` status when all submitted trials have reached a terminal trial status. For experiments without partial generation failure, this equals `planned_trial_count`. For experiments where partial workload generation failure reduced the actual number of trials submitted (see Failure Modes and FR-3), the experiment completes when all actually-submitted trials have reached a terminal trial status; the manifest's `planned_trial_count` is not the completion criterion in this case. This is the mechanical completion criterion. It does not imply that the trials produced sufficient quality-eligible evidence to support the experiment's research question.

**Partial completion:**
An experiment in `Running` status with some trials in terminal states and others still in progress is partially complete. The trial results collection (Artifact 2) reflects all trials that have reached terminal status and is queryable at any point during execution. No experiment summary artifact (Artifact 3) is produced during the `Running` state. Summary statistics are not pre-computed during execution; they are derived at `Completed` time from the full trial results collection. The final experiment summary artifact is produced once, when the experiment reaches `Completed` status.

**Failed experiment:**
The experiment transitions to `Failed` status when:
- The manifest fails validation (unrecognized backend_ids, unresolvable problem_ids, missing required fields)
- Workload generation fails for all instances in generated mode (if some instances succeed, the experiment proceeds with available instances)
- No trials can be submitted due to a persistent infrastructure failure

An experiment that begins trial execution never transitions to `Failed` regardless of individual trial outcomes.

**Acceptance Criteria:**
- `ExperimentStatus` transitions are observable through the experiment status record
- The experiment transitions from `Running` to `Completed` when the last trial reaches a terminal status
- The experiment summary artifact is produced when the experiment reaches `Completed`
- An experiment where all trials have `solver_outcome = Failed` still reaches `Completed` status
- A `Failed` experiment records the failure reason in the manifest record

---

### FR-12: Evidence Collection

**Description:**
The harness collects evidence from completed trials by reading from the Evidence Log (SPEC-006). The harness never writes to the Evidence Log. Evidence is collected at three levels: per trial, per (problem, solver) aggregate, and per experiment.

**Per-trial evidence:**
For each trial that reaches `Completed` status, the harness reads the following from the Evidence Log after the associated job reaches a terminal state:

| Evidence Field | Source | Notes |
|---|---|---|
| `job_id` | Trial record | Links to the full Evidence Log record set (SPEC-006 FR-2) |
| `solver_outcome` | SolverResponse.outcome (SPEC-004 FR-4) | Succeeded, Timeout, Cancelled, Failed |
| `solution_present` | SolverResponse | True when a RoutePlan was returned |
| `hindsight_quality` | QualityEvaluationResult (SPEC-007 FR-7) | Total route distance in km; null when no solution or simulation not performed |
| `total_route_distance_km` | QualityMetrics (SPEC-007 FR-6) | Identical to hindsight_quality; present for explicit field labeling |
| `max_route_distance_km` | QualityMetrics (SPEC-007 FR-6) | Null when no solution |
| `routes_used` | QualityMetrics (SPEC-007 FR-6) | Null when no solution |
| `time_window_feasible` | QualityMetrics (SPEC-007 FR-6) | Null when no solution or simulation not performed |
| `time_window_violation_count` | QualityMetrics (SPEC-007 FR-6) | Null when no solution or simulation not performed |
| `quality_comparison_eligible` | QualityMetrics (SPEC-007 FR-6) | False when time windows present and plan is infeasible |
| `execution_duration_ms` | ExecutionStatistics.execution_duration_ms (SPEC-004 FR-6) | Null for some Timeout and Cancelled outcomes |
| `trial_seed` | Harness trial record | The seed used for this trial's problem submission |
| `trial_type` | Harness trial record | "reproducible" or "non_reproducible" |
| `retry_count` | Harness trial record | Number of retry attempts; 0 for non-hardware trials |

Hardware-specific per-trial evidence (collected from `extension_metadata` for `quantum_hardware` backend trials only):

| Evidence Field | extension_metadata key | Notes |
|---|---|---|
| `hardware_queue_wait_ms` | `qaoa.hardware.queue_wait_ms` | Queue wait duration; null if job timed out before hardware execution began |
| `hardware_calibration_id` | `qaoa.hardware.calibration_id` | Calibration state at execution time; null if job timed out before hardware execution |
| `hardware_estimated_cost` | `qaoa.hardware.estimated_cost` | Monetary cost estimate; null if hardware execution did not begin |
| `hardware_provider_backend` | `qaoa.hardware.provider_backend` | Hardware device identifier |

**Per (problem, solver) aggregate evidence:**
Computed across all `repetition_count` trials for one (problem, solver) pairing, after all repetitions reach terminal status.

| Field | Type | Description |
|---|---|---|
| `problem_id` | UUID | |
| `backend_id` | string | |
| `trials_total` | uint32 | Total trials for this pairing (`repetition_count`) |
| `trials_succeeded` | uint32 | Trials with `solver_outcome = Succeeded` |
| `trials_timeout` | uint32 | Trials with `solver_outcome = Timeout` |
| `trials_failed` | uint32 | Trials with `solver_outcome = Failed` |
| `trials_cancelled` | uint32 | Trials with `solver_outcome = Cancelled` |
| `trials_scheduler_rejected` | uint32 | Trials with `status = SchedulerRejected` |
| `trials_harness_error` | uint32 | Trials with `status = HarnessError` |
| `trials_quality_eligible` | uint32 | Trials where `quality_comparison_eligible = true` |
| `quality_stats` | QualityStats \| null | Null when `trials_quality_eligible = 0`. Defined in FR-13. |
| `runtime_stats` | RuntimeStats \| null | From all trials with `execution_duration_ms` present. Null when no timing data available. Defined in FR-13. |

**Per experiment aggregate evidence:**
Computed across all (problem, solver) pairings when the experiment reaches `Completed` status.

| Field | Type | Description |
|---|---|---|
| `experiment_id` | UUID | |
| `trials_planned` | uint32 | From manifest |
| `trials_completed` | uint32 | Trials in terminal state (all statuses) |
| `per_solver_summary` | list[SolverSummary] | One entry per `backend_id` in the solver set: outcome distribution totals across all problems and repetitions, combined runtime statistics |
| `cross_solver_comparison` | list[ProblemComparison] | One entry per problem: for each backend in the solver set, the `quality_stats` and `runtime_stats` for that (problem, solver) pairing, ranked by `mean_km` (ascending) among quality-eligible entries |

**Acceptance Criteria:**
- Per-trial evidence is collected after each trial reaches a terminal state
- Per (problem, solver) aggregates are computed when all repetitions for that pairing have reached terminal status
- The experiment aggregate is computed when the experiment reaches `Completed` status
- Evidence collection reads from the Evidence Log and does not modify it
- Failed trials contribute to `trials_failed` but not to `quality_stats`
- The `cross_solver_comparison` ranks backends by `mean_km` only for entries where `quality_stats` is non-null; null-quality-stats entries are listed separately as "no quality data"

---

### FR-13: Statistical Analysis Model

**Description:**
The harness computes descriptive statistics over quality-eligible and runtime-eligible trial sets. All statistics are computed by the harness from Evidence Log data; they are not computed by the Evidence Log or individual trial records.

**Quality-eligible trial definition:**
A trial is included in quality statistics when all of the following are true:
- `quality_comparison_eligible = true` (SPEC-007 FR-6)
- `solution_present = true`
- Route simulation completed successfully (SPEC-007 FR-13 conditions did not apply)

**Runtime-eligible trial definition:**
A trial is included in runtime statistics when `execution_duration_ms` is non-null.

**Outcome-eligible trial definition:**
All trials in a pairing contribute to the outcome distribution counts regardless of quality or runtime eligibility.

**QualityStats (computed over quality-eligible trials for one (problem, solver) pairing):**

| Field | Type | Notes |
|---|---|---|
| `sample_count` | uint32 | Number of quality-eligible trials |
| `mean_km` | float64 | Arithmetic mean of `hindsight_quality` |
| `median_km` | float64 | Median of `hindsight_quality` |
| `min_km` | float64 | Minimum `hindsight_quality` |
| `max_km` | float64 | Maximum `hindsight_quality` |
| `stddev_km` | float64 \| null | Population standard deviation of `hindsight_quality`; null when `sample_count < 2` |

**RuntimeStats (computed over runtime-eligible trials for one (problem, solver) pairing):**

| Field | Type | Notes |
|---|---|---|
| `sample_count` | uint32 | Number of trials with `execution_duration_ms` present |
| `mean_ms` | float64 | Arithmetic mean of `execution_duration_ms` |
| `median_ms` | float64 | Median of `execution_duration_ms` |
| `min_ms` | float64 | Minimum `execution_duration_ms` |
| `max_ms` | float64 | Maximum `execution_duration_ms` |
| `stddev_ms` | float64 \| null | Population standard deviation; null when `sample_count < 2` |

**Treatment of specific outcome types:**

| Outcome | Quality Statistics | Runtime Statistics | Outcome Distribution |
|---|---|---|---|
| Failed | Excluded | Excluded (execution_duration_ms typically null) | Counted in `trials_failed` |
| Timeout, no solution | Excluded | Included if execution_duration_ms present | Counted in `trials_timeout` |
| Timeout, solution present, quality_comparison_eligible = true | Included | Included | Counted in `trials_timeout` and `trials_quality_eligible` |
| Timeout, solution present, quality_comparison_eligible = false | Excluded | Included | Counted in `trials_timeout` |
| Scheduler rejected | Excluded | Excluded | Counted in `trials_scheduler_rejected` |
| Harness error | Excluded | Excluded | Counted in `trials_harness_error` |

**Cross-solver comparison validity:**
`hindsight_quality` values are dimensionally comparable only within the same routing problem (SPEC-007 FR-7). The harness produces cross-solver comparisons at the per-problem level. Aggregating `hindsight_quality` across problems with different geometries is not performed by the harness.

In Generated Mode, the per-(problem_slot, solver) QualityStats aggregate across `repetition_count` routing problem instances generated from the same structural parameters. Cross-solver comparisons within a problem slot are valid: the instance-sharing invariant (FR-4) guarantees that for each repetition index, all backends executed against the same generated problem instance (same `problem_id`). The QualityStats distribution reflects each solver's quality behavior across a sample drawn from the problem class defined by the generation configuration; it is not a within-a-single-problem comparison.

**No advanced statistical tests:**
The harness computes descriptive statistics only: mean, median, min, max, standard deviation. Hypothesis testing, confidence intervals, p-values, and significance thresholds are not computed. These are deferred to OQ-4.

**Acceptance Criteria:**
- `sample_count` in QualityStats is the count of quality-eligible trials for the (problem, solver) pairing
- All five descriptive statistics are computed from `hindsight_quality` values of quality-eligible trials only
- `stddev_km` and `stddev_ms` are null when the respective `sample_count < 2`
- Failed trials are not present in quality or runtime statistics
- Cross-solver quality comparisons use per-problem `mean_km` values, not cross-problem aggregates

---

### FR-14: Artifact Generation

**Description:**
The experiment harness produces four classes of artifact. Each artifact is structured data. Rendering, visualization, and presentation are owned by SPEC-021 and SPEC-022. The harness does not produce rendered HTML, charts, or publication-ready output.

**Artifact 1: Experiment Manifest**

Produced: when the experiment is submitted (FR-3).
Content: the complete immutable experiment configuration as defined in FR-3.
Format: structured data (JSON), persisted to the experiment persistence store (OQ-1).
Consumers: SPEC-021 (API exposure), reproducibility workflows.
Ownership: SPEC-020.

**Artifact 2: Trial Results Collection**

Produced: incrementally as trials reach terminal status. The collection grows as the experiment executes.
Content: one TrialResult record per trial containing all per-trial evidence fields defined in FR-12.
Format: structured data (JSON records), persisted alongside trial records.
Consumers: statistical analysis (FR-13), SPEC-021 (API exposure).
Ownership: SPEC-020.

The trial results collection is queryable at any point during experiment execution. It contains only trial records that have reached terminal status. Incomplete trials are not included.

**Artifact 3: Experiment Summary**

Produced: when the experiment reaches `Completed` status. Produced once. Not updated after production.
Content:
- Experiment identity fields (experiment_id, benchmark_id, experiment_name, reproducibility_class)
- The complete trial results collection (Artifact 2)
- Per (problem, solver) aggregate evidence (FR-12)
- Per-experiment aggregate evidence and cross-solver comparison (FR-12)
- All QualityStats and RuntimeStats (FR-13)
- Outcome distribution by solver and by problem
- Reproducibility annotation for the experiment and for non-reproducible backends

Format: structured data (JSON), persisted to the experiment persistence store.
Consumers: SPEC-021 (API exposure), SPEC-022 (visualization).
Ownership: SPEC-020.

**Artifact 4: Benchmark Summary**

Produced: when any experiment under the `benchmark_id` reaches `Completed` status. Updated each time an additional experiment under the same `benchmark_id` completes.
Content:
- Benchmark definition fields (from the benchmark manifest, FR-2): benchmark_id, benchmark_name, research_question, hypothesis, null_hypothesis, controls, independent variables, dependent variables
- A list of all experiments under this benchmark_id: experiment_id, experiment_name, status, reproducibility_class
- For each completed experiment: per-solver outcome distribution summary and QualityStats summary
- Cross-experiment notes: which experiments are directly comparable (same problem instances, same repetition count) and which are not

Format: structured data (JSON), persisted under the benchmark_id.
Consumers: SPEC-022 (visualization), technical writing workflows.
Ownership: SPEC-020.

**Individual evidence reports (reference only):**
Individual job evidence reports are generated by SPEC-009 for each trial's job_id. The harness references these reports by `job_id` in the Trial Results Collection. The harness does not reproduce, own, or modify SPEC-009 artifacts.

**Artifact ownership boundaries:**

| Artifact | Produced By | API-Exposed By | Visualized By |
|---|---|---|---|
| Benchmark Manifest | SPEC-020 | SPEC-021 | n/a |
| Experiment Manifest | SPEC-020 | SPEC-021 | n/a |
| Trial Results Collection | SPEC-020 | SPEC-021 | n/a |
| Experiment Summary | SPEC-020 | SPEC-021 | SPEC-022 |
| Benchmark Summary | SPEC-020 | SPEC-021 | SPEC-022 |
| Individual Job Evidence Reports | SPEC-009 | SPEC-021 | n/a |

**Acceptance Criteria:**
- The experiment manifest is produced and persisted before any trials are submitted
- The trial results collection is queryable after each trial completes
- The experiment summary is produced when the experiment reaches `Completed` status and not before
- The benchmark summary is produced or updated when any member experiment reaches `Completed` status
- Artifacts contain structured data only; no rendered HTML, charts, or visualization output is embedded
- The trial results collection references individual job evidence reports by `job_id`; it does not reproduce their content

---

### FR-15: Observability

**Description:**
The experiment harness emits observable signals for experiment progress, trial progress, solver progress, and evidence completeness. These are in addition to the existing observability obligations of the job execution pipeline (SPEC-005, SPEC-006).

**Experiment progress:**
The following structured events are emitted:
- Experiment submitted: experiment_id, benchmark_id, planned_trial_count, workload_mode, solver_set count, reproducibility_class
- Experiment started (first trial submitted): experiment_id, timestamp
- Experiment completed: experiment_id, trials_planned, trials_in_each_terminal_state, trials_quality_eligible
- Experiment failed: experiment_id, failure_reason

**Trial progress:**
The following structured events are emitted:
- Trial submitted: experiment_id, trial_id, problem_id, backend_id, repetition_index, job_id
- Trial completed: experiment_id, trial_id, solver_outcome, solution_present, quality_comparison_eligible, execution_duration_ms
- Trial SchedulerRejected: experiment_id, trial_id, backend_id
- Trial HarnessError: experiment_id, trial_id, error_reason
- Hardware retry: experiment_id, trial_id, retry_attempt_number, previous_failure_code

**Solver progress:**
For each trial in `Executing` state, solver execution progress is observable through the underlying job status endpoint (SPEC-008, SPEC-021). The harness does not add additional solver-level observability beyond what the job pipeline provides.

**Evidence completeness:**
Evidence completeness is defined as `trials_quality_eligible / planned_trial_count` at any point during execution. It is a value in [0.0, 1.0]. A completeness of 0.0 means no quality-eligible results are available. A completeness of 1.0 means all planned trials produced quality-eligible results.

Evidence completeness is observable as a derived metric from the running trial counts. It is reported in the experiment progress record on each trial completion.

**What constitutes successful experiment completion:**
- `ExperimentStatus = Completed`
- All `planned_trial_count` trials are in a terminal trial status
- The experiment summary artifact has been produced

**What constitutes partial completion:**
- `ExperimentStatus = Running`
- Some trials have reached terminal status; others have not
- The trial results collection (Artifact 2) reflects completed trials and is queryable; no experiment summary artifact (Artifact 3) exists in this state

**What constitutes a failed experiment:**
- `ExperimentStatus = Failed`
- No trials were submitted
- The failure reason is recorded in the manifest record
- No summary artifact is produced

**Acceptance Criteria:**
- `ExperimentStatus` transitions are observable as structured events
- `trials_completed` (running count) and `planned_trial_count` are observable at any point during execution
- Evidence completeness is computable and observable from the trial counts
- A structured event is emitted when the experiment reaches `Completed` status
- A structured event is emitted when the experiment reaches `Failed` status

---

# Non-Requirements

- No solver algorithm implementation. The harness submits jobs; it does not implement optimization.
- No routing problem generation logic. Problem generation is delegated to SPEC-002; the harness supplies generation configurations and seed values.
- No scheduler policy modification. Solver eligibility decisions are made by the Scheduler at trial execution time; the harness records but does not influence them.
- No quality evaluation computation. All quality metrics are read from the Evidence Log where they were computed by Core (SPEC-007) and persisted by the Worker (SPEC-005).
- No report rendering. Individual evidence reports per job are generated by SPEC-009. The harness references them by job_id.
- No dashboard visualization. Visualization is owned by SPEC-022.
- No publication template generation. Technical writing artifacts are out of scope; raw benchmark data is the output.
- No distributed or parallel trial execution. All trial submissions are sequential in the MVP. Large-scale parallel execution is deferred (OQ-5).
- No experiment amendment. An experiment manifest cannot be modified after submission. Amending a benchmark requires submitting a new experiment.
- No advanced statistical tests. Hypothesis testing, p-values, confidence intervals, and significance thresholds are not computed by the harness.
- No cross-problem quality comparison. `hindsight_quality` statistics are computed per problem, not aggregated across problems with different geometries (SPEC-007 FR-7).
- No SolverResponse redesign. The harness consumes existing SolverOutcome, SolverFailureDetail, and ExecutionStatistics fields without modification.
- No Evidence Log redesign. The harness reads from existing Evidence Log tables; it does not add fields or tables to the Evidence Log schema.

---

# Assumptions

- The existing job execution pipeline (SPEC-005 Worker, SPEC-004 Solver Contract, SPEC-006 Evidence Log) is operational and persists evidence records for all submitted jobs.
- Core quality evaluation (SPEC-007) produces quality metrics for all trials where a solution is present.
- All backends in the solver set are registered in the Scheduler at experiment submission time.
- A `quantum_hardware` trial that returns `Timeout` due to queue exhaustion is a valid evidence data point, not a system malfunction.
- The Evidence Log persists evidence records in a queryable form; the harness can retrieve evidence for any job_id.
- Statistical descriptive summaries are computed with double-precision floating-point arithmetic. No special handling for floating-point edge cases is required beyond what SPEC-007 already applies.
- For generated workload experiments, SPEC-002 can produce distinct routing problem instances for any derived seed within the experiment's generation configuration.
- A `repetition_count` of 1 produces valid (if single-sample) statistical summaries. The adequacy of single-sample evidence is the experiment designer's responsibility.

---

# Constraints

- The harness must not modify the Evidence Log. It reads from the Evidence Log; the Worker is the exclusive Evidence Log writer (SPEC-006 FR-1.3).
- The harness must not invoke solver backends directly. Trial execution uses the existing job submission API (SPEC-008), not direct backend invocation.
- The harness must not re-implement quality evaluation. All quality metrics are read from the Evidence Log.
- The harness must comply with ADR-010 when deriving per-trial seeds in generated workload mode. The derived trial seed must be submitted as the routing problem's `seed` field so that the Worker's execution_seed derivation (ADR-010 Decision 4) produces the correct value.
- The experiment manifest is immutable after creation. No amendment mechanism is defined.
- The harness must not modify the Scheduler policy. Solver eligibility is determined by the Scheduler; the harness records rejections as evidence.

---

# Inputs

**Benchmark manifest submission:**

| Field | Type | Required |
|---|---|---|
| `benchmark_id` | string | Yes |
| `benchmark_name` | string | Yes |
| `research_question` | string | Yes |
| `hypothesis` | string | Yes |
| `null_hypothesis` | string | Yes |
| `controls` | list[string] | Yes |
| `independent_variables` | list[string] | Yes |
| `dependent_variables` | list[string] | Yes |

**Experiment submission:**

| Field | Type | Required |
|---|---|---|
| `benchmark_id` | string | Yes |
| `experiment_name` | string | Yes |
| `experiment_seed` | uint64 | Yes |
| `workload_set` | WorkloadSetDefinition | Yes |
| `solver_set` | list[backend_id] | Yes |
| `repetition_count` | uint32 | Yes |
| `execution_timeout_ms_per_trial` | uint32 | Yes |
| `scheduler_config_id` | UUID | Optional |
| `hardware_policy` | HardwarePolicy | Optional |

**Sources:** Developer CLI (SPEC-016); pending benchmark submission command.

---

# Outputs

- **Experiment manifest** (FR-14, Artifact 1): immutable experiment configuration
- **Trial results collection** (FR-14, Artifact 2): per-trial evidence records, produced incrementally
- **Experiment summary** (FR-14, Artifact 3): aggregate statistics and cross-solver comparison, produced at completion
- **Benchmark summary** (FR-14, Artifact 4): cross-experiment comparison under a benchmark_id, produced and updated as experiments complete
- **ExperimentStatus events** (FR-15): structured progress signals emitted at each experiment and trial transition

---

# Failure Modes

### Invalid Experiment Manifest

Cause: Missing required fields, unrecognized backend_ids, unresolvable problem_ids in fixed mode, or malformed generation configuration in generated mode.

Expected behavior: The experiment transitions to `Failed` status. No trials are submitted. The failure reason is recorded.

Expected fallback: None. The submitter must correct the manifest and resubmit.

---

### Workload Generation Failure (Generated Mode)

Cause: SPEC-002 cannot generate one or more problem instances from the derived seeds.

Expected behavior: If no instances can be generated, the experiment transitions to `Failed`. If some instances are generated and others fail, the harness proceeds with available instances and records the partial generation failure.

Expected fallback: The experiment continues if at least one instance is available. The manifest's `planned_trial_count` is not modified. The experiment's runtime state records the partial generation failure and tracks an `effective_trial_count` reflecting the actual number of trials to be submitted. The experiment summary's `trials_planned` field reflects the effective count, not `planned_trial_count`.

---

### Trial Submission Failure

Cause: The harness cannot submit a trial to the job API.

Expected behavior: The trial transitions to `HarnessError`. The harness continues with remaining trials. The experiment summary reports the `HarnessError` count.

Expected fallback: None. The experiment completes with the available evidence.

---

### Scheduler Eligibility Rejection

Cause: The Scheduler determines a backend is ineligible for a specific routing problem at trial execution time.

Expected behavior: The trial transitions to `SchedulerRejected`. No job is submitted. The rejection is recorded as evidence. The experiment continues.

Expected fallback: None. `SchedulerRejected` trials are evidence of scheduler behavior, not failures.

---

### Hardware Provider Failure

Cause: A `quantum_hardware` trial returns `Failed` with `InternalError` due to provider unavailability.

Expected behavior: The harness retries up to `retry_limit` times (FR-10). After retry exhaustion, the trial terminates with `solver_outcome = Failed`. The experiment continues.

Expected fallback: None. Hardware failures are evidence data points.

---

### Evidence Log Read Failure

Cause: The harness cannot retrieve Evidence Log records for a completed job.

Expected behavior: The trial transitions to `HarnessError` with `harness_failure_reason` identifying the read failure. The trial does not contribute to quality statistics. The experiment continues.

Expected fallback: None. HarnessError trials are counted in the experiment summary.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer (C++ Core) | No |
| API Layer (C# ASP.NET) | Yes — experiment and benchmark submission endpoints; status and artifact retrieval endpoints |
| Persistence | Yes — experiment manifests, trial records, artifact storage. Not covered by existing Evidence Log schema (OQ-1). |
| Solver Runtime (Worker) | No — the Worker executes individual jobs; it is unaware of experiments |
| Evidence Log (SPEC-006) | Yes — harness reads from Evidence Log; no writes to Evidence Log tables |
| Observability | Yes — experiment progress events, trial progress events, evidence completeness metrics |
| Configuration | Yes — hardware policy, retry limits, seed derivation algorithm |
| Report Generator (SPEC-009) | No — individual evidence reports are generated per job as normal; the harness references them by job_id |
| CLI (SPEC-016) | Yes — CLI is the harness orchestration executor (SPEC-016 owns "Experiment orchestration: multi-job submission and result collection"). The CLI runs the trial submission loop, polls job status endpoints, and notifies the API when each trial reaches terminal status. A SPEC-016 amendment is required for experiment and benchmark manifest submission commands; see Documentation Updates Required. |

The experiment harness is a new system component with new persistence requirements. Its schema is not covered by the existing Evidence Log specification (SPEC-006) or the routing problem schema (SPEC-012). A new persistence schema or an extension to SPEC-012 is required (OQ-1).

---

# Testability

**Manifest and submission:**
- A valid experiment manifest is persisted before any trials are submitted
- An invalid manifest produces `Failed` status with a structured error identifying the invalid field
- `planned_trial_count = |problems| × |solver_set| × repetition_count` is correctly computed
- A benchmark manifest is persisted and retrievable by `benchmark_id`

**Workload set:**
- In fixed mode: a problem_id that does not resolve in PostgreSQL causes the experiment to fail at validation before any trials are submitted
- In generated mode: the harness generates `instance_count × repetition_count` distinct problem instances

**Trial execution:**
- Each (problem, solver, repetition_index) combination produces exactly one trial record
- In fixed mode: repetitions for the same (problem, solver) submit the same `problem_id`
- In generated mode: each repetition uses a distinct trial seed and produces a distinct problem instance
- The total trial count equals `planned_trial_count`

**Seed strategy:**
- In generated mode: no two trials in the same experiment share the same derived trial seed
- In generated mode: the same experiment_seed and manifest configuration produce the same trial seed assignments across two identical submissions

**Reproducibility classification:**
- A trial over a `quantum_hardware` backend receives `trial_type = non_reproducible`
- A trial over any other registered backend receives `trial_type = reproducible`
- An experiment containing only SPEC-013, SPEC-014, SPEC-015, SPEC-018 backends receives `fully_reproducible`
- An experiment containing SPEC-019 and any other backend receives `partially_reproducible`

**Hardware policy:**
- A hardware trial returning `Failed` with `InternalError` is retried up to `retry_limit` times
- A hardware trial returning `Timeout` is not retried
- After retry exhaustion, the trial terminates with `solver_outcome = Failed` and retry history present in the trial record

**Scheduler rejection:**
- A trial where the Scheduler rejects the backend receives `status = SchedulerRejected`; no `job_id` is assigned; no job is submitted

**Evidence collection:**
- Per-trial evidence fields are correctly read from the Evidence Log after job completion
- A trial with `quality_comparison_eligible = false` is excluded from QualityStats
- A trial with `solver_outcome = Failed` contributes to `trials_failed` and not to quality or runtime statistics
- Hardware extension_metadata fields are collected for `quantum_hardware` backend trials

**Statistical analysis:**
- `mean_km`, `median_km`, `min_km`, `max_km` are computed only from quality-eligible trials
- `stddev_km` is null when `sample_count < 2`
- `stddev_ms` is null when the runtime `sample_count < 2`
- A Timeout trial with a quality-eligible solution is included in QualityStats
- A Timeout trial with no solution is excluded from QualityStats and counted in `trials_timeout`

**Artifact generation:**
- The experiment summary is produced when the experiment reaches `Completed` status
- The benchmark summary is produced or updated when any member experiment reaches `Completed` status
- Artifacts contain structured data only; no rendered HTML or visualization content

**Observability:**
- `ExperimentStatus` transitions produce observable events
- `trials_completed` increments as each trial reaches a terminal status
- Evidence completeness is computable from observable trial counts

---

# Observability Requirements

**Structured log events:**
- Experiment submitted: experiment_id, benchmark_id, planned_trial_count, workload_mode, solver_set, reproducibility_class
- Experiment started: experiment_id, timestamp of first trial submission
- Trial submitted: experiment_id, trial_id, problem_id, backend_id, repetition_index, job_id
- Trial completed: experiment_id, trial_id, solver_outcome, solution_present, quality_comparison_eligible, execution_duration_ms
- Trial SchedulerRejected: experiment_id, trial_id, backend_id
- Trial HarnessError: experiment_id, trial_id, error_reason
- Hardware retry: experiment_id, trial_id, retry_attempt_number, previous_failure_code
- Experiment completed: experiment_id, trials_planned, trials_succeeded, trials_timeout, trials_failed, trials_quality_eligible
- Experiment failed: experiment_id, failure_reason

**Metrics:**
- Active experiments count (gauge)
- Trial outcome counts by experiment_id and backend_id (counter: Succeeded, Timeout, Failed, Cancelled, SchedulerRejected, HarnessError)
- Evidence completeness per experiment (gauge: 0.0 to 1.0)
- Hardware retry attempts by backend_id (counter)
- Experiment duration from first trial to completion (histogram)

**Trace spans:**
- `experiment.submit`: covers manifest validation and persistence
- `experiment.run`: covers the full experiment execution lifecycle; parent of trial spans
- `trial.submit`: covers job submission for one trial; child of `experiment.run`
- `trial.collect`: covers Evidence Log read and evidence collection after trial completion; child of `experiment.run`
- `experiment.summarize`: covers experiment summary artifact generation at completion; child of `experiment.run`

---

# Security Considerations

The experiment harness submits routing jobs on behalf of the experiment submitter using the existing job API. No new authentication or authorization mechanisms are defined in the MVP; existing API security applies.

**Input validation:** All experiment manifest fields must be validated before any trials are submitted. Invalid solver_ids, unresolvable problem_ids, and out-of-range repetition counts must be rejected at submission time before any jobs are submitted.

**Experiment scope:** A large experiment can generate a high volume of job submissions (`planned_trial_count` jobs). Rate limiting and experiment-size constraints are deferred (OQ-5) but should be addressed before exposing experiment submission to untrusted callers.

**Cost exposure:** Experiments that include `quantum_hardware` backends may incur monetary cost (SPEC-019 extension_metadata `qaoa.hardware.estimated_cost`). Cost evidence is included in trial records. No cost cap or pre-authorization mechanism is defined in this specification; see OQ-9. Additionally, no experiment-level cancellation mechanism is defined (OQ-8); a hardware experiment that begins cannot be halted at the harness level once the CLI process has started the submission loop. The absence of a cost cap and the absence of an abort mechanism are compounding risks for hardware experiments.

**Seed handling:** Experiment seeds are non-sensitive configuration values. They may appear in logs and artifact outputs. No special secret handling is required.

---

# Performance Considerations

**Sequential trial submission:** The MVP submits trials sequentially. For experiments with a large `planned_trial_count`, sequential submission adds overhead proportional to API round-trip latency. The submission overhead must be measured during implementation.

**Evidence collection latency:** The harness must poll or wait for each trial to reach a terminal state before collecting evidence. The polling interval and collection overhead must not dominate total experiment duration for small workload experiments.

**Hardware queue times:** Hardware experiments (SPEC-019) may have significant queue wait times counted against `execution_timeout_ms` per trial. An experiment containing hardware trials may take substantially longer to complete than a classical experiment with the same trial count. This is observable behavior, not a harness performance issue.

**Artifact generation:** Experiment summary computation traverses all trial records for an experiment. For large trial counts, the traversal time must be measured and must not exceed the time needed to complete the last trial.

Do not invent specific latency targets. These areas require measurement during implementation.

---

# Documentation Updates Required

- docs/architecture.md: Add Benchmark and Experiment Harness to Major Components; add experiment artifacts to the data flow description
- SPEC-002: Confirm GenerationConfig fields are compatible with the harness WorkloadSetDefinition (FR-4 Mode B); confirm that SPEC-002 generation from a derived seed with fixed structural parameters produces the expected instance
- SPEC-008 (API Control Plane): SPEC-020 defines the functional requirements for the following new API endpoints; their addition requires a SPEC-008 amendment: experiment manifest submission, benchmark manifest submission, experiment status retrieval, trial results retrieval, trial evidence collection (CLI-triggered after job terminal state), experiment summary retrieval, benchmark summary retrieval. SPEC-020 owns these functional requirements; SPEC-008 owns the HTTP interface contracts (method, request/response schema, error codes). These backend API contracts are not owned by SPEC-021. The SPEC-008 amendment is a governance artifact subject to Engineering Review, Architecture Review, and Acceptance Review; it should be drafted alongside or immediately after SPEC-020 acceptance.
- SPEC-016 (CLI): SPEC-016 FR-1 defines an existing `daedalus experiment run <file>` command for single-problem, multi-configuration experiments. SPEC-020's experiment model (workload sets, solver sets, repetitions, benchmark manifests) is a superset of this simpler concept. A SPEC-016 amendment is required to either: (a) extend `experiment run` to accept the SPEC-020 manifest format; or (b) introduce a distinct command (e.g., `daedalus benchmark run <manifest.json>`) for SPEC-020 experiments while preserving the simpler existing command. The CLI is the harness orchestration executor; the SPEC-016 amendment must account for the long-running CLI session requirement for hardware experiments.
- SPEC-001 (Routing Problem Model): Add an optional experiment provenance field to routing problem submissions so that harness-generated problems (Generated Mode) are distinguishable from regular job submissions by carrying the generating `experiment_id`. Amendment scope is minimal (one optional field); routing problem semantics, solver contract, and Evidence Log are unaffected.
- SPEC-021 (pending specification): Define API endpoint contracts for all experiment and benchmark artifact operations
- SPEC-022 (pending specification): Define visualization requirements for experiment summary and benchmark summary artifacts

---

# Open Questions

## OQ-1 (Blocking): Experiment Persistence Schema

What is the persistence schema for experiment manifests, trial records, and experiment artifacts?

The experiment harness introduces new persistence requirements not covered by the existing Evidence Log schema (SPEC-006) or the routing problem schema (SPEC-012). SPEC-012 is the project persistence authority (per ADR-004: single PostgreSQL instance for MVP). Extension of SPEC-012 to cover experiment manifests, trial records, and artifact storage is the expected resolution path. A new schema specification is an alternative if the experiment schema warrants a separate governance lifecycle.

This blocks implementation. The harness cannot be built without a defined persistence layer.

**Classification:** Blocking

---

## OQ-2 (Resolved): Experiment Persistence Store

**Resolved by ADR-004.** ADR-004 commits PostgreSQL as the persistence technology for all durable storage in Project DAEDALUS. Experiment manifests, trial records, and artifact summaries use the existing PostgreSQL instance. No separate data store is introduced for experiment persistence at MVP scope. This is not an open decision.

**Resolution:** Existing PostgreSQL instance. Extension of SPEC-012 to cover experiment manifests, trial records, and artifact storage is the expected resolution path for OQ-1.

---

## OQ-3 (Non-Blocking): Statistical Threshold for Evidentiary Adequacy

What minimum `sample_count` is required for a QualityStats summary to be considered evidentiary?

An experiment with `repetition_count = 1` produces `sample_count = 1` where `stddev_km = null`. The summary is technically complete but provides no variance information. The benchmarking standard recommends multiple runs for stochastic systems. Defining a minimum threshold for "publication-ready" evidence is a Project Owner decision.

**Classification:** Non-Blocking (Project Owner decision)

---

## OQ-4 (Non-Blocking): Advanced Statistical Tests

Should the harness compute hypothesis tests (Mann-Whitney U, Wilcoxon signed-rank) or confidence intervals for cross-solver comparisons?

The benchmarking standard recommends statistical rigor. Advanced tests are excluded from this specification to constrain MVP scope. Adding statistical tests is a non-breaking extension to the experiment summary structure.

**Classification:** Non-Blocking (deferred to post-MVP)

---

## OQ-5 (Non-Blocking): Large-Scale and Parallel Execution

For experiments with large `planned_trial_count` values, sequential trial submission may be impractical. Parallel trial submission, batch scheduling, and distributed execution are not defined in this specification.

**Classification:** Non-Blocking (deferred to post-MVP)

---

## OQ-6 (Non-Blocking): Publication Formats

What format should benchmark summary artifacts take for use in technical publications?

The harness produces structured JSON artifacts. Conversion to publication formats (Markdown tables, LaTeX, rendered HTML benchmark reports) is deferred. This may be addressed by the Report Generator (SPEC-009), SPEC-021, or a future toolchain specification.

**Classification:** Non-Blocking (deferred)

---

## OQ-7 (Non-Blocking): Quantum Hardware Qualification Criteria

SPEC-019 declares `is_provisional = true` until qualification criteria are satisfied (SPEC-019 OQ-5). Experiments that include `quantum_hardware` backend trials should be annotated as provisional until the backend is fully qualified. The process for removing the provisional annotation is not defined in this specification.

**Classification:** Non-Blocking (pending SPEC-019 qualification resolution)

---

## OQ-8 (Non-Blocking): Experiment Cancellation

No mechanism exists to cancel an experiment in `Running` status. An in-progress experiment has no defined abort path; CLI process termination does not cancel submitted jobs, and the harness has no mechanism to stop submitting new trials in response to an external signal. Individual jobs may be cancelled via SPEC-008, but this does not halt the experiment. For hardware experiments with significant queue wait times and monetary cost, the absence of experiment-level cancellation is a production readiness concern.

**Classification:** Non-Blocking for Draft and Proposed. Blocks production enablement for experiments containing `quantum_hardware` backends. The limitation is acceptable for MVP but must be resolved before hardware-backed experiments are exposed beyond controlled developer use.

---

## OQ-9 (Non-Blocking): Hardware Experiment Cost Controls

No cost pre-authorization, acknowledgment step, or cumulative cost cap is defined in this specification. The `qaoa.hardware.estimated_cost` extension_metadata field (SPEC-019) makes per-trial cost estimates available in the trial evidence record, but the harness does not halt execution when cumulative estimated cost exceeds any threshold. Hardware cost controls are a production readiness concern for experiments that include `quantum_hardware` backends.

**Classification:** Non-Blocking (deferred to production readiness phase)

---

# Acceptance Checklist

- [ ] Problem is clearly defined.
- [ ] Domain concept is defined.
- [ ] Scope and responsibility boundary is documented.
- [ ] Requirements are testable.
- [ ] Non-requirements are documented.
- [ ] Assumptions are explicit.
- [ ] Failure modes are defined.
- [ ] Observability requirements exist.
- [ ] Security considerations exist.
- [ ] Documentation updates are identified.
- [ ] Open questions are classified (Blocking / Non-Blocking).
- [ ] OQ-1 (persistence schema) is acknowledged as blocking.
- [ ] Hardware reproducibility classification is explicitly defined.
- [ ] Statistical analysis model excludes advanced tests.
- [ ] Artifact ownership boundaries relative to SPEC-021 and SPEC-022 are defined.
- [ ] Harness execution component is identified as CLI-driven orchestration (SPEC-016 executor, API state authority).
- [ ] SPEC-016 existing `experiment run` command relationship is addressed in Documentation Updates Required.
- [ ] `planned_trial_count` immutability is consistent with partial generation failure handling (`effective_trial_count` in runtime state).
- [ ] Generated Mode instance-sharing invariant is specified: one problem instance per (problem_config_index, repetition_index), shared across all backends (FR-4, FR-8).
- [ ] Trial submission ordering is specified: (problem_config_index asc, repetition_index asc, solver_set_index asc) (FR-4).

---

# Definition of Done

This feature is complete when:

- The experiment manifest schema is defined (pending OQ-1 resolution), persisted, and retrievable by experiment_id.
- The benchmark manifest schema is defined, persisted, and retrievable by benchmark_id.
- Trials are submitted through the existing job API for each (problem_config, solver, repetition_index) combination.
- Trial records are populated with evidence from the Evidence Log after each trial completes.
- Per (problem, solver) aggregate statistics are computed correctly across all repetitions.
- Experiment summary artifacts are produced when experiments reach `Completed` status.
- Benchmark summary artifacts are produced or updated when member experiments complete.
- Hardware retry policy is implemented and verified for `quantum_hardware` backend trials.
- Reproducibility classification is applied to all trials, experiments, and artifacts.
- Seed derivation in generated workload mode satisfies the uniqueness and determinism requirements in FR-8.
- All observability requirements are implemented.
- OQ-1 (persistence schema) is resolved.
- Security review passes.
- Engineering review passes.
- Production readiness review passes.
- Specification status is updated to Verified.

The feature is not complete simply because the harness can submit jobs.
