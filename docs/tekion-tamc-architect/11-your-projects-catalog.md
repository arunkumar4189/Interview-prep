# Your projects — use this catalog for every interview example

When the Tekion JD asks for Java, distributed systems, APIs, caching, events, etc., **do not invent a generic “campaign service.”** Pick a row below and tell that story. At the end, one sentence can map to automotive marketing if they push product domain.

| ID | Project (your resume) | Primary artifacts to name | Best for JD topics |
|----|------------------------|---------------------------|-------------------|
| **P1** | North Star B2B SMB/NSE ordering migration | CXP GraphQL aggregate, Domain services, CJCM case layer, Spring Reactive, Cassandra | Distributed architecture, API composition, scale, 99.99% |
| **P2** | Bulk order engine | 10,000+ orders per payload, RabbitMQ, Netflix OSS, per-line consumers | Event-driven, throughput, idempotency, performance |
| **P3** | CartMs, ConfigMs, PaymentMs | Ground-up microservices (2016–2018) | Service boundaries, domain split |
| **P4** | Async B2B cart validation | Large cart payloads off the HTTP thread | API 202/async, system design |
| **P5** | SMB Enrollment & Add-A-Line | OCR ScanID, adaptive auth, automated credit, PEGA | Adapters, fail-closed external checks, workflow |
| **P6** | CJCM case management | Pause & resume on long-running cases | State machines, concurrency on case updates |
| **P7** | Quote-to-order automation | Legacy quoting adapter, internal API libraries | API contracts, cross-team adoption |
| **P8** | Internet-facing path | Akamai, 42Crunch, APIGEE | Edge security, rate limits, gateway vs service auth |
| **P9** | Agentic Universe | React UI, FastAPI, MCP, PostgreSQL, human-in-the-loop | AI platform, persistence, approval gates |
| **P10** | SRE Triaging Service | LangGraph, OpenSearch logs, Git/Jira/K8s context, FastAPI | Observability, AI at the edge, timeouts |
| **P11** | Retail Flex Flow (ATG 11) | Oracle ATG Web Commerce, in-store POS | Early commerce pipelines, transaction time |
| **P12** | Architecture board + GenAI adoption | Guidelines, Claude Code / Copilot / MCP companion | Reviews, quality bars, mentorship |

## Default example per file in this guide

| Study file | Default project for all code/diagrams |
|------------|----------------------------------------|
| 02-java-spring-boot | **P2** bulk batch accept + **P5** credit adapter |
| 03-distributed-microservices | **P1** layers + **P3** Cart/Config/Payment |
| 04-api-and-system-design | **P2** + **P4** + **P7** internal libraries |
| 05-databases-events-caching | **P1** Cassandra + **P9** Postgres + **P2** RabbitMQ |
| 06-concurrency-performance-dsa | **P2** workers + **P6** pause/resume |
| 07-cloud-cicd-observability | **P1/P2** on EKS + **P10** log correlation |
| 08-good-to-have-stack | **P2** RabbitMQ→Kafka, Redis on **P3** ConfigMs, **P10** OpenSearch |
| 09-tamc-domain-scenarios | Map Tekion *prompts* → **P2/P5/P9** (analogy only) |

## One-line analogies (use only after you told the real project)

| Tekion / automotive idea | Your project analogy |
|--------------------------|----------------------|
| Campaign launch fan-out | **P2** bulk payload → RabbitMQ lines |
| Audience snapshot | **P4** frozen cart snapshot at validation start |
| Consent / policy ALLOW-DENY | **P5** credit + adaptive auth fail-closed |
| Pause campaign in flight | **P6** PEGA case pause & resume |
| Payment idempotency | **P3** PaymentMs + forum answer pattern |
| Sold VIN suppress | **P4** line cancelled after cart accepted—dropout tracking (**P1** New Relic) |
| LLM on request path | **P10** RCA is async + human reviewed—not auto-remediate |
| Long-running agent | **P9** Agentic Universe run history + approval |
