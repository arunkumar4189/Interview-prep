# Technical Architect Interview Guide

> JD-focused refresh guide with brief answers, real-world examples, and sample programs.  
> Stack emphasis: Node.js, Java, AWS, microservices, MongoDB, Redis, ElasticSearch, Temporal/Airflow.

---

## Table of Contents

1. [How to Structure Answers](#1-how-to-structure-answers)
2. [Architecture Fundamentals](#2-architecture-fundamentals)
3. [Microservices & Cloud-Native AWS](#3-microservices--cloud-native-aws)
4. [Distributed Systems](#4-distributed-systems)
5. [Workflow Orchestration (Airflow vs Temporal)](#5-workflow-orchestration-airflow-vs-temporal)
6. [MongoDB](#6-mongodb)
7. [ElasticSearch / OpenSearch](#7-elasticsearch--opensearch)
8. [Redis (Distributed Cache)](#8-redis-distributed-cache)
9. [Algorithms & Data Structures](#9-algorithms--data-structures)
10. [Node.js](#10-nodejs)
11. [Java](#11-java)
12. [Design Patterns & SOLID](#12-design-patterns--solid)
13. [Testing, TDD & Agile](#13-testing-tdd--agile)
14. [Observability (Datadog)](#14-observability-datadog)
15. [DevOps: Docker, Kubernetes, Jenkins](#15-devops-docker-kubernetes-jenkins)
16. [System Design Walkthroughs](#16-system-design-walkthroughs)
17. [Behavioral & Communication](#17-behavioral--communication)
18. [Quick Reference Cheat Sheet](#18-quick-reference-cheat-sheet)

---

## 1. How to Structure Answers

**Framework:** Context → Options → Trade-offs → Decision → Outcome

**Example (verbal):**

> *Context:* E-commerce checkout at 5k orders/min with occasional payment timeouts.  
> *Options:* (1) Sync REST chain, (2) Saga with compensations, (3) 2PC across DBs.  
> *Trade-offs:* Sync is simple but fragile; 2PC hurts availability; saga adds complexity but scales.  
> *Decision:* Choreographed saga — reserve inventory → charge payment → confirm order; compensate on failure.  
> *Outcome:* 99.95% checkout success; payment retries isolated; no distributed locks.

---

## 2. Architecture Fundamentals

### Scalability

| Type | Meaning | Example |
|------|---------|---------|
| Vertical | Bigger machine | 8 → 32 GB RAM |
| Horizontal | More machines | 3 → 30 API pods behind ALB |

**Rule:** Prefer horizontal scaling for stateless services; use caching, sharding, and async for stateful bottlenecks.

### CAP Theorem

During a network partition, choose **Consistency** or **Availability** — not both.

- **CP:** Banking ledger (strong consistency matters)
- **AP:** Social feed likes count (eventual consistency OK)

### Consistency Models

| Model | Behavior |
|-------|----------|
| Strong | Read always returns latest write |
| Eventual | Replicas converge over time |
| Read-your-writes | User sees their own updates immediately |

### Key Patterns

| Pattern | Use When |
|---------|----------|
| API Gateway | Single entry, auth, rate limiting |
| Circuit Breaker | External dependency failing |
| CQRS | Read/write load profiles differ |
| Saga | Distributed transaction across services |
| Strangler Fig | Gradual monolith migration |

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

- Shared database across services
- Synchronous call chains 5+ deep
- Distributed monolith (tight coupling via sync calls)

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

1. Set timeouts on every outbound call
2. Retry with exponential backoff + jitter
3. Circuit breaker after N failures
4. Bulkhead: isolate thread pools per dependency
5. Graceful degradation (serve cached/stale data)

---

## 5. Workflow Orchestration (Airflow vs Temporal)

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

---

## 6. MongoDB

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
Shard key: tenantId (good — isolates tenants)
Shard key: createdAt (bad — hot shard on writes)
```

---

## 7. ElasticSearch / OpenSearch

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

- Mapping changes on live index require reindex
- Deep aggregations on huge datasets are expensive
- Keep transactional writes in primary DB

---

## 8. Redis (Distributed Cache)

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

> *"If Redis is down, we bypass cache and hit the database with circuit breaker limits. We never serve silently wrong financial data from stale cache."*

---

## 9. Algorithms & Data Structures

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
O(log n)   — binary search, balanced tree
O(n)       — linear scan
O(n log n) — efficient sort
```

---

## 10. Node.js

### Event Loop (brief)

```
Timers → Pending callbacks → Idle → Poll → Check → Close
```

- **Good for:** I/O-bound APIs, real-time, high concurrency
- **Bad for:** CPU-heavy work on main thread

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

## 11. Java

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

### JVM / GC (interview sound bite)

- **Throughput collectors (G1):** batch workloads
- **Low-latency (ZGC/Shenandoah):** strict p99 latency SLAs
- Tune based on metrics, not defaults

### Node.js vs Java Decision

| Factor | Node.js | Java |
|--------|---------|------|
| I/O-heavy APIs | Strong | Strong |
| CPU-heavy | Weak (without workers) | Strong |
| Ecosystem | npm, fast iteration | Spring, enterprise maturity |
| Team skill | JS full-stack | Enterprise backend |

---

## 12. Design Patterns & SOLID

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

---

## 13. Testing, TDD & Agile

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

---

## 14. Observability (Datadog)

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

---

## 15. DevOps: Docker, Kubernetes, Jenkins

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

---

## 16. System Design Walkthroughs

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

### Design B: Daily Analytics Pipeline

```
App logs → Kinesis Firehose → S3 (raw)
Airflow DAG (nightly): S3 → transform → Redshift/Athena
Dashboards → Datadog / QuickSight
```

**Trade-off:** Real-time (Kinesis + Flink) costs more; batch (Airflow) is cheaper for T+1 reports.

---

## 17. Behavioral & Communication

### STAR Story Templates

**1. Architectural decision**

- **S:** Monolith couldn't scale checkout during peak sales
- **T:** Design scalable order flow
- **A:** Proposed saga-based microservices; wrote ADR; ran POC on Temporal
- **R:** Handled 3x traffic; p95 checkout latency dropped 60%

**2. Production incident**

- **S:** Payment errors spiked after deploy
- **T:** Restore service within 15 min SLO
- **A:** Rolled back, identified bad idempotency key logic, hotfixed, added alert
- **R:** MTTR 12 min; added contract test to prevent recurrence

**3. Influence without authority**

- **S:** Teams used 3 different logging formats
- **T:** Standardize observability
- **A:** Proposed structured logging RFC, built shared library, paired with teams
- **R:** 100% services on standard; incident debug time cut in half

### Questions to Ask Interviewers

- How are architecture decisions documented (ADRs, RFCs)?
- Current SLOs and biggest reliability gap?
- Platform vs product team structure?
- Top technical debt priority for next year?

---

## 18. Quick Reference Cheat Sheet

```
ARCHITECTURE
  Stateless + horizontal scale > bigger VMs
  Prefer async/events over deep sync chains
  Document decisions in ADRs

DISTRIBUTED SYSTEMS
  At-least-once + idempotency > exactly-once fantasy
  Timeouts + retries + circuit breakers on every outbound call
  Shard by high-cardinality key (userId, tenantId)

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
3. State Airflow vs Temporal in one sentence each
4. Explain cache-aside + stampede protection
5. Walk through prod debug: metrics → traces → logs → deploy → mitigate
6. Tell your best architectural decision story (60 seconds)

---

*Good luck on Monday. Lead with trade-offs, back claims with examples, and communicate risks proactively.*
