# Java and Spring / Spring Boot (detailed)

Your production Java story is **Spring Reactive + GraphQL + Cassandra** through 2025. TAMC will likely use **Spring Boot** for domain services. You must explain **both** and when you pick each.

---

## 1. Spring Boot service layout (campaign domain)

Typical modules for a clean boundary:

```
campaign-domain/          ← entities, ports (interfaces), domain services
campaign-application/     ← use cases orchestrating domain
campaign-adapters/        ← JDBC, Kafka relay, ESP client, scoring client
campaign-api/             ← @RestController, DTO mapping, validation
```

**Rule:** `campaign-domain` has **zero** dependencies on Kafka, Redis, LLM SDKs, or Spring Web.

---

## 2. Worked example: accept a campaign launch

### Requirement

- Client may retry `POST` on timeout.
- Launch must be **durable** before any send work runs.
- No message broker call inside the DB transaction.

### Code sketch

```java
@Service
public class LaunchService {
  private final LaunchRepository launches;
  private final OutboxRepository outbox;

  @Transactional
  public LaunchAck accept(String idempotencyKey, LaunchRequest req) {
    return launches.findByIdempotencyKey(idempotencyKey)
        .map(Launch::toAck)
        .orElseGet(() -> {
          Launch launch = launches.save(Launch.accepted(idempotencyKey, req));
          outbox.save(OutboxEvent.launchAccepted(launch.getId(), req.getAudienceSnapshotId()));
          return launch.toAck();
        });
  }
}
```

### Schema (Postgres)

```sql
CREATE TABLE launch (
  tenant_id         TEXT NOT NULL,
  launch_id         TEXT NOT NULL,
  idempotency_key   TEXT NOT NULL,
  campaign_id       TEXT NOT NULL,
  audience_snapshot_id TEXT NOT NULL,
  status            TEXT NOT NULL,
  created_at        TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (tenant_id, launch_id),
  UNIQUE (tenant_id, idempotency_key)
);

CREATE TABLE outbox (
  id           BIGSERIAL PRIMARY KEY,
  tenant_id    TEXT NOT NULL,
  aggregate_id TEXT NOT NULL,
  event_type   TEXT NOT NULL,
  payload      JSONB NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL,
  published_at TIMESTAMPTZ
);
```

### Why each piece exists

| Piece | Reason |
|-------|--------|
| `UNIQUE (tenant_id, idempotency_key)` | Two concurrent retries: one wins insert, the other gets unique violation → map to existing row (or catch and re-read). |
| `@Transactional` on both inserts | Launch without outbox → crash → launch exists but no work ever scheduled. Outbox without launch → orphan event. |
| No Kafka in `@Transactional` | Broker ACK ≠ DB commit ordering; dual-write bug. |
| `202 Accepted` + poll | Fan-out of 200k sends is not one HTTP transaction. |

### Relay (separate component)

```java
@Scheduled(fixedDelay = 500)
public void publishOutbox() {
  List<OutboxEvent> batch = outbox.fetchUnpublished(100);
  for (OutboxEvent e : batch) {
    try {
      kafka.send(e); // or rabbitTemplate
      outbox.markPublished(e.getId());
    } catch (Exception ex) {
      break; // retry next tick; row stays unpublished
    }
  }
}
```

**Interview line:** “Outbox gives **at-least-once publish aligned with commit**; consumers must be **idempotent** on `event_id` or business `attempt_id`.”

---

## 3. `@Transactional` pitfalls (they still ask at architect level)

### Self-invocation

```java
@Service
public class Bad {
  public void outer() {
    inner(); // NO proxy → NO transaction
  }
  @Transactional
  public void inner() { ... }
}
```

Fix: call `inner` from another bean, or make `outer` transactional and keep logic inline.

### Rollback

- Default: rollback on **unchecked** exceptions.
- Checked exception you want to roll back → `@Transactional(rollbackFor = Exception.class)` or don’t catch it.

### Read-only

`@Transactional(readOnly = true)` on heavy reports—routes to replica if you use routing datasource.

---

## 4. Spring Reactive (your ordering stack) vs Spring MVC (TAMC CRUD)

| | Spring WebFlux / Reactive | Spring MVC + Boot |
|--|---------------------------|-------------------|
| Thread model | Few event-loop threads | Servlet thread per request (or virtual threads on Tomcat) |
| Good when | Many concurrent I/O waits, backpressure matters | CRUD, straightforward request/response |
| JDBC | **Blocking**—must use `subscribeOn(boundedElastic)` or R2DBC | Natural fit with JPA/JDBC |
| Your history | Ordering aggregate, high concurrency waits | Campaign create/approve/read |

**Architect answer:**

> We used Reactive on ordering because the aggregate waited on several downstream checks with strict latency goals. Blocking JDBC on the event loop is a production incident. For campaign metadata and launch accept—which are short transactional writes—I’d default to **Boot MVC** unless profiling shows thread exhaustion.

### Virtual threads (Java 21+)

- Good for **blocking I/O** at scale **if** you still bound downstream (DB pool, HTTP client max connections).
- **Does not** increase Postgres `max_connections`.
- Avoid long `synchronized` blocks on VT carriers (pinning)—prefer `ReentrantLock` on hot paths.

---

## 5. Integration with external systems (adapter in Spring)

```java
public interface CreditCheckPort {
  CreditResult check(CreditRequest req, Duration budget);
}

@Component
public class PegaCreditAdapter implements CreditCheckPort {
  private final WebClient client;
  private final CircuitBreaker breaker;

  @Override
  public CreditResult check(CreditRequest req, Duration budget) {
    return breaker.executeSupplier(() ->
        client.post()
            .uri("/credit")
            .bodyValue(req)
            .retrieve()
            .bodyToMono(CreditResult.class)
            .block(budget)
    );
  }
}
```

Domain service:

```java
public void submitEnrollment(Enrollment e) {
  repo.save(e.pendingCredit());
  CreditResult r = creditPort.check(e.toCreditRequest(), Duration.ofSeconds(2));
  repo.transition(e.getId(), r.toStatus());
}
```

**Never** open a JPA transaction across `creditPort.check` if the check is slow—save first, then call.

---

## 6. Testing (proves hands-on)

- **Domain:** `LaunchService` with in-memory `LaunchRepository` + fake outbox.
- **Integration:** `@DataJpaTest` or Testcontainers Postgres for unique constraint on idempotency key.
- **Adapter:** WireMock for ESP/timeouts; assert circuit opens and fallback path runs.

---

## 7. Practice exercises

1. Write `LaunchService.accept` with duplicate key handling (catch `DataIntegrityViolationException`, re-fetch).
2. Explain what breaks if outbox relay marks `published` before Kafka ACK.
3. Convert one Reactive “call adapter then save” flow to “save PENDING then adapter then UPDATE status.”

See [05-databases-events-caching.md](./05-databases-events-caching.md) for outbox + consumer idempotency.
