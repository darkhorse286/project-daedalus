# Feature Specification

## Metadata

**Feature ID:** SPEC-016

**Title:** Daedalus Command Line Interface

**Status:** Draft

**Author:** Darkhorse286

**Created:** 2026-06-20

**Last Updated:** 2026-06-20 (Revised post-architecture-review: NB-1 OQ-1 ownership clarification, NB-2 generator embedding migration note)

**Supersedes:** None

**Superseded By:** None

**Related ADRs:** ADR-002, ADR-003, ADR-004, ADR-006, ADR-009

**Related Specs:** SPEC-001, SPEC-002, SPEC-003, SPEC-005, SPEC-008, SPEC-009

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
| Wrapping SPEC-008 API calls for developer and automation use | **SPEC-016 (this spec)** |
| Hosting the SPEC-002 workload generator (OQ-2 resolution) | **SPEC-016 (this spec)** |
| Structured output formats for human and machine consumers | **SPEC-016 (this spec)** |
| Experiment orchestration (multi-job submission and result collection) | **SPEC-016 (this spec)** |

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
| `experiment` | Multi-job experiment orchestration |

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
| `daedalus experiment run <file>` | Run a named experiment: submit a problem with multiple configurations and collect results |

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

**Flags:** All flags from FR-3 (submit) and FR-4 (generate) apply, with one disambiguation: in this combined command, `--save-manifest` retains its FR-3 meaning (write the API submission response body) and a distinct flag is provided for the generation manifest. Additionally:

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
4. If `--open`: open the file in the system default browser.

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
- `--open` does not fail the command if the system browser cannot be located; the file is saved regardless and a warning is printed to stderr.
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

**Description:** `daedalus experiment run <experiment_file>` submits a routing problem (or generates one) with multiple scheduler configurations and collects results for all submitted jobs. This command supports demonstrating the project thesis by comparing scheduler decisions across objective modes.

**Experiment file format:**

```json
{
  "name": "capacity-only-baseline",
  "description": "Compare CheapestValid, Balanced, and BestQuality on a small capacity-only problem.",
  "problem": {
    "source": "generate",
    "generation_config": "./configs/small-capacity-only.json"
  },
  "scheduler_configs": [
    { "config_id": "c1d2e3f4-..." },
    { "config_id": "d2e3f4g5-..." },
    {
      "inline": {
        "objective_mode": "BestQuality"
      }
    }
  ],
  "wait": true,
  "wait_timeout_seconds": 600
}
```

**Problem source options:**
- `"source": "generate"` with `generation_config`: generate a problem from a SPEC-002 config file.
- `"source": "file"` with `"path": "./problem.json"`: read a pre-authored problem file.

**Scheduler config resolution:**
- `config_id`: references an existing stored configuration.
- `inline`: creates a new scheduler configuration from the inline definition before submission.

**Normal behavior:**
1. Resolve the routing problem (generate or load from file).
2. For each scheduler config entry:
   a. If `inline`: create the configuration via `POST /v1/scheduler-configs`; capture the new `config_id`.
   b. Submit the routing problem with the resolved `config_id` via `POST /v1/jobs`.
   c. Capture the `job_id`.
3. If `wait: true`: wait for all submitted jobs to reach a terminal state (polling FR-7 behavior, bounded by `wait_timeout_seconds`).
4. Print the experiment summary table.
5. Write the experiment result to a JSON manifest file (default: `<experiment_name>.result.json`). If this file already exists, it is overwritten without prompt. To preserve prior results, callers must rename or relocate the existing file before re-running the experiment.

**Text output format (after all jobs complete):**
```
Experiment: capacity-only-baseline
  problem_id: e5f6g7h8-...
  generated:  2026-06-20T10:00:00Z

CONFIG ID         OBJECTIVE MODE  JOB ID          STATUS     SOLVER OUTCOME  REPORT
c1d2e3f4-...      CheapestValid   a1b2c3d4-...    Completed  Succeeded       available
d2e3f4g5-...      Balanced        b2c3d4e5-...    Completed  Succeeded       available
(inline created)  BestQuality     c3d4e5f6-...    Completed  Timeout         available
```

**Acceptance Criteria:**
- The same routing problem document is submitted for all scheduler configurations in the experiment (same `problem_id` is not shared across jobs per SPEC-001, but all jobs are submitted with identical problem content).
- Inline scheduler configuration entries are created before the first job is submitted.
- Job submissions proceed even if one configuration entry fails validation; the failed entry is noted in the result.
- If `wait: true` and one or more jobs time out, the summary notes the timed-out jobs and exits with code 4.
- The experiment result JSON manifest captures all `job_id` values, all `problem_id` values, all `config_id` values, all solver outcomes, and report availability.
- An experiment with no scheduler configuration entries is rejected with a usage error before any API call.
- Inline scheduler configuration entries (`inline` key) create new persistent configurations in the system via `POST /v1/scheduler-configs` on each experiment invocation. Duplicate configurations are not detected or prevented (SPEC-008 FR-16: `POST /v1/scheduler-configs` is not idempotent). Accumulation of inline configurations across repeated experiment runs is an accepted MVP limitation. Callers who wish to avoid accumulation should pre-create configurations and reference them by `config_id`.

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
- All errors are also written to stdout as a JSON object with `error_code` and `message` fields (consistent with the SPEC-008 error model).
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
| 2 | `VALIDATION_ERROR` | The submitted routing problem or configuration was rejected by the API or local pre-validation. |
| 3 | `GENERATION_FAILURE` | The workload generator failed (capacity infeasibility, achievability violation, invalid generation config). |
| 4 | `WAIT_TIMEOUT` | The `job wait` or `problem run --wait` polling timeout expired before the job reached a terminal state. |
| 5 | `JOB_FAILED` | A waited job reached the terminal state `Failed` (Worker lifecycle failure, not a solver failure). |

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
- Exit code 5 is used when a waited job reaches `status = Failed` (Worker lifecycle failure). It is not used for any `solver_outcome` value on a `Completed` job.
- Exit code 2 is used for both local validation errors (bad generation config) and API validation errors (HTTP 400 from SPEC-008).
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
| `cli.command.exit` | Command exits | `exit_code`, `duration_ms` |

**Log safety:** Log events must not include routing problem coordinate arrays, full stop lists, or `execution_seed` values. Job identifiers and stop counts are safe to log.

**Acceptance Criteria:**
- No OTel spans are emitted by the CLI.
- Debug log events appear on stderr only when `DAEDALUS_LOG=debug` is set.
- Debug log events are suppressed when `DAEDALUS_LOG` is unset or set to any other value.
- Debug log events are valid JSON, one event per line.
- Log events never include routing problem coordinate arrays.

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

---

# Assumptions

- The Daedalus API is reachable at the configured `api_url` before the CLI is invoked. The CLI does not wait for the API to become healthy.
- The Docker Compose environment is the expected deployment context for local development. The default `http://localhost:5000` API URL is consistent with the Docker Compose port mapping.
- The SPEC-002 generator is embedded as a library within the CLI binary. No separate generator process is required.
- The system clock on the machine running the CLI is approximately synchronized. Timestamps printed by the CLI are derived from API responses, not the local clock.
- The CLI is single-invocation per command. It does not run as a daemon or background process.
- Report files produced by the Report Generator are HTML and are served verbatim by the API. The CLI does not attempt to parse or render HTML in the terminal.
- The `experiment run` command submits all jobs using the same generated problem content but as distinct job submissions. Because each `POST /v1/jobs` produces a distinct `problem_id`, the problems are not shared at the persistence layer — they are identical in content but independently persisted.

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
| Experiment file | Developer, automation script | JSON per FR-12 schema | `experiment run` |
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
| Experiment result manifest | Developer, CI | JSON | `<name>.result.json` |
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

### Experiment Partial Failure

**Cause:** One or more jobs in an experiment reach `status = Failed`, or wait timeout expires for one or more jobs.
**Expected behavior:** The experiment summary table notes the failed or timed-out jobs. The CLI exits with the highest exit code observed across all jobs (code 4 for timeout, code 5 for Worker failure). Successfully completed jobs are still reported.
**Expected fallback:** Investigate the failed jobs individually using `job status <job_id>`.
**User-visible result:** Experiment summary table with per-job status; non-zero exit code reflecting the worst outcome.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Domain Layer | Indirect — CLI embeds SPEC-002 generator; generator uses SPEC-001 domain model |
| API Layer | Indirect — CLI is a caller of the SPEC-008 API; no new API endpoints required for MVP CLI |
| Persistence | None — CLI does not access PostgreSQL |
| Solver Runtime | None — CLI does not invoke solvers |
| Simulation Framework | Yes — CLI is the primary entry point for the simulation framework (SPEC-002 generator host) |
| Observability | Minimal — no OTel; CLI debug logs are development-only |
| Security | Minimal — CLI is the external client; API is the trust boundary |
| Configuration | Yes — CLI introduces a config discovery protocol (`daedalus.yaml`, `DAEDALUS_API_URL`) |

**SPEC-002 OQ-2 resolution:** SPEC-016 resolves SPEC-002's open question about generator process location. The generator is a library embedded in the CLI binary. This is the "CLI layer" option from SPEC-002 OQ-2 candidate list: "a subcommand of the Daedalus CLI — generator logic in the CLI layer; produces a JSON document without full domain type access." This does not require a revision to SPEC-002 beyond noting the resolution.

**No new SPEC-008 endpoints required at MVP:** The CLI's `job list` command (FR-9) requires a `GET /v1/jobs` list endpoint. If this endpoint is not defined in SPEC-008 at implementation time, `job list` is deferred per OQ-1.

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

- Every command exits with exactly one of the defined codes (0–5).
- Exit code 0 for successful submission (no `--wait`).
- Exit code 0 for a waited job that completes with `solver_outcome = Infeasible`.
- Exit code 0 for a waited job that completes with `solver_outcome = Timeout`.
- Exit code 0 for a waited job that completes with `solver_outcome = Cancelled`.
- Exit code 0 for a waited job that completes with `solver_outcome = Failed` (solver-level failure within a `Completed` job; distinct from `status = Failed`).
- Exit code 0 for a waited job that completes with `solver_outcome = ContractViolation`.
- Exit code 5 only for a job that reaches `status = Failed` (Worker lifecycle failure).
- Exit code 2 for any API validation error (HTTP 400).
- Exit code 3 for any generator construction self-check failure.

## Experiment Behaviors

- An experiment file with two scheduler configs submits two jobs with identical problem content.
- A generation failure in an experiment exits with code 3 before any job is submitted.
- An experiment with `wait: true` collects results from all jobs.
- When one job reaches `Failed` in an experiment, the CLI exits with code 5 and reports all other job outcomes.
- The experiment result manifest (`<name>.result.json`) contains: the experiment `name`, the generation config or problem file source, all submitted `job_id` values, all resolved `config_id` values, all `solver_outcome` values (for completed jobs), and all `report_available` flags.
- If `<name>.result.json` already exists from a prior run, the CLI overwrites it without prompting.

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

**Polling interval:** The default `--poll-interval 2` seconds for `job wait` is a balance between responsiveness and API load. Polling more frequently than once per second is discouraged for automation scripts running many concurrent experiments.

**Experiment concurrency:** `experiment run` submits all jobs sequentially, not concurrently. Concurrent submission is not required at MVP scope and simplifies error handling.

**Generator performance:** The SPEC-002 generator is expected to complete in milliseconds at all MVP stop counts. No CLI-level timeout is required for generation.

**Report download size:** HTML evidence reports are bounded by the report content defined in SPEC-009. At MVP scope, report file sizes are not expected to be a performance concern. Large reports should be written to file with `--output <file>` rather than to stdout to avoid terminal rendering overhead.

---

# Documentation Updates Required

- **README.md**: Add a CLI Quick Start section demonstrating: `docker compose up`, `daedalus problem run <config>`, `daedalus report show <job_id>`. This is the primary portfolio demonstration path.
- **docs/architecture.md**: The System Context diagram shows `CLI → API`. SPEC-016 is the authoritative definition of the CLI component. The architecture description of the CLI can reference SPEC-016.
- **SPEC-002 OQ-2**: Mark as resolved. The generator is embedded in the CLI binary (CLI layer option). No SPEC-002 revision is required; the resolution is noted in SPEC-016 FR-4 and the Architectural Impact section.
- **SPEC-008 Documentation Updates Required**: A CLI Specification reference was listed as pending. SPEC-016 satisfies this item. SPEC-008 FR-2 is the HTTP contract the CLI wraps.

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

**Why it matters:** SPEC-008 defines eight endpoints (FR-15 Versioning section) and does not include a job listing endpoint. The `job list` command requires this endpoint. Without it, `job list` cannot be implemented and must remain deferred.

**Resolution path:** A future SPEC-008 revision adds `GET /v1/jobs` to the endpoint inventory (FR-15) and defines its full contract. The CLI consumer requirements above are the minimum SPEC-008 must satisfy for SPEC-016's `job list` command to be implementable.

**Blocking:** Blocking for `job list` implementation only. Not blocking for any other CLI command. `job list` should remain deferred in the Definition of Done until OQ-1 is resolved via SPEC-008 revision.

---

### OQ-2: CLI Implementation Language

**Question:** What language is the Daedalus CLI implemented in?

**Why it matters:** The CLI embeds the SPEC-002 generator (FR-4, FR-5), which in Core is a C++ library. Three options exist:
- (a) C++ — shares generator code directly with Core; requires a C++ CLI framework.
- (b) C# — consistent with the API implementation language (ADR-002); requires the generator to be reimplemented or called via FFI.
- (c) Another language (Go, Rust) — standalone; requires the generator to be independently implemented.

**Architectural note:** If the CLI is implemented in C++ (option a), it naturally hosts the same generator library as Core, eliminating a reimplementation risk. If implemented in another language, the generator must be independently implemented, and divergence from Core's implementation (PRNG sequence, Earth radius constant, distribution algorithms) is a risk.

**Owner:** Project Owner decision. Resolution required before implementation begins.

**Blocking:** Blocking for implementation. Not blocking for specification acceptance.

---

### OQ-3: Experiment Result Persistence

**Question:** Should experiment results be persisted beyond the local filesystem manifest file, for example in PostgreSQL via the API?

**Why it matters:** The `experiment run` command produces a local JSON manifest. If the manifest is lost, the experiment results are only recoverable by re-querying individual job statuses. For long-running experiments or CI pipelines, a server-side experiment record would enable retrieval after the CLI session ends.

**MVP position:** Local manifest only. Experiment results are captured at run time and saved to `<name>.result.json`. No API persistence of experiment records is required at MVP scope.

**Owner:** Project Owner decision if API persistence of experiments is desired. The CLI specification does not require it.

**Blocking:** Not blocking for MVP.

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
- [x] Exit codes are defined.
- [x] Output formats are defined for both text and JSON modes.
- [x] SPEC-002 OQ-2 (generator process location) is resolved.
- [ ] OQ-1 resolved — `GET /v1/jobs` list endpoint added to SPEC-008 via future revision (blocking for `job list` command only; endpoint contract owned by SPEC-008, not SPEC-016).
- [ ] OQ-2 resolved — CLI implementation language (blocking for implementation).
- [ ] OQ-3 resolved — experiment result persistence (non-blocking at MVP).
- [ ] OQ-4 resolved — `--wait` default behavior (non-blocking; current spec uses opt-in).

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
- `experiment run` (FR-12) submits a problem with multiple configurations, collects results, and writes the experiment manifest.
- All exit codes (FR-14) are consistent across command paths and verified by testability behaviors.
- All error outputs (FR-15) are structured and do not expose internal details.
- OQ-2 (implementation language) is resolved and the implementation language is recorded.
- `job list` (FR-9) is either implemented (if OQ-1 is resolved) or explicitly deferred with a note.
- The generated routing problem document produced by `problem generate` passes SPEC-001 domain validation when submitted to the API.
- The portfolio demonstration path — `daedalus problem run <config> --wait` followed by `daedalus report show <job_id>` — completes end-to-end against a running Docker Compose environment.
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
