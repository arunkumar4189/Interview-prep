# Mock Interview Practice

Three timed rounds. Do them **out loud** — typing answers is not enough.

**Setup:** 45–90 min quiet block, whiteboard or paper, phone timer, optional voice recorder.

**Scoring rubric (per question):**

| Dimension | 0–1 weak | 2 solid | 3 excellent |
|-----------|----------|---------|-------------|
| Structure | Jumps to solution | Uses COTDO mostly | Crisp Context → Options → Trade-offs → Decision → Outcome |
| Depth | Buzzwords only | Correct mechanics | Failure modes, idempotency, ops |
| Communication | >3 min or <30 sec | On time | Pauses, offers "go deeper on X?" |
| Evidence | No numbers | Some metrics | Latency, throughput, error rates, cost |

**Pass threshold:** average **≥2.0** per round; **≥2.5** on system design questions.

Answers and hints: [`technical-architect-interview-guide.md`](./technical-architect-interview-guide.md).

---

## Round 1 — Foundation (45 minutes)

*Simulates: phone screen or first technical round.*

### Part A — Warm-up (15 min, ~3 min each)

**Q1.** How do you structure an architectural answer when you don't know the "right" answer?

<details>
<summary>Interviewer probes</summary>

- Give me an example where you changed your mind after listing trade-offs.
- How do you avoid sounding indecisive when you present multiple options?

</details>

**Q2.** Explain CAP theorem with a real example from e-commerce.

<details>
<summary>Interviewer probes</summary>

- Is DynamoDB CP or AP?
- How would you handle inventory during a partition?

</details>

**Q3.** When is eventual consistency acceptable, and how do you handle it in the UI?

<details>
<summary>Interviewer probes</summary>

- What maximum staleness would you document for product search?
- What happens if the user refreshes and sees old data?

</details>

**Q4.** Explain Saga — choreography vs orchestration. When do you pick each?

<details>
<summary>Interviewer probes</summary>

- What if compensation fails?
- Why not 2PC?

</details>

**Q5.** How do you define microservice boundaries?

<details>
<summary>Interviewer probes</summary>

- When would you *not* split a service?
- How do you handle shared data?

</details>

---

### Part B — Stack spot-check (15 min, ~2 min each)

**Q6.** RabbitMQ vs Kafka — when do you choose each?

**Q7.** Compare cache-aside, write-through, and write-behind. What breaks when Redis fails?

**Q8.** Airflow vs Temporal — one sentence each plus a use case.

**Q9.** How do you keep OpenSearch in sync with MongoDB without dual writes?

**Q10.** Node.js event loop — why can CPU-heavy work hurt latency?

---

### Part C — Behavioral (15 min)

**Q11.** Tell me about a significant architectural decision you made. *(2 minutes max)*

**Q12.** Tell me about a production incident you led or helped resolve. *(2 minutes max)*

**Q13.** What questions do you have for us? *(Prepare 3 — see guide §18)*

---

### Round 1 debrief (5 min)

- Which questions scored **0–1**? Schedule those sections tomorrow.
- Did any answer exceed 3 minutes? Cut the middle, keep trade-offs.
- Re-record Q11 or Q12 once — target 90 seconds, then expand to 2 min if asked.

---

## Round 2 — System design + depth (75 minutes)

*Simulates: onsite or panel system design + follow-ups.*

### Part A — System design (45 min)

**Prompt:** Design an **e-commerce order platform**.

**Minutes 0–5:** Clarifying questions only (do not draw yet).

Suggested questions to ask:
- Orders/day and peak orders/minute?
- Inventory consistency — can we oversell?
- Read/write ratio? Multi-region?
- Checkout latency SLA?

**Minutes 5–15:** High-level diagram — client through API to services and data stores.

**Minutes 15–30:** Deep-dive **checkout path** — saga steps, compensations, idempotency.

**Minutes 30–40:** Scalability and failure modes — DB shard key, cache, queue backpressure.

**Minutes 40–45:** What you'd add with more time (DR, CQRS, event sourcing).

<details>
<summary>Common follow-ups (answer each in 2 min)</summary>

| Follow-up | Strong answer touches |
|-----------|----------------------|
| Payment gateway timeout mid-charge | Idempotency keys, reconcile job, saga state |
| Inventory oversell during flash sale | Atomic reserve, Redis lock optional, queue at edge |
| Search index 5s behind catalog | Acceptable for search; not for checkout stock |
| 10x traffic next Black Friday | HPA, queue depth scaling, load test, cache warm |
| Why Temporal over choreography-only | Visibility, timeouts, compensation ordering |

</details>

---

### Part B — Deep technical (20 min)

**Q14.** Walk through debugging a production latency spike step by step.

**Q15.** Design a distributed rate limiter with Redis.

**Q16.** Top-K trending products from 10M events/hour — which structure and why?

**Q17.** Explain expand-migrate-contract for zero-downtime DB migration.

---

### Part C — Influence (10 min)

**Q18.** Tell me about a time you influenced without authority.

**Q19.** Tell me about disagreeing with leadership on a technical approach.

---

### Round 2 debrief (10 min)

Score the system design separately:

| Criteria | 3 | 2 | 1 |
|----------|---|---|---|
| Asked clarifying questions | | | |
| Named components with *why* | | | |
| Checkout saga + compensation | | | |
| Failure modes | | | |
| Scale numbers (even estimates) | | | |

If any row is **1**, redo only that slice tomorrow (15 min).

---

## Round 3 — Ticket booking + rapid fire (40 minutes)

*Simulates: second design question or "pressure round".*

### Part A — Alternate system design (25 min)

**Prompt:** Design a **ticket booking system** (concerts / flights — pick one).

Focus areas interviewers care about:
- Seat locking and expiry
- Preventing double booking
- Payment + confirmation saga
- On-sale spike (1M users)

Use guide **§17 Design C** for self-grade after.

<details>
<summary>Must mention at least 3 of these</summary>

- [ ] Lock TTL (e.g. Redis SET NX + expiry)
- [ ] Idempotent confirm / payment
- [ ] Compensation if payment succeeds but confirm fails
- [ ] Wait room or queue at edge for on-sale
- [ ] SQL vs Redis for seat map trade-off

</details>

---

### Part B — Rapid fire (15 min, 90 sec each)

**Q20.** Consistent hashing vs `hash % N`

**Q21.** Bloom filter — false positives vs false negatives

**Q22.** B-Tree vs hash index in databases

**Q23.** Exactly-once processing — realistic approach?

**Q24.** SOLID at service level, not just classes

**Q25.** How do you balance tech debt with feature delivery?

**Q26.** SLO example for checkout — SLI, target, error budget

**Q27.** When Java over Node for a new microservice?

**Q28.** Circuit breaker — when to use, when to avoid

**Q29.** Contract testing — why at architect level?

**Q30.** What would you do differently on your last major migration?

---

## Panel simulation (optional, 90 min)

If you have a friend or second device:

| Segment | Time | Actor |
|---------|------|-------|
| Intro + behavioral | 10 min | Friend asks Q11 |
| System design | 45 min | Friend reads Round 2 prompt + 3 follow-ups |
| Stack deep-dive | 20 min | Friend picks 4 from Round 1 Part B |
| Your questions | 10 min | You ask 3 from §18 |
| Feedback | 5 min | Friend scores using rubric |

---

## Answer templates (if you blank)

**System design opening:**
> "Before I draw, I'd like to clarify scale, consistency requirements, and latency SLA. I'll assume [X] unless you want different constraints."

**Trade-off sandwich:**
> "Option A is simpler but [risk]. Option B scales better but [cost/complexity]. I'd choose B because [business constraint]."

**Failure mode closer:**
> "When [dependency] fails, we [degrade/fail closed/circuit break], and we detect it via [metric/alert]."

**Honest 'I don't know':**
> "I haven't operated that at your scale. My approach would be [first principles], and I'd validate with a spike and load test."

---

## Progress tracker

| Round | Date | Avg score /3 | Weakest topic | Retest date |
|-------|------|--------------|---------------|-------------|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |

When all rounds **≥2.0** and system design rows **≥2**, you're interview-ready for most Technical Architect loops.

---

*Pair with [`interview-study-plan.md`](./interview-study-plan.md) for daily scheduling.*
