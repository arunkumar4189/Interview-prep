# API design and backend system design (detailed)

---

## Part A — API design

### 1. REST conventions for TAMC

**Resources (nouns):**

- `/v1/campaigns/{campaignId}`
- `/v1/campaigns/{campaignId}/launches`
- `/v1/launches/{launchId}`
- `/v1/audience-snapshots/{snapshotId}`
- `/v1/audience-snapshots/{snapshotId}/members`

**Verbs via HTTP methods:**

- `POST` create launch (non-idempotent without key)
- `GET` read status
- `PATCH` pause/resume campaign (state transition)

### 2. Idempotency (required on money and launch)

**Client generates** `Idempotency-Key` (UUID) and sends on **every retry** of the same logical operation.

Server:

```sql
UNIQUE (tenant_id, idempotency_key)
```

On conflict: return **same** `202` body as first success (don't create second launch).

**Why server-generated keys fail:** Retry after timeout gets a new key → double launch.

### 3. Example: launch API

```http
POST /v1/campaigns/cmp_44/launches
Authorization: Bearer …
Idempotency-Key: 6f1c0e2a-1b4d-4e0a-9c33-0a9e2f7b11d4
Content-Type: application/json

{
  "audienceSnapshotId": "snap_8821",
  "channels": ["email", "sms"]
}
```

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{
  "launchId": "ln_100",
  "status": "ACCEPTED",
  "statusUrl": "/v1/launches/ln_100"
}
```

Status poll:

```json
{
  "launchId": "ln_100",
  "status": "RUNNING",
  "accepted": 200000,
  "sent": 84110,
  "failed": 120,
  "suppressed": 902
}
```

### 4. Pagination (audience snapshot members)

**Do not** use `OFFSET` on large snapshots—rows shift, pages skip/duplicate.

**Cursor:**

```http
GET /v1/audience-snapshots/snap_8821/members?limit=500&cursor=eyJpZCI6Ijg0MTEifQ
```

Cursor encodes last `member_id` in stable sort order `(snapshot_id, member_id)`.

### 5. Errors (RFC 7807 problem+json)

```json
{
  "type": "https://api.tekion.example/problems/launch-conflict",
  "title": "Campaign is paused",
  "status": 409,
  "detail": "Campaign cmp_44 is PAUSED; resume before launch.",
  "tenantId": "acme",
  "campaignId": "cmp_44"
}
```

Clients branch on `type`, not English `detail`.

| Code | Use |
|------|-----|
| 400 | Malformed JSON, unknown channel |
| 404 | Unknown campaign/snapshot |
| 409 | Illegal state transition |
| 422 | Snapshot expired / not approved |
| 429 | Tenant rate limit |
| 503 | Dependency down; retry with same idempotency key |

### 6. Versioning

- **Additive:** new optional JSON fields on `202` response—stay on `/v1`.
- **Breaking:** remove enum value, rename field → `/v2` + deprecation header.

Your **internal API libraries** at Verizon succeeded because error codes and resource names were stable—same discipline here.

---

## Part B — System design: campaign launch (full walkthrough)

### Step 0 — Clarify (2 minutes)

Ask:

- Recipients per launch? (200k study number)
- Channels? Email + SMS
- ESP rate limit? (50 req/s)
- Consent requirements? Fail-closed
- Sold VIN during send? Must suppress within 30s

**State NFRs (Tekion rejects “no numbers”):**

| NFR | Target |
|-----|--------|
| Accept API p99 | < 200 ms |
| Accept availability | 99.9% |
| Complete 200k sends | 15 min (≈222/s average; burst limited by ESP) |
| Double-send rate | 0 |
| VIN suppress p99 | < 30 s after `SOLD` event |

### Step 1 — High-level diagram

```
[Dealer UI] → [API GW] → [Campaign Service] → Postgres (campaign, launch, outbox)
                              │
                              ▼
                        [Outbox Relay] → Kafka topic campaign.launch.v1
                              │
                              ▼
                        [Send Workers] → [Consent] → [Rate limit Redis]
                              │                    → [Delivery adapters: ESP, SMS]
                              ▼
                        Postgres (delivery_attempt, receipt)
                        
[DMS feed] → [Inventory Service] → Postgres + Redis SET (live VINs)
         → event VIN_SOLD → workers re-check before vendor call
```

### Step 2 — Data model (core tables)

See [05-databases-events-caching.md](./05-databases-events-caching.md).

### Step 3 — Write path (launch)

Already in [02-java-spring-boot.md](./02-java-spring-boot.md)—outbox + chunk messages.

**Chunk message payload:**

```json
{
  "eventId": "evt_uuid",
  "tenantId": "acme",
  "launchId": "ln_100",
  "campaignId": "cmp_44",
  "chunkIndex": 17,
  "recipientIds": ["r1", "r2", ...]
}
```

**Partition key:** `tenantId:campaignId` (ordering per campaign).

### Step 4 — Consumer (per recipient)

```text
FOR each recipientId in chunk:
  IF NOT consent_ok(tenant, recipient, channel): SKIP
  IF capped(tenant, recipient, channel): SKIP
  IF campaign_uses_vin AND vin NOT IN live_set: SKIP
  attempt_id = hash(launchId, recipientId, channel)
  INSERT delivery_attempt ... ON CONFLICT DO NOTHING
  IF inserted:
      CALL esp.send(..., idempotencyKey=attempt_id)
      UPDATE attempt status SENT / FAILED
```

### Step 5 — Read path (dealer dashboard)

Materialized counters in Postgres or Redis updated by receipt events—not `COUNT(*)` on 200M row table per page load.

### Step 6 — Deep dives interviewers ask

**Pause campaign mid-flight:**

- Set campaign `PAUSED`.
- Workers check status at chunk start; in-flight chunk may complete—product decision.
- New chunks not published.

**Restaurant closes after order (Tekion food delivery):**

- Order state `ACCEPTED` → if restaurant closes → `CANCELLED` + refund saga.
- **Parallel:** launch `RUNNING` → campaign `PAUSED` + suppress new chunks.

**Payment succeeded, inventory failed (Tekion senior):**

- Payment row `CAPTURED`; inventory reservation failed → `NEEDS_REFUND` event; finance job; never double-charge on retry.

### Step 7 — What you gave up

- **Strong consistency** on dashboard counts (eventual by seconds).
- **Global ordering** across tenants (only per-campaign).
- **Synchronous LLM** on send path (creative pre-approved offline).

---

## Part C — Map to your bulk engine

| Bulk order engine | Campaign launch |
|-------------------|-----------------|
| Batch accept | Launch accept |
| Line message | Chunk message |
| `batch_id + line_id` unique | `launch_id + recipient_id + channel` unique |
| RabbitMQ | Kafka (in JD) |
| Dropout metric | `accepted - terminal` launches |

Say this mapping in one minute—it proves distributed systems experience without claiming fake Kafka ops history.
