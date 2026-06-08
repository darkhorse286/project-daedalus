# ADR-003: RabbitMQ as the Asynchronous Execution Boundary

## Status

Proposed

**Date:** 2026-06-07

**Related Feature(s):** Daedalus API, Daedalus Worker

**Related ADR(s):** ADR-001, ADR-002, ADR-004

---

# Context

Routing job execution is long-running and must not block the API request lifecycle. The system requires an asynchronous boundary between the control plane and the worker so that job submission returns immediately and execution proceeds independently.

The queue must provide durable message delivery so that routing jobs are not lost if the worker restarts. It must provide dead-letter handling for jobs that fail or expire. It must support consumer acknowledgment semantics so that an unacknowledged job is not dropped on worker crash.

The MVP runs entirely in Docker Compose and must not depend on managed cloud services.

---

# Decision

RabbitMQ is the message broker for asynchronous routing job execution.

Initial queue topology:

- routing-jobs: consumed by Daedalus Worker
- routing-jobs-dead-letter: receives failed or expired jobs

---

# Alternatives Considered

## Redis Streams

### Description

Redis-native stream data structure for durable message passing.

### Benefits

Low operational overhead. Simple Docker Compose integration. Redis may already be present for other purposes.

### Drawbacks

Weaker durability guarantees than RabbitMQ. Dead-letter semantics require additional configuration. Message routing is less expressive.

### Reason Not Selected

Routing job execution is expensive. A failed or dropped job represents significant wasted compute. Durability and dead-letter semantics are first-class requirements. RabbitMQ provides these with greater operational maturity than Redis Streams.

---

## Apache Kafka

### Description

Distributed event streaming platform designed for high-throughput, log-based message storage.

### Benefits

High throughput. Durable log-based storage with long retention. Strong ecosystem.

### Drawbacks

Heavyweight for a single-worker MVP. Designed for event streaming at scale, not work queue semantics. Significant operational complexity.

### Reason Not Selected

Operational complexity is disproportionate to the MVP requirements. Kafka is the wrong tool for a job queue pattern at this scale.

---

## NATS (with JetStream)

### Description

Lightweight open-source messaging system with JetStream for durable storage.

### Benefits

Simple. Fast. Low operational overhead.

### Drawbacks

JetStream (durable persistence) adds configuration complexity. Dead-letter semantics are less mature than RabbitMQ.

### Reason Not Selected

Durability and dead-letter semantics are weaker for the targeted job queue use case compared to RabbitMQ.

---

## AWS SQS / Azure Service Bus

### Description

Managed cloud message queue services.

### Benefits

Fully managed. Operationally simple.

### Drawbacks

Cloud dependency. Inconsistent with the Docker Compose local execution model. Requires external credentials and network access.

### Reason Not Selected

Cloud dependency is incompatible with the offline-capable local development and demonstration environment.

---

# Consequences

## Positive

Durable delivery ensures routing jobs survive worker restarts.

Dead-letter queue enables explicit failed job handling without data loss.

Consumer acknowledgment prevents job loss on worker crash.

RabbitMQ runs as a standard Docker Compose service requiring no cloud dependency.

## Negative

RabbitMQ is an additional infrastructure service to operate and monitor.

AMQP protocol semantics and RabbitMQ management add operational knowledge requirements.

## Accepted Risks

The MVP runs a single RabbitMQ node. High availability clustering is not configured. A node failure loses in-flight unacknowledged messages.

Dead-letter queue processing strategy is not defined in this ADR. It will be addressed during Worker implementation planning.

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
| Security       | Yes    |
| Deployment     | Yes    |

The API publishes job messages to RabbitMQ. The Worker consumes them. Queue depth and consumer lag are operational metrics that must be observable. RabbitMQ credentials must be managed in environment configuration.

---

# Supporting Evidence

- README.md: architectural intent for asynchronous job execution
- docs/architecture.md: Open Architecture Decisions section lists "Queue technology: RabbitMQ"
- docs/architecture.md: queue names routing-jobs and routing-jobs-dead-letter defined

No benchmark evidence exists at this stage. This is a pre-implementation decision.

---

# Assumptions

A single RabbitMQ node provides sufficient throughput and availability for the MVP workload.

The routing-jobs queue will not exceed a volume that requires partitioned consumer groups.

---

# Limitations

Single-node RabbitMQ provides no high availability. A node failure loses unacknowledged in-flight messages.

Dead-letter queue processing strategy is undefined. Reprocessing logic must be defined during Worker implementation planning.

---

# Documentation Updates

- Architecture documentation (addressed by this ADR)

---

# Review Triggers

Single-node availability becomes insufficient for the workload.

Dead-letter queue volume requires automated reprocessing behavior.

A different queue technology demonstrates a measurable advantage for the specific job queue pattern used here.

---

# Employer Signaling

- Distributed Systems
- Reliability Engineering

---

# Decision Summary

**Decision:** RabbitMQ for asynchronous routing job execution with dead-letter support.

**Primary Benefit:** Durable delivery with dead-letter semantics appropriate for expensive compute jobs that must not be silently dropped.

**Primary Cost:** Additional infrastructure service; single-node availability limitation in the MVP.

**Evidence Supporting the Decision:** Architecture document intent. No benchmark evidence at this stage.

**Next Review Trigger:** Single-node availability becomes insufficient, or dead-letter queue reprocessing requires automated handling.
