# Reported Tekion interview questions — detailed answers

**Sources:** Interview Experiences (~32 write-ups), LeetCode Discuss, Reddit r/leetcode, LinkedIn/Medium through 2026. Loops differ by role (SDE-1/2, SSE, Staff, Architect). **Not official Tekion questions.**

**How to answer every question:** Clarify → NFRs with numbers → one design → anchor to **P1–P12** ([catalog](./11-your-projects-catalog.md)). Do not invent a generic “campaign service.”

**Answer rhythm (60–90 seconds per subsection):**

1. “Let me clarify scale and consistency needs…”
2. “I’ll use a system I ran in production…”
3. Draw or narrate boxes + one failure mode + one metric.

---

## Table of contents

1. [Kafka & messaging](#1-kafka--messaging)
2. [Payments & idempotency](#2-payments--idempotency)
3. [Policy engine (LLD)](#3-policy-engine-lld)
4. [System design: bulk / food delivery](#4-system-design-bulk--food-delivery)
5. [System design: booking (service center, IRCTC, seats)](#5-system-design-booking)
6. [System design: chat, e-commerce, Slack](#6-system-design-chat-e-commerce-slack)
7. [System design: cache, URL filter](#7-system-design-cache-url-filter)
8. [System design: AI agent platform & chat](#8-system-design-ai-agent-platform--chat)
9. [LLD machine-coding prompts](#9-lld-machine-coding-prompts)
10. [DSA coding rounds](#10-dsa-coding-rounds)
11. [Java, concurrency, Redis vs DB](#11-java-concurrency-redis-vs-db)
12. [Manager & behavioral](#12-manager--behavioral)
13. [Architect loop: your risks](#13-architect-loop-your-risks)

---

## 1. Kafka & messaging

*Reported: SDE-2 Bangalore Mar 2026 (LLD), HM round Kashish Medium 2025 (at-least-once).*

### Q1.1 Explain Kafka and its components

**Clarify:** “Are you asking for producer/consumer basics, or how we guarantee no duplicate side effects?”

**Conceptual answer (2 min):**

| Component | What it does |
|-----------|----------------|
| **Topic** | Named log of records |
| **Partition** | Ordered sub-log; unit of parallelism |
| **Offset** | Consumer position in a partition |
| **Broker** | Stores partitions, serves produce/fetch |
| **Producer** | Appends records; chooses partition via key |
| **Consumer group** | Consumers share work; one consumer per partition at a time |
| **Replication / ISR** | Durability; `acks=all` waits for in-sync replicas |

**Your production anchor (P2):**

> “I operated **RabbitMQ** on the **bulk order engine**: one accepted payload of **10,000+ order lines** fan-out to per-line messages. The business rules are identical on Kafka: **at-least-once delivery**, **idempotent consumer** on `(batch_id, line_id)`, **DLQ** for poison lines, alert on queue depth / lag.”

**Kafka mapping you state explicitly:**

```
Rabbit queue bulk.order.lines  →  topic bulk.order.line.v1
routing key tenant.batch     →  partition key tenantId:batchId
manual ack after DB write      →  commit offset after line row durable
```

**Follow-up they ask:** “Exactly-once?”

> “End-to-end exactly-once does not exist across network + DB + vendor. We used **at-least-once + unique business key**. Kafka EOS only helps inside a transactional Kafka app—not PaymentMs or PEGA.”

---

### Q1.2 Producer publishes; consumer does not consume — how do you debug?

**Answer as an ordered checklist (say you used this on Rabbit; same on Kafka):**

1. **Consumer group / subscription** — Is the app subscribed to the correct topic name and cluster? Typos between staging and prod are common.
2. **Consumer running?** — Pods up, no crash loop. Check logs for `Connection refused` to broker.
3. **Partition assignment** — `kafka-consumer-groups.sh --describe`: are all partitions assigned? Rebalance stuck? New consumer joined and stole partitions mid-batch?
4. **Offset position** — Consumed to end? `LOG_END_OFFSET` vs `CURRENT_OFFSET`. Producer writing to topic A, consumer on topic B.
5. **Poison message** — Consumer throws on one offset, retries forever, appears “stuck.” Fix: skip to DLQ after N tries, alert on single-partition lag.
6. **ACL / auth** — Producer credentials ≠ consumer credentials.
7. **Serialization** — Consumer deserializer fails silently in bad code paths—check error rate metric.

**P2 story:**

> “On bulk processing we paged on **queue depth**, not ‘consumer alive.’ Depth growing with flat completion rate meant workers failing or not bound to queue. **New Relic dropout** (accepted lines − completed lines) caught silent loss when consumer was up but not committing line status.”

---

### Q1.3 At-least-once vs at-most-once (HM round)

| Mode | When offset/ack committed | Failure effect |
|------|---------------------------|----------------|
| **At-most-once** | Before processing | Message lost, no duplicate |
| **At-least-once** | After processing succeeds | Duplicate possible → need idempotency |

**P3 example:**

> “For **PaymentMs**, we must be **at-least-once** on the worker with **`UNIQUE (tenant_id, idempotency_key)`** on payment intent. At-most-once would lose a payment message and under-bill.”

**Config names (if they push):** `enable.auto.commit=false`, commit after insert; Rabbit `manual ack` after Cassandra/DB write.

---

## 2. Payments & idempotency

*Reported: SDE-2 Bangalore Mar 2026; Senior onsite Apr 2026 (payment ok, later step fails).*

### Q2.1 Retries must not deduct balance twice

**Clarify:**

- Who generates idempotency key? **Client or upstream worker** (same key on retry).
- Retention? Longer than max retry window (24–72h typical).

**Design (P3 PaymentMs + P2 line worker):**

```sql
CREATE TABLE payment_intent (
  tenant_id       TEXT NOT NULL,
  payment_id      TEXT NOT NULL,
  idempotency_key TEXT NOT NULL,
  order_line_ref  TEXT NOT NULL,  -- batch_id + line_id
  amount_cents    BIGINT NOT NULL,
  status          TEXT NOT NULL,  -- CREATED, CAPTURED, FAILED
  provider_ref    TEXT,
  created_at      TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (tenant_id, payment_id),
  UNIQUE (tenant_id, idempotency_key)
);
```

**Flow:**

1. Worker processes line → derive `idempotency_key = hash(batchId, lineId, "payment")`.
2. `INSERT ... ON CONFLICT (tenant_id, idempotency_key) DO NOTHING` or catch unique violation → **SELECT** existing row.
3. If status already `CAPTURED` → return success (retry safe).
4. If `CREATED` → call payment provider with **same** provider idempotency key.
5. Update to `CAPTURED` only after provider success.

**State machine (say out loud):**

```
CREATED → CAPTURED (success)
CREATED → FAILED (hard decline)
CAPTURED → (terminal, no delete)
```

**Outbox (P2 pattern) if events needed:**

Same transaction: `payment_intent` row + `outbox` row → relay publishes `PaymentCaptured` → downstream analytics idempotent on `payment_id`.

**What not to say:** “We’ll just make the API idempotent” without DB unique constraint—two pods still double-charge.

---

### Q2.2 Payment succeeded; inventory / notification / order step failed

*Tekion scenario: multi-step pipeline.*

**Clarify:** Is payment reversible? Is partial batch OK?

**Answer (P2 + P3 + P6):**

1. **Payment row stays `CAPTURED`** — never DELETE to “fix” state.
2. Line status → `FULFILLMENT_FAILED` with `failure_reason=PEGA_TIMEOUT` (or order domain error).
3. **Other lines in batch continue** — 10,000-line payload design: line 99 failure must not roll back line 1–98.
4. **Compensation:** async job `REFUND_REQUESTED` or manual ops queue for that `payment_id`; new idempotency key on refund attempt.
5. **Case (P6):** if PEGA case required and fails, enrollment/order stays in explicit state—not “unknown success.”

**Saga narrative:**

> “We avoided distributed 2PC across PaymentMs, order domain, and PEGA. Each step is **local transaction + event**. Failure after payment is a **business exception state**, not a rolled-back payment row.”

**Metric:** `lines_payment_captured_but_line_failed` — alert if > 0 sustained.

---

## 3. Policy engine (LLD)

*Reported: Reddit r/leetcode, Tekion hiring drive Apr 4 2026.*

### Q3.1 ALLOW / DENY from AND/OR condition tree

**Their sample policy:**

```
(creditScore >= 720 AND accountStatus == "Active") OR (receivableDays < 30)
```

**Your anchor (P5):**

> “On **SMB enrollment** we had **automated credit** and **adaptive auth** with fail-closed gating before activation. I’d implement Tekion’s policy engine the same way: **pure evaluator** over **versioned policy** + **fact snapshot**, not `eval()` on strings.”

**Data model:**

```java
sealed interface Condition permits And, Or, Compare { }
record And(Condition left, Condition right) implements Condition {}
record Or(Condition left, Condition right) implements Condition {}
record Compare(String field, Op op, Object literal) implements Condition {}

record Policy(String id, int version, Condition root) {}
record EvaluationRequest(String policyId, Map<String, String> facts) {}
enum Decision { ALLOW, DENY }
```

**Evaluator (recursive):**

```java
Decision evaluate(Condition c, Map<String, String> facts) {
  return switch (c) {
    case And(var l, var r) -> 
        evaluate(l, facts) == ALLOW && evaluate(r, facts) == ALLOW ? ALLOW : DENY;
    case Or(var l, var r) -> 
        evaluate(l, facts) == ALLOW || evaluate(r, facts) == ALLOW ? ALLOW : DENY;
    case Compare(var field, var op, var lit) -> 
        compare(facts.get(field), op, lit) ? ALLOW : DENY;
  };
}
```

**Type coercion:** Their sample had `"creditScore": "750"` as string—parser should coerce numerics or deny with clear error.

**Persistence:** `policy_id`, `version`, `json_tree`; enrollment stores `policy_version` + `decision` on row for audit.

**Tests:** P1 OR branch true; AND one false → DENY; missing field → **DENY** (fail-closed for activation).

**Follow-up:** “Add NOT?” → `Not(Condition child)` with short-circuit.

---

## 4. System design: bulk / food delivery

*Reported: Chennai SDE-2 Jun 2026 (Swiggy HLD + TTL cache LLD); Bangalore Mar 2026 (food delivery LLD + indexing/sharding).*

### Q4.1 Design food delivery (Swiggy-like)

**Do not design Swiggy from scratch as your only story.**

**Opening:**

> “I’ll map this to the **bulk order engine (P2)** I architected: high-volume **order pipeline**, **async fan-out**, **per-unit idempotency**, **PaymentMs (P3)**. Food-delivery entities differ, but the **throughput and failure patterns** are what I operated.”

**Clarify (food delivery):**

- Orders per minute per city?
- Payment before kitchen confirm?
- Restaurant can close after accept?

**If they insist on food nouns — after P2 anchor:**

| Food | Your system |
|------|-------------|
| Place order | Accept bulk batch / single order API |
| Kitchen / restaurant | Downstream fulfillment adapter |
| Delivery dispatch | Async worker pool |
| Payment | PaymentMs idempotency |
| Restaurant closes after order | Line invalid mid-batch → cancel + refund saga (Q2.2) |

**NFRs (example numbers to state):**

| NFR | Target |
|-----|--------|
| Place-order API p99 | < 300 ms |
| Peak orders/min | 5k/city (ask them) |
| Duplicate charge | 0 |
| Order visible to user | < 2 s after accept |

**Core services (food):** user, restaurant/menu (read-heavy), order, payment, dispatch (async).

**Write path:** `POST /orders` → persist `order` PENDING → payment authorize → restaurant confirm → DISPATCHED.

**P2 parallel:** batch accept → lines → workers; dropout metric.

**Edge case (reported):** restaurant closes after order → state `CANCELLED`, refund job, notify user; do not delete payment row.

**Indexing/sharding (Mar 2026 follow-up):**

- Index `orders(restaurant_id, status, created_at)` for kitchen queue.
- Shard by `city_id` or `tenant_id` when single DB saturates.
- **Not** shard by `status` alone.

---

### Q4.2 Design TTL cache library (Chennai LLD — code expected)

**Anchor P3 ConfigMs:**

> “We cached **catalog/config** with TTL in Redis; this LLD is the in-process library equivalent.”

**Interfaces:**

```java
public interface TtlCache<K, V> {
  void put(K key, V value, Duration ttl);
  Optional<V> get(K key);  // empty if missing or expired
  void remove(K key);
}
```

**Implementation points:**

- `ConcurrentHashMap<K, Entry<V>>` where `Entry` holds `value`, `expiresAtEpochMs`.
- **Lazy expiry** on `get()`.
- **Optional** `ScheduledExecutorService` sweeper every N minutes for memory reclamation.
- Thread-safety: all public methods synchronized or CHM + atomic replace.
- **Design pattern:** Strategy for eviction if max size (LRU) — Tekion may ask extension.

**Complexity:** get/put O(1) average.

---

## 5. System design: booking

*Reported: Senior SWE Jul 2026 (car service center — reject on HLD); SSE LeetCode IRCTC; airplane booking; parking lot LLD.*

### Q5.1 Car service-center booking (search + slot + concurrency)

**Clarify:** slot granularity (30 min?), same vehicle multiple bookings?, geo radius?

**NFRs:**

| NFR | Target |
|-----|--------|
| Search nearby | p99 < 200 ms (cached) |
| Book slot | strong consistency — no double book |
| Availability | 99.9% |

**Architecture:**

1. **Search read model** — centers indexed by geo (Postgres PostGIS / ES). Cache hot cities in Redis. *Your tie-in:* **ConfigMs catalog cache (P3)**.
2. **Slot table** — `slot(center_id, start_time, capacity)` or one row per physical bay.
3. **Booking write** — single transaction:

```sql
INSERT INTO booking (booking_id, center_id, slot_start, customer_id, status)
SELECT ..., 'CONFIRMED'
WHERE NOT EXISTS (
  SELECT 1 FROM booking
  WHERE center_id = $1 AND slot_start = $2 AND status = 'CONFIRMED'
);
-- 0 rows inserted → 409 SLOT_TAKEN
```

**Or:** `UPDATE slot SET remaining = remaining - 1 WHERE ... AND remaining > 0`.

**Your anchor:**

> “This is the same concurrency class as **PaymentMs budget reserve (P3)** and **case version (P6)**: conditional SQL or optimistic version, not read-then-write in Java.”

**Why candidates failed (per write-up):** drew search + DB but no **race** story.

**HA:** multi-AZ DB, stateless API, idempotent `Idempotency-Key` on book API.

---

### Q5.2 IRCTC — search and booking only

**Search:** trains/runs read-only, heavy QPS → cache + ES optional, CDN for static schedule.

**Booking:** **seat hold** is the hard part:

- Hold: `INSERT hold (seat_id, user_id, expires_at)` unique on `seat_id` where active.
- Payment within 10 min → `CONFIRMED`; else expire hold via job.
- **P3 parallel:** payment idempotency on confirm.

---

### Q5.3 Airplane seat selection

**Entities:** Flight, Seat, Hold, Booking.

**Flow:** hold seat (TTL 10 min) → payment → confirm. Unique `(flight_id, seat_no)` for active hold/confirmed.

**Your line:** Same as service center + **PaymentMs**.

---

### Q5.4 Parking lot LLD

**Reported:** SSE Bangalore, SDE-1 offer round — Redis, parallelism vs concurrency.

**Classes:** `ParkingLot`, `Floor`, `Spot` (type: compact/large), `Ticket`, `PaymentService`.

**Entry:** assign smallest fitting spot; **transaction** on spot occupancy.

**Exit:** compute fee; **payment idempotency** `pay:{ticket_id}:{amount}`.

**Follow-up:** notify user 1 hour after park — delayed queue (Rabbit delayed / Redis ZSET) — *same delayed work pattern as async line retry (P2)*.

**Redis:** cache free spot counts per floor; **not** source of truth—DB is.

---

## 6. System design: chat, e-commerce, Slack

*Reported: LinkedIn SDE-2 (web chat + e-commerce, architect-led); Apr 2026 Slack LLD+HLD.*

### Q6.1 Web chat application (RADIO)

**Requirements:** 1:1 and group, online presence, message history, search, read receipts (ask).

**Architecture:**

- **WebSocket gateway** (sticky sessions or Redis pub/sub for fan-out).
- **Message service** — append-only messages `channel_id, seq, body, sender`.
- **Presence** — Redis: `user_id → online`, TTL heartbeat.
- **Search** — ES/OpenSearch index message body — **your P10** log search experience.
- **Storage** — Cassandra or Postgres partitioned by `channel_id`.

**P8:** APIGEE for auth, rate limits; service validates `tenant_id` / user owns channel.

**Optimization:** write path ack after durable store; push via socket async.

---

### Q6.2 E-commerce platform (HM round)

**Cover:** catalog (read scale), cart (**P3 CartMs**), checkout, payment (**P3**), order (**P1**), inventory reservation saga.

**Your shortcut:** “Cart/checkout/payment split is how we built **CartMs, ConfigMs, PaymentMs**; North Star extended that to GraphQL aggregate.”

**Security/i18n (reported):** APIGEE + Akamai (**P8**); separate locale on product read model.

---

### Q6.3 Slack LLD + HLD (same day)

**LLD:** `Workspace`, `Channel`, `Message`, `UserMembership`, `MessageRepository`.

**HLD:** fan-out to online channel members via pub/sub; persistence before ACK; unread counts as derived Redis counters.

**Optional:** they asked AI agent to generate code — you can describe **Agentic Universe (P9)** approval before trusting generated code in prod.

---

## 7. System design: cache, URL filter

*Reported: Senior onsite Apr 2026 (distributed cache); Staff Dec 2025 (URL filtering + trie).*

### Q7.1 Distributed cache

**Clarify:** read/write ratio, cache size, eviction policy, HA?

**Design:**

- **Consistent hashing** across cache nodes; replicas for HA.
- **LRU/LFU** eviction; TTL per key.
- **Cache aside:** read miss → load DB → set; write → invalidate or update.
- **Stampede:** singleflight — **ConfigMs (P3)**.

**Your scale story:** Redis for catalog on ordering, not for payment truth.

**Staff follow-up:** mention **trie** for prefix URL blocklist (below).

---

### Q7.2 URL filtering service

**Requirements:** allow/deny URLs; prefix rules (`*.bad.com`).

**Structure:**

- Exact URLs → hash set.
- Prefix rules → **trie** of path segments.
- Full list in DB; **local trie cache** refreshed on pub/sub.

**P8:** edge WAF (Akamai) for coarse block; app-level filter for tenant-specific rules.

---

## 8. System design: AI agent platform & chat

*Reported: Staff Software Engineer AI Sep 2026 (agent platform, ChatGPT LLD, RAG eval, resource intervals DSA).*

### Q8.1 AI agent execution platform (HLD) — **strongest match: P9 + P10**

**Their prompt (summary):** long-running tools, human approval, workflows (branch/loop), sandboxes, fire-and-forget vs live watch, no data loss on crash.

**Clarify:** max run duration, tenant isolation, tool types (HTTP, shell)?

**Architecture (Agentic Universe — P9):**

```
[Client] → [API FastAPI] → Postgres (agent_run, step_log, approval)
                ↓
         [Worker pool] → LangGraph / workflow engine
                ↓
         [MCP tool adapters] — logs, Git, Jira, K8s (P10 patterns)
```

**Crash safety:**

- After each step: `UPDATE agent_run SET current_step=n, state=json, version=v+1`.
- On restart: resume from `current_step`, not from beginning.
- Tool calls **idempotent** where possible (read-only log fetch).

**Human approval:**

- State `AWAITING_APPROVAL`; no side-effect tools until `agent_run_approval` row exists.

**Fire-and-forget vs watch:**

- POST returns `run_id`; GET `/runs/{id}` or SSE for step stream.

**Sandbox pool:** max concurrent runs per tenant; queue excess (backpressure — **P2** worker semaphore same idea).

**Do not** put this graph on **P1** GraphQL order submit path.

---

### Q8.2 ChatGPT-style multi-tenant chat (LLD)

**Entities:** `Workspace(tenantId)`, `Conversation`, `Message`, `User`.

**Streaming:** store assistant message incrementally or post-hoc; SSE to client.

**Tenant isolation:** `tenant_id` on every row; row-level security in Postgres.

**Search history:** OpenSearch (**P10**) on message index.

**Edit message:** new `message_version`; model regen async job, not blocking send API.

---

### Q8.3 RAG eval pipeline

**Your anchor (resume GenAI eval):** groundedness, citations, instruction-following, unsafe actions.

**Pipeline:**

1. Golden dataset: `(question, expected_sources, rubric)`.
2. Run retrieval + generation frozen model version.
3. Auto-score + human spot-check.
4. CI gate: regression > ε blocks promote model/prompt version.

**P10:** SRE service proves multi-source retrieval; eval is offline batch, not user-facing latency.

---

### Q8.4 Maximum resource usage over intervals (DSA — Staff AI)

Intervals `[1,4]→3`, `[3,5]→2`, `[4,6]→4` → peak **6** at t=4.

**Approach:** sort by start; min-heap on end times; sweep adding resources, or difference array on timeline.

*Tie-in:* sizing **bulk worker pool (P2)** for peak concurrent downstream calls.

---

## 9. LLD machine-coding prompts

### Q9.1 Elevator (Staff Dec 2025)

**Classes:** `Elevator` (id, floor, direction, state), `Controller`, `Request` (source floor, dest, type EXTERNAL/INTERNAL).

**Flow:** external press → controller picks elevator (nearest/SCAN) → move → open door.

**State diagram:** IDLE → MOVING → DOOR_OPEN → IDLE.

**Concurrency:** lock per elevator when updating floor queue.

---

### Q9.2 Log4j-style logging (Senior SWE Jul 2026)

**Interfaces:** `Logger`, `Appender` (Console, File), `Layout`, `Level` enum.

**Async:** `BlockingQueue` + background thread drains appenders — **backpressure** if queue full (drop or block policy).

**Your tie:** observability on **P1** ordering; never log PII/full cart in prod (**P12** board rule).

---

### Q9.3 LFU cache (Kashish Medium LLD)

LeetCode 460 pattern: `Map key→Node`, `Map freq→DLL of keys`, O(1) get/put.

---

### Q9.4 Chess / Snake & Ladder / Progress bar / File manager

| Prompt | Minimum bar |
|--------|-------------|
| Chess | Board, Piece moves, Turn, Check win — extensible for new games |
| Snake & Ladder | Board, dice, player position — report extensibility |
| Progress bar | Sequential queue vs max 3 concurrent — semaphore |
| File manager | `mkdir`, `cp`, `mv`, `pwd` — tree `Node`, path resolution |

**Anchor when stuck:** “Extensibility is how we published **internal API libraries (P7)**—stable interfaces, swap implementations.”

---

## 10. DSA coding rounds

*Reported across SDE-2 Chennai/Bangalore, Staff, DSA elimination failures Apr 2026.*

| Problem | Approach | Java note |
|---------|----------|-----------|
| **Common intervals** | Two pointers on sorted lists | O(n+m); include `[4,5]` between [1,5] and [4,10] |
| **Largest rectangle histogram** | Monotonic stack | O(n); reported Chennai Jun 2026 |
| **Next greater element** | Stack | Pair with histogram study |
| **Search rotated sorted array** | Binary search on half | Mar 2026 Bangalore |
| **Longest substring no repeat** | Sliding window + HashMap | LinkedIn SDE-2 |
| **Number of islands** | DFS/BFS | Senior Jul 2026 |
| **Koko eating bananas** | Binary search on answer | Senior Jul 2026 |
| **Coin change / single number** | DP / XOR | Staff Dec 2025 |
| **Merge overlapping intervals** | Sort by start | LinkedIn round 1 |

**Interview tip (Apr 2026 reject):** must produce **running code**—practice writing clean Java in 35–40 min without IDE.

Full sketches: [06-concurrency-performance-dsa.md](./06-concurrency-performance-dsa.md).

---

## 11. Java, concurrency, Redis vs DB

*Reported: SDE-1 HM (concurrency), parking lot (Redis not primary DB, parallelism vs concurrency).*

### Q11.1 Java concurrency

**Topics:** `synchronized` vs `ReentrantLock`, `ConcurrentHashMap`, deadlock avoidance (lock ordering), thread pool sizing.

**Your story (P1):** Reactive + `boundedElastic` for blocking legacy quote (**P7**); never block event loop.

**Your story (P2):** semaphore on concurrent PEGA/payment calls per worker.

---

### Q11.2 Why Redis cannot replace primary database

> “On **PaymentMs (P3)** and batch acceptance (**P2**), durability and constraints matter. Redis eviction/restart loses data; no multi-row ACID for payment + outbox. We used Redis on **ConfigMs** for **catalog cache** with TTL and rebuild from DB.”

---

### Q11.3 CAP (managerial / parking lot)

- **Postgres/Cassandra strong row (payment, booking):** prefer consistency on that row.
- **Cassandra (P1 order history):** availability + partition tolerance; query-shaped tables.
- **Cache:** AP with TTL; source of truth remains DB.

---

## 12. Manager & behavioral

*Reported: Chennai manager 30 min; Staff fumbled “hardest project” before HLD.*

### Q12.1 Describe the hardest system you architected (prepare **before every round**)

**Use P1 + P2 (3–4 min):**

1. **Context:** B2B SMB/NSE North Star migration (**P1**), Spring Reactive, GraphQL CXP, Cassandra, CJCM/PEGA.
2. **Hardest part:** **Bulk engine (P2)** — 10k+ lines, RabbitMQ, idempotent consumers, **order dropout** in New Relic at **99.99%** platform goal.
3. **Decision:** outbox / accept-then-fan-out; fail-closed credit (**P5**); adapters for PEGA/OCR/credit.
4. **Failure:** one bad line must not poison batch; metric caught silent drops.
5. **Your role:** principal architect; mentored vendors on Cassandra keys and Reactive boundaries.

---

### Q12.2 Why Tekion / why leave Verizon

> “I want the backend to **be** the product. Tekion TAMC is a new cloud-native line close to workflows and scale. My last four years were **ordering at scale**; this year I shipped **Agentic Universe** and **SRE triaging**—I want one role that combines **Java distributed systems** and **production AI at the edge**, not strategy-only.”

---

### Q12.3 Disagreement on technical decision

**Template (use real example):** Product wanted every legacy quote edge case in v1 (**P7**). You shipped **narrow state machine** + explicit manual queue for rare cases; faster cutover, quote-to-order retired manual re-key.

---

### Q12.4 Production incident under pressure

**Template:** Dropout alert ↑ (**P1**); trace via APIGEE **`traceId` (P8)**; root cause downstream timeout; circuit breaker + temporary rate limit; postmortem added DLQ depth alert.

---

### Q12.5 Team size / how you operate

Have numbers ready: multi-vendor teams mentored on Reactive/Cassandra; architecture board (**P12**); still commit to FastAPI services (**P9/P10**).

---

### Q12.6 Issue near release / pre-prod checks (backend reports)

- Idempotency integration tests on payment path (**P3**).
- Smoke: bulk accept with test key → 202 (**P2**).
- Rollback plan: additive migrations only.

---

## 13. Architect loop: your risks

| Risk | What to do |
|------|------------|
| Lead with AI only | Open **P2** bulk + **P1** dropout metric |
| Kafka depth | RabbitMQ story + §1.2 debug checklist |
| No automotive product | **P5** policy/credit; **P6** pause; map in one sentence |
| Python vs Java | “**P9** boundaries; **P1/P2** Java at scale” |
| Too many FRs, no NFRs | Write 3 numbers before boxes (SSE reject lesson) |
| DSA round | Confirm if Architect loop includes coding; practice Java §10 |

---

## Quick reference: question → primary project

| Tekion question | Lead with |
|-----------------|-----------|
| Kafka / consumer stuck | **P2** RabbitMQ |
| Payment idempotency | **P3** + **P2** |
| Payment ok, later fails | **P2/P3/P6** saga |
| Policy engine | **P5** credit gating |
| Food delivery / bulk | **P2** |
| TTL cache | **P3** ConfigMs |
| Service booking / seats | **P3** conditional SQL |
| Parking lot | **P3** payment + Redis cache |
| Chat / e-commerce | **P3/P1** + **P8** |
| Distributed cache | **P3** Redis catalog |
| Agent platform | **P9** + **P10** |
| RAG eval | GenAI eval rubric + **P10** |
| Logging LLD | **P1** observability |
| Hardest project | **P1** + **P2** |

---

## Sources

- https://interviewexperiences.in/company/tekion  
- https://interviewexperiences.in/company/tekion/interview-questions  
- Reddit: [policy engine LLD](https://www.reddit.com/r/leetcode/comments/1sd136r/tekion_interview_design_a_policy_engine_lld/) (Apr 2026)  
- Reddit: [senior SSE onsite](https://www.reddit.com/r/leetcode/comments/1snz0c4/tekion_interview_experience_senior_software/) (Apr 2026)  

**Related guides:** [04-api-and-system-design.md](./04-api-and-system-design.md) (P2 walkthrough), [01-resume-stories.md](./01-resume-stories.md) (scripts).
