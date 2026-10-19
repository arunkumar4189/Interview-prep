# Design: Cloud-native Automotive Dealership Management System (DMS)

A system design reference modeled on what **Tekion** publicly positions: a **cloud-native** platform for automotive retail (dealership operations, inventory, sales, service, F&I, parts, CRM) with **OEM and partner integrations**, multi-store **dealer groups**, and extension points for **marketing (TAMC)** and analytics.

**How to use in interview:** Clarify → NFRs → tenant model → bounded contexts → 2–3 deep flows → integrations → events → ops. Tie to your work where honest ([P1–P12](./11-your-projects-catalog.md)).

---

## 1. Clarifying questions (first 2–3 minutes)

| Question | Why it matters |
|----------|----------------|
| **Scope:** full DMS vs one pillar (inventory + deals only)? | Box count and depth |
| **Tenancy:** single rooftop vs **dealer group** with 50+ stores? | Partition keys, RBAC, reporting |
| **OEM mix:** one brand vs many on same rooftop? | OEM feed formats, co-op rules |
| **Regulation:** US state titling/F&I vs India market? | Compliance services, data residency |
| **Scale:** VINs per group, ROs/day, concurrent users? | Cassandra vs Postgres split |
| **Legacy:** replace CDK/Reynolds in one cut vs strangler? | Adapter layer, dual-write period |
| **Real-time:** must website reflect sold VIN in seconds? | Event pipeline to digital retail |

**Assumptions for this document (state them aloud):**

- **Dealer group** tenant with many **rooftops** (stores).
- **US-centric** retail: new/used inventory, deal jacket, service repair orders (RO), parts counter.
- **10k VINs** in stock across group; **2k ROs/day** group-wide; **500** concurrent users peak.
- **Cloud-native** on AWS, Java/Spring microservices, Kafka backbone, Postgres + Cassandra + Redis + OpenSearch.

---

## 2. Non-functional requirements (put numbers on the board)

| NFR | Target | Notes |
|-----|--------|--------|
| API availability (core OLTP) | 99.95% | Sales/service during business hours |
| Read inventory for digital retail | p99 < 150 ms | Cached projection |
| Post deal / post RO write | p99 < 500 ms | Includes local commit + outbox |
| VIN status visible across channels | < 5 s eventual | After deal funded or manual sold |
| RPO / RTO (platform) | 1 h / 4 h | Per rooftop data isolated |
| Audit | 7 years deal/F&I artifacts | Immutable event log |
| Tenant isolation | Zero cross-tenant reads | Enforced in API + DB key |

---

## 3. Business capabilities (bounded contexts)

Tekion-style DMS is not one database. Split by **business capability** and **change cadence**.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Experience / BFF layer                            │
│  Dealer Portal (web) │ Mobile (service advisor) │ OEM APIs │ Partner API │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌───────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Inventory     │     │ Sales / Deal    │     │ Service         │
│ (VIN, stock)  │     │ (CRM, desking)  │     │ (RO, scheduling)│
└───────┬───────┘     └────────┬────────┘     └────────┬────────┘
        │                      │                       │
        ▼                      ▼                       ▼
┌───────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Parts         │     │ F&I / Lender    │     │ Customer / CRM  │
│ (parts desk)  │     │ (contracts)     │     │ (party, consent)│
└───────────────┘     └─────────────────┘     └─────────────────┘
        │                      │                       │
        └──────────────────────┼───────────────────────┘
                               ▼
                    ┌─────────────────────┐
                    │ Accounting / GL     │  (often adapter to external ERP)
                    └─────────────────────┘
                               │
                    ┌─────────────────────┐
                    │ Integration hub     │  OEM, DMV, lenders, marketing (TAMC)
                    └─────────────────────┘
```

**Supporting platforms (horizontal):**

- **Identity & RBAC** — group admin, rooftop roles (sales, F&I, service, parts).
- **Document store** — deal jackets, signed PDFs, OCR’d IDs (S3 + metadata in Postgres).
- **Notification** — SMS/email with consent gate.
- **Search** — OpenSearch for customer/VIN/stock search.
- **Workflow / cases** — long-running (lender pend, OEM approval) — your **PEGA/CJCM (P6)** pattern.

---

## 4. Tenancy and identity model

```
dealer_group_id  (tenant root — billing, group policies)
    └── rooftop_id (store — inventory location, tax jurisdiction, GL branch)
            └── user_id (roles scoped to rooftop or group)
```

**Rules:**

- Every row/event carries `dealer_group_id`; most operational rows also carry `rooftop_id`.
- **VIN** is unique in context of `(rooftop_id, vin)` or group-wide depending on product—define one rule and enforce in API.
- **RBAC:** sales user at rooftop A cannot desk a deal on rooftop B unless group role.
- **Marketing (TAMC):** campaigns scoped to `dealer_group_id` with optional rooftop filter; suppress ads when VIN sells (event).

**Your parallel:** B2B `tenant_id` + partner on **bulk engine (P2)**; APIGEE token claims vs row checks (**P8**).

---

## 5. Core data model (conceptual)

### 5.1 Vehicle / inventory (system of record)

```text
Vehicle
  vin (PK component)
  dealer_group_id, rooftop_id
  stock_number
  status: IN_TRANSIT | IN_STOCK | RESERVED | SOLD | WHOLESALE
  new_used, year, make, model, trim
  odometer, color, price, hold_reason
  version (optimistic lock)
  last_event_id
```

**Indexes:** by rooftop + status (lot walk), by VIN (global lookup within group).

**Hot read path:** Redis projection `inv:{group}:{rooftop}` → set or hash of `vin → {status, price, photo_url}` for website; invalidated on `VehicleStatusChanged`.

### 5.2 Deal (sales)

```text
Deal
  deal_id, dealer_group_id, rooftop_id
  status: QUOTE | PENDING_APPROVAL | SOLD | FUNDED | CANCELLED
  buyer_party_id, co_buyer_party_id
  vehicle_vin (nullable until attached)
  trade_ins[], fees[], taxes (jurisdiction)
  version

DealLine / Desking
  products: vehicle, accessories, warranties
```

**Invariant:** one **active** deal cannot attach the same VIN as another active deal—conditional update on `Vehicle` when attaching VIN.

### 5.3 Service

```text
RepairOrder (RO)
  ro_id, rooftop_id, status: OPEN | IN_PROGRESS | READY | CLOSED | VOID
  vehicle_vin, customer_party_id
  jobs[] (labor, parts lines)
  promised_time
```

### 5.4 Customer / CRM

```text
Party (person or business)
  party_id, dealer_group_id
  pii_encrypted: name, phone, email, address
  consents: { channel, purpose, granted_at, revoked_at }
```

Consent is **Postgres source of truth**; marketing reads via API or consent-changed events.

### 5.5 Parts

```text
Part, StockLocation, PartsInvoice
  link to RO or counter sale
```

---

## 6. Store strategy (SQL vs NoSQL)

| Data | Store | Why |
|------|-------|-----|
| Deal, RO, Party, consent, payments | **Postgres** (per group shard or schema) | ACID, constraints, joins for reporting |
| Vehicle events, high-volume status history | **Cassandra** | Write-heavy timeline by VIN |
| Inventory snapshot for web | **Redis** | p99 reads |
| Search (customer, stock #) | **OpenSearch** | Full text, facets |
| Documents | **S3** | PDF, images |
| Analytics | **Warehouse** (Snowflake/BQ) via CDC | OEM reporting |

**Your parallel:** Cassandra order history by tenant (**P1**); Postgres for Agentic run state (**P9**).

---

## 7. Key flows (design in depth)

### 7.1 Flow A — Vehicle arrives to “in stock” (OEM / DMS intake)

1. OEM file or API → **Integration hub** adapter normalizes to `VehicleReceived` event.
2. **Inventory service** upserts `Vehicle` IN_STOCK in Postgres (transaction).
3. Same TX: `outbox` row → Kafka `inventory.vehicle.v1` key=`vin`.
4. Consumers: update **Redis** projection; index OpenSearch; notify digital retail.
5. Idempotency: `oem_event_id` unique.

**Failure:** duplicate OEM message → unique constraint → no double VIN row.

---

### 7.2 Flow B — Desk deal and reserve VIN (sales)

1. Sales creates `Deal` QUOTE in Postgres.
2. Attach VIN:

```sql
UPDATE vehicle
SET status = 'RESERVED', version = version + 1
WHERE dealer_group_id = $g AND vin = $v
  AND status = 'IN_STOCK' AND version = $expected;
-- 0 rows → 409 VIN_NOT_AVAILABLE
```

3. Update `Deal` with `vehicle_vin` in same transaction if possible, or saga: reserve then attach with compensating release on deal cancel.
4. Publish `VehicleReserved`, `DealUpdated`.

**Cancel deal:** `RESERVED → IN_STOCK` if not SOLD; event drives marketing “available again.”

**Your parallel:** conditional budget reserve (**P3**); case pause blocking new work (**P6**).

---

### 7.3 Flow C — Fund deal → SOLD (F&I, lenders)

1. F&I sends contracts to lender adapters (timeout + circuit breaker).
2. On **FUNDED**:
   - `Deal` → FUNDED
   - `Vehicle` → SOLD (same TX or saga with outbox)
   - Emit `VehicleSold` — **critical for TAMC suppress**, website, OEM sales reporting
3. **Idempotency:** lender callback `idempotency_key` on funding record.

**Tekion-style requirement:** sold VIN must propagate in **seconds**—monitor lag on `VehicleSold` consumer (marketing, web).

**Your parallel:** payment CAPTURED then fulfillment fail (**P2/P3** saga)—here funding is terminal for VIN; rollback is exceptional (unwind deal) not silent delete.

---

### 7.4 Flow D — Service RO (appointment → close)

1. Schedule: read model of bay capacity; book slot with **unique (rooftop, bay, start)** or capacity counter (service-center booking pattern).
2. Open RO: link VIN + customer; parts reserve from inventory.
3. Close RO: post labor/parts; payment; GL export event.
4. RO events feed CRM (service history on Party).

---

### 7.5 Flow E — Marketing cloud (TAMC) integration

DMS is **system of record** for VIN and consent; TAMC is **downstream**.

```
DMS Kafka: inventory.vehicle.v1, crm.consent.v1, deal.deal.v1
    → TAMC: audience builder, campaign suppress, co-op eligibility
```

- **Suppress:** on `VehicleSold`, TAMC removes VIN from live ad sets (your **P2** line-invalid mid-batch analogy).
- **Audience:** segment = customers with RO in last 90 days — **CDC** or domain events, not live DMS SQL from TAMC.
- **Hexagonal:** TAMC calls `InventoryReadPort` / subscribes to events; DMS does not embed LLM.

---

## 8. API design (examples)

```http
GET  /v1/groups/{groupId}/rooftops/{rooftopId}/inventory?status=IN_STOCK&cursor=...
GET  /v1/groups/{groupId}/vehicles/{vin}
POST /v1/groups/{groupId}/rooftops/{rooftopId}/deals
POST /v1/groups/{groupId}/deals/{dealId}/attach-vehicle
     Idempotency-Key: ...
POST /v1/groups/{groupId}/rooftops/{rooftopId}/repair-orders
```

**GraphQL BFF** optional for dealer portal (your **P1 CXP** pattern)—compose deal + customer + vehicle in one round trip with per-field timeouts.

Errors: problem+json; `409` for VIN conflict, stale deal version.

---

## 9. Event-driven backbone (Kafka)

| Topic | Key | Consumers |
|-------|-----|-----------|
| `inventory.vehicle.v1` | vin | Web cache, TAMC, OEM outbound, analytics |
| `sales.deal.v1` | deal_id | CRM, GL, notifications |
| `service.ro.v1` | ro_id | Parts, CRM, warranty OEM |
| `crm.consent.v1` | party_id | TAMC, SMS gateway |
| `integration.oem.v1` | oem_batch_id | Inventory |

**Patterns:**

- **Outbox** in each service Postgres transaction.
- **At-least-once** consumers; idempotent on `event_id`.
- **Ordering** per VIN partition for status transitions.

**Your production story:** **P2** RabbitMQ bulk lines → same rules on Kafka.

---

## 10. Integrations (adapter layer)

| External | Direction | Pattern |
|----------|-----------|---------|
| OEM (inventory, warranties) | In/out | Hub + idempotent `oem_event_id` |
| Lenders (F&I) | Out/in | Async callbacks, signing e-docs |
| DMV / titling | Out | Batch + status poll |
| Accounting (GL) | Out | Events → ERP adapter |
| Credit bureaus | Out | **P5**-style port, timeout, fail-closed |
| OCR (driver license) | Out | ScanID adapter (**P5**) |
| Legacy DMS (migration) | In | Strangler **P7** quote-to-order style |

No vendor SDK in domain modules—**ports** in Java, adapters in `integration-*` jars.

---

## 11. Security and compliance

- **P8:** APIGEE/Akamai at edge; OAuth2 dealer users; mTLS partner APIs.
- **PII:** encrypt at rest; tokenize in logs; field-level RBAC (F&I sees SSN, sales may not).
- **Audit:** append-only `audit_log` for deal status changes (who/when/old/new).
- **Consent** before SMS/email (**TCPA**).

---

## 12. Observability and SLOs

| Signal | Alert |
|--------|--------|
| `VehicleSold` lag (emit → marketing ack) | p99 > 30s |
| Deal attach conflict rate | spike (data or UX bug) |
| Integration hub DLQ depth | > 0 sustained |
| API error rate by rooftop | tenant-specific outage |

**Dropout-style metric:** deals `FUNDED` in DMS but vehicle not `SOLD` in projection—analogous to **order dropout (P1)**.

**P10:** SRE triaging correlates logs/traces by `deal_id`, `vin`, `traceId`.

---

## 13. Migration from legacy DMS (strangler)

Phases:

1. **Inventory + digital retail** read from new system; writes dual-publish or nightly reconcile.
2. **Desking** on new deal service; legacy read-only.
3. **Service RO** migrate rooftop by rooftop.
4. Cut GL adapter last (finance cannot tolerate gaps).

**Reconciliation jobs:** compare VIN counts legacy vs new; adjust with audited `INTEGRATION_ADJUST` events.

---

## 14. What you say in a 35-minute interview (script)

**0–3 min:** Clarify scope (full DMS vs inventory+deals), tenancy, scale numbers above.

**3–8 min:** Draw bounded contexts + tenancy (`dealer_group`, `rooftop`).

**8–18 min:** Deep-dive **Flow B + C** (reserve VIN, fund deal, `VehicleSold` event)—SQL conditional update, outbox, Kafka, TAMC suppress.

**18–25 min:** Data stores (Postgres vs Cassandra vs Redis) and one integration (OEM or lender).

**25–30 min:** NFRs, security, migration strangler.

**30–35 min:** Your experience:

> “I have not built Tekion’s DMS, but I ran **high-volume B2B ordering (P1)** with **Cassandra**, **adapter-heavy enrollment (P5)**, **case pause (P6)**, and **bulk idempotent fan-out (P2)**. The mechanisms map directly: **VIN reserve is conditional SQL**, **sold VIN is an event with dropout monitoring**, **marketing is a consumer not inside the deal transaction**.”

---

## 15. Common follow-up questions

| Follow-up | Answer sketch |
|-----------|----------------|
| Two deals same VIN? | DB conditional update; one wins, other 409 |
| Sold online, store not updated? | Event lag SLO; Redis projection + source Postgres |
| OEM sends wrong status? | Versioned `last_event_id`; reject stale AVAILABLE after SOLD |
| Group report across rooftops? | CDC to warehouse; not cross-shard join at request time |
| Offline store? | Mobile RO with local queue + conflict resolve (advanced)—mention if asked |

---

## 16. Related reading in this repo

- [04-api-and-system-design.md](./04-api-and-system-design.md) — bulk batch / idempotency (**P2**)
- [09-tamc-domain-scenarios.md](./09-tamc-domain-scenarios.md) — marketing ↔ inventory
- [10-reported-tekion-questions.md](./10-reported-tekion-questions.md) — booking, payments, Kafka
