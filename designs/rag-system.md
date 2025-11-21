# Retrieval-Augmented Generation (RAG) — Enterprise Guide

Prepared by: Debasnan Singh

---

## Purpose

This document is a practical, production-focused guide for building Retrieval-Augmented Generation (RAG) systems: ingestion pipelines, embedding/versioning strategies, vector stores and ANN choices, retriever + reranker designs, LLM integration and prompt engineering, caching, monitoring, and operational playbooks.

Target readers: architects, ML engineers, platform engineers, and SREs building RAG-powered features.

---

## Table of Contents

1. Executive summary
2. Core concepts & intent
3. High-level architecture (diagrams)
4. Components and responsibilities
5. Data ingestion & embedding pipeline
6. Vector stores & ANN choices
7. Retriever design & reranking
8. Prompting, LLM serving & safety
9. Caching, freshness & consistency
10. Operational concerns: monitoring, costs, scaling
11. Security & privacy considerations
12. Kubernetes & deployment examples
13. Example artifacts (code snippets)
14. Trade-offs and alternatives
15. Runbook checklist & SLOs
16. Next steps and recommended repo layout

---

## 1 — Executive summary

RAG augments generative models with relevant external context retrieved from a vector-indexed corpus. This improves factuality and domain coverage while reducing hallucinations when prompts include the retrieved context. Building production RAG requires attention to data quality, embedding/version management, retrieval latency/recall trade-offs, caching, cost control, and observability.

---

## 2 — Core concepts & intent

- Document store: source text (docs, KB, logs). Clean, chunk, and add metadata.
- Embeddings: numerical vectors representing semantic meaning (model + dimension).
- Vector index (ANN): efficient nearest-neighbor store (HNSW, IVFFlat, etc.).
- Retriever: finds k nearest vectors for a query.
- Reranker: (optional) ranks candidates using a cross-encoder or lightweight model.
- Generator (LLM): consumes retrieved context + prompt to produce a response.

Intent: provide a resilient, auditable, and cost-effective RAG pipeline suitable for production.

---

## 3 — High-level architecture (diagrams)

### ASCII (simplified)

```
User -> Retriever -> (Vector DB + Metadata) -> Reranker -> LLM -> Response
		 ^                                   |
		 |---- Ingest -> Chunk -> Embed ------|
```

---

## 4 — Components and responsibilities

- Ingest pipeline: connectors, text extraction, normalization, chunking, metadata capture.
- Embedder: batch/online embedding service, model versioning, batching, rate limits.
- Vector store: ANN index, sharding, persistence, snapshotting, backups.
- Retriever: candidate search (approx/neural search), filters, semantic + lexical hybrids.
- Reranker: cross-encoder model or learned-to-rank, optionally run on GPU/CPU.
- LLM service: model selection, prompt templates, safety filters, token budgeting.
- Cache layer: query result caching, response caching, TTL strategy.
- Orchestration: workflows for re-indexing, index rebuilds, schema migration.
- Observability: request traces, query latency, index health, embedding failures, token usage.

---

## 5 — Data ingestion & embedding pipeline

Key steps:

1. Connectors: pull from sources (S3, DBs, Confluence, Git, web crawl). Track source + version.
2. Extraction & cleaning: remove HTML noise, normalize unicode, dedupe.
3. Chunking: split into chunks ~200–1000 tokens depending on use-case; preserve natural boundaries and metadata.
4. Embedding: call embedding model (batch/stream). Tag embeddings with model name + version + timestamp.
5. Store: write vector + metadata to vector DB and persist raw chunk to metadata store.
6. Out-of-band validation: sample embeddings for drift, run similarity sanity checks.

Example chunking pseudo-code (Python):

```python
def chunk_text(text, max_tokens=512):
		# naive splitter on paragraphs/sentences; real systems use tokenizer-aware splitting
		paragraphs = text.split('\n\n')
		chunks = []
		cur = ''
		for p in paragraphs:
				if len(cur) + len(p) > max_tokens:
						chunks.append(cur)
						cur = p
				else:
						cur += '\n\n' + p
		if cur:
				chunks.append(cur)
		return chunks
```

Embedding call pattern (batch):

```python
embeddings = embed_model.encode(batch_of_texts)  # returns NxD float arrays
```

Versioning note: always store embedding_model_name and embedding_model_version with each vector. When upgrading models, re-embed or store multi-version indices.

---

## 6 — Vector stores & ANN choices

Common options:

- Self-hosted: Milvus, Weaviate, Vespa, FAISS (on disk), Annoy.
- Managed: Pinecone, Zilliz Cloud, Qdrant Cloud, AWS Kendra (vector features), Azure Cognitive Search.

ANN algorithms & properties:

- HNSW: high recall, fast, good for dynamic indexes; memory heavy.
- IVF(Flat) + PQ: efficient for very large corpora; requires offline indexing.
- Faiss: library with multiple index types; common for research and self-hosted infra.

Selection factors:

- Latency requirements (sub-100ms vs sub-second).
- Index size and update frequency (dynamic vs batch rebuild).
- Hardware (CPU vs GPU support).
- Consistency & durability guarantees.

Index maintenance:

- Snapshotting and backups.
- Periodic compaction and tuning (HNSW ef/ef_construction).
- Metrics: index size, average distance, recall on gold queries.

---

## 7 — Retriever design & reranking

Retriever patterns:

- Dense-only: embedding-based vector similarity.
- Hybrid: vector + lexical (BM25) reranking or combined scores.
- Two-stage: candidate retrieval (ANN) → reranking (cross-encoder).

Reranker trade-offs:

- Cross-encoders (BERT-style) give better ranking but are more expensive (CPU/GPU).
- Lightweight rerankers (Dot-product with small model) are cheaper but less accurate.

Example retrieval flow (pseudo):

```text
1. Query embedding q = embed(query)
2. ANN search top-K candidates
3. Optionally apply metadata filters (source, date range)
4. Run reranker on candidates -> top-N
5. Merge with business logic and create context for LLM
```

Precision/Recall tuning: vary K (ANN fetch) and reranker budget. Log recall vs latency for monitoring.

---

## 8 — Prompting, LLM serving & safety

Prompt templates: include source snippets with citations (id + score + text). Limit total tokens.

Example prompt (shortened):

```text
You are a helpful assistant. Use the following documents (show id and short excerpt) to answer. Do not hallucinate.

Documents:
[1] (score: 0.92) "..."
[2] (score: 0.88) "..."

Question: {user_question}

Answer strictly based on the provided documents. If insufficient, say "I don't know."
```

LLM serving patterns:

- Hosted API (OpenAI, Anthropic, Llama 2 cloud) — easiest, pay-per-token.
- Self-hosted (Llama.cpp, vLLM, Ray Serve) — control and cost savings at scale.

Token budgeting: compute cost per call = tokenizer(tokens_in_prompt + tokens_expected) \* cost_per_token. Cache repeated queries.

Safety & hallucination mitigation:

- Response filters: safety classifier, profanity filter.
- Source tracing: attach source IDs and scores to answers.
- Refusal/warning policy for unsupported requests.

---

## 9 — Caching, freshness & consistency

Cache strategies:

- Response cache: cache final LLM outputs for identical query+context signatures (TTL short for dynamic data).
- Retriever cache: cache top-K candidate IDs for hot queries.
- Embedding cache: cache embeddings for frequently updated docs or queries.

Freshness patterns:

- Batch reindex: nightly or hourly for low-churn corpora.
- Streaming ingest: near-real-time embeddings for high-change sources.
- Versioned vectors: keep embeddings with model version tags to support gradual rollout.

Cache invalidation: when content updates, remove associated vector IDs and purge caches for affected keys.

---

## 10 — Operational concerns: monitoring, costs, scaling

Monitoring signals:

- Query latency (retrieval time, reranker time, LLM time).
- Recall/precision (via periodic golden queries).
- Token usage and cost per request.
- Index health: compaction stats, memory/CPU usage, disk utilization.
- Embedding pipeline failures and lag (backlog length).

Cost controls:

- Use caching and warm-start rerankers to reduce LLM calls.
- Use smaller models for reranking or distilled models.
- Set hard limits on token usage and fall back to cheaper prompts.

Scaling patterns:

- Separate throughput-critical components: ANN cluster, embedding workers, and LLM pool.
- Autoscale based on queue length / event lag (KEDA for queues).

---

## 11 — Security & privacy considerations

- PII handling: mask or redact sensitive fields before indexing.
- Access controls: restrict who can query which corpora; enforce tenant isolation.
- Encryption at rest for vector DB and metadata store.
- Audit logs: track queries, responses, and data access for compliance.

---

## 12 — Kubernetes & deployment examples

Small examples below show how to deploy embedding workers and a retriever service. For production, split services into separate deployments, use HPA, and autoscale by custom metrics.

### Example: embedding-worker (Deployment snippet)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: embedding-worker
spec:
	replicas: 2
	template:
		spec:
			containers:
				- name: embedder
					image: ghcr.io/debasnan10/embedder:1.0.0
					env:
						- name: EMBED_MODEL
							value: "openai-embed-002"
					resources:
						requests:
							cpu: "500m"
							memory: "1Gi"
						limits:
							cpu: "2"
							memory: "4Gi"
```

### Example: retriever service (Deployment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
	name: retriever
spec:
	replicas: 3
	template:
		spec:
			containers:
				- name: retriever
					image: ghcr.io/debasnan10/retriever:1.0.0
					env:
						- name: VECTOR_DB_ENDPOINT
							value: "vectordb:8000"
					resources:
						requests:
							cpu: "250m"
							memory: "512Mi"
```

---

## 13 — Example artifacts (code snippets)

### Example: Querying a vector DB + LLM (Python pseudo)

```python
def answer_query(query):
		q_emb = embed(query)
		candidates = vector_db.search(q_emb, top_k=20)
		reranked = reranker.rank(query, candidates[:50])
		context = format_context(reranked[:5])
		prompt = PROMPT_TEMPLATE.format(context=context, question=query)
		if cache.exists(hash(prompt)):
				return cache.get(hash(prompt))
		response = llm.generate(prompt)
		cache.set(hash(prompt), response, ttl=60)
		return attach_sources(response, reranked[:5])
```

### OTel collector minimal config (example)

```yaml
receivers:
	otlp:
		protocols:
			http:
			grpc:
exporters:
	prometheus:
		endpoint: "0.0.0.0:8889"
service:
	pipelines:
		traces:
			receivers: [otlp]
			exporters: [prometheus]
```

---

## 14 — Trade-offs and alternatives

- Managed vs self-hosted vector DB: managed reduces ops but costs more; self-hosted gives control.
- Heavy reranking improves accuracy but increases cost/latency.
- Streaming ingestion gives freshness at higher cost; batch is cheaper and simpler.

---

## 15 — Runbook checklist & SLOs

Suggested SLOs:

- Query availability: 99.9%.
- Retrieval latency: P95 < 200ms (ANN + network).
- End-to-end response P95 < 1s for cached queries; < 3s for cold LLM calls.

Runbook starters:

- Detect: alert on query error rate, embedding pipeline lag, index OOMs.
- Triage: check recent deploys, index health, embedding backlog.
- Mitigate: scale index nodes, roll back deploy, disable expensive reranker.

---

## 16 — Next steps & recommended repo layout

Suggested repo layout:

```
rag-system/
├── apis/                # OpenAPI, AsyncAPI specs
├── ingest/              # ingestion connectors and pipelines
├── embedder/            # embedding service code
├── vectordb/            # index management and scripts
├── retriever/           # retriever + reranker code
├── llm/                 # prompt templates and LLM wrappers
├── k8s/                 # kubernetes manifests
├── observability/       # OTEL, prometheus rules, grafana
└── docs/                # design docs (this file)
```

---

## 17 — References & further reading

- HNSW: https://arxiv.org/abs/1603.09320
- FAISS: https://github.com/facebookresearch/faiss
- RAG paper and recent blog posts from tech orgs (Pinecone, Weaviate)

---

## 18 — Appendix: Example event (order.created) and embedding note

Event envelope (order.created):

```json
{
  "event": "order.created",
  "version": "1.0.0",
  "data": {
    "orderId": "uuid",
    "customerId": "uuid",
    "items": [{ "sku": "ABC", "qty": 2 }],
    "total": 123.45,
    "createdAt": "2025-11-20T10:00:00Z"
  }
}
```

Embedding note: store model name and version with each vector. When rolling out new embedding models, consider running A/B: keep old vectors available or re-embed on a rolling schedule.

---

**Author:** Debasnan Singh

**Copyright (c) 2025 Debasnan Singh. Licensed under the MIT License.**
