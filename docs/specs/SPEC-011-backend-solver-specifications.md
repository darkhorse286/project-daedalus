# SPEC-011 — Backend Solver Specifications

## Metadata

**Feature ID:** SPEC-011
**Title:** Backend Solver Specifications
**Status:** Accepted
**Author:** Darkhorse286
**Created:** 2026-06-14
**Last Modified:** 2026-06-20
**Dependencies:** SPEC-001, SPEC-003, SPEC-004, SPEC-005, SPEC-006, ADR-008, ADR-010
**Blocks:** Individual backend solver specifications (SPEC-013, SPEC-014, SPEC-015)
**Related ADRs:** ADR-005, ADR-006, ADR-007, ADR-008, ADR-010, ADR-011

---

## Problem Statement

No specification currently defines the requirements shared across all solver backends. SPEC-004 defines the solver contract interface — what data structures flow between the Worker and a backend (SolverRequest, SolverResponse, SolverOutcome) — but does not define:

- How backends are classified by algorithmic category
- What a backend specification must declare as its capability profile for Scheduler consumption
- What reproducibility obligations bind stochastic versus deterministic backends
- Which SolverOutcome values each category of backend must and must not return
- What observability obligations a backend carries at the specification level
- What evidence a backend must provide to support downstream quality evaluation, regret analysis, and evidence reporting
- How a backend is registered into the system and what prerequisites an individual backend specification must satisfy before implementation proceeds

Without SPEC-011:

- The three MVP backends (nearest-neighbor, greedy insertion, QUBO simulated annealing) would each independently interpret their SPEC-004, SPEC-003, and ADR-010 obligations. Cross-backend consistency would depend on convention rather than policy.
- No unified classification model would exist to govern which SolverOutcome values each type of backend may legally return. SPEC-004 FR-4 notes that `Infeasible` is reserved for exact solvers but does not define an enforced classification structure.
- No single document would define what "adding a new backend" requires. The prerequisites are currently scattered across SPEC-003 FR-4, SPEC-004 FR-1, SPEC-004 FR-11, and ADR-010.
- The seed usage policy documentation obligation in SPEC-004 FR-1 would have no unified home, and individual solver specifications would duplicate or inconsistently represent ADR-010 requirements.

SPEC-011 is the framework specification. It defines what every backend must be, declare, and satisfy. Individual backend specifications (SPEC-013, SPEC-014, SPEC-015) conform to this framework. SPEC-011 does not define any backend's algorithm.

---

## Business Value

- Establishes the uniform framework all backend solver specifications must follow, preventing inconsistency between classical and quantum-inspired backend specifications
- Provides the prerequisite structure that allows individual backend specifications to be evaluated and accepted independently
- Creates the extensibility framework for adding future backends (exact solvers, Python adapter, quantum hardware) without requiring modifications to the Scheduler or Worker
- Demonstrates ability to design plugin-style extension architectures with formal contracts, classification models, and registration requirements
- Closes the specification gap between SPEC-004 (what the contract looks like) and individual solver specifications (how a specific backend fulfills it)

---

## Employer Signaling

- System Design
- Backend / Plugin Architecture Design
- Reliability Engineering
- Optimization
- Observability

SPEC-011 demonstrates the ability to design a formal extensibility framework for a heterogeneous plugin system. The challenge is non-trivial: the framework must be strict enough to ensure behavioral consistency across backends with different algorithmic categories (deterministic construction heuristics versus stochastic annealing), different reproducibility models, and different infeasibility semantics — while remaining flexible enough to accommodate future backends that do not yet exist.

---

## Domain Concept

A solver backend in Project DAEDALUS is a component that accepts a routing problem and execution configuration, executes an optimization algorithm, and returns a route plan together with execution evidence. The backend is accessed exclusively through the normalized SolverContract interface (ADR-008, SPEC-004). The Scheduler selects among registered backends using their declared capability profiles (SPEC-003 FR-4). The Worker invokes the selected backend, enforces timeouts, and persists the result in the Evidence Log (SPEC-005, SPEC-006).

The backend is not a microservice. In the MVP, all C++ backends are invoked in-process by the Worker. The Python adapter backend (ADR-005) is deferred; its invocation model is out of scope for MVP.

Backends are classified by their algorithmic category. Classification determines reproducibility obligations, supported failure outcomes, and whether the backend can prove infeasibility. This classification is a property of the backend's algorithm, not a runtime configuration parameter.

SPEC-011 defines the framework. Each individual backend specification is a child of SPEC-011 and must conform to its requirements before implementation of that backend begins.

---

## Requirements

### FR-1: Responsibility Scope and Component Boundaries

**Description:**
SPEC-011 defines the solver specification framework's ownership boundaries and explicitly allocates adjacent responsibilities to the correct owners.

**3.1.1 SPEC-011 owns the following responsibilities:**
- Solver classification model (FR-2)
- Solver metadata requirements applicable to every backend specification (FR-3)
- Capability declaration requirements — what every backend specification must document as its Backend Capability Profile for Scheduler consumption (FR-4)
- Supported SolverOutcome values per classification (FR-5)
- Reproducibility requirements — seed usage obligations, PRNG algorithm, entropy source policy (FR-6)
- Solver output requirements — RoutePlan obligations, ExecutionStatistics obligations, extension_metadata documentation obligations (FR-7)
- Failure model — which failure outcomes and failure codes backends must support (FR-8)
- Observability requirements — backend obligations toward the `solver.execute` span and structured logs (FR-9)
- Lifecycle and registration requirements — prerequisites for adding a new backend to the Scheduler's eligible pool (FR-10)
- MVP backend inventory (FR-11)
- Required structure for individual backend solver specifications (FR-12)

**3.1.2 SPEC-011 does not own the following responsibilities:**

| Responsibility | Owner |
|---|---|
| Scheduler policy, backend selection algorithm, objective scoring formulas | SPEC-003 |
| Backend eligibility evaluation rules, eligibility rejection reasons | SPEC-003 |
| Backend Capability Profile schema consumed by the Scheduler | SPEC-003 FR-4 |
| SolverRequest / SolverResponse interface definition | SPEC-004 |
| SolverOutcome enum definition | SPEC-004 FR-4 |
| RoutePlan structural schema | SPEC-004 FR-5 |
| ExecutionStatistics schema | SPEC-004 FR-6 |
| SolverFailureDetail schema | SPEC-004 FR-8 |
| Solver contract versioning | SPEC-004 FR-14 |
| `solver.execute` span emission | SPEC-004 FR-15, SPEC-005 |
| Worker execution lifecycle orchestration | SPEC-005 |
| External timeout enforcement | SPEC-005 |
| Evidence persistence schemas | SPEC-006 |
| Report generation | SPEC-009 |
| Workload feature computation | SPEC-010 |
| Routing problem validation | SPEC-001, ADR-009 |
| API, queue, or Worker behavior | SPEC-005, architecture.md |
| Python adapter transport protocol | SPEC-017 (Accepted); ADR-005 (Accepted) |
| Quantum hardware execution model | ADR-007 (deferred) |
| Individual backend algorithms or pseudocode | Individual solver specifications; implementation planning |

**3.1.3 Ownership allocation for individual solver specifications:**

| Responsibility | Owner |
|---|---|
| Seed usage policy (how the backend seeds its internal PRNG from `execution_seed`) | Individual solver specification (per SPEC-004 FR-1, FR-6.5) |
| PCG64 stream constant for stochastic backends | Individual solver specification (per ADR-010 Decision 1) |
| PRNG draw ordering for stochastic backends | Individual solver specification (per ADR-010 Limitations) |
| Backend algorithm description | Individual solver specification |
| Backend-specific extension_metadata key documentation | Individual solver specification (per FR-7.4) |
| Empirical latency_profile and quality_profile values | Individual solver specification (per FR-4) |
| Backend-specific failure conditions beyond the failure model in FR-8 | Individual solver specification |

**Acceptance Criteria:**
- No individual solver specification defines capability profile consumption logic (owned by SPEC-003)
- No individual solver specification alters the SolverRequest/SolverResponse interface (owned by SPEC-004)
- No individual solver specification performs evidence persistence or lifecycle orchestration (owned by SPEC-005, SPEC-006)
- Every individual solver specification conforms to the structure defined in FR-12

---

### FR-2: Solver Classification Model

**Description:**
SPEC-011 defines the authoritative classification model for all backend categories in Project DAEDALUS. Classification is a property of the backend's algorithmic approach and determines which SolverOutcome values the backend supports (FR-5), what reproducibility obligations apply (FR-6), and whether the backend can prove infeasibility.

**3.2.1 Backend categories:**

| Category | Identifier | Description | Determinism Class | Infeasibility Proof |
|---|---|---|---|---|
| Classical Deterministic | `classical_deterministic` | Polynomial-time construction heuristics. No stochastic computation. Produces an identical route plan for identical inputs unconditionally, without PRNG seeding. | Deterministic | No |
| Quantum-Inspired Stochastic | `quantum_inspired_stochastic` | Stochastic optimization heuristics using PRNG-dependent search over a solution space. Produces reproducible output given the same `execution_seed` via PCG64 (ADR-010). | Stochastic (reproducible) | No |
| Exact | `exact` | Exact optimization solvers capable of proving infeasibility. Not in MVP scope. | Deterministic or Stochastic (reproducible) | Yes |
| Quantum Hardware | `quantum_hardware` | Backends executing on real quantum hardware. Deferred per ADR-007. Non-reproducible by nature; hardware entropy precludes reproducibility invariant. | Stochastic (non-reproducible) | No |

**3.2.2 MVP backends:**

The following backends are in scope for the MVP. Each is categorized here; implementation is not authorized until the individual backend specification is Accepted (FR-10).

| Backend ID | Display Name | Category |
|---|---|---|
| `nearest-neighbor` | Nearest Neighbor Heuristic | `classical_deterministic` |
| `greedy-insertion` | Greedy Insertion Heuristic | `classical_deterministic` |
| `qubo-simulated-annealing` | QUBO Simulated Annealing | `quantum_inspired_stochastic` |

**3.2.3 Classification determination:**

Classification is determined by the backend's algorithm, not by runtime configuration. A backend that uses any PRNG-dependent stochastic process is classified `quantum_inspired_stochastic` regardless of whether the stochastic portion is small or optional. A backend that can prove infeasibility may be classified `exact`; heuristics cannot claim this classification regardless of solution quality.

**Acceptance Criteria:**
- Every solver specification declares exactly one backend category from the table in FR-2.1
- No MVP backend specifies a category not in this table
- No `classical_deterministic` or `quantum_inspired_stochastic` backend claims infeasibility proof capability

**3.2.4 Python Adapter — transport and deployment note:**

The Python Adapter (ADR-005) is a deployment and transport mechanism, not an algorithmic backend category. When the Python Adapter transport is specified and implemented, Python adapter backends will carry one of the existing algorithmic categories from FR-2.1 based on the algorithm they implement — for example, `classical_deterministic` for a Python construction heuristic or `quantum_inspired_stochastic` for a Python stochastic search. The transport mechanism (ADR-005 process boundary and wire protocol) is governed by ADR-005 and a future adapter specification; it is not a property of the backend's algorithm and does not determine FR-5 or FR-6 obligations.

ADR-005 OQ-1 (transport protocol) is resolved by SPEC-017 (Accepted 2026-06-20): JSON over HTTP is the accepted MVP transport. Individual Python backend specifications (SPEC-018+) may now be written; each will declare an algorithmic category from FR-2.1 (governing FR-5 and FR-6 obligations) and conform to SPEC-011 framework requirements through the mechanisms defined in SPEC-017.

---

### FR-3: Solver Metadata Requirements

**Description:**
Every backend solver specification must document the following metadata fields. These fields establish the backend's stable identity and enable the Scheduler, Evidence Log, and report artifacts to reference the backend unambiguously.

**3.3.1 Required metadata fields:**

| Field | Type | Description |
|---|---|---|
| `backend_id` | string | Unique system identifier for this backend. Used as the primary key in the Backend Capability Profile (SPEC-003 FR-4), in the `solver.execute` span (SPEC-004 FR-15), and in the Evidence Log solver run record (SPEC-006). Kebab-case, lowercase, stable across specification versions. Must not change after the individual solver specification is Accepted. |
| `display_name` | string | Human-readable name used in evidence reports and structured logs. May include spaces and mixed case. |
| `backend_category` | BackendCategory | Classification from FR-2. One of: `classical_deterministic`, `quantum_inspired_stochastic`, `exact`, `quantum_hardware`. The Python Adapter is a transport mechanism, not an algorithmic category; see FR-2.4. |
| `implementation_language` | string | Primary implementation language: `C++`, `Python`, or `Other`. MVP backends are `C++`. |
| `determinism_class` | DeterminismClass | One of: `Deterministic` (identical output for identical input) or `Stochastic (reproducible)` (identical output for identical input and `execution_seed`). |
| `supported_contract_version` | uint32 | The SPEC-004 contract_version this backend targets. Must match the `supported_contract_version` declared in the Backend Capability Profile (FR-4). Must equal 1 for all MVP backends (SPEC-004 FR-14). |
| `specification_version` | uint32 | Monotonically increasing version for this individual solver specification. Starts at 1. Increment on any behavioral change. Any change to the PCG64 stream constant, supported SolverOutcome values, or capability profile fields is a behavioral change. A PCG64 stream constant change additionally requires the full ADR-010 Decision 5 breaking change procedure, which includes updating ADR-010 with an explicit change record documenting the previous and replacement value. A `specification_version` increment is one component of that procedure, not a substitute for it. |

**3.3.2 Stability obligation:**

`backend_id` is frozen once the solver specification is Accepted. Changing a backend's `backend_id` is a breaking change to the Evidence Log, Scheduler configuration, and any persisted evidence records that reference the old `backend_id`. A renamed backend must be treated as a new backend (new specification, new registration).

**Acceptance Criteria:**
- Every individual solver specification includes all seven fields from the table in FR-3.1
- `backend_id` is kebab-case, lowercase
- `supported_contract_version` matches the value in the Capability Profile declared per FR-4
- `specification_version` starts at 1

---

### FR-4: Capability Declaration Requirements

**Description:**
Every solver specification must document a complete Backend Capability Profile conforming to the schema defined in SPEC-003 FR-4. The capability profile is the registration artifact consumed by the Scheduler. A backend with an incomplete or inaccurate capability profile cannot be used correctly by the Scheduler.

**3.4.1 Required capability profile fields:**

| Field | Type | Constraints per SPEC-003 FR-4 | Declaration Obligation |
|---|---|---|---|
| `backend_id` | string | Must match FR-3 `backend_id` | Required |
| `supported_size_classes` | set<SizeClass> | One or more SizeClass values; must reflect actual supported problem sizes | Required |
| `supports_time_windows` | bool | Must accurately reflect whether the backend's construction strategy incorporates time window constraints during route building. When `true`, the backend is eligible for time-window-constrained problems in Scheduler eligibility evaluation (SPEC-003 FR-5 Phase 1). This declaration does not indicate that the backend guarantees time window feasibility in its produced RoutePlan; Core Quality Evaluation (SPEC-007) is the authoritative evaluator of time window satisfaction. A `Succeeded` response does not imply time window feasibility (SPEC-004 FR-7). | Required |
| `supports_capacity_constraints` | bool | Must accurately reflect whether the backend respects capacity constraints in its route construction | Required |
| `latency_profile` | map<SizeClass, seconds> | One entry per supported size class; median expected execution duration derived from empirical measurement on representative problems | Required |
| `quality_profile` | QualityTier | One of: `Baseline`, `Competitive`, `Near-Optimal`; must reflect empirically observed solution quality relative to known-optimal or best-known solutions | Required |
| `cost_profile` | positive number | Expected execution cost per invocation; units defined by the Scheduler configuration; must reflect the actual resource cost of executing this backend | Required |
| `is_provisional` | bool | `true` for backends not yet qualified for production use; `false` for qualified backends (see OQ-4 for qualification criteria) | Required |
| `supported_contract_version` | uint32 | Must match FR-3 `supported_contract_version` | Required |

**3.4.2 Accuracy obligation:**

Capability profile declarations are not aspirational. The Scheduler uses declared values directly for eligibility filtering (SPEC-003 FR-5) and objective scoring (SPEC-003 FR-7). A backend that misrepresents its capability profile — overstating quality, understating latency, or incorrectly declaring constraint support — produces incorrect Scheduler decisions that undermine the project's evidence thesis.

Each individual solver specification must state the basis for its declared `latency_profile` and `quality_profile` values (for example, offline empirical measurement on benchmark problems of each supported size class). For MVP backends, initial values may be estimates pending empirical validation; `is_provisional = true` must be declared until empirical values are established.

**3.4.3 Capability profile registration:**

The mechanism by which capability profiles are registered at runtime (static configuration file, startup registration code, compile-time registration, or runtime service) is an open question in SPEC-003 OQ-2 and is not resolved by SPEC-011. Each individual solver specification must document its complete capability profile field values. The registration mechanism is SPEC-003's responsibility to resolve before any backend capability profile is registered (Prerequisite 7 from FR-10.1).

**Acceptance Criteria:**
- Every individual solver specification contains a capability profile table with all nine fields from FR-4.1
- `backend_id` in the capability profile matches `backend_id` in FR-3 metadata
- `supported_contract_version` in the capability profile matches FR-3 `supported_contract_version`
- The accuracy basis for `latency_profile` and `quality_profile` is stated in the specification

---

### FR-5: Supported SolverOutcome Values

**Description:**
The SolverOutcome values a backend must and must not return are determined by its backend category (FR-2). SPEC-004 FR-4 defines the full SolverOutcome enum; this section defines which values each category must support.

**3.5.1 Outcome support matrix:**

| Outcome | `classical_deterministic` | `quantum_inspired_stochastic` | `exact` (future) | Notes |
|---|---|---|---|---|
| `Succeeded` | Required | Required | Required | All backends must support `Succeeded`. |
| `Timeout` | Required | Required | Required | All backends must detect and return `Timeout` when `execution_timeout_ms` is exceeded. |
| `Cancelled` | Required | Required | Required | All backends must return `Cancelled` when an external cancellation signal is received. |
| `Failed` | Required | Required | Required | All backends must return `Failed` for internal errors that prevent execution completion. |
| `Infeasible` | **Not Supported** | **Not Supported** | Required | Exact solvers only. Heuristic backends must not return `Infeasible`. |

**3.5.2 Infeasible restriction for heuristic backends:**

`classical_deterministic` and `quantum_inspired_stochastic` backends must not return `Infeasible`. Heuristic backends produce the best solution they can find; they cannot prove that no feasible solution exists. A heuristic backend that fails to find a complete solution (a degenerate case for pathological inputs) must return `Failed` with `failure_code = InternalError`, not `Infeasible`. Returning `Infeasible` without proof is a misrepresentation of the backend's output that would corrupt the Scheduler's regret analysis.

**3.5.3 Supported outcome declaration:**

Each individual solver specification must include an explicit table of supported SolverOutcome values. A backend may not return any SolverOutcome value not listed in its declared set.

**Acceptance Criteria:**
- Every individual solver specification contains an explicit supported SolverOutcome table
- No `classical_deterministic` or `quantum_inspired_stochastic` specification lists `Infeasible` as supported
- Every specification lists `Succeeded`, `Timeout`, `Cancelled`, and `Failed` as supported

---

### FR-6: Reproducibility Requirements

**Description:**
ADR-010 governs deterministic randomness policy for all Project DAEDALUS components. The following requirements bind all backend solver specifications to ADR-010. FR-6 applies at the specification level; it defines what each individual solver specification must document. Implementation planning owns the concrete PRNG code, but the specification must make the choices observable before implementation begins.

**3.6.1 Seed Usage Policy:**

- **Stochastic backends** (`quantum_inspired_stochastic`): The backend must seed its internal PRNG from the `execution_seed` value in the SolverRequest (SPEC-004 FR-2). `execution_seed` is derived by the Worker from `RoutingProblem.seed` (SPEC-005 FR-7, ADR-010 Decision 4, ODR-4). `execution_seed` is the exclusive authorized entropy source for all reproducibility-critical stochastic operations within the backend. The PRNG must be seeded exactly once per solver execution from `execution_seed` (ADR-010 Decision 4). Reseeding during execution is prohibited.

- **Deterministic backends** (`classical_deterministic`): The backend must accept the `execution_seed` field in the SolverRequest. Deterministic backends do not use `execution_seed` as PRNG input because they have no PRNG. Deterministic backends produce identical outputs for identical routing problem inputs unconditionally.

- **Prohibited entropy sources (all backends):** System time, process ID, OS random sources (`/dev/urandom`, `getrandom(2)`, `std::random_device`), and hardware entropy are prohibited as inputs to any reproducibility-critical computation (ADR-010 Decision 4). Their use makes output non-reproducible from `execution_seed` alone.

**3.6.2 PRNG Algorithm:**

- All `quantum_inspired_stochastic` backends must use PCG64 as their PRNG for reproducibility-critical paths (ADR-010 Decision 1).
- No other PRNG algorithm is permitted in reproducibility-critical paths for MVP `quantum_inspired_stochastic` backends.
- Each individual stochastic solver specification must document its **PCG64 stream constant** (increment). The stream constant is frozen once the individual solver specification is Accepted. Changing a stream constant requires the full ADR-010 Decision 5 breaking change procedure, which includes updating ADR-010 with an explicit change record documenting the previous and replacement value. A `specification_version` increment is one component of that procedure, not a substitute for it.
- Each individual stochastic solver specification must document its **PRNG draw ordering** — the fixed sequence in which PCG64 draws are consumed during a solver execution (ADR-010 Limitations).

**3.6.3 Distribution Sampling:**

All `quantum_inspired_stochastic` backends must use the distribution sampling algorithms approved in ADR-010 Decision 3. ADR-010 Decision 3 is the authoritative source for the algorithm specifications, constraints, and prohibited standard library alternatives. The approved algorithm categories are: uniform floating-point on (0, 1], bounded uniform integer, and normal distribution sampling.

Key normative constraints from ADR-010 Decision 3 applicable to all stochastic solver specifications:
- `std::uniform_int_distribution` is prohibited in reproducibility-critical paths
- `std::normal_distribution` is prohibited in reproducibility-critical paths

Each individual `quantum_inspired_stochastic` solver specification must:
- Identify which approved algorithm categories it uses (uniform floating-point, bounded uniform integer, and/or normal distribution)
- Confirm that only ADR-010 Decision 3 approved algorithms are used and that prohibited standard library alternatives are not used in reproducibility-critical paths
- Document any zero-case handling for uniform float conversion if the backend passes PRNG output to transcendental functions, with the specific handling method noted

**3.6.4 Floating-Point Determinism:**

- All floating-point computation in reproducibility-critical solver paths must use IEEE 754 double precision (ADR-010 Decision 6).
- Extended precision (x87 80-bit) must be disabled in reproducibility-critical code. The specific compiler flags required to enforce this are an implementation planning concern per ADR-010.
- Components must not assert bitwise equality on transcendental-function-derived floating-point values. Semantic equivalence within IEEE 754 double precision is the standard (ADR-010 Decision 2).

**3.6.5 Seed Usage Policy Documentation (per SPEC-004 FR-1):**

Each individual solver specification must include a dedicated **Seed Usage Policy** section containing:

- The determinism class of this backend (Deterministic or Stochastic reproducible)
- For stochastic backends: the PCG64 stream constant, a description of the PRNG draw ordering, and an explicit statement that `execution_seed` is the exclusive entropy source
- For deterministic backends: an explicit statement that `execution_seed` is accepted in the SolverRequest but not used for any PRNG operation, and that output is deterministic unconditionally

This section satisfies the documentation obligation stated in SPEC-004 FR-1.

**Acceptance Criteria:**
- Every individual solver specification includes a Seed Usage Policy section
- Every stochastic solver specification names PCG64 as its PRNG
- Every stochastic solver specification documents its stream constant placeholder (or value, once determined)
- No solver specification names a prohibited entropy source as a seeding input
- Every stochastic solver specification specifies its approved distribution sampling algorithms

---

### FR-7: Solver Output Requirements

**Description:**
All backends must produce outputs conforming to the SolverResponse schema defined in SPEC-004 FR-3. This section specifies the behavioral expectations that apply uniformly across all backend categories. Individual solver specifications may add backend-specific constraints within these bounds but must not contradict them.

**3.7.1 RoutePlan structural obligations:**

| Outcome | Route Plan Presence | Structural Requirements |
|---|---|---|
| `Succeeded` | Required | Every customer stop assigned exactly once. Route count = vehicle_count (empty routes are valid and represent unused vehicles, per SPEC-004 FR-5). Capacity per route ≤ capacity_per_vehicle. Depot not included in any route (SPEC-004 FR-5). |
| `Timeout` | Optional | A complete RoutePlan satisfying all SPEC-004 FR-5 structural validity requirements may be present if the backend found a complete solution before the deadline. If no complete solution was found, route plan is absent. |
| `Cancelled` | Optional | A complete RoutePlan satisfying all SPEC-004 FR-5 structural validity requirements may be present if the backend found a complete solution before cancellation. If no complete solution was found, route plan is absent. |
| `Infeasible` | Absent | No route plan. |
| `Failed` | Absent | No route plan. |

**3.7.2 Success semantics:**

`Succeeded` means the backend produced a complete, capacity-valid solution. It does not guarantee that time window constraints are satisfied. Time window feasibility is evaluated by Core quality evaluation after the Worker receives the SolverResponse (SPEC-004 FR-7, SPEC-007). A backend must not return `Succeeded` unless every customer stop is assigned exactly once, routes.size() equals vehicle_count (empty routes are valid), and no route exceeds capacity_per_vehicle.

**3.7.3 ExecutionStatistics obligations:**

- `execution_duration_ms` is required for all outcomes. This value is populated in the `solver.execute` span's `execution_duration_ms` attribute by the Worker (SPEC-004 FR-15).
- `solution_count` is optional. Stochastic backends that maintain and evaluate multiple candidate solutions during search may populate this field. Its value informs evidence analysis of search depth. Classical deterministic backends that produce a single solution in one pass declare `solution_count = 1` or leave the field absent.

**3.7.4 Extension metadata obligations:**

- A backend may return backend-specific metadata in `extension_metadata` (SPEC-004 FR-13, `map<string, string>`).
- All keys a backend uses in `extension_metadata` must be documented in the individual solver specification. Undocumented keys are permitted but should not be treated as contract-stable across `specification_version` changes.
- `extension_metadata` must not contain routing problem data (raw coordinates, demands, time windows, or other problem input fields). Evidence integrity depends on the problem data being accessed exclusively from the canonical routing problem record (SPEC-006 FR-2.4).
- Consumers silently ignore unrecognized `extension_metadata` keys (SPEC-004 FR-13).
- Individual solver specifications that use no `extension_metadata` must include an explicit statement: "Extension metadata: none."

**Acceptance Criteria:**
- Every individual solver specification states whether it uses `extension_metadata` and documents all keys it uses
- No individual solver specification permits route plan presence under `Infeasible` or `Failed`
- Every individual solver specification confirms that `execution_duration_ms` is always populated

---

### FR-8: Failure Model

**Description:**
All backends must implement a defined set of failure outcomes and must use the failure codes defined in SPEC-004 FR-8. This section defines the failure obligations that apply at the framework level; individual solver specifications may document backend-specific failure conditions in addition to these.

**3.8.1 Required failure outcomes:**

All backends in all categories must support `Timeout`, `Cancelled`, and `Failed` (FR-5). The behavioral obligations for each:

- **`Timeout`:** The backend exceeded the `execution_timeout_ms` budget provided in the SolverRequest. The backend should self-terminate before or at the `execution_timeout_ms` deadline (SPEC-004 FR-10). The Worker enforces an external timeout as the authoritative backstop (SPEC-005); backend self-termination is preferred to allow the backend to include in the response any complete solution found before the deadline. A self-terminating backend sets `outcome = Timeout` and `failure_code = ExecutionTimeout`. If a complete RoutePlan satisfying all SPEC-004 FR-5 structural validity requirements was found before the deadline, it may be included in the response.
- **`Cancelled`:** An external cancellation signal was received from the Worker during solver execution. The backend returns `Cancelled`. If a complete RoutePlan satisfying all SPEC-004 FR-5 structural validity requirements was found before the cancellation signal was received, it may be included in the response.
- **`Failed`:** An internal error prevented execution from completing or producing a valid route plan. The backend must return a `SolverFailureDetail` with a populated `failure_code` and `failure_message`.

**3.8.2 Failure codes:**

Backends must use the failure codes defined in SPEC-004 FR-8. No backend may invent failure codes outside this set.

| Code | Usage |
|---|---|
| `InfeasibleProblem` | Exact solvers only; proves no feasible solution exists. Prohibited for heuristic backends. |
| `ExecutionTimeout` | Use when returning `Timeout`. The Worker may also populate this code when constructing a synthetic response for an unresponsive backend (SPEC-005). |
| `ExecutionCancelled` | Use when returning `Cancelled`. |
| `ContractVersionMismatch` | Use when the `contract_version` in the SolverRequest does not match the backend's `supported_contract_version`. |
| `InternalError` | Catch-all for backend-internal errors. Must be accompanied by a diagnostic `failure_message`. |

**3.8.3 ContractVersionMismatch obligation:**

All backends must validate the `contract_version` field in the SolverRequest against their declared `supported_contract_version`. If they do not match, the backend must immediately return `Failed` with `failure_code = ContractVersionMismatch` and a `failure_message` identifying the received version and the expected version. This validation must occur before any routing problem processing begins.

**3.8.4 Failure detail requirements:**

- All `Failed` outcomes must include a `SolverFailureDetail` with both `failure_code` and `failure_message` populated.
- `failure_message` must be diagnostic (identifying what failed and why, in developer-readable terms), not user-facing.
- `Timeout` and `Cancelled` outcomes are not required to include `SolverFailureDetail` but may include one for diagnostic purposes.

**Acceptance Criteria:**
- Every individual solver specification enumerates its backend-specific failure conditions
- No heuristic backend specification permits `InfeasibleProblem` as a failure code
- Every solver specification states that `ContractVersionMismatch` validation occurs before routing problem processing

---

### FR-9: Observability Requirements

**Description:**
ADR-006 governs the observability stack (OpenTelemetry SDK, Collector, Prometheus, Grafana). ADR-011 governs W3C TraceContext propagation across the API → RabbitMQ → Worker boundary. This section defines observability obligations at the backend specification level — what backends must provide and what they must not do.

**3.9.1 The `solver.execute` span:**

The `solver.execute` span is emitted by the Worker, not by the backend (SPEC-004 FR-15). The Worker constructs this span using data from the SolverResponse.

Required `solver.execute` span attributes and their sources:

| Attribute | Source |
|---|---|
| `backend_id` | SolverRequest.backend_id |
| `job_id` | SolverRequest.job_id |
| `decision_id` | SolverRequest.decision_id |
| `problem_size_class` | Workload features (SPEC-010 FR-3.2) |
| `outcome` | SolverResponse.outcome |
| `execution_duration_ms` | SolverResponse.statistics.execution_duration_ms |
| `stop_count` | Routing problem stop count |

The backend provides `outcome` and `execution_duration_ms` through the SolverResponse; all other attributes are available to the Worker independent of the backend.

**3.9.2 Backend observability obligations:**

- The backend must return an accurate `outcome` value in the SolverResponse. This value is the primary observability signal for solver execution events.
- The backend must return an accurate `execution_duration_ms` in ExecutionStatistics. This value must reflect actual wall-clock execution time within the backend process.
- The backend **must not** emit OpenTelemetry spans, metrics, or structured logs directly. Span emission is a Worker responsibility. Backends must not write to stdout or stderr; all diagnostic output must be returned through the SolverResponse contract (`SolverFailureDetail.failure_message` for errors, `extension_metadata` for backend-specific signals).

**3.9.3 Backend extension_metadata as diagnostic channel:**

Backends may use `extension_metadata` to communicate backend-specific diagnostic information that the Worker may include in structured logs or pass to the Evidence Log. All keys used for this purpose must be documented in the individual solver specification (FR-7.4). Examples for stochastic backends might include annealing parameters, iteration counts, or energy values.

**Acceptance Criteria:**
- No individual solver specification permits the backend to emit OpenTelemetry spans
- Every individual solver specification confirms that `execution_duration_ms` reflects actual backend wall-clock execution time
- No individual solver specification permits the backend to write to stdout or stderr

---

### FR-10: Solver Lifecycle and Registration Requirements

**Description:**
A backend is not eligible for Scheduler selection until it satisfies all of the following prerequisites. No backend may be included in the Scheduler's eligible pool until its individual solver specification is Accepted and its capability profile is registered.

**3.10.1 Prerequisites for a new backend:**

| Prerequisite | Description | Owner |
|---|---|---|
| 1. Write and accept an individual solver specification | A specification conforming to FR-12 must exist and be in Accepted status. The specification is a child of SPEC-011. Implementation is not authorized until this prerequisite is satisfied. | Individual solver specification |
| 2. Declare a complete capability profile | The accepted solver specification must document all capability profile fields (FR-4). Declared values must be accurate (FR-4.2). | Individual solver specification |
| 3. Document seed usage policy | The accepted solver specification must include a Seed Usage Policy section (FR-6.5, SPEC-004 FR-1). | Individual solver specification |
| 4. Document supported SolverOutcome values | The accepted solver specification must declare its supported SolverOutcome values in an explicit table (FR-5.3, SPEC-004 FR-1). | Individual solver specification |
| 5. Document extension_metadata keys | If extension_metadata is used: all keys documented in the accepted solver specification (FR-7.4). If not used: explicit "none" statement. | Individual solver specification |
| 6. Implement the SolverContract | After Prerequisites 1–5 are satisfied (specification Accepted), the backend must implement the SolverContract C++ abstract type (SPEC-004). For Python adapter backends: the transport protocol must be resolved (ADR-005 OQ-1) before this step is addressable. | Backend implementation |
| 7. Register the capability profile | The capability profile must be registered through the mechanism resolved by SPEC-003 OQ-2. | SPEC-003 OQ-2 resolution |

**3.10.2 Backend removal:**

A backend may be removed from the Scheduler's eligible pool by marking `is_provisional = true` (soft removal, allowing existing in-flight jobs to complete) or by deregistering the capability profile. Backend removal is a configuration and deployment concern, not a specification concern.

**Acceptance Criteria:**
- No backend proceeds to implementation without an Accepted individual solver specification (Prerequisite 1 from FR-10.1)
- Every Accepted individual solver specification satisfies Prerequisites 2–5 from FR-10.1
- SPEC-003 OQ-2 is resolved before any backend capability profile is registered (Prerequisite 7 from FR-10.1)

---

### FR-11: MVP Backend Inventory

**Description:**
The following backends are in scope for the MVP. This inventory is authoritative for the set of individual solver specifications that must be written before MVP implementation begins.

**3.11.1 MVP backends:**

| Backend ID | Display Name | Category | Solver Specification Status |
|---|---|---|---|
| `nearest-neighbor` | Nearest Neighbor Heuristic | `classical_deterministic` | Accepted — SPEC-013 |
| `greedy-insertion` | Greedy Insertion Heuristic | `classical_deterministic` | Accepted — SPEC-014 |
| `qubo-simulated-annealing` | QUBO Simulated Annealing | `quantum_inspired_stochastic` | Accepted — SPEC-015 |

**3.11.2 Deferred backends:**

| Backend | Deferral Reason | Governing ADR |
|---|---|---|
| Python Adapter | Transport protocol resolved — SPEC-017 (Accepted 2026-06-20) defines JSON over HTTP transport and the full adapter behavioral contract, resolving ADR-005 OQ-1. ADR-005 is Accepted. Individual Python backend specifications (SPEC-018+) may now be written; each will declare an algorithmic category from FR-2.1 (see FR-2.4) and must conform to SPEC-011 framework requirements through the mechanisms defined in SPEC-017. Python backend support exists; no individual Python backend is yet specified. | ADR-005, SPEC-017 |
| Quantum Hardware | Out of MVP scope. Quantum hardware backends are non-reproducible by nature (hardware entropy); the architectural implications of a non-reproducible backend on the evidence system are deferred beyond MVP. | ADR-007 |

**Acceptance Criteria:**
- All three MVP backends have Accepted individual solver specifications before any backend implementation begins
- Python adapter and quantum hardware backends are not included in any MVP capability profile registration

---

### FR-12: Required Structure for Individual Solver Specifications

**Description:**
Each individual backend solver specification is a child of SPEC-011. It must conform to this structural requirement. An individual solver specification that omits any section listed here is incomplete and cannot be Accepted.

**3.12.1 Required sections:**

| Section | Content Requirement |
|---|---|
| **Metadata** | All FR-3 metadata fields plus standard spec fields (Feature ID, Title, Status, Author, Created, Last Modified, Dependencies, Blocks) |
| **Problem Statement** | What optimization problem this backend solves; its algorithmic approach at a high level; why this backend belongs in the MVP inventory |
| **Algorithm Description** | High-level description of the algorithm's approach (construction strategy, search strategy, termination conditions). Implementation planning owns pseudocode and complexity analysis. |
| **Capability Profile Declaration** | A complete table of all FR-4 capability profile fields. Must include the accuracy basis for `latency_profile` and `quality_profile`. |
| **Seed Usage Policy** | Per FR-6.5. Satisfies SPEC-004 FR-1. |
| **Supported SolverOutcome Values** | An explicit table of supported outcomes per FR-5.3. Satisfies SPEC-004 FR-1. |
| **RoutePlan Output Requirements** | Any backend-specific constraints on the route plan beyond FR-7.1. If none: explicit statement. |
| **Failure Model** | Backend-specific failure conditions beyond FR-8; any `failure_message` patterns for `InternalError` conditions. |
| **Extension Metadata** | All `extension_metadata` keys used by this backend and their semantics. If none: explicit "Extension metadata: none." |
| **Performance Characteristics** | Expected execution behavior (latency, solution quality) per supported size class. This is the empirical basis for `latency_profile` and `quality_profile` declarations. |
| **Testability** | How correctness, reproducibility, outcome compliance, and capability profile accuracy are verified for this backend. |
| **Open Questions** | Any unresolved questions specific to this backend, with classification (Implementation Planning Decision, Project Owner Decision Required, or Non-Blocking). |
| **Acceptance Checklist** | All SPEC-011 framework obligations checked; backend-specific acceptance criteria listed. |
| **Definition of Done** | Conditions under which this backend's implementation is complete. |

**Acceptance Criteria:**
- Every individual solver specification submitted for review contains all sections from FR-12.1
- No individual solver specification contradicts a framework requirement from FR-1 through FR-11

---

## Non-Requirements

SPEC-011 does not define and must not be extended to include:

- Solver algorithms or pseudocode (implementation planning concern; individual solver specifications provide high-level algorithm descriptions only)
- Scheduler backend selection policy, objective scoring formulas, or confidence score calculation (SPEC-003)
- Workload feature computation (SPEC-010)
- Routing problem validation (SPEC-001, ADR-009)
- Worker execution lifecycle (SPEC-005)
- Evidence persistence schemas or quality evaluation record schemas (SPEC-006, SPEC-007)
- Report generation (SPEC-009)
- API, queue, or Worker behavior
- Python adapter transport protocol (ADR-005, deferred)
- Quantum hardware execution model (ADR-007, deferred)
- QUBO energy landscape representation or annealing schedule parameters (QUBO solver specification and implementation planning)
- PCG64 stream constant values for specific backends (individual solver specifications)
- PRNG draw ordering for specific backends (individual solver specifications)
- Capability profile registration mechanism (SPEC-003 OQ-2)
- Backend qualification criteria for `is_provisional = false` (see OQ-4)

---

## Assumptions

1. All MVP backends are implemented in C++ and accessed through the SolverContract C++ interface directly. No adapter transport layer is required for MVP in-process backends. The Python adapter backend (ADR-005) is deferred past MVP.

2. The Scheduler capability profile registration mechanism (SPEC-003 OQ-2) will be resolved before any backend capability profile is registered (Prerequisite 7 from FR-10.1). SPEC-011 FR-4 documents the capability profile values that must be declared; how they enter the system is SPEC-003's responsibility.

3. No MVP backend requires proving infeasibility. The problem sizes and structure in scope for MVP are addressable by construction and stochastic heuristics.

4. Backend `latency_profile` and `quality_profile` values in capability profiles are derived from offline empirical measurement on representative benchmark problems of each supported size class before initial registration. Runtime dynamic adjustment of declared profiles is out of scope for MVP.

5. All MVP backends are executed within the same C++17 toolchain and Docker Compose Linux environment established by ADR-001. The IEEE 754 double precision and PCG64 cross-platform reproducibility guarantees of ADR-010 apply in this environment.

6. The PCG64 reference implementation from pcg-random.org (or equivalent header-only library) will be adoptable as a project dependency without licensing conflict. This is confirmed as an implementation planning concern by ADR-010.

---

## Constraints

- All `quantum_inspired_stochastic` backends must use PCG64 for reproducibility-critical PRNG operations. No other PRNG algorithm is permitted in reproducibility-critical paths (ADR-010 Decision 1). `quantum_hardware` backends are exempt from PCG64 obligations; their randomness requirements are governed by ADR-007.
- All backends must implement the normalized SolverContract (ADR-008). No backend-specific logic is permitted in the Scheduler.
- `Infeasible` outcome is reserved for exact solvers. Heuristic backends (`classical_deterministic`, `quantum_inspired_stochastic`) must not return `Infeasible` (SPEC-004 FR-4, FR-5.2).
- `execution_seed` is derived by the Worker from `RoutingProblem.seed` (SPEC-005 FR-7, ADR-010 Decision 4, ODR-4). Backends consume the provided `execution_seed`; they do not derive seeds.
- The `solver.execute` span is emitted by the Worker (SPEC-004 FR-15). Backends must not emit OpenTelemetry spans.
- The `contract_version` field in every SolverRequest must be validated by the backend before processing begins. A mismatch returns `ContractVersionMismatch` (FR-8.3).
- A backend's `backend_id` is frozen once its individual solver specification is Accepted. Renaming a backend is equivalent to introducing a new backend.
- Individual solver specification PCG64 stream constants are frozen once the specification is Accepted. Changing a stream constant requires the full ADR-010 Decision 5 breaking change procedure, which includes updating ADR-010 with an explicit change record documenting the previous and replacement value. A `specification_version` increment alone is insufficient.

---

## Architectural Impact

| Component | Impact | Notes |
|---|---|---|
| Domain Layer (C++ Core / backends) | Yes | SPEC-011 defines the behavioral requirements for all C++ backend implementations |
| Scheduler (SPEC-003) | Yes | FR-4 defines what backend specifications must declare; SPEC-003 FR-4 defines what the Scheduler consumes. No change to SPEC-003 schema required; the obligation is on the backend specification side. |
| Solver Contract (SPEC-004) | Yes | Each solver specification satisfies SPEC-004 FR-1 obligations (seed usage policy, supported outcomes). No schema changes to SPEC-004. |
| Worker (SPEC-005) | No | SPEC-011 does not alter Worker lifecycle behavior. |
| Evidence Log (SPEC-006) | No | SPEC-011 does not alter evidence persistence schemas. |
| Quality Evaluation (SPEC-007) | No | SPEC-011 does not alter quality evaluation behavior. |
| API Layer | No | |
| Persistence | No | |
| Infrastructure | No | |
| Configuration | Yes | Capability profile field documentation informs Scheduler configuration. SPEC-003 OQ-2 resolution determines how capability profiles enter configuration. |
| Observability | Yes | FR-9 defines backend observability obligations consistent with ADR-006 and SPEC-004 FR-15. No new instrumentation required beyond the Worker-emitted `solver.execute` span. |
| Security | No | |
| Deployment | No | |

---

## Testability

**Framework-level testability (SPEC-011):**

SPEC-011 is a framework specification; its testability requirements are expressed through the individual solver specifications. The following test obligations must be included in every individual solver specification.

**Required test coverage in individual solver specifications:**

| Test Category | Obligation |
|---|---|
| Contract validation | The backend returns `ContractVersionMismatch` for a SolverRequest with a mismatched `contract_version`, before processing the routing problem. |
| Reproducibility (stochastic backends) | Given two SolverRequests with identical routing problem and identical `execution_seed`, the backend returns structurally identical SolverResponses (same `outcome`, same route assignment). |
| Determinism (deterministic backends) | Given two SolverRequests with identical routing problem, the backend returns structurally identical SolverResponses regardless of `execution_seed` value. |
| Outcome compliance | The backend never returns `Infeasible` (for heuristic backends). The backend never returns a route plan under `Failed` or `Infeasible`. |
| RoutePlan structural validity (`Succeeded`) | All stops assigned exactly once. Route count = vehicle_count (empty routes are valid). Capacity per route ≤ capacity_per_vehicle. Depot not present. |
| Timeout behavior | The backend returns `Timeout` before `execution_timeout_ms` is significantly exceeded. If a complete RoutePlan satisfying all SPEC-004 FR-5 structural validity requirements was found before the deadline, it is present in the response. If no complete solution was found before the deadline, route plan is absent. |
| Extension metadata | All documented `extension_metadata` keys are present in the SolverResponse when expected conditions are met. |
| Capability profile accuracy | Empirical execution duration on representative problems falls within the declared `latency_profile` bounds. |

---

## Security Considerations

SPEC-011 does not introduce security surface beyond what is established in SPEC-004. The normalized SolverContract (ADR-008) isolates backends from direct access to infrastructure, queues, or external systems. Backends receive a routing problem and return a route plan; they have no access to the Evidence Log, API, or persistence layer.

The prohibition on backends writing to stdout or stderr (FR-9.2) prevents diagnostic output from leaking routing problem data outside the controlled SolverResponse channel.

`extension_metadata` must not contain routing problem raw data (FR-7.4). This constraint prevents problem data from escaping the canonical routing problem record (SPEC-001) through an uncontrolled side channel.

---

## Performance Considerations

SPEC-011 does not set performance targets for individual backends. Performance characteristics are documented in each individual solver specification and materialized as `latency_profile` and `quality_profile` declarations in the capability profile (FR-4). The Scheduler uses these values for objective scoring (SPEC-003 FR-7).

Framework-level performance obligations:
- The `ContractVersionMismatch` validation (FR-8.3) must be the first check a backend performs. It must not incur routing problem deserialization or memory allocation before this check completes.
- `execution_duration_ms` in ExecutionStatistics must be measured from the start of optimization (after any internal initialization) to the point at which the route plan is finalized. It must not include SolverRequest deserialization or SolverResponse serialization time, as those are Worker-owned overhead.

---

## Documentation Updates Required

**SPEC-004 FR-1 (Solver Contract — Responsibility Table):**
- SPEC-004 FR-1 includes a note: "Document seed usage policy in solver specification." SPEC-011 FR-6.5 formalizes this obligation as a required section in every individual solver specification. No change to SPEC-004 text is required. SPEC-011 is the authoritative owner of the documentation structure for this obligation.

**SPEC-004 FR-1 (Solver Contract — Responsibility Table):**
- SPEC-004 FR-1 includes a note: "Identify supported SolverOutcome values in solver specification." SPEC-011 FR-5.3 formalizes this as a required section. No change to SPEC-004 text is required.

**SPEC-003 FR-4 (Scheduler — Backend Capability Profile):**
- SPEC-003 FR-4 defines the Backend Capability Profile schema consumed by the Scheduler. SPEC-011 FR-4 defines the obligation on solver specifications to declare complete and accurate capability profiles. No structural change to SPEC-003 FR-4 is required.

**ADR-008 (Solver Contract and Backend Neutrality):**
- ADR-008 must be advanced to Accepted status before SPEC-011 advances to Accepted. SPEC-004 (Accepted) identified ADR-008 advancement as a follow-on action following SPEC-004 acceptance (see SPEC-004 Documentation Updates Required). This is a governance prerequisite — ADR-008's Backend Neutrality principle is architecturally operative and is the authoritative basis for FR-1.2, FR-12, and the Constraints section of this specification. The required action is a status change to ADR-008; no content change to SPEC-011 is required.

**Individual backend solver specifications (to be created per FR-11, FR-12):**
- `docs/specs/SPEC-013-nearest-neighbor-solver.md` — `nearest-neighbor` backend
- `docs/specs/SPEC-014-greedy-insertion-solver.md` — `greedy-insertion` backend
- `docs/specs/SPEC-015-qubo-simulated-annealing-solver.md` — `qubo-simulated-annealing` backend

Each specification is a child of SPEC-011 and must conform to FR-12. These specifications must be Accepted before implementation of the corresponding backend begins (FR-10).

---

## Open Questions

### OQ-1: PCG64 Stream Constants for Stochastic Backends

**Classification:** Implementation Planning Decision (per-backend)

**Question:** What PCG64 stream constants (increment values) should be assigned to the `qubo-simulated-annealing` backend and any future stochastic backends?

**Context:** ADR-010 Decision 1 requires that stream constants be documented in each stochastic component specification and treated as frozen once the specification is Accepted. Stream constants must be selected with statistical independence in mind; if future development runs multiple stochastic backends simultaneously or in sequence on the same problem seed, distinct stream constants prevent correlation in their PRNG output sequences. The specific value for each backend is an individual solver specification concern, not a SPEC-011 concern.

**Blocking:** Does not block SPEC-011 acceptance. Blocks each stochastic backend's individual solver specification from being Accepted until the stream constant is determined.

---

### OQ-2: Capability Profile Registration Mechanism

**Classification:** Inherited Open Question (SPEC-003 OQ-2)

**Question:** What mechanism do backends use to register their capability profiles at runtime — static configuration file, startup registration code, compile-time registration macro, or runtime registration service?

**Context:** SPEC-003 OQ-2 is open. SPEC-011 FR-4 defines what must be declared in a capability profile; it does not define how that profile enters the Scheduler's registry. The registration mechanism determines how a new backend is "installed" and how the Scheduler discovers available backends at startup.

**Blocking:** Does not block SPEC-011 acceptance. Blocks all backend capability profile registrations until SPEC-003 OQ-2 is resolved. Does not block implementation of the SolverContract (Prerequisite 6 from FR-10.1).

---

### OQ-3: Classical Deterministic Backend Timeout Self-Termination Strategy

**Classification:** Implementation Planning Decision (per-backend)

**Question:** How should classical deterministic heuristics self-terminate before the `execution_timeout_ms` deadline, given that construction heuristics typically have no natural checkpoint loop?

**Context:** FR-8.1 states that a backend should self-terminate before or at the `execution_timeout_ms` deadline (consistent with SPEC-004 FR-10, which is advisory). The Worker enforces an external timeout as the authoritative backstop (SPEC-005). For construction heuristics (`nearest-neighbor`, `greedy-insertion`), the algorithm processes stops sequentially or in a priority structure. If a complete solution is found before the deadline, it may be included in the `Timeout` response (satisfying SPEC-004 FR-5 structural validity). If no complete solution was found when the deadline arrives, the route plan is absent. The specific self-termination approach — whether to poll elapsed time, whether to attempt early completion when time is short, or to rely entirely on the Worker backstop — is an implementation planning concern for each backend specification.

**Blocking:** Does not block SPEC-011 acceptance. Each classical deterministic backend specification (SPEC-013, SPEC-014) should address its specific self-termination approach.

---

### OQ-4: Backend Qualification Criteria for `is_provisional = false`

**Classification:** Project Owner Decision Required

**Question:** What conditions must a backend satisfy to declare `is_provisional = false` in its capability profile, qualifying it for inclusion in Scheduler eligibility phases that exclude provisional backends?

**Context:** SPEC-003 FR-4 defines `is_provisional` as a boolean. SPEC-003 FR-5 Phase 2 eligibility excludes provisional backends under some objective modes. FR-4.2 requires an accuracy basis for capability profile values. The criteria for transitioning from provisional to qualified — whether empirical quality threshold, volume of benchmark runs, code review gate, or development lifecycle designation — are not defined in SPEC-003 or SPEC-011. All MVP backends should declare `is_provisional = true` initially; the qualification gate determines when they may declare `is_provisional = false`.

**Blocking:** Does not block SPEC-011 acceptance. Blocks meaningful use of the Scheduler's provisional-backend exclusion policy in FR-5 Phase 2 eligibility.

---

### OQ-5: Partial Assignment Support Under Timeout and Cancelled

**Classification:** Inherited Open Question (SPEC-004 OQ-3)

**Question:** Should backends be permitted to return a RoutePlan with some stops unassigned (a "partial assignment") under `Timeout` or `Cancelled` outcomes in the future?

**Context:** SPEC-004 FR-5 is the current authoritative prohibition: any RoutePlan present under any outcome — including `Timeout` and `Cancelled` — must assign all stops. FR-7.1 reflects this prohibition: a RoutePlan is optional under `Timeout` and `Cancelled`, but if present, it must satisfy all SPEC-004 FR-5 structural validity requirements. SPEC-004 OQ-3 asks whether this prohibition should be relaxed in the future. Structural questions such as enumeration of unassigned stops, capacity validity for partial routes, and SPEC-007 quality evaluation applicability to partial plans are all contingent on SPEC-004 OQ-3 resolution. Individual backend specifications should not anticipate partial assignment support until SPEC-004 OQ-3 is resolved.

**Blocking:** Does not block SPEC-011 acceptance. Permitting partial assignments requires SPEC-004 OQ-3 resolution and a corresponding update to SPEC-004 FR-5 before any backend specification may declare partial assignment support.

---

## Acceptance Checklist

- [ ] FR-1: Responsibility boundaries defined and consistent with SPEC-003, SPEC-004, SPEC-005, SPEC-006; no overlapping or contradictory ownership claims
- [ ] FR-2: Solver classification model defines all MVP categories; future categories identified; classification determines FR-5 outcome support and FR-6 reproducibility obligations
- [ ] FR-3: Solver metadata fields complete; `backend_id` stability obligation stated; `specification_version` increment obligations defined
- [ ] FR-4: Capability declaration requirements consistent with SPEC-003 FR-4 field set; accuracy obligation stated; dependency on SPEC-003 OQ-2 acknowledged; `supported_contract_version` cross-reference correct
- [ ] FR-5: Supported SolverOutcome matrix complete; `Infeasible` restriction for heuristic backends consistent with SPEC-004 FR-4; declaration obligation stated
- [ ] FR-6: Reproducibility requirements consistent with ADR-010; seed usage policy (FR-6.1), PRNG algorithm (FR-6.2), distribution sampling (FR-6.3), floating-point determinism (FR-6.4), and documentation obligation (FR-6.5) all covered
- [ ] FR-7: Solver output requirements consistent with SPEC-004 FR-3 through FR-7; RoutePlan presence table consistent with FR-5; extension_metadata restriction consistent with SPEC-004 FR-13
- [ ] FR-8: Failure model consistent with SPEC-004 FR-8; all failure codes documented; `ContractVersionMismatch` validation obligation stated; `InfeasibleProblem` restriction consistent with FR-5
- [ ] FR-9: Observability requirements consistent with ADR-006 and SPEC-004 FR-15; Worker-emitted span model preserved; backend span prohibition stated
- [ ] FR-10: All seven registration prerequisites listed; implementation authorization gate stated
- [ ] FR-11: MVP backend inventory consistent with architecture.md; deferred backends cite correct ADRs (ADR-005, ADR-007)
- [ ] FR-12: Required solver specification structure complete; all sections listed; conforms to template
- [ ] OQ-1 through OQ-5: Each open question classified (Implementation Planning Decision, Project Owner Decision Required, or Inherited Open Question); blocking status stated
- [ ] No open questions with unknown classification remain

---

## Definition of Done

- SPEC-011 is Accepted
- SPEC-003 OQ-2 (capability profile registration mechanism) is resolved before any backend capability profile is registered (Prerequisite 7 from FR-10.1)
- Three individual backend solver specifications conforming to FR-12 are written and Accepted:
  - `nearest-neighbor` (`classical_deterministic`)
  - `greedy-insertion` (`classical_deterministic`)
  - `qubo-simulated-annealing` (`quantum_inspired_stochastic`)
- Each MVP backend's individual solver specification is Accepted before implementation of that backend begins
- OQ-4 (backend qualification criteria for `is_provisional = false`) is resolved by Project Owner Decision before any backend declares `is_provisional = false` in its capability profile
