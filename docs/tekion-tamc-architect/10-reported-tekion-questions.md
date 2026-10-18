# Reported Tekion interview questions (with detailed answers)

Sources: Interview Experiences (32 write-ups), LeetCode Discuss, Reddit r/leetcode, LinkedIn/Medium posts through 2026. **Not official**—loops vary by role and level.

---

## How to use this doc

For each question:

1. **Clarify** (30–60 seconds).
2. **State NFRs with numbers**.
3. **Draw one design** (don’t list every alternative—SSE reject lesson).
4. **Anchor to a project ID (P1–P12)** — see [11-your-projects-catalog.md](./11-your-projects-catalog.md). Never answer with a fictional campaign service.

---

## Kafka

### Q: Explain Kafka components.

**Answer structure:** topic, partition, offset, broker, producer, consumer group, replication, ISR. Commit offset after idempotent side effect.

**Your line:** RabbitMQ fan-out for 10k orders; same idempotency on Kafka.

### Q: Producer publishes; consumer does not consume.

See checklist in [05-databases-events-caching.md](./05-databases-events-caching.md).

### Q: At-least-once vs at-most-once (HM, Medium 2025)

| | Commit | Risk |
|---|--------|------|
| At-most-once | Before process | Loss |
| At-least-once | After process | Dup → idempotency key |

---

## Payments / idempotency

### Q: Retries must not deduct balance twice (SDE-2 Bangalore, Mar 2026)

Answer with **P3 PaymentMs** + **P2** line worker:

```sql
UNIQUE (tenant_id, idempotency_key)  -- payment_intent
```

State: `CREATED → CAPTURED`. Retry with same key returns same row—no second capture for that order line.

Optional: payment row + outbox in one TX (**P2** pattern) if you emitted payment events.

### Q: Payment success, inventory/notification fails (Senior onsite, Apr 2026)

**P2 + P3:** Payment `CAPTURED` for line N; order domain or PEGA fails → line `FAILED`, other lines continue; refund/compensation workflow—never delete `CAPTURED` row.

---

## System design prompts

### Food delivery / Swiggy (Chennai SDE-2 selected, Jun 2026)

After clarifying, anchor to **P2** bulk batch + **P3** PaymentMs:

- “Order placed, restaurant closes” → line/batch item invalid mid-flight; worker marks line failed; payment saga (**P3**).
- Payment idempotency → PaymentMs keys on bulk line reference.

Optional one-line Tekion analogy only after you finish the Verizon design.

### Car service-center booking (Senior SWE, rejected on HLD, Jul 2026)

**Needs:** geo search, slots, concurrent booking, HA.

**Core:**

- Search index for centers (read model).
- `slot_booking` with `UNIQUE (center_id, slot_start)` or row lock.
- Transaction: insert booking only if slot free.

**Why candidates fail:** only boxes, no concurrency story.

### IRCTC search + booking (SSE, LeetCode 2024)

- Search: read-heavy, cache, ES optional.
- Booking: **strong consistency** on seat—Postgres row lock / conditional update.

### Web chat + e-commerce (SDE-2, architect-led, LinkedIn 2024)

**RADIO:** Requirements, Architecture, Data model, Interfaces, Optimization.

Cover: message store, presence (Redis), WebSocket gateway, search across history (ES), rate limits, auth.

**Your edge:** APIGEE + Akamai for public ingress security.

### Distributed cache HLD (Senior onsite, Apr 2026)

Replication, eviction (LRU/LFU), HA, consistent hashing.

Mention **trie** for URL prefix rules in URL-filter variant (Staff Dec 2025).

### Slack LLD + HLD same round (Apr 2026)

Channels, messages, fan-out to online users, persistence.

### AI agent execution platform (Staff AI, Sep 2026)

**Requirements:** long runs, tools, human approval, workflows, sandboxes, fire-and-forget vs live watch, crash-safe.

**Answer:** Agentic Universe—Postgres step log, resume from last step, MCP tools, approval gate, bounded worker pool.

### ChatGPT-style multi-tenant chat (Staff AI LLD)

Streaming tokens, tenant isolation, search history, edit message.

Postgres per tenant partition or `tenant_id` on all rows; SSE/WebSocket stream; ES for search.

### RAG eval pipeline (Staff AI)

Offline golden set, retrieval metrics, regression gate—tie to your GenAI eval rubric.

---

## LLD coding prompts

| Prompt | Key detail |
|--------|------------|
| TTL cache | Thread-safe, expiry on read + sweeper |
| LFU cache | Frequency buckets |
| Elevator | State diagram, external vs internal buttons |
| Parking lot | Spot types, payment idempotency |
| Log4j-like | Levels, appenders, async queue |
| Policy engine | AND/OR tree evaluator |
| Airplane seats | Hold + confirm + timeout |
| File manager `cp/mv/mkdir` | Machine coding |
| Progress bar UI | Sequential vs concurrent fill (max 3) |

---

## DSA (if in your loop)

| Problem | Where reported |
|---------|----------------|
| Common intervals | SDE-2 Mar 2026 |
| Largest rectangle histogram | Chennai Jun 2026 |
| Next greater element | Chennai Jun 2026 |
| Search rotated array | Multiple |
| Coin change + single number | Staff Dec 2025 |
| Resource sweep intervals | Staff AI |

Implement in Java—see [06-concurrency-performance-dsa.md](./06-concurrency-performance-dsa.md).

---

## Manager / behavioral

- Past project deep dive (**prepare North Star before any round**—Staff down-level story).
- Why Tekion, team size, how you operate.
- Disagreement, production pressure, quality under deadline.
- Issue near release / pre-prod checks (Backend engineer reports).

---

## Architect-specific prep for you

| Risk | Mitigation |
|------|------------|
| Sound like AI strategist | Open with bulk engine + Spring + dropout metric |
| Kafka gap | RabbitMQ → Kafka mapping + debug checklist |
| No ad-tech | VIN suppress + co-op policy engine |
| Python recent | “Boundaries in Agentic Universe; implementation in Java for TAMC core” |

---

## Sources (for your own reading)

- https://interviewexperiences.in/company/tekion
- https://interviewexperiences.in/company/tekion/interview-questions
- Reddit: policy engine LLD (Apr 2026), senior SSE onsite (Apr 2026)
- Medium: Ritwik SDE-2 breakdown, Kashish Kafka HM question
