# ADR-006: Observability Stack Selection

## Status

Accepted

**Date:** 2026-06-07

**Related Feature(s):** All runtime components

**Related ADR(s):** ADR-001, ADR-002

---

# Context

Project DAEDALUS produces evidence as a first-class output. Every scheduler decision must be explainable, every solver execution must be traceable, and every result must be auditable. This is not incidental observability added after implementation. It is an architectural requirement.

The system spans two languages (C# and C++) and multiple container processes. Traces must correlate across the API and Worker. Metrics must be collectable without modifying application code after deployment. Logs must be structured and machine-readable.

The architecture document defines nine required spans:

- job.submit
- job.consume
- problem.load
- features.extract
- scheduler.score_solvers
- solver.execute
- result.evaluate
- report.generate
- job.complete

The MVP runs entirely in Docker Compose and must not depend on managed cloud services.

---

# Decision

The observability stack for Project DAEDALUS is:

- OpenTelemetry SDK: instrumentation in C# (API) and C++ (Worker)
- OpenTelemetry Collector: collection and export pipeline
- Prometheus: metrics storage
- Grafana: metrics visualization
- Structured JSON logs: written to stdout

This stack runs entirely within Docker Compose.

---

# Alternatives Considered

## Jaeger (Distributed Tracing Only)

### Description

Open-source distributed tracing platform, compatible with OpenTelemetry.

### Benefits

Simpler to configure for tracing only. Well-established. OpenTelemetry-compatible.

### Drawbacks

Tracing only. Does not address metrics. A separate metrics stack would be required alongside it.

### Reason Not Selected

A combined traces-and-metrics stack reduces operational complexity. The OpenTelemetry Collector with Prometheus covers both without adding Jaeger as a separate dependency. OpenTelemetry is the preferred instrumentation layer.

---

## ELK / EFK Stack (Elasticsearch, Logstash or Fluentd, Kibana)

### Description

Log aggregation, search, and visualization stack.

### Benefits

Powerful log querying and rich visualization.

### Drawbacks

Elasticsearch is resource-heavy. Docker Compose memory requirements for an ELK stack would be significant relative to the MVP workload. Disproportionate for MVP log volume.

### Reason Not Selected

Operational overhead is disproportionate to the expected log volume. Structured JSON logs to stdout are sufficient for the MVP.

---

## Datadog / New Relic / Honeycomb

### Description

Managed SaaS observability platforms.

### Benefits

Fully managed. Rich dashboards. No infrastructure to operate locally.

### Drawbacks

SaaS dependency. Cost. Requires external network access. Inconsistent with the offline-capable local execution model.

### Reason Not Selected

Cloud SaaS dependency is incompatible with the local Docker Compose execution model. The MVP must run without external service dependencies.

---

## Zipkin

### Description

Distributed tracing system.

### Benefits

Simple and lightweight.

### Drawbacks

Less ecosystem momentum than OpenTelemetry. Fewer integration options. Does not address metrics.

### Reason Not Selected

OpenTelemetry is the emerging industry standard for instrumentation. Zipkin adds no advantage over the OpenTelemetry Collector in this context.

---

# Consequences

## Positive

OpenTelemetry SDK is vendor-neutral. Switching the backend (Jaeger, Tempo, an OTLP-compatible managed service) does not require re-instrumentation.

Prometheus and Grafana are well-understood, widely deployed, and Docker Compose-compatible.

The full stack runs locally without cloud dependencies.

Required spans defined in the architecture document can be implemented directly with the OpenTelemetry SDK.

## Negative

The OpenTelemetry C++ SDK is less mature than the .NET SDK. C++ instrumentation may require more implementation effort and workarounds for specific attribute types.

Four observability containers (Collector, Prometheus, Grafana, and log infrastructure) increase Docker Compose complexity and resource requirements.

Grafana dashboard definitions require ongoing maintenance and are not defined for the MVP.

## Accepted Risks

The OpenTelemetry C++ SDK maturity may require workarounds for specific span attributes or propagation formats. This must be evaluated during Worker implementation planning.

Prometheus metric cardinality must be managed. Unbounded label cardinality causes memory growth.

---

# Architectural Impact

| Component      | Impact |
| -------------- | ------ |
| Domain Layer   | No     |
| API Layer      | Yes    |
| Persistence    | No     |
| Infrastructure | Yes    |
| Configuration  | Yes    |
| Observability  | Yes    |
| Security       | No     |
| Deployment     | Yes    |

API requires OpenTelemetry .NET SDK. Worker requires OpenTelemetry C++ SDK. Infrastructure requires Collector, Prometheus, and Grafana containers. Collector pipeline configuration is required.

---

# Supporting Evidence

- README.md: OpenTelemetry traces listed as an MVP requirement
- docs/architecture.md: Observability section enumerates required spans, components, and log format

No benchmark evidence exists at this stage. This is a pre-implementation decision.

---

# Assumptions

The OpenTelemetry C++ SDK is sufficiently mature to instrument the required spans and attributes.

Prometheus and Grafana Docker images are available and suitable for the development environment resource profile.

---

# Limitations

No log aggregation beyond structured JSON to stdout is defined for the MVP. Log search requires reading container logs directly.

Grafana dashboard definitions are not defined. They will emerge during implementation.

Alert configuration is out of scope for the MVP.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)
- **ADR-011 (Trace Context Propagation, Accepted):** ADR-011 resolves the cross-process W3C TraceContext propagation mechanism for the API → RabbitMQ → Worker boundary, extending the observability architecture established by this ADR. ADR-006 remains the authoritative observability stack selection decision; ADR-011 addresses the accepted C++ SDK propagation risk identified in this ADR as requiring assessment during Worker implementation planning. ADR-011 does not alter the stack selection (OpenTelemetry SDK, Collector, Prometheus, Grafana), the required span hierarchy defined in architecture.md, or the in-process span context propagation model.

---

# Review Triggers

The OpenTelemetry C++ SDK lacks support for a required instrumentation capability.

Prometheus cardinality becomes a resource concern under the actual label structure used.

A superior observability stack demonstrates a meaningful advantage for the specific requirements of this system.

---

# Employer Signaling

- Observability
- Reliability Engineering

---

# Decision Summary

**Decision:** OpenTelemetry SDK and Collector, Prometheus, Grafana, structured JSON logs.

**Primary Benefit:** Vendor-neutral instrumentation with a complete traces-and-metrics stack running locally without cloud dependencies.

**Primary Cost:** Multiple additional Docker Compose services; OpenTelemetry C++ SDK maturity is an open risk.

**Evidence Supporting the Decision:** Architecture document observability requirements. No benchmark evidence at this stage.

**Next Review Trigger:** OpenTelemetry C++ SDK lacks a required instrumentation capability, or Prometheus cardinality becomes a resource concern.
