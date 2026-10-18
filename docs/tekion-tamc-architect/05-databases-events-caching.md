# Database design, caching, and event-driven systems (detailed)

---

## 1. Relational design (Postgres) — money, consent, workflow

### Campaign state machine

```sql
CREATE TABLE campaign (
  tenant_id    TEXT NOT NULL,
  campaign_id  TEXT NOT NULL,
  status       TEXT NOT NULL CHECK (status IN (
    'DRAFT','IN_REVIEW','APPROVED','SCHEDULED',
    'RUNNING','PAUSED','COMPLETED','CANCELLED'
  )),
  version      INT NOT NULL DEFAULT 1,
  PRIMARY KEY (tenant_id, campaign_id)
);

-- Legal transition example:
UPDATE campaign
SET status = 'RUNNING', version = version + 1
WHERE tenant_id = $1 AND campaign_id = $2
  AND status = 'APPROVED' AND version = $3;
-- 0 rows updated → 409 conflict
```

**Why version column:** optimistic locking without locking whole table.

### Ledger (co-op / dealer spend)

Append-only entries; balance is sum or projection.

```sql
CREATE TABLE ledger_entry (
  tenant_id     TEXT NOT NULL,
  entry_id      TEXT NOT NULL,
  budget_id     TEXT NOT NULL,
  attempt_id    TEXT NOT NULL,
  entry_type    TEXT NOT NULL CHECK (entry_type IN (
    'RESERVE','COMMIT','RELEASE','ADJUST'
  )),
  amount_cents  BIGINT NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (tenant_id, entry_id),
  UNIQUE (tenant_id, attempt_id, entry_type)
);
```

**Reserve in same transaction as accepting a billable send attempt:**

```sql
UPDATE budget
SET reserved_cents = reserved_cents + $amount
WHERE tenant_id = $t AND budget_id = $b
  AND reserved_cents + $amount <= limit_cents;
```

`0 rows` → reject send (or queue).

### Consent

```sql
CREATE TABLE consent (
  tenant_id    TEXT NOT NULL,
  consumer_id  TEXT NOT NULL,
  channel      TEXT NOT NULL,
  allowed      BOOLEAN NOT NULL,
  updated_at   TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (tenant_id, consumer_id, channel)
);
```

Send path: **read Postgres** (or short TTL cache with tenant in key). **Never** infer consent from Elasticsearch.

### Audience snapshot

Immutable for a launch:

```sql
CREATE TABLE audience_snapshot (
  tenant_id    TEXT NOT NULL,
  snapshot_id  TEXT NOT NULL,
  campaign_id  TEXT NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (tenant_id, snapshot_id)
);

CREATE TABLE audience_member (
  tenant_id    TEXT NOT NULL,
  snapshot_id  TEXT NOT NULL,
  member_id    TEXT NOT NULL,
  consumer_id  TEXT NOT NULL,
  PRIMARY KEY (tenant_id, snapshot_id, member_id)
);
```

CRM can change tomorrow; launch audit uses snapshot only.

---

## 2. NoSQL (Cassandra) — your ordering model

### Table for “recent orders by tenant”

```sql
CREATE TABLE orders_by_tenant (
  tenant_id text,
  created_at timestamp,
  order_id text,
  status text,
  PRIMARY KEY ((tenant_id), created_at, order_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

**Valid query:** `WHERE tenant_id = ? LIMIT 20`

**Invalid on this table alone:** `WHERE status = 'FAILED'` (full cluster scan).

**Fix:** second table `orders_by_tenant_status` with `PRIMARY KEY ((tenant_id, status), created_at, order_id)` fed by same write path.

### Partition key lesson (you taught vendors)

- Hot partition = one `tenant_id` with all traffic → add **bucket** `tenant_id + yyyyMMdd` for history.
- **10k orders per payload** is write burst—size batches, async fan-out, don’t single-row update a counter 10k times in one partition without thinking.

### Agentic Universe (Postgres, not Cassandra)

Run history, approval, workflow state—**transactions and joins** matter. Use that as “when SQL wins.”

---

## 3. Caching strategies

### 3.1 Cache-aside (campaign config)

```
read:
  v = redis.get(key)
  if v: return v
  v = db.load(campaign)
  redis.setex(key, 60, v)
  return v

invalidate on:
  campaign approve/pause/VIN suppress event → redis.del(key)
```

Key: `camp:v1:{tenantId}:{campaignId}`

### 3.2 Thundering herd

20 workers miss → 20 DB queries.

**Singleflight:**

```
if redis.set(lockKey, 1, NX, EX=5):
    v = db.load(...)
    redis.set(dataKey, v, EX=60)
    redis.del(lockKey)
else:
    sleep/backoff; read dataKey
```

### 3.3 What must NOT be cache-only

| Data | Why |
|------|-----|
| Ledger balance | Eviction → overspend |
| Consent (long-term) | Legal record in Postgres |
| Frequency cap counter | Unless atomic store (Aerospike/Redis Lua) |

Redis **rate limit** (50 ESP calls/s) is OK—see [08-good-to-have-stack.md](./08-good-to-have-stack.md).

### 3.4 Tekion LLD: TTL cache library

They asked candidates to **code** a TTL cache (Chennai SDE-2, 2026):

- Interface `get/put`
- Per-entry expiry
- Optional background sweeper
- Thread-safe `ConcurrentHashMap` + `ScheduledExecutor` or lazy expiry on read

Connect to your Redis usage: same semantics, different scale.

---

## 4. Event-driven systems

### 4.1 Your RabbitMQ bulk engine

| Step | Behavior |
|------|----------|
| Accept | Persist batch |
| Publish | One message per line |
| Consume | Idempotent on line id |
| Failure | DLQ + alert |
| Metric | accepted - completed |

### 4.2 Kafka mapping (JD + forum questions)

**Topic:** `campaign.launch.v1`  
**Key:** `tenantId:campaignId`  
**Consumer group:** `send-workers`  
**Commit:** after `delivery_attempt` row inserted (idempotent)

**At-least-once:**

```
retry from broker → same message → ON CONFLICT DO NOTHING on attempt_id → no second ESP call
```

**At-most-once (wrong for sends):**

```
commit offset before work → crash → message lost
```

**Exactly-once myth:** Network cannot guarantee; **at-least-once + idempotent effect** = indistinguishable for business.

### 4.3 Outbox vs Kafka transactions

| Mechanism | Solves |
|-----------|--------|
| Outbox + relay | DB commit aligned with publish |
| Kafka EOS (transactional producer) | Atomic write across Kafka partitions + offsets **inside Kafka app**—not your Postgres row |

Payments interview answer: **outbox** is industry standard for DB + Kafka.

### 4.4 Ordering

- Per **VIN**: partition by `vin` → `AVAILABLE` then `SOLD` processed in order.
- Handle out-of-order: store `version` or `event_time`; ignore stale `AVAILABLE` after `SOLD`.

### 4.5 Debug: producer OK, consumer silent (Tekion Mar 2026)

Checklist:

1. Consumer `group.id` matches deployment?
2. Subscribed topic name / cluster?
3. Stuck on one offset (poison pill loop)?
4. Partition assignment—all partitions have live consumer?
5. `max.poll.interval` exceeded processing huge batch?
6. ACL / SASL on consumer only?

---

## 5. Indexing and sharding (food delivery LLD follow-up)

Tekion asked: **which tables to index/shard?**

**Postgres:**

- `order(restaurant_id, status, created_at)` index for kitchen queue
- `booking(slot_id)` unique where status confirmed
- Shard by `city_id` or `tenant_id` when single DB saturates

**Never shard by low-cardinality `status` alone.**

---

## 6. OpenSearch / Elasticsearch role

- **Not** transactional store.
- Creative search, log RCA (your SRE service), operational explore.
- Send path uses Postgres snapshot + approved `template_id`.

---

## 7. Study drill

Write SQL + one Kafka message for:

1. Launch accepted → outbox → chunk published.
2. Duplicate chunk delivery.
3. Reserve budget fails mid-chunk.

Answers in [04-api-and-system-design.md](./04-api-and-system-design.md).
