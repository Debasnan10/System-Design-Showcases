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
