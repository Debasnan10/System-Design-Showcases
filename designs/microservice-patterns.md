# Microservice Patterns

Summary

Author: Debasnan Singh

This document collects common microservice architecture patterns and practical notes: CQRS and when to prefer it vs CRUD, event-driven design and messaging guarantees, idempotency and retry strategies, and saga-based long-running transactions.

Table of Contents

- [Intent](#intent)
- [Patterns](#patterns)
  - [CQRS vs CRUD](#cqrs-vs-crud)
  - [Event-driven design](#event-driven-design)
  - [Sagas and long-running transactions](#sagas-and-long-running-transactions)
  - [Idempotency and retries](#idempotency-and-retries)
- [Operational concerns](#operational-concerns)
- [Trade-offs](#trade-offs)
- [References](#references)

Intent

Provide clear guidance for choosing and implementing service patterns in a distributed environment. This is practical advice for engineers designing new services or refactoring legacy monoliths.

Patterns

CQRS vs CRUD

- CQRS (Command Query Responsibility Segregation) splits read and write models. Use it when reads and writes have very different performance, scale, or consistency requirements.
- CRUD is simpler and often preferable when consistency and simplicity matter more than scaling read/write independently.

Event-driven design

- Use an event backbone (Kafka, Pulsar) for decoupling services and enabling asynchronous workflows.
- Discusses message ordering, at-least-once vs exactly-once semantics, and consumer idempotency.

Idempotency and retries

- Design idempotent handlers for safe retry. Use unique request identifiers (dedup-store) or idempotency keys.
- Implement exponential backoff with jitter for retries.

Operational concerns

- Observability: request tracing across services, metrics for message lag, consumer group health.
- Backpressure and throttling to protect downstream systems.

Trade-offs

- CQRS adds complexity (multiple models, dual writes). Event-driven systems introduce eventual consistency and operational overhead.

References

- Add references and links to canonical articles and pattern catalogs here.
