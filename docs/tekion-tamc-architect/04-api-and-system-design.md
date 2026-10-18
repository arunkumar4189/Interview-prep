# API design and backend system design (detailed)

**All examples:** bulk order engine (**P2**), async cart validation (**P4**), GraphQL CXP aggregate (**P1**), internal API libraries (**P7**), PaymentMs (**P3**).

Catalog: [11-your-projects-catalog.md](./11-your-projects-catalog.md).

---

## Part A — API design from your platforms

### 1. Resources you actually exposed

| API surface | Project | Example paths |
|-------------|---------|----------------|
| Bulk batch | **P2** | `POST /v1/bulk-batches`, `GET /v1/bulk-batches/{batchId}` |
| Cart | **P3/P4** | `POST /v1/carts/{cartId}/validate` (async), `GET .../validation-status` |
| Enrollment | **P5** | `POST /v1/enrollments`, `GET /v1/enrollments/{id}` |
| Case pause/resume | **P6** | `POST /v1/cases/{caseId}/pause`, `.../resume` |
| GraphQL | **P1** | Single CXP schema composing domain fields |

Internal **published libraries** (**P7**) wrapped these contracts so other Verizon teams did not fork DTOs.

### 2. Idempotency — bulk batch (P2)

Partner retries upload after gateway timeout.

```http
POST /v1/bulk-batches
Authorization: Bearer …
Idempotency-Key: 7a2b9c10-4e5f-6a7b-8c9d-0e1f2a3b4c5d
Content-Type: application/json

{ "partnerId": "prt_88", "lines": [ /* 10k+ order lines */ ] }
```

```http
HTTP/1.1 202 Accepted

{
  "batchId": "bat_9001",
  "status": "ACCEPTED",
  "statusUrl": "/v1/bulk-batches/bat_9001"
}
```

Poll:

```json
{
  "batchId": "bat_9001",
  "status": "PROCESSING",
  "linesAccepted": 10240,
  "linesCompleted": 6102,
  "linesFailed": 14
}
```

**DB:** `UNIQUE (tenant_id, idempotency_key)` on `bulk_batch`.

### 3. Async cart validation (P4)

Large B2B cart cannot block servlet/reactive thread for minutes.

```http
POST /v1/carts/cart_771/validations
→ 202 { "validationJobId": "val_12", "status": "QUEUED" }

GET /v1/carts/cart_771/validations/val_12
→ 200 { "status": "COMPLETED", "errors": [...] }
```

Same pattern as batch accept: **acknowledge work**, persist job id, process off-thread.

### 4. GraphQL aggregate (P1) — API composition

Client one round-trip:

```graphql
mutation submitEnrollment($input: EnrollmentInput!) {
  submitEnrollment(input: $input) {
    enrollmentId
    status
    caseId   # CJCM / PEGA case when applicable
  }
}
```

Resolver fans out to **domain services** with per-call timeouts—not one monolith stored procedure.

### 5. Errors (libraries other teams relied on — P7)

```json
{
  "type": "https://api.verizon.example/problems/case-paused",
  "title": "Case is paused",
  "status": 409,
  "detail": "Case case_441 is PAUSED; resume before submitting add-a-line.",
  "tenantId": "b2b-smb",
  "caseId": "case_441"
}
```

| Code | Your use |
|------|----------|
| 409 | Illegal transition (paused case, batch already completed) |
| 422 | Cart line fails business validation |
| 503 | Credit/PEGA down—retry with **same** Idempotency-Key |

### 6. Pagination

For batch line status export: **cursor** on `(batch_id, line_id)`, not OFFSET across 10k+ rows.

---

## Part B — System design walkthrough: bulk order engine (P2) — what you built

Use this when they say “design a high-throughput backend workflow.” It **is** your answer.

### Step 0 — Clarify (state numbers)

| NFR | Your platform (study talking points) |
|-----|--------------------------------------|
| Lines per payload | **10,000+** (resume) |
| Accept API p99 | Low—only persist header + outbox |
| Platform availability | **99.99%** on mission-critical B2B ordering (**P1**) |
| Duplicate line processing | **0** (unique `batch_id + line_id`) |
| Silent loss | **0**—**order dropout** metric in New Relic |

### Step 1 — Diagram (say this out loud)

```
[B2B Partner] → [APIGEE P8] → [Bulk Batch API P2]
                                    │
                    Cassandra / store: batch header, outbox
                                    │
                                    ▼
                          [Outbox Relay] → RabbitMQ (bulk.order.lines)
                                    │
                                    ▼
                          [Line Workers + Netflix OSS edge]
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
        [Order domain P1]    [PaymentMs P3]      [PEGA case P6]
              │
              ▼
        New Relic: accepted_lines vs completed_lines (dropout)
```

### Step 2 — Write path

1. **Accept** batch + outbox (transaction).
2. Relay publishes **one message per line** (or chunk of lines if you chunked—state which you used).
3. Consumer creates/updates order for line with idempotent key.

**Message body (conceptual):**

```json
{
  "eventId": "evt_uuid",
  "tenantId": "b2b-smb",
  "batchId": "bat_9001",
  "lineId": "line_0042",
  "orderPayload": { ... }
}
```

Routing key / queue: enough isolation that one partner’s poison line does not block all B2B traffic (partition by `tenant_id` or `batch_id`).

### Step 3 — Consumer logic (per line)

```text
INSERT processing_attempt (batch_id, line_id) ON CONFLICT DO NOTHING
IF inserted:
    CALL order domain create/update
    IF payment required → PaymentMs with idempotency key derived from line
    IF case needed → PEGA adapter (P6)
    MARK line COMPLETED or FAILED with reason
ELSE
    skip duplicate delivery
```

### Step 4 — Failure modes you operated

| Failure | What we did |
|---------|-------------|
| Rabbit redelivery | Idempotent line key |
| Credit timeout on one enrollment line in batch | Line FAILED; others continue (**P5** rule on single line) |
| PEGA pause | **P6**—case paused, worker stops new work on that case id |
| Partner thinks batch lost | Idempotency key returns same `batchId` |

### Step 5 — Tekion forum parallels (answer using P2)

| Their prompt | Your answer |
|--------------|-------------|
| Food delivery / payment edge cases | **PaymentMs** idempotency + line FAILED without failing whole batch |
| Payment success, later step fails | Payment captured on line; order fulfillment failed → compensating workflow / manual queue |
| Kafka consumer stuck | Same debug story on **Rabbit** queue depth + DLQ |

### Step 6 — What we gave up

- **Strong consistency** on batch progress UI (eventual line counters).
- **Synchronous** processing of 10k lines in one HTTP request.
- **Global ordering** across all B2B tenants (only ordering inside one batch if required).

---

## Part C — Second system design: North Star enrollment path (P1 + P5)

Shorter HLD if they want “customer journey” not bulk:

1. GraphQL **CXP** mutation.
2. Domain enrollment service persists.
3. Parallel adapters: OCR ScanID, adaptive auth, credit—with budgets.
4. CJCM opens **PEGA** case; pause/resume on rep workflow.
5. Cassandra for enrollment read models; dropout on any step.

This is the “workflow + adapters” design without bulk fan-out.

---

## Part D — One sentence if they ask automotive marketing

> “I have not built TAMC; the closest live system I ran is **10k-line bulk accept with RabbitMQ and idempotent line consumers**, and **enrollment with fail-closed credit adapters**—same split between durable aggregate, async fan-out, and external ports.”
