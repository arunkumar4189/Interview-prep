# Interview Answers (Resume-Only)

Answers derived **only** from the two resumes (`Arun_Kumar_DataAnnotation_Resume_0384.pdf` and `Arun_Kumar_Resume.pdf_4c49.pdf`). Where the resumes do not state something, that is called out explicitly.

---

## Which backend service do you know best? What did it do, and how much traffic or data went through it?

**Verizon B2B ordering**: Spring Reactive, GraphQL, Cassandra; SMB Enrollment, Add-A-Line, OCR ScanID, PEGA; North Star migration with CXP Aggregate, Domain, and CJCM layers. A **bulk order engine** processes **10,000+ orders per payload** using event-driven microservices (RabbitMQ, Netflix OSS). The resumes cite **99.99% uptime** on mission-critical B2B systems. They do **not** give overall requests/sec, daily order volume, or data-store size.

---

## How many other teams depended on it?

The resumes say you **mentored multi-vendor teams**, published **secure, reusable internal API libraries adopted across cross-functional organizational lines**, and worked on **org-wide** Claude Code / Copilot / MCP adoption. They do **not** give a team count.

---

## What was the hardest problem you fixed in it?

The resumes do **not** label a single “hardest” fix. Related facts: **proactive order-dropout tracking** with **New Relic**; mentoring on **reactive services, observability, and incident debugging**; bulk processing at **10K+ orders per payload**.

---

## What is a design decision you made that other teams had to follow? How many teams?

Documented decisions include:

- **North Star architecture** (reactive microservices, GraphQL, Cassandra; CXP Aggregate / Domain / CJCM).
- **Enterprise architecture board** guidance: **API security and quality bars** for production AI and ordering; internet-facing path **Akamai, 42Crunch, APIGEE** with **minimal legacy impact** (second resume).
- **Internal API libraries** adopted **across cross-functional organizational lines**.

**Team count is not stated.**

---

## What did you choose to buy or adopt instead of building? How did it turn out?

| Adopted (per resume) | Outcome on resume |
|----------------------|-------------------|
| **RabbitMQ**, **Netflix OSS** (bulk engine) | Not described beyond use in bulk ordering |
| **PEGA** (case management, pause & resume, integrations) | Not described |
| **Oracle ATG** (earlier retail flex flow) | **Cut in-store POS transaction times** |
| **Claude Code**, **GitHub Copilot**, **MCP Agent Companion** | **Org-wide** rollout; planning, tests, code review |
| **LangGraph** (SRE Triaging Service) | Automates incident RCA with cited context |
| **Akamai, 42Crunch, APIGEE** | Secured internet-facing path; **minimal legacy impact** |
| **New Relic** | Observability; **order-dropout tracking**; supports **99.99% uptime** narrative |

---

## What have you written down that other engineers still follow today?

The resumes mention **architecture board** standards (**API security and quality bars**), **internal API libraries** used cross-functionally, and **GenAI evaluation** criteria: **groundedness, citations, instruction-following, unsafe actions**. They do **not** name specific ADRs or doc titles.

---

## What did you personally write code for in the last three months?

**Sep 2025 – Present** (Distinguished Engineer):

- **Agentic Universe** — React, FastAPI, MCP, PostgreSQL; human-in-the-loop agent validation.
- **SRE Triaging Service** — FastAPI, LangGraph; multi-agent RCA citing **logs, Git, Jira, K8s** (second resume: **OpenSearch** logs, Git, Jira, K8s change windows).
- **Org-wide** Claude Code / Copilot and **MCP Agent Companion** (Java, Python, TS).
- **GenAI evaluation**: scoring agent traces and LLM outputs.

Resumes do **not** split your lines vs others’ on these systems.

---

## When did you last fix a live production problem yourself, and what was it?

Resumes mention **incident debugging** and **99.99% uptime** B2B work but **no dated production incident** or specific fix attributed to you.

---

## Roughly how much of your week is spent writing or reviewing code?

**Not on the resumes.** They do say you **evaluate AI-generated code and plans** daily and lead **code review** via Copilot/MCP Companion.

---

## What is the AI part of a system you built yourself? Describe what it does, step by step.

**SRE Triaging Service** (FastAPI, **LangGraph**): automates incident **RCA** via multi-agent workflows that pull **OpenSearch logs, Git changes, Jira issues, and K8s change windows** into **evidence-backed RCA narratives**; outputs are **reviewed** (first resume: review multi-agent RCA outputs citing logs, git, Jira, K8s).

**Agentic Universe**: self-service **multi-agent hosting**, **visual workflow builders**, **run history**, **human-in-the-loop validation** (React, FastAPI, MCP, PostgreSQL).

Resumes do **not** give a step-by-step internal graph; only these capabilities.

---

## How many people or requests used it, and how often?

**Not on the resumes** (no user counts or request rates for Agentic Universe or SRE Triaging).

---

## What did it get wrong, and what did you change?

**Not on the resumes** for those production systems.

**M.Tech / portfolio** (separate from Verizon): medical QA with **verify-and-correct loop**; prompt eval harness with **zero/few-shot, self-critique, JSON function-calling, CoT/ToT/ReAct, strategy router** — implies iteration on prompts/strategies, not a stated production failure.

---

## Which parts did you write, and which did someone else write?

**Not on the resumes.**

---

## Which AI tools do you use at work, and what for?

**Claude Code**, **GitHub Copilot**, **MCP Agent Companion** — planning, tests, code review (Java, Python, TypeScript). Daily work **evaluating AI-generated code and plans** (Claude Code, Copilot, MCP agents). Stack includes **LangGraph, MCP, Hugging Face, PEFT** where relevant to your roles and coursework.

---

## What do you never let an AI assistant write?

**Not on the resumes.**

---

## Tell me about a time AI-written code caused a problem. What changed afterwards?

**Not on the resumes.**

---

## Have you got other engineers working this way? What changed for them?

You **drove org-wide adoption** of **Claude Code, GitHub Copilot, and MCP-based agent tooling** across **multi-stack teams**; designed **MCP Agent Companion** for planning, testing, and reviews. Resumes do **not** describe measurable changes for those engineers.

---

## What stops one of your AI features from doing something it should not?

Resumes state:

- **Human-in-the-loop agent validation** (Agentic Universe).
- Scoring **unsafe actions** (GenAI evaluation).
- **API security and quality bars** on the architecture board for **production AI**.

No further guardrail detail (tool allowlists, RBAC, etc.) on the resumes.

---

## What happens if the AI asks for information the user is not allowed to see?

**Not on the resumes.**

---

## Has an AI feature of yours ever done something unintended? What did you change?

**Not on the resumes.**

---

## What does one of your AI features cost to run? How do you know?

**Not on the resumes.**

---

## What change made it cheaper or faster, and by how much?

**Not on the resumes.**

---

## How do you decide which AI model to use for which job?

**Not on the resumes** for Verizon production.

**Coursework**: prompt eval harness with multiple strategies and a **strategy router**; models mentioned include **GPT-2 Medium (QLoRA)**, **Flan-T5**, **BART**, **DistilBERT+LoRA**, etc., in academic projects — not tied to a Verizon routing policy on the resume.

---

## An AI feature gives a customer a wrong answer. How do you find out what happened?

Resumes: you **score agent traces and LLM outputs** on **groundedness, citations, instruction-following, unsafe actions**. They do **not** describe an end-to-end customer-incident process.

---

## How do you know a change has not made an AI feature worse?

Resumes: **rubric-based LLM scoring**, **prompt evaluation harnesses**, and ongoing **GenAI evaluation** (groundedness, citations, instruction-following, unsafe actions). No deploy gates or regression percentages stated.

---

## What do you measure on an AI feature that you would not measure on an ordinary service?

From your resumes, AI-specific evaluation includes:

- **Groundedness**
- **Citations**
- **Instruction-following**
- **Unsafe actions**

(Plus general platform **99.99% uptime** and **order-dropout** style ops metrics on B2B systems.)

---

## Summary

Your resumes support strong answers on **B2B ordering (10K+/payload, 99.99%, reactive/GraphQL/Cassandra)**, **SRE Triaging / Agentic Universe**, **org-wide AI dev tools**, and **eval rubrics**. They do **not** support specific numbers on dependent teams, traffic, cost, % coding time, production war stories, or AI guardrail edge cases unless you add those from memory in the interview.
