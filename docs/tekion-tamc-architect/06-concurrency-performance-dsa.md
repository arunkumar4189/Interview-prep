# Concurrency, performance tuning, and DSA (detailed)

**Examples from:** bulk engine (**P2**), North Star Reactive (**P1**), case pause/resume (**P6**), PaymentMs (**P3**), cart validation (**P4**).

Catalog: [11-your-projects-catalog.md](./11-your-projects-catalog.md).

---

## Part 1 — Concurrency

### 1.1 Race on payment or inventory hold (P3 — Tekion payment question)

**Broken (review comment we blocked):**

```java
if (payment.getRemaining() >= amount) {
  payment.setRemaining(remaining - amount);
}
```

**Fixed — conditional update (PaymentMs):**

```sql
UPDATE payment_budget
SET reserved_cents = reserved_cents + :amount
WHERE tenant_id = :t AND partner_id = :p
  AND reserved_cents + :amount <= limit_cents;
```

`executeUpdate() == 0` → reject that **line’s** payment attempt; other lines in **P2** batch continue.

When interviewer asks **movie seat / service slot**, say: “Same SQL pattern we used on **payment reservation per order line**.”

### 1.2 Optimistic locking — PEGA case pause/resume (P6)

```java
@Transactional
public void resumeCase(CaseId id, int expectedVersion) {
  if (caseRepo.resumeIfVersion(id, expectedVersion) == 0)
    throw new ConflictException("Case updated by another rep or worker");
}
```

Two reps clicking pause/resume or a rep + bulk worker touching same case id.

### 1.3 Bulk line workers (P2)

- **Semaphore** on concurrent downstream order-domain calls per pod (bound to Cassandra/PEGA pool—not “unlimited threads”).
- **Hikari:** `(max_connections - admin) / pod_count`—we saw incidents when fan-out opened one session per line.

### 1.4 Spring Reactive (P1)

Legacy quote adapter (**P7**) on event loop = outage.

```java
return enrollmentService.load(id)
  .flatMap(e -> Mono.fromCallable(() -> legacyQuoteClient.fetch(e))
      .subscribeOn(Schedulers.boundedElastic()))
  .flatMap(this::persist);
```

### 1.5 Parallelism vs concurrency (Tekion parking lot)

- **P2:** 10k lines **concurrent** in queue; **parallel** workers capped by pool and downstream.
- **P11 ATG POS:** fewer parallel checkout lanes, many concurrent sessions—Retail Flex Flow optimized transaction time without unbounded threads.

### 1.6 Log4j-style async appender (Tekion LLD)

If asked: async queue + worker thread draining to appenders—same backpressure lesson as **boundedElastic** queue on **P1**.

---

## Part 2 — Performance tuning (measured on ordering)

### 2.1 Business metric — order dropout (P1 + P2)

**New Relic:** `lines_accepted - lines_completed` (or order accepted vs order created).

CPU can look healthy while dropouts climb—**that** is what we paged on.

### 2.2 Bulk payload knobs (P2)

| Knob | Tuning story |
|------|----------------|
| Accept path | Only header + outbox—p99 stays low |
| Messages per batch | Line-per-message vs chunk—trade retry blast radius |
| Worker count | Scale until **PEGA/credit/order DB** saturates, not until CPU hits 100% |
| JDBC/Cassandra batch writes | Batch status updates in groups |

**10,000+** is **lines per payload**, not RPS—state clearly.

### 2.3 Async cart validation (P4)

Moved CPU/IO heavy validation off HTTP thread—same accept/poll API as bulk batch.

### 2.4 GraphQL N+1 (P1)

DataLoader or batched domain calls—otherwise enrollment screen fans out 50 REST calls.

---

## Part 3 — DSA (Tekion reports — practice in Java)

Still use LeetCode for interview loop; **explain using your context** where possible.

### 3.1 Common intervals

Reported in Tekion SDE-2 round—useful for **merging validation windows** on cart promotions or **time-bounded bulk upload windows** (conceptual). Algorithm: two pointers—see previous doc for `[1,4],[4,5],...` output.

### 3.2 Largest rectangle in histogram

Chennai Tekion SDE-2 (2026)—monotonic stack O(n). No direct ordering tie-in required; pass coding round.

### 3.3 Resource usage over intervals (Staff AI Tekion)

Sweep line for **concurrent bulk workers** or **concurrent enrollment checks**—max overlapping intervals = peak pool size you sized.

### 3.4 Policy engine evaluator (Tekion Reddit)

Implement AND/OR tree—**same structure as credit rules** you integrated (**P5**): leaves compare fields, composite nodes AND/OR.

---

## Part 4 — LLD prompts → your projects

| Tekion LLD | Answer from your work |
|------------|----------------------|
| TTL cache | ConfigMs catalog cache (**P3**) with per-entry expiry |
| Elevator | Use if asked generically; optional tie: “task scheduling for line workers” (**P2**) |
| Parking lot | Payment hold + slot = **PaymentMs** reserve pattern |
| Food delivery | **P2** batch + **P3** payment + line failure isolation |
| LFU cache | Less common in your story—TTL cache from ConfigMs is stronger |

---

## Part 5 — 3-hour practice

1. Whiteboard **P2** accept → Rabbit → idempotent consumer.
2. SQL conditional payment update (**P3**).
3. Java: common intervals + histogram once each.
