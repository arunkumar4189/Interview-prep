# Good-to-have stack (detailed) — tied to your projects

| Tech | Your production/study anchor |
|------|------------------------------|
| RabbitMQ | **P2** bulk lines (primary event bus you operated) |
| Kafka | Same patterns as **P2**; say you’d migrate partition key `tenant:batchId` |
| Redis | **P3** ConfigMs / hot catalog (**resume lists Redis**) |
| Cassandra | **P1** order + batch line tables |
| Postgres | **P9** Agentic Universe runs + approvals |
| OpenSearch | **P10** SRE log correlation |
| MongoDB | Not on resume—optional journey doc; skip unless asked |
| Aerospike | Not operated—compare to **Cassandra** read path + atomic counter need |

Catalog: [11-your-projects-catalog.md](./11-your-projects-catalog.md).

---

## 1. Kafka (JD) — speak from RabbitMQ (P2)

### Components

Topic, partition, offset, consumer group, ISR—see prior guide.

### Your mapping

```
exchange/queue: bulk.order.lines  →  topic: bulk.order.line.v1
routing key: tenant.batch         →  key: tenantId:batchId
consumer ack after line durable   →  commit offset after (batch_id,line_id) write
DLQ queue                         →  DLQ topic
```

### Debug “producer OK, consumer silent”

Wrong vhost/queue, poison message, prefetch stuck, no consumers bound—same mental model you used on **Rabbit** during incidents.

### At-least-once (Tekion HM)

Commit/ack **after** idempotent Cassandra/DB write for the line.

---

## 2. Redis (P3 ConfigMs)

### Catalog cache

```
config:v1:{tenant}:{offerId}  TTL 120s
invalidate on config publish
```

### Rate limit (if downstream order system throttled during bulk)

Sliding window Lua on `rate:{tenant}:order-submit`—protect **order domain** when **P2** spikes.

### Not primary store for

Payment intent (**P3**), batch header (**P2**), agent approval (**P9**).

---

## 3. OpenSearch / Elasticsearch (P10)

Index: application logs with `traceId`, `batchId`, `lineId` fields (if you added them in logging spec).

SRE Triaging node: query last 30m errors for trace—feeds LangGraph, does not update orders.

**Creative/audience search at Tekion:** “I have not built that; I built **log evidence retrieval** for RCA with the same index technology.”

---

## 4. MongoDB

If pressed: **journey-style JSON** is weaker on your resume than **PEGA case + CJCM (P6)** and **Agentic workflow JSON in Postgres (P9)**. Prefer:

> “Long-running workflow and approval state for agents is in **Postgres on Agentic Universe**; case pause/resume was **PEGA + CJCM** on ordering.”

---

## 5. Aerospike (JD)

Honest:

> “I have not run Aerospike. On ordering we used **Cassandra** for high-cardinality key access by tenant/batch. A profile/frequency-cap workload would need atomic increment—I’d evaluate Aerospike or Redis with Lua; **consent and payment stay in Postgres/Cassandra** as system of record.”

Tie to **P5** fail-closed gates, not marketing caps you never built.

---

## 6. AI integration (P9 + P10)

| Path | Project | Latency |
|------|---------|---------|
| Multi-step RCA | **P10** LangGraph | Minutes; human reads output |
| Hosted agents | **P9** Agentic Universe | Async; **AWAITING_APPROVAL** |
| Inline on order create | **We did not** put LLM on **P1** submit path | Credit/OCR are bounded adapters |

**ScoringPort / CreativePort** in Java are the same boundary you already have with **CreditCheckPort** and **PegaCasePort**.

---

## 7. Kubernetes + AWS

**EKS + Docker + Jenkins/GitLab** from resume—deploy **bulk-batch-api** and **bulk-line-worker** (**P2**), not fictional campaign services.

See [07-cloud-cicd-observability.md](./07-cloud-cicd-observability.md).
