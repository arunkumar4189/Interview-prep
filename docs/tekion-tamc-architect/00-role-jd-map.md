# Role overview and JD map

## What Tekion is hiring for (TAMC Architect)

**Product:** Tekion Automotive Marketing Cloud (TAMC)—AI-native marketing for automotive partners (dealers, OEMs).

**Technical spine (from the JD):**

- Java microservices, event-driven pipelines, caching/serving (Redis, Aerospike).
- APIs and service contracts at enterprise scale.
- **Hexagonal / adapter patterns** so AI, model serving, and external intelligence do not couple to core business logic.
- Production excellence: CI/CD, observability, security, mentorship—**while still writing code**.

**Location:** Chennai.

---

## Skills & Qualifications → study file

| JD requirement | What interviewers actually probe | Read |
|----------------|----------------------------------|------|
| 12+ years backend & architecture | Timeline, hardest system, what you still own in code | [01-resume-stories.md](./01-resume-stories.md) |
| Java, Spring / Spring Boot | Transaction boundaries, idempotency, Reactive vs Boot | [02-java-spring-boot.md](./02-java-spring-boot.md) |
| Distributed systems & microservices | Service boundaries, sync vs async, failure modes | [03-distributed-microservices.md](./03-distributed-microservices.md) |
| Backend system design | End-to-end flows, NFRs with numbers, one final design | [04-api-and-system-design.md](./04-api-and-system-design.md) |
| API design | Idempotency, errors, pagination, versioning | [04-api-and-system-design.md](./04-api-and-system-design.md) |
| Multithreading & concurrency | Races, pools, virtual threads, DB as lock | [06-concurrency-performance-dsa.md](./06-concurrency-performance-dsa.md) |
| Data structures & algorithms | Medium coding if loop includes it | [06-concurrency-performance-dsa.md](./06-concurrency-performance-dsa.md) |
| Database design | Postgres vs Cassandra, schema for workflow + ledger | [05-databases-events-caching.md](./05-databases-events-caching.md) |
| Caching strategies | Cache-aside, stampede, what must not be cache-only | [05-databases-events-caching.md](./05-databases-events-caching.md) |
| Event-driven systems | At-least-once, ordering, outbox, DLQ | [05-databases-events-caching.md](./05-databases-events-caching.md) |
| Performance tuning | Bottleneck you measured, backpressure, pools | [06-concurrency-performance-dsa.md](./06-concurrency-performance-dsa.md) |
| High-throughput, low-latency, resilient | Bulk engine, dropout metrics, degradation | [06-concurrency-performance-dsa.md](./06-concurrency-performance-dsa.md) |
| Cloud-native, containers, deployment | EKS, probes, rollouts | [07-cloud-cicd-observability.md](./07-cloud-cicd-observability.md) |
| CI/CD, observability, production ops | Pipeline, alerts on business signals | [07-cloud-cicd-observability.md](./07-cloud-cicd-observability.md) |
| Relational & NoSQL | When each store, CAP in one sentence | [05-databases-events-caching.md](./05-databases-events-caching.md) |
| Architecture reviews & quality | Concrete review comments you block | [01-resume-stories.md](./01-resume-stories.md) |
| Collaboration across product & engineering | Quote-to-order, disagreement story | [01-resume-stories.md](./01-resume-stories.md) |
| Currently hands-on | Recent modules (Agentic Universe, SRE, bulk engine) | [01-resume-stories.md](./01-resume-stories.md) |

## Good to have → study file

| JD good to have | Your honest position | Read |
|-----------------|----------------------|------|
| Kafka | RabbitMQ in production; learn Kafka debug + mapping | [08-good-to-have-stack.md](./08-good-to-have-stack.md) |
| Redis | On resume; rate limit + cache patterns | [08-good-to-have-stack.md](./08-good-to-have-stack.md) |
| Elasticsearch | OpenSearch in SRE service | [08-good-to-have-stack.md](./08-good-to-have-stack.md) |
| MongoDB | Not on resume; one justified document use case | [08-good-to-have-stack.md](./08-good-to-have-stack.md) |
| Aerospike | Not operated; profile/cap pattern from Cassandra+Redis | [08-good-to-have-stack.md](./08-good-to-have-stack.md) |
| AI / GenAI integration | Agentic Universe, LangGraph, MCP | [09-tamc-domain-scenarios.md](./09-tamc-domain-scenarios.md) + [08-good-to-have-stack.md](./08-good-to-have-stack.md) |
| Decoupled domain from intelligence | Ports/adapters; PEGA/credit/OCR precedent | [09-tamc-domain-scenarios.md](./09-tamc-domain-scenarios.md) |
| Kubernetes, Docker, AWS | EKS, Jenkins/GitLab CI on resume | [07-cloud-cicd-observability.md](./07-cloud-cicd-observability.md) |

---

## Interview loop (from public Tekion reports—not guaranteed for Architect)

Many candidates report **knockout rounds**:

1. **Coding** — 1–2 mediums (intervals, histogram, rotated array, islands).
2. **LLD** — classes + invariants (elevator, parking lot, TTL cache, policy engine, Log4j-style logger).
3. **HLD** — architect-led (food delivery, service booking, e-commerce, chat, distributed cache, agent platform).
4. **Manager** — your project in depth; behavioral.

**Staff-level failure mode:** fumbling “describe your hardest project” *before* the design starts.

Confirm with recruiter whether **Architect** includes coding and which stack (Java expected for TAMC).

See [10-reported-tekion-questions.md](./10-reported-tekion-questions.md).
