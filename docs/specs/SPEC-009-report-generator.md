# SPEC-009: Report Generator

## Metadata

**Feature ID:** SPEC-009

**Title:** Report Generator

**Status:** Accepted

**Author:** Darkhorse286

**Created:** 2026-06-13

**Last Updated:** 2026-06-25

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-001, ADR-004, ADR-006, ADR-010, ADR-011, ADR-013

**Related Specs:** SPEC-001, SPEC-002, SPEC-003, SPEC-004, SPEC-005, SPEC-006, SPEC-007, SPEC-008

---

# Problem Statement

The architecture defines the Daedalus Report Generator as the component responsible for producing HTML evidence reports from persisted job execution data. No specification currently defines the invocation interface, report content model, file naming convention, storage structure, report metadata model, failure behavior, idempotency requirements, or observability obligations governing the Report Generator's production of a report.

Without a concrete Report Generator specification:

- The Worker's implementation of SPEC-005 FR-17 is blocked on the Worker-invocable report generation interface that this specification must define. SPEC-005 FR-17 explicitly states: "The report content and format are defined by the Report Generator component specification (not yet authored)."
- The API's implementation of SPEC-008 FR-13 (report retrieval) cannot be completed because the mechanism for locating the physical report file on the report volume is undefined. SPEC-008 FR-13 explicitly defers the file naming convention, storage structure, and file location mechanism to this specification.
- The SPEC-006 FR-9 report metadata record schema is incomplete: SPEC-006 defines minimum fields and defers schema extension to this specification (SPEC-006 OQ-2, non-blocking).
- The evidence reports described in the README.md MVP scope ("HTML evidence reports suitable for technical writing") have no defined content model.
- The report discovery and retrieval contract the API exposes (SPEC-008 FR-12, FR-13) depends on a stable file location contract owned by this specification.

SPEC-009 provides the authoritative definition for the Report Generator component. It closes the gap between the abstract architecture description and the implementable behavior contract for evidence report production.

---

# Business Value

- Unblocks SPEC-005 FR-17 implementation by defining the Worker-invocable report generation interface
- Unblocks SPEC-008 FR-13 implementation by defining the file naming convention, storage structure, and file path resolution mechanism
- Extends SPEC-006 FR-9 report metadata schema with the fields required for report discovery and retrieval
- Produces the HTML evidence reports described in the README.md MVP objective as human-readable artifacts suitable for technical writing and portfolio demonstration
- Closes the evidence loop: every job execution produces a retrievable report explaining the scheduler decision, solver outcome, and quality evaluation result
- Demonstrates production-grade component design: deterministic rendering, idempotent re-generation, explicit failure handling, and clear ownership boundaries

---

# Employer Signaling

- System Design
- Reliability Engineering
- Observability
- Software Engineering

The Report Generator demonstrates disciplined component design: explicit ownership boundaries, deterministic rendering, idempotency under at-least-once semantics, and a content model that makes the project's core thesis ("most routing problems should never touch a quantum computer") legible from a single HTML document without post-processing.

---

# Domain Concept

The Report Generator is a deterministic rendering component. It accepts a complete job evidence record set, renders an HTML document from that record set, writes the document to the report volume, and returns a structured result to the Worker. It does not optimize, evaluate quality, or make scheduling decisions. It translates persisted evidence into a human-readable form.

The report is the evidence record in readable form. Every field it renders comes from the persisted evidence: the scheduler's decision, the solver's outcome, the quality evaluation result, the regret analysis. The report does not recompute anything. It is not an analysis component. It is a rendering component.

The Report Generator is invoked by the Worker after evidence persistence is complete (SPEC-005 FR-17). It is not invoked by the API. The API reads the report file from the report volume and serves it to callers; it does not invoke or interact with the Report Generator directly. The Worker writes the report metadata record to PostgreSQL after the Report Generator returns a successful result; the Report Generator returns the `report_id` and `file_path` to the Worker for that purpose.

A report generation failure does not change the job's terminal state. Evidence has already been persisted before the Report Generator is invoked. The absence of a report does not constitute a missing evidence record. The Worker proceeds to job completion regardless of report generation outcome (SPEC-005 FR-17 failure handling).

---

# Requirements

### FR-1: Responsibility Scope and Component Boundaries

**Description:**
SPEC-009 defines the Report Generator's responsibilities and explicitly allocates the adjacent responsibilities it must not perform.

| Component | Responsibilities at This Boundary | Explicitly Not Responsible For |
|---|---|---|
| **Report Generator** (SPEC-009) | Accepting the invocation from the Worker; rendering the HTML report from the passed evidence record; writing the report file to the report volume; returning the `report_id` and `file_path` to the Worker; defining the file naming convention; defining the storage structure; defining the file path resolution contract for the API | Solver execution; backend selection; quality evaluation; evidence persistence (report metadata record persistence is owned by the Worker, who writes it after receiving the Report Generator's result); job lifecycle management; API serving; workload feature extraction |
| **Worker** (SPEC-005) | Invoking the Report Generator after evidence persistence (SPEC-005 FR-17); writing the report metadata record to PostgreSQL using the `report_id` and `file_path` returned by the Report Generator; emitting the `report.generate` OTel span covering the full Report Generator invocation; handling report generation failure without transitioning the job to `Failed` | Defining report content; defining file naming; defining storage structure |
| **Evidence Log** (SPEC-006) | Defining the minimum report metadata record schema (SPEC-006 FR-9.2); defining upsert semantics for the metadata record | Producing report files; defining report content; defining naming conventions |
| **API** (SPEC-008) | Serving report metadata (SPEC-008 FR-12) and report files (SPEC-008 FR-13) to callers; locating report files via the `file_path` field in the report metadata record | Generating reports; defining naming conventions; defining storage structure; writing to the report volume |

**Acceptance Criteria:**
- No Worker code path determines report file naming, storage structure, or file path construction; those decisions are owned by the Report Generator Specification
- No API code path constructs report file paths from caller-supplied identifiers; file paths are resolved exclusively through the `file_path` field in the report metadata record (FR-7)
- Adding a new section to the HTML report requires no changes to Worker lifecycle behavior, Evidence Log schema, or API endpoints

---

### FR-2: Report Generation Trigger Conditions

**Description:**
The Report Generator is invoked by the Worker when the job has reached the Reporting stage (SPEC-005 FR-2, worker-internal execution stage 8) and all evidence artifacts have been successfully persisted to PostgreSQL (SPEC-005 FR-16).

**Report generation for all solver outcomes:**
The Report Generator is invoked for every job that reaches the Reporting stage, regardless of solver outcome. This includes jobs where the solver outcome was `Infeasible`, `Timeout`, `Cancelled`, `Failed`, or `ContractViolation`. For outcomes without a route plan, report sections that depend on quality evaluation are rendered with appropriate "No solution produced" or "Evaluation not performed" content (FR-5).

**Report generation is not invoked for `Failed` jobs:**
Jobs that fail before the evidence persistence stage (SPEC-005 FR-13 permanent failures) do not reach the Reporting stage. The Report Generator is never invoked for a job in `Failed` terminal state.

**Acceptance Criteria:**
- The Report Generator is invoked for every job that reaches the Reporting stage, regardless of solver outcome
- The Report Generator is never invoked for a job that failed before evidence persistence
- The Report Generator is invoked exactly once per job per execution attempt; re-execution on message redelivery may invoke it again (FR-8 idempotency)

---

### FR-3: Invocation Interface

**Description:**
The Worker invokes the Report Generator as a synchronous in-process call. The Report Generator is a C++ component within the Worker process, consistent with Core being an in-process library per SPEC-005 Assumption 3.

**Invocation inputs:**
The Worker passes the following on every invocation. All required inputs must be non-null; a missing required input is an invocation contract violation (Failure Modes).

| Input | Required | Type | Source | Notes |
|---|---|---|---|---|
| `job_id` | Yes | UUID string | Worker (from job message) | Evidence record set root identifier (SPEC-006 FR-2) |
| `job_record` | Yes | JobRecord | In-memory (loaded by Worker in FR-4, updated through FR-18) | Job lifecycle state, timestamps, `problem_id`, `scheduler_config_id`, `backend_id` (nullable; present when `backend_id` was specified in the job submission per ADR-013; added to `jobs` table by SPEC-012 amendment) (SPEC-006 FR-4.2) |
| `decision_record` | Yes | DecisionRecord | In-memory (from Core invocation FR-5, updated with quality evaluation FR-15) | Scheduler decision including backend selection, predicted values, `actual_outcome`, `hindsight_quality` (SPEC-003 FR-10; SPEC-007 FR-4) |
| `solver_run_record` | Yes | SolverRunRecord | In-memory (composed from SolverResponse, persisted in FR-16) | Solver execution inputs and outputs including outcome, statistics, reproducibility tuple (SPEC-006 FR-6.2) |
| `quality_evaluation_record` | No | QualityEvaluationRecord or null | In-memory (from Core quality evaluation FR-15, persisted in FR-16) | Quality evaluation result (SPEC-007 FR-9); null when quality evaluation was not invoked (no route plan present) |

The Worker passes the in-memory evidence record set — the same artifacts it just persisted — rather than re-reading from PostgreSQL. This avoids a redundant database round-trip and ensures the Report Generator operates on a consistent snapshot.

**Invocation output:**
The Report Generator returns a structured result to the Worker:

| Field | Present | Description |
|---|---|---|
| `generation_status` | Always | `Completed` or `Failed` |
| `report_id` | On success | UUID assigned to this report artifact |
| `file_path` | On success | Absolute path of the written report file on the report volume |
| `generated_at` | On success | Wall-clock UTC timestamp at which the Report Generator completed stage 4 |
| `failure_stage` | On failure | Where in the lifecycle the failure occurred: `invocation_contract_violation`, `rendering_error`, `file_write` |

The Worker uses `report_id`, `file_path`, and `generated_at` from the successful result to populate the report metadata record it persists to PostgreSQL (SPEC-006 FR-9).

**Acceptance Criteria:**
- The Report Generator receives all required inputs on every invocation
- A null `quality_evaluation_record` is not an invocation contract violation; it is the expected input for solver outcomes without a route plan
- The Report Generator does not re-read evidence artifacts from PostgreSQL; it operates exclusively on the inputs passed by the Worker
- The Worker uses the returned `report_id`, `file_path`, and `generated_at` to write the report metadata record to PostgreSQL

---

### FR-4: Report Content Model

**Description:**
The Report Generator renders an HTML document from the evidence record set. The document is organized into named sections. All rendered content derives exclusively from the passed evidence; no section recomputes, infers, or retrieves data from external sources.

**Report sections:**

#### Section 1: Job Summary

Rendered for every job.

| Rendered field | Source |
|---|---|
| Job ID | `job_record.job_id` |
| Problem ID | `job_record.problem_id` |
| Scheduler configuration ID | `job_record.scheduler_config_id` |
| Job status | `job_record.status` (always `Completed` at report generation time) |
| Created at | `job_record.created_at` (ISO 8601 UTC) |
| Completed at | `job_record.completed_at` (ISO 8601 UTC) |
| Solver outcome | `solver_run_record.solver_outcome` |

#### Section 2: Routing Problem Summary

Rendered for every job.

| Rendered field | Source |
|---|---|
| Stop count | `decision_record.workload_features_snapshot.stop_count` |
| Vehicle count | `decision_record.workload_features_snapshot.vehicle_count` |
| Problem size class | Derived from stop count using SPEC-002 size class boundaries |
| Time window constrained | `decision_record.workload_features_snapshot.time_window_constrained` |

The routing problem numeric seed value is not rendered. See Security Considerations.

#### Section 3: Scheduler Decision

Rendered for every job.

| Rendered field | Source |
|---|---|
| Objective mode | `decision_record.objective_mode` |
| Decision status | `decision_record.decision_status` |
| Selection mode | `decision_record.selection_mode` (`policy_selected` or `explicitly_targeted`) |
| Selected backend | `decision_record.selected_backend_id` |
| Requested backend | Rendered only when `selection_mode = explicitly_targeted`. Source: `decision_record.selected_backend_id` when `decision_status = Selected` (targeted backend was eligible and selected); `job_record.backend_id` when `decision_status ≠ Selected` (targeted backend failed eligibility; see targeted-path note below). |
| Predicted latency | `decision_record.predicted_latency` |
| Predicted quality tier | `decision_record.predicted_quality` |
| Predicted cost | `decision_record.predicted_cost` |
| Confidence score | `decision_record.confidence_score`. `0.0` when `selection_mode = explicitly_targeted` and `decision_status = Selected` (scoring bypassed; `selection_mode` distinguishes this from the standard-path single-eligible-backend case); null when `selection_mode = explicitly_targeted` and `decision_status = NoEligibleSolver` (rendered as "N/A" per content rendering rules). |
| Candidate scores | `decision_record.candidate_scores` (all evaluated backends with scores; empty when `selection_mode = explicitly_targeted` because scoring is bypassed in the targeted path) |
| Rejection reasons | `decision_record.rejection_reasons` (per-backend rejection detail; in the targeted path, contains only the specified backend's rejection reason when eligibility checking was performed and failed) |
| Decision timestamp | `decision_record.timestamp` (ISO 8601 UTC) |

**Targeted-path rendering note:** When `selection_mode = explicitly_targeted`, the Scheduler bypassed candidate scoring and directed execution to the specified backend without policy evaluation. `candidate_scores` is empty and `confidence_score` is `0.0` (when `decision_status = Selected`) or null (when `decision_status = NoEligibleSolver`); `selection_mode = explicitly_targeted` distinguishes these values from their standard-path counterparts. The `requested_backend` field is rendered alongside `selection_mode` to allow readers to verify that the evidence was produced by the claimed backend. The `job_record.backend_id` source for `requested_backend` applies when `decision_status ≠ Selected` (i.e., the targeted backend failed eligibility); per SPEC-009 FR-2 and SPEC-005 FR-13, such jobs reach terminal state `Failed` and the Report Generator is not invoked at MVP scope — this data source is documented for completeness per ADR-013.

#### Section 4: Solver Execution

Rendered for every job.

| Rendered field | Source |
|---|---|
| Backend executed | `solver_run_record.backend_id` |
| Solver outcome | `solver_run_record.solver_outcome` |
| Execution duration | `solver_run_record.execution_statistics.execution_duration_ms` when non-null; "Not reported" otherwise |
| Contract version | `solver_run_record.contract_version` |
| Structural validation status | `solver_run_record.structural_validation_status` |
| Failure detail | `solver_run_record.failure_detail` when non-null |

#### Section 5: Quality Evaluation

Rendered for every job. Content varies by evaluation availability (FR-5).

When `quality_evaluation_record` is non-null and `evaluation_status = Completed`:

| Rendered field | Source |
|---|---|
| Time window feasible | `quality_evaluation_record.actual_outcome.time_window_feasible` |
| Time window violation count | `quality_evaluation_record.quality_metrics.time_window_violation_count` |
| Total route distance (km) | `quality_evaluation_record.quality_metrics.total_route_distance_km` |
| Routes used / routes total | `quality_evaluation_record.quality_metrics.routes_used` / `quality_evaluation_record.quality_metrics.routes_total` |
| Hindsight quality (km) | `quality_evaluation_record.hindsight_quality` |
| Quality comparison eligible | `quality_evaluation_record.quality_metrics.quality_comparison_eligible` |
| Per-vehicle distances (km) | `quality_evaluation_record.quality_metrics.per_vehicle_distance_km` |
| Evaluation schema version | `quality_evaluation_record.evaluation_schema_version` |

`violated_stop_ids` is never rendered. See Security Considerations.

When `quality_evaluation_record` is null or `evaluation_status != Completed`, the section renders per FR-5.

#### Section 6: Regret Analysis

Rendered when `quality_evaluation_record` is non-null. Individual fields render as "Not available" when null, per SPEC-007 FR-8 partial result semantics.

| Rendered field | Source |
|---|---|
| Predicted latency (ms) | `quality_evaluation_record.regret_analysis.predicted_latency_ms` |
| Actual execution duration (ms) | `quality_evaluation_record.regret_analysis.actual_execution_duration_ms` |
| Latency delta (ms) | `quality_evaluation_record.regret_analysis.latency_delta_ms` |
| Predicted quality tier | `quality_evaluation_record.regret_analysis.predicted_quality_tier` |
| Actual hindsight quality (km) | `quality_evaluation_record.regret_analysis.actual_hindsight_quality_km` |
| Confidence score | `quality_evaluation_record.regret_analysis.confidence_score` |

When `quality_evaluation_record` is null, this section is omitted.

#### Section 7: Reproducibility

Rendered for every job.

| Rendered field | Source |
|---|---|
| Problem ID | `solver_run_record.problem_id` |
| Backend ID | `solver_run_record.backend_id` |
| Contract version | `solver_run_record.contract_version` |

The `execution_seed` is not rendered. See Security Considerations.

**Content rendering rules:**
- All timestamps are rendered in ISO 8601 UTC format.
- All floating-point distance values are rendered to 4 decimal places.
- All score and weight values are rendered to 4 decimal places.
- Null values in tabular fields are rendered as "N/A".
- The `violated_stop_ids` field from quality metrics must not appear in any rendered section.

**Acceptance Criteria:**
- Every rendered section derives all content from the passed evidence record; no external data access occurs during rendering
- The `execution_seed` value does not appear in any rendered section
- The `violated_stop_ids` list does not appear in any rendered section
- A report rendered from identical evidence inputs is byte-identical (FR-12)
- `selection_mode` is rendered in Section 3 on every invocation
- When `selection_mode = explicitly_targeted`, the `requested_backend` field is rendered in Section 3, sourced from `decision_record.selected_backend_id` when `decision_status = Selected`, or from `job_record.backend_id` when `decision_status ≠ Selected`
- When `selection_mode = policy_selected`, the `requested_backend` field is not rendered in Section 3

---

### FR-5: Report Content for Non-Succeeded Solver Outcomes

**Description:**
The Report Generator produces a complete, meaningful HTML document for every job that reaches the Reporting stage, including those where the solver did not produce a route plan.

**Section rendering by solver outcome:**

| Solver outcome | Route plan present | Quality evaluation record | Section 5 | Section 6 |
|---|---|---|---|---|
| `Succeeded` | Yes | Full (`Completed`) | Full quality metrics | Full regret analysis |
| `Timeout` (with solution) | Yes | Full (`Completed`) | Full quality metrics | Full regret analysis |
| `Cancelled` (with solution) | Yes | Full (`Completed`) | Full quality metrics | Full regret analysis |
| `Timeout` (no solution) | No | Null | "No route plan produced (Timeout). Quality evaluation was not performed." | Omitted |
| `Cancelled` (no solution) | No | Null | "No route plan produced (Cancelled). Quality evaluation was not performed." | Omitted |
| `Infeasible` | No | Null | "No route plan produced (Infeasible). Quality evaluation was not performed." | Omitted |
| `Failed` / `InternalError` | No | Null | "No route plan produced (solver error). Quality evaluation was not performed." | Omitted |
| `ContractViolation` | No | Null | "No route plan produced (contract violation). Quality evaluation was not performed." | Omitted |
| Any outcome | Attempted | Partial (`Failed`) | Partial: input-derived `actual_outcome` fields rendered; simulation-derived fields shown as "Evaluation failed — route simulation could not be completed" | Partial regret analysis (latency fields only) |

**Row precedence:** The final row ("Any outcome | Attempted | Partial (`Failed`)") takes precedence over all solver-outcome-specific rows above it when `quality_evaluation_record.evaluation_status = Failed` (infrastructure failure per SPEC-007 FR-13). For example, a job with `solver_outcome = Succeeded` where quality evaluation experienced an infrastructure failure renders Section 5 with partial content and Section 6 with latency fields only — not full quality metrics or full regret analysis.

Sections 1, 2, 3, 4, and 7 are always rendered in full regardless of solver outcome.

**Acceptance Criteria:**
- The Report Generator produces a complete HTML document for every invocation, regardless of solver outcome
- Section 5 is never omitted entirely; it always renders either full content, partial content, or an appropriate explanatory message
- The Report Generator does not panic or produce an error output from a null `quality_evaluation_record`

---

### FR-6: Report File Naming Convention

**Description:**
The Report Generator defines the authoritative file naming convention for report files on the report volume. This convention is the authoritative definition consumed by SPEC-008 FR-13.

**Convention:**

```
{report_id}.html
```

Where `report_id` is a UUID assigned by the Report Generator at the start of the generation lifecycle (FR-9 stage 2).

**Rationale:**
Using `report_id` as the filename aligns the file with the API's retrieval key (SPEC-008 FR-13 uses `report_id` as the endpoint path parameter). The filename convention allows the API to confirm that the file referenced by the metadata record's `file_path` field is consistent with the `report_id`. Using `report_id` rather than `job_id` avoids overwriting prior execution attempts' files in place; idempotency is handled via `job_id`-keyed upsert on the metadata record (FR-8), not via filename replacement.

**Acceptance Criteria:**
- Every report file is named `{report_id}.html` where `report_id` matches the UUID in the corresponding report metadata record
- No report file uses `job_id`, backend name, timestamp, or any other identifier as part of the filename
- The `.html` extension is lowercase

---

### FR-7: Report Storage Structure

**Description:**
The Report Generator writes all report files to a single flat directory on the report volume. No subdirectory structure is used at MVP scope.

**Storage structure:**

```
{report_volume_root}/
    {report_id_1}.html
    {report_id_2}.html
    ...
```

Where `report_volume_root` is the absolute path of the report volume mount point in the Worker container, provided via configuration at deployment time (OQ-1).

**File path in the report metadata record:**
The Worker writes a `file_path` field to the report metadata record (FR-10) after receiving a successful result from the Report Generator. The value is:

```
file_path = {report_volume_root}/{report_id}.html
```

The API uses this `file_path` to locate the physical file at retrieval time (SPEC-008 FR-13). The API does not construct file paths from `report_id` or any caller-supplied value; it reads `file_path` from the report metadata record.

**Acceptance Criteria:**
- All report files are written to the flat root directory of the report volume; no subdirectories are created
- The `file_path` in the report metadata record is the complete absolute path of the report file
- The API locates report files exclusively by reading the `file_path` field from the report metadata record
- The Report Generator writes only to `{report_volume_root}`; it does not write outside the designated directory

---

### FR-8: Idempotency Behavior

**Description:**
The Report Generator must be idempotent under at-least-once message delivery (SPEC-005 FR-14). A re-invocation for the same `job_id` due to Worker message redelivery must produce a new report file and an updated report metadata record without creating duplicate metadata records.

**Idempotency model:**

1. On re-invocation for the same `job_id`, the Report Generator assigns a new `report_id` UUID.
2. A new report file is written to `{report_volume_root}/{new_report_id}.html`.
3. The Report Generator returns the new `report_id` and `file_path` to the Worker.
4. The Worker updates the report metadata record via `job_id`-keyed upsert (SPEC-006 FR-9.4), replacing the prior `report_id` and `file_path`.
5. The prior report file (from the earlier attempt) remains on the volume; it is not deleted. Volume cleanup is an operational concern (OQ-2).

**Why a new `report_id` on re-execution:**
The `report_id` is the API retrieval key (SPEC-008 FR-13). On re-execution, the Worker may have produced different evidence (a different solver outcome is possible). The report metadata record is updated via `job_id`-keyed upsert to reference the newly generated report. Prior `report_id` values from earlier attempts are not resolvable via the current metadata record after re-execution.

**Acceptance Criteria:**
- A re-invocation for the same `job_id` produces a new `report_id`, a new report file, and an updated metadata record
- No duplicate report metadata records are created for the same `job_id`
- The metadata record for a given `job_id` always references the most recently generated report file after any re-execution
- The Report Generator does not fail if a prior report file with a different `report_id` already exists for the same `job_id`

---

### FR-9: Report Generation Lifecycle

**Description:**
The Report Generator executes the following stages in order on every invocation.

**Stages:**

1. **Input validation** — Verify that all required inputs are non-null. A null required input returns a structured `invocation_contract_violation` failure immediately; no further stages execute.

2. **Report ID assignment** — Generate a new UUID as the `report_id` for this invocation. This UUID determines the filename.

3. **Content rendering** — Render the HTML document from the passed evidence record set (FR-4, FR-5). This stage is purely computational; no file I/O or network access occurs during rendering. A rendering error returns a structured `rendering_error` failure.

4. **File write** — Write the rendered HTML to `{report_volume_root}/{report_id}.html`. This is the only I/O operation in the lifecycle. A file write failure returns a structured `file_write` failure. No metadata record is written if this stage fails.

5. **Return result** — Return a structured success result to the Worker containing `report_id`, `file_path`, and `generated_at` (wall-clock UTC timestamp captured at the moment stage 4 completes). The Worker is responsible for persisting the report metadata record to PostgreSQL using this result.

**Ordering constraint:**
The file write (stage 4) must complete successfully before the Worker writes the metadata record (stage 5 is the Worker's action). A failure at stage 4 means no metadata record is written for this attempt.

**Acceptance Criteria:**
- The lifecycle stages execute in order; the Report Generator never returns a `file_path` for a file that was not written
- A successful invocation always returns `report_id`, `file_path`, and `generated_at`
- A failed invocation always returns a structured failure with `failure_stage` identifying where in the lifecycle the failure occurred

---

### FR-10: Report Metadata Model

**Description:**
The Worker writes the report metadata record to PostgreSQL after receiving a successful result from the Report Generator. This specification extends the minimum schema defined in SPEC-006 FR-9.2 with the additional fields required for report discovery and retrieval.

**Extended report metadata record:**

| Field | Type | Required | Notes |
|---|---|---|---|
| `job_id` | string | Yes | SPEC-006 FR-9.2 minimum; foreign key to job record; `job_id`-keyed upsert key |
| `report_id` | UUID string | Yes | SPEC-006 FR-9.2 minimum; system-assigned by Report Generator; API retrieval key (SPEC-008 FR-13) |
| `report_format` | string | Yes | SPEC-006 FR-9.2 minimum; `"html"` for MVP |
| `generated_at` | timestamp UTC | Yes | SPEC-006 FR-9.2 minimum; wall-clock time at which the Report Generator completed stage 4 |
| `generation_status` | string | Yes | SPEC-006 FR-9.2 minimum; `"Completed"` or `"Failed"` |
| `file_path` | string | When `Completed` | Absolute path of the report file on the report volume; null when `generation_status = Failed` |
| `report_schema_version` | uint32 | Yes | Schema version of the report content model (FR-11); current value: `1` |
| `solver_outcome` | string | Yes | The `solver_outcome` value from `solver_run_record`; denormalized to support metadata queries without joining to the solver run record |

**Persistence ownership:**
The Worker writes the report metadata record. The Report Generator returns `report_id`, `file_path`, and `generated_at` in its successful result; the Worker supplements these with `job_id`, `report_format`, `generation_status`, `report_schema_version`, and `solver_outcome` before writing the record.

**Upsert semantics:**
The report metadata record must support `job_id`-keyed upsert per SPEC-006 FR-9.4. A re-execution for the same `job_id` overwrites the prior record.

**Acceptance Criteria:**
- Every report metadata record includes all fields listed above
- `file_path` is null only when `generation_status = Failed`
- `report_id` is a UUID unique across all report metadata records
- The Worker writes the metadata record using `job_id`-keyed upsert semantics

---

### FR-11: Report Schema Versioning

**Description:**
The report metadata record carries a `report_schema_version` field identifying the content model version used to produce the report. This enables evidence records produced at different lifecycle stages of the system to be correctly interpreted if the report content model evolves.

**Current version:** `1`

**Version 1 defines:**
- The seven sections enumerated in FR-4
- The content rendering rules in FR-4 and FR-5

**Breaking changes (MUST increment `report_schema_version`):**
- Removing a section
- Renaming a section
- Changing which persisted field populates a rendered field
- Changing the interpretation semantics of any rendered field or calculation

**Non-breaking changes (do NOT increment `report_schema_version`):**
- Adding a new section
- Adding new rendered fields within an existing section
- Adjusting decimal precision or timestamp formatting in backward-compatible ways
- Adding fields to the report metadata record (FR-10 extension fields)

**Naming convention and storage structure changes:**
The file naming convention (FR-6) and storage structure (FR-7) are not content model concerns and do not affect `report_schema_version`. Changes to either require a SPEC-009 revision and implementation coordination but do not increment `report_schema_version`. Existing `file_path` values in metadata records remain valid for files written under the prior convention; the Report Generator does not retroactively update or move existing files.

**Breaking change procedure:**
1. `report_schema_version` is incremented.
2. SPEC-009 is revised with an explicit change record documenting the previous and replacement definition.
3. SPEC-006 FR-9 schema is updated if the metadata record is affected.

**Acceptance Criteria:**
- Every report metadata record carries `report_schema_version = 1` (at current version)
- A breaking change to the content model increments `report_schema_version` and requires a SPEC-009 revision with an explicit change record

---

### FR-12: Determinism Expectations

**Description:**
The Report Generator must produce deterministic output. Given identical evidence inputs, the same HTML content must be rendered on every invocation.

**Determinism invariant:**
Given identical `job_record`, `decision_record`, `solver_run_record`, and `quality_evaluation_record`, the Report Generator produces byte-identical HTML content on every invocation.

**Excluded from the determinism invariant:**
- `generated_at` in the metadata record (wall-clock timestamp; it is metadata, not HTML content)
- `report_id` (newly assigned UUID on each invocation)
- `file_path` (depends on the newly assigned `report_id`)

**Requirements:**
1. No stochastic operations in the rendering path (ADR-010).
2. No external state access during rendering: no database reads, no configuration file reads, no network calls.
3. Template expansion and field formatting are pure functions of the passed evidence inputs.
4. Floating-point values are rendered using fixed decimal precision per FR-4 content rendering rules; no locale-sensitive formatting.

**Consequence for re-execution:**
On message redelivery, the Worker re-executes the job and passes the newly produced evidence record to the Report Generator. If the evidence is identical (same solver outcome, same quality evaluation result), the Report Generator produces byte-identical HTML content. If the evidence differs (for example, a different solver outcome on the second attempt), the report reflects the new evidence.

**Acceptance Criteria:**
- Two invocations with identical evidence inputs produce byte-identical HTML content
- The rendering path contains no system time, random state, or external resource access
- Changing `generated_at` between otherwise identical invocations produces no difference in rendered HTML content

---

# Non-Requirements

- The Report Generator does not execute optimization or invoke solver backends.
- The Report Generator does not perform quality evaluation or recompute quality metrics (SPEC-007 responsibility).
- The Report Generator does not own the Evidence Log persistence schema beyond the report metadata record extension in FR-10.
- The Report Generator does not own the report volume; volume creation, mounting, and backup are infrastructure concerns.
- The Report Generator does not serve reports to callers; API serving is owned by SPEC-008 FR-13.
- The Report Generator does not define the API endpoint contract for report discovery or retrieval.
- The Report Generator does not delete or archive existing report files; volume cleanup is an operational concern (OQ-2).
- The Report Generator does not implement multi-format output at MVP scope; only HTML is produced.
- The Report Generator does not generate reports for `Failed` jobs; those jobs do not reach the Reporting stage (SPEC-005 FR-2).
- The Report Generator does not implement pagination or multi-page reports at MVP scope.
- The Report Generator does not produce machine-readable (JSON, XML) report formats at MVP scope.
- The Report Generator does not render geographic coordinate maps or route visualization plots.
- The Report Generator does not include `execution_seed` or `violated_stop_ids` in any output.
- The Report Generator does not write the report metadata record to PostgreSQL; that is a Worker responsibility (FR-3, FR-10).

---

# Assumptions

1. The report volume is mounted and writable by the Worker container before any report generation is attempted. Volume availability is an infrastructure guarantee, not a Report Generator concern.
2. The API container has read access to the same report volume. The shared volume topology is defined in architecture.md.
3. The evidence artifacts passed by the Worker reflect the evidence just persisted to PostgreSQL and are consistent with the database state. The Report Generator does not verify consistency.
4. SPEC-006 (Evidence Log, Accepted) defines a persistence schema and upsert behavior compatible with the report metadata record extension described in FR-10. SPEC-009 extends SPEC-006 FR-9.2 minimum fields; the Evidence Log infrastructure supports upsert operations on the extended schema.
5. SPEC-007 (Core Quality Evaluation, Accepted) guarantees that `quality_evaluation_record` is null only when quality evaluation was not invoked (no route plan present, per SPEC-007 FR-2). The Report Generator does not need to handle a case where evaluation was invoked but the record is missing.
6. The `workload_features_snapshot` in the decision record (SPEC-003 FR-10) contains `stop_count`, `vehicle_count`, and `time_window_constrained` sufficient to render Section 2 without accessing the routing problem record directly.
7. The problem size class boundaries referenced in Section 2 rendering are defined by SPEC-002 and are stable within the current specification version.

---

# Constraints

1. The Report Generator must be implemented in C++ (ADR-001).
2. The file naming convention (FR-6) and storage structure (FR-7) are frozen once this specification is Accepted. Changes require a SPEC-009 revision and implementation coordination but do not increment `report_schema_version`, which tracks the content model only (FR-11).
3. `execution_seed` must not appear in any report output (SPEC-005 Security Considerations; SPEC-006 FR-6.4).
4. `violated_stop_ids` must not appear in any report output (SPEC-007 Security Considerations).
5. The Report Generator must not perform any network calls or external data access during the rendering stage (FR-12 determinism requirement).
6. The Report Generator must write only within `{report_volume_root}`; it must not write to any other filesystem path.

---

# Inputs

| Input | Source | Format | Validation |
|---|---|---|---|
| `job_id` | Worker (from job message) | UUID string | Required; null triggers invocation contract violation |
| `job_record` | Worker (in-memory) | SPEC-006 FR-4.2 fields | Required; null triggers invocation contract violation |
| `decision_record` | Worker (in-memory) | SPEC-003 FR-10 schema | Required; null triggers invocation contract violation |
| `solver_run_record` | Worker (in-memory) | SPEC-006 FR-6.2 schema | Required; null triggers invocation contract violation |
| `quality_evaluation_record` | Worker (in-memory) | SPEC-007 FR-9 schema | Optional; null is valid when no route plan was present |

---

# Outputs

| Output | Consumer | Format | Notes |
|---|---|---|---|
| HTML report file | Report volume (read by API, served per SPEC-008 FR-13) | `text/html` | Written to `{report_volume_root}/{report_id}.html`; deterministic from evidence |
| Structured result | Worker (SPEC-005 FR-17) | `{generation_status, report_id, file_path, generated_at}` on success; `{generation_status, failure_stage}` on failure | Used by Worker to write report metadata record to PostgreSQL |

---

# Failure Modes

### Invocation Contract Violation

**Condition:** A required input (`job_id`, `job_record`, `decision_record`, or `solver_run_record`) is null.
**Behavior:** The Report Generator returns a structured failure with `failure_stage = invocation_contract_violation`. No file is written. The Worker does not write a report metadata record for this failure.
**Worker response (SPEC-005 FR-17):** Worker emits `worker.report.generation.failed` log event, emits `report.generate` span with Unset status, and proceeds to job completion. Job reaches `Completed`.
**Fallback:** Evidence is persisted; no report is produced. Manual re-invocation from the stored evidence record may be possible after investigation.

---

### Report Volume Unavailable or Write Failure

**Condition:** The report volume is not mounted, not writable, or the file write fails (disk full, permissions error).
**Behavior:** The Report Generator returns a structured failure with `failure_stage = file_write`. No metadata record is written (the file write must succeed before the Worker writes the metadata record, per FR-9 ordering).
**Worker response:** Same as invocation contract violation.
**Fallback:** Evidence is persisted; the report file is absent from the volume. No metadata record exists for this job.

**Edge case — partial write:** If the file write is interrupted after beginning, a partially-written file may exist on the volume. The Report Generator does not read or validate existing files before writing. On re-invocation (message redelivery), a new `report_id` is assigned and a fresh write begins. The partial file from the prior attempt is abandoned.

---

### PostgreSQL Unavailable During Metadata Persistence

**Condition:** The Report Generator succeeds (file written, result returned to Worker) but PostgreSQL is unavailable when the Worker attempts to write the report metadata record.
**Behavior:** This is the Worker's persistence failure path, not the Report Generator's. Per SPEC-005 FR-17, a report generation failure (which this is at the Worker level) does not cause the job to transition to `Failed`; the Worker proceeds to job completion. The report file exists on the volume but no metadata record is written to PostgreSQL. The report is not discoverable via the API (SPEC-008 FR-12 returns 404 with no metadata record).
**Fallback:** On message redelivery (idempotent re-execution), the Report Generator generates a new report and the Worker re-attempts metadata persistence. If PostgreSQL is available on re-execution, the metadata record is persisted and the report becomes discoverable.

---

### Rendering Failure

**Condition:** An unexpected error occurs during HTML rendering (unhandled null in an unexpected location, assertion failure in template expansion).
**Behavior:** The Report Generator returns a structured failure with `failure_stage = rendering_error`. No file is written.
**Worker response:** Same as invocation contract violation.
**Fallback:** Evidence is persisted; no report is produced. The rendering error is logged.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer | Yes — Report Generator is a C++ component added to the Worker process; it consumes Core's quality evaluation output (SPEC-007) as a rendering input |
| API Layer | Yes — SPEC-008 FR-13 is unblocked by FR-7 (file naming, storage structure, `file_path` in metadata) |
| Persistence | Yes — extends SPEC-006 FR-9 report metadata schema (FR-10); Worker writes extended metadata record |
| Solver Runtime | None |
| Observability | Yes — `report.generate` OTel span (SPEC-005 FR-19) covers Report Generator invocation; new Prometheus metrics (Observability Requirements) |
| Security | Yes — `execution_seed` and `violated_stop_ids` must not appear in report output |
| Configuration | Yes — `report_volume_root` is a deployment configuration parameter (OQ-1) |
| Deployment | Yes — Report Generator requires write access to the report volume in the Worker container |

**SPEC-006 FR-9 (OQ-2):** The report metadata schema extension is defined in FR-10 of this specification. SPEC-006 OQ-2 is resolved.

**SPEC-008 FR-13:** The file naming convention (FR-6), storage structure (FR-7), and `file_path` field in the metadata record (FR-7, FR-10) resolve the open dependency stated in SPEC-008 Documentation Updates Required. SPEC-008 FR-13 implementation is unblocked.

**SPEC-005 FR-17:** The Worker-invocable interface (FR-3), lifecycle (FR-9), and failure handling are defined. SPEC-005 FR-17's dependency on the Report Generator Specification is satisfied.

---

# Testability

1. **Unit: Determinism invariant** — Two invocations with identical `job_record`, `decision_record`, `solver_run_record`, and `quality_evaluation_record` produce byte-identical HTML content. `generated_at`, `report_id`, and `file_path` are excluded from the HTML content comparison.

2. **Unit: Succeeded outcome — all sections rendered** — A job with `solver_outcome = Succeeded` and a complete quality evaluation record (`evaluation_status = Completed`) produces a report containing all seven sections with non-null content in all rendered fields.

3. **Unit: Infeasible outcome — Section 5 explanation** — A job with `solver_outcome = Infeasible` and `quality_evaluation_record = null` produces a report where Section 5 contains "No route plan produced (Infeasible)" and Section 6 is omitted.

4. **Unit: Partial quality evaluation — Section 5 partial** — A job with `quality_evaluation_record.evaluation_status = Failed` produces a report where Section 5 renders the input-derived `actual_outcome` fields and notes that route simulation failed; it does not render null quality metrics as empty values.

5. **Unit: execution_seed not rendered** — Inspect all rendered HTML output for a complete job execution. Verify that the numeric `execution_seed` value does not appear anywhere in the HTML content or metadata record fields.

6. **Unit: violated_stop_ids not rendered** — Inspect all rendered HTML output for a job with time window violations (`time_window_violation_count > 0`). Verify that specific stop ID values from `violated_stop_ids` do not appear in the HTML content.

7. **Unit: File naming convention** — Given a Report Generator invocation that succeeds, verify the written file is named `{report_id}.html` at `{report_volume_root}/{report_id}.html`.

8. **Unit: Metadata result fields** — Verify that the Report Generator's success result contains `report_id` (a valid UUID), `file_path` (the absolute path of the written file), `generated_at` (a valid UTC timestamp), and `generation_status = Completed`.

9. **Unit: Idempotency — re-invocation produces new report_id** — Given a prior successful invocation for `job_id = X` that produced `report_id = R1`, invoke the Report Generator again for the same `job_id = X`. Verify: a new `report_id = R2` is returned; a new file `{R2}.html` exists on the volume; the file `{R1}.html` also still exists on the volume (not deleted).

10. **Unit: Invocation contract violation — null decision record** — A Report Generator invocation with `decision_record = null` returns a structured failure with `failure_stage = invocation_contract_violation`. No file is written. No result `report_id` or `file_path` is returned.

11. **Unit: Report volume unavailable** — Given a file system that rejects writes, the Report Generator returns a structured failure with `failure_stage = file_write`. No metadata result is returned.

12. **Unit: Section 7 reproducibility fields rendered** — The rendered Section 7 contains `problem_id`, `backend_id`, and `contract_version`. The `execution_seed` value does not appear.

13. **Integration: Full report generation lifecycle** — A complete successful job execution produces a report file at `{report_volume_root}/{report_id}.html` and a report metadata record in PostgreSQL with all fields from FR-10 populated, including non-null `file_path` and `report_id`.

14. **Integration: API report discovery and retrieval** — After report generation, `GET /v1/jobs/{job_id}/report` (SPEC-008 FR-12) returns HTTP 200 with `report_url`. `GET /v1/reports/{report_id}` (SPEC-008 FR-13) returns HTTP 200 with `Content-Type: text/html` and the full HTML content.

15. **Integration: report.generate span** — A complete job execution produces a `report.generate` OTel span (SPEC-005 FR-19) as a child of `job.consume`. Span status is OK on success; Unset on report generation failure. Span attributes include `job_id` and `outcome`.

16. **Property: No external access during rendering** — Instrument the rendering path (FR-9 stage 3). Verify that no PostgreSQL queries, network calls, or file system reads occur between input validation and file write.

17. **Integration: Idempotency — metadata record upsert** — Given message redelivery for `job_id = X` after a prior successful report generation, verify that after re-execution exactly one report metadata record exists for `job_id = X`, its `report_id` matches the most recently generated file, and no duplicate records were created.

18. **Unit: Standard-path report — Section 3 `selection_mode`** — A job with `decision_record.selection_mode = policy_selected` produces a report where Section 3 renders `selection_mode = policy_selected`. The `requested_backend` field is absent from Section 3.

19. **Unit: Targeted-path report — Section 3 targeted backend** — A job with `decision_record.selection_mode = explicitly_targeted` and `decision_status = Selected` produces a report where Section 3 renders `selection_mode = explicitly_targeted` and `requested_backend` sourced from `decision_record.selected_backend_id`. `candidate_scores` renders as empty and `confidence_score` renders as `0.0000`.

---

# Observability Requirements

## Operational Questions

The Report Generator's observability must answer the following:

1. Was a report successfully generated for a given job?
2. How long did report generation take, and what fraction of total job latency does it represent?
3. How often does report generation fail, and at which lifecycle stage?
4. Is the report volume writable?
5. Are report metadata records successfully persisted to PostgreSQL after successful file writes?

## Structured Log Events

The Worker emits the following structured log events for Report Generator outcomes (defined in SPEC-005 FR-19; reproduced here for traceability):

| Event name | Condition | Required fields |
|---|---|---|
| `worker.report.generation.failed` | Report generation fails for any reason | `job_id`, `error_type`, `failure_stage` (`invocation_contract_violation`, `rendering_error`, `file_write`) |

The Worker also emits the `report.generate` OTel span per SPEC-005 FR-19, with `job_id` and `outcome` attributes, covering the full Report Generator invocation from call to return.

The Report Generator does not emit log events directly. All report generation telemetry is owned by the Worker.

## Prometheus Metrics

The following Prometheus metrics observe report generation outcomes. These extend the metric set defined in SPEC-005:

| Metric name | Type | Labels | Description |
|---|---|---|---|
| `daedalus_report_generation_total` | Counter | `status` (`success`, `failed`), `failure_stage` (`invocation_contract_violation`, `rendering_error`, `file_write`, `none`) | Total report generation attempts by outcome and failure stage |
| `daedalus_report_generation_duration_seconds` | Histogram | `status` (`success`, `failed`) | End-to-end report generation duration from Worker invocation to result return |

## OTel Trace Spans

The `report.generate` span is owned and emitted by the Worker (SPEC-005 FR-19). It covers the complete Report Generator invocation. Per SPEC-005 FR-19 span status rules, `report.generate` span status is Unset on failure — this preserves the parent `job.consume` span's OK status for a `Completed` job that had a report generation failure.

The Report Generator does not emit its own OTel spans. The `report.generate` Worker span is the observable unit at this boundary.

---

# Security Considerations

**`execution_seed` isolation:**
The `execution_seed` must not appear in any report output, including rendered HTML content. This restriction is established by SPEC-005 Security Considerations and SPEC-006 FR-6.4. Because `execution_seed = RoutingProblem.seed` (ADR-010 ODR-4), the numeric problem seed value must also not be rendered, as it would effectively disclose the execution seed. The `problem_id` UUID is the safe reproducibility identifier for Section 7.

**`violated_stop_ids` restriction:**
The `violated_stop_ids` field from `quality_evaluation_record.quality_metrics` must not appear in any report output. The rationale is established in SPEC-007 Security Considerations: in non-synthetic deployments, stop IDs may map to customer premises, and their inclusion in violation reports could constitute location-identifying exposure. The time window violation count and `time_window_feasible` flag are sufficient for evidence reporting without exposing individual stop identifiers.

**Report file path construction:**
The report file path is constructed from the deployment-configured `report_volume_root` and a system-generated `report_id` UUID. No caller-supplied value contributes to the file path. This eliminates path traversal risk in the report generation path. The API's file location mechanism (SPEC-008 FR-13: reads `file_path` from metadata record rather than constructing from request parameters) eliminates path traversal risk on the serving path.

**Report content data sensitivity:**
Report content includes stop counts and vehicle counts, which are safe aggregate statistics. Geographic coordinates, raw stop arrays, and individual address data are not rendered, consistent with SPEC-001 Security Considerations and SPEC-005 Security Considerations for log safety.

**Volume write isolation:**
The Report Generator writes only to `{report_volume_root}`. It must not construct or follow paths outside this directory. The `report_id` UUID is system-generated and contains no directory traversal sequences.

---

# Performance Considerations

**Rendering cost:**
HTML rendering is purely computational with O(N) cost in the number of sections and rendered fields. For seven sections and the field volumes defined in FR-4, rendering cost is expected to be negligible relative to solver execution and quality evaluation time. This must be measured during implementation.

**File write latency:**
Report files are bounded in size by the evidence record content. Estimated file size is in the range of 5-50 KB for MVP-scale routing problems. Write latency on local Docker Compose volume mounts is expected to be sub-millisecond. This must be measured.

**Total job latency contribution:**
The `report.generate` span captures the complete Report Generator invocation duration including file write and the Worker's subsequent metadata persistence. SPEC-005 Performance Considerations identifies report generation as a potential bottleneck and notes that asynchronous report generation is a future architectural option if latency becomes significant. This is out of scope for SPEC-009.

---

# Documentation Updates Required

- **SPEC-006 FR-9 (Accepted):** The report metadata schema extension defined in FR-10 resolves SPEC-006 OQ-2. SPEC-006 FR-9.3 states that the Report Generator Specification "may extend the report metadata record schema with additional fields." The extended fields (`file_path`, `report_schema_version`, `solver_outcome`) defined in FR-10 are that extension. SPEC-006 FR-9.3 should be updated to reference SPEC-009 FR-10 as the authoritative schema extension.

- **SPEC-008 FR-13 (Accepted):** The file naming convention (SPEC-009 FR-6), storage structure (SPEC-009 FR-7), and `file_path` field in the metadata record (SPEC-009 FR-7, FR-10) resolve the open dependency stated in SPEC-008 Documentation Updates Required: "(1) the file naming convention and storage structure on the report volume; and (2) the mechanism by which the API locates the physical file." The mechanism is explicit `file_path` in the report metadata record. SPEC-008 FR-13 implementation is unblocked.

- **SPEC-005 FR-17 (Accepted):** The Worker-invocable interface (SPEC-009 FR-3), report generation lifecycle (SPEC-009 FR-9), and failure handling are defined. SPEC-005 FR-17's note that "The report content and format are defined by the Report Generator component specification (not yet authored)" is satisfied. SPEC-005 FR-17 implementation is unblocked. SPEC-005 FR-17 must also be revised to update its invocation input list to explicitly include `job_record` as a required Report Generator invocation input, consistent with SPEC-009 FR-3. The current SPEC-005 FR-17 text lists "job_id, decision record, solver run record, quality evaluation result" and omits `job_record`, which is required for rendering Section 1 content (`job_record.created_at`, `job_record.completed_at`, `job_record.status`, `job_record.scheduler_config_id`). The invocation contract is authoritative in SPEC-009 FR-3; SPEC-005 FR-17 must align with it.

- **SPEC-002 (pending):** Section 2 of the report (FR-4) derives the "Problem size class" label from stop count using SPEC-002 size class boundaries (SPEC-009 Assumption 7). When SPEC-002 is accepted, or if its size class boundaries subsequently change, the rendered output of Section 2 changes. Changes to SPEC-002 size class boundaries must be evaluated for impact on `report_schema_version` at the time of the SPEC-002 revision. SPEC-002 must be stable before Report Generator implementation of Section 2 begins.

- **docs/architecture.md:** The System Components section lists "Daedalus Report Generator" as a component. SPEC-009 is the authoritative specification for that component. No changes to architecture.md are required. The `report volume` in the container topology diagram is consistent with SPEC-009 FR-7.

---

# Open Questions

### OQ-1: Report Volume Root Configuration Mechanism

**Question:** How is `report_volume_root` provided to the Report Generator at runtime?

**Why it matters:** The `file_path` persisted in the report metadata record (FR-7, FR-10) must be the absolute path of the report file on the volume. The Report Generator must know the mount point to construct this path. Hardcoding creates deployment inflexibility.

**Recommendation:** Environment variable injected at Worker container startup, consistent with the credential injection pattern used by SPEC-008 Security Considerations for PostgreSQL and RabbitMQ credentials. The environment variable name is an implementation planning concern.

**Blocking:** Blocking for FR-7 and FR-9 implementation. Not blocking for Draft status.

---

### OQ-2: Abandoned Report File Cleanup

**Question:** Over time, message redeliveries and re-executions accumulate abandoned report files on the volume (the `{prior_report_id}.html` files from prior attempts). What is the cleanup strategy?

**Why it matters:** Without cleanup, the volume grows without bound in deployments with frequent redeliveries.

**Recommendation:** Operational concern, out of scope for this specification. A periodic cleanup task that removes volume files older than the evidence retention period (90 days, SPEC-006 FR-15.1) and not referenced by any active report metadata record is a reasonable approach. Deferred to operations planning.

**Blocking:** Not blocking for Draft status or implementation. Volume growth at MVP scale is bounded and manageable.

---

### OQ-3: Report Template Technology

**Question:** What HTML template technology is used for rendering?

**Why it matters:** The template technology determines implementation complexity and the effort required to add or restructure report sections.

**Owner:** Implementation planning concern. SPEC-009 defines what is rendered (FR-4, FR-5) and the determinism requirement (FR-12); it does not define the internal rendering mechanism.

**Blocking:** Not blocking for Draft or Accepted status.

---

# Acceptance Checklist

- [x] Problem is clearly defined
- [x] Domain concept is defined
- [x] Responsibility scope and component boundaries are defined (FR-1)
- [x] Report generation trigger conditions are defined (FR-2)
- [x] Invocation interface is defined (FR-3)
- [x] Report content model is defined (FR-4)
- [x] Report content for non-Succeeded outcomes is defined (FR-5)
- [x] Report file naming convention is defined (FR-6)
- [x] Report storage structure is defined (FR-7)
- [x] Idempotency behavior is defined (FR-8)
- [x] Report generation lifecycle is defined (FR-9)
- [x] Report metadata model is defined (FR-10)
- [x] Report schema versioning is defined (FR-11)
- [x] Determinism expectations are defined (FR-12)
- [x] Non-requirements are documented
- [x] Assumptions are explicit
- [x] Failure modes are defined
- [x] Observability requirements exist
- [x] Security considerations exist
- [x] Performance considerations exist
- [x] Documentation updates are identified
- [ ] OQ-1 resolved — report volume root configuration mechanism (blocking for implementation; not blocking for Draft)
- [ ] OQ-2 resolved — abandoned report file cleanup strategy (non-blocking; operational concern)
- [ ] OQ-3 resolved — report template technology (non-blocking; implementation planning)

---

# Definition of Done

This feature is complete when:

- All functional requirements (FR-1 through FR-12) are implemented and acceptance criteria pass
- OQ-1 (report volume root configuration mechanism) is resolved and incorporated in FR-7 and FR-9
- All test contracts defined in the Testability section pass
- The `report.generate` span is emitted and verifiable in the OpenTelemetry Collector on every job execution
- A complete HTML evidence report is generated for a `Succeeded` job and is retrievable via `GET /v1/reports/{report_id}` (SPEC-008 FR-13)
- A targeted-path evidence report (from a job with `decision_record.selection_mode = explicitly_targeted`) correctly renders `selection_mode = explicitly_targeted` and `requested_backend` in Section 3, with empty `candidate_scores` and `confidence_score = 0.0`
- Report metadata records are persisted with all required fields from FR-10, including non-null `file_path`
- SPEC-006 FR-9 is updated to reference SPEC-009 FR-10 as the authoritative report metadata schema extension
- Engineering review passes
- Specification status is updated to Verified

The feature is not complete simply because the Report Generator writes an HTML file without errors.
