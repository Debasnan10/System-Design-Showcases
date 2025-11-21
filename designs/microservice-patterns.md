# Microservice Architecture — Enterprise Deep Dive

**Prepared by:** Debasnan Singh

---

## Purpose

This document is an enterprise-grade, end-to-end guide to designing, building, operating, and evolving production microservice architectures. It includes principles, patterns, code/layout examples, Mermaids + ASCII diagrams, Kubernetes manifests, CI/CD workflows, logging/tracing examples, and operational practices. Read it cover-to-cover to get a working mental model and enough artifacts to bootstrap production-ready microservice systems.

---

## Table of Contents

1. Executive summary
2. Principles & trade-offs
3. Microservice types & boundaries
4. High-level architecture (diagrams)
5. Service contracts: REST + Async patterns + examples
6. Data management & consistency patterns (SAGA, CQRS)
7. Inter-service communication patterns (sync/async, grpc, Kafka)
8. Resilience & reliability (circuit breakers, retry, bulkheads)
9. Observability: logs, metrics, traces, dashboards, alerts
10. Security & compliance (identity, authz, secrets)
11. Deployment strategies & CI/CD (blue-green, canary, GitOps)
12. Kubernetes best practices & YAML examples
13. Infrastructure as Code & Terraform snippets
14. Testing strategies (unit, contract, integration, chaos)
15. Performance, scaling & cost optimization
16. Anti-patterns & common pitfalls
17. Folder structure & repo patterns (sample trees)
18. Example artifacts: API contract, GitHub Actions, OpenTelemetry JSON, Prometheus config, Grafana hint
19. Operational runbook checklist & SLOs
20. Next steps & how to use this guide

---

## 1 — Executive summary

Microservices break a monolith into independently deployable services. Benefits: modularity, team autonomy, independent scaling, and faster deployments. Trade-offs: operational complexity, distributed-systems challenges (consistency, latency), and higher platform needs (observability, automation). This guide surfaces battle-tested patterns and practical artifacts for production readiness.

---

## 2 — Principles & trade-offs

### Core principles

- Single Responsibility: each service owns one business capability.
- Bounded Context: align services with business contexts (DDD).
- Autonomy: teams control code → deploy → operate.
- API-First: design contracts first, then implement.
- Observability-by-default: logs/metrics/traces baked in.
- Resilience-over-simplicity: expect failures and plan for them.
- Automate everything: IaC, CI/CD, provisioning.

### Trade-offs to accept

- Operational complexity increases; platform/infra investment required.
- Network latency replaces local calls—measure and optimize.
- Eventual consistency is typical—choose transactional patterns carefully.

---

## 3 — Microservice types & boundaries

Common service types:

- CRUD services: domain data owners.
- Orchestration services: coordinate multi-step flows.
- Worker services: background/batch processing.
- Gateway/Edge services: auth, rate-limit, routing.
- Adapter services: external system interfaces.
- AI/ML inference services: model serving (GPU/scale concerns).

Boundary guidance: identify bounded contexts (events & domain), size by change frequency, own your data.

---

## 4 — High-level architecture (diagrams)

### Mermaid diagram (end-to-end)

```mermaid
flowchart LR
  subgraph U[Users]
    UX[Web/Mobile/CLI]
  end

  subgraph Edge
    G[API Gateway (Auth, Rate Limit, TLS)]
  end

  subgraph Services
    S1[Orders Service]
    S2[Inventory Service]
    S3[Pricing Service]
    S4[Billing Service]
    S5[AI Inference Service]
    Worker[Background Workers]
  end

  subgraph Messaging
    K[Kafka / Event Broker]
  end

  subgraph Data
    P[(Postgres)]
    V[(Vector DB)]
    C[(Cache - Redis)]
  end

  UX -->|HTTPS| G --> S1
  G --> S2
  S1 -->|sync| S3
  S1 -->|async produce order.created| K
  K --> S2
  K --> S4
  S5 -->|calls| V
  Worker -->|consume| K
  S2 --> P
  S1 --> P
  S1 --> C
```

### ASCII (simple)

```
[User] --> [API Gateway] --> [Order Service] --> [Postgres]
                                  |
                                  v (async: order.created)
                               [Kafka Topics] --> [Billing Service]
                                               --> [Workers]
```

---

## 5 — Service contracts: REST + Async patterns + examples

API-first: use OpenAPI / AsyncAPI. Version schemas; keep contracts explicit.

#### Example OpenAPI (Orders) snippet

```yaml
openapi: 3.0.3
info:
  title: Orders API
  version: 1.0.0
paths:
  /orders:
    post:
      summary: Create an order
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/OrderCreate"
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Order"
components:
  schemas:
    OrderCreate:
      type: object
      properties:
        customerId: { type: string }
        items:
          type: array
          items:
            $ref: "#/components/schemas/OrderItem"
      required: [customerId, items]
    OrderItem:
      type: object
      properties:
        sku: { type: string }
        qty: { type: integer }
    Order:
      allOf:
        - $ref: "#/components/schemas/OrderCreate"
        - type: object
          properties:
            id: { type: string }
            status: { type: string }
```

#### Async event (order.created)

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

Guidelines: small envelopes (event, version, data, metadata); use schema registry for strong typing; support idempotency.

---

## 6 — Data management & consistency patterns

Each service is the single source of truth for its data; avoid cross-service DB writes.

### SAGA (choreography vs orchestration)

- Choreography: services publish/subscribe to events — decentralized.
- Orchestration: central orchestrator coordinates steps and compensating actions.

Example choreography SAGA:

1. Order created -> order.created
2. Inventory reserves -> inventory.reserved
3. Billing charges -> payment.completed
4. Order confirms; failures trigger compensations (inventory.release)

### CQRS

Separate write models from read (denormalized) views for scale; adds complexity in sync.

---

## 7 — Inter-service communication

### Synchronous (REST/gRPC)

Use for low-latency request/response (price lookups). gRPC for typed, high-throughput internal calls.

### Asynchronous (Kafka, RabbitMQ)

Use for decoupling, durability, and scaling. Events enable eventual consistency.

#### Outbox pattern (pseudo-steps)

1. Begin DB transaction.
2. Write business data + outbox row.
3. Commit.
4. Outbox relay reads row, publishes to broker, marks sent.

---

## 8 — Resilience & reliability

- Retries: exponential backoff with jitter.
- Circuit Breakers: fail fast for unhealthy downstreams.
- Bulkheads: isolate resources per downstream.
- Timeouts: always set sensible client/server timeouts.
- Rate limiting: implement at gateway and per-service.
- Health checks: readiness and liveness probes.

---

## 9 — Observability: logs, metrics, traces

### Logs (structured JSON)

```json
{
  "timestamp": "2025-11-20T10:01:02Z",
  "service": "orders",
  "level": "info",
  "message": "order created",
  "orderId": "uuid",
  "traceId": "trace-123"
}
```

### Metrics

Emit counters, histograms, gauges (Prometheus client libs). Standard metrics: latency histograms, error rates, queue lengths.

### Traces

Use OpenTelemetry; propagate W3C trace-context headers.

**OTel span (pseudo):**

```json
{ "traceId": "abc", "spanId": "123", "name": "POST /orders" }
```

**Prometheus scrape example:**

```yaml
scrape_configs:
  - job_name: "orders"
    static_configs:
      - targets: ["orders:9100"]
```

Recommended stack: Prometheus, Grafana, Loki/ELK, Jaeger/Tempo.

---

## 10 — Security & compliance

- Identity: OAuth2/OpenID Connect; verify JWTs or introspect tokens.
- mTLS/service mesh for service-to-service auth in sensitive environments.
- Authorization: central policy service or RBAC with claims.
- Secrets: Vault / AWS Secrets Manager / KMS-encrypted Kubernetes Secrets.
- Network: Kubernetes NetworkPolicies, namespace isolation.
- Compliance: log access events, PII redaction, retention policies.

---

## 11 — Deployment strategies & CI/CD

- CI: automated builds, unit tests, static analysis, SAST on PRs.
- CD models: Blue-Green, Canary, Rolling. GitOps (ArgoCD) for declarative deployments.

**GitHub Actions example (build → push):**

```yaml
name: CI
on: [push]
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker
        run: docker build -t ghcr.io/debasnan10/orders:latest .
      - name: Publish
        run: echo "publish to registry"
```

Promotion pipeline: dev → staging → prod with smoke and canary checks.

---

## 12 — Kubernetes best practices & YAML examples

Core ideas: small immutable images, resource requests/limits, readiness/liveness probes, HPA.

**Deployment (orders):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: orders
          image: ghcr.io/debasnan10/orders:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 20
```

**HPA:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orders
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

---

## 13 — Infrastructure as Code & Terraform snippet

**S3 + ECR sample:**

```hcl
provider "aws" { region = "us-east-1" }

resource "aws_ecr_repository" "orders" {
  name = "orders"
}

resource "aws_s3_bucket" "artifacts" {
  bucket = "debasnan-artifacts-unique-123"
  acl    = "private"
}
```

Best practices: modules, remote state (S3 + DynamoDB locking), separate state per env.

---

## 14 — Testing strategies

- Unit tests: fast, isolated.
- Contract tests: provider/consumer checks (Pact).
- Integration tests: testcontainers or lightweight infra.
- E2E tests: critical flows on staging.
- Chaos tests: kill pods, inject latency.
- Performance tests: k6 / JMeter.

CI: PRs run unit & static checks; nightly integration/e2e; canary pipelines for promotion.

---

## 15 — Performance, scaling & cost optimization

- Autoscaling: HPA + Cluster Autoscaler; KEDA for queue-driven scaling.
- Caching: Redis for hot reads; prefer simple TTL or event-driven invalidation.
- Latency: use gRPC and regional co-location for internal calls.
- Cost: right-size pods/nodes, use spot for non-critical, node pool separation.

---

## 16 — Anti-patterns & common pitfalls

- Distributed monolith (chatty sync calls) — revisit boundaries.
- Shared DB across services — breaks autonomy.
- No observability — running blind.
- Manual releases — unstable and slow.

---

## 17 — Folder structure & repo patterns (sample)

Suggested service repo layout:

```text
orders-service/
├── src/
│   ├── api/
│   ├── domain/
│   ├── infra/
│   └── workers/
├── tests/
├── Dockerfile
├── helm/
├── k8s/
├── .github/workflows/ci.yml
└── README.md
```

Platform repo:

```text
platform/
├── terraform/
├── charts/
├── observability/
├── scripts/
└── README.md
```

---

## 18 — Example artifacts

### a) GitHub Actions (build + push)

```yaml
name: Build & Push
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v3
        with:
          dotnet-version: "8.0.x"
      - name: Build
        run: dotnet publish -c Release -o out
      - name: Build Docker Image
        run: docker build -t ghcr.io/debasnan10/orders:${{ github.sha }} .
      - name: Push to GHCR
        run: echo "docker push..."
```

### b) OpenTelemetry collector (minimal)

```yaml
receivers:
  otlp:
    protocols:
      http:
      grpc:
exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  otlp/tempo:
    endpoint: tempo:4317
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      exporters: [prometheus]
```

### c) Logging JSON sample

```json
{
  "timestamp": "2025-11-20T10:00:00Z",
  "service": "orders",
  "env": "prod",
  "level": "error",
  "message": "payment failed",
  "orderId": "uuid",
  "traceId": "abc-123",
  "error": "insufficient-funds"
}
```

### d) Prometheus alert (latency)

```yaml
groups:
  - name: orders.rules
    rules:
      - alert: HighOrderLatency
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="orders"}[5m])) by (le))
        for: 2m
        labels:
          severity: "page"
        annotations:
          summary: "Orders P95 latency high"
```

### e) Grafana dashboard hint

Panels: Request rate, P95 latency, Error rate, CPU usage, Heap memory, Queue length, Inflight requests.

---

## 19 — Operational runbook checklist & SLOs

SLO examples:

- Availability: 99.95% monthly target (SLA 99.9%).
- Latency: P95 < 200ms for API calls.
- Error budget: defined % tracked daily.

Runbook starters: incident detection, triage checklist, mitigation steps, postmortem template.

---

## 20 — Next steps & how to use this guide

Immediate repo actions:

- Add this file to `docs/microservice-architecture.md`.
- Add OpenAPI spec for Orders to `apis/orders/openapi.yaml`.
- Add sample `k8s/` manifests and `.github/workflows/` CI for the sample service.
- Add observability config to `platform/observability/`.

I can generate any subset of these artifacts (OpenAPI, k8s manifests, GitHub Actions, Helm skeleton, OTEL config). Tell me which ones and I will create them under the repository structure.

---

## Appendix A — Sample Order flow (end-to-end)

Client POST /orders → API Gateway → Orders service (creates DB row, returns 202/201).

Orders service writes to DB + outbox row. Outbox processor publishes `order.created` to Kafka. Inventory reserves, Billing charges, and Orders service finalizes status based on resolution events. Workers handle notifications, invoices.

---

## Appendix B — Glossary

- Outbox: DB table for events awaiting publish.
- CQRS: Command Query Responsibility Segregation.
- SAGA: Pattern for distributed transactions.
- HPA: Horizontal Pod Autoscaler.
- OTel: OpenTelemetry.

---

**Author:** Debasnan Singh

## **Copyright (c) 2025 Debasnan Singh. Licensed under the MIT License.**

## 18 — Example artifacts

### a) GitHub Actions (build + push + deploy to staging)

```yaml
name: Build & Push
on: [push]
jobs:
	build:
		runs-on: ubuntu-latest
		steps:
			- uses: actions/checkout@v4
			- uses: actions/setup-dotnet@v3
				with:
					dotnet-version: '8.0.x'
			- name: Build
				run: dotnet publish -c Release -o out
			- name: Build Docker Image
				run: docker build -t ghcr.io/debasnan10/orders:${{ github.sha }} .
			- name: Push to GHCR
				run: echo "docker push..."
```

### b) OpenTelemetry (pseudo config)

Add OTel SDK to services; export to OTLP endpoint (Tempo/Collector).

**OTel collector config (minimal):**

```yaml
receivers:
	otlp:
		protocols:
			http:
			grpc:
exporters:
	prometheus:
		endpoint: "0.0.0.0:8889"
	otlp/tempo:
		endpoint: tempo:4317
service:
	pipelines:
		traces:
			receivers: [otlp]
			exporters: [otlp/tempo]
		metrics:
			receivers: [otlp]
			exporters: [prometheus]
```

### c) Logging JSON format sample

```json
{
  "timestamp": "2025-11-20T10:00:00Z",
  "service": "orders",
  "env": "prod",
  "level": "error",
  "message": "payment failed",
  "orderId": "uuid",
  "traceId": "abc-123",
  "error": "insufficient-funds"
}
```

### d) Prometheus alert (latency)

```yaml
groups:
- name: orders.rules
	rules:
	- alert: HighOrderLatency
		expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="orders"}[5m])) by (le))
		for: 2m
		labels: { severity: "page" }
		annotations:
			summary: "Orders P95 latency high"
```

### e) Grafana dashboard hint

Panels: Request rate, P95 latency, Error rate, CPU usage, Heap memory, Queue length, Inflight requests.

---

## 19 — Operational runbook checklist & SLOs

### SLO examples

- **Availability:** 99.95% per month (target); SLA 99.9%.
- **Latency:** 95th percentile < 200ms for API calls.
- **Error budget:** define allowed error %; track daily.

### Runbook starters

- **Incident detection:** alert triggers, runbook link, paging plan.
- **Initial triage:** check leaderboards (Grafana), recent deploys, failed health checks.
- **Mitigation:** roll back or scale out, circuit break, drain nodes.
- **Postmortem:** document timeline, root cause, action items.

---

## 20 — Next steps & how to use this guide

Immediate actions for repo:

- Add this document to System-Design-Showcase/docs/microservice-architecture.md.
- Add the OpenAPI spec for one service (Orders) into system-design-showcase/apis/orders/.
- Create sample k8s deployments/ and CI workflows for the sample services.
- Add observability config to platform/observability/.

Want me to generate artifacts for the repo?
I can create:

- Full apis/orders/openapi.yaml (complete).
- k8s/ manifests for orders, including HPA & Service.
- GitHub Actions full CI for a .NET service.
- OpenTelemetry Collector config & Prometheus rule files.
- A sample Helm chart skeleton.

Tell me which you want and I’ll generate them ready-to-paste into the System-Design-Showcase repo.

---

## Appendix A — Sample Order flow (end-to-end sequence)

Client POST /orders → API Gateway → Orders service (creates DB row, returns 202 or 201).

Orders service writes to DB + outbox row.

Outbox processor reads outbox and publishes order.created to Kafka.

Inventory service consumes order.created, reserves items, emits inventory.reserved.

Billing service consumes order.created, attempts charge; emits payment.succeeded or payment.failed.

Orders service consumes resolution events and updates order status.

Workers perform async tasks (send invoice, notify customer).

---

## Appendix B — Quick glossary

- **Outbox:** DB table storing events to be sent to broker to avoid partial failure.
- **CQRS:** Command Query Responsibility Segregation.
- **SAGA:** Pattern for distributed transactions.
- **HPA:** Horizontal Pod Autoscaler.
- **OTel:** OpenTelemetry.
- **RAG:** Retrieval Augmented Generation.

---

## Final notes (authorship & usage)

This document is crafted to be a canonical, enterprise-ready microservice architecture spec for a system-design showcase. Use it as a base for your repository, adjust names and fields to your domain, and I can auto-generate all referenced artifacts (OpenAPI, k8s manifests, GitHub workflows, Helm chart skeletons, OTEL config).

**Author:** Debasnan Singh

## **Copyright (c) 2025 Debasnan Singh. Licensed under the MIT License.**

## 13 — Infrastructure as Code & Terraform snippet

**S3 + ECR sample:**

```hcl
provider "aws" { region = "us-east-1" }

resource "aws_ecr_repository" "orders" {
	name = "orders"
}

resource "aws_s3_bucket" "artifacts" {
	bucket = "debasnan-artifacts-unique-123"
	acl    = "private"
}
```

**Terraform best practice:**

- Use modules for repeatable patterns (k8s cluster module, networking module).
- Store state in remote backend (S3 + DynamoDB locking).
- Use workspaces per environment or separate state per environment.

---

## 14 — Testing strategies

- **Unit tests:** fast, single-service.
- **Contract tests:** ensure provider/consumer compatibility (Pact or similar).
- **Integration tests:** spin up dependent services or mocks with testcontainers.
- **End-to-end tests:** run critical user flows against staging.
- **Chaos testing:** randomly kill pods, add latency, test recovery (Chaos Monkey, Litmus).
- **Performance tests:** use k6 or JMeter for load tests.

**CI recommendations:**

- PR builds run unit & static checks.
- Nightly jobs run integration & e2e.
- Canary / staging runs smoke tests and monitors metrics before promotion.

---

## 15 — Performance, scaling & cost optimization

### Autoscaling

Use HPA + Cluster Autoscaler. For event-driven workloads, use KEDA for scaling on queue length.

### Cache strategy

Use Redis for hot-read caches. Cache invalidation policy should be simple (TTL) or event-driven for strong consistency.

### Latency reduction

Co-locate services by network region, use gRPC for internal calls where latency matters.

### Cost optimization

Right-size pods and nodes. Use spot instances for non-critical workloads. Use autoscaler with node pool separation for CPU/GPU workloads.

---

## 16 — Anti-patterns & common pitfalls

- **Distributed monolith:** broken decomposition that results in chatty, synchronous calls.
- **Shared database between services:** kills service independence.
- **No observability:** running blind is worse than having fewer services.
- **Large services per deploy:** frequent deploy conflicts.
- **Manual releases:** no automation → unstable releases.

---

## 17 — Folder structure & repo patterns (sample)

### Monorepo vs Polyrepo

- **Polyrepo:** each service in separate repo (clean ownership, independent releases).
- **Monorepo:** multiple services in one repo (easier refactors, single CI config).

**Suggested service repo layout**

```text
orders-service/
├── src/
│   ├── api/
│   ├── domain/
│   ├── infra/
│   └── workers/
├── tests/
├── Dockerfile
├── helm/
├── k8s/
├── .github/workflows/ci.yml
├── README.md
└── docs/
```

**Platform repo (infra + shared libs)**

```text
platform/
├── terraform/
├── charts/
├── observability/
│  ├── grafana/
│  └── prometheus/
├── scripts/
└── README.md
```

---

---

## 7 — Inter-service communication

### Synchronous (REST/gRPC)

Use for request/response where low latency and immediate result are required (e.g., price lookups).

gRPC is stronger for binary performance, typed contracts, and internal services.

### Asynchronous (Kafka, RabbitMQ)

Use for decoupling, resilience, and scaling.

Events are the primary method for eventual consistency and reactivity.

#### Patterns

- Request-Reply via message bus when async response is okay.
- Event Sourcing for append-only audit trail (higher complexity).
- Outbox pattern to ensure atomic write + event publish (write DB + outbox table + broker relay).

**Outbox pseudo-steps:**

1. Begin DB transaction.
2. Write business data + event row into outbox table.
3. Commit DB transaction.
4. Separate process reads outbox and sends to broker; marks sent.

---

## 8 — Resilience & reliability

### Retry & Backoff

Implement exponential backoff with jitter. Avoid synchronous retries that block threads/resources.

### Circuit Breaker

Short-circuit calls to an unhealthy downstream service to fail fast and allow recovery.

### Bulkheads

Isolate resources (thread pools, connection pools) per downstream to avoid cascading failures.

### Timeouts

Never use infinite timeouts. Set appropriate client & server timeouts.

### Rate Limiting & Throttling

Implement at the gateway and per-service (leaky bucket / token bucket).

### Health Checks

- **Liveness:** restart when the process is unhealthy.
- **Readiness:** only route traffic when dependencies are available.

---

## 9 — Observability: logs, metrics, traces

Observability is required to run microservices in production.

### Logs

Structured logs (JSON) with contextual fields: trace_id, span_id, service, env, level, timestamp.

**Example log JSON:**

```json
{
  "timestamp": "2025-11-20T10:01:02Z",
  "service": "orders",
  "level": "info",
  "message": "order created",
  "orderId": "uuid",
  "traceId": "trace-123"
}
```

### Metrics

Emit application metrics (counters, gauges, histograms) via Prometheus client libs.

**Standard metrics:** request latency histogram, error rate, inflight requests, queue length.

### Traces

Use OpenTelemetry (OTel) for distributed tracing. Every request should propagate traceparent headers (W3C trace-context).

**OpenTelemetry Span example (pseudo JSON):**

```json
{
  "traceId": "abc",
  "spanId": "123"
}
```

### Observability stack recommendation

- Prometheus for metrics collection
- Grafana for dashboards & alerting
- Loki / ELK (Elasticsearch, Fluentd/Logstash, Kibana) for logs

**Prometheus scrape config (example):**

```yaml
scrape_configs:
	- job_name: 'orders'
		static_configs:
			- targets: ['orders:9100']
```

---

---

## 4 — High-level architecture (diagrams)

### Mermaid diagram (end-to-end microservices)

```mermaid
flowchart LR
	subgraph U[Users]
		UX[Web/Mobile/CLI]
	end

	subgraph Edge
		G[API Gateway (Auth, Rate Limit, TLS)]

	subgraph Services
		S1[Orders Service]
		S2[Inventory Service]
		S3[Pricing Service]
		S4[Billing Service]
		S5[AI Inference Service]
		Worker[Background Workers]
	end

	subgraph Data
		P[(Postgres)]
		V[(Vector DB)]
		C[(Cache - Redis)]
	end

	UX -->|HTTPS| G --> S1
	G --> S2
	S1 -->|sync| S3
	S1 -->|async produce order.created| K
	K --> S2
	K --> BillingService[(Billing Service)]
	S5 -->|calls| V
	Worker -->|consume| K
	S2 --> P
	S1 --> P
	S1 --> C

### ASCII (simple)
```

[User] --> [API Gateway] --> [Order Service] --> [Postgres]

````

---

## 5 — Service contracts: REST + Async patterns + examples

**API first:** use OpenAPI / AsyncAPI to define contracts. Keep schemas explicit and versioned.

#### Example REST contract (OpenAPI snippet for Orders)
```yaml
openapi: 3.0.3
info:
	title: Orders API
	version: 1.0.0
paths:
	/orders:
		post:
			summary: Create an order
			requestBody:
				required: true
				content:
					application/json:
						schema:
							$ref: '#/components/schemas/OrderCreate'
			responses:
				'201':
					description: Created
					content:
						application/json:
							schema:
								$ref: '#/components/schemas/Order'
components:
	schemas:
		OrderCreate:
			type: object
			properties:
				customerId:
					type: string
				items:
					type: array
					items:
						$ref: '#/components/schemas/OrderItem'
			required: [customerId, items]
		OrderItem:
			type: object
			properties:
				sku: { type: string }
				qty: { type: integer }
		Order:
			allOf:
				- $ref: '#/components/schemas/OrderCreate'
				- type: object
					properties:
						status: { type: string }
````

#### Async contract (Kafka event example - order.created)

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

**Guidelines:**

- Use small, consistent event envelopes with event, version, data, metadata fields.
- Accept versioning: never change event fields in place—create v2 or add optional fields.
- Prefer idempotence: consumer must ignore duplicate events (store event id).
- Use schema registry (Avro/Protobuf) for strong typing in Kafka.

---

## 6 — Data management & consistency patterns

### Ownership & single source of truth

Each service owns its data. No cross-service database writes.

### Transactions & SAGA

Long-running multi-service transactions should use either:

- **Choreography:** services publish events; other services react. (Decentralized)
- **Orchestration:** central orchestrator coordinates steps and compensations.

#### SAGA pattern (choreography example):

1. Order service emits order.created.
2. Inventory service reserves stock; emits inventory.reserved.
3. Billing service charges; emits payment.completed.
4. Order service marks order confirmed.
5. If billing fails: Billing emits payment.failed → Inventory service releases stock via inventory.release.

Idempotence & compensations are crucial.

### CQRS

Separate read models (denormalized views) from write models to optimize scaling and reduce contention.

---

## 7 — Inter-service communication

### Synchronous (REST/gRPC)

Use for request/response where low latency and immediate result are required (e.g., price lookups).

gRPC is stronger for binary performance, typed contracts, and internal services.

### Asynchronous (Kafka, RabbitMQ)

Use for decoupling, resilience, and scaling.

Events are the primary method for eventual consistency and reactivity.

#### Patterns

- Request-Reply via message bus when async response is okay.
- Event Sourcing for append-only audit trail (higher complexity).
- Outbox pattern to ensure atomic write + event publish (write DB + outbox table + broker relay).

**Outbox pseudo-steps:**

1. Begin DB transaction.
2. Write business data + event row into outbox table.
3. Commit DB transaction.
4. Separate process reads outbox and sends to broker; marks sent.

---

---

## 2 — Principles & trade-offs

### Core principles

- **Single Responsibility:** each service owns one business capability.
- **Bounded Context:** align services with business contexts (DDD).
- **Autonomy:** teams control the full lifecycle (code → deploy → operate).
- **API-First:** design contracts first, then implement.
- **Observability-by-default:** logs/metrics/traces are baked in.
- **Resilience-over-simplicity:** expect failures and code for them.
- **Automate everything:** infra as code, CI/CD, infra provisioning.

### Trade-offs to accept

- Operational complexity increases; you need a platform team.
- Network latency replaces function calls—optimize and measure.
- Data consistency is eventual by default—select transactional patterns carefully.

---

## 3 — Microservice types & boundaries

Microservices fall into patterns:

- **CRUD services:** handle domain data.
- **Orchestration services:** coordinate flows across services.
- **Worker services:** background processing and batch jobs.
- **Gateway/Edge services:** rate-limit, auth, routing (API gateway).
- **Adapter services:** interface with external third-party systems.
- **AI/ML inference services:** host model servers (scalable, GPU-backed).

**Boundary guidance:** Use domain events to identify owned data, APIs to define interactions, and size services by change frequency, not lines of code.

---

## 4 — High-level architecture (diagrams)

### Mermaid diagram (end-to-end microservices)

```mermaid
flowchart LR
	subgraph U[Users]
		UX[Web/Mobile/CLI]
	end

	subgraph Edge
		G[API Gateway (Auth, Rate Limit, TLS)]
	end

	subgraph Services
		S1[Orders Service]
		S2[Inventory Service]
		S3[Pricing Service]
		S4[Billing Service]
		S5[AI Inference Service]
		Worker[Background Workers]
	end

	subgraph Messaging
		K[Kafka / Event Broker]
	end

	subgraph Data
		P[(Postgres)]
		V[(Vector DB)]
		C[(Cache - Redis)]
	end

	UX -->|HTTPS| G --> S1
	G --> S2
	S1 -->|sync| S3
	S1 -->|async produce order.created| K
	K --> S2
	K --> BillingService[(Billing Service)]
	S5 -->|calls| V
	Worker -->|consume| K
	S2 --> P
	S1 --> P
	S1 --> C
```

### ASCII (simple)

```
[User] --> [API Gateway] --> [Order Service] --> [Postgres]
																	|
																	v (async: order.created)
															 [Kafka Topics] --> [Billing Service]
																							 --> [Workers]
```

---

## Table of Contents

1. Executive summary
2. Principles & trade-offs
3. Microservice types & boundaries
4. High-level architecture (diagrams)
5. Service contracts: REST + Async patterns + examples
6. Data management & consistency patterns (SAGA, CQRS)
7. Inter-service communication patterns (sync/async, grpc, Kafka)
8. Resilience & reliability (circuit breakers, retry, bulkheads)
9. Observability: logs, metrics, traces, dashboards, alerts
10. Security & compliance (identity, authz, secrets)
11. Deployment strategies & CI/CD (blue-green, canary, GitOps)
12. Kubernetes best practices & YAML examples
13. Infrastructure as Code & Terraform snippets
14. Testing strategies (unit, contract, integration, chaos)
15. Performance, scaling & cost optimization
16. Anti-patterns & common pitfalls
17. Folder structure & repo patterns (sample trees)
18. Example artifacts: API contract, GitHub Actions, OpenTelemetry JSON, Prometheus config, Grafana hint
19. Operational runbook checklist & SLOs
20. Next steps & how to use this guide

---

## 1 — Executive summary

Microservices break a monolith into independently deployable services. The benefits are modularity, team autonomy, independent scaling, and optimized deployment. The trade-offs are operational complexity, distributed systems challenges (consistency, latency), and higher platform needs (observability, automation). This doc helps you adopt microservices while minimizing pitfalls by applying battle-tested patterns and modern platform tooling (Kubernetes, CI/CD, GitOps, observability stacks).

---

## 2 — Principles & trade-offs

### Core principles

- **Single Responsibility:** each service owns one business capability.
- **Bounded Context:** align services with business contexts (DDD).
- **Autonomy:** teams control the full lifecycle (code → deploy → operate).
- **API-First:** design contracts first, then implement.
- **Observability-by-default:** logs/metrics/traces are baked in.
- **Resilience-over-simplicity:** expect failures and code for them.
- **Automate everything:** infra as code, CI/CD, infra provisioning.

### Trade-offs to accept

- Operational complexity increases; you need a platform team.
- Network latency replaces function calls—optimize and measure.
- Data consistency is eventual by default—select transactional patterns carefully.

---

# Microservice Architecture — Enterprise Deep Dive

**Prepared by:** Debasnan Singh

---

## Purpose

This document is an enterprise-grade, end-to-end guide to designing, building, operating, and evolving production microservice architectures. It includes principles, patterns, code/layout examples, Mermaids + ASCII diagrams, Kubernetes manifests, CI/CD workflows, logging/tracing examples, and operational practices. It’s written to be practical and original — read it cover-to-cover and you’ll have a working mental model and enough artifacts to bootstrap production-ready microservice systems.

---

## Table of Contents

1. Executive summary
2. Principles & trade-offs
3. Microservice types & boundaries
4. High-level architecture (diagrams)
5. Service contracts: REST + Async patterns + examples
6. Data management & consistency patterns (SAGA, CQRS)
7. Inter-service communication patterns (sync/async, grpc, Kafka)
8. Resilience & reliability (circuit breakers, retry, bulkheads)
9. Observability: logs, metrics, traces, dashboards, alerts
10. Security & compliance (identity, authz, secrets)
11. Deployment strategies & CI/CD (blue-green, canary, GitOps)
12. Kubernetes best practices & YAML examples
13. Infrastructure as Code & Terraform snippets
14. Testing strategies (unit, contract, integration, chaos)
15. Performance, scaling & cost optimization
16. Anti-patterns & common pitfalls
17. Folder structure & repo patterns (sample trees)
18. Example artifacts: API contract, GitHub Actions, OpenTelemetry JSON, Prometheus config, Grafana hint
19. Operational runbook checklist & SLOs
20. Next steps & how to use this guide

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
