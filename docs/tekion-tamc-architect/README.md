# Tekion TAMC Architect — technical study guide (markdown)

Detailed interview prep for **Architect, Tekion Automotive Marketing Cloud (TAMC)**, written for a **hands-on backend architect** audience—not executive summaries.

**Tailored to:** Arun Kumar (Verizon Distinguished Engineer, 15+ years, Spring Reactive ordering, bulk engine, Agentic Universe, SRE triaging).

## How to use these files

1. Read [00-role-jd-map.md](./00-role-jd-map.md) once—every JD bullet mapped to a file.
2. Study [02-java-spring-boot.md](./02-java-spring-boot.md) and [05-databases-events-caching.md](./05-databases-events-caching.md) until you can **write** the examples on paper.
3. Rehearse [01-resume-stories.md](./01-resume-stories.md) out loud with **numbers and failure modes**.
4. Use [10-reported-tekion-questions.md](./10-reported-tekion-questions.md) for mock loops (Kafka, payment idempotency, booking, policy engine).
5. Skim [09-tamc-domain-scenarios.md](./09-tamc-domain-scenarios.md) before the interview—dealer/VIN/consent/spend language.

## File index

| File | JD coverage |
|------|-------------|
| [00-role-jd-map.md](./00-role-jd-map.md) | Full Skills & Qualifications + Good to Have → where to study |
| [01-resume-stories.md](./01-resume-stories.md) | 12+ years, hands-on, reviews, cross-team—**scripts with proof** |
| [02-java-spring-boot.md](./02-java-spring-boot.md) | Java, Spring / Spring Boot, transactions, Reactive vs MVC |
| [03-distributed-microservices.md](./03-distributed-microservices.md) | Distributed systems, boundaries, failures, your North Star layers |
| [04-api-and-system-design.md](./04-api-and-system-design.md) | API design, backend system design, campaign launch walkthrough |
| [05-databases-events-caching.md](./05-databases-events-caching.md) | DB design, SQL/NoSQL, caching, event-driven, outbox |
| [06-concurrency-performance-dsa.md](./06-concurrency-performance-dsa.md) | Concurrency, performance tuning, DSA Tekion has asked |
| [07-cloud-cicd-observability.md](./07-cloud-cicd-observability.md) | Cloud-native, K8s, Docker, AWS, CI/CD, logging, monitoring |
| [08-good-to-have-stack.md](./08-good-to-have-stack.md) | Kafka, Redis, Elasticsearch, MongoDB, Aerospike |
| [09-tamc-domain-scenarios.md](./09-tamc-domain-scenarios.md) | Marketing cloud: campaign, VIN, consent, ledger, AI at the edge |
| [10-reported-tekion-questions.md](./10-reported-tekion-questions.md) | Forum write-ups with **how you answer from your work** |

## One sentence to keep

**Commit the business record in our database, publish work after commit, and treat every vendor and model as an adapter with a timeout and a defined failure state.**

That sentence is how you ran ordering at 99.99%; it is how TAMC should treat ESPs, ad platforms, and LLMs.
