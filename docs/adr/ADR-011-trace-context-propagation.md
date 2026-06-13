# ADR-011: Trace Context Propagation

## Status

Proposed

**Date:** 2026-06-12

**Related Feature(s):** SPEC-005 (Worker Execution Lifecycle), SPEC-006 (Evidence Log), SPEC-007 (Core Quality Evaluation), SPEC-008 (API / Control Plane)

**Related ADR(s):** ADR-001, ADR-002, ADR-003, ADR-006

---

## Context

Project DAEDALUS separates job submission from job execution across two isolated processes:

- The **Daedalus API** (C# ASP.NET Core, ADR-002) accepts routing job submissions and publishes messages to the `routing-jobs` queue.
- The **Daedalus Worker** (C++, ADR-001) consumes those messages and drives the full optimization lifecycle.

These two processes communicate exclusively through RabbitMQ (ADR-003). The API returns HTTP 202 immediately; the Worker processes the job asynchronously. This asynchronous execution boundary severs the in-process trace context that OpenTelemetry propagates automatically in synchronous call chains.

The architecture (architecture.md) mandates nine required OpenTelemetry spans. Of these, two span the asynchronous boundary:

- `job.submit` — emitted by the API; covers request receipt through HTTP 202 (SPEC-008 FR-17)
- `job.consume` — emitted by the Worker; covers message consumption through ACK/NACK (SPEC-005 FR-19)

`job.consume` is the root span of the entire Worker trace. All subsequent Worker spans (`problem.load`, `solver.execute`, `result.evaluate`, `worker.evidence.persist`, `report.generate`, `job.complete`) are children of `job.consume`. Core-emitted spans (`features.extract`, `scheduler.score_solvers`) propagate the in-process context received from the Worker and appear as descendants of `job.consume`. End-to-end trace continuity therefore requires only that `job.consume` be navigably linked to `job.submit`.

ADR-006 selects the OpenTelemetry SDK and Collector as the observability stack for both components. It explicitly acknowledges that the OpenTelemetry C++ SDK is less mature than the .NET SDK, noting that "instrumentation may require more implementation effort and workarounds for specific attribute types," and designates this as an accepted risk requiring assessment during Worker implementation planning.

SPEC-008 FR-17 requires trace continuity but deferred the propagation mechanism to a future architectural decision (SPEC-008 OQ-6). SPEC-005 FR-19 similarly states: "The specific context propagation mechanism is an implementation planning concern." The SPEC-008 Architecture Review (AR-1) elevated OQ-6 from implementation planning to architectural decision, finding that any propagation mechanism touches either the shared AMQP message schema (ADR-003, SPEC-008 FR-5, SPEC-005 FR-3) or the shared job record schema (SPEC-006, ODR-6) — both of which require an ADR to authorize changes. This ADR is the resolution of AR-1 and OQ-6.

Core-emitted span context propagation within the Worker process is in-process. Because Core is a C++ library invoked directly within the Worker process, the OpenTelemetry C++ SDK propagates context automatically through in-process facilities. No architectural decision is required for intra-process propagation; this ADR addresses only the cross-process boundary between API and Worker.

---

## Problem Statement

Given that `job.submit` completes in the API process and `job.consume` starts in the Worker process following asynchronous RabbitMQ message delivery:

**How is the trace context from `job.submit` made available to the Worker so that `job.consume` can be navigably linked to it?**

Without an explicit propagation mechanism, `job.submit` and `job.consume` appear as disconnected traces in any OpenTelemetry-compatible backend. The architectural requirement for end-to-end trace continuity — stated in architecture.md, ADR-006, SPEC-008 FR-17, and SPEC-005 FR-19 — cannot be satisfied without transmitting trace context across the RabbitMQ boundary.

---

## Decision Drivers

1. **End-to-end trace continuity is a first-class architectural requirement.** architecture.md mandates both `job.submit` and `job.consume` as required spans. ADR-006 and both specifications require them to form a distributed trace. This is not an operational nicety — it is the mechanism that makes submission-to-completion job observability possible.

2. **Minimal schema impact is preferred.** Changes to the job message payload schema (SPEC-008 FR-5, SPEC-005 FR-3) require revisions to accepted/proposed specifications. Changes to the job record schema (SPEC-006 FR-4.2) require the ODR-6 process and Project Owner approval. A mechanism that avoids both is strongly preferred.

3. **Cross-language compatibility is required.** The mechanism must be implementable in both the .NET OpenTelemetry SDK (API) and the C++ OpenTelemetry SDK (Worker). ADR-006 acknowledges C++ SDK immaturity; the mechanism must not depend on a capability unavailable in C++.

4. **At-least-once delivery must not invalidate the trace link.** On message re-delivery (SPEC-005 FR-14), the trace context carried with the message must remain valid, and the re-delivered execution must remain linkable to the original `job.submit`.

5. **Future deployment compatibility.** The mechanism must remain correct if the API and Worker are eventually deployed across separate hosts or networks, not just within a single Docker Compose environment.

6. **Standards alignment.** Preference for mechanisms aligned with OpenTelemetry semantic conventions and W3C standards, which reduces implementation novelty and improves interoperability with existing observability tooling.

---

## Considered Alternatives

### Alternative 1: W3C TraceContext in AMQP Message Metadata

**Description:**
The API injects W3C TraceContext (`traceparent`, optionally `tracestate`) into the AMQP message metadata at publication time. The Worker extracts this context from the consumed message and uses it to establish a navigable trace relationship for `job.consume`.

W3C TraceContext (W3C Recommendation) defines a standard format for encoding distributed trace context in propagation headers. The AMQP 0-9-1 `application_headers` table — a key-value store in AMQP message properties, distinct from the message payload body — serves as the metadata carrier. The API injects context at publication; the Worker extracts it at consumption using OpenTelemetry-compatible propagation facilities. The concrete SDK adapter, carrier implementation, and library integration choices are implementation planning concerns.

**Schema impact:** The message payload fields (`job_id`, `problem_id`, `scheduler_config_id`) defined in SPEC-008 FR-5 and SPEC-005 FR-3 are unchanged. The job record schema (SPEC-006 FR-4.2) is unchanged. AMQP message metadata is outside the scope of the payload schemas defined in either specification.

**Behavior under re-delivery:** On message re-delivery, the same `traceparent` value redelivered with the message is available to the Worker. The Worker creates a new `job.consume` span with a trace relationship to the same original `job.submit` context. Both the original execution attempt and any re-delivered attempt are linked to the same originating submission trace.

**Benefits:**
- Zero impact on message payload schema (SPEC-008 FR-5, SPEC-005 FR-3) and job record schema (SPEC-006 FR-4.2, ODR-6)
- Aligns with the OpenTelemetry semantic conventions for messaging systems, which specify message metadata — not payload — as the propagation carrier
- W3C TraceContext format is vendor-neutral; compatible with Jaeger, Zipkin, Grafana Tempo, and any OTLP-compatible backend without re-instrumentation
- Compatible with future distributed deployment; AMQP message metadata travels with the message regardless of network topology
- Standard pattern in .NET, Java, Go, and Python OTel ecosystems; implementation precedent is well-established

**Drawbacks:**
- The C++ Worker must extract W3C TraceContext from AMQP message metadata using OpenTelemetry-compatible propagation facilities; no built-in AMQP integration exists in the C++ SDK at this writing, and the concrete integration approach is an implementation planning concern
- The AMQP client library chosen for the C++ Worker must expose the `application_headers` table at message consumption time; this must be confirmed during implementation planning

---

### Alternative 2: Trace Context in AMQP Message Payload Body

**Description:**
The API includes `traceparent` (and optionally `tracestate`) as fields in the JSON message body alongside `job_id`, `problem_id`, and `scheduler_config_id`. The Worker parses these fields from the JSON payload.

**Benefits:**
- No dependency on AMQP metadata API
- Simpler extraction in C++: only JSON parsing required

**Drawbacks:**
- Modifies the message payload schema defined in SPEC-008 FR-5 and SPEC-005 FR-3, requiring revisions to both specifications
- Couples the business message schema to the observability stack; `traceparent` is not business data
- Diverges from OpenTelemetry semantic conventions, which specify message metadata, not payload, as the propagation carrier

**Reason not selected:** Schema impact on accepted/proposed payload definitions and semantic coupling of business payload to observability concerns.

---

### Alternative 3: Trace Context Persisted in PostgreSQL Job Record

**Description:**
The API writes the active `job.submit` trace context to the job record in PostgreSQL at job creation time. The Worker reads the job record — already required for idempotency checks (SPEC-005 FR-14) — and extracts the trace context to establish a trace relationship for `job.consume`.

**Benefits:**
- No dependency on AMQP metadata API
- Trace context is durable in PostgreSQL; survives broker restarts
- Enables retrospective trace correlation from the Evidence Log

**Drawbacks:**
- Requires a new field in the job record schema (SPEC-006 FR-4.2), triggering the ODR-6 process: a SPEC-006 revision and Project Owner approval are required
- Couples the Evidence Log schema to the observability stack; trace context is instrumentation data, not evidence data
- Creates a temporal ordering problem: the Worker extracts context from the database before starting `job.consume`, inverting the natural ordering of span creation and database access

**Reason not selected:** Triggers ODR-6 (SPEC-006 schema change), couples the Evidence Log to the observability stack, and creates an unnatural dependency between span creation and database reads. This alternative is designated as the escalation fallback (see Accepted Risks) if AMQP metadata propagation proves infeasible.

---

### Alternative 4: No Propagation — `job_id` as Query-Time Correlation

**Description:**
No trace context is propagated. Both `job.submit` and `job.consume` include `job_id` as a required span attribute (SPEC-008 FR-17, SPEC-005 FR-19). Operators correlate submission and execution by querying the observability backend for all spans sharing a given `job_id`.

**Benefits:**
- No implementation risk
- Already partially implemented: `job_id` is a required attribute on both spans

**Drawbacks:**
- Does not produce a distributed trace; `job.submit` and `job.consume` remain disconnected in the trace topology
- Does not satisfy the architectural requirement for end-to-end distributed trace continuity stated in architecture.md and SPEC-008 FR-17

**Reason not selected:** Does not satisfy the first-class architectural requirement for distributed trace continuity.

---

## Decision

**W3C TraceContext propagation in AMQP message metadata (Alternative 1).**

The API injects W3C TraceContext (`traceparent`, optionally `tracestate`) into the AMQP message `application_headers` at publication to the `routing-jobs` queue. The Worker extracts this context from the consumed message and uses it to establish a navigable trace relationship between `job.consume` and the `job.submit` trace context.

**Trace relationship model:** For asynchronous producer-consumer patterns, the OpenTelemetry specification for messaging semantic conventions recommends that the consumer span establish a linked relationship to the producer context rather than a strict parent-child relationship. This is because the producer's span (`job.submit`) completes before the consumer's span (`job.consume`) begins — a temporal ordering that does not match the parent-child semantics of synchronous sub-operations. The Worker must establish an OpenTelemetry-compatible trace relationship — whether a span link, a child relationship under the extracted context, or another supported model — such that `job.submit` and `job.consume` are navigably connected in the observability backend. The specific relationship model is an implementation planning concern subject to the capabilities of the OpenTelemetry C++ SDK and the chosen observability backend.

`job.consume` remains the root of the Worker trace hierarchy. All subsequent Worker spans remain children of `job.consume` as specified in SPEC-005 FR-19. The effect is that the submission trace (`job.submit`) and the execution trace (rooted at `job.consume`) are navigably connected.

**AMQP metadata:** The AMQP 0-9-1 `application_headers` table carries the W3C TraceContext values. The API injects them at publication. The Worker extracts them at consumption using OpenTelemetry-compatible propagation facilities. The concrete SDK integration — including any adapter or carrier implementation required by the C++ SDK — is an implementation planning concern.

**No schema changes:** The message payload fields (`job_id`, `problem_id`, `scheduler_config_id`) defined in SPEC-008 FR-5 and SPEC-005 FR-3 are unchanged. The job record schema (SPEC-006 FR-4.2) is unchanged. AMQP `application_headers` are message metadata, not payload, and are outside the scope of the payload schemas defined in either specification.

**Re-delivery behavior:** On message re-delivery (SPEC-005 FR-14), the `traceparent` value from the original publication travels with the redelivered message. The Worker establishes a trace relationship to the same original `job.submit` context. Both the original execution attempt and any re-delivered attempt are linked to the same originating submission trace.

---

## Consequences

### Positive

- End-to-end distributed trace continuity is achieved: a trace viewer can navigate from `job.submit` to `job.consume` and all descendent Worker spans, providing full submission-to-completion observability for every job.
- No schema changes to the message payload (SPEC-008 FR-5, SPEC-005 FR-3) or the job record (SPEC-006 FR-4.2); the ODR-6 process is not required.
- W3C TraceContext format is vendor-neutral; the decision does not create dependency on a specific trace backend beyond the OTel Collector already selected by ADR-006. Future backend migration requires no re-instrumentation.
- Compatible with future distributed deployment; AMQP metadata travels with the message regardless of network topology or infrastructure changes beyond the Docker Compose MVP.
- Aligned with OpenTelemetry semantic conventions for messaging; reduces implementation novelty and improves interoperability with the established ecosystem of observability tooling.

### Negative

- The C++ Worker must extract W3C TraceContext from AMQP message metadata and establish a trace relationship using OpenTelemetry-compatible propagation facilities. The C++ SDK does not provide a built-in AMQP integration at this writing; the concrete integration approach is an implementation planning concern that must be resolved before Worker implementation begins.
- The AMQP client library for the C++ Worker must expose the `application_headers` table at message consumption time. This must be confirmed during Worker implementation planning; if the chosen library does not expose metadata in a usable form, a different library must be selected.

### Accepted Risks

- **C++ SDK propagation feasibility:** If the OpenTelemetry C++ SDK cannot support W3C TraceContext extraction from AMQP message metadata through any available integration approach, this decision must be escalated back to the ADR process. Alternative 3 (trace context in the PostgreSQL job record) is the designated fallback, subject to the ODR-6 process for the required SPEC-006 schema change.
- **Trace relationship rendering:** The navigability of trace relationships varies across OTel-compatible backends depending on which relationship model is used. If the chosen relationship model is not visually rendered by the observability backend, query-time correlation via the `job_id` attribute remains available. This is operationally acceptable: `job_id` is a required attribute on both spans (SPEC-008 FR-17, SPEC-005 FR-19). The trace relationship is present in the trace data regardless of backend rendering behavior.

---

## Architectural Impact

| Component | Impact |
|---|---|
| API Layer (C# ASP.NET Core, ADR-002) | Yes — must inject W3C TraceContext into AMQP message `application_headers` at publication (SPEC-008 FR-5) |
| Worker (C++, ADR-001) | Yes — must extract W3C TraceContext from AMQP message `application_headers` at consumption and establish a navigable trace relationship for `job.consume` (SPEC-005 FR-3, FR-19); concrete integration approach is an implementation planning concern |
| RabbitMQ (ADR-003) | Yes — AMQP `application_headers` table is now used for trace context propagation; this is an additive metadata use of existing AMQP 0-9-1 message structure, not a queue topology change |
| Observability (ADR-006) | Yes — both SDK configurations must enable W3C TraceContext propagation; the chosen trace relationship model must be supported by the SDK and observable in the selected backend |
| PostgreSQL / Evidence Log (SPEC-006, ODR-6) | No — job record schema is unchanged |
| Message Payload Schema (SPEC-008 FR-5, SPEC-005 FR-3) | No — payload fields unchanged; AMQP metadata is outside the payload schema boundary |
| Core / Scheduler | No — Core spans propagate context received from the Worker via in-process facilities; this ADR does not change intra-process propagation behavior |
| Report Generator | No |

---

## Impacted Artifacts

The following specifications require follow-on revisions to incorporate this decision. All changes are additive; no existing functional requirement is modified and no existing schema definition changes.

**SPEC-008 FR-5 (Queue Publication):**
Add: At publication time, the API injects W3C TraceContext (`traceparent`, optionally `tracestate`) from the active `job.submit` span context into the AMQP message `application_headers`. The message payload fields (`job_id`, `problem_id`, `scheduler_config_id`) are unchanged.

**SPEC-008 FR-17 (Observability — Trace Context Propagation):**
Replace the OQ-6 deferral paragraph with: Trace context is propagated from the API's `job.submit` span to the Worker's `job.consume` span via W3C TraceContext carried in AMQP message `application_headers` (ADR-011). The Worker establishes an OpenTelemetry-compatible trace relationship between `job.consume` and the extracted `job.submit` context. The requirement for cross-component trace continuity is not deferred; ADR-011 is the authoritative decision.

Add to FR-17 Acceptance Criteria: The `job.submit` span context is injected as W3C TraceContext in AMQP message `application_headers` on every successful publication (FR-5).

**SPEC-008 OQ-6:**
Mark Resolved. Resolution: W3C TraceContext in AMQP message `application_headers`; Worker establishes a navigable trace relationship between `job.consume` and the extracted context. Reference ADR-011.

**SPEC-008 Acceptance Checklist:**
Check OQ-6 as resolved.

**SPEC-005 FR-3 (Job Consumption — RabbitMQ):**
Add: At message consumption, the Worker extracts W3C TraceContext from the AMQP message `application_headers` using OpenTelemetry-compatible propagation facilities and uses the extracted context to establish a navigable trace relationship for `job.consume` (ADR-011). If no trace context is present in the message headers (e.g., a message published without instrumentation), `job.consume` is created without a trace relationship. The concrete SDK integration approach is an implementation planning concern.

**SPEC-005 FR-19 (Telemetry — Span Context Propagation):**
Replace: "The specific context propagation mechanism is an implementation planning concern."
With: Trace context is propagated from the API's `job.submit` span via W3C TraceContext carried in AMQP message `application_headers` (ADR-011). The Worker extracts this context at message consumption and establishes a navigable trace relationship between `job.consume` and the `job.submit` context; the concrete integration approach is an implementation planning concern. Core-emitted spans continue to propagate the trace context received from the Worker using in-process propagation facilities.

---

## Required Follow-On Changes

All changes are additive to existing specifications. None are behavioral revisions to accepted functional requirements. None require schema changes to accepted data models.

| Item | Target | Change |
|---|---|---|
| 1 | SPEC-008 FR-5 | Add AMQP metadata injection requirement at publication |
| 2 | SPEC-008 FR-17 | Replace OQ-6 deferral text with ADR-011 mechanism; add acceptance criterion |
| 3 | SPEC-008 OQ-6 | Mark Resolved; reference ADR-011 |
| 4 | SPEC-008 Acceptance Checklist | Check OQ-6 resolved |
| 5 | SPEC-005 FR-3 | Add context extraction and trace relationship requirement; add graceful fallback for absent context |
| 6 | SPEC-005 FR-19 | Replace "implementation planning concern" with ADR-011 mechanism; preserve implementation flexibility for concrete integration |

Completing items 1–4 unblocks SPEC-008 from advancing from Draft to Proposed status (AR-1 was the blocking Architecture Review finding). SPEC-008's Acceptance Review remains blocked on AR-5 (ADR-002, ADR-003, ADR-004, ADR-006, ADR-009 promotion to Accepted). Items 5–6 are required before SPEC-005 can advance to Verified.

---

## Open Questions

### OQ-1: C++ OpenTelemetry SDK Integration Approach for AMQP Metadata

**Question:** Can the C++ Worker extract W3C TraceContext from AMQP message `application_headers` and establish a navigable trace relationship using available OpenTelemetry C++ SDK capabilities? If so, which integration approach?

**Why it matters:** The C++ SDK does not provide a built-in AMQP integration at this writing (ADR-006 accepted risk). The concrete integration approach — whether via a custom carrier, direct context construction from parsed header values, or another SDK mechanism — must be assessed during Worker implementation planning. The Worker's AMQP client library must also expose `application_headers` at message consumption time in a form compatible with the chosen integration approach.

**Owner:** Worker implementation planning.

**Blocking:** Blocking for SPEC-005 FR-3 and FR-19 implementation. Not blocking for ADR-011 acceptance, SPEC-008 Proposed status advancement, or SPEC-008 Acceptance Review.

**Escalation:** If no viable integration approach exists within the C++ SDK and AMQP client library combination, this decision must be escalated back to the ADR process. Alternative 3 (trace context in the PostgreSQL job record, requiring SPEC-006 revision under ODR-6) is the designated fallback.
