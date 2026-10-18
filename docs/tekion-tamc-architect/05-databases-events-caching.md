# Database design, caching, and event-driven systems (detailed)

**Examples from:** North Star + Cassandra (**P1**), bulk engine + RabbitMQ (**P2**), Cart/Config/Payment (**P3**), Agentic Universe Postgres (**P9**), SRE OpenSearch (**P10**).

Catalog: [11-your-projects-catalog.md](./11-your-projects-catalog.md).

---

## 1. Cassandra (P1) — orders and batch lines

### Orders by tenant (what you modeled)

```sql
CREATE TABLE orders_by_tenant (
  tenant_id text,
  created_at timestamp,
  order_id text,
  status text,
  batch_id text,
  PRIMARY KEY ((tenant_id), created_at, order_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

**Legal:** recent orders for SMB tenant.

**Illegal alone:** `WHERE status = 'FAILED'` across cluster → need `orders_by_tenant_status`.

### Batch line processing (P2)

```sql
CREATE TABLE batch_line_status (
  tenant_id text,
  batch_id text,
  line_id text,
  status text,
  last_error text,
  PRIMARY KEY ((tenant_id, batch_id), line_id)
);
```

Partition `(tenant_id, batch_id)` keeps one partner upload colocated.

**Teaching moment (you mentored vendors):** hot partition = one `tenant_id` with all history in one partition without bucketing—add `tenant_id + yyyyMMdd` for historical tables if needed.

---

## 2. Postgres (P9 Agentic Universe) — workflow you shipped recently

```sql
CREATE TABLE agent_run (
  run_id        UUID PRIMARY KEY,
  tenant_id     TEXT NOT NULL,
  workflow_id   TEXT NOT NULL,
  status        TEXT NOT NULL,  -- RUNNING, AWAITING_APPROVAL, COMPLETED, FAILED
  current_step  INT NOT NULL,
  version       INT NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL
);

CREATE TABLE agent_run_approval (
  run_id       UUID PRIMARY KEY,
  approved_by  TEXT,
  approved_at  TIMESTAMPTZ,
  decision     TEXT NOT NULL
);
```

**Why Postgres here:** human approval + audit trail + transactions—not a good fit for Cassandra on this workload.

**Pause/resume analogy (P6):** `AWAITING_APPROVAL` is like case paused—no downstream side effect until approval row exists.

---

## 3. Payment idempotency (P3 PaymentMs — Tekion forum question)

```sql
CREATE TABLE payment_intent (
  tenant_id         TEXT NOT NULL,
  payment_id        TEXT NOT NULL,
  idempotency_key   TEXT NOT NULL,
  order_line_ref    TEXT NOT NULL,
  status            TEXT NOT NULL,  -- CREATED, CAPTURED, FAILED
  PRIMARY KEY (tenant_id, payment_id),
  UNIQUE (tenant_id, idempotency_key)
);
```

Retry from partner or worker → same key → return existing `CAPTURED` row—**never** second charge for same line.

Ledger-style append (if finance required):

```sql
UNIQUE (tenant_id, attempt_id, entry_type)  -- RESERVE, COMMIT, RELEASE
```

Same pattern you would describe for co-op spend at Tekion; **your proof** is PaymentMs + bulk line reference.

---

## 4. Enrollment state (P5) — optimistic version

```sql
UPDATE enrollment
SET status = 'ACTIVE', version = version + 1
WHERE tenant_id = $1 AND enrollment_id = $2
  AND status = 'CREDIT_APPROVED' AND version = $3;
```

0 rows → concurrent rep and automated job conflict → 409 to client.

---

## 5. Caching (P3 ConfigMs + catalog reads)

### Config / product rules cache

**Problem:** ConfigMs read on every cart validation (**P4**).

```
key: config:v1:{tenantId}:{productOfferId}
TTL: 120s
invalidate: on config publish event from admin tool
```

### Thundering herd on hot offer

During promotion, 50 workers miss cache → 50 ConfigMs DB hits.

**Singleflight** with Redis `SET lock:config:... NX EX 5`—same pattern you’d use on any hot catalog key.

### What we did **not** cache as sole source of truth

- Payment status (**P3**)
- Final enrollment activation (**P5**)
- Batch acceptance record (**P2**)

Redis eviction must not create a “free” duplicate payment.

---

## 6. Event-driven (P2 RabbitMQ — your production path)

| Step | Bulk engine |
|------|-------------|
| 1 | API stores `bulk_batch` + outbox |
| 2 | Relay publishes to `bulk.order.lines` |
| 3 | Consumer idempotent on `(batch_id, line_id)` |
| 4 | Poison → DLQ; ops replay after fix |
| 5 | Metric: lines accepted − lines terminal = **dropout** |

### Mapping to Kafka (JD / Tekion question — honest)

> “I did not operate Kafka for this path; I operated **RabbitMQ** with the same rules: **at-least-once delivery**, **idempotent consumer**, **DLQ**, lag/depth alerts. Partition key would be `tenant_id:batch_id` if we moved to Kafka.”

### Outbox

Batch row + outbox row in **one transaction**—the fix we applied when reviewing dual-write on ordering services.

### Ordering

If line 2 depends on line 1 (rare in bulk)—use same routing key / single consumer for that batch; otherwise lines are independent (**why one bad line did not fail 9,999 others**).

---

## 7. OpenSearch (P10) — not transactional

SRE Triaging queries logs by `traceId`, time, service name.

**Not used for:** deciding if payment captured or enrollment active.

Index is **evidence** for LangGraph nodes; domain state stays in Postgres / Cassandra.

---

## 8. Indexing / sharding (Tekion LLD follow-up — answer from P1/P2)

**Which tables to index?**

- `batch_line_status (tenant_id, batch_id, status)` for ops dashboard
- `payment_intent (tenant_id, order_line_ref)` for support lookup
- Cassandra: primary key is the index—design query first

**Shard when:** single Cassandra/Postgres node saturates on **tenant_id** write rate—split by tenant shard or move batch headers to dedicated store.

---

## 9. Drill

Write from memory:

1. Accept `bat_9001` with idempotency key (SQL + outbox).
2. Duplicate Rabbit message for `line_0042`.
3. Credit timeout on line 99 only—batch continues.

Answers are in [04-api-and-system-design.md](./04-api-and-system-design.md) Part B.
