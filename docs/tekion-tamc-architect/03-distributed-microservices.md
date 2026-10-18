# Distributed systems and microservices (detailed)

---

## 1. What “microservices” means in an interview

Not “many small REST apps.” It means:

1. **Independent deployability** — change consent service without redeploying campaign UI BFF.
2. **Data ownership** — each service owns its tables; no shared DB as integration.
3. **Failure isolation** — ESP down does not take down campaign CRUD.
4. **Explicit contracts** — APIs + events with versioning.

**Anti-pattern:** 40 services sharing one Oracle schema—you have distributed monolith coupling.

---

## 2. Your reference architecture (Verizon ordering)

Map every Tekion box to this:

| Component | Responsibility | Integration style |
|-----------|----------------|-------------------|
| GraphQL aggregate | Experience composition | Sync, deadlines, circuit breakers |
| Domain microservices | Business rules + store | Sync API + async events |
| Case/CJCM | Long-lived workflow | Commands + state machine |
| Adapters | PEGA, credit, OCR, legacy quote | Sync with timeout; async retry |
| Bulk engine | 10k payload fan-out | Message queue, idempotent consumers |

**Strangler fig:** Quote-to-order did not rewrite legacy in one night—adapter + new path + retire manual step.

TAMC strangler: start with **campaign + consent + one channel adapter**; add VIN suppress when inventory feed exists.

---

## 3. Sync vs async decision tree

```
Is the user waiting for the result in the same HTTP request?
├─ YES → Is it < 200ms p99 and no vendor SLA risk?
│         ├─ YES → sync call with timeout + fallback
│         └─ NO  → accept + async job + poll/webhook
└─ NO  → message/event; idempotent consumer
```

**Examples:**

| Operation | Sync/async | Why |
|-----------|------------|-----|
| Create campaign draft | Sync | Small DB write |
| Launch 200k sends | Async | Minutes of work |
| Credit check for activation | Sync with tight budget OR async | Product: fail-closed vs wait page |
| Generate LLM email body | Async + human approve | Seconds–minutes |
| Scoring on send decision | Sync ≤25ms | Rules fallback on timeout |
| VIN sold suppress | Event-driven | Inventory event → suppress projection |

---

## 4. Distributed transactions: what not to do

**Two-phase commit across Postgres + Kafka + Google Ads** — operationally fragile; interviewers want **sagas / outbox**.

### Outbox (preferred for launch)

1. TX1: `launch` + `outbox` commit.
2. Relay publishes to Kafka.
3. Consumer: idempotent `delivery_attempt` insert → call ESP.

### Saga (payment + inventory + notify)

If payment succeeds and inventory fails:

- **Do not** silently delete payment row.
- States: `PAYMENT_CAPTURED`, `INVENTORY_FAILED`, compensating action (refund job) or manual queue.

Tekion senior onsite (Apr 2026) asked exactly this pattern.

---

## 5. Consistency models (say precisely)

| Pattern | Guarantee | Your example |
|---------|-----------|--------------|
| Strong (single row) | Linearizable update on one aggregate | `UPDATE budget WHERE ...` |
| Eventual | Projections catch up | Receipt count on dashboard |
| Causal | Related events keep order | VIN `SOLD` before suppress applied |

**CAP one-liner for Tekion managerial rounds:**

- **Postgres (ledger, booking slot):** CP—sacrifice availability on partition for constraint correctness.
- **Cassandra (order history by tenant):** AP—tunable consistency; design for **query-specific** tables.
- **Redis cache:** not source of truth; rebuild from DB/events.

---

## 6. Failure modes checklist (draw on whiteboard)

For each external dependency:

| Failure | Detection | Mitigation |
|---------|-----------|------------|
| Timeout | Client timer, 504 rate | Circuit breaker, retry with same idempotency key |
| Slow | p99 latency | Bulkhead thread pool / semaphore |
| Duplicate delivery | Same message twice | Unique `attempt_id` |
| Partial batch | 499/500 of 500 sent | Per-chunk status; replay DLQ |
| Broker lag | Consumer lag metric | Scale workers; cap accept rate |
| Cache stampede | DB spike on hot key | Singleflight |
| Wrong tenant data | Missing tenant in key | Code review rule + integration test |

---

## 7. API gateway vs service auth (your APIGEE work)

**Gateway (Akamai/APIGEE):** TLS, WAF, rate limit, API key/OAuth, request size.

**Service still validates:**

- Tenant from token claims matches `tenant_id` in path/body.
- Idempotency key scope per tenant.
- Authorization: can this principal launch this campaign?

Gateway is not a substitute for **row-level tenant checks** in SQL (`WHERE tenant_id = ?`).

---

## 8. Microservice boundaries for TAMC (study design)

| Service | Owns | Does not own |
|---------|------|--------------|
| Campaign | Definition, approval state, schedule | Sending email |
| Audience | Segment definition, **snapshot** materialization | CRM master (adapter) |
| Consent | DNC, channel opt-in record | ESP template |
| Inventory projection | Live VIN set per rooftop | DMS (adapter) |
| Delivery orchestrator | Chunks, attempt ids, receipts | Creative text |
| Ledger | Reserve/commit/release | Google billing file (reconcile) |
| Scoring adapter | Model call, timeout, fallback | Campaign rules |

**Wrong boundary:** “AIService” that imports campaign entities and calls OpenAI inside `@Transactional`.

---

## 9. Exercise

**Prompt:** Dealer launches campaign; 5 minutes later VIN sells; ESP still has in-flight batch.

Write:

1. Events (names + keys).
2. Where last-chance VIN check happens.
3. Metric for suppress lag.

Answer skeleton in [09-tamc-domain-scenarios.md](./09-tamc-domain-scenarios.md).
