# Tekion TAMC Architect — technical study guide (markdown)

Detailed interview prep for **Architect, Tekion Automotive Marketing Cloud (TAMC)**.

**Rule for this guide:** Every worked example (code, SQL, diagrams, metrics) comes from **your projects**—Verizon ordering, bulk engine, Cart/Config/Payment, Agentic Universe, SRE triaging—not generic “campaign services.”

**Start here:** [11-your-projects-catalog.md](./11-your-projects-catalog.md) (P1–P12).

**Tailored to:** Arun Kumar — Distinguished Engineer, Chennai, 15+ years.

## How to use

1. [11-your-projects-catalog.md](./11-your-projects-catalog.md) — which project to cite per topic.
2. [02-java-spring-boot.md](./02-java-spring-boot.md) + [05-databases-events-caching.md](./05-databases-events-caching.md) — write examples on paper (**P2** bulk accept).
3. [01-resume-stories.md](./01-resume-stories.md) — rehearse aloud.
4. [10-reported-tekion-questions.md](./10-reported-tekion-questions.md) — map each forum question → **P1–P12**.
5. [09-tamc-domain-scenarios.md](./09-tamc-domain-scenarios.md) — only when they ask automotive product; answer with **P2/P5/P9** first.

## File index

| File | Primary projects used |
|------|------------------------|
| [00-role-jd-map.md](./00-role-jd-map.md) | JD → files |
| [01-resume-stories.md](./01-resume-stories.md) | P1–P10 narratives |
| [02-java-spring-boot.md](./02-java-spring-boot.md) | **P2**, **P5** |
| [03-distributed-microservices.md](./03-distributed-microservices.md) | **P1**, **P3**, **P2** |
| [04-api-and-system-design.md](./04-api-and-system-design.md) | **P2**, **P4**, **P1**, **P7** |
| [05-databases-events-caching.md](./05-databases-events-caching.md) | **P1**, **P2**, **P3**, **P9** |
| [06-concurrency-performance-dsa.md](./06-concurrency-performance-dsa.md) | **P2**, **P3**, **P6**, **P1** |
| [07-cloud-cicd-observability.md](./07-cloud-cicd-observability.md) | **P1/P2**, **P8**, **P10** |
| [08-good-to-have-stack.md](./08-good-to-have-stack.md) | **P2** Rabbit, **P3** Redis, **P10** OpenSearch |
| [09-tamc-domain-scenarios.md](./09-tamc-domain-scenarios.md) | Tekion prompts → your projects |
| [10-reported-tekion-questions.md](./10-reported-tekion-questions.md) | **Detailed** forum Q&A + scripts (Kafka, payments, HLD/LLD, DSA, behavioral) |
| [11-your-projects-catalog.md](./11-your-projects-catalog.md) | **Master index** |

## One sentence to keep

**Commit the batch or order in our store, publish line work after commit, and treat credit, PEGA, quote, and models as adapters with a timeout and a defined failure state.**

That is **P2 + P5**; it is the same architectural rule Tekion wants for marketing adapters.
