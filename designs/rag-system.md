# Retrieval-Augmented Generation (RAG) System Design

Summary

Author: Debasnan Singh

This document describes a typical RAG architecture: how data is ingested and indexed, where embeddings are stored, how retrieval + reranking works, and how the LLM is integrated and monitored. It also covers caching and cost-control patterns for production deployments.

Table of Contents

- [Intent](#intent)
- [High-level architecture](#high-level-architecture)
- [Components](#components)
- [Data ingestion and embedding pipeline](#data-ingestion-and-embedding-pipeline)
- [Retriever and Reranker](#retriever-and-reranker)
- [LLM serving, caching, and prompts](#llm-serving-caching-and-prompts)
- [Monitoring, scaling and cost controls](#monitoring-scaling-and-cost-controls)
- [Trade-offs](#trade-offs)
- [References](#references)

High-level architecture

1. Ingest: collect documents and metadata; clean and transform text.
2. Embed: compute vector embeddings and store them in an ANN-backed vector store.
3. Retrieve: run a similarity search to get candidate documents.
4. Rerank: (optional) run a reranker model to refine candidates.
5. LLM: send context + prompt to the LLM, apply response caching and safety filters.

Operational concerns

- Freshness vs cost: batch vs streaming ingestion.
- Embedding consistency and versioning when models change.
- Query latency: trade-offs between recall and speed in the vector store.
- Monitoring: latency, token usage, error rates, and index health.

Trade-offs

- Caching reduces cost but can serve stale answers. Reranking improves relevance at extra compute cost.

References

- Add references to vector databases, ANN algorithms (HNSW), and RAG papers/tools here.
