# Java and Spring / Spring Boot (detailed)

**Every example below is from your work:** bulk order engine (**P2**), North Star ordering (**P1**), SMB enrollment adapters (**P5**), Cart/Config/Payment microservices (**P3**).

See [11-your-projects-catalog.md](./11-your-projects-catalog.md).

---

## 1. Module layout (how we structured ordering services)

Same pattern we used splitting **CartMs, ConfigMs, PaymentMs** and later North Star domain services:

```
order-batch-domain/       ← Batch, OrderLine, ports (no Rabbit, no PEGA SDK)
order-batch-application/  ← AcceptBulkBatch, ProcessLine commands
order-batch-adapters/     ← Cassandra repos, RabbitTemplate, PegaCaseClient, CreditClient
order-batch-api/          ← REST controllers for bulk accept + status
```

**Rule:** `order-batch-domain` does not depend on RabbitMQ, Netflix OSS glue, or PEGA client jars.

---

## 2. Worked example: accept a bulk order batch (P2 — 10,000+ lines per payload)

### Requirement (what we actually needed)

- Partner submits **one payload** with 10,000+ order lines.
- HTTP client **retries** on timeout.
- Batch must be **durable** before any line hits RabbitMQ.
- **No** `rabbitTemplate.send()` inside the Cassandra/DB transaction.

### Code sketch (Spring Boot style; production was Reactive + Cassandra)

```java
@Service
public class BulkBatchAcceptService {
  private final BulkBatchRepository batches;
  private final OutboxRepository outbox;

  @Transactional
  public BulkBatchAck accept(String idempotencyKey, BulkBatchRequest req) {
    return batches.findByIdempotencyKey(idempotencyKey)
        .map(BulkBatch::toAck)
        .orElseGet(() -> {
          BulkBatch batch = batches.save(BulkBatch.accepted(idempotencyKey, req));
          outbox.save(OutboxEvent.batchAccepted(batch.getBatchId(), req.getLineCount()));
          return batch.toAck();
        });
  }
}
```

### Storage (Cassandra-friendly keys; same idea on relational outbox)

```sql
-- Illustrative: bulk_batch + outbox (if relational store used for batch header)
CREATE TABLE bulk_batch (
  tenant_id         TEXT NOT NULL,
  batch_id          TEXT NOT NULL,
  idempotency_key   TEXT NOT NULL,
  status            TEXT NOT NULL,  -- ACCEPTED, PROCESSING, COMPLETED, FAILED
  line_count        INT NOT NULL,
  created_at        TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (tenant_id, batch_id),
  UNIQUE (tenant_id, idempotency_key)
);
```

Line detail and order outcomes lived in **partitioned Cassandra tables** keyed by `(tenant_id, batch_id, line_id)` on the North Star platform.

### Why each piece exists

| Piece | Tied to your incident class |
|-------|-----------------------------|
| `UNIQUE (tenant_id, idempotency_key)` | Partner retry must not create **batch 2** for the same upload. |
| Transaction wraps batch + outbox | Crash after accept → either both exist or neither—no “accepted in UI, zero lines queued.” |
| No Rabbit in `@Transactional` | Same dual-write bug we avoided on ordering writes. |
| `202 Accepted` + status poll | 10k lines cannot complete inside one HTTP round-trip. |

### Outbox relay → RabbitMQ (what you ran in prod)

```java
@Scheduled(fixedDelay = 500)
public void publishOutbox() {
  for (OutboxEvent e : outbox.fetchUnpublished(100)) {
    try {
      rabbitTemplate.convertAndSend("bulk.order.lines", routingKey(e), e.payload());
      outbox.markPublished(e.getId());
    } catch (Exception ex) {
      break; // retry; lines not lost because outbox row remains
    }
  }
}
```

**Interview line:** “On the bulk engine we had **at-least-once** on RabbitMQ; consumers were idempotent on **`batch_id + line_id`**. That is the same guarantee Tekion describes with Kafka—we operated it on Rabbit.”

---

## 3. `@Transactional` pitfalls (seen on ordering code reviews)

### Self-invocation

Calling `@Transactional` method from same class → no proxy → **no transaction**. We enforced “call accept from controller or `BulkBatchFacade` bean only.”

### External call inside transaction

**Anti-pattern we blocked:** open transaction → call PEGA/credit → commit.

**P5 enrollment pattern:**

```java
public void submitEnrollment(Enrollment e) {
  repo.save(e.withStatus(PENDING_CREDIT));
  CreditResult r = creditPort.check(e.toCreditRequest(), Duration.ofSeconds(2));
  repo.transition(e.getId(), r.toStatus()); // separate short update
}
```

Credit adapter (`PegaCreditAdapter` / vendor client) uses **WebClient + circuit breaker**, same as APIGEE-protected downstreams on **P8**.

---

## 4. Spring Reactive (P1) vs Spring Boot MVC (P3-style CRUD)

| | **P1** North Star aggregate | **P3** CartMs/ConfigMs |
|--|-----------------------------|-------------------------|
| Stack | WebFlux + GraphQL CXP | Boot MVC + REST |
| Why | Many concurrent I/O waits (enrollment checks) | Straightforward CRUD |
| JDBC | On `boundedElastic`, bounded queue | Blocking JDBC OK |

**What you say:** “Reactive was a measured choice on the **GraphQL aggregate** when we profiled thread blocking on downstream checks—not fashion. **CartMs**-style services stayed imperative Boot because the path was simple reads/writes.”

### Virtual threads

If Boot 21+ on aggregate: still cap **Hikari** and **PEGA HTTP pool**—virtual threads do not grow Cassandra connection limits.

---

## 5. Adapter example: automated credit check on SMB Enrollment (P5)

```java
public interface CreditCheckPort {
  CreditResult check(CreditRequest req, Duration budget);
}

@Component
public class AutomatedCreditAdapter implements CreditCheckPort {
  private final WebClient client;
  private final CircuitBreaker breaker;

  @Override
  public CreditResult check(CreditRequest req, Duration budget) {
    return breaker.executeSupplier(() ->
        client.post().uri("/credit").bodyValue(req)
            .retrieve().bodyToMono(CreditResult.class)
            .block(budget));
  }
}
```

**OCR ScanID** and **adaptive auth** were separate adapters with the same timeout + explicit failure state—not logic copied into enrollment entities.

---

## 6. Testing (proves “currently hands-on”)

- **P2:** `BulkBatchAcceptService` with in-memory repo + fake outbox; duplicate idempotency key returns same ack.
- **P5:** WireMock credit timeout → enrollment stays `PENDING` / `UNKNOWN`, no double activation.
- **P1:** Integration test on Cassandra mapper for `(tenant_id, batch_id, line_id)` uniqueness.

---

## 7. Practice (from your systems)

1. Handle `DataIntegrityViolationException` on idempotency key → re-fetch batch **P2**.
2. Explain outage if relay marks outbox published before Rabbit ACK.
3. Walk **P6** pause: optimistic version on case row while bulk lines still processing.

Next: [05-databases-events-caching.md](./05-databases-events-caching.md) for line consumer idempotency and Cassandra keys.
