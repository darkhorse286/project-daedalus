# Implementation Work Package
## P1-OBJ-001 — Docker Compose Infrastructure Foundation

---

## Metadata

**Work Package ID:** P1-OBJ-001

**Title:** Docker Compose Infrastructure Foundation

**Status:** Ready

**Author:** Darkhorse286

**Created:** 2026-08-01

**Last Updated:** 2026-08-01

**Superseded By:** —

**Supersedes:** —

**Phase Plan Reference:** docs/implementation/phase-1-plan.md §3 P1-OBJ-001

---

## Objective

Establish the shared Docker Compose development environment used by all Phase 1 integration tests and all subsequent implementation phases.

The outcome is a `docker compose up -d` invocation that starts five infrastructure service containers — PostgreSQL, RabbitMQ, OpenTelemetry Collector, Prometheus, and Grafana — and brings all five to healthy state in a single operation, repeatably, on a fresh clone.

No application code, migration scripts, or source scaffolding is produced. The repository state after this work package is merged must be identical to the current state except for the files listed in the Scope section.

---

## Governing Artifacts

| Artifact | Authority |
|---|---|
| ADR-003 (RabbitMQ, Accepted) | Queue topology: `routing-jobs` and `routing-jobs-dead-letter` queues must exist |
| ADR-004 (PostgreSQL, Accepted) | PostgreSQL is the sole durable store; single-node; no HA at MVP |
| ADR-006 (Observability Stack, Accepted) | OTel Collector, Prometheus, Grafana; structured JSON logs to stdout |
| ADR-014 (Browser Client Architecture, Accepted) | `http://localhost:3000` is reserved for the `web-ui` service (Decision 5); Grafana must not use host port 3000 |
| docs/architecture.md §MVP Container Topology | Canonical set of infrastructure containers: `postgres`, `rabbitmq`, `otel-collector`, `prometheus`, `grafana` |
| docs/implementation/phase-0-decisions.md | Approved implementation baseline; no Phase 0 decisions directly govern this objective |
| docs/implementation/phase-1-plan.md §3 P1-OBJ-001 | Objective scope, exclusions, acceptance evidence, review points, escalation conditions, PR boundary |

---

## Scope

The implementation agent is authorized to create the following files. No other files may be created or modified.

### Permitted Files

| File | Action | Purpose |
|---|---|---|
| `docker-compose.yml` | Create | Service definitions, health checks, volume mounts, network, port mappings |
| `.env.example` | Create | Template for all required environment variables; placeholder values only; no secrets |
| `config/otel-collector.yaml` | Create | OTel Collector pipeline: OTLP/gRPC receiver → Prometheus exporter + logging exporter |
| `config/prometheus.yaml` | Create | Prometheus scrape configuration targeting the OTel Collector metrics endpoint |

### Service Requirements

**`postgres`**

- Official PostgreSQL image (stable release; implementation agent selects the specific tag — see Local Implementation Authority)
- Named volume `postgres-data` for data directory persistence across `docker compose down` + `docker compose up` cycles
- Health check: `pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}` with appropriate interval, timeout, retries, and start_period
- Published host port: 5432 (default)
- Credentials sourced from `.env` variables: `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`
- No initialization scripts in this work package; the schema is applied by P1-OBJ-002

**`rabbitmq`**

- Official RabbitMQ image with management plugin enabled (e.g., `rabbitmq:3-management` or equivalent stable tag)
- Queue topology per ADR-003: `routing-jobs` (primary work queue) and `routing-jobs-dead-letter` (dead-letter destination)
- Queue topology must be declared in the RabbitMQ configuration, not deferred to application code. Acceptable approaches include: a `rabbitmq.conf` + `definitions.json` file mounted as a volume, or a RabbitMQ definitions file mounted at the standard definitions import path
- Health check: `rabbitmq-diagnostics -q ping` or equivalent management API check
- Published host ports: 5672 (AMQP), 15672 (management UI)
- Credentials sourced from `.env` variables: `RABBITMQ_DEFAULT_USER`, `RABBITMQ_DEFAULT_PASS`
- Virtual host: `vhost` named `/` (default) is acceptable unless a project-specific vhost is needed; document the choice in `.env.example`

**`otel-collector`**

- OpenTelemetry Collector Contrib image (stable release)
- Pipeline: OTLP/gRPC receiver (port 4317) + OTLP/HTTP receiver (port 4318) → Prometheus exporter (port 8889 for scraping) + logging exporter (stdout, detailed level for debug visibility)
- Configuration file: `config/otel-collector.yaml` mounted into the container
- Health check: OTel Collector health_check extension on port 13133, or HTTP GET `/` on the health extension endpoint
- Published host ports: 4317 (OTLP gRPC), 4318 (OTLP HTTP)
- The Prometheus exporter in the Collector must expose metrics at a path and port that the `prometheus` service can scrape

**`prometheus`**

- Official Prometheus image (stable release)
- Scrape configuration: `config/prometheus.yaml` mounted into the container at the standard Prometheus config path
- `prometheus.yaml` must include a scrape job targeting the OTel Collector's Prometheus exporter endpoint (internal Docker network address, not localhost)
- Health check: HTTP GET `/-/healthy` on the Prometheus port (default 9090)
- Published host port: 9090

**`grafana`**

- Official Grafana image (stable release)
- Named volume `grafana-data` for dashboard and data source persistence
- Health check: HTTP GET `/api/health` on the Grafana port
- Published host port: implementation agent's choice — **must not be 3000** (reserved for `web-ui` per ADR-014 Decision 5); must not conflict with any other published port in this Compose file; document the chosen port in `.env.example` as `GRAFANA_PORT`
- Credentials sourced from `.env` variables: `GRAFANA_ADMIN_USER`, `GRAFANA_ADMIN_PASSWORD`
- No dashboard definitions in this work package (ADR-006 explicitly defers dashboard definitions)
- No alert configuration (ADR-006 explicitly defers alerts)

**Named Volumes**

| Volume | Service | Purpose |
|---|---|---|
| `postgres-data` | `postgres` | PostgreSQL data directory persistence |
| `grafana-data` | `grafana` | Grafana dashboard and data source persistence |
| `reports` | (reserved) | Evidence report file storage; declared in this work package; no service mounts it yet |

**`.env.example`**

Must document every environment variable required for the stack to start, with safe placeholder values. Required entries include at minimum:

```
# PostgreSQL
POSTGRES_USER=daedalus
POSTGRES_PASSWORD=<replace>
POSTGRES_DB=daedalus

# RabbitMQ
RABBITMQ_DEFAULT_USER=daedalus
RABBITMQ_DEFAULT_PASS=<replace>

# Grafana
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=<replace>
GRAFANA_PORT=<chosen-port>

# OTel Collector
# (add any environment variables the collector config requires)
```

The implementation agent may add additional variables required by the chosen configuration. All secret values must use `<replace>` as the placeholder. The `.env.example` file is committed. The `.env` file must be listed in `.gitignore` (verify `.gitignore` already excludes `.env`; if it does not, add it as part of this work package — `.gitignore` is the only existing file that may be modified).

---

## Non-Scope

The following are explicitly outside this work package. Implementing any of the following items is a stop condition.

### Prohibited Files and Components

- Any file under `src/` (no application source code)
- Any file under `infrastructure/postgres/` (no migration scripts, no seed scripts)
- Any modification to existing files in `config/` (`backend-profile.schema.json`, `config/backends/*.json` must not be touched)
- `docker-compose.override.yml` or any secondary Compose file
- Any Dockerfile for application services (`api`, `worker`, `python-adapter`, `web-ui`, `db-migrate`)
- Any CI/CD pipeline file (GitHub Actions, GitLab CI, Jenkins, etc.)
- Any `Makefile` or scripts in `scripts/` (these are created in P1-OBJ-005)
- `README.md` (not modified in this work package)
- Any test file

### Prohibited Behaviors

- No application containers (`api`, `worker`, `python-adapter`, `web-ui`) in `docker-compose.yml`
- No migration runner container (`db-migrate`) in `docker-compose.yml`
- No schema initialization or seed data of any kind
- No secrets committed to any file (no real passwords, tokens, or keys in any committed file)
- No Grafana dashboard definitions
- No Prometheus alert rules
- No RabbitMQ policies beyond what is required to declare the two ADR-003 queues
- No PCG64, solver contract, scheduling logic, or any domain behavior

---

## Dependencies

| Dependency | Classification | Detail |
|---|---|---|
| Current repository state | Non-Blocking | Repository is clean on branch `phase1/implementation-planning`; `docker-compose.yml` is an empty file (0 bytes); `infrastructure/postgres/migrations/` and `infrastructure/postgres/seed/` contain only `.gitkeep` |
| P1-OBJ-002 through P1-OBJ-005 | Non-Blocking | All later Phase 1 objectives depend on this work package; none of them are prerequisites for this work package |
| Docker Engine ≥ 20.10 and Docker Compose v2 plugin | Environment Assumption | The developer environment must have Docker Engine and the Compose v2 plugin (`docker compose`, not `docker-compose`) installed; this is a developer workstation prerequisite, not a repository deliverable |

**Dependency Assumptions the Agent May Rely On:**

- The `.gitignore` at the repository root already excludes `.env` (verify before committing; add if absent)
- The `config/` directory exists and is writable; the two new config files (`otel-collector.yaml`, `prometheus.yaml`) are safe to add alongside existing files
- No other process holds port 5432, 5672, 15672, 4317, 4318, or 9090 on the developer workstation at test time

---

## Acceptance Criteria

### AC-1: Configuration Validity

**Description:** The Compose file is syntactically valid and produces no errors or warnings when the configuration is evaluated.

**Evidence required:** `docker compose config` exits with code 0 and produces no error output. A human reviewer can read the resolved configuration and verify it matches the requirements in this work package.

---

### AC-2: All Five Services Start and Reach Healthy State

**Description:** All five infrastructure services start without error and each reaches the `healthy` state as reported by Docker Compose health checks.

**Evidence required:** `docker compose up -d` exits 0. `docker compose ps` shows all five services with `(healthy)` status. All five containers are running; no container exits, restarts, or remains in `starting` state after the health check grace period.

---

### AC-3: Reproducible Cold Start

**Description:** A full teardown followed by fresh start produces the same healthy state without manual intervention.

**Evidence required:** After `docker compose down -v`, `docker compose up -d` brings all five services to healthy state again. The sequence must complete without manual intervention.

---

### AC-4: PostgreSQL Reachable

**Description:** PostgreSQL accepts connections from outside the Docker network using the credentials in `.env.example`.

**Evidence required:** `pg_isready -h localhost -p 5432 -U ${POSTGRES_USER}` exits 0. Alternatively, a `psql` connection to the published port succeeds and returns from `SELECT 1;`.

---

### AC-5: RabbitMQ Queue Topology Correct

**Description:** Both ADR-003 queues exist and are accessible through the management API.

**Evidence required:** `GET http://localhost:15672/api/queues` (with management credentials) returns a JSON array containing exactly two queue objects with names `routing-jobs` and `routing-jobs-dead-letter`. Queue durability is `true` for both.

---

### AC-6: RabbitMQ Management API Healthy

**Description:** The RabbitMQ management API reports healthy status.

**Evidence required:** `GET http://localhost:15672/api/health/checks/alarms` returns HTTP 200.

---

### AC-7: OTel Collector Healthy

**Description:** The OTel Collector health extension responds to health checks.

**Evidence required:** `GET http://localhost:13133/` (or the configured health check endpoint) returns HTTP 200 with a body indicating the collector is running.

---

### AC-8: Prometheus Reachable and OTel Scrape Target Active

**Description:** Prometheus is reachable and shows the OTel Collector metrics endpoint as a `UP` scrape target.

**Evidence required:** `GET http://localhost:9090/-/healthy` returns HTTP 200. `GET http://localhost:9090/api/v1/targets` returns a response where the OTel Collector scrape target has `health: "up"`.

---

### AC-9: Grafana Reachable on Non-3000 Port

**Description:** Grafana is reachable on its configured host port, and that port is not 3000.

**Evidence required:** `GET http://localhost:${GRAFANA_PORT}/api/health` returns HTTP 200. The `GRAFANA_PORT` value in `.env.example` is not 3000.

---

### AC-10: No Secrets Committed

**Description:** No real credential, token, or key appears in any committed file.

**Evidence required:** All secret-valued fields in `.env.example` use the literal placeholder `<replace>`. A `.env` file is absent from the repository (either not created, or listed in `.gitignore`). `git status` shows no `.env` file and no file containing a real password.

---

### AC-11: Named Volumes Declared

**Description:** All three named volumes (`postgres-data`, `grafana-data`, `reports`) are declared in `docker-compose.yml`.

**Evidence required:** `docker compose config` output includes a `volumes:` top-level section listing `postgres-data`, `grafana-data`, and `reports`. `docker volume ls` after `docker compose up -d` shows all three volumes created.

---

## Required Verification Evidence

The following evidence must be produced and included in the Implementation Completion Report.

| Evidence | Method | Required Outcome |
|---|---|---|
| Compose config valid | `docker compose config` terminal output | Exit 0, no errors |
| All services healthy | `docker compose ps` terminal output | Five services, all `(healthy)` |
| Cold-start reproducibility | `docker compose down -v && docker compose up -d && docker compose ps` terminal output | All five services healthy after fresh start |
| PostgreSQL connectivity | `pg_isready` or `psql` connection output | Exit 0 / successful connection |
| RabbitMQ queue topology | `GET /api/queues` response body (redacted credentials) | Both queues present, durable |
| RabbitMQ management health | `GET /api/health/checks/alarms` HTTP status | HTTP 200 |
| OTel Collector health | Health extension HTTP response | HTTP 200 |
| Prometheus health | `GET /-/healthy` HTTP status | HTTP 200 |
| OTel scrape target state | `GET /api/v1/targets` JSON (scrape target `health` field) | `"up"` |
| Grafana health | `GET /api/health` HTTP status on `${GRAFANA_PORT}` | HTTP 200 |
| No secrets committed | `git diff HEAD` or `git show` of all committed files | No real credentials in any committed file |
| Named volumes created | `docker volume ls` output | Three volumes present |

---

## Local Implementation Authority

The implementation agent may make the following choices without escalation. Each choice must be documented in `.env.example` or in a comment in the relevant configuration file.

| Choice | Constraint | Documentation Requirement |
|---|---|---|
| PostgreSQL image tag | Stable/LTS release only (e.g., `postgres:16` or `postgres:17`); no `latest` tag in production-facing Compose files | Record chosen tag in `docker-compose.yml` comment or inline |
| RabbitMQ image tag | Stable release with management plugin (e.g., `rabbitmq:3.13-management`); no `latest` | Record chosen tag |
| OTel Collector image tag | Stable release of `otel/opentelemetry-collector-contrib` | Record chosen tag |
| Prometheus image tag | Stable release of `prom/prometheus` | Record chosen tag |
| Grafana image tag | Stable release of `grafana/grafana` | Record chosen tag |
| Grafana host port | Any port not already claimed by another service in this Compose file and not 3000; suggested range 3001–3099 | Document as `GRAFANA_PORT` in `.env.example` |
| OTel Collector logging exporter verbosity | `detailed`, `normal`, or `basic` — choose what provides useful debug output | Comment in `otel-collector.yaml` |
| RabbitMQ queue declaration mechanism | `definitions.json` volume mount, `rabbitmq.conf` + enabled plugins file, or management API via a startup script — choose the approach most appropriate for Docker Compose; the result must be that queues exist before the first health check passes | Comment in `docker-compose.yml` explaining the approach |
| PostgreSQL port (host-side) | 5432 is default; may use an alternate if 5432 is occupied on the developer workstation, but must document the non-default port in `.env.example` | |
| Docker Compose internal network name | Default or named; implementation agent's choice | |
| Health check intervals | `interval`, `timeout`, `retries`, and `start_period` values are local choices; they must be long enough to avoid false failures on a cold start and short enough to not delay CI; reasonable defaults: 10s interval, 5s timeout, 5 retries, 30s start_period | |

---

## Mandatory Human-Review Points

These points require the implementation agent to produce clear documentation in the pull request description so a human reviewer can make an informed decision. The implementation agent must NOT block on these points waiting for approval — choose a reasonable default, implement it, and flag it clearly in the PR.

### HRP-1: Grafana Host Port

The chosen Grafana host port becomes a project-wide convention referenced in developer documentation and the Container Topology diagram. Record the chosen port value prominently in the PR description. The reviewer confirms this port does not conflict with any other service in the planned topology.

### HRP-2: RabbitMQ Queue Topology Mechanism

The queue declaration approach (definitions file vs. management API script vs. config file) becomes the established pattern for all future queue topology changes. Document which approach was chosen and why. The reviewer confirms the `routing-jobs` and `routing-jobs-dead-letter` queues are durable and that the chosen mechanism is maintainable.

### HRP-3: `.env.example` Credentials Structure

The variable names and placeholder conventions established here are inherited by all subsequent Compose services. The reviewer confirms every service's credential variables are present and that no real secrets exist anywhere in the committed files.

---

## Stop Conditions

The implementation agent must stop and escalate if any of the following conditions arise. Do not proceed, guess, or work around a stop condition.

| Condition | Action |
|---|---|
| A required service cannot be made healthy within the Docker Compose health check model — for example, the OTel Collector health extension is not available in the selected image variant | Stop. Report the specific image, the missing capability, and the nearest available alternative. Do not ship a service without a working health check. |
| The RabbitMQ queue topology cannot be declared without writing application-side initialization code (i.e., there is no Compose-native or configuration-file mechanism) | Stop. Report the constraint. The queue topology is an ADR-003 requirement; it must not be deferred to application startup. |
| A required port conflicts with another service in the file and no alternative port is locally authoritative | Stop. Report the conflict. Port assignments have cross-component documentation implications. |
| Any service requires a secret (API key, license token, external service credential) that cannot be satisfied with a placeholder in `.env.example` | Stop. Do not commit a real credential. Report the requirement. |
| The OTel Collector cannot forward metrics to the Prometheus exporter endpoint using the standard `otel/opentelemetry-collector-contrib` image and a standard pipeline configuration | Stop. Report the specific pipeline configuration attempted and the error. Do not substitute a different observability stack. |
| An implementation discovery arises that would require modifying a file outside the permitted file list to complete this work package | Stop. Report the file, the reason, and the impact. Do not modify prohibited files. |
| A conflict is discovered between the scope defined here and any accepted ADR or accepted specification | Stop. Report the specific conflict with artifact references. Do not silently resolve it. |

---

## Notes

**On the `reports` volume:** The `reports` named volume is declared in this work package but no service mounts it yet. The Worker and API services will mount it in later phases. Declaring it now prevents an error when those services are added to the Compose file in later PRs.

**On the `web-ui` service:** ADR-014 Decision 5 defines the `web-ui` container at `http://localhost:3000`. This service is not added in this work package — its Dockerfile does not yet exist. The `web-ui` service will be added in a later phase. Reserving port 3000 is accomplished solely by not assigning it to Grafana here.

**On OTel Collector pipeline for the API:** The API service (added in P1-OBJ-004) will export OTLP spans to the Collector via port 4317 (gRPC). Configuring those ports in the Collector now ensures the API has a working export target when it is implemented. No API-specific configuration is needed in this work package; OTLP/gRPC on 4317 is a standard receiver.

**On `docker-compose.yml` content stability:** Later work packages (P1-OBJ-002, P1-OBJ-004) will add services to `docker-compose.yml`. This work package owns the infrastructure services only. Do not pre-configure service entries for `api`, `worker`, `db-migrate`, `python-adapter`, or `web-ui` as placeholders or stubs.

**On the `config/` directory:** Two files (`otel-collector.yaml`, `prometheus.yaml`) are added to the existing `config/` directory. The existing files (`backend-profile.schema.json`, `backends/*.json`) must not be modified.

**On authority order:** Per the Engineering Lifecycle (docs/process.md §Authority Order), ADR-003 governs the RabbitMQ queue topology. The phase plan (docs/implementation/phase-1-plan.md) is implementation planning and is subordinate to ADR-003. If the phase plan and ADR-003 appear to conflict on any queue detail, ADR-003 is authoritative and the phase plan must yield.

---

## Expected Implementation Completion Report Content

The Implementation Completion Report (ICR) for this work package must document:

1. **Files created:** list of all committed files with one-line description of each
2. **Local implementation choices made:** image tags chosen, Grafana port chosen, RabbitMQ queue declaration mechanism chosen, OTel Collector logging verbosity chosen — with brief rationale for each
3. **Acceptance criteria status:** for each AC-1 through AC-11, pass or fail, with the verification output that demonstrates the result (terminal output, HTTP response body excerpts, or equivalent)
4. **Mandatory human-review points:** for each HRP-1 through HRP-3, the choice made and the evidence supporting it
5. **Implementation discoveries:** any finding encountered during implementation that was not anticipated in this work package, classified as: local reversible choice, accepted implementation decision candidate, specification contradiction, defect, or future enhancement
6. **Stop conditions encountered:** none expected; if any were encountered, describe the condition, the escalation action taken, and the resolution
7. **Pull request reference:** PR number or branch name

The ICR must be a repository file, not a conversational summary.

---

## Pull Request Boundary

**One mergeable pull request.** The PR contains exactly and only the four files listed in the Permitted Files section, plus `.gitignore` if `.env` is not already excluded.

The pull request must be:
- Independently reviewable,
- Buildable,
- Revertible.

**PR title format:** `feat(infra): Docker Compose infrastructure foundation (P1-OBJ-001)`

**PR description must include:**
- The Grafana host port chosen (HRP-1)
- The RabbitMQ queue declaration mechanism chosen (HRP-2)
- Confirmation that no secrets are committed (HRP-3)
- A checklist of all AC-1 through AC-11 items with pass/fail status
- Link to or inline content of the Implementation Completion Report

**No application source code in this PR.** If `src/`, `infrastructure/postgres/`, or any test file appears in the diff, the PR is out of scope and must be revised before review.
