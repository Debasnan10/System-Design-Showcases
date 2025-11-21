# Designs — Index

This folder contains curated system-design write-ups and diagrams. Read these in the suggested order to build a mental model from patterns to advanced systems.

Contents

- `microservice-patterns.md` — Core microservice architecture patterns: CQRS vs CRUD, Event Sourcing, Sagas, idempotency patterns, messaging and failure handling.
- `rag-system.md` — Retrieval-Augmented Generation (RAG) system architecture: ingestion, embeddings, retriever+ranker, LLM serving, caching, and monitoring concerns.

How to read a design doc

1. Summary: quick one-paragraph intent and when to use this pattern.
2. Components: key parts and responsibilities.
3. Data flow / sequence diagrams.
4. Trade-offs: alternatives and when not to use.
5. Operational concerns: scaling, monitoring, cost, and security.

Add new designs by following the `CONTRIBUTING.md` guidelines. Put diagrams in `diagrams/` or `designs/assets/` and reference them relatively.
