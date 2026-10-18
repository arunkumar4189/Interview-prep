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
| **P2** bulk 10k+ lines | Async | RabbitMQ fan-out |
| **P4** large cart validate | Async | 202 + job |
| **P5** credit / OCR / auth | Sync with tight budget | Fail-closed on activation |
| **P9** agent creative step | Async + human approve | Not on order GraphQL path |
| **P10** RCA LangGraph | Async minutes | Human reads; no auto-fix prod |
| Catalog rule change mid-batch | Event or version check | Worker re-reads **ConfigMs** |

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
- Authorization: can this partner submit a bulk batch for this `tenant_id`? (**P2**)

Gateway is not a substitute for **row-level tenant checks** in SQL (`WHERE tenant_id = ?`).

---

## 8. Microservice boundaries you actually drew (P3 + P1 + P2)

| Service | Owns | Does not own |
|---------|------|--------------|
| **CartMs** (P3) | Cart lines, validation trigger | Payment capture |
| **ConfigMs** (P3) | Offers, rules, catalog reads | Order submission |
| **PaymentMs** (P3) | Payment intent, idempotency | PEGA case state |
| **Bulk batch API** (P2) | Batch accept, outbox | Per-line order logic |
| **Line worker** (P2) | Consume line, idempotent processing | GraphQL schema |
| **CXP aggregate** (P1) | GraphQL composition | Domain persistence |
| **Domain enrollment** (P5) | Enrollment aggregate | Credit bureau SDK |
| **CJCM / case** (P6) | Pause/resume orchestration | PEGA internals |
| **Adapters** | PEGA, credit, OCR, legacy quote (P5/P7) | Core order tables |

**Wrong boundary we rejected in review:** enrollment service importing PEGA client and calling it inside one giant `@Transactional` method.

**Tekion analogy (one line):** CartMs ≈ campaign definition; bulk batch ≈ launch fan-out; PaymentMs ≈ spend ledger—see [11-your-projects-catalog.md](./11-your-projects-catalog.md).

---

## 9. Exercise (from your domain)

**Prompt:** Partner bulk batch in flight; mid-batch catalog rule changes and line 500 becomes invalid.

Write using **P2/P4**:

1. Event or rule version on batch header.
2. Worker re-validates line against **ConfigMs** (**P3**) before order create.
3. Metric: invalid lines vs **dropout** (New Relic).

If interviewer insists automotive: map invalid line → “suppress send” using [09-tamc-domain-scenarios.md](./09-tamc-domain-scenarios.md).
