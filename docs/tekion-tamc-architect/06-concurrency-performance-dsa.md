# Concurrency, performance tuning, and DSA (detailed)

---

## Part 1 — Concurrency

### 1.1 The classic race (budget / slot / cap)

**Broken:**

```java
int remaining = budget.getRemaining();
if (remaining >= amount) {
  budget.setRemaining(remaining - amount); // two threads both pass
}
```

**Fixed (Postgres):**

```sql
UPDATE budget
SET reserved_cents = reserved_cents + :amount
WHERE tenant_id = :t AND budget_id = :b
  AND reserved_cents + :amount <= limit_cents;
```

JDBC `executeUpdate() == 0` → conflict to client.

Same pattern for **service-center slot** (Tekion senior HLD) and **movie seat** booking.

### 1.2 Optimistic locking (campaign version)

```java
@Transactional
public void pause(CampaignId id, int expectedVersion) {
  int updated = repo.pauseIfVersion(id, expectedVersion);
  if (updated == 0) throw new ConflictException();
}
```

### 1.3 Thread pools and backpressure

**Bulk send workers:**

- Semaphore `32` concurrent ESP calls per pod.
- Queue depth bounded; else memory blow-up.
- **Hikari pool size** ≈ `(postgres_max_connections - admin) / num_pods` — not “200 because we use virtual threads.”

### 1.4 Spring Reactive (your stack)

Blocking call on event loop thread → latency collapse for all requests.

Pattern:

```java
return repository.find(...)
  .flatMap(order -> 
      Mono.fromCallable(() -> legacyClient.call(order))
          .subscribeOn(Schedulers.boundedElastic()))
  .flatMap(this::save);
```

`boundedElastic` queue must be **bounded**; monitor rejected tasks.

### 1.5 Virtual threads (Java 21)

- Tomcat can use VT per request.
- Still limit outbound HTTP client max connections.
- Avoid pinning: don’t hold `synchronized` across I/O on hot path.

### 1.6 Parallelism vs concurrency (Tekion parking-lot round)

- **Concurrency:** many tasks in progress (IO wait).
- **Parallelism:** many CPUs doing work simultaneously.

Send workers: high **concurrency**, limited **parallelism** to ESP (rate limit).

### 1.7 Java concurrency interview topics (SDE-1 HM round)

- `synchronized` vs `ReentrantLock`
- `ConcurrentHashMap` for in-memory TTL cache
- Deadlock: two locks opposite order—avoid by lock ordering
- **Thread-safe** Log4j-style appender queue (async logging LLD)

---

## Part 2 — Performance tuning

### 2.1 Measure first (your credibility)

**Ordering:** New Relic **order dropout**—business metric.

**TAMC equivalents:**

| Metric | Meaning |
|--------|---------|
| `launch_accepted - launch_terminal` | Stuck launches |
| Consumer lag | Backlog |
| ESP timeout rate | Adapter health |
| `sold_event_time - suppress_observed_time` | VIN suppress SLA |
| p99 accept API | User-facing |

### 2.2 Bulk payload tuning (10k orders / 200k recipients)

| Knob | Too small | Too large |
|------|-----------|-----------|
| Chunk size | Broker overhead | One retry repeats huge vendor batch |
| JDBC batch | Many round trips | Long transactions |
| Worker count | Slow drain | DB/ESP saturation |

**Study numbers (campaign):**

- Chunk 500 recipients
- 32 concurrent ESP calls per pod
- 8 pods → still capped by **50 ESP rps** account limit → use **Redis rate limiter**

### 2.3 N+1 queries

Launch status: don’t load 200k attempts for dashboard—**counter table** updated by events.

### 2.4 Caching for read perf

Campaign config in Redis—see [05-databases-events-caching.md](./05-databases-events-caching.md).

### 2.5 Async validation (your 2018–2021 work)

Large B2B cart validation off HTTP thread—same as launch accept: **202 + job**.

---

## Part 3 — DSA (Tekion-reported problems)

Confirm if Architect loop includes coding. If yes, practice in **Java**.

### 3.1 Common intervals (two pointers)

**Input:**

- A: `[1,5], [7,9], [9,11]`
- B: `[0,4], [4,10], [11,13]`

**Algorithm:**

```java
List<Interval> common(List<Interval> a, List<Interval> b) {
  List<Interval> out = new ArrayList<>();
  int i = 0, j = 0;
  while (i < a.size() && j < b.size()) {
    int start = Math.max(a.get(i).start, b.get(j).start);
    int end = Math.min(a.get(i).end, b.get(j).end);
    if (start <= end) out.add(new Interval(start, end));
    if (a.get(i).end < b.get(j).end) i++; else j++;
  }
  return out;
}
```

**Output:** `[1,4], [4,5], [7,9], [9,10], [11,11]`  
(Some public write-ups omit `[4,5]`—include it.)

**Complexity:** O(n + m) time, O(output) space.

### 3.2 Largest rectangle in histogram (monotonic stack)

Heights: `[2,1,5,6,2,3]`

Maintain stack of **indices** with increasing heights. When pop, width = current index - new top - 1.

**Answer:** max area **10** (heights 5 and 6).

```java
int largestRectArea(int[] h) {
  Deque<Integer> st = new ArrayDeque<>();
  int best = 0;
  for (int i = 0; i <= h.length; i++) {
    int cur = (i == h.length) ? 0 : h[i];
    while (!st.isEmpty() && h[st.peek()] > cur) {
      int height = h[st.pop()];
      int width = st.isEmpty() ? i : i - st.peek() - 1;
      best = Math.max(best, height * width);
    }
    st.push(i);
  }
  return best;
}
```

### 3.3 Other frequent Tekion problems

| Problem | Technique |
|---------|-----------|
| Search rotated sorted array | Modified binary search |
| Longest substring without repeat | Sliding window + HashMap |
| Next greater element | Monotonic stack |
| Number of islands | DFS/BFS O(mn) |
| Koko eating bananas | Binary search on answer |
| Max sum subarray unique elements | Sliding window + Set |
| Coin change | DP |
| Resource usage over intervals (Staff AI) | Sort by start, sweep line / priority queue |

### 3.4 “Maximum resources over intervals” (Staff AI DSA)

Intervals: `[1,4]→3`, `[3,5]→2`, `[4,6]→4`

Sweep time points; add/subtract resources → max concurrent **6** at t=4.

---

## Part 4 — Low-level design patterns Tekion likes

| Prompt | Core classes |
|--------|----------------|
| Elevator | `Elevator`, `Controller`, `Request`, state `IDLE/MOVING` |
| Parking lot | `Lot`, `Spot`, `Ticket`, payment idempotency |
| TTL cache | `Cache<K,V>`, `Entry` with expiry, sweeper |
| Log4j-like | `Logger`, `Appender`, `Layout`, async queue |
| Policy engine | `Policy`, `Condition` AND/OR tree, `Evaluator` |
| LFU cache | `Map` + frequency buckets (LeetCode 460) |

**State diagram** for elevator—Tekion explicitly asked in one loop.

---

## Part 5 — Practice plan (3 hours)

1. Implement common intervals + histogram in Java without IDE hints.
2. Explain budget `UPDATE` race on whiteboard.
3. Walk bulk engine → Kafka mapping with idempotency keys.
