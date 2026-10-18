# Resume-backed stories (detailed scripts)

Use these as **spoken answers**, not bullet slides. Each section: **situation → constraint → your design → failure mode → metric → TAMC bridge**.

---

## Story A — North Star B2B ordering (principal architect, 2021–2025)

### Situation

Verizon B2B SMB and NSE ordering was moving off a rep-assisted quote path onto a **reactive microservices platform**. You were principal architect for the migration.

### Architecture you should be able to draw

```
[Client / channels]
        │
        ▼
┌───────────────────┐
│ CXP Aggregate     │  GraphQL — shapes one experience request
│ (BFF / aggregate) │
└─────────┬─────────┘
          │ sync calls with deadlines
          ▼
┌───────────────────┐     ┌───────────────────┐
│ Domain services   │     │ Case / CJCM layer │
│ enrollment,       │◄───►│ pause & resume,   │
│ add-a-line, cart  │     │ long-running case │
└─────────┬─────────┘     └─────────┬─────────┘
          │                         │
          ▼                         ▼
   Cassandra (key-shaped writes)   Case store
          │
          ▼
   Adapters: credit, OCR ScanID, adaptive auth, PEGA, legacy quote
```

**Layers in plain language:**

- **Aggregate:** Combines multiple domain reads/writes behind one GraphQL schema so the client does not orchestrate five REST calls.
- **Domain:** Owns enrollment, add-a-line, cart-like payloads. Each service owns its data.
- **Case layer:** Pause/resume and workflows that outlive a single HTTP request.
- **Adapters:** External systems with **timeouts** and explicit failure states—not hidden inside a “god service.”

### Hardest correctness problem (credit / external check)

**Wrong design:** Hold an open DB transaction while calling credit/PEGA/OCR. If the vendor is slow, you exhaust connection pools; if the vendor succeeds and your commit fails, you have inconsistent state.

**Your rule:**

1. Persist application/order in **your** store with status `PENDING_EXTERNAL`.
2. Call adapter with a **short budget** (e.g. 2s).
3. On success → transition state with a **single-row update** (optionally with version).
4. On timeout → `UNKNOWN` + async retry with **same business id** (idempotent retry).
5. **Fail-closed** for activation/SMS-like gates; **fail-open with fallback** only where product accepts degraded behavior (scoring, recommendations).

### Metric

**99.99% platform uptime** on mission-critical B2B ordering—not “we watched CPU.” **Order dropout**: accepted upstream, never completed downstream. New Relic tracked that gap.

Be ready: *how was 99.99% measured?* (window, scope, shared vs team-owned SLO). Answer honestly.

### TAMC bridge (say explicitly)

| Ordering | TAMC |
|----------|------|
| Order aggregate | Campaign + launch |
| Cart / bulk payload | Audience snapshot (immutable for that launch) |
| Pause/resume case | Pause campaign in flight |
| Credit check adapter | Consent + policy + scoring port |
| PEGA / legacy quote | ESP / Google / Meta / DMS feed |
| Dropout alert | Launch accepted, receipt missing |

---

## Story B — Bulk order engine (10K+ orders per payload)

### What the number means

**10,000+ orders inside one accepted payload**—not “10k RPS.” The API must **accept** quickly; work is **fan-out**.

### Flow (this is your event-driven answer)

1. `POST /batches` with idempotency key → insert `batch` row + `outbox` row → **202**.
2. Publisher emits **one message per order line** (or chunk) to **RabbitMQ**.
3. Consumer processes line `n` independently; line `n+1` still runs if `n` fails (unless product says otherwise).
4. **Unique (batch_id, line_id)** prevents duplicate order on retry.
5. Poison message → **DLQ** for manual replay.
6. Metric: `accepted_lines - terminal_lines` by tenant.

### Kafka mapping (honest)

“I operated this on **RabbitMQ**. On **Kafka** I would partition by `tenantId:batchId`, commit offset **after** idempotent line write, and use a DLQ topic. Producer idempotence does **not** stop duplicate order creation at the domain service.”

---

## Story C — Agentic Universe + SRE triaging (2025–now)

### Why this matters for TAMC

The JD wants **AI at the edge**, not AI inside the transaction. You already built:

- **Agentic Universe:** FastAPI, MCP, Postgres—workflows, run history, **human-in-the-loop** before an action counts.
- **SRE Triaging:** LangGraph correlates OpenSearch, Git, Jira, K8s changes → RCA narrative; **does not auto-remediate prod**.

### Step-by-step (SRE service)

1. Incident id / alert payload arrives.
2. Graph nodes fetch evidence (logs by trace, deploy diff, ticket, change window).
3. Each node writes **partial state** to Postgres (crash-safe resume).
4. Final narrative stored; engineer approves/edits.
5. No node calls “restart pod” without a separate approval path.

### TAMC mapping

| Agentic pattern | TAMC |
|-----------------|------|
| Human approval on run | Human approval on generated creative before send |
| MCP tool server | DeliveryPort, ScoringPort, CreativePort adapters |
| Long-running graph | Off the synchronous send path |
| Eval / scoring traces | RAG eval pipeline (Staff AI loop question) |

**Say clearly:** recent services are **Python/FastAPI**; TAMC core is **Java/Spring**—same **boundaries**, different language.

---

## Story D — Architecture board & edge security

### What you did

- Enterprise technical board: cloud + AI guidelines.
- Internet path: **Akamai, 42Crunch, APIGEE** with minimal legacy impact.
- Org-wide Claude Code / Copilot / MCP companion.

### Review comments you can cite (specific)

| Block | Comment | Accept |
|-------|---------|--------|
| Dual write | “Save then publish to broker” | Outbox in same transaction as business row |
| Tenant missing in cache key | Cross-tenant leakage | `tenantId` in every key |
| Spend check in Java | Two pods both pass | `UPDATE ... WHERE reserved + amt <= limit` |
| LLM SDK in domain module | Vendor change rewrites core | `ScoringPort` interface in domain module |

---

## Story E — Quote-to-order (cross-team)

**Situation:** Reps quoted in legacy system, re-keyed into ordering—errors and delay.

**Design:** Adapter ingests quote → creates order via **stable internal APIs** (libraries other teams adopted).

**Disagreement template:** Product wanted every legacy edge case in v1. You kept a **small state machine** + explicit **manual** state for rare cases.

**TAMC:** Dealer ops wants every OEM co-op rule in v1; same negotiation—**policy engine port**, versioned rules, fail-closed on send.

---

## Behavioral (manager round)

Prepare **written** answers for:

- Disagreement on technical decision (real example).
- Production incident under pressure (detection gap + guardrail).
- Tight deadline without skipping idempotency/tests on money path.
- Why Tekion / why leave Verizon (product backend is the product; TAMC greenfield + automotive domain).

**Chennai SDE-2 report (June 2026):** manager asked team size, how you operate, what you bring—have integers ready (teams mentored, vendors, payload size, uptime target).

---

## 45-second intro (technical, not executive)

> I’m a Distinguished Engineer at Verizon in Chennai. For four years I was principal architect on B2B SMB ordering: Spring Reactive, GraphQL, Cassandra, and a bulk engine that accepts **more than 10,000 orders per payload** on RabbitMQ. We held **99.99%** by tracking **order dropout**, not just CPU.  
> I still ship code—this year **Agentic Universe** (FastAPI, MCP, Postgres, human approval) and an **SRE triaging** service with LangGraph over OpenSearch and deploy/ticket context.  
> For Tekion TAMC I’d use the same rule I used on ordering: **commit the batch/order and payment intent locally**; credit, PEGA, quote, and models are **adapters** with timeouts and defined failures.

Stop. Let them pick a thread.
