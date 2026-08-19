# Technical Architect Interview — Study Plan

Use this with [`technical-architect-interview-guide.md`](./technical-architect-interview-guide.md).  
**Goal:** move from *knowing topics* to *delivering crisp 90–120 second answers* under pressure.

---

## Feedback Remediation — 5-Day Plan

> Use if your panel said you need deeper **Java runtime, ORM, messaging, K8s, and complex scenarios** — see [`feedback-remediation-guide.md`](./feedback-remediation-guide.md).

| Day | Focus | Read | Practice (90 min) |
|-----|-------|------|-------------------|
| **1** | Java runtime | Remediation §1 | Draw JVM memory map; G1 vs ZGC for payment API (3 min timed); thread vs connection pool sizing |
| **2** | ORM / Hibernate | Remediation §2 | N+1 + 3 fixes; `@Transactional` bugs; optimistic vs pessimistic locking |
| **3** | Messaging deep | Remediation §3 | Kafka partitions + consumer groups; RabbitMQ ACK/prefetch/DLX; outbox pattern |
| **4** | Kubernetes | Remediation §4 | Control plane + networking diagram; rolling update; JVM pod memory sizing |
| **5** | Complex scenarios | Remediation §5 | Double-charge retry + p99 latency scenarios; [Mock Round 4](./mock-interview-practice.md#round-4--feedback-remediation-60-minutes) |

**Daily:** 45 min read + 30 min timed spoken + 15 min remediation self-assessment.

**Priority:** Score remediation self-assessment ≥2 before returning to generic HLD review — panels will re-probe these gaps.

---

## Before You Start (30 minutes)

1. **Read §1** (COTDO framework) — this is how every answer should sound.
2. **Skim the Appendix checklist** — mark each question: ✅ confident | ⚠️ shaky | ❌ can't answer.
3. **Pick 2 STAR stories** from §18 and write them on one page (bullet points only).
4. **Block calendar time** — 60–90 min/day beats cramming the night before.

---

## If You Have 7 Days

| Day | Focus | Read (guide) | Practice (timed) | Done when… |
|-----|-------|--------------|------------------|------------|
| **1** | Answer structure + fundamentals | §1, §2 | 5 questions: CAP, eventual consistency, saga, scalability, ADR | You can recite COTDO without looking |
| **2** | Microservices + AWS | §3 | Whiteboard: one service on AWS (ALB, EKS, Mongo, Redis, IAM) | You name *why* for each box, not just labels |
| **3** | Distributed systems + messaging | §4, §5 | RabbitMQ fan-out + Kafka replay (2 min each); RabbitMQ vs Kafka decision | You can draw both flows from memory |
| **4** | Data layer | §7, §8, §9 | Cache-aside + stampede; Mongo schema; OpenSearch sync | You state failure modes for cache and index lag |
| **5** | Workflows + runtime | §6, §11, §12, §16 | Airflow vs Temporal; Node event loop; Java vs Node choice | One sentence each for Airflow vs Temporal |
| **6** | System design | §17 | Full 45-min design: **e-commerce** OR **ticket booking** | You ask clarifying questions before drawing |
| **7** | Behavioral + mock | §18, [`mock-interview-practice.md`](./mock-interview-practice.md) | 2 STAR stories (60s + 2min); Round 1 mock | You finish answers in ≤2 min with trade-offs |

**Daily routine (60 min):**
- 15 min — re-read one section
- 30 min — timed spoken answers (record yourself)
- 15 min — update self-assessment scores (see guide Appendix)

---

## If You Have 14 Days

Follow the 7-day plan above, then add:

| Day | Extra focus |
|-----|-------------|
| **8** | §10 Top 10 DS — LRU, Top-K heap, Bloom filter, consistent hashing (spoken + complexity) |
| **9** | §13, §14 — SOLID, circuit breaker, TDD at system level, tech debt |
| **10** | §15 — latency spike debug walkthrough + SLO table from memory |
| **11** | §17 Design B — analytics pipeline (batch vs near-real-time trade-off) |
| **12** | Weak areas from self-assessment — only ⚠️ and ❌ topics |
| **13** | Full **90-min mock** — [`mock-interview-practice.md`](./mock-interview-practice.md) Round 2 |
| **14** | Light review only — §19 cheat sheet + Monday 15-min review list |

---

## If You Have 24 Hours

Do **not** read the whole guide. Prioritize:

| Block | Time | Action |
|-------|------|--------|
| 1 | 20 min | §1 + §19 cheat sheet |
| 2 | 25 min | §17 e-commerce **spoken answer** (read aloud once) |
| 3 | 25 min | §5 RabbitMQ vs Kafka + §6 Airflow vs Temporal |
| 4 | 20 min | §18 — one architectural STAR + one incident STAR |
| 5 | 30 min | Mock Round 3 (quick) in `mock-interview-practice.md` |
| 6 | 10 min | Questions to ask interviewers (§18 table) |

Sleep > one more hour of reading.

---

## Day-Before Checklist

- [ ] Company + role: write 3 bullets on what *they* likely care about (scale, migration, reliability, cost)
- [ ] 2 STAR stories rehearsed out loud (phone voice memo is fine)
- [ ] One system design drawn on paper without notes
- [ ] 5 questions for them ready (not "what's the culture" only — use §18 table)
- [ ] Clothes, link, ID, quiet room tested if remote
- [ ] Stop studying 2 hours before sleep

---

## Day-Of (30 minutes before)

1. **5 min** — COTDO framework + one trade-off sentence: *"I always name what we gave up, not just what we gained."*
2. **10 min** — Draw e-commerce or ticket booking high-level diagram once.
3. **5 min** — RabbitMQ vs Kafka one-liner each.
4. **5 min** — Breathe; hydrate; join 2 min early.

---

## How to Practice (method that works)

### 1. Record yourself
Phone video, 90 seconds per question. Listen for:
- Did you state **context** first?
- Did you give **2–3 options** before deciding?
- Did you mention **failure modes**?
- Did you stop before 3 minutes?

### 2. Whiteboard without erasing
One diagram per session. If you erase, you won't reproduce it in the room.

### 3. "Probe me" mode
After each answer, ask yourself (or a friend):
- *What if Redis dies?*
- *What if we 10x traffic?*
- *What would you do differently?*

### 4. Score honestly

| Score | Meaning |
|-------|---------|
| **3** | Would impress a staff/architect panel; trade-offs + numbers |
| **2** | Correct but rambling or light on trade-offs |
| **1** | Know the topic but can't deliver under time |
| **0** | Need to re-read section |

Target: **≥2 on all Appendix questions** before the interview; **3 on your top 10** most likely questions.

---

## Likely Interview Formats

| Format | What to prep | Guide sections |
|--------|--------------|----------------|
| **Architecture deep-dive** | Past systems, ADRs, trade-offs | §1, §2, §18 |
| **System design (45–60 min)** | Clarify → high-level → deep-dive one path | §17, §3, §4 |
| **Stack quiz** | Mongo, Redis, Kafka, Temporal, AWS | §5–§9, §16 |
| **Coding-lite / DS** | LRU, Top-K, complexity at scale | §10 |
| **Behavioral** | Influence, incidents, disagreement | §18 |

Ask your recruiter which formats to expect if you can — adjust days 6–7 accordingly.

---

## Your Personal One-Pager (fill in)

Copy this and complete before the interview:

```
ROLE / COMPANY:
What they build:
Why this role (your 2 sentences):

MY STRONGEST PROOF POINTS:
1.
2.
3.

MY GO-TO SYSTEM DESIGN:
(e.g. e-commerce saga + Temporal)

MY GO-TO INCIDENT STORY:
(rollback + root cause + prevention)

WEAK TOPICS I'M POLISHING:
-
-

QUESTIONS I'LL ASK THEM:
1.
2.
3.
```

---

*Next step: open [`mock-interview-practice.md`](./mock-interview-practice.md) and run Round 1 timed.*
