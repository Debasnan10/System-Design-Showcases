````markdown
# Event-Driven Architecture — Enterprise Guide

**Prepared by:** Debasnan Singh

---

## Purpose

This document is an enterprise-grade, practical guide to designing, building, operating, and evolving production event-driven systems. It covers event modeling, brokers, message schemas, delivery semantics, durability patterns (outbox, CDC), event sourcing, choreography vs orchestration, observability, security, testing, and operational runbooks — plus code snippets, Kubernetes examples, and recommended repo layouts.

Target readers: architects, platform engineers, SREs, and engineers building distributed, event-driven platforms.

---

## Table of Contents

1. Executive summary
2. Core concepts & intent
3. High-level architecture (ASCII)
4. Event modeling & schema governance
5. Brokers, primitives & topology
6. Delivery semantics: at-most-once, at-least-once, exactly-once
7. Durability patterns: outbox, change-data-capture (CDC), and deduplication
8. Event sourcing & CQRS (when to use)
9. Choreography vs orchestration (SAGA patterns)
10. Ordering, partitioning & compaction
11. Observability: tracing, metrics, logging for events
12. Security, multi-tenancy & compliance
13. Testing, contract verification & chaos
14. Kubernetes & deployment examples
15. Example artifacts (producer/consumer snippets)
16. Operational runbook & SLOs
17. Trade-offs and alternatives
18. Next steps & recommended repo layout
19. References

---

## 1 — Executive summary

Event-driven architectures (EDA) decouple producers and consumers via asynchronous message passing. EDA improves scalability, resilience, and extensibility. Production EDA needs careful choices around message formats, broker topology, delivery guarantees, ordering, schema governance, and operational practices to avoid data loss, duplicate processing, and unbounded cost.

---

## 2 — Core concepts & intent

- Events: domain facts (immutable assertions) describing state changes.
- Commands: intent to perform an action (often directed to one service).
- Broker: component that routes and persists events (Kafka, Pulsar, RabbitMQ, SNS/SQS).
- Topic/Stream: logical channel grouping related events.
- Partition: unit of parallelism and ordering.
- Consumer group: horizontal scaling abstraction for consumers.

Intent: design for durability, clear schema governance, predictable ordering where needed, and operational simplicity for SREs.

---

## 3 — High-level architecture (ASCII)

```
[Producer/Service] --> [Broker/Topic] --> [Consumer Group A]
                                   |--> [Consumer Group B]
                                   |--> [Stream Processor -> Materialized View]

Ingest -> (CDC / Batch / API) -> Broker -> Consumers (workers, services, analytics)
```

---

## 4 — Event modeling & schema governance

Good event models are explicit, discoverable, and versioned.

Guidelines:

- Prefer domain events (order.created) over CRUD events (order.updated-by-api) when possible.
- Keep events small and focused; include only the data needed by subscribers.
- Use a schema registry (Avro/Protobuf/JSON Schema) and enforce compatibility rules (backward/forward).
- Include metadata: event_id, event_type, source, timestamp, schema_version, causation_id, correlation_id, producer_version.
- Avoid embedding large blobs; store references when needed.

Sample event envelope (JSON):

```json
{
  "event_id": "uuid-v4",
  "event_type": "order.created",
  "schema_version": "1.0.0",
  "produced_at": "2025-11-21T12:00:00Z",
  "producer": "orders-service:v1.2.3",
  "correlation_id": "trace-123",
  "data": {
    "orderId": "uuid",
    "customerId": "uuid",
    "items": [{ "sku": "ABC", "qty": 2 }],
    "total": 123.45
  }
}
```

Versioning rules:

- Non-breaking changes: add optional fields, extend enums carefully.
- Breaking changes: create a new event type or schema version and run a migration strategy.
- Deprecation: mark old fields and give consumers time to adapt (use schema registry compatibility checks).

---

## 5 — Brokers, primitives & topology

Common broker choices:

- Apache Kafka: high-throughput, durable, partitioned logs, strong ecosystem (Kafka Streams, ksqlDB).
- Apache Pulsar: similar to Kafka with built-in multi-tenancy and tiered storage.
- RabbitMQ: brokered queues and flexible routing; better for complex routing but less ideal for large-scale logs.
- Cloud: Amazon Kinesis, SNS+SQS, Google Pub/Sub — managed options with different semantics.

Topology considerations:

- Topic per domain entity or logical stream (orders, inventory, payments).
- Partition key selection: choose keys that balance throughput and preserve ordering (customerId vs orderId).
- Retention policy: keep data long enough for replays and analytics; use tiered storage for cost control.
- Compaction: enable log compaction in Kafka for state-topic use-cases (latest value by key).

Operational notes:

- Capacity planning: partitions × throughput × retention => storage needs.
- Broker scaling: re-partitioning is expensive; prefer careful partition planning.

---

## 6 — Delivery semantics: at-most-once, at-least-once, exactly-once

- At-most-once: message may be lost, never duplicated. Simple, low-latency.
- At-least-once: message delivered >=1 times (duplicates possible). Safer for durability but requires idempotent consumers.
- Exactly-once: strong guarantee; achieved with careful broker/transaction support (Kafka transactions) and idempotent processing.

Practical advice:

- Design consumers to be idempotent and tolerant to duplicates (dedupe by event_id, idempotency keys).
- Use transactional producers + outbox pattern to avoid lost events from partial failures.
- Where exactly-once is required, prefer broker + client features that support transactions, but weigh complexity.

---

## 7 — Durability patterns: outbox, CDC, and deduplication

Outbox pattern (recommended for producers that write to DB + publish events):

1. Persist business data and outbox record in same DB transaction.
2. An outbox relay reads unsent rows and publishes them to broker.
3. Mark outbox rows as sent on success.

CDC (Change Data Capture):

- Use Debezium or native cloud services to stream DB changes into events. Good for legacy DBs and fast bootstrapping.
- Keep a mapping layer to produce domain events from raw CDC streams.

Deduplication strategies:

- Consumer-side dedupe: store processed event_ids in a local or central store (TTL-controlled).
- Idempotent handlers: design operations that can be safely re-applied.

---

## 8 — Event sourcing & CQRS (when to use)

Event sourcing: persist every state change as an immutable event stream; materialize reads using projections.

When to use:

- Domain is complex, needs full audit/history, or requires time-travel queries.
- Requires reliable replayability and reconstruction of state.

Trade-offs:

- Pros: auditability, replay, temporal queries.
- Cons: complexity in storage, projection maintenance, and operational tooling.

CQRS: separate write model (events/commands) from read models (materialized views). Useful at scale for read-heavy workloads.

---

## 9 — Choreography vs orchestration (SAGA patterns)

Choreography (event-driven SAGA): services react to events and emit follow-up events. Decentralized, but harder to reason about for complex flows.

Orchestration: a central orchestrator (workflow engine or service) drives the saga by invoking services and triggering compensations when required.

Example: order fulfillment saga

- Choreography:

  - Orders service emits order.created
  - Inventory reserves and emits inventory.reserved
  - Billing charges and emits payment.completed
  - Orders subscribes to events and moves to completed

- Orchestration:
  - Orchestrator receives request, calls Inventory.reserve
  - Orchestrator calls Billing.charge
  - Orchestrator writes final state; orchestrator triggers compensations if a step fails

Choose orchestration for complex multi-step flows where centralized visibility and retries matter; choose choreography for simpler, loosely coupled flows.

---

## 10 — Ordering, partitioning & compaction

Ordering:

- Ordering is per-partition. If you need strict ordering for a business key, choose partition key carefully.
- Avoid global ordering requirements — they limit throughput.

Partitioning:

- Partition count determines parallelism and throughput. More partitions => more consumers can read in parallel.
- Repartition carefully; plan ahead since increasing partitions impacts ordering and requires rebalancing.

Compaction & retention:

- Use log compaction for state topics where you care about the latest value per key.
- Use retention for event-store topics for analytics and replay; consider tiered or cold storage for long retention.

---

## 11 — Observability: tracing, metrics, logging for events

Key signals:

- Broker metrics: lag per consumer group, throughput, broker CPU/RAM, disk usage.
- Consumer metrics: processing latency, error rates, success/failure counts, commit latency.
- End-to-end tracing: propagate correlation_id and trace context across producers and consumers.

Example metrics to export (Prometheus):

- consumer_lag{group,topic,partition}
- consumer_processing_duration_seconds
- outbox_queue_depth
- events_published_total{topic}

Tracing for async flows:

- Emit trace spans for produce and consume operations; link spans via correlation_id and parent trace when possible.
- Example span flow: Producer (send) -> Broker append -> Consumer (receive/process) -> downstream calls

Logging:

- Log structured events with event_id and trace ids for correlation.

---

## 12 — Security, multi-tenancy & compliance

Security controls:

- Authentication: mTLS for brokers or SASL for Kafka; cloud providers use IAM-based auth.
- Authorization: ACLs for topics, RBAC for multi-tenant clusters.
- Encryption: TLS in transit, encryption at rest.

Multi-tenancy patterns:

- Topic per tenant vs shared topics with tenant key. Balance operational isolation vs management overhead.

Compliance & PII:

- Mask or redact PII fields before publishing. Use tokenization or references to secure stores.
- Maintain audit logs for data access and transformations.

---

## 13 — Testing, contract verification & chaos

Testing levels:

- Schema tests: validate events against schema registry on CI.
- Contract tests: use provider/consumer verification to ensure producers and consumers agree on event formats.
- Integration: local broker (testcontainers) or staging infra for end-to-end tests.
- Chaos: inject failures (broker node loss, disk pressure, network partitions) to validate resilience.

CI checks:

- Reject incompatible schema changes via schema registry checks.
- Run lightweight integration tests with an embedded broker.

---

## 14 — Kubernetes & deployment examples

Small examples for producer service and a stream processor / consumer. Production deployments should separate producer, consumer, connector, and outbox components.

### Producer (Deployment snippet)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-producer
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: orders
          image: ghcr.io/debasnan10/orders-producer:1.0.0
          env:
            - name: BROKER_URL
              value: "kafka:9092"
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
```

### Consumer / Stream Processor (Deployment snippet)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-processor
spec:
  replicas: 4
  template:
    spec:
      containers:
        - name: inventory
          image: ghcr.io/debasnan10/inventory-processor:1.0.0
          env:
            - name: BROKER_URL
              value: "kafka:9092"
            - name: CONSUMER_GROUP
              value: "inventory-group"
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
```

---

## 15 — Example artifacts (producer & consumer snippets)

### Minimal producer (Python, kafka-python)

```python
from kafka import KafkaProducer
import json
import uuid
from datetime import datetime

producer = KafkaProducer(bootstrap_servers=['kafka:9092'], value_serializer=lambda v: json.dumps(v).encode('utf-8'))

def produce_order(order):
    event = {
        "event_id": str(uuid.uuid4()),
        "event_type": "order.created",
        "schema_version": "1.0.0",
        "produced_at": datetime.utcnow().isoformat() + 'Z',
        "producer": "orders-service:1.0.0",
        "data": order
    }
    producer.send('orders', key=order['orderId'].encode('utf-8'), value=event)
    producer.flush()

```

### Minimal consumer (Node.js, node-rdkafka)

```js
const Kafka = require("node-rdkafka");

const consumer = new Kafka.KafkaConsumer(
  {
    "group.id": "inventory-group",
    "metadata.broker.list": "kafka:9092",
  },
  {}
);

consumer.connect();

consumer
  .on("ready", () => {
    consumer.subscribe(["orders"]);
    consumer.consume();
  })
  .on("data", (message) => {
    const event = JSON.parse(message.value.toString());
    // idempotent handling by event.event_id
    console.log("received", event.event_type, event.event_id);
  });
```

---

## 16 — Operational runbook & SLOs

Suggested SLOs:

- Broker availability: 99.95%
- Consumer lag: keep consumer lag below threshold (e.g., < 10k messages) for critical streams
- Processing success rate: 99.9% (per-topic)

Runbook starters:

- Detect: alert on consumer_lag, broker_disk_usage, broker_under_replicated_partitions, outbox backlog
- Triage: check recent deployments, partition rebalances, broker logs
- Mitigate: scale consumers, increase partitions (carefully), reprocess from offsets, promote replicas, reduce retention to free disk temporarily

Recovery patterns:

- Reprocess: change consumer group id (or use a replayer) to reprocess from a topic offset or timestamp.
- Dead-letter queue: redirect poison messages to DLQs with example metadata for later inspection.

---

## 17 — Trade-offs and alternatives

- Kafka (log-centric) vs message-queue models: use Kafka for large-scale, append-only streams; use queues for point-to-point workloads.
- Exactly-once vs at-least-once: prefer idempotency + at-least-once in many domains; use transactions when strong guarantees are required.
- CDC vs application events: CDC is fast for lifting existing DBs into streams; app events express intent and are cleaner for long-term architecture.

---

## 18 — Next steps & recommended repo layout

Suggested repository layout (event-driven project):

```
event-driven/
├── apis/                # OpenAPI or AsyncAPI files
├── src/                 # producer and consumer code
├── connectors/          # outbox relays, CDC connectors
├── schemaregistry/      # schema artifacts (Avro/Proto/JSON Schema)
├── k8s/                 # k8s manifests (producers, consumers, connectors)
├── observability/       # OTEL, Prometheus rules, Grafana dashboards
├── tests/               # contract/integration tests
└── docs/                # design docs (this file)
```

---

## 19 — References

- Kafka: https://kafka.apache.org
- Debezium (CDC): https://debezium.io
- Avro & Schema Registry: https://avro.apache.org
- Event Sourcing patterns: Greg Young's materials

---

**Author:** Debasnan Singh

**Copyright (c) 2025 Debasnan Singh. Licensed under the MIT License.**
````
