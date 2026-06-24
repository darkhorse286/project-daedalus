# Feature Specification

## Metadata

**Feature ID:** SPEC-016

**Title:** Daedalus Command Line Interface

**Status:** Accepted

**Author:** Darkhorse286

**Created:** 2026-06-20

**Last Updated:** 2026-06-23 (Accepted: Engineering Review completed; Architecture Review completed; Acceptance Review completed; NB-A JSON error schema prose corrected; NB-B FR-5 flag inheritance clarified; NB-C --open stdout interaction specified. Amended: ADR-012 / SPEC-020 experiment orchestration requirements applied — FR-12 replaced with SPEC-020-compliant orchestration model; FR-17 through FR-22 added; benchmark command group added; long-running session exception documented; OQ-3 superseded. Architecture Review revisions applied: ARCH-001 Outputs table FR references corrected; ARCH-002 FR-17 benchmark manifest schema expanded to full SPEC-008 FR-19 contract; ARCH-003 FR-25 idempotency claim corrected and TRIAL_ALREADY_SUBMITTED recovery defined; ARCH-004 submission phase error handling specified; ARCH-005 Architectural Impact table updated. Acceptance Review (amendment) revisions applied: AC-SF-001 benchmark submit testability entries added; AC-SF-002 FR-12 result manifest AC and testability entry updated to include problem_id and evidence_status; AC-SF-003 experiment run JSON output mode testability entries added; AC-SF-004 submission phase error handling testability entries added; AC-SF-005 evidence_status = Error non-fatal testability entry added; AC-NTH-001 experiment summary ACs differentiate EXPERIMENT_NOT_FOUND from EXPERIMENT_SUMMARY_NOT_FOUND; AC-NTH-002 evidence_status added to cli.experiment.evidence_collect log event. OQ-2 resolved — CLI implementation language is C++; ADR-001 and ADR-010 added to Related ADRs.)

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-001, ADR-002, ADR-003, ADR-004, ADR-006, ADR-009, ADR-010, ADR-012

**Related Specs:** SPEC-001, SPEC-002, SPEC-003, SPEC-005, SPEC-008, SPEC-009, SPEC-012, SPEC-020

---

# Problem Statement

Project DAEDALUS defines a runtime that submits routing problems, executes optimization, and produces evidence reports. No developer-facing interface currently exists for driving this system. Absent a CLI:

- Developers cannot submit routing problems without authoring raw HTTP requests.
- The synthetic workload generator (SPEC-002) has no execution interface.
- Automation scripts cannot drive the system, poll for completion, or retrieve reports in a structured way.
- Experiments — submitting multiple problems with varied configurations and comparing their outcomes — have no defined launch interface.
- The project thesis ("most routing problems should never touch a quantum computer") cannot be demonstrated from a developer workstation without manual API interaction.

The Daedalus CLI is the developer-facing and automation-facing interface for driving the full Project DAEDALUS runtime.

---

# Business Value

- Provides a single command that generates a synthetic routing problem, submits it to the runtime, and returns a report — demonstrating the full system in a terminal session
- Enables reproducible experiment runs that generate the evidence corpus for the project thesis
- Supports automated benchmarking pipelines that can drive the runtime from CI without manual HTTP interaction
- Makes the system demonstrable from first principles: a developer can clone the repository, run `docker compose up`, and immediately drive the runtime via the CLI
- Signals practical systems integration and production-grade developer tooling as portfolio capabilities

---

# Employer Signaling

- System Design
- Distributed Systems
- Reliability Engineering
- Performance Engineering
- Software Engineering

---

# Scope and Responsibility Boundary

The CLI is the primary interface for developers, automation, and benchmark scripts interacting with Project DAEDALUS. It wraps the HTTP API defined in SPEC-008 and hosts the SPEC-002 synthetic workload generator.

| Responsibility | Owner |
|---|---|
| HTTP API contract for job submission, status, cancellation, configuration, and report access | SPEC-008 |
| Routing problem domain model and validation | SPEC-001 |
| Synthetic workload generation | SPEC-002 |
| Job execution, solver selection, quality evaluation | SPEC-005, Worker |
| Report generation and content | SPEC-009 |
| Experiment and benchmark lifecycle: state, trial records, artifact production | SPEC-008, SPEC-012 |
| Experiment execution model, trial set, and artifact requirements | SPEC-020 |
| Wrapping SPEC-008 API calls for developer and automation use | **SPEC-016 (this spec)** |
| Hosting the SPEC-002 workload generator (OQ-2 resolution) | **SPEC-016 (this spec)** |
| Structured output formats for human and machine consumers | **SPEC-016 (this spec)** |
| Experiment orchestration executor: trial submission loop, polling, evidence collection trigger | **SPEC-016 (this spec)** (ADR-012 Decision 1) |

The CLI does not invoke solver backends directly, does not run workers, and does not read from PostgreSQL. All system interaction is through the Daedalus API.

---

# Requirements

## FR-1: Command Structure

**Description:** The CLI is a single binary named `daedalus`. Commands are organized in a two-level hierarchy: `daedalus <group> <command> [options]`. The command groups are:

| Group | Purpose |
|---|---|
| `problem` | Routing problem authoring, generation, and submission |
| `job` | Job lifecycle management: status, polling, cancellation |
| `report` | Report discovery and retrieval |
| `config` | Scheduler configuration management |
| `experiment` | Experiment orchestration, state retrieval, and summary access |
| `benchmark` | Benchmark manifest submission and summary retrieval |

**Full command inventory:**

| Command | Purpose |
|---|---|
| `daedalus problem submit <file>` | Submit a routing problem JSON file to the API |
| `daedalus problem generate <file>` | Generate a synthetic routing problem from a generation config file |
| `daedalus problem run <file>` | Generate and submit a routing problem in a single step |
| `daedalus job status <job_id>` | Print the current status of a job |
| `daedalus job wait <job_id>` | Poll until a job reaches a terminal state |
| `daedalus job cancel <job_id>` | Submit a cancellation request for a job |
| `daedalus job list` | List recent jobs |
| `daedalus report show <job_id>` | Retrieve and display or save the evidence report for a job |
| `daedalus config create <file>` | Create a scheduler configuration from a JSON file |
| `daedalus config list` | List all stored scheduler configurations |
| `daedalus config get <config_id>` | Retrieve a specific scheduler configuration |
| `daedalus config default` | Print the default scheduler configuration |
| `daedalus experiment run <manifest>` | Submit a SPEC-020 experiment manifest and drive the full trial orchestration loop |
| `daedalus experiment status <experiment_id>` | Print the current status and trial counts for an experiment |
| `daedalus experiment trials <experiment_id>` | List per-trial status for an experiment |
| `daedalus experiment summary <experiment_id>` | Retrieve the experiment summary artifact |
| `daedalus benchmark submit <file>` | Submit a benchmark manifest to create a named benchmark container |
| `daedalus benchmark summary <benchmark_id>` | Retrieve the benchmark summary artifact |

**Global flags** (available on all commands):

| Flag | Default | Description |
|---|---|---|
| `--api-url <url>` | From config discovery (FR-2) | Base URL of the Daedalus API |
| `--output-format <format>` | `text` | Output format: `text` or `json` |
| `--quiet` | false | Suppress non-essential output; print only primary result |
| `--no-color` | false | Disable terminal color codes |
| `--help`, `-h` | — | Print command help |

**Acceptance Criteria:**
- `daedalus --help` prints the command inventory.
- `daedalus <group> --help` prints the group's command inventory.
- `daedalus <group> <command> --help` prints usage for that command.
- An unrecognized command group or command name prints an error and exits with code 1 (FR-13).
- Global flags are accepted by all commands and take precedence over environment variables (FR-2).

---

## FR-2: Configuration Discovery

**Description:** The CLI must locate the Daedalus API base URL before making any API call. It discovers configuration from sources in the following priority order (highest to lowest):

1. **CLI flag:** `--api-url <url>` — explicit override; highest precedence.
2. **Environment variable:** `DAEDALUS_API_URL` — set in environment.
3. **Project config file:** `daedalus.yaml` or `daedalus.json` in the current working directory or any parent directory up to the filesystem root.
4. **User config file:** `~/.daedalus/config.yaml` or `~/.daedalus/config.json`.
5. **Default:** `http://localhost:5000` — assumed Docker Compose local development environment.

**Config file format:**

```yaml
api_url: http://localhost:5000
```

```json
{
  "api_url": "http://localhost:5000"
}
```

The config file supports only `api_url` at MVP scope.

**Configuration display:** `daedalus config default` prints the resolved `api_url` and the source that produced it, in the active output format.

**Acceptance Criteria:**
- When `--api-url` is provided, all other sources are ignored.
- When `DAEDALUS_API_URL` is set and `--api-url` is absent, the environment variable is used.
- When neither flag nor environment variable is set, the CLI searches for a config file starting from the current directory.
- If no config file is found and no environment variable is set, the CLI uses `http://localhost:5000`.
- If a config file is found but `api_url` is absent or empty, the CLI falls through to the next priority level.
- The CLI never prompts for configuration interactively; it fails with a structured error if the API is unreachable.

---

## FR-3: Problem Submit Command

**Description:** `daedalus problem submit <file>` reads a SPEC-001-conforming routing problem JSON file from `<file>` or from stdin when `<file>` is `-`, and submits it to `POST /v1/jobs` (SPEC-008 FR-2).

**Flags:**

| Flag | Description |
|---|---|
| `--scheduler-config <config_id>` | Associate the job with this scheduler configuration. When absent, the API default is used. |
| `--wait` | Poll for job completion before exiting (invokes job wait behavior per FR-7). |
| `--wait-timeout <seconds>` | Maximum seconds to wait before exiting. Default: 300. Only meaningful with `--wait`. |
| `--save-manifest <file>` | Write the submission response body to a file. Useful for automation that needs to capture `job_id` and `problem_id`. |

**Normal behavior:**
1. Read and parse the routing problem JSON from `<file>` or stdin.
2. Submit to `POST /v1/jobs` with the resolved `scheduler_config_id` (if provided).
3. On HTTP 202: print the `job_id`, `problem_id`, `status_url`, and current `status` in the active output format.
4. If `--wait` is provided: enter the job wait loop (FR-7) using the returned `job_id`.
5. If `--save-manifest` is provided: write the full submission response JSON to the named file.

**Text output format (no `--wait`):**
```
Submitted.
  job_id:     a1b2c3d4-...
  problem_id: e5f6g7h8-...
  status:     Pending
  status_url: http://localhost:5000/v1/jobs/a1b2c3d4-...
```

**JSON output format (no `--wait`):**
```json
{
  "job_id": "a1b2c3d4-...",
  "problem_id": "e5f6g7h8-...",
  "status": "Pending",
  "status_url": "http://localhost:5000/v1/jobs/a1b2c3d4-..."
}
```

**Acceptance Criteria:**
- A valid routing problem JSON file produces a submission to the API and prints the response.
- A missing required field in the routing problem JSON (detected by the API) prints the API validation errors and exits with code 2 (FR-13).
- `--scheduler-config` with an invalid UUID format is rejected locally before the API call, with a structured error.
- `--wait` causes the CLI to enter the polling loop (FR-7) after receiving HTTP 202.
- When `<file>` is `-`, the CLI reads from stdin.
- A non-existent `<file>` exits with code 1 before any API call.
- `--save-manifest` writes the full HTTP 202 response body as JSON to the named path; an error writing the file is logged but does not change the exit code.

---

## FR-4: Workload Generation Command

**Description:** `daedalus problem generate <config_file>` invokes the SPEC-002 synthetic workload generator using the generation configuration in `<config_file>`, and produces a routing problem document and a generation manifest.

This command hosts the SPEC-002 generator within the CLI process (resolving SPEC-002 OQ-2: generator process location). The generator is invoked as an in-process library call. No API call is made by this command.

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--output <file>` | stdout | Write the routing problem document (JSON) to `<file>`. When `-`, write to stdout. |
| `--manifest <file>` | `<output>.manifest.json` | Write the generation manifest (JSON) to `<file>`. When absent, the manifest filename is derived from `--output` by appending `.manifest.json`. When `--output` is `-`, the manifest is written to stderr unless `--manifest <file>` is explicitly provided. |
| `--seed <uint64>` | From config file | Override the `seed` value in the generation configuration file. |

**Generation config file format:** A JSON document conforming to the SPEC-002 FR-2 generation configuration contract. All required parameters must be present, except `seed`, which may be supplied via `--seed`.

**Normal behavior:**
1. Read and parse the generation configuration from `<config_file>`.
2. Apply `--seed` override if provided.
3. Invoke the SPEC-002 generator.
4. On success: write the routing problem document to `--output`; write the generation manifest to `--manifest`.
5. Print a generation summary in the active output format.

**Text output format (to stderr when `--output` is stdout):**
```
Generated.
  seed:              42
  scenario_type:     capacity-only
  difficulty_tier:   Easy
  stop_count:        20
  time_windowed:     0
  total_demand:      87
  total_capacity:    100
```

**JSON output format (to stderr when `--output` is stdout):**
The generation manifest JSON contains all of these fields per SPEC-002 FR-8.

**Acceptance Criteria:**
- A valid generation config file produces a routing problem document and generation manifest.
- A generation config file with a missing required field is rejected before generation begins, with a structured error identifying the offending field (SPEC-002 FR-2 structural validation).
- `--seed` overrides the `seed` field in the config file; the seed in the output document matches the override value.
- A capacity infeasibility failure (SPEC-002 FR-4 construction self-check) prints the failure reason and exits with code 3 (FR-13).
- A time window achievability failure (SPEC-002 FR-5 construction self-check) prints the failure reason with the offending stop index and exits with code 3.
- When `--output` is `-`, the routing problem document is written to stdout and the manifest is written to either stderr or the `--manifest` path.
- The generated routing problem document is valid JSON that passes SPEC-001 domain validation when submitted to the API.

---

## FR-5: Combined Generate-and-Submit Command

**Description:** `daedalus problem run <config_file>` combines FR-4 (generate) and FR-3 (submit) in a single command. The caller provides a generation configuration file; the CLI generates the routing problem and immediately submits it to the API.

**Flags:** All flags from FR-3 (submit) apply. All flags from FR-4 (generate) apply except `--manifest`, which is superseded by `--save-generation-manifest` in this combined command (see below). `--save-manifest` retains its FR-3 meaning (write the API submission response body). Additionally:

| Flag | Default | Description |
|---|---|---|
| `--save-problem <file>` | Not saved | Write the generated routing problem document to a file before submission. When absent, the document exists only in memory during the generation-submission pipeline. |
| `--save-generation-manifest <file>` | Not saved | Write the SPEC-002 generation manifest to a file. Equivalent to `--manifest` in FR-4 but renamed to avoid collision with `--save-manifest` (submission response) when both flags apply in this command. |

**Normal behavior:**
1. Read and parse the generation configuration from `<config_file>`.
2. Apply `--seed` override if provided.
3. Invoke the SPEC-002 generator.
4. If generation succeeds: optionally save the problem document and generation manifest per `--save-problem` and `--save-generation-manifest`.
5. Submit the routing problem document to `POST /v1/jobs` with the resolved `scheduler_config_id`.
6. On HTTP 202: print the combined generation summary and submission result.
7. If `--wait` is provided: enter the job wait loop (FR-7).

**Acceptance Criteria:**
- A valid generation config file produces a successful submission and prints the `job_id`, `problem_id`, and generation summary.
- A generation failure (capacity infeasibility, achievability failure) exits before any API call is made, with exit code 3.
- A submission validation failure (API returns HTTP 400) exits with code 2; the generation is not retried.
- `--save-problem` and `--save-generation-manifest` write the respective generation artifacts independently of whether the submission succeeds.
- `--save-manifest` (submission response) writes the HTTP 202 response body independently of whether `--save-problem` or `--save-generation-manifest` are present.

---

## FR-6: Job Status Command

**Description:** `daedalus job status <job_id>` retrieves the current status of a job via `GET /v1/jobs/{job_id}` (SPEC-008 FR-8) and prints it in the active output format.

**Text output format:**
```
Job: a1b2c3d4-...
  status:         Completed
  solver_outcome: Succeeded
  created_at:     2026-06-20T10:00:00Z
  completed_at:   2026-06-20T10:00:04Z
  report_available: true
```

**JSON output format:** The full SPEC-008 FR-8 response body as JSON.

**Acceptance Criteria:**
- A valid `job_id` for an existing job prints the current status.
- A `job_id` that does not exist prints an error and exits with code 1.
- When `status = Completed`, `solver_outcome` is always printed.
- When `status = Failed`, `failure_reason` is always printed.
- When `report_available = true`, the text format includes a hint showing the `daedalus report show <job_id>` command.

---

## FR-7: Job Wait Command

**Description:** `daedalus job wait <job_id>` polls `GET /v1/jobs/{job_id}` at a configurable interval until the job reaches a terminal state (`Completed` or `Failed`), or until a timeout expires.

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--poll-interval <seconds>` | 2 | Seconds between status polls. |
| `--timeout <seconds>` | 300 | Maximum seconds to wait before exiting with a timeout error. |
| `--quiet` | false | Suppress intermediate status output; print only the terminal result. |

**Normal behavior:**
1. Poll `GET /v1/jobs/{job_id}` at `--poll-interval` intervals.
2. On each poll: print the current status in the active output format (unless `--quiet`).
3. When `status = Completed` or `status = Failed`: print the terminal status and exit.
4. If `--timeout` seconds elapse before a terminal state is reached: print a timeout error and exit with code 4 (FR-13).

**Text output format (in-progress poll):**
```
[10:00:01] status: Processing
[10:00:03] status: Processing
[10:00:05] status: Completed  solver_outcome: Succeeded
```

**Exit on completion:** Exit code 0 when `status = Completed`, exit code 5 when `status = Failed` (FR-13). These distinct codes allow automation to branch on solver-level vs. system-level failure.

**Acceptance Criteria:**
- The command polls until a terminal state is reached or timeout expires.
- `--poll-interval` governs the inter-poll delay. Values less than 1 second are rejected with a usage error before polling begins.
- A timeout exit (code 4) does not cancel the job. The job continues executing; the CLI simply stops waiting.
- The terminal status printout includes `solver_outcome` (for `Completed`) or `failure_reason` (for `Failed`).
- `--quiet` suppresses all intermediate polls and prints only the terminal state.
- When invoked internally by `problem submit --wait` or `problem run --wait`, the same polling behavior applies.

---

## FR-8: Job Cancel Command

**Description:** `daedalus job cancel <job_id>` submits a cancellation request via `POST /v1/jobs/{job_id}/cancel` (SPEC-008 FR-11) and prints the result.

**Normal behavior:**
1. Issue `POST /v1/jobs/{job_id}/cancel`.
2. On HTTP 202: print the cancellation acknowledgement. Inform the caller that cancellation is not guaranteed.
3. On HTTP 409: print the conflict message indicating the job is already terminal.
4. On HTTP 404: print a not-found error.

**Text output format (202):**
```
Cancellation requested.
  job_id:                 a1b2c3d4-...
  status at request time: Processing
  note: cancellation is not guaranteed; the job may complete before the Worker observes the flag.
```

**Acceptance Criteria:**
- A cancellation request for a non-terminal job prints the 202 acknowledgement and exits with code 0.
- A cancellation request for a terminal job prints the 409 conflict and exits with code 1.
- A cancellation request for an unknown job_id prints the 404 error and exits with code 1.
- The CLI does not poll for the job to become `Cancelled`; it acknowledges the intake and exits.

---

## FR-9: Job List Command

**Description:** `daedalus job list` retrieves a list of recent jobs from `GET /v1/jobs` (SPEC-008) and prints them in the active output format.

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--status <status>` | All | Filter by job lifecycle state: `Pending`, `Processing`, `Completed`, `Failed`. |
| `--limit <n>` | 20 | Maximum number of jobs to display. |

**Text output format:**
```
JOB ID                               STATUS      SOLVER OUTCOME  CREATED AT
a1b2c3d4-...                         Completed   Succeeded       2026-06-20T10:00:00Z
b2c3d4e5-...                         Failed      —               2026-06-20T09:58:00Z
c3d4e5f6-...                         Processing  —               2026-06-20T10:01:00Z
```

**JSON output format:** Array of job summary objects. The exact schema is determined by the `GET /v1/jobs` list endpoint definition (OQ-1). The minimum required fields per entry are: `job_id`, `status`, `solver_outcome` (present when `status = Completed`), and `created_at`. The full single-job detail schema (SPEC-008 FR-8) is not required for list entries.

**Note:** This command depends on a `GET /v1/jobs` list endpoint. If no such endpoint exists in the API at implementation time, this command is deferred (see OQ-1).

**Acceptance Criteria:**
- The command prints up to `--limit` jobs, most recently created first.
- `--status` filters the result set before display.
- An empty result set prints a message stating no jobs were found, not an error.

---

## FR-10: Report Show Command

**Description:** `daedalus report show <job_id>` retrieves the evidence report for a completed job via `GET /v1/jobs/{job_id}/report` (SPEC-008 FR-12) and `GET /v1/reports/{report_id}` (SPEC-008 FR-13), and either writes it to a file or opens it in a browser.

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--output <file>` | stdout | Write the HTML report to `<file>`. When `-`, write to stdout. |
| `--open` | false | After saving, attempt to open the report in the system default browser. |

**Normal behavior:**
1. Fetch report metadata from `GET /v1/jobs/{job_id}/report`.
2. On HTTP 200: fetch the report file from `report_url`.
3. Write the HTML content to `--output`.
4. If `--open` and output was written to a file: open the file in the system default browser. If output was written to stdout (default or `--output -`), `--open` has no effect; a warning is written to stderr.

**Output destination rules:**
- When `--output <file>` names a file: the HTML is written to that file. Progress summary goes to stdout.
- When `--output -` (explicit) or `--output` is absent (default stdout): the HTML is written to stdout. No other text appears on stdout. All progress summary and metadata go to stderr.

**Text output format (when writing to a file — `--output <file>`):**
```
Report found.
  report_id:    r1s2t3u4-...
  generated_at: 2026-06-20T10:00:05Z
  report_format: html
Saved to: ./report.html
```

**Text output format (when writing to stdout — default or `--output -`):**
The metadata summary ("Report found. ...") is written to stderr. Stdout contains only the HTML content. No "Saved to:" line appears.

**Acceptance Criteria:**
- For a `Completed` job with `report_available = true`: the HTML report is written to the output destination.
- For a job that has no report (still executing, report generation failed, or `Failed` job): HTTP 404 is received; a message informs the caller that no report is available and exits with code 1.
- When output goes to stdout (either by default or via `--output -`), no text other than the HTML content appears on stdout. All progress messages and metadata go to stderr.
- When `--output <file>` names a file, the metadata summary and "Saved to:" line are written to stdout.
- `--open` does not fail the command if the system browser cannot be located; the file is saved and a warning is printed to stderr.
- When output is written to stdout (default or `--output -`), `--open` has no effect and a warning is written to stderr.
- The CLI does not transform or modify the HTML content.

---

## FR-11: Scheduler Configuration Commands

**Description:** The CLI provides commands to create, list, and retrieve scheduler configurations by wrapping the SPEC-008 FR-10 endpoints.

### `daedalus config create <file>`

Reads a scheduler configuration JSON file and submits it to `POST /v1/scheduler-configs`.

**Config file format:**
```json
{
  "objective_mode": "Balanced",
  "mode_parameters": {
    "cost_weight": 0.33,
    "latency_weight": 0.34,
    "quality_weight": 0.33
  }
}
```

**Text output format (201):**
```
Scheduler configuration created.
  config_id:      c1d2e3f4-...
  objective_mode: Balanced
  created_at:     2026-06-20T10:00:00Z
```

### `daedalus config list`

Retrieves all scheduler configurations via `GET /v1/scheduler-configs`.

**Text output format:**
```
CONFIG ID                            OBJECTIVE MODE  CREATED AT
c1d2e3f4-...                         Balanced        2026-06-20T09:55:00Z  (default)
d2e3f4g5-...                         CheapestValid   2026-06-20T10:00:00Z
```

### `daedalus config get <config_id>`

Retrieves a specific scheduler configuration via `GET /v1/scheduler-configs/{config_id}`.

### `daedalus config default`

Prints the resolved CLI configuration (FR-2) and the API's default scheduler configuration. The API default is retrieved via `GET /v1/scheduler-configs` and identified as the first entry in the response.

**Acceptance Criteria:**
- `config create` with a missing required field (`objective_mode` absent) prints the API error and exits with code 2.
- `config create` with an unrecognized `objective_mode` prints the API validation error and exits with code 2.
- `config list` includes the default configuration and marks it visually in text format.
- `config get` with an unknown `config_id` exits with code 1.
- All commands print the full configuration body in JSON format when `--output-format json`.

---

## FR-12: Experiment Run Command

**Description:** `daedalus experiment run <manifest>` is the primary experiment orchestration entry point. It reads a SPEC-020-conforming experiment manifest from `<manifest>`, submits it to `POST /v1/experiments` (SPEC-008 FR-18), and executes the full trial orchestration loop (FR-18) until all trials reach terminal state or the timeout expires.

All experiment state is durably persisted by the API (ADR-012 Decision 2). The CLI's responsibility is to drive the orchestration loop and surface progress and results. The API is the state authority; the CLI is the executor.

**Manifest format (SPEC-020 / SPEC-008 FR-18):**

| Field | Required | Description |
|---|---|---|
| `experiment_name` | Yes | Human-readable name for this experiment |
| `benchmark_id` | **Yes** | Groups this experiment under a named benchmark (SPEC-008 FR-18, SPEC-008 FR-19). The `benchmark_id` string must be non-empty; a `benchmark_manifests` record for that ID is not required to exist before experiment submission. |
| `workload_set` | Yes | Fixed or Generated workload set definition (see Workload Set Modes below) |
| `solver_set` | Yes | Ordered list of `backend_id` strings (SPEC-020 FR-5); minimum 1 element |
| `repetition_count` | Yes | Number of repetitions per (problem, solver) combination; must be ≥ 1 |
| `execution_timeout_ms_per_trial` | Yes | Per-trial job execution timeout in milliseconds; must be positive |
| `experiment_seed` | **Yes** | Top-level entropy source for trial seed derivation (uint64 as bigint per SPEC-012 FR-4.3, SPEC-020 FR-8). Required by SPEC-008 FR-18; must be non-negative. |
| `seed_derivation_algorithm_version` | **Yes** | Version identifier for the seed derivation algorithm used by the CLI to compute per-trial seeds from `experiment_seed` (SPEC-012 FR-20.4). Enables reproducibility verification across CLI versions. |
| `scheduler_config_id` | No | Scheduler configuration applied to all trial jobs; absent uses the API default |
| `reproducibility_class` | Yes | `fully_reproducible`, `partially_reproducible`, or `non_reproducible` (SPEC-020 FR-10). Derived by the CLI from the union of `determinism_class` values across `solver_set` backends. |
| `hardware_policy` | No | Retry and failure handling for `quantum_hardware` backend trials (SPEC-008 FR-18, SPEC-020 FR-10). Defaults apply when absent. |

**Workload set modes:**

- **Fixed mode** (`"mode": "fixed"`): Specifies an explicit list of `problem_ids`. Each `problem_id` must reference a routing problem already persisted in the system. For each `(problem_config_index, repetition_index)`, all solvers in `solver_set` share the same `problem_id` (ADR-012 Decision 3 instance-sharing invariant; required for valid cross-solver SPEC-007 FR-7 `hindsight_quality` comparisons). This is the only workload set mode supported at MVP scope.
- **Generated mode** (`"mode": "generated"`): Deferred pending resolution of SPEC-008 OQ-7 (the routing problem creation mechanism for generated workload sets). The CLI rejects manifests specifying `"mode": "generated"` with exit code 1 at MVP scope. See OQ-5.

**Fixed mode manifest example:**

```json
{
  "experiment_name": "capacity-only-solver-comparison-2026",
  "benchmark_id": "capacity-comparison-2026",
  "workload_set": {
    "mode": "fixed",
    "problem_ids": ["e5f6g7h8-...", "a1b2c3d4-..."]
  },
  "solver_set": ["backend-vrp-greedy-v1", "backend-vrp-metaheuristic-v1"],
  "repetition_count": 3,
  "execution_timeout_ms_per_trial": 30000,
  "experiment_seed": 9221134881191785473,
  "seed_derivation_algorithm_version": "1.0",
  "scheduler_config_id": "c1d2e3f4-...",
  "reproducibility_class": "fully_reproducible"
}
```

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--timeout <seconds>` | 3600 | Wall-clock seconds before CLI exits. The experiment continues in the API after CLI exit. |
| `--poll-interval <seconds>` | 2 | Job status polling interval in seconds. |
| `--output-manifest <file>` | `<experiment_name>.result.json` | Write the result manifest to this file on completion. |

**Normal behavior:**
1. Read and parse the experiment manifest from `<manifest>`.
2. Reject `"mode": "generated"` manifests with exit code 1 before any API call.
3. Submit to `POST /v1/experiments`; capture the returned `experiment_id`.
4. Print the experiment ID and planned trial count.
5. Execute the trial orchestration loop (FR-18): submit all trials, poll for terminal state, collect evidence per trial.
6. After all trials reach terminal state: print the experiment summary table.
7. Write the result manifest to the output manifest path (default: `<experiment_name>.result.json`). Existing files are overwritten without prompt.

**Text output format:**
```
Experiment submitted.
  experiment_id:  7f8a9b0c-...
  experiment_name: capacity-only-solver-comparison-2026
  planned_trials:  12

Submitting trials...
  [1/12] trial a1b2c3d4-... submitted → job f1e2d3c4-...
  ...

Collecting results...
  [1/12] job f1e2d3c4-... → Completed (Succeeded) — evidence collected
  ...

Experiment complete.
  experiment_id: 7f8a9b0c-...
  status:        Completed

TRIAL  PROBLEM ID      BACKEND                       JOB ID          TRIAL STATUS  OUTCOME
1      e5f6g7h8-...    backend-vrp-greedy-v1         a1b2c3d4-...    Completed     Succeeded
2      e5f6g7h8-...    backend-vrp-metaheuristic-v1  b2c3d4e5-...    Completed     Succeeded
...
```

**JSON output format (`--output-format json`):** In JSON mode, all text progress output is suppressed during the orchestration loop. On normal completion (exit code 0 or 5), the CLI emits a single JSON object to stdout:

```json
{
  "experiment_id": "7f8a9b0c-...",
  "experiment_name": "capacity-only-solver-comparison-2026",
  "benchmark_id": "capacity-comparison-2026",
  "status": "Completed",
  "planned_trial_count": 12,
  "trials": [
    {
      "trial_id": "a1b2c3d4-...",
      "problem_id": "e5f6g7h8-...",
      "backend_id": "backend-vrp-greedy-v1",
      "trial_status": "Completed",
      "job_id": "f1e2d3c4-...",
      "solver_outcome": "Succeeded",
      "evidence_status": "Collected"
    }
  ]
}
```

On code 6 (`experiment.status = Failed`), the same object is emitted with `"status": "Failed"`. On exit code 4 (timeout) or 1 (signal), no JSON object is emitted; the standard JSON error schema from FR-15 is emitted instead, with `request_id: null` (no API error response triggered the exit).

**Long-running session behavior:** `experiment run` is a scoped exception to the CLI's single-invocation posture (see Assumptions). For large experiments, the command may run for minutes to hours. The `--timeout` flag provides an upper bound. If the CLI exits before completion (timeout or signal), the experiment state is preserved by the API. See FR-21.

**Acceptance Criteria:**
- A valid Fixed mode manifest produces an experiment submission and enters the orchestration loop (FR-18).
- A manifest specifying `"mode": "generated"` exits with code 1 before any API call.
- A manifest missing `benchmark_id`, `experiment_seed`, or `seed_derivation_algorithm_version` is rejected by the API with HTTP 400; CLI prints all `field_errors` and exits with code 2.
- A manifest rejected by the API for any other reason (HTTP 400) prints all `field_errors` and exits with code 2.
- A manifest with an empty `solver_set` is rejected with code 2.
- A manifest with `repetition_count` less than 1 is rejected with code 2.
- When all trials complete normally: the result manifest captures `experiment_id`, `experiment_name`, `benchmark_id`, planned trial count, and per-trial `trial_id`, `problem_id`, `job_id`, `backend_id`, `trial_status`, `solver_outcome` (when available), and `evidence_status`.
- When `experiment.status = Failed`: the CLI exits with code 6.
- When any trial's job reaches `status = Failed`: the CLI exits with code 5 (unless experiment also fails, in which case code 6 takes precedence).
- If the output manifest file already exists, it is overwritten without prompt.
- In `--output-format json` mode, no progress text appears on stdout during the orchestration loop. A single JSON result object is emitted on stdout on successful completion (exit codes 0 and 5). On exit code 4 or 1, the JSON error schema from FR-15 is emitted.
- No result manifest is written to disk on timeout (exit code 4) or signal (exit code 1). Experiment state is recoverable via `experiment status` and `experiment trials`.

---

## FR-13: Output Format Contract

**Description:** All CLI commands support two output formats, selected by `--output-format`.

**`text` format (default):**
- Human-readable output with labelled fields.
- Color-coded status values when the terminal supports color codes (suppressed by `--no-color` or when stdout is not a terminal).
- Informational messages and hints printed alongside primary output.
- Progress indicators (polling status) printed to stderr when stdout carries a primary output (e.g., a report file).

**`json` format:**
- Primary output is a single JSON object or array written to stdout.
- No decorative text, no color codes, no interactive hints.
- All errors are also written to stdout as a JSON object with `error_code`, `message`, `request_id`, and `field_errors` fields (consistent with the SPEC-008 error model; see JSON error schema below).
- Suitable for piping to `jq` or automation scripts.

**JSON error schema (used when `--output-format json` and an error occurs):**
```json
{
  "error_code": "string",
  "message": "string",
  "request_id": "string | null",
  "field_errors": [
    { "field": "string", "message": "string" }
  ]
}
```

`request_id` is present and non-null when the error originates from an API response (SPEC-008 FR-14 error model). It is `null` or absent when the error is CLI-local (file not found, generation failure, local pre-validation, network failure before an API response was received). This preserves the SPEC-008 FR-14 `request_id` for observability correlation (ADR-006, ADR-011) without fabricating a value for non-API errors.

**Acceptance Criteria:**
- When `--output-format json`, stdout contains only valid JSON. No decorative text appears on stdout.
- When `--output-format text`, output is human-readable and formatted.
- All commands produce valid JSON when `--output-format json`, including on error paths.
- When an API error response is received, the `request_id` from the SPEC-008 FR-14 error body is included in the CLI's JSON error output unchanged. It is not transformed or regenerated by the CLI.
- When an error is CLI-local (no API response received), `request_id` is `null` in the JSON error output.
- `--quiet` suppresses non-primary output in both formats. In text mode, only the most essential result line is printed. In JSON mode, `--quiet` has no effect (JSON output is already compact).

---

## FR-14: Exit Codes

**Description:** The CLI uses a defined set of exit codes. Automation scripts must be able to rely on these codes to branch on outcomes.

| Code | Name | Meaning |
|---|---|---|
| 0 | `SUCCESS` | Command completed successfully. |
| 1 | `ERROR` | General error: unknown command, resource not found, API unreachable, configuration error, unexpected failure. |
| 2 | `VALIDATION_ERROR` | The submitted routing problem, experiment manifest, or configuration was rejected by the API or local pre-validation. |
| 3 | `GENERATION_FAILURE` | The workload generator failed (capacity infeasibility, achievability violation, invalid generation config). |
| 4 | `WAIT_TIMEOUT` | The `job wait`, `problem run --wait`, or `experiment run --timeout` elapsed before the job or experiment reached a terminal state. |
| 5 | `JOB_FAILED` | A waited job reached the terminal state `Failed` (Worker lifecycle failure, not a solver failure); or one or more trial jobs in an experiment reached `status = Failed`. |
| 6 | `EXPERIMENT_FAILED` | The experiment itself transitioned to `status = Failed` (experiment-level infrastructure failure preventing completion). Distinct from individual trial job failures (code 5). |

**Solver-level outcomes for `Completed` jobs** do not cause CLI failure. SPEC-008 FR-9 defines the complete set of `solver_outcome` values that a `Completed` job may carry: `Succeeded`, `Infeasible`, `Timeout`, `Cancelled`, `Failed`, `ContractViolation`. All six produce exit code 0 when a waited job reaches `status = Completed`. The CLI distinguishes solver-level outcomes from Worker lifecycle outcomes:

| `status` | `solver_outcome` | CLI exit code | Notes |
|---|---|---|---|
| `Completed` | `Succeeded` | 0 | Solver found a valid solution |
| `Completed` | `Infeasible` | 0 | Solver determined problem has no feasible solution |
| `Completed` | `Timeout` | 0 | Solver exhausted time budget without finding a solution |
| `Completed` | `Cancelled` | 0 | Cancellation flag was observed before solver dispatch |
| `Completed` | `Failed` | 0 | Solver returned an execution failure code (solver-level, not Worker lifecycle) |
| `Completed` | `ContractViolation` | 0 | Solver returned a structurally invalid response (SPEC-008 FR-9) |
| `Failed` | _(absent)_ | 5 | Worker could not complete its execution lifecycle; no evidence record produced |

`solver_outcome = Failed` (a solver-level outcome for a `Completed` job) is distinct from `status = Failed` (a Worker lifecycle failure). A job with `status = Completed` and `solver_outcome = Failed` exits the CLI with code 0. Only `status = Failed` produces exit code 5. Callers must inspect `solver_outcome` to determine the optimization result; exit code 0 from a waited job does not imply solver success.

**Acceptance Criteria:**
- Every command path exits with exactly one of the defined codes.
- Exit code 0 is used only when the command's primary operation succeeded.
- Exit code 5 is used when a waited job reaches `status = Failed` (Worker lifecycle failure), or when one or more trial jobs in an `experiment run` reach `status = Failed`. It is not used for any `solver_outcome` value on a `Completed` job.
- Exit code 6 is used when `experiment.status = Failed` and takes precedence over exit code 5 when both conditions are true.
- Exit code 2 is used for both local validation errors (bad generation config, invalid manifest) and API validation errors (HTTP 400 from SPEC-008).
- Exit code 4 is used when `experiment run --timeout` expires before the experiment reaches terminal state.
- Exit codes are consistent across `text` and `json` output formats.
- Exit code 0 is produced for a waited job with `solver_outcome = ContractViolation` (a `Completed` job with an invalid solver response).
- Exit code 0 is produced for a waited job with `solver_outcome = Failed` (a `Completed` job with a solver execution failure).

---

## FR-15: Error Output Contract

**Description:** The CLI produces structured error output for all failure conditions. Errors follow the same format as normal output (FR-13).

**Text format error:**
```
Error: validation failed (exit 2)
  average_vehicle_speed_kmh: must be a positive number
  stops[3].demand: must be non-negative
```

**JSON format error (API validation error — `request_id` propagated from SPEC-008 FR-14 response):**
```json
{
  "error_code": "VALIDATION_ERROR",
  "message": "Routing problem validation failed",
  "request_id": "f1e2d3c4-b5a6-7890-abcd-ef1234567890",
  "field_errors": [
    { "field": "average_vehicle_speed_kmh", "message": "must be a positive number" },
    { "field": "stops[3].demand", "message": "must be non-negative" }
  ]
}
```

**JSON format error (CLI-local error — no API response received, `request_id` is null):**
```json
{
  "error_code": "GENERATION_FAILURE",
  "message": "Generation failed: total demand (432) exceeds total capacity (400)",
  "request_id": null,
  "field_errors": []
}
```

**Error categories:**

| Category | Source | Presentation |
|---|---|---|
| API validation error | HTTP 400 from SPEC-008 FR-14 | Print all `field_errors` from the API response |
| API not found | HTTP 404 | Print the resource type and identifier |
| API conflict | HTTP 409 | Print the conflict reason |
| API server error | HTTP 500 | Print `INTERNAL_ERROR` without exposing internal details |
| Local validation | CLI pre-validation (config file, flags) | Print the offending field and rule |
| Generation failure | SPEC-002 construction self-check | Print the failure mode with diagnostic context (stop index, computed values per FR-4) |
| Network error | TCP/DNS failure reaching the API | Print the URL and the low-level error; suggest checking `--api-url` |
| Timeout | Wait polling timeout | Print the `job_id` and elapsed time; note the job continues executing |

**Acceptance Criteria:**
- When an API error response is received, the `error_code`, `message`, `request_id`, and `field_errors` fields from the SPEC-008 FR-14 error body are propagated to CLI output without transformation. No field is removed, added, or renamed.
- When an error is CLI-local (generation failure, file not found, pre-validation, network failure before an API response was received), `request_id` is `null` in JSON output and absent in text output.
- Internal details (stack traces, connection strings, PostgreSQL error codes) never appear in CLI output.
- Network-level errors include the API URL that was attempted, enabling the caller to diagnose misconfigured `--api-url`.
- All errors are written to stderr in text format. In JSON format, errors are written to stdout so that automation can parse them.

---

## FR-16: Observability

**Description:** The CLI is a developer tool with no OpenTelemetry instrumentation obligation. It does not emit OTel spans. It does not contribute to the system's distributed trace.

For experiment endpoint interactions (manifest submission, trial submission, evidence collection trigger), W3C TraceContext propagation is handled at the API layer per ADR-011 and ADR-012. The CLI does not inject `traceparent` headers and does not generate trace context. The API creates trace roots for all experiment endpoint calls. `request_id` values from SPEC-008 FR-14 error responses are propagated through CLI output per FR-13 and FR-15, enabling post-hoc correlation with API traces.

The CLI emits structured log events to stderr in JSON format when `DAEDALUS_LOG=debug` is set in the environment. These events are diagnostic aids for CLI development and troubleshooting.

**Log events:**

| Event | Trigger | Fields |
|---|---|---|
| `cli.command.start` | Command begins | `command`, `args`, `api_url` |
| `cli.api.request` | HTTP request issued | `method`, `url`, `body_size_bytes` |
| `cli.api.response` | HTTP response received | `status_code`, `latency_ms` |
| `cli.generation.start` | Generator invoked | `scenario_type`, `seed`, `stop_count` |
| `cli.generation.complete` | Generator returned | `difficulty_tier`, `time_windowed_stop_count`, `total_demand` |
| `cli.poll.tick` | Status poll completed | `job_id`, `status`, `elapsed_seconds` |
| `cli.experiment.submit` | Experiment manifest submitted | `experiment_id`, `experiment_name`, `planned_trial_count` |
| `cli.experiment.trial_submit` | Trial linked to job | `experiment_id`, `trial_id`, `job_id`, `problem_config_index`, `repetition_index`, `solver_set_index` |
| `cli.experiment.trial_complete` | Trial job reaches terminal state | `experiment_id`, `trial_id`, `job_id`, `job_status`, `solver_outcome` |
| `cli.experiment.evidence_collect` | Evidence collection triggered | `experiment_id`, `trial_id`, `job_id`, `http_status`, `evidence_status` |
| `cli.experiment.complete` | Experiment reaches terminal status | `experiment_id`, `experiment_status`, `total_trials`, `elapsed_seconds` |
| `cli.command.exit` | Command exits | `exit_code`, `duration_ms` |

**Log safety:** Log events must not include routing problem coordinate arrays, full stop lists, or `execution_seed` values. Job identifiers, trial identifiers, stop counts, and experiment identifiers are safe to log.

**Acceptance Criteria:**
- No OTel spans are emitted by the CLI.
- The CLI does not inject `traceparent` or `tracestate` headers into outgoing API requests.
- Debug log events appear on stderr only when `DAEDALUS_LOG=debug` is set.
- Debug log events are suppressed when `DAEDALUS_LOG` is unset or set to any other value.
- Debug log events are valid JSON, one event per line.
- Log events never include routing problem coordinate arrays.
- `cli.experiment.trial_submit` events appear for each trial in the orchestration loop.
- `cli.experiment.evidence_collect` events appear for each trial when evidence collection is triggered.

---

## FR-17: Benchmark Manifest Submission Command

**Description:** `daedalus benchmark submit <file>` submits a benchmark manifest to `POST /v1/benchmarks` (SPEC-008 FR-19), creating a named benchmark container. Benchmarks group related experiments under a stable identifier. A benchmark is created before the experiments that reference it via `benchmark_id`.

**Benchmark manifest format (SPEC-008 FR-19 / SPEC-020 FR-2):**

| Field | Type | Required | Description |
|---|---|---|---|
| `benchmark_id` | string | Yes | Stable, human-readable identifier (e.g., `capacity-comparison-2026`). Must be unique across all benchmarks. Serves as the lookup key for all member experiments and the benchmark summary. |
| `benchmark_name` | string | Yes | Human-readable display name for the benchmark. |
| `research_question` | string | Yes | The question the benchmark attempts to answer (SPEC-020 FR-2). Example: "Does QUBO simulated annealing produce lower total route distance than nearest-neighbor on small routing problems?" |
| `hypothesis` | string | Yes | What is expected to happen (SPEC-020 FR-2). |
| `null_hypothesis` | string | Yes | What would be true if the proposed improvement does not exist (SPEC-020 FR-2). |
| `controls` | array of strings | Yes | Variables held constant across experiments; minimum 1 element (SPEC-020 FR-2). Example: `["problem_size_class", "fleet_configuration"]`. |
| `independent_variables` | array of strings | Yes | Variables that change between experiments; minimum 1 element (SPEC-020 FR-2). Example: `["backend_id"]`. |
| `dependent_variables` | array of strings | Yes | Variables measured; minimum 1 element (SPEC-020 FR-2). Example: `["hindsight_quality", "execution_duration_ms"]`. |

All string fields must be non-empty. All array fields must contain at least one element. These constraints are enforced by the API (SPEC-008 FR-19 validation).

**Example:**

```json
{
  "benchmark_id": "capacity-comparison-2026",
  "benchmark_name": "Classical vs QUBO SA — Capacity-Only VRP",
  "research_question": "Does QUBO simulated annealing produce lower total route distance than nearest-neighbor on small capacity-only VRP problems?",
  "hypothesis": "QUBO SA produces lower total route distance than nearest-neighbor on problems with 15–25 stops.",
  "null_hypothesis": "There is no statistically significant difference in total route distance between QUBO SA and nearest-neighbor on the evaluated problem distribution.",
  "controls": ["problem_size_class", "fleet_configuration", "time_window_presence"],
  "independent_variables": ["backend_id"],
  "dependent_variables": ["hindsight_quality", "execution_duration_ms", "solver_outcome"]
}
```

**Normal behavior:**
1. Read and parse the benchmark manifest from `<file>`.
2. Submit to `POST /v1/benchmarks`.
3. On HTTP 201: print `benchmark_id`, `benchmark_name`, `created_at`.
4. On HTTP 409: print the conflict (benchmark already exists) and exit with code 1.

**Text output format:**
```
Benchmark submitted.
  benchmark_id:   capacity-comparison-2026
  benchmark_name: Classical vs QUBO SA — Capacity-Only VRP
  created_at:     2026-06-23T10:00:00Z
```

**Acceptance Criteria:**
- A valid benchmark manifest (all required fields present and non-empty, all arrays non-empty) produces HTTP 201 and prints the response fields.
- A duplicate `benchmark_id` returns HTTP 409; CLI prints the conflict and exits with code 1.
- A manifest missing any required field (`benchmark_id`, `benchmark_name`, `research_question`, `hypothesis`, `null_hypothesis`, `controls`, `independent_variables`, `dependent_variables`) is rejected with HTTP 400 (`error_code = BENCHMARK_CONFIG_INVALID`); CLI prints all `field_errors` and exits with code 2.
- A manifest with an empty string value for any required string field is rejected with HTTP 400; CLI prints field errors and exits with code 2.
- A manifest with an empty array for `controls`, `independent_variables`, or `dependent_variables` is rejected with HTTP 400; CLI prints field errors and exits with code 2.
- `--output-format json` emits the raw SPEC-008 FR-19 response body as a JSON object to stdout.

---

## FR-18: Experiment Trial Orchestration Loop

**Description:** The trial orchestration loop is the internal execution behavior of `daedalus experiment run` (FR-12) after manifest submission. The CLI iterates over all planned trials and drives each from `Pending` through submission, execution, and evidence collection.

**Trial retrieval:** Immediately after manifest submission, the CLI calls `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22) to retrieve the complete planned trial set. The response includes all `trial_id`, `problem_id`, `backend_id`, and `solver_set_index` values for each planned trial.

**Trial submission order (SPEC-020 FR-3):** Trials are submitted in ascending order of `(problem_config_index, repetition_index, solver_set_index)`. All trials are submitted sequentially before polling begins.

**Per-trial submission:**

For each trial in submission order:
1. Submit a job via `POST /v1/jobs`, specifying the trial's `problem_id` (from the Fixed workload set) and the experiment's `scheduler_config_id`.
2. Link the trial to the job by calling `POST /v1/experiments/{experiment_id}/trials/{trial_id}/submit` with the returned `job_id` (SPEC-008 FR-25).
3. Proceed to the next trial without waiting for the current job to execute.

**Submission phase error handling:** Each trial submission is a two-step sequence (`POST /v1/jobs` + `POST /v1/experiments/{id}/trials/{trial_id}/submit`). Error handling during the submission phase:

- **`POST /v1/jobs` HTTP 5xx:** Treat as fatal. Exit with code 1 without beginning trial-link step for the failed trial.
- **`POST /v1/jobs` network failure (TCP/DNS):** Retry once at the next opportunity. On second consecutive network failure for the same trial, exit with code 1.
- **`POST /v1/experiments/{id}/trials/{trial_id}/submit` HTTP 5xx (after successful job submit):** Retry up to 3 consecutive attempts. On the third consecutive 5xx, exit with code 1. The job has been created and will execute; the orphaned state is logged at `DAEDALUS_LOG=debug` for investigation.
- **`POST /v1/experiments/{id}/trials/{trial_id}/submit` network failure (after successful job submit):** Retry once. On second consecutive failure, the CLI retries with the same `job_id`; if `TRIAL_ALREADY_SUBMITTED` is returned (first attempt succeeded), the CLI recovers per the Idempotency section. If HTTP 5xx or network failure persists, exit with code 1.
- **`TRIAL_ALREADY_SUBMITTED` from `POST /v1/experiments/{id}/trials/{trial_id}/submit`:** Treat as success-on-retry. Call `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22), retrieve the already-linked `job_id` for that trial, and continue to the next trial. Do not exit.

**Per-backend job targeting — unresolved dependency (ENG-003 / OQ-6):** The current `POST /v1/jobs` contract (SPEC-008 FR-2) accepts `problem_id` and `scheduler_config_id` but provides no `backend_id` field. With a single experiment-wide `scheduler_config_id`, the Scheduler selects the backend by policy — there is no mechanism in the current API contract to guarantee that the job submitted for a `backend-vrp-greedy-v1` trial is actually processed by that backend rather than another eligible backend. Per-backend job routing is therefore an unresolved implementation dependency. See OQ-6. Until OQ-6 is resolved, multi-backend experiments cannot guarantee correct per-backend execution.

**Timeout during submission phase:** The `--timeout` clock runs from manifest submission, including the submission phase. If `--timeout` expires during the submission phase, the CLI completes the two-step submission (job submit + trial link) for the current trial, then exits with code 4 without beginning the next trial's submission. Trials already submitted and linked are preserved in the API. No result manifest is written.

**Polling phase:**

After all trials are submitted, the CLI enters the polling phase:
- At each poll interval, check the job status of each non-terminal trial via `GET /v1/jobs/{job_id}`.
- When a trial's job reaches a terminal state (`Completed` or `Failed`): immediately call `POST /v1/experiments/{experiment_id}/trials/{trial_id}/collect-evidence` (SPEC-008 FR-21).
- **Retryable conditions** from collect-evidence: `JOB_NOT_TERMINAL` (HTTP 409) — retry at the next poll interval. HTTP 5xx server errors — retry at the next poll interval, up to 3 consecutive attempts; treat as fatal (exit code 1) on the third consecutive 5xx for the same trial. Network failure (TCP/DNS) — retry once at the next poll interval; treat as fatal on the second consecutive network failure for the same trial.
- **Fatal conditions** from collect-evidence: `TRIAL_NOT_SUBMITTED` (HTTP 409) — indicates `job_id` is null despite the guard in the submission phase; this is a state inconsistency and the CLI exits with code 1.
- `evidence_status = Error` on a completed collect-evidence call: the evidence was collected but is incomplete or invalid (SPEC-008 FR-21 idempotency states this is retryable). The CLI logs the outcome and treats the trial as evidence-collection-complete for orchestration-loop progress purposes. `evidence_status = Error` does not contribute to a non-zero exit code; it is surfaced in the result manifest and the completion table.
- At each poll interval, additionally call `GET /v1/experiments/{experiment_id}`. If `experiment.status = Failed` is detected at any point during the polling phase, the CLI exits with code 6 immediately without waiting for remaining trials.
- Continue polling until all trials are terminal and evidence collection has been attempted for each, or until `experiment.status = Failed` is detected.

**Experiment completion monitoring:**

After all trials have evidence collection attempted and `experiment.status` is not yet `Failed`, the CLI polls `GET /v1/experiments/{experiment_id}` until `experiment.status` is `Completed` or `Failed`. The API auto-transitions the experiment on the API side (SPEC-008 FR-21 auto-completion); the CLI confirms the final status. `SchedulerRejected` and `HarnessError` trial statuses are set by the API during evidence collection based on collected evidence; they require no special CLI handling and are treated as terminal `trial_status` values for orchestration-loop progress purposes.

**Idempotency:** Evidence collection (SPEC-008 FR-21) is idempotent at the API layer: repeated calls for a trial with `evidence_status = Collected` return HTTP 200 with the existing state. Trial submission linkage (SPEC-008 FR-25) is **not idempotent**: a second call for a trial whose `job_id` is already set returns HTTP 409 `TRIAL_ALREADY_SUBMITTED`. If the CLI receives `TRIAL_ALREADY_SUBMITTED` from `POST /v1/experiments/{experiment_id}/trials/{trial_id}/submit` during a retry (indicating the first call succeeded but the response was not received), the CLI calls `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22) to retrieve the already-linked `job_id` for that trial and proceeds without treating the 409 as fatal. Full automatic CLI resumption after process interruption is deferred to post-MVP implementation planning per ADR-012.

**Acceptance Criteria:**
- Trials are submitted in `(problem_config_index asc, repetition_index asc, solver_set_index asc)` order.
- All trials are submitted before polling begins.
- Evidence collection is triggered per trial immediately when that trial's job reaches a terminal state.
- The CLI does not call collect-evidence for a trial whose `job_id` is null.
- `JOB_NOT_TERMINAL` from collect-evidence is retried at the next poll interval; it does not cause exit.
- `TRIAL_NOT_SUBMITTED` (HTTP 409) from collect-evidence causes immediate exit with code 1 (fatal state inconsistency).
- HTTP 5xx from collect-evidence is retried up to 3 consecutive attempts per trial; on the third consecutive 5xx, the CLI exits with code 1.
- `evidence_status = Error` on a collect-evidence response is recorded in the result manifest and completion table; it does not cause a non-zero exit code.
- At each poll interval, `GET /v1/experiments/{experiment_id}` is called; if `experiment.status = Failed` is detected, the CLI exits with code 6.
- If `--timeout` expires during the submission phase, the CLI completes the current trial's two-step submission and exits with code 4 without beginning the next trial.
- Experiment auto-completion status is confirmed via `GET /v1/experiments/{experiment_id}` after all trials report terminal.
- When any trial's job reaches `status = Failed`, evidence collection is still attempted for that trial. The final exit code reflects the worst outcome (code 5 for any trial `status = Failed`; code 6 for `experiment.status = Failed`; code 6 takes precedence).
- `POST /v1/jobs` returning HTTP 5xx during the submission phase causes immediate exit with code 1; the trial-link step is not attempted.
- `POST /v1/jobs` network failure during the submission phase is retried once; on a second consecutive failure for the same trial, the CLI exits with code 1.
- `POST /v1/experiments/{id}/trials/{trial_id}/submit` returning HTTP 5xx (after successful job submit) is retried up to 3 consecutive attempts; on the third consecutive 5xx, the CLI exits with code 1.
- `TRIAL_ALREADY_SUBMITTED` (HTTP 409) from the trial-link endpoint is treated as success-on-retry: the CLI calls `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22) to retrieve the already-linked `job_id` and proceeds; it does not exit.

---

## FR-19: Experiment State Retrieval Commands

**Description:** Three `experiment` subcommands retrieve experiment state from the API without orchestrating a run. These are the read-only complement to `experiment run`.

### `daedalus experiment status <experiment_id>`

Calls `GET /v1/experiments/{experiment_id}` (SPEC-008 FR-20) and prints the current status and trial counts.

**Text output format:**
```
EXPERIMENT ID                         NAME                                  STATUS   TRIALS  COMPLETE
7f8a9b0c-...                          capacity-only-solver-comparison-2026  Running  12      7
```

**Acceptance Criteria:**
- An unknown `experiment_id` exits with code 1.
- Outputs include `status`, `planned_trial_count`, and count of trials in terminal state.
- `--output-format json` emits the raw SPEC-008 FR-20 response object.

---

### `daedalus experiment trials <experiment_id>`

Calls `GET /v1/experiments/{experiment_id}/trials` (SPEC-008 FR-22) and prints the per-trial status table. All columns are derived from the SPEC-008 FR-22 response; no additional per-job API calls are issued.

**Text output format:**
```
TRIAL ID         PROBLEM ID      BACKEND                       JOB ID          TRIAL STATUS  EVIDENCE STATUS  OUTCOME
a1b2c3d4-...     e5f6g7h8-...    backend-vrp-greedy-v1         f1e2d3c4-...    Completed     Collected        Succeeded
b2c3d4e5-...     e5f6g7h8-...    backend-vrp-metaheuristic-v1  —               Pending       —                —
```

**Acceptance Criteria:**
- An unknown `experiment_id` exits with code 1.
- Trials with `job_id = null` (not yet submitted) display `—` in the `JOB ID`, `EVIDENCE STATUS`, and `OUTCOME` columns.
- `--output-format json` emits the raw SPEC-008 FR-22 response array, including all fields defined in SPEC-008 FR-22 (trial seeds, indices, evidence fields).

---

### `daedalus experiment summary <experiment_id>`

Calls `GET /v1/experiments/{experiment_id}/summary` (SPEC-008 FR-23) and prints or saves the experiment summary artifact.

**Flags:**

| Flag | Description |
|---|---|
| `--output <file>` | Write the summary JSON to this file. When absent, prints to stdout. |

**Acceptance Criteria:**
- An unknown `experiment_id` (HTTP 404, `error_code = EXPERIMENT_NOT_FOUND`) prints a not-found message identifying the `experiment_id` and exits with code 1.
- An experiment that exists but whose `status` is not `Completed` (HTTP 404, `error_code = EXPERIMENT_SUMMARY_NOT_FOUND`) prints a not-yet-available message and exits with code 1.
- When `--output <file>` is provided, the summary JSON is written to that file and the file path is printed to stdout.
- `--output-format json` emits the raw SPEC-008 FR-23 response.

---

## FR-20: Benchmark Summary Retrieval Command

**Description:** `daedalus benchmark summary <benchmark_id>` calls `GET /v1/benchmarks/{benchmark_id}/summary` (SPEC-008 FR-24) and prints or saves the benchmark summary artifact.

**Flags:**

| Flag | Description |
|---|---|
| `--output <file>` | Write the summary JSON to this file. When absent, prints to stdout. |

**Text output format (summary not yet available):**

When SPEC-008 FR-24 returns HTTP 404 with `error_code = BENCHMARK_NOT_FOUND` (no member experiment has yet reached `Completed` status):
```
Benchmark summary for capacity-comparison-2026 is not yet available.
```

**Acceptance Criteria:**
- An unknown `benchmark_id` or a benchmark with no completed experiments returns HTTP 404 (`error_code = BENCHMARK_NOT_FOUND`); CLI prints the not-available message and exits with code 1.
- When `--output <file>` is provided, the summary JSON is written to that file and the file path is printed to stdout.
- `--output-format json` emits the raw SPEC-008 FR-24 response on HTTP 200, or the standard JSON error schema from FR-15 on HTTP 404.

---

## FR-21: Long-Running Session and Interruption Handling

**Description:** `daedalus experiment run` (FR-12) is a scoped exception to the CLI's general single-invocation posture. All other commands complete in seconds. `experiment run` may run for minutes to hours depending on experiment size and solver execution time.

**Session lifetime:** The CLI process remains alive throughout the trial orchestration loop. The wall-clock duration is bounded by `--timeout <seconds>` (default: 3600).

**Timeout behavior:** When the `--timeout` duration expires mid-experiment:
1. If in the submission phase: complete the current trial's two-step submission (job submit + trial link for the in-progress trial), then stop without submitting further trials (see FR-18).
2. If in the polling phase: exit at the next poll cycle boundary.
3. Print the current trial progress (submitted count, terminal count, pending count) and the `experiment_id`.
4. Exit with code 4. No result manifest is written.
5. All submitted trial jobs continue executing in the system. Experiment state is preserved in PostgreSQL (ADR-012 Decision 2).

**SIGINT / SIGTERM handling:** When the CLI receives SIGINT (Ctrl-C) or SIGTERM:
1. Print the `experiment_id` and last-known trial progress.
2. Exit with code 1. No result manifest is written.
3. Running trial jobs are not cancelled. The experiment state in the API is unaffected.

**Result manifest on interrupted exit:** No result manifest is written to disk on exit code 4 (timeout) or exit code 1 (signal). The developer retrieves current state via `experiment status <experiment_id>` and `experiment trials <experiment_id>`. The result manifest is written only when the CLI completes the orchestration loop normally (all trials terminal + experiment auto-completion confirmed).

**State preservation guarantee:** The API is the durable state authority (ADR-012 Decision 2). After CLI exit for any reason, the developer can query progress via `experiment status <experiment_id>` and `experiment trials <experiment_id>`. Full automatic CLI resumption of a partially-completed experiment is deferred to post-MVP implementation planning per ADR-012.

**Acceptance Criteria:**
- When `--timeout` expires mid-experiment, the CLI exits with code 4 after printing current trial counts and the `experiment_id`.
- When `--timeout` expires during the submission phase, the current trial's two-step submission is completed before exiting; the next trial's submission is not begun.
- When SIGINT is received, the CLI exits with code 1 after printing the `experiment_id` and progress state.
- No result manifest is written on exit code 4 or exit code 1.
- After CLI exit, experiment state is recoverable via `experiment status <experiment_id>` and `experiment trials <experiment_id>`.
- The CLI does not issue a cancellation request for the experiment or its trial jobs on interrupted exit.
- The `experiment run` long-running behavior does not apply to any other CLI command. All other commands are single-invocation.

---

# Non-Requirements

- No OTel span emission. The CLI is a developer tool, not an instrumented runtime component.
- No direct PostgreSQL access. All system interaction is through the Daedalus API (SPEC-008).
- No Worker spawning. The CLI does not start, stop, or manage Worker processes.
- No local solver execution. The CLI does not invoke solver backends directly.
- No RabbitMQ interaction. The CLI does not publish to or consume from RabbitMQ.
- No authentication or authorization. The CLI mirrors the API's MVP authentication posture (no auth at MVP scope — SPEC-008 Security Considerations).
- No report generation. The CLI retrieves reports from the API; it does not invoke the Report Generator.
- No problem versioning or amendment. Each `problem submit` is a new problem with a new `problem_id`.
- No multi-tenant support. The CLI assumes a single user/environment at MVP scope.
- No interactive terminal UI (TUI). The CLI is command-oriented; there is no interactive menu or live dashboard.
- No streaming solver output. Solver execution output is not available until the job completes.
- No webhook or push registration. The CLI polls; it does not register for push notifications.
- No batch job file (submitting many problems from a single invocation without the experiment model). Use `experiment run` for multi-job orchestration.
- No config file mutation commands. The CLI reads config files but does not write them.
- No shell completion generation at MVP scope.
- No automatic experiment resumption after CLI interruption. If `daedalus experiment run` exits before completion (timeout or signal), the developer must manually query state via `experiment status` and `experiment trials`. Automatic CLI resume of a partially-completed experiment is deferred to post-MVP per ADR-012.

---

# Assumptions

- The Daedalus API is reachable at the configured `api_url` before the CLI is invoked. The CLI does not wait for the API to become healthy.
- The Docker Compose environment is the expected deployment context for local development. The default `http://localhost:5000` API URL is consistent with the Docker Compose port mapping.
- The SPEC-002 generator is embedded as a library within the CLI binary. No separate generator process is required.
- The system clock on the machine running the CLI is approximately synchronized. Timestamps printed by the CLI are derived from API responses, not the local clock.
- The CLI is single-invocation per command, with one scoped exception: `daedalus experiment run` is a long-running process that may remain active for minutes to hours (FR-21). All other commands complete in a short, bounded time. The CLI does not run as a daemon or background process outside of `experiment run`.
- Report files produced by the Report Generator are HTML and are served verbatim by the API. The CLI does not attempt to parse or render HTML in the terminal.
- In the SPEC-020 experiment model (FR-12 Fixed mode), all solvers in the `solver_set` targeting the same `(problem_config_index, repetition_index)` share one `problem_id` (ADR-012 Decision 3 instance-sharing invariant). This is a deliberate design: shared problem instances are required for valid cross-solver `hindsight_quality` comparisons (SPEC-007 FR-7). This supersedes the previous assumption that each job submission produces a distinct `problem_id`.

---

# Constraints

- The CLI must not access PostgreSQL directly. All data access is through the SPEC-008 API.
- The CLI must not start or stop Docker Compose services or Worker processes.
- The CLI must not perform routing problem domain validation independently of the API. Local pre-validation is limited to structural checks (valid JSON, flag format validation). SPEC-001 domain validation authority is the API and Core (ADR-009).
- The SPEC-002 generator embedded in the CLI must use the same Earth radius constant (6371.0 km, SPEC-001 FR-5) and the same PRNG algorithm (PCG64, ADR-010 Decision 1) as the Core implementation. Divergence in generation logic between the CLI-hosted generator and the authoritative generator in Core (if Core also implements one) is a defect.
- The CLI must not expose `execution_seed`, `decision_id`, or internal solver execution identifiers in any output (per SPEC-008 FR-7 internal identifier isolation).
- The CLI binary must not require root or elevated privileges to execute.

---

# Inputs

| Input | Source | Format | Notes |
|---|---|---|---|
| Routing problem file | Developer, automation script | JSON per SPEC-001 FR-11 | `problem submit` and `problem run` (when `source: file`) |
| Generation config file | Developer, automation script | JSON per SPEC-002 FR-2 | `problem generate` and `problem run` |
| Experiment manifest | Developer, automation script | JSON per SPEC-020 FR-2 / SPEC-008 FR-18 schema | `experiment run` (FR-12) |
| Benchmark manifest | Developer, automation script | JSON per FR-17 schema | `benchmark submit` (FR-17) |
| Scheduler config file | Developer, automation script | JSON per FR-11 schema | `config create` |
| CLI flags | Command line | Per-command flag definitions | Override all other sources per FR-2 |
| Environment variables | Shell environment | `DAEDALUS_API_URL`, `DAEDALUS_LOG` | Configuration discovery per FR-2 |
| Project/user config file | Filesystem | YAML or JSON per FR-2 | Configuration discovery per FR-2 |
| API HTTP responses | Daedalus API | JSON per SPEC-008 | All commands that make API calls |

---

# Outputs

| Output | Consumer | Format | Notes |
|---|---|---|---|
| Command result | Developer, automation | Text or JSON per FR-13 | Written to stdout |
| Error messages | Developer, automation | Text or JSON per FR-15 | Text: stderr. JSON: stdout |
| Generated routing problem document | Developer, file system | JSON per SPEC-001 FR-11 | `--output` flag or stdout |
| Generation manifest | Developer, file system | JSON per SPEC-002 FR-8 | `--manifest` flag (FR-4); `--save-generation-manifest` flag (FR-5) |
| Report file | Developer, browser | `text/html` from API | `--output` flag or stdout |
| Experiment result manifest | Developer, CI | JSON | `<experiment_name>.result.json` (FR-12 `--output-manifest`) |
| Experiment summary artifact | Developer, CI | JSON per SPEC-008 FR-23 | `experiment summary --output <file>` (FR-19) |
| Benchmark summary artifact | Developer, CI | JSON per SPEC-008 FR-24 | `benchmark summary --output <file>` (FR-20) |
| Submission manifest | Developer, CI | JSON | `--save-manifest` flag |
| Debug log events | Developer | JSON lines on stderr | When `DAEDALUS_LOG=debug` |

---

# Failure Modes

### API Unreachable

**Cause:** The Daedalus API is not running, the `api_url` is wrong, or the network is unavailable.
**Expected behavior:** The CLI prints the URL it attempted to connect to and the network error. Exits with code 1.
**Expected fallback:** None. The caller must correct the `--api-url` or start the API.
**User-visible result:** `Error: could not connect to http://localhost:5000 — connection refused. Check --api-url or ensure the Docker Compose environment is running.`

---

### Routing Problem Validation Failure

**Cause:** The submitted routing problem fails SPEC-001 fast domain validation at the API layer (HTTP 400).
**Expected behavior:** The CLI prints all `field_errors` from the API response and exits with code 2.
**Expected fallback:** None. The caller must correct the routing problem and resubmit.
**User-visible result:** Structured field error list identifying each invalid field.

---

### Generation Failure (Capacity Infeasibility)

**Cause:** The SPEC-002 generator produces total stop demand exceeding total fleet capacity (SPEC-002 FR-4 construction self-check).
**Expected behavior:** Generation fails before any API call. The CLI prints the total demand, total capacity, and shortfall, and exits with code 3.
**Expected fallback:** None. The caller must adjust the generation config (`demand_max`, `vehicle_count`, `capacity_per_vehicle`, `stop_count`).
**User-visible result:** `Generation failed: total demand (432) exceeds total capacity (400). Adjust demand_max, vehicle_count, or capacity_per_vehicle.`

---

### Generation Failure (Time Window Achievability)

**Cause:** A time-windowed stop is not individually reachable from the depot under the configured speed (SPEC-002 FR-5 Speed-Dependent Achievability construction self-check).
**Expected behavior:** Generation fails before any API call. The CLI prints the offending stop index, computed travel time, and `time_window_close` value. Exits with code 3.
**Expected fallback:** None. The caller must adjust `average_vehicle_speed_kmh`, `bounding_box`, `time_window_width_seconds`, or `planning_horizon_seconds`.
**User-visible result:** `Generation failed: stop 7 is not reachable from the depot. Computed travel time: 3820 seconds, time_window_close: 3600 seconds. Adjust speed, bounding box, or time window parameters.`

---

### Job Wait Timeout

**Cause:** A job did not reach a terminal state before the `--timeout` duration elapsed.
**Expected behavior:** The CLI prints the elapsed time and the last known status. Exits with code 4. The job continues executing.
**Expected fallback:** The caller may re-issue `job wait <job_id>` with a longer timeout, or poll manually with `job status`.
**User-visible result:** `Wait timeout: job a1b2c3d4-... has not completed after 300 seconds. Current status: Processing. Use 'daedalus job wait --timeout <seconds>' to continue waiting.`

---

### Report Not Available

**Cause:** The job has no report (`report_available = false`), or the report generation failed.
**Expected behavior:** The CLI prints a message explaining that no report is available. Exits with code 1.
**Expected fallback:** The caller may check job status to determine if the job is still executing.
**User-visible result:** `No report available for job a1b2c3d4-.... The job may still be executing or report generation may have failed.`

---

### Experiment Partial Failure (Trial-Level)

**Cause:** One or more trial jobs in a `daedalus experiment run` reach `status = Failed` (Worker lifecycle failure).
**Expected behavior:** Evidence collection is still attempted for failed trial jobs. The experiment summary table notes the failed trials. The CLI exits with code 5. Successfully completed trials are still reported.
**Expected fallback:** Investigate the failed trial jobs individually using `job status <job_id>`.
**User-visible result:** Experiment summary table with per-trial status; exit code 5.

---

### Experiment Infrastructure Failure

**Cause:** The experiment itself transitions to `status = Failed` at the API level (e.g., a persistent infrastructure failure preventing trial orchestration from completing).
**Expected behavior:** The CLI prints the experiment failure status and exits with code 6. Code 6 takes precedence over code 5 when both conditions are present.
**Expected fallback:** Query `experiment status <experiment_id>` for API-level error context.
**User-visible result:** `Experiment 7f8a9b0c-... transitioned to Failed. Use 'daedalus experiment status <experiment_id>' for details.`

---

### Experiment Run Interrupted

**Cause:** The CLI process is interrupted (SIGINT, SIGTERM) or the `--timeout` duration expires before all trials complete.
**Expected behavior:** The CLI prints the `experiment_id` and current progress, then exits with code 1 (signal) or code 4 (timeout). Trial jobs in flight are not cancelled. Experiment state in the API is preserved.
**Expected fallback:** Query `experiment status <experiment_id>` and `experiment trials <experiment_id>` to review current state. Resume or re-attempt is a post-MVP capability.
**User-visible result:** `Experiment run interrupted. experiment_id: 7f8a9b0c-.... Submitted: 8/12 trials. Completed: 5/12. Use 'daedalus experiment status' to check progress.`

---

### Experiment Manifest Rejected

**Cause:** The submitted experiment manifest is rejected by the API with HTTP 400 (invalid field values, unknown `problem_id`, unregistered `backend_id`, etc.).
**Expected behavior:** The CLI prints all `field_errors` from the API response and exits with code 2. No trial jobs are submitted.
**Expected fallback:** Correct the manifest and resubmit.
**User-visible result:** Structured field error list identifying each invalid field.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer | Indirect — CLI embeds SPEC-002 generator; generator uses SPEC-001 domain model |
| API Layer | Yes — CLI wraps SPEC-008 FR-2 (job submit), FR-8/FR-9/FR-11 (job lifecycle), FR-10 (scheduler config), FR-12/FR-13 (reports), FR-18 (experiment manifest submission), FR-19 (benchmark manifest submission), FR-20 (experiment status), FR-21 (trial evidence collection), FR-22 (trial results), FR-23 (experiment summary), FR-24 (benchmark summary), FR-25 (trial submission linkage). No API endpoints beyond SPEC-008 are required. |
| Persistence | None — CLI does not access PostgreSQL |
| Solver Runtime | None — CLI does not invoke solvers |
| Simulation Framework | Yes — CLI is the primary entry point for the simulation framework (SPEC-002 generator host) |
| Observability | Minimal — no OTel; CLI debug logs are development-only |
| Security | Minimal — CLI is the external client; API is the trust boundary |
| Configuration | Yes — CLI introduces a config discovery protocol (`daedalus.yaml`, `DAEDALUS_API_URL`) |

**SPEC-002 OQ-2 resolution:** SPEC-016 resolves SPEC-002's open question about generator process location. The generator is a library embedded in the CLI binary. This is the "CLI layer" option from SPEC-002 OQ-2 candidate list: "a subcommand of the Daedalus CLI — generator logic in the CLI layer; produces a JSON document without full domain type access." This does not require a revision to SPEC-002 beyond noting the resolution.

**SPEC-008 endpoint footprint:** The amendment added the experiment and benchmark command groups (FR-12, FR-17 through FR-21), which wrap SPEC-008 FR-18 through FR-25. The Architectural Impact table above reflects the full endpoint footprint across both the original SPEC-016 commands and the ADR-012 amendment. The CLI's `job list` command (FR-9) requires a `GET /v1/jobs` list endpoint not yet defined in SPEC-008; `job list` is deferred per OQ-1.

---

# Testability

The following behaviors must be proven before this feature is considered complete.

## Command Invocation Behaviors

- `daedalus --help` exits with code 0 and prints the command inventory.
- An unrecognized command group exits with code 1.
- An unrecognized command within a recognized group exits with code 1.
- Global flag `--output-format json` produces valid JSON on stdout for every command.
- Global flag `--no-color` suppresses ANSI escape codes in text output.

## Configuration Discovery Behaviors

- When `--api-url` is set, it overrides `DAEDALUS_API_URL` and any config file entry.
- When `DAEDALUS_API_URL` is set and `--api-url` is absent, the environment variable is used.
- When a `daedalus.yaml` file is present in the current directory, it is used when no flag or environment variable overrides it.
- When no config source provides an `api_url`, `http://localhost:5000` is used.

## Problem Submit Behaviors

- A valid routing problem JSON file produces HTTP 202 and prints `job_id`, `problem_id`, and `status`.
- A routing problem JSON with `average_vehicle_speed_kmh` absent is rejected with HTTP 400; the CLI prints the `field_errors` and exits with code 2.
- `--scheduler-config` with a valid `config_id` is sent in the submission; the status response reflects the provided `config_id`.
- `--wait` causes the CLI to poll until the job is terminal.
- `--save-manifest` writes the HTTP 202 response body as JSON to the named path.

## Workload Generation Behaviors

- A valid generation config file produces a routing problem document and manifest.
- A generation config with `average_vehicle_speed_kmh` absent exits with code 3 before any API call.
- A generation failure (capacity infeasibility) exits with code 3 and prints total demand and total capacity.
- A time window achievability failure exits with code 3 and identifies the offending stop, computed travel time, and `time_window_close`.
- `--seed` overrides the seed in the config file; the produced document carries the override seed.
- The produced routing problem document passes SPEC-001 API validation when submitted without modification.

## Job Status and Wait Behaviors

- `job status <job_id>` for an existing job prints the current status.
- `job status <job_id>` for a non-existent `job_id` exits with code 1.
- `job wait <job_id>` polls until terminal state and exits with code 0 (`Completed`) or 5 (`Failed`).
- `job wait <job_id>` with `--timeout 5` exits with code 4 after 5 seconds if no terminal state is reached.
- A waited job whose solver outcome is `Timeout` exits the CLI with code 0 (solver timeout is not a CLI failure).
- A waited job whose lifecycle state is `Failed` exits the CLI with code 5.

## Cancellation Behaviors

- `job cancel <job_id>` for a non-terminal job prints the 202 acknowledgement and exits with code 0.
- `job cancel <job_id>` for a terminal job prints the 409 conflict and exits with code 1.
- `job cancel <job_id>` for a non-existent job exits with code 1.

## Report Behaviors

- `report show <job_id>` for a job with `report_available = true` writes the HTML file to the specified output.
- `report show <job_id>` for a job with `report_available = false` exits with code 1.
- When output goes to stdout (either by default or via `--output -`), only the HTML content appears on stdout; all progress messages and metadata go to stderr.
- When `--output <file>` names a file, the metadata summary and "Saved to:" line appear on stdout alongside the file write.

## Config Command Behaviors

- `config create` with a valid config file returns `201 Created` and prints the new `config_id`.
- `config create` with missing `objective_mode` exits with code 2.
- `config list` includes the default configuration.
- `config get` with a known `config_id` prints the configuration.
- `config get` with an unknown `config_id` exits with code 1.
- `config default` prints the resolved `api_url` and the source that produced it (flag, environment variable, project config file, user config file, or built-in default).
- `config default` prints the API's default scheduler configuration including its `config_id` and `objective_mode`.
- `config default` exits with code 1 when the API is unreachable (the resolved `api_url` is always printed before the API call, so the URL source is visible even on failure).

## Output Format Behaviors

- `--output-format json` produces valid parseable JSON on stdout for every command, on both success and error paths.
- `--output-format json` produces no decorative text on stdout.
- In text format, errors appear on stderr. In JSON format, errors appear on stdout.
- `--quiet` suppresses intermediate polling output in `job wait`.
- `--no-color` produces text output with no ANSI escape code sequences; status values and headings appear as plain text.

## Debug Log Behaviors

- When `DAEDALUS_LOG=debug` is set, a `cli.command.start` event appears on stderr before any API call is made.
- When `DAEDALUS_LOG=debug` is set and an HTTP request is issued, a `cli.api.request` event appears on stderr containing `method`, `url`, and `body_size_bytes`.
- When `DAEDALUS_LOG=debug` is set and an HTTP response is received, a `cli.api.response` event appears on stderr containing `status_code` and `latency_ms`.
- When `DAEDALUS_LOG=debug` is set, a `cli.command.exit` event appears on stderr after the command exits, containing `exit_code` and `duration_ms`.
- When `DAEDALUS_LOG` is unset, no debug log events appear on stderr.
- When `DAEDALUS_LOG` is set to any value other than `debug`, no debug log events appear on stderr.
- All debug log events are valid JSON, one event per line.
- Debug log events do not include routing problem coordinate arrays.

## Exit Code Behaviors

- Every command exits with exactly one of the defined codes (0–6).
- Exit code 0 for successful submission (no `--wait`).
- Exit code 0 for a waited job that completes with `solver_outcome = Infeasible`.
- Exit code 0 for a waited job that completes with `solver_outcome = Timeout`.
- Exit code 0 for a waited job that completes with `solver_outcome = Cancelled`.
- Exit code 0 for a waited job that completes with `solver_outcome = Failed` (solver-level failure within a `Completed` job; distinct from `status = Failed`).
- Exit code 0 for a waited job that completes with `solver_outcome = ContractViolation`.
- Exit code 5 only for a job that reaches `status = Failed` (Worker lifecycle failure).
- Exit code 6 only when `experiment.status = Failed` (experiment-level infrastructure failure). Code 6 takes precedence over code 5.
- Exit code 4 when `experiment run --timeout` expires before experiment reaches terminal state.
- Exit code 2 for any API validation error (HTTP 400).
- Exit code 3 for any generator construction self-check failure.

## Experiment Behaviors

- A valid Fixed mode manifest submits to `POST /v1/experiments` and enters the trial orchestration loop.
- A manifest specifying `"mode": "generated"` exits with code 1 before any API call is issued.
- A manifest with an empty `solver_set` exits with code 2.
- A manifest rejected by the API (HTTP 400) prints all `field_errors` and exits with code 2.
- Trials are submitted in `(problem_config_index asc, repetition_index asc, solver_set_index asc)` order.
- `POST /v1/jobs` returning HTTP 5xx during the submission phase causes the CLI to exit with code 1; the trial-link step is not attempted for the failed trial.
- `POST /v1/jobs` network failure during the submission phase is retried once; on a second consecutive failure for the same trial, the CLI exits with code 1.
- `POST /v1/experiments/{id}/trials/{trial_id}/submit` returning HTTP 5xx (after a successful job submit) is retried up to 3 consecutive attempts; on the third consecutive 5xx, the CLI exits with code 1.
- `TRIAL_ALREADY_SUBMITTED` (HTTP 409) from the trial-link endpoint is treated as success-on-retry: the CLI calls `GET /v1/experiments/{id}/trials` to retrieve the already-linked `job_id` and continues to the next trial; it does not exit.
- Evidence collection is triggered for each trial immediately when its job reaches a terminal state.
- When collect-evidence returns HTTP 200 with `evidence_status = Error`, the CLI records the outcome in the result manifest and completion table and does not exit; the overall exit code is not affected by `evidence_status = Error`.
- When one trial's job reaches `status = Failed`, the CLI exits with code 5 and reports all other trial outcomes.
- When `experiment.status = Failed`, the CLI exits with code 6.
- In `--output-format json` mode, no progress text appears on stdout during the orchestration loop.
- In `--output-format json` mode, on exit codes 0 and 5, a single JSON result object is emitted to stdout.
- In `--output-format json` mode, on exit code 4 (timeout) or 1 (signal), the FR-15 JSON error schema is emitted to stdout; the result object is not emitted.
- In `--output-format json` mode, on exit code 6 (`experiment.status = Failed`), the JSON result object is emitted with `"status": "Failed"`.
- The experiment result manifest (`<experiment_name>.result.json`) contains: `experiment_id`, `experiment_name`, `benchmark_id` (if present), planned trial count, and per-trial `trial_id`, `problem_id`, `job_id`, `backend_id`, `trial_status`, `solver_outcome` (when available), and `evidence_status`.
- If the output manifest file already exists from a prior run, the CLI overwrites it without prompting.
- `experiment status <experiment_id>` for an unknown experiment exits with code 1.
- `experiment trials <experiment_id>` displays `—` for trials with `job_id = null`.
- `experiment summary <experiment_id>` for an experiment whose status is not `Completed` exits with code 1.
- A valid `benchmark submit` manifest (all 8 required fields present and non-empty; `controls`, `independent_variables`, and `dependent_variables` arrays each containing at least one element) produces HTTP 201 and prints `benchmark_id`, `benchmark_name`, and `created_at`; exits with code 0.
- `benchmark submit` with a manifest missing any required field (`benchmark_id`, `benchmark_name`, `research_question`, `hypothesis`, `null_hypothesis`, `controls`, `independent_variables`, or `dependent_variables`) exits with code 2 and prints all `field_errors`.
- `benchmark submit` with an empty string value for any required string field exits with code 2 and prints all `field_errors`.
- `benchmark submit` with an empty array for `controls`, `independent_variables`, or `dependent_variables` exits with code 2 and prints all `field_errors`.
- `benchmark submit` with `--output-format json` emits the raw SPEC-008 FR-19 response body as a JSON object to stdout on HTTP 201.
- `benchmark submit` with a duplicate `benchmark_id` exits with code 1.
- `benchmark summary <benchmark_id>` for an unknown benchmark exits with code 1.
- After CLI exit (SIGINT or timeout), `experiment status <experiment_id>` returns the preserved experiment state.
- Under `DAEDALUS_LOG=debug`, a `cli.experiment.trial_submit` event appears for each trial submitted in the loop.
- Under `DAEDALUS_LOG=debug`, a `cli.experiment.evidence_collect` event appears for each trial when evidence collection is triggered; the event includes the `evidence_status` value from the collect-evidence response.

---

# Observability Requirements

The CLI is a developer tool. No Prometheus metrics and no OTel spans are required.

The following questions must be answerable by a developer running the CLI:

1. What API URL is the CLI using for this invocation? (Answered by `config default` command and `--api-url` flag behavior.)
2. What HTTP requests did the CLI issue and what responses did it receive? (Answered by `DAEDALUS_LOG=debug` output.)
3. What was the generation configuration for a submitted problem? (Answered by `--manifest` in `problem generate` or `--save-generation-manifest` in `problem run`, which capture generation parameters.)
4. What were the final outcomes for all jobs in an experiment? (Answered by the experiment result manifest and the summary table.)

---

# Security Considerations

**External trust boundary:** The CLI is a trusted developer tool operating within the same trust zone as the Docker Compose environment. The API is the external trust boundary for routing problem validation. The CLI does not need to independently validate routing problem domain rules beyond ensuring inputs are structurally parseable.

**No authentication at MVP scope:** The CLI does not include authentication credentials, API keys, or tokens. This mirrors the API's MVP security posture (SPEC-008 Security Considerations). Authentication and authorization must be added to both the API and CLI before any exposure to untrusted networks.

**Config file sensitivity:** `daedalus.yaml` and `~/.daedalus/config.json` may contain the API base URL. At MVP scope (localhost), this is not sensitive. In environments with non-trivial API URLs, the config file should be excluded from source control. The CLI does not write credential material to config files.

**Internal identifier isolation:** The CLI must not surface `execution_seed`, `decision_id`, or solver run record identifiers, consistent with SPEC-008 FR-7. These do not appear in API responses and are therefore naturally excluded from CLI output.

**Log safety:** CLI debug log events must not include routing problem coordinate arrays or full stop lists, consistent with SPEC-001, SPEC-005, and SPEC-008 log safety requirements.

**Report file serving:** When `report show` writes an HTML file to the local filesystem, the file path is determined by the `--output` flag value, which comes from the developer. The CLI does not construct file paths from API-returned data. The `report_id` from the API is used only as a query parameter in the API URL, not in filesystem path construction.

---

# Performance Considerations

**Polling interval:** The default `--poll-interval 2` seconds for `job wait` and `experiment run` is a balance between responsiveness and API load. Polling more frequently than once per second is discouraged for automation scripts running many concurrent experiments.

**Experiment trial submission:** `experiment run` submits all trial jobs sequentially, not concurrently. Concurrent submission is not required at MVP scope and simplifies error handling and submission-order guarantees (SPEC-020 FR-3).

**Experiment duration:** `experiment run` is a long-running command. Wall-clock duration is proportional to `planned_trial_count × execution_timeout_ms_per_trial`. The `--timeout` flag (default: 3600 seconds) bounds the CLI process lifetime, not individual trial execution. Experiments requiring more than one hour of total execution should increase `--timeout` accordingly.

**Generator performance:** The SPEC-002 generator is expected to complete in milliseconds at all MVP stop counts. No CLI-level timeout is required for generation.

**Report download size:** HTML evidence reports are bounded by the report content defined in SPEC-009. At MVP scope, report file sizes are not expected to be a performance concern. Large reports should be written to file with `--output <file>` rather than to stdout to avoid terminal rendering overhead.

---

# Documentation Updates Required

- **README.md**: Add a CLI Quick Start section demonstrating: `docker compose up`, `daedalus problem run <config>`, `daedalus report show <job_id>`. This is the primary portfolio demonstration path. Also add a Benchmarking Quick Start demonstrating: `daedalus benchmark submit <manifest>`, `daedalus experiment run <manifest>`, `daedalus experiment summary <id>`.
- **docs/architecture.md**: The System Context diagram shows `CLI → API`. SPEC-016 is the authoritative definition of the CLI component. Update the CLI description to note that it is the experiment orchestration executor per ADR-012 Decision 1.
- **SPEC-002 OQ-2**: Mark as resolved. The generator is embedded in the CLI binary (CLI layer option). No SPEC-002 revision is required; the resolution is noted in SPEC-016 FR-4 and the Architectural Impact section.
- **SPEC-008 Documentation Updates Required**: A CLI Specification reference was listed as pending. SPEC-016 satisfies this item. SPEC-008 FR-2 is the HTTP contract the CLI wraps. SPEC-008 Related Specs should be updated to include SPEC-016.
- **ADR-012**: SPEC-016 amendment satisfies the SPEC-016-R1 requirements listed in ADR-012 Consequences. Mark SPEC-016-R1 items as resolved in ADR-012 Implementation Tracking (if that section exists) or note completion in a future ADR-012 revision.

---

# Open Questions

### OQ-1: `GET /v1/jobs` List Endpoint

**Question:** Does the Daedalus API expose a `GET /v1/jobs` list endpoint that the `job list` command (FR-9) can consume?

**Ownership clarification:** SPEC-016 is not the authoritative owner of this endpoint. The CLI is a consumer: it expresses a requirement that the API expose a job listing endpoint, but the endpoint contract, schema, and implementation belong exclusively to SPEC-008. Resolving this question requires a future SPEC-008 revision. SPEC-016 does not define the endpoint; it documents the CLI's minimum consumer requirements.

**CLI consumer requirements (minimum fields required in the API response):**
- `job_id` — job correlation handle
- `status` — current lifecycle state (`Pending`, `Processing`, `Completed`, `Failed`)
- `solver_outcome` — present when `status = Completed`; absent otherwise
- `created_at` — ISO 8601 UTC timestamp

SPEC-016 also requires support for optional `?status=` and `?limit=` query parameters to support the `--status` and `--limit` flags in FR-9. The complete endpoint schema — including response field names, ordering, pagination, and error behavior — is owned by SPEC-008 and will be defined in a future SPEC-008 revision.

**Why it matters:** SPEC-008 does not include a `GET /v1/jobs` list endpoint. The `job list` command requires this endpoint. Without it, `job list` cannot be implemented and must remain deferred.

**Resolution path:** A future SPEC-008 revision adds `GET /v1/jobs` to the endpoint inventory (FR-15) and defines its full contract. The CLI consumer requirements above are the minimum SPEC-008 must satisfy for SPEC-016's `job list` command to be implementable.

**Blocking:** Blocking for `job list` implementation only. Not blocking for any other CLI command. `job list` should remain deferred in the Definition of Done until OQ-1 is resolved via SPEC-008 revision.

---

### CLI Implementation Language

The Daedalus CLI is implemented in C++.

The CLI shares the routing problem model, synthetic workload generator, and ADR-010 reproducibility implementation with the Core runtime.

This decision eliminates duplicate implementations of PCG64, approved distribution algorithms, and seed derivation behavior.

---

### OQ-3: Experiment Result Persistence *(Superseded)*

**Status: Superseded by ADR-012 Decision 2.**

This question asked whether experiment results should be persisted beyond the local CLI filesystem. ADR-012 Decision 2 resolves it: the API is the durable state authority for all experiment state. Experiment records, trial records, and artifacts are persisted in PostgreSQL via the SPEC-008 / SPEC-012 data model. The CLI's local result manifest (`<experiment_name>.result.json`) is an additional convenience output; it is not the authoritative source. State is recoverable via `experiment status` and `experiment trials` after any CLI exit.

**No further action required on this question.**

---

### OQ-4: `--wait` Default Behavior

**Question:** Should `problem submit` and `problem run` enable `--wait` by default?

**Why it matters:** Without `--wait`, the CLI returns immediately after submission and requires the developer to explicitly poll. This is the correct automation-friendly default (submit-and-continue). However, for interactive use, "fire and forget" requires a follow-up `job wait` or `job status` command to see results, which increases friction for the most common portfolio demonstration path (`run → report`).

**Options:**
- (a) `--wait` is opt-in (current spec). Clean automation behavior.
- (b) `--wait` is the default for `problem run`, opt-out with `--no-wait`. Optimizes for interactive demonstration use.

**Recommendation:** Option (b) for `problem run`, option (a) for `problem submit`. `problem run` is the interactive demonstration path; `problem submit` is the automation path.

**Owner:** Project Owner decision. Either option is implementable without changing the rest of the spec.

**Blocking:** Not blocking for specification acceptance. The current spec specifies option (a) as the default.

---

### OQ-5: Generated Mode Implementation (SPEC-008 OQ-7 Dependency)

**Question:** When and how will Generated Mode workload sets (SPEC-020 FR-5, SPEC-008 FR-18) be implemented in the CLI?

**Why it matters:** `daedalus experiment run` (FR-12) currently rejects manifests with `"mode": "generated"` at MVP scope. Generated Mode requires a mechanism for the CLI (or API) to create routing problems on behalf of the experiment at manifest submission time. This mechanism is not yet defined.

**Dependency:** Resolution of SPEC-008 OQ-7 (generated workload set routing problem creation mechanism). SPEC-016 cannot implement Generated Mode until SPEC-008 OQ-7 is resolved and the API contract for it is defined.

**MVP position:** Generated Mode is deferred. The CLI exits with code 1 on a `"mode": "generated"` manifest.

**Owner:** SPEC-008 owns the API contract for generated workload sets. SPEC-016 amendment required once SPEC-008 OQ-7 is resolved.

**Blocking:** Blocking for Generated Mode CLI support only. Not blocking for Fixed Mode or any other CLI command.

---

### OQ-6: Per-Backend Job Targeting Mechanism

**Question:** How does `daedalus experiment run` ensure that the job submitted for a specific trial runs on the backend identified by the trial's `backend_id`?

**Why it matters:** `POST /v1/jobs` (SPEC-008 FR-2) accepts `problem_id` and `scheduler_config_id` but no `backend_id`. For an experiment with `solver_set: ["backend-vrp-greedy-v1", "backend-vrp-metaheuristic-v1"]`, the CLI submits two jobs per (problem, repetition) combination. With a single experiment-wide `scheduler_config_id`, the Scheduler selects the backend by policy for both jobs — there is no mechanism to guarantee that the job for the `backend-vrp-greedy-v1` trial actually runs on that backend. This renders multi-backend experiments non-deterministic with respect to which backend produces which trial's evidence.

SPEC-008 FR-18 acknowledges this as "Backend capability-profile validation gap (F-13)." SPEC-008 FR-25 (Trial Submission Linkage) validates `problem_id` alignment but not `backend_id` alignment. Without a resolution, a multi-backend experiment produces results where the per-backend attribution is unverifiable.

**Candidate resolutions (for SPEC-008-R1 scope):**
- (a) Extend `POST /v1/jobs` with an optional `backend_id` field for experiment context that directs the Scheduler to a specific backend.
- (b) Require each trial to use a per-backend `scheduler_config_id` (one config per backend in the `solver_set`) instead of a single experiment-wide config.
- (c) Add backend-identity validation to SPEC-008 FR-25 (Trial Submission Linkage), rejecting the link if the job's actual backend (from the Scheduler decision record) does not match the trial's `backend_id`.
- (d) Accept that backend targeting is best-effort at MVP and rely on Scheduler determinism for single-backend deployments; document this as an accepted risk.

**Owner:** SPEC-008 owns the `POST /v1/jobs` contract and FR-25 validation. Resolution requires a SPEC-008 amendment. SPEC-016 cannot unilaterally resolve this; FR-18 documents the dependency.

**Blocking:** Blocking for correct multi-backend experiment implementation. Fixed Mode experiments with a single-backend `solver_set` are unaffected. The CLI can submit and link trials without resolution; correctness of per-backend attribution is the gap.

---

# Acceptance Checklist

- [x] Problem is clearly defined.
- [x] Scope and responsibility boundary are defined.
- [x] Requirements are testable.
- [x] Non-requirements are documented.
- [x] Assumptions are explicit.
- [x] Failure modes are defined.
- [x] Observability requirements exist.
- [x] Security considerations exist.
- [x] Documentation updates are identified.
- [x] Exit codes are defined (codes 0–6).
- [x] Output formats are defined for both text and JSON modes.
- [x] SPEC-002 OQ-2 (generator process location) is resolved.
- [x] OQ-3 resolved — experiment result persistence superseded by ADR-012 Decision 2 (API is durable state authority).
- [x] ADR-012 CLI requirements (SPEC-016-R1) applied: benchmark command group added; long-running exception documented; OQ-3 superseded; problem-sharing assumption updated.
- [x] SPEC-020 orchestration model applied: SPEC-020 manifest format, trial submission order, evidence collection trigger, instance-sharing invariant documented.
- [x] Trace propagation posture documented (ADR-011, ADR-012): CLI does not inject trace context; API owns trace roots.
- [ ] OQ-1 resolved — `GET /v1/jobs` list endpoint added to SPEC-008 via future revision (blocking for `job list` command only; endpoint contract owned by SPEC-008, not SPEC-016).
- [x] OQ-2 resolved — CLI is implemented in C++ (ADR-001, ADR-010).
- [ ] OQ-4 resolved — `--wait` default behavior (non-blocking; current spec uses opt-in).
- [ ] OQ-5 resolved — Generated Mode implementation (depends on SPEC-008 OQ-7; non-blocking for Fixed Mode).
- [ ] OQ-6 resolved — per-backend job targeting mechanism (blocking for correct multi-backend experiments; requires SPEC-008-R1 amendment).

---

# Definition of Done

This feature is complete when:

- All commands defined in FR-1 are implemented and produce correct output in both `text` and `json` formats.
- Configuration discovery (FR-2) correctly resolves `api_url` from all four sources in the documented priority order.
- `problem submit` (FR-3) submits a routing problem to the API and surfaces validation errors per SPEC-008 FR-14.
- `problem generate` (FR-4) invokes the SPEC-002 generator and produces a conforming routing problem document that passes API submission without modification.
- `problem run` (FR-5) generates and submits in a single command.
- `job status`, `job wait`, and `job cancel` (FR-6, FR-7, FR-8) correctly interact with SPEC-008 FR-8, FR-11.
- `report show` (FR-10) retrieves and writes the HTML report via SPEC-008 FR-12, FR-13.
- `config create`, `config list`, `config get`, `config default` (FR-11) correctly interact with SPEC-008 FR-10.
- `experiment run` (FR-12) accepts a SPEC-020 Fixed mode manifest, submits it to `POST /v1/experiments`, drives the trial orchestration loop (FR-18), collects evidence per trial, confirms experiment auto-completion, and writes the result manifest.
- `experiment run` rejects Generated mode manifests with exit code 1 (OQ-5 dependency).
- `experiment run` exits with code 6 on `experiment.status = Failed`; code 5 on any trial `status = Failed`; code 4 on `--timeout` expiry.
- `benchmark submit` (FR-17) submits a benchmark manifest and prints the response.
- `experiment status`, `experiment trials`, `experiment summary` (FR-19) retrieve experiment state from the API.
- `benchmark summary` (FR-20) retrieves the benchmark summary artifact.
- Long-running session and interruption handling (FR-21) is verified: SIGINT exits with code 1; timeout exits with code 4; experiment state is preserved in the API after CLI exit.
- Experiment observability events (FR-16) are emitted under `DAEDALUS_LOG=debug` for each trial submission and evidence collection.
- All exit codes (FR-14, codes 0–6) are consistent across command paths and verified by testability behaviors.
- All error outputs (FR-15) are structured and do not expose internal details.
- The CLI is implemented in C++ (OQ-2 resolved).
- `job list` (FR-9) is either implemented (if OQ-1 is resolved) or explicitly deferred with a note.
- The generated routing problem document produced by `problem generate` passes SPEC-001 domain validation when submitted to the API.
- The portfolio demonstration path — `daedalus problem run <config> --wait` followed by `daedalus report show <job_id>` — completes end-to-end against a running Docker Compose environment.
- The benchmark demonstration path — `daedalus benchmark submit <manifest>`, `daedalus experiment run <manifest>`, `daedalus experiment summary <id>` — completes end-to-end against a running Docker Compose environment using a single-backend `solver_set` (OQ-6 does not affect single-backend experiments).
- OQ-6 (per-backend job targeting) is either resolved via SPEC-008-R1 or explicitly accepted as a risk for multi-backend experiments, with the limitation documented.
- Engineering review passes.
- Architecture review passes.
- Specification status is updated to Verified.

---

# Future Considerations

The following concerns are not MVP requirements. They are documented here so that future specifications can account for them without revisiting the architecture of this specification from scratch.

## Generator Embedding Migration

SPEC-016 FR-4 resolves SPEC-002 OQ-2 by embedding the SPEC-002 generator as an in-process library in the CLI binary. This is the correct MVP decision. The generator is invoked by the developer at a terminal, not by a networked client, and the CLI binary is the appropriate host for that use case.

If any of the following become requirements in future work, the generator hosting architecture should be reconsidered:

- **Dashboard generation:** A web dashboard that generates synthetic workloads on behalf of a browser-based user would not be able to invoke the CLI binary. The generator would need to be exposed as a SPEC-008 API endpoint or moved to the Core library.
- **CI/CD pipeline generation:** Automated CI pipelines that generate workloads server-side (rather than running the CLI on a build agent) would have the same constraint.
- **API-mediated generation:** If the system needs to generate workloads programmatically from a service (for example, a benchmark orchestrator running inside Docker Compose), direct library embedding in the CLI binary is inaccessible.
- **Multi-client generation:** If multiple clients (CLI, dashboard, mobile, CI) all need access to the generator, moving the generator to a server-side endpoint or a shared service would eliminate per-client reimplementation risk.

None of these scenarios are in scope for the MVP. README.md explicitly defers the full dashboard. This note is recorded so that the architectural consequence of the current embedding decision is visible when dashboard or multi-client generation work begins.

**No changes to OQ-2, FR-4, or the Constraints section are required as a result of this note.**
