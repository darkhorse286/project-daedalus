# ADR-014: Browser Client Architecture

## Status

Accepted

**Date:** 2026-06-26

**Related Feature(s):** Web UI (SPEC-021), Experiment Dashboard (SPEC-022)

**Related ADR(s):** ADR-002, ADR-006, ADR-009, ADR-011, ADR-012, ADR-013

---

# Context

SPEC-021 (Web UI) and SPEC-022 (Experiment Dashboard) are the two browser-facing capabilities of Project DAEDALUS. Both specifications are accepted into governance. Both define functional requirements implementable against the existing SPEC-008 API contract without modification to any backend component.

Four structural questions remain unresolved across the two specifications.

**Application boundary.** SPEC-022 OQ-1 asks whether the Experiment Dashboard is implemented as a view set within the SPEC-021 Web UI application or as a separately deployed browser application. Neither specification can answer this question. It is an architectural decision above the feature specification level.

**Technology stack.** SPEC-021 OQ-2 defers the browser-side technology choice to an ADR. SPEC-022 OQ-2 is resolved by the same ADR if OQ-1 selects a unified application. The technology choice has portfolio, toolchain, and Docker Compose implications.

**Deployment model and origin.** SPEC-021 OQ-3 identifies three serving options: a dedicated Docker Compose container, assets served from the API container, or a developer-served process outside Docker Compose. The serving mechanism determines the CORS origin the API must permit and whether a new container appears in the topology.

**API base URL discovery.** SPEC-021 OQ-4 and SPEC-022 OQ-4 identify three candidate mechanisms for communicating the API base URL to the browser application: build-time environment variable, runtime configuration file, and same-origin derivation. The correct mechanism depends on the serving approach.

The System Context diagram and Container Topology in architecture.md contain no browser client nodes. SPEC-021 Architectural Impact and SPEC-022 Architectural Impact both note that architecture.md must be updated once OQ-1 and OQ-3 are resolved.

The CLI (SPEC-016) is the existing primary developer and automation interface. It does not interact with the browser client at any level. The browser client is a new system context node with no involvement in the experiment orchestration loop, the evidence collection path, or the solver execution pipeline.

---

# Decision

Eight architectural decisions govern the browser client in Project DAEDALUS.

## Decision 1: Single SPA — SPEC-021 and SPEC-022 Are Feature Areas in One Browser Application

SPEC-021 Web UI and SPEC-022 Experiment Dashboard are implemented as feature areas within a single browser application. They share one build artifact, one Docker Compose service, one React application tree, and one API client instance. They remain separate specifications because they define distinct product capabilities and ownership boundaries; that separation is a specification concern, not a deployment boundary.

The unified application resolves SPEC-022 OQ-1 (relationship to SPEC-021) as Option 1 (Dashboard views added to the Web UI application). SPEC-022 OQ-2 (technology stack) is answered by Decisions 2, 3, and 4.

## Decision 2: React with TypeScript, Built with Vite

React with TypeScript is the browser client framework and language. Vite is the build tooling.

React with TypeScript provides a production-grade component model, strong type safety for API response shapes, and an ecosystem suited to the UI patterns required by SPEC-021 and SPEC-022: form-heavy job submission with dynamic stop rows, auto-polling status views, complex data tables with custom cell logic, evidence report sandboxed rendering, and file download. TypeScript enforces API response contract discipline at compile time, a direct alignment with the project's evidence-integrity principle.

Vite provides fast hot-module replacement, native TypeScript support, build-time environment variable injection via the `import.meta.env.VITE_*` convention, and static asset optimization. Vite's environment variable convention is the mechanism used by Decision 6 (API base URL discovery).

No server-side rendering is adopted. The browser client is a pure static SPA. SSR adds a Node.js server runtime to the container without benefit for a developer-tool UI backed by a local API on localhost.

## Decision 3: UI Component Strategy — shadcn/ui (Radix UI + Tailwind CSS) with TanStack Table

UI components are sourced from shadcn/ui, which combines Radix UI headless primitives with Tailwind CSS utility styling. TanStack Table is used for the data-heavy tabular views required by SPEC-021 FR-8.2 (trial results table) and SPEC-022 FR-4 (Trial Matrix), FR-5 (Solver Comparison tables), and FR-9 (Execution Metadata table).

This combination provides:

- **Accessibility**: Radix UI primitives implement WAI-ARIA patterns for dialogs, selects, dropdowns, and interactive controls without custom ARIA authoring.
- **Control**: shadcn/ui components are copied into the project at initialization and modified without constraint from library APIs or theming systems. Cell-level behavior in the Trial Matrix — best-quality highlight, placeholder rendering for zero-evidence solvers — requires this level of control.
- **Tailwind CSS**: Utility-first styling eliminates CSS specificity conflicts that arise from mixed third-party component stylesheets.
- **TanStack Table**: Headless table logic decouples the row model, column definitions, sorting, and cell rendering from the DOM structure. The SPEC-022 FR-4 Trial Matrix layout — rows indexed by `(problem_config_index, repetition_index)`, columns by `backend_id`, one cell per intersection — is not a standard grid layout and requires a headless table to implement without constraint.

No full-featured component library (MUI, Mantine, Ant Design) is adopted. Full-featured libraries constrain styling through theming systems and expose complex component APIs that add friction at the cell-level customization required by the Trial Matrix and Solver Comparison views.

## Decision 4: State Management — TanStack Query for Server State, React Router for Navigation

TanStack Query is the server-state management layer. React Router is the client-side routing library.

TanStack Query handles fetching, caching, polling, and background revalidation:

- **Caching**: API responses are cached by query key (`[experiment, experiment_id]`, `[trials, experiment_id]`, `[job, job_id]`). Subsequent navigations to the same experiment reuse cached data without re-fetching.
- **Polling**: Auto-polling for non-terminal job and experiment status (SPEC-021 FR-6, FR-8.1; SPEC-022 FR-3, FR-4) is implemented as a `refetchInterval` parameter. Polling stops when status transitions to a terminal state by setting `refetchInterval: false` conditionally on status.
- **Cache sharing**: SPEC-022 FR-4 (Trial Matrix) and FR-9 (Execution Metadata View) share the same `[trials, experiment_id]` cache key. Both views are derived from one API response without a separate fetch, eliminating the temporal inconsistency between the two views identified in SPEC-022 Performance Considerations.

React Router provides URL-based navigation. Experiment ID, job ID, and benchmark ID are URL path parameters (`/experiments/:experimentId`, `/jobs/:jobId`). Breadcrumb derivation in SPEC-022 FR-11 and SPEC-021 navigation flows are driven by the route context without a separate breadcrumb store.

No Redux, Zustand, or global state management library is adopted at MVP scope. Server state is owned by TanStack Query. Navigation state is owned by React Router URL parameters. Local UI state uses React's built-in `useState` and `useReducer`. This division covers all state patterns required by SPEC-021 and SPEC-022 without additional libraries.

TanStack Query and React Router are selected at the ADR level rather than deferred to implementation planning because accepted specification requirements directly determine the required architectural behavior, not merely a preference between interchangeable alternatives. TanStack Query is the only practical implementation path for the polling lifecycle, terminal-state stop, and shared server-state caching defined by SPEC-021 FR-6, FR-8.1 and SPEC-022 FR-3, FR-4 and SPEC-022 Performance Considerations; no manual polling implementation or alternative library eliminates the cache-sharing requirement without equivalent complexity. React Router is the only practical implementation path for the URL-based experiment and job navigation, breadcrumb derivation, and direct-URL deep-linking described in SPEC-021 and SPEC-022; URL state management is an architectural pattern, not a library preference. Library choices within the React ecosystem that are not driven by accepted specification requirements remain implementation-planning decisions, consistent with ADR-002's approach of deferring choices that are not architecturally determined.

## Decision 5: Deployment Model — Dedicated web-ui Docker Compose Container

The browser client is deployed as a dedicated `web-ui` service in the Docker Compose environment. The `web-ui` container runs Nginx serving the Vite static build output (HTML, JS, CSS). The container is accessible at `http://localhost:3000` (host port 3000) by default.

The `web-ui` Dockerfile is a multi-stage build: a Node.js build stage executes `vite build` to produce the static output; the runtime stage copies that output into an Nginx base image. The final runtime image contains only Nginx and static files; the Node.js build environment does not appear in the runtime image.

The Nginx configuration applies the HTML5 History API fallback (`try_files $uri /index.html`) required for React Router client-side navigation. Without this, direct URL navigation (e.g., `http://localhost:3000/experiments/some-uuid`) returns HTTP 404 from Nginx instead of loading the SPA.

The `web-ui` container does not act as a reverse proxy for API requests. The browser client origin and the API origin are distinct; CORS headers are required on the API permitting the browser client origin. In the standard Docker Compose configuration, the browser client is at `http://localhost:3000` and the API is at `http://localhost:5000`. This is consistent with the CORS requirements stated in SPEC-021 Architectural Impact, SPEC-021 Assumptions, and SPEC-022 Architectural Impact.

The API container (port 5000) and all other Docker Compose services are unchanged. One Browser Client service (`web-ui`) is added to the Docker Compose deployment. This resolves SPEC-021 OQ-3 as Option 1 (separate `web-ui` container).

## Decision 6: API Base URL Discovery — Build-Time Environment Variable

The API base URL is supplied to the browser application as a build-time environment variable (`VITE_API_BASE_URL`). The value is embedded in the static build artifact during the Vite build step. The default value is `http://localhost:5000`, matching the `api` container's default published port.

The Docker Compose `web-ui` service definition specifies `VITE_API_BASE_URL` as a build argument passed to the Node.js build stage. Vite embeds the value into the compiled JavaScript via `import.meta.env.VITE_API_BASE_URL`. The browser application reads the embedded value at runtime without an additional HTTP request.

Changing the API base URL requires rebuilding the `web-ui` Docker image with the updated build argument. No runtime reconfiguration is supported at MVP scope. This constraint is acceptable: the Docker Compose environment defines the API port at Compose configuration time, and both the API and Web UI images are rebuilt together under normal development workflow.

This resolves SPEC-021 OQ-4 and SPEC-022 OQ-4 as Option 1 (build-time environment variable).

## Decision 7: Browser Observability — No OTel Spans, request_id Correlation

Browser clients do not emit OpenTelemetry spans. Browser clients do not inject `traceparent` or `tracestate` headers into API requests. API calls issued from the browser arrive at the API as trace roots; the API creates a new span context per inbound request, consistent with CLI behavior per ADR-011 and architecture.md CLI Observability.

The `request_id` field from SPEC-008 FR-14 error responses is displayed in the browser UI for all API error conditions, as required by SPEC-021 FR-9 and SPEC-022 FR-12. This enables operators to correlate browser-observed errors with API-side OTel spans and Prometheus metrics without browser-side trace propagation.

Browser-side OTel instrumentation adds SDK bundle weight, a browser-to-OTel-Collector network dependency, and initialization complexity. None of these costs are justified for a developer-tool UI operating in a local Docker Compose environment. ADR-011 established that trace continuity across the API-to-Worker boundary is satisfied by W3C TraceContext in AMQP headers; no trace gap exists at the browser-to-API boundary that instrumentation would close.

For successful API requests, the `job_id` returned in the HTTP 202 response body is displayed in the browser UI per SPEC-021 FR-2 and FR-6. This `job_id` is the operational correlation handle for locating the corresponding `job.submit` span in the API and all descendent Worker spans in the observability backend. This is consistent with ADR-011's treatment of `job_id` as a required attribute on both `job.submit` and `job.consume` spans (SPEC-008 FR-17, SPEC-005 FR-19) and its designation as a correlation handle sufficient for success-path operator queries.

## Decision 8: Future Evolution — Dashboard Extraction Is a Deferred Option

The Experiment Dashboard (SPEC-022) may in a future state be extracted into a micro-frontend, an independently deployed browser application, or a web component. Such extraction is an architectural option, not the current architecture. Any extraction requires a future ADR.

No micro-frontend infrastructure — module federation, import maps, shared dependency coordination across separately deployed bundles — is introduced at MVP scope. Decision 1 (single SPA) is adopted without forward-compatibility provisions for extraction. The added complexity of micro-frontend infrastructure is not justified by MVP requirements.

**Extraction invariant:** Any future extraction of the Experiment Dashboard into a separate deployment unit, micro-frontend, or web component shall preserve the API-only interaction model established by this ADR. Extracted Dashboard deployments remain passive consumers of the SPEC-008 API; they do not interact with the Control Plane, Scheduler, Worker, or any infrastructure component directly. This invariant is binding on any future ADR that governs Dashboard extraction.

## Browser Client Responsibilities

Browser clients own presentation and user interaction. Business rules, domain validation, scheduling decisions, evidence computation, and other domain behavior remain authoritative within the Control Plane and downstream services.

The browser client presents API-sourced data, performs structural input validation only (not domain validation, per ADR-009), orchestrates user interaction flows, and renders experiment evidence sourced from SPEC-008 endpoints without recomputation. The browser client does not perform domain validation, compute quality statistics or outcome distributions, make scheduling decisions, communicate with the Scheduler, Worker, or any infrastructure component directly, or bypass the Control Plane. This boundary reinforces ADR-009 (domain validation authority), ADR-012 (API as state authority, browser as consumer), and ADR-013 (backend targeting is a Control Plane decision).

---

# Alternatives Considered

## Alternative to Decision 1: Separate Deployed Application for SPEC-022

### Description

SPEC-022 Experiment Dashboard is deployed as an independent browser application distinct from the SPEC-021 Web UI, with its own Docker Compose service and npm dependency tree.

### Benefits

Independent deployment and versioning of the Dashboard. Dashboard releases do not require a Web UI rebuild.

### Drawbacks

SPEC-022 FR-4, FR-7, FR-9, FR-10, and FR-11 navigate to SPEC-021 FR-6 for job detail presentation. With separate browser applications, this navigation becomes a cross-origin redirect or requires duplicating SPEC-021 FR-6 in SPEC-022 — both of which conflict with the specifications' boundary definitions. Two Docker Compose containers, two build pipelines, and two CORS origins on the API are required. The portfolio signaling surface is unchanged; two separate applications do not signal more capability than one unified application.

### Reason Not Selected

The navigation dependency from SPEC-022 to SPEC-021 FR-6 is incompatible with independently deployed browser applications without cross-origin navigation (poor user experience) or view duplication (specification boundary violation). Infrastructure cost exceeds the capability benefit at MVP scale.

---

## Alternative to Decision 2: Vue.js with TypeScript

### Description

Vue.js 3 with TypeScript and Vite as the SPA framework, replacing React.

### Benefits

Smaller baseline bundle. Vue Composition API is ergonomic for reactive state management. Strong TypeScript integration. Vite is Vue's recommended build tool.

### Drawbacks

React has broader adoption in enterprise and startup engineering environments as a recognizable portfolio signal. The React ecosystem provides the widest selection of headless component primitives (Radix UI) and table libraries (TanStack Table). The functional difference between Vue and React for the UI patterns required by SPEC-021 and SPEC-022 is not significant.

### Reason Not Selected

Employer-signaling value of React is higher than Vue for the target audience of this portfolio project. Ecosystem breadth for the required UI patterns — headless components, complex table structures, polling abstractions — is wider on React.

---

## Alternative to Decision 2: Next.js (React with Server-Side Rendering)

### Description

Next.js as the React framework, providing SSR, file-system routing, and API route proxying within the same application container.

### Benefits

API route proxying eliminates CORS configuration. SSR provides faster initial render for users on slow networks.

### Drawbacks

SSR requires a Node.js server process in Docker Compose — a dynamic runtime rather than a static Nginx file server. All browser client content is loaded from the API after initial render; there is no content to pre-render. Next.js API routes proxy SPEC-008 requests, adding a layer between browser and API that duplicates the SPEC-008 contract without benefit. The deployment model expands from a static Nginx container to a Node.js process requiring process management.

### Reason Not Selected

SSR provides no rendering benefit for a developer-tool UI where all meaningful content is API-derived. The Node.js runtime in the deployment adds complexity that Decision 5 (static Nginx container) avoids. The CORS elimination benefit of API proxying does not justify the runtime dependency.

---

## Alternative to Decision 3: Full-Featured Component Library

### Description

A full-featured React component library — Material UI (MUI) or Mantine — provides buttons, forms, tables, modals, and layout components as pre-styled npm packages.

### Benefits

Faster initial implementation for standard form and table patterns. Consistent styling without Tailwind configuration. Strong accessibility baseline.

### Drawbacks

The SPEC-022 FR-4 Trial Matrix — rows indexed by `(problem_config_index, repetition_index)`, columns by `backend_id`, cells at the intersection with per-cell best-quality highlight and placeholder logic — does not map to standard data grid component APIs without significant customization. Full-featured library theming systems and CSS specificity interact with Tailwind utility classes in ways that require workarounds. Library version upgrades introduce breaking changes that require periodic migration effort.

### Reason Not Selected

The Trial Matrix structure requires a headless table library regardless of which component library is chosen. The component library adds a dependency that provides marginal value when a headless approach (TanStack Table + shadcn/ui) covers all required patterns with full control over cell rendering.

---

## Alternative to Decision 4: Redux Toolkit with RTK Query

### Description

Redux Toolkit as the global state manager with RTK Query as the data-fetching and caching layer.

### Benefits

Redux DevTools for state inspection. Predictable state update model. RTK Query provides caching and polling comparable to TanStack Query.

### Drawbacks

Redux requires a store, reducers, action creators, and slice configuration for state that is fundamentally server-derived and already managed by a fetching layer. For an application where all meaningful state is API-backed, TanStack Query eliminates Redux boilerplate without sacrificing observability. Redux is appropriate when substantial client-side business logic requires predictable, testable state transitions. SPEC-021 and SPEC-022 have no such logic; all domain logic resides in the API, the Worker, and the experiment harness.

### Reason Not Selected

TanStack Query is a direct fit for an API-heavy read-mostly application. The polling patterns (SPEC-021 FR-6, FR-8.1; SPEC-022 FR-3, FR-4), cache sharing requirement (SPEC-022 FR-4 and FR-9 sharing trial results), and background revalidation are first-class TanStack Query features. Redux overhead is not justified at this application's complexity level.

---

## Alternative to Decision 5: Serve Browser Assets from the API Container

### Description

Vite static build output is included in the ASP.NET Core API container image. ASP.NET Core serves the browser client at a sub-path (`/ui`). Browser origin and API origin are the same; CORS configuration is not required.

### Benefits

Single Docker Compose container for the API and browser client. No CORS configuration. API URL is same-origin; OQ-4 discovery is trivially solved by same-origin derivation.

### Drawbacks

The Web UI build artifact is coupled to the API container image. Changing a Tailwind class requires rebuilding and redeploying the API container. The Node.js build toolchain (Node.js, npm, Vite) must be integrated into the API's Dockerfile, adding a multi-stage build to what is currently a pure C# image. The `api` container's responsibilities expand beyond the Control Plane contract established by ADR-002.

### Reason Not Selected

Coupling the browser client build artifact to the API container violates ADR-002: the `api` container is the C# ASP.NET Core Control Plane. Adding frontend asset serving to the API container is an architectural boundary violation. Build pipeline coupling increases the cost of independent frontend changes. The CORS benefit is not proportionate to the boundary cost.

---

## Alternative to Decision 6: Runtime Configuration File

### Description

A `config.json` file served by Nginx alongside the static assets at a well-known path (`/config.json`). The browser application fetches this file at startup before issuing any API calls. The `apiBaseUrl` field specifies the API endpoint.

### Benefits

API URL reconfiguration without rebuilding the Docker image. Operators modify `config.json` or mount it as a Docker volume.

### Drawbacks

The browser application cannot issue any API call until `config.json` is fetched and parsed. This introduces an initialization loading state — a conditional rendering layer before the application is usable. The startup sequence becomes: load HTML → load JS → fetch `/config.json` → initialize API client → render application. At MVP localhost scope the latency is imperceptible, but the code complexity of the initialization gate is real. Volume mounting for runtime configuration is an operational complexity not warranted at MVP.

### Reason Not Selected

Build-time environment variable injection is simpler and sufficient for a Docker Compose developer environment where image rebuilds are standard practice. Runtime reconfiguration without rebuild is a production deployment concern, not an MVP developer-tool concern.

---

# Consequences

## Positive

- All six open questions governing browser client architecture (SPEC-021 OQ-2, OQ-3, OQ-4; SPEC-022 OQ-1, OQ-2, OQ-4) are resolved by a single ADR with a coherent, minimal-complexity architecture.
- SPEC-022 navigation to SPEC-021 FR-6 for job detail is a same-application internal route transition. No cross-origin navigation issue exists.
- TanStack Query's polling behavior (`refetchInterval: false` on terminal status) directly implements the auto-polling stop-on-terminal-state requirement in SPEC-021 FR-6, FR-8.1 and SPEC-022 FR-3, FR-4 without custom polling infrastructure.
- The `[trials, experiment_id]` cache key shared by SPEC-022 FR-4 and FR-9 satisfies the trial results cache consistency requirement in SPEC-022 Performance Considerations without manual cache coordination.
- Docker Compose topology addition is minimal: one `web-ui` Browser Client service. All existing Docker Compose services are unchanged.
- No browser-side OTel SDK dependency reduces bundle size and eliminates a tracing endpoint accessibility requirement from the browser origin.
- React + TypeScript provides the highest employer-signaling value of the candidate frameworks for this portfolio project.

## Negative

- CORS configuration on the API is required for the browser client origin before the browser client can issue any API request.
- Changing the API base URL requires rebuilding the `web-ui` Docker image with an updated build argument.
- Node.js build toolchain (Vite, npm) is added to the repository alongside C++, C#, and Python. Toolchain diversity increases onboarding surface.
- shadcn/ui component initialization copies component source files into the project. This is a one-time setup cost per component type and must be repeated when adding new component categories.

## Accepted Risks

- TanStack Query's cache invalidation correctness depends on query keys being consistently keyed by identifier. Incorrect or missing cache invalidation after a mutation (job cancellation returning HTTP 202) produces stale data in sibling views. This is a known implementation risk mitigated by the testability cases for SPEC-021 FR-6 cancellation behavior and SPEC-022 FR-4/FR-9 cache sharing.
- Nginx must be configured with `try_files $uri /index.html` for React Router history mode. Omitting this configuration causes direct URL navigation to return HTTP 404 from Nginx. This is a required deployment configuration noted for implementors.
- CORS permitting the browser client origin applies to the local Docker Compose environment only. Any internet-facing deployment must review and narrow the CORS policy to the specific deployment origin before exposure.

---

# Architectural Impact

| Component | Impact |
|---|---|
| Browser Client (SPEC-021, SPEC-022) | New component. Consumes SPEC-008 endpoints from a browser context. |
| API Layer (SPEC-008, ASP.NET Core) | CORS configuration required for the browser client origin. No endpoint contract changes. |
| Docker Compose Topology | One new Browser Client service: `web-ui` (Nginx). Default host port: 3000. All existing services unchanged. |
| CLI (SPEC-016) | None. Browser client has no interaction with CLI responsibilities or execution paths. |
| Worker (SPEC-005) | None. |
| Scheduler | None. |
| Observability (ADR-006) | Browser API calls produce spans owned by the API (SPEC-008 FR-17). No new browser-emitted spans. |
| Persistence (SPEC-012) | None. |
| Security | Browser surface introduced. No authentication at MVP. CORS scoped to localhost origin. Report rendering sandbox applies (SPEC-021, SPEC-022 Security Considerations). |
| System Context (architecture.md) | New browser client node (`Web UI / Dashboard`) required. Container Topology requires `web-ui` node. |

---

# Supporting Evidence

- **SPEC-021 FR-10 and SPEC-022 FR-1 Acceptance Criteria**: both specifications require that the browser client not emit OTel spans and not inject trace headers. Decision 7 is derived from these accepted acceptance criteria.
- **SPEC-022 FR-4 Trial Matrix layout**: rows by `(problem_config_index, repetition_index)`, columns by `backend_id`. This structure requires a headless table library. Decision 3 (TanStack Table) follows from this structural requirement.
- **SPEC-022 Performance Considerations (Trial results cache consistency)**: "the implementation should serve both views from the same cached API response." TanStack Query shared cache keys are the direct implementation of this requirement. Decision 4 (TanStack Query) is informed by this.
- **SPEC-021 FR-6 and FR-8.1, SPEC-022 FR-3 and FR-4 (auto-polling stop-on-terminal)**: polling ceases when job or experiment reaches `Completed` or `Failed`. TanStack Query `refetchInterval: false` conditioned on status is the direct implementation. Decision 4 is informed by these patterns.
- **ADR-002**: the `api` container is the C# ASP.NET Core Control Plane. Serving static browser assets from the API container would expand its responsibilities beyond the ADR-002 boundary. Decision 5 (separate container) preserves ADR-002.
- **ADR-011 and architecture.md CLI Observability**: "The CLI does not inject `traceparent` or `tracestate` headers into outgoing API requests. The API creates a new trace root per inbound request." Decision 7 (no browser trace injection) is consistent with this established posture for non-Worker API callers.
- **SPEC-021 OQ-3 Option 1**: "A dedicated container (e.g., Nginx serving static build output) added to Docker Compose. The container serves the Web UI at a distinct port (e.g., `http://localhost:3000`). The API must emit CORS headers for this origin." Decision 5 selects this option as described.
- **SPEC-022 OQ-1 Option 1**: "Dashboard views are added to the Web UI application. Single deployment unit, same origin, shared technology stack. No additional CORS configuration." Decision 1 selects this option. (No additional CORS origin is required for the Dashboard beyond what Decision 5 establishes for the unified application.)

---

# Assumptions

1. The Docker Compose build environment provides a Node.js runtime capable of running Vite and npm during the `web-ui` image build stage. The runtime image contains only Nginx and static build output; Node.js is not present in the runtime stage.
2. `http://localhost:5000` is the default and expected API base URL in the standard Docker Compose configuration. This matches the existing `api` service port mapping and Assumption 1 in SPEC-021 and SPEC-022.
3. CORS configuration on the ASP.NET Core API (permitting the browser client origin) is achievable via standard ASP.NET Core middleware configuration without any endpoint contract changes or new functional requirements.
4. React's concurrent rendering features and React Router's route parameter and loader APIs are forward-compatible with the component patterns required by SPEC-021 and SPEC-022. No React Server Component or React Server Action APIs are used.
5. TanStack Query's `refetchInterval` parameter accepts a function receiving the latest query data, enabling status-conditional polling stop (`refetchInterval: (data) => isTerminal(data?.status) ? false : 3000`). This is a documented and stable TanStack Query API.

---

# Limitations

- This ADR does not specify the exact Nginx configuration for the `web-ui` container beyond the `try_files` requirement noted in Accepted Risks. Specific `server` block configuration is an implementation planning detail.
- This ADR does not specify the npm package manager (npm, pnpm, or yarn). This is an implementation planning decision.
- This ADR does not specify the React Router route map structure beyond the URL parameter patterns described in Decision 4. The full route hierarchy is an implementation planning concern.
- This ADR does not specify the TanStack Query query key taxonomy beyond the examples given in Decision 4. Consistent key structure is an implementation discipline, not a decision governed here.
- SPEC-021 OQ-1 (Job List API Endpoint, `GET /v1/jobs`) and SPEC-022 OQ-3 (Experiment List Endpoint, `GET /v1/experiments`) are not resolved by this ADR. These questions govern API capabilities owned by SPEC-008 and its amendment process.
- Browser report rendering sandbox policy is not specified by this ADR. The browser architecture permits sandboxed rendering of HTML evidence reports in an embedded frame. The concrete sandbox policy — which script, navigation, and form-submission restrictions to apply — remains an implementation planning decision governed by SPEC-021 Security Considerations and SPEC-022 Security Considerations. Both specifications describe the sandboxing requirement and classify the specific policy as implementation planning.

---

# Documentation Updates

- **docs/architecture.md**: The System Context diagram requires a new browser client node (`Web UI / Dashboard`) with an arrow to the Daedalus API. The Container Topology diagram requires a new `web-ui` (Nginx) service node at port 3000. Both diagrams should be updated in a single revision after this ADR is accepted. The Governing Specifications section count should be updated to reflect the additional ADR.
- **SPEC-021**: OQ-2, OQ-3, and OQ-4 should be marked resolved, citing ADR-014. The Documentation Updates Required section of SPEC-021 notes that architecture.md and SPEC-008 must be updated; those updates are now governed by this ADR's documentation note and by the CORS consequence below. Additionally, because ADR-014 resolves SPEC-022 OQ-1 as a unified Browser Client application, SPEC-021 Documentation Updates Required should be expanded to note that SPEC-021 FR-8 (basic experiment observation views) and SPEC-022 structured visualization views must be reconciled during implementation planning — including the intentional behavioral difference in `executing`-trial rendering between SPEC-021 FR-8.1 (renders all trial status categories) and SPEC-022 FR-3 (suppresses the `executing` count as always-zero at MVP scope). This is an implementation-planning coordination item, not a specification amendment. Additionally, SPEC-021 FR-1 (Job Submission Form) should be assessed against ADR-013 during implementation planning to determine whether the browser job submission form exposes `backend_id` as a user-visible field. ADR-013 (Accepted) added an optional `backend_id` field to `POST /v1/jobs` applicable to all callers, including the browser client; standard-path job submission remains valid when `backend_id` is absent. Whether to expose this field in the browser UI is a SPEC-021 product decision, not an architectural decision governed by this ADR.
- **SPEC-022**: OQ-1, OQ-2, and OQ-4 should be marked resolved, citing ADR-014.
- **SPEC-008**: The CORS requirement is now specific: the API must permit the browser client origin. In the standard Docker Compose configuration, this is `http://localhost:3000`. A SPEC-008 amendment or Constraints update should record this as a deployment configuration requirement. No endpoint contract changes are required.

---

# Review Triggers

- A new browser-facing feature requires separate authentication, separate origin, or isolation from the SPEC-021/SPEC-022 view set. Decision 1 (single SPA) should be revisited.
- The project is extended to a multi-user or internet-facing deployment. CORS permitting a localhost origin is not appropriate in that context; Decision 5 and Decision 6 require revision.
- React reaches a major version with breaking changes to the concurrent rendering model or hook semantics that require a migration decision.
- TanStack Query introduces breaking changes to the `refetchInterval` or cache key APIs that conflict with the polling patterns in SPEC-021 or SPEC-022.
- React Router introduces breaking changes to the history-mode routing model or URL parameter APIs that conflict with the navigation patterns in SPEC-021 or SPEC-022, or requires changes to the `try_files $uri /index.html` Nginx fallback required by Decision 5.
- ADR-007 review trigger is satisfied (IBM Quantum Hardware integration begins): if the Experiment Dashboard requires real-time hardware execution monitoring, browser-side OTel instrumentation may become warranted. Revisit Decision 7.
- A non-CLI caller — automation service, CI pipeline, or another non-browser client — requires experiment submission capability. ADR-012 Decision 1 review trigger already identifies this case; if it results in a server-side orchestration model, the browser client's relationship to that model should be assessed.

---

# Employer Signaling

- Frontend Development
- Full-Stack Engineering
- Modern React Ecosystem (TanStack Query, TanStack Table, shadcn/ui, Radix UI, Tailwind CSS, Vite, TypeScript)
- Production-Style Deployment (multi-stage Docker build, Nginx static serving, CORS configuration)
- API Contract Consumption
- System Integration

---

# Decision Summary

**Decision 1:** SPEC-021 and SPEC-022 are feature areas of one SPA. No separate browser application is deployed for the Experiment Dashboard.

**Decision 2:** React with TypeScript and Vite. No server-side rendering.

**Decision 3:** shadcn/ui (Radix UI + Tailwind CSS) for UI components. TanStack Table for data-heavy tabular views (Trial Matrix, Execution Metadata, Solver Comparison).

**Decision 4:** TanStack Query for server state (fetching, caching, polling, background revalidation). React Router for client-side navigation and URL state. No Redux.

**Decision 5:** Dedicated `web-ui` Docker Compose container (Nginx, multi-stage build). Browser client and API are at distinct origins; CORS required on the API for the browser client origin. Default Docker Compose configuration: browser client at `http://localhost:3000`, API at `http://localhost:5000`. One Browser Client service is added to the Docker Compose deployment; all existing services are unchanged.

**Decision 6:** Build-time environment variable (`VITE_API_BASE_URL`, default `http://localhost:5000`) embedded in the Vite static build artifact. Rebuild required to change the API URL.

**Decision 7:** No OTel spans. No trace header injection. `request_id` from SPEC-008 FR-14 error responses surfaced in browser UI for operator correlation.

**Decision 8:** Micro-frontend extraction of SPEC-022 is a deferred option requiring a future ADR. MVP architecture is a single SPA without extraction provisions.

**Open Questions Resolved:** SPEC-021 OQ-2 (technology stack), SPEC-021 OQ-3 (serving mechanism), SPEC-021 OQ-4 (API base URL discovery), SPEC-022 OQ-1 (relationship to SPEC-021), SPEC-022 OQ-2 (technology stack), SPEC-022 OQ-4 (API base URL discovery).

**Primary Benefit:** Six open questions across two accepted specifications resolved in a single ADR with a coherent, minimal-complexity architecture that preserves all existing component boundaries (ADR-002 API separation, ADR-011 trace posture, ADR-012 CLI independence) and requires one new Docker Compose container.

**Primary Cost:** CORS configuration on the API for the browser client origin is a deployment prerequisite. Changing the API URL requires a container rebuild. Node.js build toolchain is added to the repository.

**Evidence Supporting the Decisions:** SPEC-022 FR-4 Trial Matrix structure (headless table requirement); SPEC-021 FR-6/FR-8.1 and SPEC-022 FR-3/FR-4 polling patterns (TanStack Query alignment); SPEC-022 Performance Considerations cache sharing requirement (shared TanStack Query key); ADR-002 API boundary (separate container required); ADR-011 CLI observability posture (no browser trace injection); SPEC-021 OQ-3 Option 1 and SPEC-022 OQ-1 Option 1 (selected options as described in the accepted specifications).

**Next Review Trigger:** Multi-user or internet-facing deployment requires CORS and origin strategy revision; React major version migration required; ADR-007 hardware integration review trigger satisfied and real-time hardware monitoring is required at the Dashboard.
