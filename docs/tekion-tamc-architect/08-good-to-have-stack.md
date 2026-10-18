# Good-to-have stack: Kafka, Redis, Elasticsearch, MongoDB, Aerospike (detailed)

Honest baseline: **RabbitMQ, Redis, Cassandra, Postgres, OpenSearch** on your resume. **Kafka, MongoDB, Aerospike**—study patterns + say what you have not operated.

---

## 1. Kafka

### 1.1 Components (Tekion asked explicitly, Mar 2026)

| Term | Role |
|------|------|
| Topic | Named log |
| Partition | Ordered sub-log; parallelism unit |
| Offset | Consumer position in partition |
| Producer | Appends records |
| Broker | Stores partitions |
| Consumer group | One consumer per partition max (in group) |
| Replication factor | Durability across brokers |
| ISR | In-sync replicas eligible for leader |

### 1.2 Producer settings (interview)

- `acks=all` — wait for ISR ack
- `enable.idempotence=true` — dedupe producer retries **to partition**
- **Does not** dedupe at ESP or email recipient

### 1.3 Consumer

```properties
spring.kafka.consumer.enable-auto-commit=false
```

Commit **after** idempotent DB write.

### 1.4 Topic design for campaign

```
topic: campaign.launch.v1
partitions: 48 (scale workers)
key: tenantId + ":" + campaignId
value: { eventId, launchId, chunkIndex, recipientIds[] }
```

DLQ: `campaign.launch.v1.dlq`

### 1.5 At-least-once vs at-most-once (HM round, Kashish Medium)

| Mode | Commit order | Risk |
|------|--------------|------|
| At-most-once | Before process | Lost messages |
| At-least-once | After process | Duplicates (fix with idempotency) |

### 1.6 Debug consumer not consuming

See checklist in [05-databases-events-caching.md](./05-databases-events-caching.md).

### 1.7 Your bridge sentence

> I ran fan-out and idempotent consumers on **RabbitMQ** for 10k-line batches. On Kafka I’d use the same business keys and outbox relay; I’d operate lag and DLQ the same way I operated queue depth and DLQ.

---

## 2. Redis

### 2.1 Use cases in TAMC

| Pattern | Structure |
|---------|-----------|
| Campaign config cache | String JSON, TTL 60s |
| Live VIN set per rooftop | SET `vins:{tenant}:{rooftop}` |
| ESP rate limit | ZSET sliding window + Lua |
| Singleflight lock | `SET lockKey NX EX 5` |
| Delay retry queue | ZSET score = next_attempt_epoch |

### 2.2 Sliding window rate limit (50 req/s)

Lua (atomic):

```lua
-- KEYS[1] = rate:{tenant}:{channel}
local now = tonumber(ARGV[1])
local window = 1000 -- ms
local limit = 50
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, now - window)
local n = redis.call('ZCARD', KEYS[1])
if n >= limit then return 0 end
redis.call('ZADD', KEYS[1], now, ARGV[2])
redis.call('PEXPIRE', KEYS[1], 2000)
return 1
```

### 2.3 Why Redis is not the primary DB (Tekion parking-lot round)

- No multi-row ACID for ledger.
- Eviction / restart can drop data.
- Use as **cache / rate limit / rebuildable projection**.

---

## 3. Elasticsearch / OpenSearch

### 3.1 Your production use

**SRE Triaging Service:** query logs by `traceId`, time range, service—feed LangGraph node.

### 3.2 TAMC use

- **Creative library search** (full text)
- **Audience explore** (complex filters)—index built from CRM export, not live send path

### 3.3 Index design sketch

```json
PUT creatives_v1/_doc/cr_1
{
  "tenantId": "acme",
  "campaignId": "cmp_44",
  "channel": "email",
  "body": "...",
  "approvalStatus": "APPROVED",
  "updatedAt": "2026-10-18T12:00:00Z"
}
```

Query: `bool must: [term tenantId, match body]`

### 3.4 Pitfall

Search index is **eventually consistent** (~1s refresh). Approved creative for send comes from **Postgres `template_id`**, not search hit.

---

## 4. MongoDB

Not on your resume. One **credible** use:

**Journey definition** (nested steps, branches) while product changes shape weekly.

```json
{
  "_id": "j_19",
  "tenantId": "acme",
  "version": 4,
  "steps": [
    { "id": "s1", "type": "email", "templateId": "t_9" },
    { "id": "s2", "type": "wait", "hours": 48 },
    { "id": "s3", "type": "sms", "when": "opened s1" }
  ]
}
```

**Running instance** of journey for a consumer → still **Postgres** (`journey_run` with state, current_step).

Money, consent, ledger → Postgres.

---

## 5. Aerospike

Not operated by you. JD calls it out for **low-latency profile + caps**.

### 5.1 Access pattern

- Key: `tenant:consumerId`
- Bins: `consentEmail`, `consentSms`, `capEmail`, `capSms`, `windowStart`
- **Generation** for optimistic concurrency:

```
read record (gen=17)
if capEmail < limit:
  write with gen check
else reject
on gen conflict: re-read
```

### 5.2 vs Redis vs Postgres for caps

| Store | Caps |
|-------|------|
| Postgres | Correct, slow at billions of reads |
| Redis | Fast; risky if only copy of daily cap |
| Aerospike | Fast + large cardinality + atomic gen check |

Postgres remains **audit**; Aerospike is **decision cache** refreshed from events.

---

## 6. Comparison table (quick revision)

| Tech | You | TAMC role |
|------|-----|-----------|
| RabbitMQ | Prod bulk engine | Same patterns as Kafka |
| Kafka | Study + forum Q | Launch pipeline |
| Redis | Resume | Rate limit, config, VIN set |
| Cassandra | Prod orders | Historical sends by tenant key? |
| Postgres | Agentic + ledger | Campaign, consent, money |
| OpenSearch | SRE logs | Logs + search |
| Mongo | Study only | Journey defs optional |
| Aerospike | Study only | Profile/cap hot path |
