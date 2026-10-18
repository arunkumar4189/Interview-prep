# Answering automotive / TAMC questions using only your projects

You have **not** shipped Tekion Marketing Cloud. Interviewers still ask product-shaped questions. **Answer with P1–P12 first**, then one analogy sentence.

Catalog: [11-your-projects-catalog.md](./11-your-projects-catalog.md).

---

## 1. Vocabulary (listen for → map to your system)

| They say | You answer with |
|----------|-----------------|
| Campaign launch | **P2** bulk batch accept + Rabbit fan-out |
| Audience snapshot | **P4** cart frozen at validation start / batch payload immutable after accept |
| Consent / TCPA | **P5** adaptive auth + credit fail-closed; no activation without pass |
| Pause in flight | **P6** PEGA case pause & resume |
| Spend / co-op budget | **P3** PaymentMs idempotency + conditional reserve |
| Sold VIN suppress | **P4** line invalid after catalog change; **P1** dropout if fulfillment proceeds wrongly |
| LLM creative | **P9** generate → human approve → then side effect |
| Scoring on hot path | **P5** credit with 2s budget—not unbounded LLM |
| Policy ALLOW/DENY | **P5** credit rules tree / enrollment eligibility |
| Dealer tenant | B2B **partner_id / tenant_id** on every key |

---

## 2. Scenario: “200k marketing send” (they ask) → you answer with P2

**Do not** design a fictional ESP.

**Do:**

1. Clarify: “I’ll use the **bulk order engine I architected: 10,000+ lines per payload**, same mechanics at higher scale.”
2. NFR: accept p99, zero duplicate lines, dropout metric, 99.99% platform context (**P1**).
3. Draw: APIGEE (**P8**) → bulk API → outbox → Rabbit → workers → PaymentMs/Order/PEGA.
4. Idempotency: partner key + `(batch_id, line_id)`.
5. Analogy (one sentence): “A recipient is a line; ESP is order-domain handoff; suppress is line cancelled when inventory/rules change.”

---

## 3. Scenario: Policy engine (Tekion Reddit, Apr 2026)

**Their sample:** creditScore, accountStatus, receivableDays.

**Your answer:**

> “On **SMB enrollment** we evaluated **automated credit** and adaptive auth with explicit ALLOW/DENY style outcomes before activation. I’d model policy as a versioned condition tree—same as Tekion’s sample—with evaluation pure function over a snapshot, stored decision on the enrollment row (**P5**). PEGA case (**P6**) held exceptions reps resolved.”

Optional Java: `PolicyPort.evaluate(policyId, facts)` next to `CreditCheckPort`.

---

## 4. Scenario: Payment retry (Tekion Mar 2026)

Answer entirely with **P3 PaymentMs** + **P2** line worker:

- `idempotency_key` unique
- States CREATED → CAPTURED
- Retry returns stored row
- Payment captured, order domain fails → line FAILED, batch continues; compensating refund workflow

---

## 5. Scenario: Service-center booking (Tekion senior HLD)

> “Concurrent booking is the same concurrency class as **payment reserve** and **case version** on our platform: one conditional SQL update or optimistic version, not check-then-set in Java. Search nearby centers is a read model like **ConfigMs catalog** cache (**P3**).”

Then draw slot/booking generically if they insist.

---

## 6. Scenario: AI agent platform (Staff AI Tekion)

**This is your strongest match—use P9 + P10, not ordering.**

| Their requirement | Your system |
|-------------------|-------------|
| Long-running tools | **P10** LangGraph nodes (logs, Git, Jira, K8s) |
| Human approval mid-run | **P9** `AWAITING_APPROVAL` |
| Fire-and-forget vs watch | **P9** run id + status API |
| Crash-safe | **P9** Postgres `current_step` + version |
| Sandbox pool | **P9** worker pool limits (conceptual) |
| RAG eval | GenAI eval rubric on your resume (groundedness, citations) |

---

## 7. Scenario: Car service booking (automotive flavored)

Bridge only:

> “Vehicle **service slot** booking parallels **add-a-line enrollment** slots in peak: concurrent updates on case/enrollment version (**P6/P5**), read-heavy search cached like **ConfigMs**.”

---

## 8. Hexagonal Java (JD) — your ports today

| Port | Your adapter |
|------|----------------|
| `CreditCheckPort` | Automated credit (**P5**) |
| `OcrScanPort` | ScanID (**P5**) |
| `CasePort` | PEGA (**P6**) |
| `QuotePort` | Legacy quote (**P7**) |
| `McpToolPort` | **P9** MCP servers |
| `LogEvidencePort` | **P10** OpenSearch |

Campaign/ESP ports are **the same pattern** you already shipped.

---

## 9. Whiteboard drill (20 min)

**Prompt:** “Design marketing launch.”

**Response:** “I’ll design **bulk batch processing for B2B ordering**—my production system.” Then draw P2. Add P5 adapter for gating. Close with Tekion analogy table (section 1).

---

## 10. What not to do

- Invent Aerospike war stories.
- Describe LLM on GraphQL submit path (**P1** used bounded adapters, not GPT).
- Lead with Agentic Universe when they asked Java ordering architect—lead **P1/P2**, then **P9** for AI.
