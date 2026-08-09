# Technical Architect Interview Guide

> Comprehensive interview preparation guide with **detailed spoken answers**, real-world examples, trade-off analysis, and sample programs.  
> Stack emphasis: Node.js, Java, AWS, microservices, MongoDB, Redis, ElasticSearch, Temporal/Airflow, RabbitMQ, Kafka.

---

## Table of Contents

1. [How to Structure Answers](#1-how-to-structure-answers)
2. [Architecture Fundamentals](#2-architecture-fundamentals)
3. [Microservices & Cloud-Native AWS](#3-microservices--cloud-native-aws)
4. [Distributed Systems](#4-distributed-systems)
5. [Event-Driven Architecture (RabbitMQ & Kafka)](#5-event-driven-architecture-rabbitmq--kafka)
6. [Workflow Orchestration (Airflow vs Temporal)](#6-workflow-orchestration-airflow-vs-temporal)
7. [MongoDB](#7-mongodb)
8. [ElasticSearch / OpenSearch](#8-elasticsearch--opensearch)
9. [Redis (Distributed Cache)](#9-redis-distributed-cache)
10. [Algorithms & Data Structures](#10-algorithms--data-structures)
11. [Node.js](#11-nodejs)
12. [Java](#12-java)
13. [Design Patterns & SOLID](#13-design-patterns--solid)
14. [Testing, TDD & Agile](#14-testing-tdd--agile)
15. [Observability (Datadog)](#15-observability-datadog)
16. [DevOps: Docker, Kubernetes, Jenkins](#16-devops-docker-kubernetes-jenkins)
17. [System Design Walkthroughs](#17-system-design-walkthroughs)
18. [Behavioral & Communication](#18-behavioral--communication)
19. [Quick Reference Cheat Sheet](#19-quick-reference-cheat-sheet)

---

## 1. How to Structure Answers

### The COTDO Framework

Use this structure for every technical and architectural question:

| Step | What to Say | Time |
|------|-------------|------|
| **C**ontext | Business problem, scale, constraints | 15–20 sec |
| **O**ptions | 2–3 realistic alternatives | 20–30 sec |
| **T**rade-offs | Pros/cons of each option | 30–45 sec |
| **D**ecision | What you chose and why | 15–20 sec |
| **O**utcome | Measurable result or risk mitigated | 10–15 sec |

**Total target:** 90–120 seconds for a standard question; 3–5 minutes for system design.

### Why This Works

Interviewers at the architect level are not testing whether you know Redis exists. They are testing whether you can **reason under constraints**, **communicate trade-offs clearly**, and **justify decisions with evidence**. A senior answer always names what you gave up, not just what you gained.

### Full Example Answer (Checkout at Scale)

> **Context:** At my previous company, we ran an e-commerce platform processing about 5,000 orders per minute during peak sales events. Checkout involved three services — inventory, payment, and order confirmation — and we were seeing intermittent payment gateway timeouts that caused duplicate charges and inventory stuck in a reserved state.
>
> **Options:** We evaluated three approaches. First, a synchronous REST chain where the API gateway calls inventory, then payment, then order service in sequence. Second, a choreographed saga where each service publishes events and compensates on failure. Third, two-phase commit (2PC) across all three databases for atomicity.
>
> **Trade-offs:** The sync chain was simplest to build and debug, but a single slow payment call blocked the entire checkout thread, and retries without idempotency caused duplicate charges. 2PC gave us strong consistency but required distributed locks, hurt availability during network partitions, and none of our databases supported XA transactions natively at the scale we needed. The saga added operational complexity — we needed workflow state, compensating transactions, and monitoring — but it decoupled services, allowed independent retries, and scaled horizontally.
>
> **Decision:** We implemented a choreographed saga using Temporal. The workflow was: reserve inventory → charge payment → confirm order. On payment failure, we ran a compensation to release inventory. Each step was an idempotent activity with its own retry policy and timeout.
>
> **Outcome:** Checkout success rate improved from 97.2% to 99.95%. P95 checkout latency dropped from 4.2 seconds to 1.1 seconds because we no longer blocked on synchronous chains. We eliminated duplicate charges by enforcing idempotency keys at the payment service. The main risk we accepted was eventual consistency in the order status UI — customers might see "processing" for 2–3 seconds, which we mitigated with optimistic UI updates.

### Common Mistakes to Avoid

| Mistake | Why It Hurts | Fix |
|---------|--------------|-----|
| Jumping to a solution | Shows no analytical thinking | Always state context and options first |
| Only naming technologies | Sounds like buzzword bingo | Explain *why* that technology fits |
| Ignoring failure modes | Architects own reliability | Mention what happens when things break |
| No numbers | Claims feel hollow | Use latency, throughput, error rates, cost |
| Talking for 5+ minutes | Loses interviewer attention | Pause and ask "should I go deeper on X?" |

### Follow-Up Handling

When the interviewer asks "What would you do differently?", answer honestly:

> "In hindsight, I would have invested in contract testing between the inventory and payment services earlier. We caught two breaking API changes in production that contract tests would have blocked in CI. I would also have added a dead-letter queue for failed compensations — we had one edge case where a compensation retry exhausted and needed manual intervention."

---

## 2. Architecture Fundamentals

### Scalability

| Type | Meaning | Example | When to Use |
|------|---------|---------|-------------|
| Vertical | Bigger machine (more CPU/RAM) | 8 → 32 GB RAM | Quick fix, DB that can't shard easily |
| Horizontal | More machines | 3 → 30 API pods behind ALB | Stateless services, long-term scale |

**Rule:** Prefer horizontal scaling for stateless services; use caching, sharding, and async for stateful bottlenecks.

#### Q: How do you decide between vertical and horizontal scaling?

> **Detailed Answer:** I start by identifying whether the bottleneck is **stateless or stateful**. For stateless API services — REST endpoints that don't hold in-memory session state — horizontal scaling is almost always the right answer. You add more instances behind a load balancer, use auto-scaling groups based on CPU or request rate, and you get linear throughput gains up to the point where your database becomes the bottleneck.
>
> Vertical scaling makes sense in three situations. First, when you have a **single-threaded bottleneck** — some legacy Java apps or certain database operations can't parallelize across cores effectively, so a bigger machine helps. Second, for **managed databases** during a migration period — it's faster to upgrade an RDS instance from `db.r5.xlarge` to `db.r5.4xlarge` than to implement sharding. Third, for **licensed software** where per-node licensing makes horizontal scaling expensive.
>
> In practice, I use vertical scaling as a **bridge** while designing the horizontal solution. For example, when our PostgreSQL primary hit 80% CPU, we vertically scaled immediately to buy time, then implemented read replicas and eventually sharded by `tenant_id` over the next quarter. The key metric I watch is **cost per transaction** — if horizontal scaling gives you 3x throughput for 2.5x cost, that's a win. If it gives 1.2x throughput for 3x cost because of coordination overhead, you need a different approach.

### CAP Theorem

During a network partition, a distributed system must choose between **Consistency** (every read returns the latest write) and **Availability** (every request gets a response) — you cannot guarantee both.

- **CP (Consistency + Partition tolerance):** Banking ledger, inventory counts during flash sales, distributed locks
- **AP (Availability + Partition tolerance):** Social feed likes, shopping cart, DNS, CDN edge caches
- **CA (Consistency + Availability):** Only possible in a single-node system with no partitions — not realistic in distributed systems

#### Q: Explain CAP theorem with a real example.

> **Detailed Answer:** CAP theorem states that in a distributed system, when a network partition occurs, you must choose between consistency and availability. You cannot have both. Partition tolerance is non-negotiable in any real distributed system because networks fail — switches go down, AZs lose connectivity, packets get dropped.
>
> Let me give a concrete example. Imagine a shopping platform with inventory data replicated across two data centers — US-East and US-West. A network partition occurs between them. A customer in US-East buys the last unit of a product.
>
> If we choose **consistency (CP)**, the US-West replica must reject reads and writes until it confirms the inventory change from US-East. Customers in US-West see errors or timeouts — the system is unavailable for those users, but inventory count is always accurate. This is what you want for a limited-edition product drop where overselling is unacceptable.
>
> If we choose **availability (AP)**, both data centers continue serving requests. A customer in US-West might also "buy" the last unit because their replica hasn't received the update yet. You've oversold by one unit, but no customer sees an error. You reconcile later — cancel one order, offer a discount, or source from another warehouse. This is acceptable for low-value items or when business policy allows backorders.
>
> In practice, most systems are **not purely CP or AP** — they use tunable consistency. Amazon DynamoDB lets you choose `strong` or `eventual` consistency per read. Cassandra offers `ONE`, `QUORUM`, and `ALL` consistency levels. MongoDB replica sets default to primary reads (strong) but allow `readPreference: secondary` for eventual consistency on analytics queries. As an architect, my job is to map each data domain to the right consistency level based on business impact of stale reads.

### Consistency Models

| Model | Behavior | Example Use Case |
|-------|----------|------------------|
| Strong | Read always returns latest write | Account balance, inventory reservation |
| Eventual | Replicas converge over time | Product catalog, user profile cache |
| Read-your-writes | User sees their own updates immediately | User edits profile, sees change instantly |
| Monotonic reads | User never sees data go "backwards in time" | Social media timeline |
| Causal | Causally related operations seen in order | Comment threads, chat messages |

#### Q: When is eventual consistency acceptable, and how do you handle it in the UI?

> **Detailed Answer:** Eventual consistency is acceptable when the **business cost of a stale read is low** and the **benefit of higher availability and lower latency is high**. Common examples: product search results that are 2–3 seconds behind the catalog database, social media like counts, recommendation feeds, and analytics dashboards.
>
> It is **not** acceptable for: financial transactions, inventory reservations during limited stock, authorization/permission checks, or any operation where acting on stale data causes irreversible harm.
>
> For the UI, I use several patterns to mask eventual consistency. **Optimistic updates** — when a user updates their cart, the UI reflects the change immediately while the API call is in flight. If the API fails, we roll back the UI state and show an error. **Version vectors or timestamps** — the client sends the last known version with each update; the server rejects updates based on stale versions (optimistic concurrency control). **Polling with backoff** — after a write, poll the read endpoint every 500ms for up to 5 seconds until the expected state appears. **WebSocket/SSE push** — the server notifies the client when the write has propagated to the read replica.
>
> I always document the **maximum staleness SLA** for each data domain. For example: "Product search results may be up to 5 seconds behind the source of truth." This sets expectations with product teams and helps QA write correct test scenarios.

### Key Patterns

| Pattern | Use When | Avoid When |
|---------|----------|------------|
| API Gateway | Single entry, auth, rate limiting, protocol translation | Simple internal service-to-service calls |
| Circuit Breaker | External dependency failing repeatedly | Every call — adds latency and complexity |
| CQRS | Read/write load profiles differ significantly | Simple CRUD with similar read/write patterns |
| Saga | Distributed transaction across services | Single-service transactions (use local ACID) |
| Strangler Fig | Gradual monolith migration | Greenfield projects |
| Event Sourcing | Full audit trail needed, temporal queries | Simple state storage without history |
| Bulkhead | Isolate failures between dependency pools | Small apps with few dependencies |

#### Q: Explain the Saga pattern in detail. Choreography vs Orchestration?

> **Detailed Answer:** A saga is a sequence of **local transactions** across multiple services, where each step has a corresponding **compensating transaction** to undo its effect if a later step fails. Unlike 2PC, sagas do not hold locks across services — each service commits its own transaction independently.
>
> **Example — Order checkout saga:**
> 1. `ReserveInventory` → compensation: `ReleaseInventory`
> 2. `ChargePayment` → compensation: `RefundPayment`
> 3. `ConfirmOrder` → compensation: `CancelOrder`
> 4. `SendConfirmationEmail` → compensation: `SendCancellationEmail` (or no-op if email already sent)
>
> If step 2 fails, we run compensations for step 1 in reverse order. The system ends up in a consistent state — no reserved inventory, no charge, no confirmed order.
>
> **Choreography** means each service listens for events and decides what to do next. Inventory service publishes `InventoryReserved` → Payment service hears it and charges → publishes `PaymentCharged` → Order service hears it and confirms. **Pros:** Loose coupling, no central coordinator, services are autonomous. **Cons:** Hard to understand the full flow, difficult to debug, implicit dependencies, no single place to see workflow state.
>
> **Orchestration** means a central coordinator (workflow engine like Temporal, or an orchestrator service) tells each service what to do and in what order. **Pros:** Explicit workflow definition, easy to visualize, centralized error handling and retry, supports long-running workflows with human approval steps. **Cons:** Orchestrator is a single point of failure (mitigated by Temporal's durability), adds a dependency, can become a god-service if not careful.
>
> **My decision framework:** Use choreography for simple flows (2–3 steps, all services owned by one team, failure handling is straightforward). Use orchestration for complex flows (4+ steps, multiple teams, compensations, human-in-the-loop, workflows lasting hours or days). In my last project, we used Temporal for orchestration because our checkout flow had 7 steps, spanned 3 teams, and needed to survive pod restarts mid-workflow.

### ADR (Architecture Decision Record) Template

```markdown
# ADR-007: Use Redis for session cache

## Status
Accepted

## Context
10M sessions/day; DB session lookups add 40ms p95 latency.

## Decision
Store sessions in Redis with 24h TTL; sticky sessions avoided.

## Alternatives
- DB sessions: simple but slow at scale
- JWT only: stateless but hard to revoke instantly

## Consequences
+ Lower latency, horizontal API scaling
- New failure domain; need Redis HA (cluster mode)
```

---

## 3. Microservices & Cloud-Native AWS

### Service Boundary Rule

Split by **business capability** (bounded context), not technical layer.

```
❌ user-db-service, user-api-service (layer split)
✅ identity-service, billing-service, catalog-service (domain split)
```

### AWS Service Map

| Need | Service |
|------|---------|
| Containers | ECS, EKS |
| Serverless | Lambda |
| API edge | API Gateway |
| Load balancing | ALB, NLB |
| Relational | RDS, Aurora |
| Document/NoSQL | DynamoDB |
| Cache | ElastiCache (Redis) |
| Search | OpenSearch |
| Queue | SQS |
| Pub/Sub | SNS, EventBridge |
| Object store | S3 |
| Secrets | Secrets Manager |

### Sample: Node.js health check behind ALB

```javascript
// health.js — readiness must check dependencies
const express = require('express');
const { MongoClient } = require('mongodb');

const app = express();
let db;

app.get('/health/live', (_, res) => res.json({ status: 'ok' }));

app.get('/health/ready', async (_, res) => {
  try {
    await db.admin().ping();
    res.json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ status: 'not_ready', error: err.message });
  }
});

async function start() {
  const client = new MongoClient(process.env.MONGO_URI);
  await client.connect();
  db = client.db('orders');
  app.listen(3000);
}

start();
```

### Microservices Anti-Patterns to Avoid

- Shared database across services (creates hidden coupling — schema changes break multiple services)
- Synchronous call chains 5+ deep (cascading latency and failure)
- Distributed monolith (tight coupling via sync calls — worst of both worlds)
- Nano-services (one entity per service — operational overhead exceeds benefit)
- Chatty services (50 API calls to render one page — use BFF pattern or GraphQL)

#### Q: How do you define service boundaries in a microservices architecture?

> **Detailed Answer:** I define service boundaries using **Domain-Driven Design (DDD) bounded contexts**, not technical layers. The question is not "should we have a database service and an API service?" but rather "what business capabilities does our system provide, and which ones change together?"
>
> **Step 1 — Event Storming or domain workshop:** Gather product owners, engineers, and domain experts. Map business events (`OrderPlaced`, `PaymentReceived`, `InventoryReserved`) and identify which entities and rules cluster together. Each cluster is a candidate bounded context.
>
> **Step 2 — Apply the "change together" test:** If a requirement change in the billing domain requires modifying the catalog service, your boundaries are wrong. Services should be independently deployable — a team should be able to ship a billing feature without coordinating a catalog deployment.
>
> **Step 3 — Data ownership:** Each service owns its data exclusively. No shared tables. If the order service needs product name, it either calls the catalog API, caches the name locally, or receives it via an event. Never join across service databases.
>
> **Step 4 — Team alignment (Conway's Law):** Service boundaries should align with team boundaries. A team of 6–8 engineers should own 1–3 services, not 15. If two teams constantly need to coordinate releases, consider merging the services or splitting the team.
>
> **Real example:** In an e-commerce platform, I separated `Identity` (users, auth, roles), `Catalog` (products, categories, pricing rules), `Order` (cart, checkout, order history), `Inventory` (stock levels, reservations), and `Fulfillment` (shipping, tracking). The temptation was to put pricing in Catalog and Order — we kept it in Catalog because pricing rules change with merchandising, not with order processing. Order service receives the price at checkout time and stores it as a snapshot on the order line item.

#### Q: Walk me through designing a microservice on AWS from scratch.

> **Detailed Answer:** Let me walk through designing an `order-service` on AWS for a mid-size e-commerce platform handling 2,000 orders/minute at peak.
>
> **Compute:** EKS (Kubernetes) for container orchestration. Three pods minimum for HA, HPA scaling on CPU (target 60%) and custom metric (SQS queue depth). Each pod runs a Node.js Express app in a Docker container stored in ECR. Why EKS over ECS? Team already had K8s expertise and needed custom metrics-based scaling. Why not Lambda? Order processing involves long-running Temporal workflows (30+ seconds), which exceeds Lambda's comfortable execution window.
>
> **API Layer:** ALB routes `/api/orders/*` to the order-service target group. API Gateway sits in front for external clients — handles JWT validation, rate limiting (1,000 req/min per API key), and request/response transformation. Internal service-to-service calls go directly through ALB with mTLS via service mesh (Istio).
>
> **Data:** MongoDB Atlas on AWS (M10 cluster, 3-node replica set across 3 AZs). Order documents are denormalized — each order embeds line items, shipping address snapshot, and payment status. Separate `order-events` collection for audit trail. Why MongoDB? Order schema evolves frequently (new fields for marketplace sellers, subscription orders, gift wrapping) and we don't need cross-order joins.
>
> **Cache:** ElastiCache Redis cluster mode (3 shards, 1 replica each). Used for: cart state (TTL 24h), idempotency keys (TTL 48h), rate limit counters. Cache-aside pattern with stampede protection.
>
> **Messaging:** SQS standard queue for `order.created` events consumed by search indexer, email service, and analytics pipeline. SNS fan-out for notifications. EventBridge for event routing with filtering rules.
>
> **Workflow:** Temporal Cloud for checkout saga orchestration. Activities call inventory-service, payment-service, and fulfillment-service with individual retry policies.
>
> **Observability:** Datadog APM for distributed tracing, CloudWatch for infrastructure metrics, structured JSON logs shipped to Datadog via Fluent Bit DaemonSet. RED metrics dashboard per service. SLO: 99.9% of order creation requests complete in < 2 seconds.
>
> **Security:** IAM roles for service accounts (IRSA) — pods assume IAM roles, no static credentials. Secrets in AWS Secrets Manager, injected via External Secrets Operator. VPC with private subnets for pods, NAT gateway for outbound only. Security groups restrict MongoDB access to order-service pods only.
>
> **CI/CD:** Jenkins pipeline — build → unit tests → integration tests (Testcontainers) → Docker build → push to ECR → deploy to staging → smoke tests → manual approval → rolling deploy to production with readiness probes.

---

## 4. Distributed Systems

### Core Concepts

| Concept | Purpose |
|---------|---------|
| Load balancer | Distribute traffic |
| Sharding | Partition data by key (user_id, tenant_id) |
| Replication | HA + read scaling |
| Idempotency | Safe retries |
| Backpressure | Protect system under overload |

### Idempotency Key Example (Java + Spring)

```java
// PaymentService.java
@Service
public class PaymentService {
  private final PaymentRepository repo;

  public PaymentResult charge(ChargeRequest req) {
    Optional<Payment> existing = repo.findByIdempotencyKey(req.idempotencyKey());
    if (existing.isPresent()) {
      return PaymentResult.from(existing.get()); // safe retry
  }
    Payment payment = repo.save(new Payment(req));
    return gateway.charge(payment);
  }
}
```

### Rate Limiter — Token Bucket (Node.js)

```javascript
class TokenBucket {
  constructor(capacity, refillPerSec) {
    this.tokens = capacity;
    this.capacity = capacity;
    this.refillPerSec = refillPerSec;
    this.lastRefill = Date.now();
  }

  refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsed * this.refillPerSec);
    this.lastRefill = now;
  }

  allow() {
    this.refill();
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true;
    }
    return false;
  }
}

// Usage: 100 requests/sec per API key
const limiter = new TokenBucket(100, 100);
```

### Failure Handling Checklist

1. Set timeouts on every outbound call (default: 3s for internal, 10s for external)
2. Retry with exponential backoff + jitter (base 100ms, max 30s, max 3 retries)
3. Circuit breaker after N failures (threshold: 5 failures in 30s, half-open after 60s)
4. Bulkhead: isolate thread pools per dependency (don't let a slow payment service starve inventory calls)
5. Graceful degradation (serve cached/stale data for non-critical reads)
6. Dead letter queues for messages that fail after max retries
7. Health checks that verify dependencies (not just "I'm alive")

#### Q: How do you ensure exactly-once processing in a distributed system?

> **Detailed Answer:** True exactly-once semantics across a distributed system is **theoretically impossible** without coordination overhead that kills performance. What we actually implement is **effectively-once processing** — at-least-once delivery combined with idempotent consumers.
>
> **The problem:** A message broker (SQS, Kafka) delivers a message. Your consumer processes it and crashes before acknowledging. The broker redelivers. Without idempotency, you process the same payment twice.
>
> **Solution — three layers:**
>
> **Layer 1 — Idempotency keys:** Every mutating API call carries an `Idempotency-Key` header (UUID generated by the client). The server stores the key + response in a database or Redis with a TTL (48 hours). On duplicate key, return the stored response without re-executing. This is how Stripe, AWS, and most payment APIs work.
>
> **Layer 2 — Idempotent consumers:** Before processing a message, check if its `messageId` or business key (e.g., `orderId + eventType`) has already been processed. Store processed IDs in a deduplication table. In Kafka, use compacted topics or transactional producers with `read_committed` isolation.
>
> **Layer 3 — Database constraints:** Use unique constraints as a last line of defense. `UNIQUE(order_id, event_type)` on an events table prevents duplicate event processing even if layers 1 and 2 fail.
>
> **Real example:** Our payment service received duplicate SQS messages during an AZ failover. Without idempotency keys, we would have double-charged 340 customers. With idempotency keys stored in Redis, all 340 duplicates were caught and returned the original payment response. The customer saw one charge; our reconciliation matched perfectly.

#### Q: Explain consistent hashing and when you'd use it.

> **Detailed Answer:** Consistent hashing solves the problem of **distributing data across a cluster of nodes** such that adding or removing a node only moves a fraction of the data, not everything.
>
> **The naive approach** — `hash(key) % N` — fails when N changes. If you have 10 shards and add an 11th, nearly 100% of keys need remapping because the modulo result changes for almost every key. This causes a massive cache miss storm or data migration.
>
> **How consistent hashing works:** Imagine a circle (ring) with positions 0 to 2^32. Each node is hashed to a position on the ring (e.g., Node A at position 100, Node B at 500, Node C at 900). Each key is also hashed to a position. The key is assigned to the **first node clockwise** from its position. Key at position 150 → Node B. Key at position 950 → Node A (wraps around).
>
> When you **add Node D** at position 400, only keys between Node A (100) and Node D (400) move — roughly 1/N of the data. The rest stay put. When you **remove a node**, its keys redistribute to the next node clockwise.
>
> **Virtual nodes:** Production systems place multiple virtual nodes per physical node on the ring (e.g., 150 virtual nodes per server). This ensures even distribution even when the number of physical nodes is small.
>
> **Where I use it:** Redis Cluster uses hash slots (16,384 slots) which is a form of consistent hashing. Cassandra's partitioner distributes rows across nodes. CDN edge routing. Distributed caches (Memcached with consistent hashing in client libraries). Load balancers for sticky session routing.
>
> **When NOT to use it:** If your shard count is fixed and never changes, simple modulo is fine. If you need range queries across shards (e.g., "all orders from January"), hash-based sharding breaks range queries — use range-based or directory-based sharding instead.

---

## 5. Event-Driven Architecture (RabbitMQ & Kafka)

### What Is Event-Driven Architecture?

In **event-driven architecture (EDA)**, services communicate by **producing and consuming events** instead of calling each other synchronously. An event is a fact that already happened — `OrderPlaced`, `PaymentCharged`, `InventoryReserved` — not a command asking another service to do something.

```
Synchronous (request/response):
  Order Service ──HTTP──► Inventory Service ──HTTP──► Payment Service
  (caller waits; failures cascade; tight coupling)

Event-driven (async):
  Order Service ──publishes──► Broker ──consumes──► Inventory Service
                              │
                              └──consumes──► Payment Service
                              └──consumes──► Notification Service
  (producer doesn't wait; consumers scale independently; loose coupling)
```

### Core Concepts

| Concept | Meaning |
|---------|---------|
| **Producer / Publisher** | Service that emits events after a state change |
| **Consumer / Subscriber** | Service that reacts to events |
| **Broker / Bus** | Middleware that stores and routes events (RabbitMQ, Kafka, SQS, SNS) |
| **Event** | Immutable record of something that happened (past tense) |
| **Command** | Request to do something (imperative) — different from an event |
| **Topic / Exchange / Queue** | Routing and storage constructs (names vary by broker) |
| **Consumer group** | Set of consumers that share work for the same event stream |
| **At-least-once delivery** | Events may be delivered more than once → consumers must be **idempotent** |
| **Ordering** | Guarantees about sequence (per-key, per-queue, or none) |
| **Dead Letter Queue (DLQ)** | Where poison messages go after max retries |

### Why Architects Choose EDA

| Benefit | Explanation |
|---------|-------------|
| **Loose coupling** | Order service doesn't know who listens — add Notification later without changing Order |
| **Independent scaling** | Spike in email volume? Scale email consumers only |
| **Resilience** | If payment is down, events queue up; when it recovers, it catches up |
| **Auditability** | Event log is a natural audit trail of what happened |
| **Fan-out** | One event → many consumers (search index, analytics, email, fraud) |

| Cost / Risk | Mitigation |
|-------------|------------|
| Eventual consistency | Design UI and SLAs for lag; use read-your-writes where needed |
| Harder debugging | Correlation IDs, distributed traces, event schemas |
| Duplicate processing | Idempotency keys + dedup tables |
| Schema evolution | Version events; use Avro/Protobuf + schema registry |
| Operational complexity | Start with managed brokers (MSK, CloudAMQP, Amazon MQ) |

### Event Styles: Notification vs Event-Carried State Transfer

| Style | Payload | Use When |
|-------|---------|----------|
| **Thin notification** | `{ "orderId": "123", "type": "OrderPlaced" }` | Consumer should fetch latest state; avoid large payloads |
| **Event-carried state** | Full order snapshot in the event | Consumer shouldn't call back; offline processing; audit |
| **Domain event** | Business meaning + key fields | Cross-service choreography (saga) |
| **Integration event** | Stable contract for external systems | Public APIs, partner integrations |

---

### RabbitMQ vs Kafka — Architect Decision Table

| Dimension | RabbitMQ | Kafka |
|-----------|----------|-------|
| **Model** | Message broker (queues + exchanges) | Distributed log (topics + partitions) |
| **Message lifetime** | Deleted after consumer ACKs | Retained for hours/days/weeks (configurable) |
| **Consumption** | Competing consumers pull from a queue | Consumer groups read from partitions; offset-based |
| **Replay** | Hard — message is gone after ACK | Easy — reset offset and re-read history |
| **Ordering** | Per-queue (or priority queues) | Per-partition (key → same partition) |
| **Throughput** | High (tens of thousands msg/sec) | Very high (millions msg/sec with partitions) |
| **Latency** | Very low (milliseconds) | Low (ms), optimized for throughput |
| **Routing** | Rich (direct, topic, fanout, headers exchanges) | Topic + partition key; filters in consumers |
| **Best for** | Task queues, RPC-style work, complex routing, classic microservices messaging | Event streaming, analytics, CDC, replay, many independent consumers |
| **Ops complexity** | Moderate | Higher (ZooKeeper/KRaft, partitions, rebalancing) |

**Interview one-liner:** *"RabbitMQ is a smart broker that routes and forgets after delivery; Kafka is a dumb durable log that remembers so many consumers can read and replay independently."*

---

### Example 1: Order Fulfillment with RabbitMQ

**Business scenario:** E-commerce checkout. When an order is placed, inventory must reserve stock, payment must charge the card, and email must notify the customer. Services should not call each other over HTTP in a chain.

#### Topology

```
Order Service
    │ publishes OrderPlaced
    ▼
┌─────────────────────────────────────────────────────┐
│  RabbitMQ                                            │
│  Exchange: orders.events  (type: topic)              │
│       │                                              │
│       ├── routing key order.placed                   │
│       │      ├── Queue: inventory.reserve            │
│       │      ├── Queue: payment.charge               │
│       │      └── Queue: email.notify                 │
│       │                                              │
│       └── DLX: orders.dlx → Queue: orders.dead       │
└─────────────────────────────────────────────────────┘
    │                    │                    │
    ▼                    ▼                    ▼
Inventory Service   Payment Service    Email Service
(ACK after reserve) (ACK after charge) (ACK after send)
```

#### Why RabbitMQ here?

- Classic **work distribution** — each message should be processed once by one worker of each type
- **Complex routing** — topic exchange can route `order.placed`, `order.cancelled`, `order.refunded` to different queues
- **Low latency** task handoff between services
- Messages don't need long retention once processed — ACK deletes them
- Dead-letter exchange handles poison messages cleanly

#### Event Payload

```json
{
  "eventId": "evt_7f3a2c",
  "eventType": "OrderPlaced",
  "occurredAt": "2024-06-15T10:30:00Z",
  "correlationId": "req_abc123",
  "data": {
    "orderId": "ORD-2024-001234",
    "userId": "u123",
    "items": [
      { "productId": "p456", "qty": 1, "price": 79.99 }
    ],
    "total": 79.99,
    "currency": "USD"
  }
}
```

#### Node.js Producer (amqplib)

```javascript
const amqp = require('amqplib');

async function publishOrderPlaced(order) {
  const conn = await amqp.connect(process.env.RABBITMQ_URL);
  const ch = await conn.createChannel();

  await ch.assertExchange('orders.events', 'topic', { durable: true });

  const event = {
    eventId: crypto.randomUUID(),
    eventType: 'OrderPlaced',
    occurredAt: new Date().toISOString(),
    correlationId: order.correlationId,
    data: {
      orderId: order.id,
      userId: order.userId,
      items: order.items,
      total: order.total,
      currency: order.currency
    }
  };

  ch.publish(
    'orders.events',
    'order.placed',                          // routing key
    Buffer.from(JSON.stringify(event)),
    { persistent: true, contentType: 'application/json', messageId: event.eventId }
  );

  await ch.close();
  await conn.close();
}
```

#### Node.js Consumer — Inventory (with ACK, retry, DLQ)

```javascript
async function startInventoryConsumer() {
  const conn = await amqp.connect(process.env.RABBITMQ_URL);
  const ch = await conn.createChannel();

  await ch.assertExchange('orders.events', 'topic', { durable: true });
  await ch.assertExchange('orders.dlx', 'fanout', { durable: true });

  await ch.assertQueue('inventory.reserve', {
    durable: true,
    arguments: {
      'x-dead-letter-exchange': 'orders.dlx',
      'x-message-ttl': 60000          // optional: expire stuck messages
    }
  });
  await ch.bindQueue('inventory.reserve', 'orders.events', 'order.placed');
  await ch.assertQueue('orders.dead', { durable: true });
  await ch.bindQueue('orders.dead', 'orders.dlx', '');

  ch.prefetch(10); // fair dispatch — don't overwhelm one worker

  ch.consume('inventory.reserve', async (msg) => {
    if (!msg) return;
    try {
      const event = JSON.parse(msg.content.toString());
      await reserveStock(event.data);          // idempotent by orderId
      ch.ack(msg);                             // remove from queue
    } catch (err) {
      // nack without requeue → goes to DLX after broker rules / max retries
      console.error('inventory failed', err);
      ch.nack(msg, false, false);
    }
  });
}
```

#### Failure Handling with RabbitMQ

| Failure | Behavior |
|---------|----------|
| Consumer crash mid-processing | Message not ACK'd → redelivered to another consumer |
| Poison message (always fails) | After max retries / nack → DLQ `orders.dead` → alert + manual inspect |
| Payment service down | Messages accumulate in `payment.charge` queue → catch up when healthy |
| Duplicate delivery | `reserveStock` checks `reservations` table by `orderId` (unique constraint) |

#### Q: Explain event-driven architecture using RabbitMQ with a real example.

> **Detailed Answer:** Event-driven architecture means services react to facts that already happened instead of calling each other synchronously. In our e-commerce platform, when checkout succeeds, the Order Service does not HTTP-call Inventory, Payment, and Email. It publishes an `OrderPlaced` event to a RabbitMQ topic exchange called `orders.events` with routing key `order.placed`.
>
> RabbitMQ then **fans the same event out** to three durable queues: `inventory.reserve`, `payment.charge`, and `email.notify`. Each queue has its own competing consumers. Inventory workers reserve stock; Payment workers charge Stripe; Email workers send confirmation. The Order Service does not wait — it returns `202 Accepted` (or confirms after local DB commit + outbox publish) and continues.
>
> **Why RabbitMQ fits this use case:** We needed reliable **task distribution** with rich routing (`order.placed` vs `order.cancelled`), low latency, and dead-lettering for poison messages. We did **not** need weeks of message retention or replay for analytics — once ACK'd, the work is done. RabbitMQ's exchange/queue model maps cleanly to "do this work once."
>
> **Consistency model:** This is eventual consistency. Inventory might reserve 200ms after the order is saved. The UI shows "Processing" until we receive `InventoryReserved` and `PaymentCharged` events back (or a Temporal orchestrator drives the saga). Every consumer is **idempotent** because RabbitMQ provides at-least-once delivery — a crash before ACK redelivers the message.
>
> **Outbox pattern:** To avoid dual-write bugs (DB commit succeeds, publish fails), we write the order and an outbox row in one MongoDB transaction. A poller publishes to RabbitMQ and marks the outbox processed. That way we never lose an `OrderPlaced` event.
>
> **Outcome:** Checkout API p95 dropped because it no longer waited on payment + email. Email spikes no longer starved inventory workers. Failed payments went to a DLQ with PagerDuty alerts instead of blocking the HTTP request.

---

### Example 2: Product Catalog Streaming with Kafka

**Business scenario:** Product catalog changes must update OpenSearch (search), a recommendation engine, a cache warmer, and a data warehouse — independently, at high volume, with the ability to **replay** yesterday's events if a consumer bug is found.

#### Topology

```
Catalog Service / Debezium CDC
    │ produces to topic products.events
    ▼
┌──────────────────────────────────────────────────────────┐
│  Kafka Cluster                                            │
│  Topic: products.events                                   │
│    Partition 0 │ Partition 1 │ Partition 2 │ Partition 3  │
│    (key=productId → same product always same partition)   │
│    Retention: 7 days                                      │
└──────────────────────────────────────────────────────────┘
    │                 │                  │               │
    ▼                 ▼                  ▼               ▼
 Consumer Group    Consumer Group     Consumer Group   Consumer Group
 "search-indexer"  "recommendations"  "cache-warmer"   "warehouse-etl"
 (OpenSearch)      (ML features)      (Redis)          (S3/Redshift)
```

#### Why Kafka here?

- **Multiple independent consumers** each need the full stream — Kafka lets each consumer group track its own offset
- **Replay** — if search indexer had a bug, reset offset to yesterday and reprocess
- **High throughput** — 50K product updates/hour during catalog imports; partitions scale horizontally
- **Ordering per product** — partition key `productId` guarantees updates for one product are ordered
- **Retention** — keep 7 days of log for late-joining consumers and recovery
- CDC via Debezium can stream MongoDB/Postgres changes into Kafka as the source of truth log

#### Event Payload (Avro-friendly JSON shown)

```json
{
  "eventId": "evt_9b21",
  "eventType": "ProductUpdated",
  "eventVersion": 2,
  "occurredAt": "2024-06-15T11:00:00Z",
  "productId": "p456",
  "data": {
    "name": "Wireless Keyboard",
    "category": "electronics",
    "price": 79.99,
    "inStock": true,
    "updatedAt": "2024-06-15T11:00:00Z"
  }
}
```

#### Node.js Producer (kafkajs)

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'catalog-service',
  brokers: process.env.KAFKA_BROKERS.split(',')
});

const producer = kafka.producer({
  idempotent: true,              // broker-side dedup for retries
  maxInFlightRequests: 5,
  retry: { retries: 5 }
});

async function publishProductUpdated(product) {
  await producer.connect();
  await producer.send({
    topic: 'products.events',
    messages: [{
      key: product.id,           // CRITICAL: same product → same partition → ordered
      value: JSON.stringify({
        eventId: crypto.randomUUID(),
        eventType: 'ProductUpdated',
        eventVersion: 2,
        occurredAt: new Date().toISOString(),
        productId: product.id,
        data: {
          name: product.name,
          category: product.category,
          price: product.price,
          inStock: product.inStock,
          updatedAt: product.updatedAt
        }
      }),
      headers: { 'correlation-id': product.correlationId }
    }]
  });
}
```

#### Node.js Consumer — Search Indexer

```javascript
const consumer = kafka.consumer({
  groupId: 'search-indexer',     // independent from other groups
  sessionTimeout: 30000
});

async function startSearchIndexer() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'products.events', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value.toString());

      // Idempotent upsert by productId
      if (event.eventType === 'ProductUpdated' || event.eventType === 'ProductCreated') {
        await esClient.index({
          index: 'products',
          id: event.productId,
          document: event.data
        });
      } else if (event.eventType === 'ProductDeleted') {
        await esClient.delete({ index: 'products', id: event.productId });
      }

      // Offset commits automatically (or use manual commit for stricter control)
    }
  });
}
```

#### Kafka Partitioning & Ordering Mental Model

```
productId "p456" → hash → Partition 2
productId "p789" → hash → Partition 0

Within Partition 2: p456 update#1 → update#2 → update#3  (strict order)
Across partitions: NO global order guarantee

Rule: Put the entity ID in the message key whenever you need per-entity ordering.
```

#### Failure Handling with Kafka

| Failure | Behavior |
|---------|----------|
| Consumer crash | Another member of the group takes partitions (rebalance); resumes from last committed offset |
| Processing bug shipped | Fix code → reset consumer group offset → **replay** last N hours |
| Poison message | Skip + write to DLQ topic `products.events.dlq`; don't block the partition |
| Slow consumer | Lag metric rises (`records-lag`); scale consumers up to #partitions |
| Duplicate on retry | Idempotent upsert by `productId`; store `eventId` for dedup if needed |

#### Q: Explain event-driven architecture using Kafka with a real example.

> **Detailed Answer:** With Kafka, event-driven architecture looks more like a **durable commit log** than a task queue. In our catalog platform, every product create/update/delete was published to the `products.events` topic. The message key was `productId` so all updates for one product landed on the same partition and stayed ordered.
>
> Four independent consumer groups read the same topic: **search-indexer** wrote to OpenSearch, **recommendations** updated ML feature stores, **cache-warmer** refreshed Redis, and **warehouse-etl** wrote Parquet to S3 for Airflow. Kafka's key property here is that **each group has its own offset** — the search indexer can be at offset 10M while warehouse is at 9.8M, and neither blocks the other. RabbitMQ would require a separate queue copy per consumer type and would delete messages after ACK, making replay painful.
>
> **Why Kafka over RabbitMQ for this case:** We needed (1) multiple fan-out consumers on the same stream, (2) 7-day retention for replay after bugs, (3) high throughput during nightly catalog imports, and (4) CDC from MongoDB via Debezium as an alternative producer. Those are streaming/log use cases, not classic work queues.
>
> **Real incident:** A search indexer mapping bug corrupted brand filters for 6 hours. We fixed the mapping, reset the `search-indexer` consumer group offset to `earliest` for that day, and reprocessed ~2M events in 40 minutes. OpenSearch recovered without touching Catalog Service. That replay capability is why we chose Kafka.
>
> **Trade-offs we accepted:** Higher operational complexity (partitions, rebalances, consumer lag monitoring), eventual consistency (search lag of 1–3 seconds under load), and the need for idempotent consumers because Kafka is at-least-once by default (exactly-once requires transactional APIs and careful design).
>
> **Interview contrast:** For "do this job once" (send email, charge payment), I still prefer RabbitMQ or SQS. For "many systems must react to a stream of facts and sometimes re-read history," I choose Kafka.

---

### Side-by-Side Flow: Same Business Event, Two Brokers

| Step | RabbitMQ (OrderPlaced) | Kafka (ProductUpdated) |
|------|------------------------|------------------------|
| 1. Produce | Publish to exchange with routing key | Produce to topic with key=`productId` |
| 2. Store | Message sits in bound queues | Append to partition log |
| 3. Consume | Competing consumers on each queue | Each consumer group reads independently |
| 4. Success | ACK → message deleted | Commit offset → message still retained |
| 5. New consumer added | Create queue + bind to exchange; only gets **new** messages | New consumer group can read from beginning (replay) |
| 6. Bug fix | Reprocess from DLQ / DB source | Reset offset and replay topic |

### Common EDA Patterns

| Pattern | Description | Typical Broker |
|---------|-------------|----------------|
| **Pub/Sub fan-out** | One event, many subscribers | RabbitMQ fanout/topic; Kafka multiple groups |
| **Work queue** | Compete for tasks, process once | RabbitMQ queue; Kafka single consumer group |
| **Event sourcing** | Store state as sequence of events | Kafka (log as source of truth) |
| **CQRS + events** | Writes emit events → rebuild read models | Kafka + OpenSearch/Redis |
| **CDC streaming** | DB changes → events | Debezium → Kafka |
| **Saga choreography** | Services react to each other's domain events | Either; Temporal often preferred for complex sagas |
| **Transactional outbox** | Write DB + outbox atomically; poller publishes | Works with both |

### Decision Framework (Say This in Interviews)

```
Need task distribution + complex routing + low latency?
  → RabbitMQ (or SQS + SNS on AWS)

Need event stream + multiple independent consumers + replay?
  → Kafka (or Kinesis / MSK on AWS)

Need simple managed queue, AWS-native?
  → SQS (point-to-point) + SNS (fan-out)

Need long-running business workflow with compensations?
  → Temporal (orchestration) — events alone get messy
```

### Pitfalls to Mention Proactively

1. **Dual write without outbox** — DB commit succeeds, publish fails → lost events
2. **Non-idempotent consumers** — duplicates cause double charges / double emails
3. **Chatty thin events without correlation IDs** — impossible to debug across services
4. **Unbounded queues / lag** — no alerts on queue depth or Kafka consumer lag
5. **Using Kafka as a database** without a real system of record for business entities
6. **Global ordering assumptions** — Kafka only orders per partition; RabbitMQ per queue

---

## 6. Workflow Orchestration (Airflow vs Temporal)

### Comparison

| | Airflow | Temporal |
|---|---------|----------|
| **Best for** | Batch ETL, scheduled DAGs | Long-running business workflows |
| **Duration** | Minutes to hours | Seconds to days/months |
| **State** | Task-level | Durable workflow state |
| **Human steps** | Awkward | Native (signals, timers) |
| **Failure recovery** | Re-run tasks | Automatic replay from history |

### When to Use What

```
Nightly sales report ETL        → Airflow
Order fulfillment (3-day flow)  → Temporal
Simple cron cleanup             → EventBridge / cron
```

### Sample: Temporal Workflow (TypeScript-style pseudocode)

```typescript
// orderWorkflow.ts
export async function orderWorkflow(orderId: string): Promise<void> {
  const inventory = await reserveInventory(orderId);  // activity with retry
  try {
    await chargePayment(orderId);
    await confirmOrder(orderId);
  } catch (err) {
    await releaseInventory(inventory);  // compensation
    throw err;
  }
}
```

### Sample: Airflow DAG (Python)

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def extract(): ...
def transform(): ...
def load(): ...

with DAG('daily_sales_etl', start_date=datetime(2024, 1, 1), schedule='0 2 * * *') as dag:
    t1 = PythonOperator(task_id='extract', python_callable=extract)
    t2 = PythonOperator(task_id='transform', python_callable=transform)
    t3 = PythonOperator(task_id='load', python_callable=load)
    t1 >> t2 >> t3
```

**Interview one-liner:** *"Airflow orchestrates data pipelines on a schedule; Temporal orchestrates durable business processes with compensations."*

#### Q: When would you choose Temporal over Airflow? Give a detailed comparison.

> **Detailed Answer:** The choice between Temporal and Airflow comes down to **what you're orchestrating** — data pipelines vs business processes.
>
> **Choose Airflow when:**
> - You have **scheduled batch jobs** — nightly ETL, weekly reports, hourly data syncs
> - Tasks are **idempotent batch operations** — extract from S3, transform with Spark, load into Redshift
> - Duration is **minutes to hours** — long-running but not days
> - You need **rich scheduling** — cron expressions, backfill, dependency graphs between DAGs
> - Your team is primarily **data engineers** comfortable with Python
> - Example: "Every night at 2 AM, pull sales data from 5 sources, join, aggregate, and load into the data warehouse"
>
> **Choose Temporal when:**
> - You have **long-running business workflows** — order fulfillment over 3 days, loan approval with human review, subscription billing cycles
> - You need **durable state** that survives process crashes — if your server restarts mid-workflow, Temporal replays from history and continues
> - You need **signals and queries** — a human approves a step via API call (signal), or you query workflow state in real-time
> - You need **compensating transactions** — saga pattern with automatic rollback on failure
> - Tasks have **complex retry logic** — different retry policies per activity, exponential backoff, non-retryable error types
> - Duration is **seconds to months** — Temporal handles both short and very long workflows
> - Example: "Customer places order → reserve inventory → charge payment → wait for warehouse pick (up to 48h) → ship → wait for delivery confirmation → send review request"
>
> **Key technical differences:**
> | Aspect | Airflow | Temporal |
> |--------|---------|----------|
> | Execution model | Scheduler triggers tasks | Event-driven, always-on workers |
> | State storage | Task instance in metadata DB | Full event history per workflow |
> | Failure recovery | Re-run failed task from beginning | Replay from last checkpoint |
> | Versioning | DAG versioning is painful | Built-in workflow versioning |
> | Human-in-the-loop | Hacky (sensors, reschedule) | Native signals and timers |
> | Observability | Airflow UI shows DAG runs | Temporal UI shows full workflow timeline |
>
> **Real decision I made:** We had both. Airflow ran our analytics pipeline (50 DAGs, nightly/hourly schedules, processing 2TB/day). Temporal ran our order fulfillment (7-step workflow, average duration 3 days, 15% of workflows needed human intervention for fraud review). Trying to do order fulfillment in Airflow would have required ugly sensor hacks and wouldn't survive worker restarts gracefully.

#### Q: How does Temporal guarantee workflow durability?

> **Detailed Answer:** Temporal achieves durability through an **event sourcing** model where every workflow action is recorded as an immutable event in a persistent event history.
>
> When a workflow starts, Temporal creates a workflow execution record. Every time the workflow does something — calls an activity, sets a timer, receives a signal — Temporal appends an event to the history. The workflow code itself is **deterministic** — given the same event history, replaying the code produces the same result.
>
> If the worker process crashes mid-workflow, Temporal assigns the workflow to another worker. The new worker **replays the entire event history** from the beginning, re-executing the workflow code. For events that already completed (activities that returned results), Temporal returns the cached result instead of re-executing. The workflow continues from where it left off, as if nothing happened.
>
> This means workflow code must be deterministic — no `Math.random()`, no `Date.now()` (use Temporal's `workflow.now()`), no threading. Non-deterministic work goes in **activities**, which are regular functions with full retry and timeout support.
>
> The Temporal Server (or Temporal Cloud) stores event histories in a durable database (Cassandra or PostgreSQL). This is the source of truth — workers are stateless and disposable. You can kill all workers and restart them; workflows resume automatically.

---

## 7. MongoDB

### When to Use

- Flexible/evolving schema (product catalogs, user profiles)
- Document-oriented access patterns
- Horizontal scale via sharding

### When NOT to Use

- Heavy multi-table joins
- Strict cross-document ACID as primary concern

### Indexing Example

```javascript
// Compound index for common query: orders by user, sorted by date
db.orders.createIndex({ userId: 1, createdAt: -1 });

// Query uses index (efficient)
db.orders.find({ userId: "u123" }).sort({ createdAt: -1 }).limit(20);

// Explain plan
db.orders.find({ userId: "u123" }).explain("executionStats");
```

### Node.js Repository Pattern

```javascript
class OrderRepository {
  constructor(db) {
    this.collection = db.collection('orders');
  }

  async findByUser(userId, { limit = 20, skip = 0 } = {}) {
    return this.collection
      .find({ userId })
      .sort({ createdAt: -1 })
      .skip(skip)
      .limit(limit)
      .toArray();
  }

  async create(order) {
    const doc = { ...order, createdAt: new Date(), status: 'PENDING' };
    const { insertedId } = await this.collection.insertOne(doc);
    return { ...doc, _id: insertedId };
  }
}
```

### Sharding Mental Model

```
Shard key: tenantId (good — isolates tenants, even distribution)
Shard key: createdAt (bad — hot shard on writes, all new data hits one shard)
Shard key: userId (good — high cardinality, spreads load)
Shard key: status (bad — low cardinality, only 3-5 values)
```

#### Q: How do you design a MongoDB schema for a high-traffic application?

> **Detailed Answer:** MongoDB schema design is fundamentally different from relational design. Instead of normalizing to 3NF, you **design for your query patterns** — embed related data that is read together, reference data that is large or shared.
>
> **Embedding vs Referencing decision tree:**
> - **Embed** when: data is read together (order + line items), one-to-few relationship (< 100 sub-documents), data doesn't change independently
> - **Reference** when: one-to-many with unbounded growth (user → millions of orders), data is shared across documents (product catalog referenced by many orders), sub-documents are large and rarely needed
>
> **Example — E-commerce order document (embedded):**
> ```javascript
> {
>   _id: ObjectId("..."),
>   orderId: "ORD-2024-001234",
>   userId: "u123",
>   status: "CONFIRMED",
>   lineItems: [
>     { productId: "p456", name: "Wireless Keyboard", qty: 1, price: 79.99 },
>     { productId: "p789", name: "USB-C Hub", qty: 2, price: 34.99 }
>   ],
>   shipping: { address: "123 Main St", city: "Austin", zip: "78701" },
>   payment: { method: "visa", last4: "4242", chargeId: "ch_abc" },
>   totals: { subtotal: 149.97, tax: 12.37, shipping: 5.99, total: 168.33 },
>   createdAt: ISODate("2024-06-15T10:30:00Z"),
>   updatedAt: ISODate("2024-06-15T10:30:05Z")
> }
> ```
> We embed line items because they are always read with the order and never queried independently. We snapshot `name` and `price` at order time so catalog price changes don't affect historical orders.
>
> **Indexing strategy:**
> - Compound index `{ userId: 1, createdAt: -1 }` for "my orders, newest first" — the most common query
> - Index `{ status: 1, createdAt: 1 }` for admin dashboard filtering by status
> - Index `{ "lineItems.productId": 1 }` for "find all orders containing product X"
> - **Avoid** indexing every field — each index slows writes and consumes RAM
>
> **Sharding:** Shard on `tenantId` for multi-tenant SaaS (isolates tenants, prevents noisy neighbor). Use hashed sharding for even distribution. Never shard on a monotonically increasing field like `_id` or `createdAt` without hashing — all writes hit the last chunk (hot shard).
>
> **Performance tips:** Use projection to return only needed fields (`{ lineItems: 1, status: 1 }`). Use `$lookup` sparingly — if you need joins frequently, your schema might need redesign. Set appropriate write concern (`w: "majority"`) for critical data. Use read preferences (`secondaryPreferred`) for analytics queries to offload the primary.

#### Q: How do you handle transactions in MongoDB across multiple documents?

> **Detailed Answer:** MongoDB supports multi-document ACID transactions since version 4.0 (replica sets) and 4.2 (sharded clusters). However, as an architect, I use them **judiciously** — they have performance overhead and can cause lock contention.
>
> **When to use MongoDB transactions:**
> - Transferring value between accounts (debit one, credit another)
> - Creating an order and decrementing inventory atomically
> - Any operation where partial completion leaves inconsistent state
>
> **When NOT to use transactions:**
> - Single-document updates (already atomic)
> - Operations that can be made idempotent and retried
> - High-throughput writes where eventual consistency is acceptable
> - Cross-collection operations that can be decoupled via events
>
> **Example — inventory reservation with transaction:**
> ```javascript
> const session = client.startSession();
> try {
>   await session.withTransaction(async () => {
>     const inventory = await db.collection('inventory').findOne(
>       { productId: 'p456', quantity: { $gte: 1 } },
>       { session }
>     );
>     if (!inventory) throw new Error('Insufficient stock');
>
>     await db.collection('inventory').updateOne(
>       { productId: 'p456' },
>       { $inc: { quantity: -1 }, $set: { updatedAt: new Date() } },
>       { session }
>     );
>
>     await db.collection('reservations').insertOne({
>       productId: 'p456', orderId: 'ORD-123', status: 'RESERVED',
>       createdAt: new Date()
>     }, { session });
>   });
> } finally {
>   await session.endSession();
> }
> ```
>
> **Alternative without transactions (preferred at scale):** Use atomic `findOneAndUpdate` with a condition: `db.inventory.findOneAndUpdate({ productId: 'p456', quantity: { $gte: 1 } }, { $inc: { quantity: -1 } })`. If it returns null, stock is insufficient. This is a single atomic operation — no transaction needed. For the reservation record, insert it after the atomic decrement; if the insert fails, a cleanup job releases the inventory. This is the **at-least-once with compensation** pattern, which scales better than multi-document transactions.

---

## 8. ElasticSearch / OpenSearch

### Role

**Secondary index for search** — not the system of record.

```
Postgres/MongoDB (source of truth) → event → indexer → OpenSearch
```

### Index Mapping Example

```json
PUT /products
{
  "mappings": {
    "properties": {
      "name": { "type": "text", "analyzer": "standard" },
      "category": { "type": "keyword" },
      "price": { "type": "float" },
      "createdAt": { "type": "date" }
    }
  }
}
```

### Search Query

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [{ "match": { "name": "wireless keyboard" } }],
      "filter": [{ "term": { "category": "electronics" } }]
    }
  },
  "sort": [{ "price": "asc" }],
  "size": 20
}
```

### Node.js Index Sync (simplified)

```javascript
async function indexProduct(product) {
  await esClient.index({
    index: 'products',
    id: product.id,
    document: {
      name: product.name,
      category: product.category,
      price: product.price,
      createdAt: product.createdAt
    }
  });
}

// Called from event consumer after product.created event
```

### Pitfalls

- Mapping changes on live index require reindex (plan mappings upfront; use index aliases for zero-downtime reindex)
- Deep aggregations on huge datasets are expensive (use pre-aggregated rollups or data warehouse)
- Keep transactional writes in primary DB (OpenSearch is not a database)
- Default 5 shards per index is often too many for small datasets (1 shard for < 50GB)
- `text` fields are analyzed (tokenized); use `keyword` for exact match, filtering, sorting

#### Q: How do you keep ElasticSearch/OpenSearch in sync with your primary database?

> **Detailed Answer:** OpenSearch is a **secondary index** — never the source of truth. The primary database (MongoDB, PostgreSQL) owns the data. Search index sync is an eventually consistent replication problem. There are three main patterns:
>
> **Pattern 1 — Change Data Capture (CDC):**
> Database writes are captured from the transaction log (MongoDB oplog, PostgreSQL WAL, Debezium connector) and streamed to a message queue (Kafka, Kinesis). A consumer reads events and updates OpenSearch. **Pros:** Near real-time (seconds), decoupled from application code, captures all changes including direct DB writes. **Cons:** Infrastructure complexity, need to handle schema evolution, ordering guarantees across partitions.
>
> **Pattern 2 — Application-level events (most common):**
> After a successful DB write, the application publishes an event (`product.created`, `product.updated`) to SQS/SNS/Kafka. A dedicated indexer service consumes events and updates OpenSearch. **Pros:** Simple, explicit, easy to test. **Cons:** If the event publish fails after DB write, search is stale. If DB write fails but event publishes, search has phantom data. Must handle both with outbox pattern.
>
> **Pattern 3 — Outbox pattern (recommended for consistency):**
> In the same database transaction, write the business data AND an outbox record. A separate poller reads the outbox table and publishes events. This guarantees the event is published if and only if the DB write succeeded. **Pros:** Reliable, no dual-write problem. **Cons:** Slight latency (polling interval), extra table to manage.
>
> **My implementation:** We used the outbox pattern with MongoDB. On product update, we wrote to `products` collection and `outbox` collection in a single transaction. A Node.js poller read outbox every 500ms, published to SQS, and marked records as processed. The indexer consumed from SQS and upserted to OpenSearch. End-to-end latency was 1–3 seconds. For deletes, we published `product.deleted` events and the indexer removed the document. We ran a nightly reconciliation job comparing DB count vs index count to catch any drift.
>
> **Reindexing strategy:** Use index aliases (`products` → `products_v2`). Build new index with updated mapping, bulk-reindex from DB, swap alias atomically, delete old index. Zero downtime for search users.

#### Q: Explain how full-text search works in ElasticSearch.

> **Detailed Answer:** When you index a document, ElasticSearch processes each `text` field through an **analyzer** — a pipeline of character filters, tokenizer, and token filters.
>
> **Indexing (write path):**
> 1. **Character filters:** Clean input — lowercase, remove HTML tags, normalize unicode
> 2. **Tokenizer:** Split text into tokens (words) — standard tokenizer splits on whitespace and punctuation
> 3. **Token filters:** Transform tokens — lowercase, stemming ("running" → "run"), stop words ("the", "a" removed), synonyms ("laptop" = "notebook")
> 4. Tokens are stored in an **inverted index** — a map from each token to the list of document IDs containing it
>
> **Searching (read path):**
> 1. Query text goes through the **same analyzer** (search analyzer, which can differ from index analyzer)
> 2. ElasticSearch looks up each query token in the inverted index
> 3. Scoring algorithms (BM25 by default) rank documents by relevance — term frequency, inverse document frequency, field length normalization
> 4. Results returned sorted by score
>
> **Query types I use:**
> - `match` — full-text search with analysis ("wireless keyboard" matches "Wireless Bluetooth Keyboard")
> - `term` — exact match on `keyword` fields (filter by category = "electronics")
> - `bool` — combine queries (`must`, `should`, `filter`, `must_not`)
> - `multi_match` — search across multiple fields with different boosts (`name^3`, `description^1`)
> - `fuzzy` — typo tolerance ("keybord" matches "keyboard")
> - `wildcard` / `prefix` — pattern matching (expensive, avoid on large indices)
>
> **Performance tuning:** Use `filter` context (not `must`) for exact matches — filters are cached and don't affect scoring. Limit `_source` fields returned. Use `search_after` for deep pagination instead of `from/size` (which breaks at 10,000+). Set `refresh_interval` to 30s for indexing-heavy workloads (default 1s is expensive).

---

## 9. Redis (Distributed Cache)

### Patterns

| Pattern | Flow |
|---------|------|
| Cache-aside | App reads cache → miss → DB → populate cache |
| Write-through | App writes cache + DB together |
| TTL | Auto-expire stale entries |

### Cache-Aside (Node.js)

```javascript
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

async function getProduct(id) {
  const cacheKey = `product:${id}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const product = await db.products.findById(id);
  if (product) {
    await redis.setex(cacheKey, 3600, JSON.stringify(product)); // 1h TTL
  }
  return product;
}
```

### Cache Stampede Protection

```javascript
async function getProductSafe(id) {
  const cacheKey = `product:${id}`;
  const lockKey = `lock:product:${id}`;

  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const locked = await redis.set(lockKey, '1', 'NX', 'EX', 10);
  if (!locked) {
    await sleep(50);
    return getProductSafe(id); // another thread is loading
  }

  try {
    const product = await db.products.findById(id);
    if (product) await redis.setex(cacheKey, 3600, JSON.stringify(product));
    return product;
  } finally {
    await redis.del(lockKey);
  }
}
```

### Redis Use Cases

| Use Case | Command/Pattern |
|----------|-----------------|
| Session store | `SET session:{id}` with TTL |
| Rate limiting | `INCR` + `EXPIRE` or sliding window |
| Leaderboard | Sorted sets (`ZADD`, `ZRANGE`) |
| Pub/Sub | Real-time notifications |
| Distributed lock | `SET key NX EX` (use carefully) |

### Failure Mode Answer

> *"If Redis is down, we bypass cache and hit the database with circuit breaker limits. We never serve silently wrong financial data from stale cache. For session data, users are redirected to re-authenticate. We monitor cache hit rate — a sudden drop to 0% triggers a P2 alert."*

#### Q: Explain cache-aside, write-through, and write-behind patterns in detail.

> **Detailed Answer:**
>
> **Cache-Aside (Lazy Loading) — most common:**
> 1. App checks cache for key
> 2. Cache miss → app reads from DB → app writes to cache → returns data
> 3. On update: app writes to DB → app **invalidates** cache (deletes key)
>
> **Pros:** Simple, cache only contains data actually requested, DB is always source of truth. **Cons:** Cache miss on first read (latency spike), possible stale data between DB write and cache invalidation, cache stampede on popular key expiry.
>
> **When I use it:** Product catalog, user profiles, configuration data — anything read-heavy with tolerable staleness.
>
> **Write-Through:**
> 1. App writes to cache AND DB synchronously on every write
> 2. On read: app checks cache → hit → return (DB never touched on read)
>
> **Pros:** Cache is always consistent with DB, no stale reads. **Cons:** Write latency includes cache write, cache may contain data never read (wasted memory), cache and DB can get out of sync if cache write fails after DB write.
>
> **When I use it:** Session data, user permissions — data that must be immediately consistent and is read frequently after write.
>
> **Write-Behind (Write-Back):**
> 1. App writes to cache immediately → returns success to client
> 2. Cache asynchronously flushes to DB in batches
>
> **Pros:** Extremely fast writes, batching reduces DB load. **Cons:** Data loss risk if cache crashes before flush, complexity in handling flush failures, not suitable for critical data.
>
> **When I use it:** Analytics counters, page view counts, metrics aggregation — data where losing a few seconds of writes is acceptable.
>
> **My decision framework:**
> | Data Type | Pattern | TTL | Invalidation |
> |-----------|---------|-----|--------------|
> | Product catalog | Cache-aside | 1 hour | On product.updated event |
> | User session | Write-through | 24 hours | On logout |
> | API rate limits | Write-behind (Redis INCR) | 1 minute | Auto-expire |
> | Financial balance | No cache (read DB directly) | — | — |
>
> **Cache stampede protection:** When a popular cache key expires, thousands of requests simultaneously hit the DB. Solutions: (1) **Mutex lock** — only one request loads from DB, others wait (shown in code above). (2) **Probabilistic early expiration** — refresh cache before TTL expires with probability increasing as expiry approaches. (3) **Never expire** — background job refreshes cache periodically; stale data served until refresh completes.

#### Q: How do you design a distributed rate limiter using Redis?

> **Detailed Answer:** Rate limiting protects your API from abuse and ensures fair resource allocation. In a distributed system with multiple API instances, rate limiting state must be shared — Redis is the standard choice.
>
> **Algorithm 1 — Fixed Window Counter:**
> ```
> Key: ratelimit:{apiKey}:{minute}
> INCR key → if count > limit, reject → EXPIRE key 60
> ```
> Simple but has a **burst problem** at window boundaries — a client can send 100 requests at 0:59 and 100 at 1:00 (200 in 2 seconds).
>
> **Algorithm 2 — Sliding Window Log:**
> Store timestamps of each request in a Redis sorted set. Remove entries older than the window. Count remaining entries. If count > limit, reject.
> ```
> ZADD ratelimit:{apiKey} {timestamp} {requestId}
> ZREMRANGEBYSCORE ratelimit:{apiKey} 0 {now - windowMs}
> ZCARD ratelimit:{apiKey} → if > limit, reject
> EXPIRE ratelimit:{apiKey} {windowSeconds}
> ```
> Accurate but memory-intensive — stores every request timestamp.
>
> **Algorithm 3 — Sliding Window Counter (recommended):**
> Combines fixed windows with weighted average of current and previous window. Memory-efficient and smooth.
> ```
> current_window = floor(now / windowSize)
> previous_count = GET ratelimit:{apiKey}:{current_window - 1}
> current_count = INCR ratelimit:{apiKey}:{current_window}
> weight = (now % windowSize) / windowSize
> estimated = previous_count * (1 - weight) + current_count
> if estimated > limit, reject
> ```
>
> **Algorithm 4 — Token Bucket (shown in code above):**
> Best for allowing bursts while maintaining average rate. Each API key gets a bucket that refills at a constant rate. Requests consume tokens. Empty bucket = rejected.
>
> **Production considerations:**
> - Return `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers
> - Use Redis Cluster for HA — rate limiter downtime means either blocking all traffic (safe) or allowing all traffic (dangerous)
> - Different limits per tier: free (100/min), pro (1000/min), enterprise (10000/min)
> - Implement at API Gateway level (AWS API Gateway throttling) AND application level for defense in depth
> - Log rate limit violations for abuse detection

---

## 10. Algorithms & Data Structures

### Architect-Level Choices

| Problem | Structure | Why |
|---------|-----------|-----|
| Fast lookup | Hash map | O(1) average |
| Top-K items | Min-heap | O(n log k) |
| Relationships | Graph + BFS/DFS | Traversal, shortest path |
| Prefix search | Trie | Autocomplete |
| Shard routing | Consistent hashing | Minimal resharding |
| Dedup at scale | Bloom filter | Space-efficient |
| LRU cache | Hash map + doubly linked list | O(1) get/put |

### Consistent Hashing (simplified)

```javascript
function getShard(userId, shardCount) {
  const hash = crc32(userId) % shardCount;
  return `shard-${hash}`;
}
// Production: use consistent hashing ring for adding/removing nodes
```

### LRU Cache (Java)

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
  private final int capacity;

  LRUCache(int capacity) {
    super(capacity, 0.75f, true); // access-order
    this.capacity = capacity;
  }

  @Override
  protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return size() > capacity;
  }
}
```

### Complexity Quick Reference

```
O(1)       — hash lookup, array index
O(log n)   — binary search, balanced tree, B-tree index lookup
O(n)       — linear scan
O(n log n) — efficient sort (merge sort, quick sort average)
O(n²)      — nested loops, naive sorting
O(2^n)     — recursive subsets (exponential — avoid at scale)
```

#### Q: As a technical architect, when do you think about algorithms and data structures?

> **Detailed Answer:** As an architect, I don't implement algorithms daily, but I make **structural decisions** where algorithmic complexity determines system behavior at scale.
>
> **Scenario 1 — Shard routing:** We needed to distribute 100M users across 16 database shards. Naive `hash(userId) % 16` works until we add a 17th shard (nearly all keys remap). I chose consistent hashing with virtual nodes — adding a shard only moves ~1/17 of data. This decision is made once at architecture time but affects every request forever.
>
> **Scenario 2 — Rate limiting at scale:** We needed to check rate limits for 50K requests/second. A naive approach (store every request timestamp in a list, count entries in window) is O(n) per check and uses massive memory. I chose a sliding window counter in Redis — O(1) per check, fixed memory per API key. At 50K RPS, the difference between O(1) and O(n) is the difference between a $200/month Redis cluster and a system that can't keep up.
>
> **Scenario 3 — Deduplication in event pipeline:** Our Kafka consumer needed to deduplicate 1M events/hour. Storing every event ID in a hash set would use ~500MB RAM. I used a Bloom filter with 1% false positive rate — 10MB RAM, O(1) lookup. The 1% false positive means we occasionally skip a genuinely new event, which is acceptable because our idempotency layer catches it downstream.
>
> **Scenario 4 — Top-K trending products:** "Show top 10 trending products in the last hour" from 10M product views. Sorting all products is O(n log n). A min-heap of size 10 gives O(n log 10) = O(n) — process each view, if heap size > 10, pop smallest. At 10M views/hour, this runs in a Flink window in milliseconds.
>
> **Key principle:** I don't need to implement a B-tree, but I need to know that database indexes are B-trees (O(log n) lookup), so a query without an index is O(n) table scan, and at 50M rows that's the difference between 3ms and 30 seconds.

---

## 11. Node.js

### Event Loop (detailed)

```
   ┌───────────────────────────┐
   │        timers             │  setTimeout, setInterval callbacks
   ├───────────────────────────┤
   │   pending callbacks       │  I/O callbacks deferred to next iteration
   ├───────────────────────────┤
   │        idle, prepare      │  internal use only
   ├───────────────────────────┤
   │         poll              │  retrieve new I/O events; execute I/O callbacks
   ├───────────────────────────┤
   │         check             │  setImmediate callbacks
   ├───────────────────────────┤
   │    close callbacks        │  socket.on('close', ...)
   └───────────────────────────┘
```

- **Good for:** I/O-bound APIs, real-time (WebSockets), high concurrency with low memory per connection
- **Bad for:** CPU-heavy work on main thread (blocks the entire event loop)
- **libuv thread pool:** File system operations, DNS lookups, and some crypto run on a thread pool (default 4 threads, configurable via `UV_THREADPOOL_SIZE`)

#### Q: Explain the Node.js event loop in detail. What happens when you `await`?

> **Detailed Answer:** Node.js is single-threaded for JavaScript execution but uses an event loop (powered by libuv) to handle concurrent I/O without blocking.
>
> When your code calls `await fetch('https://api.example.com')`, here's what happens:
> 1. `fetch` initiates a network request and registers a callback with the OS (epoll/kqueue/IOCP)
> 2. The `await` suspends the current async function — control returns to the event loop
> 3. The event loop processes other callbacks (other requests, timers, etc.)
> 4. When the OS signals that the network response is ready, libuv places the callback in the poll phase queue
> 5. The event loop executes the callback, which resolves the Promise
> 6. The async function resumes from where it `await`ed
>
> **Critical insight:** `await` does NOT block the thread. It yields control back to the event loop. This is why Node.js can handle 10,000+ concurrent connections on a single core — it's not doing 10,000 things simultaneously, it's switching between them whenever one is waiting for I/O.
>
> **What blocks the event loop:** Synchronous CPU work — JSON.parse of a 50MB payload, bcrypt hashing, complex regex on large strings, tight loops. During this time, NO other requests are processed. **Symptoms:** All API latency spikes, health checks fail, Kubernetes kills the pod.
>
> **Solutions for CPU-bound work:**
> - **Worker Threads** — spawn a separate V8 isolate for CPU work (shown in code above)
> - **Child processes** — for completely isolated tasks (image processing, PDF generation)
> - **Offload to a service** — send CPU work to a Lambda, a Java service, or a dedicated processing queue
> - **Break up work** — process in chunks with `setImmediate` between chunks to yield to the event loop
>
> **Microtask queue vs macrotask queue:** Promises (`.then`, `await`) execute in the microtask queue, which is drained completely before the event loop moves to the next phase. This means a chain of `await` calls executes before any `setTimeout` callback, even if the timer has expired.

#### Q: How do you handle errors in a production Node.js microservice?

> **Detailed Answer:** Error handling in Node.js requires discipline at multiple layers because unhandled errors can crash the process.
>
> **Layer 1 — Async route handlers:** Always wrap in try/catch or use an async error wrapper. Never let a rejected Promise go unhandled in an Express route.
>
> **Layer 2 — Global error middleware:** Centralized error handler that logs structured errors, maps internal errors to safe HTTP responses (never leak stack traces to clients), and reports to error tracking (Sentry, Datadog).
>
> **Layer 3 — Process-level handlers:**
> ```javascript
> process.on('unhandledRejection', (reason, promise) => {
>   logger.error({ reason, type: 'unhandledRejection' }, 'Unhandled promise rejection');
>   // In production: report and gracefully shutdown, don't crash immediately
> });
> process.on('uncaughtException', (err) => {
>   logger.fatal({ err }, 'Uncaught exception — shutting down');
>   // Uncaught exceptions leave the process in undefined state — must exit
>   process.exit(1);
> });
> ```
>
> **Layer 4 — Graceful shutdown:**
> ```javascript
> process.on('SIGTERM', async () => {
>   logger.info('SIGTERM received, starting graceful shutdown');
>   server.close(); // stop accepting new connections
>   await drainInFlightRequests(); // wait for active requests to complete (max 30s)
>   await db.disconnect();
>   await redis.quit();
>   process.exit(0);
> });
> ```
> Kubernetes sends SIGTERM before killing a pod. Without graceful shutdown, in-flight requests are dropped during rolling deploys.
>
> **Layer 5 — Circuit breakers and timeouts on outbound calls:** Every `fetch`, database query, and Redis command should have a timeout. Use circuit breakers for external services. Never let a hung dependency hold your event loop hostage.

### Async Error Handling

```javascript
// Prefer async/await with explicit error middleware
app.get('/orders/:id', async (req, res, next) => {
  try {
    const order = await orderService.getById(req.params.id);
    if (!order) return res.status(404).json({ error: 'Not found' });
    res.json(order);
  } catch (err) {
    next(err);
  }
});

app.use((err, req, res, next) => {
  logger.error({ err, requestId: req.id });
  res.status(500).json({ error: 'Internal server error' });
});
```

### CPU-Bound Work — Worker Threads

```javascript
const { Worker } = require('worker_threads');

function runHeavyTask(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./heavy-compute.js', { workerData: data });
    worker.on('message', resolve);
    worker.on('error', reject);
  });
}
```

---

## 12. Java

### Spring Boot Microservice Sketch

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
  private final OrderService orderService;

  public OrderController(OrderService orderService) {
    this.orderService = orderService;
  }

  @GetMapping("/{id}")
  public ResponseEntity<OrderDto> get(@PathVariable String id) {
    return orderService.findById(id)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
  }

  @PostMapping
  public ResponseEntity<OrderDto> create(@Valid @RequestBody CreateOrderRequest req) {
    OrderDto created = orderService.create(req);
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
  }
}
```

### JVM / GC (detailed)

| Collector | Best For | Pause Time | Throughput |
|-----------|----------|------------|------------|
| G1 (default Java 9+) | General purpose, balanced | 50–200ms | Good |
| ZGC | Ultra-low latency (< 10ms pauses) | < 10ms | Moderate |
| Shenandoah | Low latency, similar to ZGC | < 10ms | Moderate |
| Parallel GC | Batch processing, max throughput | 100–500ms+ | Best |

- Tune based on metrics, not defaults — watch GC pause times in APM, heap usage trends, and allocation rates
- `-Xms` should equal `-Xmx` in production to avoid heap resizing pauses
- Container-aware JVM flags are automatic since Java 10 (`UseContainerSupport`)

#### Q: When do you choose Java over Node.js for a microservice?

> **Detailed Answer:** This is one of the most common architect-level questions. There is no universal answer — it depends on workload characteristics, team skills, and ecosystem requirements.
>
> **Choose Java (Spring Boot) when:**
> - **CPU-intensive processing** — complex business rules, pricing engines, fraud detection scoring, report generation. JVM JIT compilation optimizes hot paths over time; Node.js struggles without worker threads.
> - **Enterprise integration** — connecting to SAP, mainframes, SOAP services, JMS queues. Java's enterprise ecosystem (Spring Integration, Apache Camel) is unmatched.
> - **Strong typing and compile-time safety** — large teams (20+ engineers) benefit from Java's type system catching errors at compile time. Refactoring a 500K-line codebase is safer in Java.
> - **Mature observability and tooling** — JFR, JMX, extensive APM support, well-understood GC tuning.
> - **Team expertise** — if 80% of your backend team is Java, forcing Node.js creates a skills gap and slows delivery.
>
> **Choose Node.js when:**
> - **I/O-bound APIs** — CRUD services, BFF (Backend for Frontend), proxy/gateway layers where most time is spent waiting on databases and external APIs.
> - **Real-time features** — WebSockets, Server-Sent Events, chat, live notifications. Node.js event loop handles thousands of concurrent connections efficiently.
> - **Full-stack JavaScript** — same language for frontend and backend reduces context switching, enables code sharing (validation schemas, types via TypeScript).
> - **Fast iteration** — npm ecosystem, hot reload, smaller boilerplate. Prototyping a new service is faster in Node.js.
> - **Serverless** — Lambda cold starts are faster for Node.js than JVM (though GraalVM native images are closing the gap).
>
> **My real-world approach:** In my last platform, we used a **polyglot architecture**:
> - Node.js for API gateway, BFF, and real-time notification service
> - Java for order processing, payment service, and pricing engine
> - Python for ML recommendation service
> - Services communicated via REST (sync) and Kafka (async)
>
> The key principle: **choose the right tool per service, not per company**. Standardize on communication protocols, observability, and deployment — not on language. Every language added increases operational complexity, so limit to 2–3 and have clear criteria for which to use.

#### Q: Explain Spring Boot dependency injection and how it helps in microservices.

> **Detailed Answer:** Spring Boot's IoC (Inversion of Control) container manages object creation and wiring. Instead of `new StripePaymentGateway()` inside your service, you declare dependencies via constructor injection and Spring provides the implementations.
>
> **Why this matters for microservices:**
> 1. **Testability** — in unit tests, inject mock implementations: `@Mock PaymentGateway gateway`. No need for PowerMock or reflection hacks.
> 2. **Configuration-driven behavior** — switch payment provider via `application.yml` (`payment.provider=stripe`), not code changes. Spring profiles (`@Profile("prod")`) activate different beans per environment.
> 3. **Lifecycle management** — Spring manages singleton scopes, connection pool initialization, graceful shutdown of database connections.
> 4. **Cross-cutting concerns** — Spring AOP handles logging, transaction management, security, and retry logic via annotations (`@Transactional`, `@Retryable`, `@Secured`) without polluting business logic.
>
> **Best practices I follow:**
> - Constructor injection only (not `@Autowired` on fields — untestable and hides dependencies)
> - Program to interfaces: `PaymentGateway` interface, `StripeGateway` and `PayPalGateway` implementations
> - Keep `@SpringBootApplication` class thin — auto-configuration handles 90% of setup
> - Externalize all config via `application.yml` + environment variables (12-factor app)
> - Use `@ConfigurationProperties` for type-safe config binding instead of `@Value` scattered everywhere

### Node.js vs Java Decision

| Factor | Node.js | Java |
|--------|---------|------|
| I/O-heavy APIs | Strong | Strong |
| CPU-heavy | Weak (without workers) | Strong |
| Ecosystem | npm, fast iteration | Spring, enterprise maturity |
| Team skill | JS full-stack | Enterprise backend |

---

## 13. Design Patterns & SOLID

### SOLID with Examples

| Principle | Violation | Fix |
|-----------|-----------|-----|
| **S**ingle Responsibility | `OrderService` does payment + email + PDF | Split into focused services |
| **O**pen/Closed | `if (type === 'paypal')` everywhere | Strategy pattern per provider |
| **L**iskov Substitution | Subclass breaks parent contract | Design interfaces carefully |
| **I**nterface Segregation | Fat `Repository` with 20 methods | Small interfaces |
| **D**ependency Inversion | `new StripeClient()` in service | Inject `PaymentGateway` interface |

### Strategy Pattern — Payment Providers (Java)

```java
public interface PaymentGateway {
  PaymentResult charge(Money amount, String customerId);
}

@Service
public class StripeGateway implements PaymentGateway { ... }

@Service
public class PayPalGateway implements PaymentGateway { ... }

@Service
public class PaymentService {
  private final Map<Provider, PaymentGateway> gateways;

  public PaymentResult pay(Provider provider, Money amount, String customerId) {
    return gateways.get(provider).charge(amount, customerId);
  }
}
```

### Circuit Breaker (Node.js concept)

```javascript
class CircuitBreaker {
  constructor(fn, { threshold = 5, resetMs = 30000 } = {}) {
    this.fn = fn;
    this.failures = 0;
    this.threshold = threshold;
    this.resetMs = resetMs;
    this.state = 'CLOSED'; // CLOSED | OPEN | HALF_OPEN
    this.openedAt = 0;
  }

  async call(...args) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.openedAt > this.resetMs) this.state = 'HALF_OPEN';
      else throw new Error('Circuit OPEN');
    }
    try {
      const result = await this.fn(...args);
      this.failures = 0;
      this.state = 'CLOSED';
      return result;
    } catch (err) {
      this.failures++;
      if (this.failures >= this.threshold) {
        this.state = 'OPEN';
        this.openedAt = Date.now();
      }
      throw err;
    }
  }
}
```

### Other Patterns at Scale

| Pattern | Real Use |
|---------|----------|
| Repository | Abstract Mongo/Postgres access |
| Observer / Events | `order.created` → inventory, email |
| Adapter | Wrap legacy SOAP API |
| Strangler Fig | Route 10% traffic to new service |
| Saga | Distributed checkout flow |

#### Q: Explain SOLID principles with a real-world refactoring example.

> **Detailed Answer:** Let me walk through a real refactoring where we applied SOLID to a payment module that had grown into a 2,000-line god class.
>
> **Before (violations):**
> ```java
> public class OrderService {
>   public void processOrder(Order order) {
>     // Validates order (50 lines)
>     // Calculates tax based on state (80 lines)
>     // Charges via Stripe (60 lines)
>     // Sends confirmation email (40 lines)
>     // Generates PDF invoice (70 lines)
>     // Updates inventory (50 lines)
>     // Logs to analytics (30 lines)
>   }
> }
> ```
>
> **S — Single Responsibility:** Split into `OrderValidator`, `TaxCalculator`, `PaymentProcessor`, `NotificationService`, `InvoiceGenerator`, `InventoryService`. Each class has one reason to change. When tax rules changed, we only modified `TaxCalculator`.
>
> **O — Open/Closed:** Payment providers were added via new classes, not by modifying existing code:
> ```java
> public interface PaymentGateway {
>     PaymentResult charge(Money amount, String customerId);
> }
> // Adding PayPal = new PayPalGateway class, register in Spring config
> // Zero changes to PaymentProcessor or OrderService
> ```
>
> **L — Liskov Substitution:** Every `PaymentGateway` implementation must honor the contract — `charge()` always returns a `PaymentResult` or throws `PaymentException`. Never return null. Never throw unchecked exceptions that callers don't expect. We caught a bug where `PayPalGateway.charge()` returned null on timeout instead of throwing — fixed by adding integration tests that verify all implementations behave identically.
>
> **I — Interface Segregation:** Instead of one fat `OrderRepository` with 20 methods, we split into `OrderReader` (find, list), `OrderWriter` (create, update), and `OrderEventPublisher` (publish events). Services only depend on the interface they need.
>
> **D — Dependency Inversion:** `OrderService` depends on `PaymentGateway` interface, not `StripeGateway` concrete class. Spring injects the right implementation at runtime. In tests, we inject `MockPaymentGateway`.
>
> **Outcome:** The god class became 7 focused services averaging 150 lines each. Unit test coverage went from 40% to 92%. Adding a new payment provider (Apple Pay) took 2 days instead of the estimated 2 weeks (because we didn't have to untangle the god class). The team could work on tax rules and payment providers in parallel without merge conflicts.

#### Q: What is the Circuit Breaker pattern and when do you use it?

> **Detailed Answer:** A circuit breaker prevents your service from repeatedly calling a failing dependency, giving it time to recover and protecting your system from cascading failures.
>
> **Three states:**
> - **CLOSED (normal):** Requests flow through to the dependency. Failures are counted.
> - **OPEN (tripped):** After N consecutive failures (e.g., 5 in 30 seconds), the breaker trips. All requests immediately fail fast with a fallback response — no waiting for timeout. The dependency gets breathing room to recover.
> - **HALF-OPEN (testing):** After a reset timeout (e.g., 60 seconds), one probe request is allowed through. If it succeeds, breaker closes. If it fails, breaker reopens.
>
> **When to use:**
> - External APIs (payment gateways, shipping providers, third-party auth)
> - Internal services that are degraded (database under load, search index rebuilding)
> - Any dependency where a timeout + retry would make things worse (retry storm)
>
> **When NOT to use:**
> - Your own database (if DB is down, circuit breaker just hides the problem — fix the DB)
> - Idempotent reads where retry is safe and cheap
> - Services where fallback doesn't exist (no cached data to serve)
>
> **Fallback strategies:**
> - Return cached/stale data ("Prices may not be current")
> - Return a degraded response (hide recommendations section)
> - Queue the request for later processing (async retry)
> - Return a default value (empty list instead of error)
>
> **Libraries:** Resilience4j (Java), opossum (Node.js), Istio service mesh (infrastructure level). I prefer application-level circuit breakers because you control the fallback logic. Mesh-level breakers are good as a safety net.

---

## 14. Testing, TDD & Agile

### Testing Pyramid

```
        /  E2E  \          few, slow, brittle
       / Integr. \         medium
      /   Unit     \       many, fast, cheap
```

### TDD Example (JUnit)

```java
@Test
void shouldCalculateDiscountForPremiumUser() {
  // Red
  User user = new User("u1", Tier.PREMIUM);
  Money price = Money.of(100);

  // Green
  Money discounted = pricingService.applyDiscount(user, price);

  assertEquals(Money.of(90), discounted);
}
```

### Contract Testing (Microservices)

```javascript
// Consumer test with Pact (concept)
const { Pact } = require('@pact-foundation/pact');

describe('Order API contract', () => {
  it('returns order by id', async () => {
    await provider
      .given('order u123 exists')
      .uponReceiving('a request for order u123')
      .withRequest({ method: 'GET', path: '/orders/u123' })
      .willRespondWith({ status: 200, body: { id: 'u123', status: 'CONFIRMED' } });

    const order = await orderClient.get('u123');
    expect(order.status).toBe('CONFIRMED');
  });
});
```

### Agile at Architect Level

- Slice features **vertically** (UI → API → DB → deploy)
- Definition of Done includes: tests, metrics, runbook, rollback plan
- Track tech debt as explicit backlog items
- Use **spikes** (time-boxed) to de-risk unknown architecture

#### Q: How do you implement TDD at the architect/system level?

> **Detailed Answer:** TDD at the architect level goes beyond unit tests — it's about designing systems that are **testable by construction**.
>
> **Unit tests (foundation):** Every service class has unit tests with mocked dependencies. Fast (< 1ms each), run on every commit. Target 80%+ coverage on business logic. Use TDD red-green-refactor for complex algorithms (pricing rules, tax calculation, discount engines).
>
> **Integration tests:** Test service + real database using Testcontainers (spins up Docker MongoDB/PostgreSQL per test suite). Verify repository queries, transaction behavior, index usage. Run in CI on every PR. Slower (seconds) but catch real wiring bugs.
>
> **Contract tests (critical for microservices):** Consumer-driven contracts with Pact. The order-service (consumer) defines what it expects from inventory-service (provider). If inventory-service changes its API in a breaking way, the contract test fails in CI before deployment. This is the **single most valuable test** in a microservices architecture.
>
> **End-to-end tests (minimal):** 5–10 critical user journeys (place order, process refund, user registration). Run against staging environment. Slow (minutes), brittle, expensive to maintain. Keep this layer thin.
>
> **Architecture fitness functions (advanced):** Automated checks that verify architectural constraints:
> - "Order-service must not import from payment-service directly" (ArchUnit for Java)
> - "No REST endpoint without authentication middleware" (custom lint rule)
> - "All services must expose /health/ready endpoint" (integration test)
> - "P95 latency must be < 500ms" (performance test in CI)
>
> **My testing strategy for a new microservice:**
> 1. Write contract tests first (define API, consumer tests drive provider implementation)
> 2. TDD the domain logic (unit tests for business rules)
> 3. Integration tests for database layer (Testcontainers)
> 4. One E2E test for the happy path
> 5. Chaos test: kill dependencies, verify circuit breakers and fallbacks work
> 6. Load test before production (k6 or Gatling, target 2x expected peak traffic)

#### Q: How do you balance technical debt with feature delivery as an architect?

> **Detailed Answer:** Technical debt is not inherently bad — it's a tool. Taking on debt deliberately to ship faster is a valid business decision. The problem is **untracked, unmanaged debt** that compounds silently until a small change takes a week.
>
> **My framework:**
>
> **1. Make debt visible:** Every sprint, 15–20% of capacity is reserved for tech debt reduction. Debt items are in the same backlog as features, with business impact described: "Refactoring payment service reduces incident rate (currently 2 P2s/month) and enables Apple Pay (requested by sales, $2M pipeline)."
>
> **2. Categorize debt:**
> - **Deliberate debt** — "We shipped without caching to meet launch date; ticket DEBT-123 to add Redis cache" (tracked, planned)
> - **Accidental debt** — "We didn't know MongoDB sharding was needed at 10M documents" (discovered, needs spike)
> - **Bit rot** — "Library X is 3 major versions behind with known CVEs" (scheduled upgrade)
> - **Architecture debt** — "Monolith checkout can't scale; needs microservice extraction" (strategic, needs ADR and phased plan)
>
> **3. The boy scout rule:** Every PR that touches a file leaves it slightly better — add a missing test, fix a warning, improve a variable name. Small improvements compound.
>
> **4. Strangler fig for big debt:** Don't propose a 6-month rewrite. Propose: "Month 1: extract inventory service behind API, route 0% traffic. Month 2: route 10% canary. Month 3: 100% traffic, decommission old code." Each month delivers incremental value.
>
> **5. Metrics that justify debt paydown:** Incident frequency, deployment frequency, lead time for changes, MTTR. When I showed leadership that 40% of P2 incidents traced to the legacy payment module, getting approval for refactoring was easy.

---

## 15. Observability (Datadog)

### Three Pillars

| Pillar | What | Example |
|--------|------|---------|
| Metrics | Aggregated numbers | p95 latency, error rate |
| Logs | Discrete events | `{"level":"error","orderId":"x"}` |
| Traces | Request path across services | A → B → C spans |

### RED Method (per service)

- **R**ate — requests/sec
- **E**rrors — % failed
- **D**uration — latency distribution

### Structured Logging (Node.js)

```javascript
const pino = require('pino');
const logger = pino();

app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] || crypto.randomUUID();
  res.setHeader('x-request-id', req.id);
  next();
});

logger.info({ requestId: req.id, userId: req.user?.id, path: req.path }, 'request_start');
```

### SLO Example

```
SLI: % of checkout requests completing < 2s
SLO: 99.9% over 30 days
Error budget: 0.1% ≈ 43 min downtime/month
```

### Production Debug Flow

1. Check SLO dashboard — which service degraded?
2. Open trace — slow span?
3. Correlate logs by `requestId`
4. Check recent deploys / feature flags
5. Mitigate (scale, rollback, disable feature) → postmortem

#### Q: Walk me through debugging a production latency spike step by step.

> **Detailed Answer:** This is a classic architect interview question. I'll walk through a real incident where checkout P95 latency jumped from 800ms to 4.5 seconds.
>
> **Step 1 — Detect (0–2 minutes):**
> Datadog SLO dashboard showed checkout SLO burning error budget 3x faster than normal. P95 latency alert fired. PagerDuty notified on-call. I opened the service overview dashboard — order-service RED metrics showed rate stable (no traffic spike) but duration p95 up 5x and error rate at 2% (normally 0.1%).
>
> **Step 2 — Triage (2–5 minutes):**
> Checked recent deploys — order-service v1.2.3 deployed 15 minutes ago. Correlation, not causation yet. Checked infrastructure — CPU and memory normal, no pod restarts, no node pressure. Checked downstream dependencies — payment-service healthy, inventory-service healthy, but MongoDB p95 query time elevated from 12ms to 340ms.
>
> **Step 3 — Drill down (5–10 minutes):**
> Opened a slow trace in Datadog APM. The checkout request spent 3.8 of 4.5 seconds in a MongoDB `find` query on the `orders` collection. The query was `{ userId: "u123", status: "PENDING" }` — a new query added in v1.2.3 to check for duplicate pending orders. Checked MongoDB explain plan — **COLLSCAN** (full collection scan). The new query used `userId + status` but the existing index was only on `userId + createdAt`. With 50M documents, full scan took seconds.
>
> **Step 4 — Mitigate (10–12 minutes):**
> Short-term: rolled back order-service to v1.2.2. Latency returned to normal within 2 minutes (rolling deploy with readiness probes). Long-term fix: added compound index `{ userId: 1, status: 1 }` in staging, verified with explain plan (IXSCAN, 3ms), deployed index to production during low-traffic window, then re-deployed v1.2.3.
>
> **Step 5 — Postmortem (next day):**
> Root cause: new query added without index review. Contributing factors: no integration test with production-scale data, no index check in CI pipeline. Action items: (1) added MongoDB explain plan check to PR review checklist, (2) created ArchUnit-style lint rule that flags new queries without corresponding index in migration files, (3) added synthetic canary test that runs checkout every 60 seconds and alerts on latency regression.
>
> **Key lesson for interviews:** Always walk through detection → triage → root cause → mitigation → prevention. Show you think about both immediate fix AND systemic prevention.

#### Q: How do you define and measure SLOs for a microservices platform?

> **Detailed Answer:** SLOs (Service Level Objectives) translate user expectations into measurable targets that drive engineering decisions.
>
> **Step 1 — Identify SLIs (what to measure):**
> - **Availability:** % of successful requests (non-5xx responses)
> - **Latency:** % of requests completing under threshold (e.g., < 500ms)
> - **Correctness:** % of requests returning correct data (harder to measure, use synthetic checks)
> - **Freshness:** % of data updated within SLA (for async pipelines)
>
> **Step 2 — Set SLO targets (based on user expectations, not 100%):**
> | Service | SLI | SLO | Error Budget (monthly) |
> |---------|-----|-----|------------------------|
> | Checkout API | Latency < 2s | 99.9% | 43 minutes of bad latency |
> | Product Search | Latency < 500ms | 99.5% | 3.6 hours |
> | Order Status | Availability | 99.95% | 21 minutes downtime |
> | Analytics Dashboard | Freshness < 1 hour | 99% | 7.2 hours stale |
>
> **Step 3 — Implement measurement:**
> - Datadog SLO monitors track SLI continuously
> - Error budget = 1 - SLO target. When budget is exhausted, stop feature releases and focus on reliability.
> - Burn rate alerts: "At current error rate, budget will be exhausted in 6 hours" (fast burn) vs "in 3 days" (slow burn)
>
> **Step 4 — Use SLOs in decisions:**
> - "Can we deploy on Friday?" → Check error budget. If 80% consumed, defer to Monday.
> - "Should we add caching?" → Caching improves latency SLO but adds complexity. Quantify: "Redis cache will improve p95 from 800ms to 200ms, giving us 3x headroom on our latency SLO."
> - "Do we need a postmortem?" → Any incident that consumed > 10% of monthly error budget gets a blameless postmortem.

---

## 16. DevOps: Docker, Kubernetes, Jenkins

### Multi-Stage Dockerfile (Node.js)

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Kubernetes Deployment Snippet

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/order-service:1.2.3
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

### Jenkins Pipeline (declarative)

```groovy
pipeline {
  agent any
  stages {
    stage('Build') {
      steps { sh 'npm ci && npm run build' }
    }
    stage('Test') {
      steps { sh 'npm test' }
    }
    stage('Docker') {
      steps {
        sh 'docker build -t order-service:${BUILD_NUMBER} .'
        sh 'docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/order-service:${BUILD_NUMBER}'
      }
    }
    stage('Deploy') {
      steps {
        sh 'kubectl set image deployment/order-service order-service=...:${BUILD_NUMBER}'
      }
    }
  }
}
```

### Zero-Downtime Deploy

- Rolling update with **readiness probes**
- **Canary** 5% → 25% → 100% with metric gates
- DB migrations: backward-compatible first (expand → migrate → contract)

#### Q: Explain the expand-migrate-contract pattern for database migrations in microservices.

> **Detailed Answer:** Database migrations in production with zero downtime require a **three-phase approach** because your old and new code run simultaneously during rolling deploys.
>
> **Phase 1 — Expand (add, don't remove):**
> Add the new column/table/index without removing anything. Old code ignores it, new code can write to it.
> ```sql
> -- Example: splitting full_name into first_name + last_name
> ALTER TABLE users ADD COLUMN first_name VARCHAR(100);
> ALTER TABLE users ADD COLUMN last_name VARCHAR(100);
> ```
> Deploy: backward-compatible. Old code still reads/writes `full_name`. New code writes to all three columns.
>
> **Phase 2 — Migrate (backfill data):**
> Run a background job to populate new columns from old data.
> ```sql
> UPDATE users SET first_name = SPLIT_PART(full_name, ' ', 1),
>                  last_name = SPLIT_PART(full_name, ' ', 2)
> WHERE first_name IS NULL;
> ```
> Dual-write period: both old and new code paths are active. New code reads from new columns, falls back to old if null.
>
> **Phase 3 — Contract (remove old):**
> Once all data is migrated and all code reads from new columns, remove the old column.
> ```sql
> ALTER TABLE users DROP COLUMN full_name;
> ```
> Deploy: only new code is running. Old column is gone.
>
> **Critical rules:**
> - Never rename a column in one step (add new → migrate → drop old)
> - Never change column type in one step (add new column with new type → migrate → drop old)
> - Never drop a column while old code still references it
> - Each phase is a separate deployment, hours or days apart
> - Feature flags control which code path is active during migration
>
> **Real example:** We migrated order status from an enum (`PENDING, CONFIRMED, SHIPPED`) to a state machine (12 states) over 3 weeks. Week 1: added `status_v2` column, dual-write. Week 2: backfilled, new code read from `status_v2`. Week 3: dropped `status` column. Zero downtime, zero data loss.

#### Q: How do Kubernetes readiness and liveness probes work, and how do you configure them?

> **Detailed Answer:**
>
> **Liveness probe** — "Is the process alive?" If it fails, Kubernetes **kills and restarts** the pod. Use for detecting deadlocks, infinite loops, or stuck processes.
> ```yaml
> livenessProbe:
>   httpGet:
>     path: /health/live
>     port: 3000
>   initialDelaySeconds: 15  # wait for app startup
>   periodSeconds: 20        # check every 20s
>   failureThreshold: 3      # restart after 3 failures
> ```
> `/health/live` should be trivial — return 200 if the process is running. Do NOT check database connectivity here (a DB blip would kill all pods).
>
> **Readiness probe** — "Can this pod accept traffic?" If it fails, Kubernetes **removes the pod from the service endpoints** (no traffic routed to it) but does NOT restart it. Use for checking dependencies and warmup state.
> ```yaml
> readinessProbe:
>   httpGet:
>     path: /health/ready
>     port: 3000
>   initialDelaySeconds: 5
>   periodSeconds: 10
>   failureThreshold: 3
> ```
> `/health/ready` should check: database connection, Redis connection, any required warmup (cache loaded). During a rolling deploy, new pods must pass readiness before old pods are terminated — this ensures zero-downtime.
>
> **Startup probe** — for slow-starting apps (Java with large heap). Gives the app more time before liveness/readiness kick in.
> ```yaml
> startupProbe:
>   httpGet:
>     path: /health/live
>     port: 3000
>   failureThreshold: 30    # 30 * 10s = 5 minutes to start
>   periodSeconds: 10
> ```
>
> **Common mistakes:**
> - Liveness probe checks DB → DB goes down → all pods restart → thundering herd → worse outage
> - `initialDelaySeconds` too short → pod killed before app finishes starting
> - Readiness probe too aggressive → pod flaps in and out of load balancer during GC pauses
> - No probe at all → broken pods receive traffic, users see errors

---

## 17. System Design Walkthroughs

### Design A: E-Commerce Order Platform

```
Client → CloudFront → API Gateway → Order Service
                                      ↓ (Temporal workflow)
                    Inventory Svc ← → Payment Svc
                         ↓                ↓
                    MongoDB           Payment GW
Order events → SQS → Search Indexer → OpenSearch
Session/cache → Redis (ElastiCache)
Metrics/logs/traces → Datadog
Deploy → EKS, CI via Jenkins
```

**Key decisions:**

| Decision | Choice | Why |
|----------|--------|-----|
| Workflow | Temporal saga | Compensations on payment failure |
| Order DB | MongoDB per service | Flexible order schema |
| Search | OpenSearch | Product catalog full-text |
| Cache | Redis | Cart/session, rate limits |
| Events | SQS + EventBridge | Decouple indexing, notifications |

#### Full Spoken Answer: Design an E-Commerce Order Platform

> **Step 1 — Clarify requirements (always start here):**
> "Let me ask a few questions to scope this. What's the scale — orders per day? Peak vs average? Do we need real-time inventory, or is eventual consistency OK? What's the read/write ratio? Do we need multi-region? What's the latency SLA for checkout?"
>
> *Assume interviewer says:* 50K orders/day average, 5K orders/minute peak, inventory must be strongly consistent (no overselling), 80% reads (order history, tracking), single region (US), checkout < 2 seconds p95.
>
> **Step 2 — High-level architecture:**
> "I'll design this as a microservices platform on AWS with an event-driven architecture. The core flow is: client → CDN → API Gateway → order service orchestrates checkout via Temporal saga → events propagate to search, notifications, and analytics asynchronously."
>
> **Step 3 — API Layer:**
> "CloudFront caches static assets and provides DDoS protection. API Gateway handles authentication (JWT validation via Cognito), rate limiting (1,000 req/min per user), and routes to backend services. I'll use a BFF (Backend for Frontend) pattern — separate BFF for web and mobile since they need different data shapes."
>
> **Step 4 — Core services:**
> - **Order Service** (Node.js): cart management, checkout orchestration, order history. Owns MongoDB `orders` collection. Publishes `order.created`, `order.updated` events.
> - **Inventory Service** (Java): stock levels, reservations, releases. Owns MongoDB `inventory` collection. Uses atomic `findOneAndUpdate` for reservation (no overselling). Exposes `reserve`, `release`, `confirm` APIs.
> - **Payment Service** (Java): payment processing, refunds. Integrates with Stripe via adapter pattern. Stores idempotency keys in Redis. PCI scope minimized — we tokenize cards via Stripe Elements, never touch raw card data.
> - **Catalog Service** (Node.js): products, categories, pricing. Read-heavy, cached aggressively in Redis (1-hour TTL).
> - **Notification Service** (Node.js): email, SMS, push. Consumes events from SQS, uses SES and SNS.
>
> **Step 5 — Checkout flow (Temporal saga):**
> 1. Client submits checkout → Order Service starts Temporal workflow
> 2. Activity: Reserve inventory (inventory-service, timeout 5s, retry 3x)
> 3. Activity: Charge payment (payment-service, timeout 15s, retry 1x — idempotent)
> 4. Activity: Confirm order (order-service, update status to CONFIRMED)
> 5. Activity: Publish `order.created` event to SQS
> 6. On any failure: run compensations in reverse (release inventory, refund payment, cancel order)
>
> **Step 6 — Data stores:**
> - MongoDB per service (database-per-service pattern). Order documents embed line items. Inventory uses atomic operations. Separate replica sets, no shared database.
> - Redis (ElastiCache cluster mode): cart state (TTL 24h), session data, idempotency keys, rate limit counters, product catalog cache.
> - OpenSearch: product search index, order search for admin dashboard. Synced via outbox pattern + SQS.
> - S3: invoice PDFs, product images, analytics raw data.
>
> **Step 7 — Scalability:**
> - Order service: 3–20 pods, HPA on CPU (60%) and custom metric (Temporal workflow queue depth)
> - MongoDB: start with 3-node replica set, shard when > 500GB or > 10K writes/sec. Shard key: `tenantId` for multi-tenant or hashed `userId`.
> - Redis: cluster mode, 3 shards, 1 replica each. 50K ops/sec capacity.
> - SQS: standard queues for order events (at-least-once, idempotent consumers). FIFO for payment events (exact ordering per customer).
>
> **Step 8 — Reliability:**
> - Multi-AZ deployment for all services and data stores
> - Circuit breakers on all external calls (payment gateway, shipping API)
> - Idempotency keys on all mutating APIs
> - SLO: 99.9% checkout success, p95 < 2 seconds
> - Error budget policy: stop releases if budget < 20% remaining
>
> **Step 9 — What I'd add with more time:**
> - Multi-region active-passive for disaster recovery
> - CQRS for order history reads (separate read model in OpenSearch)
> - Event sourcing for full order audit trail
> - Real-time order tracking via WebSocket (API Gateway WebSocket → Lambda → DynamoDB)

### Design B: Daily Analytics Pipeline

```
App logs → Kinesis Firehose → S3 (raw)
Airflow DAG (nightly): S3 → transform → Redshift/Athena
Dashboards → Datadog / QuickSight
```

**Trade-off:** Real-time (Kinesis + Flink) costs more; batch (Airflow) is cheaper for T+1 reports.

#### Full Spoken Answer: Design a Real-Time Analytics Dashboard

> **Clarifying questions:** "What's the data volume? Latency requirement — real-time, near-real-time (minutes), or batch (hours)? Who are the consumers — business analysts, product managers, or automated systems? What's the query pattern — pre-built dashboards or ad-hoc SQL?"
>
> *Assume:* 10GB logs/day, T+1 is acceptable for business reports, but product team wants near-real-time (5 min) for feature usage metrics. Analysts need ad-hoc SQL. Budget-conscious.
>
> **Architecture:**
> 1. **Ingestion:** Application services emit structured events (JSON) to Kinesis Data Streams (4 shards, ~4MB/sec capacity). Kinesis Firehose batches and delivers to S3 every 60 seconds in Parquet format (columnar, compressed, query-efficient). Partitioned by `year/month/day/hour`.
>
> 2. **Batch pipeline (Airflow, nightly):**
> - DAG runs at 2 AM UTC
> - Task 1: `extract` — read yesterday's S3 Parquet files
> - Task 2: `transform` — aggregate: DAU, feature usage counts, funnel conversion rates, revenue by segment
> - Task 3: `load` — write aggregates to Redshift (for analyst SQL) and push summary metrics to Datadog (for engineering dashboards)
> - Task 4: `validate` — row count checks, null checks, compare with previous day (alert if > 20% deviation)
>
> 3. **Near-real-time path (for product team):**
> - Kinesis Data Analytics (Flink) computes 5-minute tumbling window aggregates: page views, feature clicks, error rates
> - Results written to DynamoDB (key: `metric#feature#window`, value: count)
> - Simple API reads from DynamoDB for product dashboard
> - Cost: ~$200/month for Flink application vs ~$2,000/month for full real-time pipeline
>
> 4. **Query layer:**
> - Business analysts: Amazon Athena (serverless SQL on S3) for ad-hoc queries, $5/TB scanned
> - Pre-built dashboards: QuickSight connected to Redshift for daily reports
> - Engineering: Datadog dashboards for real-time operational metrics
>
> **Why not just real-time everything?** At 10GB/day, batch processing costs ~$50/month (S3 + Athena + Airflow on MWAA). Full real-time with Kafka + Flink + ClickHouse would cost ~$2,000/month. The product team's 5-minute latency need is met by the lightweight Flink path without over-engineering the entire pipeline.

---

## 18. Behavioral & Communication

### STAR Story Templates

**1. Architectural decision (full 2-minute version)**

- **S:** At my previous company, we had a monolithic e-commerce application handling 2,000 orders/minute at peak. During Black Friday, checkout failure rate hit 8% due to database connection pool exhaustion and cascading timeouts across the synchronous call chain (inventory → payment → notification → email).
- **T:** I was asked to design a scalable order processing architecture that could handle 3x peak traffic (6,000 orders/min) with 99.9% availability, with a 3-month timeline and without a full rewrite.
- **A:** I proposed a phased migration using the strangler fig pattern. Phase 1: extracted inventory and payment into separate services with their own databases. Phase 2: replaced the synchronous checkout chain with a Temporal saga (reserve → charge → confirm with compensations). Phase 3: moved order history reads to OpenSearch via CQRS. I wrote ADRs for each decision, ran a 2-week POC proving Temporal could handle our workflow complexity, and created a migration runbook with rollback procedures. I presented trade-offs to leadership: saga adds complexity but eliminates cascading failures; event-driven indexing adds 2-3 second search lag but decouples services.
- **R:** System handled 6,500 orders/min on the next Black Friday with 99.95% checkout success (up from 92%). P95 checkout latency dropped from 4.2s to 1.1s. Zero downtime during migration. Team velocity increased because services could be deployed independently. The architecture became the template for extracting 3 more domains over the following year.

**2. Production incident (full 2-minute version)**

- **S:** At 2:15 AM, PagerDuty alerted that payment error rate spiked from 0.1% to 12% across all regions. Approximately 800 customers were affected in the first 10 minutes. The incident occurred 20 minutes after a scheduled deployment of payment-service v2.4.0.
- **T:** As the architect on-call, I needed to restore payment processing within our 15-minute MTTR SLO and prevent duplicate charges for affected customers.
- **A:** I joined the incident bridge within 3 minutes. First action: checked Datadog — payment-service error rate correlated with deploy timestamp. Rolled back to v2.3.9 via `kubectl rollout undo` — error rate dropped to 0.2% within 4 minutes. Then investigated root cause: v2.4.0 changed idempotency key generation from `orderId + attempt` to just `orderId`, causing legitimate retries (from gateway timeouts) to be rejected as duplicates. Identified 340 affected transactions via Datadog trace analysis. Wrote a hotfix restoring the original key format, added integration tests for retry scenarios, and manually reconciled the 340 affected payments (none were double-charged due to the duplicate rejection, but 12 legitimate retries were blocked and needed reprocessing).
- **R:** MTTR was 11 minutes (within SLO). Zero duplicate charges. Added contract tests for idempotency behavior, made idempotency key format part of the API schema (breaking change requires major version bump), and added a deploy-time canary that runs 100 payment scenarios before promoting to full traffic.

**3. Influence without authority (full 2-minute version)**

- **S:** Our platform had 12 microservices, each using different logging formats — some used `console.log`, some used Winston with different field names, some used Log4j with yet another format. During incidents, engineers spent 30–45 minutes just figuring out how to search logs across services. Mean time to diagnose (MTTD) was unacceptable.
- **T:** I needed to standardize observability across all teams, but I had no direct authority over 4 different engineering teams, each with their own priorities and timelines.
- **A:** Instead of mandating a top-down standard, I took a collaborative approach. First, I interviewed engineers from each team to understand their pain points (everyone agreed logging was broken). I drafted an RFC proposing structured JSON logging with a shared schema (`timestamp`, `level`, `service`, `requestId`, `userId`, `message`, `metadata`). I built a reference implementation — a shared Node.js and Java logging library that auto-injected `requestId` from headers and formatted output correctly. I paired with one enthusiastic team to adopt it, measured their MTTD improvement (45 min → 12 min on their next incident), and published the results. I presented at our engineering all-hands with before/after screenshots. I offered office hours to help teams migrate.
- **R:** Within 3 months, 11 of 12 services adopted the standard (the 12th was scheduled for decommission). Average MTTD across the platform dropped from 38 minutes to 14 minutes. The shared library became a platform team responsibility. This experience taught me that influence comes from solving people's problems, not from organizational authority.

### Additional STAR Stories to Prepare

**4. Technical disagreement with leadership:**
- **S:** Leadership wanted to rebuild the entire platform in Kubernetes within 6 months. Current platform ran on EC2 with manual deployments.
- **T:** I needed to advocate for a pragmatic approach without appearing obstructive.
- **A:** I prepared a comparison document: full rebuild (6 months, high risk, no features shipped) vs phased migration (strangler fig, ship features throughout). I proposed a 2-week spike to containerize one service and deploy to EKS. The spike succeeded, giving us real data on effort and risk.
- **R:** Leadership approved the phased approach. We containerized 4 services in 4 months while shipping 2 major features. Full migration completed in 9 months with zero downtime.

**5. Mentoring and growing engineers:**
- **S:** A junior engineer on my team was struggling with system design — they could code well but couldn't reason about trade-offs or design services independently.
- **T:** I needed to grow them to mid-level within 6 months so they could own a service.
- **A:** I paired weekly on design reviews (they presented options, I asked questions instead of giving answers). I assigned them a small service to own with fortnightly check-ins. I shared ADR templates and had them write ADRs for their design decisions. I recommended specific resources (DDIA book, system design primer).
- **R:** They independently designed and shipped a notification service within 5 months. Promoted to mid-level engineer. They now mentor new juniors using the same approach.

### Questions to Ask Interviewers

| Question | Why It Matters |
|----------|----------------|
| How are architecture decisions documented (ADRs, RFCs)? | Shows you value decision traceability |
| What are the current SLOs and the biggest reliability gap? | Shows you think about operational excellence |
| Platform team vs product team structure? | Helps you understand influence dynamics |
| Top technical debt priority for next year? | Shows you think long-term |
| How do teams handle on-call and incident response? | Reveals operational maturity |
| What's the deployment frequency and lead time for changes? | DORA metrics indicate engineering health |
| How do you evaluate build vs buy for infrastructure? | Shows strategic thinking |

---

## 19. Quick Reference Cheat Sheet

```
ARCHITECTURE
  Stateless + horizontal scale > bigger VMs
  Prefer async/events over deep sync chains
  Document decisions in ADRs

DISTRIBUTED SYSTEMS
  At-least-once + idempotency > exactly-once fantasy
  Timeouts + retries + circuit breakers on every outbound call
  Shard by high-cardinality key (userId, tenantId)

EVENT-DRIVEN ARCHITECTURE
  Events = facts that happened; commands = requests to do work
  RabbitMQ → smart broker, queues, complex routing, ACK deletes message
  Kafka → durable log, partitions, replay, multiple independent consumer groups
  Use outbox pattern; make every consumer idempotent
  Task queue / work distribution → RabbitMQ or SQS
  Event stream / CDC / replay / many consumers → Kafka

DATA
  MongoDB  → flexible documents, index-aware queries
  Redis    → cache/sessions/rate limits; define failure behavior
  OpenSearch → search index only; sync from source of truth

WORKFLOWS
  Airflow  → scheduled batch ETL
  Temporal → durable business sagas

AWS
  Managed services reduce ops; IAM roles not static keys
  Multi-AZ for HA; design for AZ failure

CODE QUALITY
  Testing pyramid; contract tests between services
  SOLID at module/service level, not just classes

OBSERVABILITY
  Metrics (RED) + structured logs + distributed traces
  SLOs drive release decisions

DEVOPS
  Immutable containers; readiness probes; canary deploys
```

---

## Monday Morning 15-Minute Review

1. Recite architect answer framework (Context → Options → Trade-offs → Decision → Outcome)
2. Draw e-commerce design from memory (API GW → services → Mongo/Redis/ES)
3. State Airflow vs Temporal in one sentence each, plus when to use which
4. Explain event-driven architecture: RabbitMQ order fan-out vs Kafka product stream + replay
5. Explain cache-aside + stampede protection with failure mode
6. Walk through prod debug: metrics → traces → logs → deploy → mitigate
7. Tell your best architectural decision story (60 seconds, then 2-minute version)
8. Explain saga pattern: choreography vs orchestration with compensation example
9. Describe expand-migrate-contract for zero-downtime DB migrations
10. Answer "How do you define service boundaries?" using DDD bounded contexts
11. State your Node.js vs Java decision criteria for a new microservice

---

## Appendix: Top Interview Questions Checklist

Use this to verify you can answer each question in 90–120 seconds with the COTDO framework.

| # | Question | Section |
|---|----------|---------|
| 1 | How do you structure an architectural answer? | §1 |
| 2 | Explain CAP theorem with a real example | §2 |
| 3 | When is eventual consistency acceptable? | §2 |
| 4 | Explain the Saga pattern — choreography vs orchestration | §2 |
| 5 | How do you define microservice boundaries? | §3 |
| 6 | Walk me through designing a service on AWS | §3 |
| 7 | How do you ensure exactly-once processing? | §4 |
| 8 | Explain consistent hashing | §4 |
| 9 | Explain event-driven architecture with a RabbitMQ example | §5 |
| 10 | Explain event-driven architecture with a Kafka example | §5 |
| 11 | When do you choose RabbitMQ vs Kafka? | §5 |
| 12 | When would you choose Temporal over Airflow? | §6 |
| 13 | How does Temporal guarantee durability? | §6 |
| 14 | How do you design a MongoDB schema for high traffic? | §7 |
| 15 | How do you handle MongoDB transactions? | §7 |
| 16 | How do you keep OpenSearch in sync with the primary DB? | §8 |
| 17 | Explain full-text search in ElasticSearch | §8 |
| 18 | Compare cache-aside, write-through, write-behind | §9 |
| 19 | Design a distributed rate limiter with Redis | §9 |
| 20 | When do algorithms matter at the architect level? | §10 |
| 21 | Explain the Node.js event loop | §11 |
| 22 | How do you handle errors in production Node.js? | §11 |
| 23 | When do you choose Java over Node.js? | §12 |
| 24 | Explain SOLID with a refactoring example | §13 |
| 25 | What is the Circuit Breaker pattern? | §13 |
| 26 | How do you implement TDD at the system level? | §14 |
| 27 | How do you balance tech debt with feature delivery? | §14 |
| 28 | Walk through debugging a production latency spike | §15 |
| 29 | How do you define and measure SLOs? | §15 |
| 30 | Explain expand-migrate-contract for DB migrations | §16 |
| 31 | Design an e-commerce order platform | §17 |
| 32 | Design a real-time analytics dashboard | §17 |
| 33 | Tell me about a time you influenced without authority | §18 |

---

*Good luck. Lead with trade-offs, back claims with examples, communicate risks proactively, and always ask clarifying questions before designing.*
