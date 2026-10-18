# TAMC domain scenarios and AI integration (detailed)

Automotive marketing is **not** generic email SaaS. Interviewers with Tekion product context will listen for **VIN, rooftop, OEM co-op, consent**.

---

## 1. Vocabulary

| Term | Meaning |
|------|---------|
| Dealer group | Tenant with many rooftops |
| Rooftop | Single dealership location |
| VIN | Vehicle identity; can sell while ad is in flight |
| OEM co-op | Manufacturer-funded ads with strict rules |
| DNC / consent | Legal gates (TCPA SMS/calls, email opt-in, state privacy) |

---

## 2. Scenario A — Campaign launch + sold VIN

### Trigger

Dealer approves email/SMS to 200,000 contacts referencing vehicles in stock.

### Flow

1. **Audience snapshot** frozen (`snap_8821`).
2. **Launch accepted** (outbox → chunks).
3. Worker for recipient **R** includes VIN `1HGCM82633A004352`.
4. At T+5min, DMS emits `VehicleStatusChanged(VIN, SOLD)`.
5. Inventory service updates Postgres + Redis SET remove VIN.
6. Workers **already in flight** must call `isVinLive(vin)` **immediately before** ESP/SMS API.
7. Suppress metric: time from event to last in-flight send blocked.

### Failure if you skip step 6

Cache updated but worker batch loaded old creative list → dealer advertises sold car.

### Your analogy

Cancelled line item must stop fulfillment—**order dropout** thinking applied to marketing.

---

## 3. Scenario B — OEM co-op policy (policy engine LLD)

Reddit Tekion drive (Apr 2026): **ALLOW/DENY** from AND/OR tree.

Example rule:

```
(creditScore >= 720 AND accountStatus == "Active")
OR (receivableDays < 30)
```

TAMC co-op rule (different fields):

```
(modelYear >= 2024 AND make == "OEM_X")
AND (geo IN approved_states)
AND NOT (vin IN excluded_list)
```

### Implementation

```java
public enum Decision { ALLOW, DENY }

public interface PolicyPort {
  Decision evaluate(String policyId, Map<String, Object> facts, int policyVersion);
}
```

- Store policy as **versioned JSON tree** in Postgres.
- Evaluation is **pure function**—no side effects.
- Log: `policyId`, `version`, `decision`, `launchId`—not PII in log line.

Same pattern as **credit check adapter** on enrollment.

---

## 4. Scenario C — Spend ledger (payment idempotency forum Q)

Co-op budget $10,000.

1. **Reserve** $0.05 per attempted send in same TX as `delivery_attempt` insert.
2. **Commit** on ESP ACK.
3. **Release** on hard failure after max retries.
4. **Adjust** when Google invoice CSV disagrees (reconciliation job).

Unique: `(tenant_id, attempt_id, entry_type)`.

---

## 5. AI / GenAI integration (hexagonal)

### 5.1 Two ports (never one “AIService”)

| Port | Latency | On failure |
|------|---------|------------|
| `ScoringPort` | ≤ 25 ms in send path | Rules ranker fallback |
| `CreativePort` | seconds–minutes, async | Stay DRAFT; no send |

### 5.2 Creative path (maps to Agentic Universe)

1. `CreativePort.submit(request)` → job id.
2. Adapter calls LLM with **redacted** prompt; stores `modelId`, `promptVersion`, `inputHash`.
3. Status `PENDING_APPROVAL`.
4. Human approves in UI.
5. Campaign stores `approvedCreativeArtifactId` only.
6. Send worker loads artifact by id—**no LLM on send**.

### 5.3 Scoring path

- Precompute scores at snapshot build time → store next to member (best latency).
- Or inline call with **circuit breaker** + timeout.

### 5.4 SRE triaging vs send path

LangGraph multi-hop over OpenSearch/Git/Jira/K8s is **minutes** and **human-reviewed**.

Never that topology on “should we send this SMS now.”

### 5.5 RAG eval pipeline (Staff AI interview)

1. Golden set of (question, expected_citations).
2. Retrieve passages; generate answer.
3. Score: groundedness, citation match, safety.
4. Run offline in CI; block model version promotion if regression.

You already score **groundedness, citations, unsafe actions** in GenAI eval work—cite that.

---

## 6. Java module layout (decoupled intelligence)

```
campaign-domain/
  ScoringPort.java
  CreativePort.java
  DeliveryPort.java
  PolicyPort.java

campaign-adapters/
  LlmScoringAdapter.java
  RulesScoringAdapter.java
  SendGridDeliveryAdapter.java

campaign-api/
  LaunchController.java
```

Domain module `pom.xml` has **no** OpenAI, LangChain, or Kafka dependencies.

---

## 7. Whiteboard exercise (20 min)

**Prompt:** 200k launch, sold VIN at T+5min, co-op budget, SMS consent fail-closed.

Draw:

1. Services (5 boxes max).
2. Two idempotency keys (launch, attempt).
3. One conditional SQL (budget or cap).
4. One metric alert.

Compare to [04-api-and-system-design.md](./04-api-and-system-design.md).
