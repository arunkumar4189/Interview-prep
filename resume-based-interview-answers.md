# Interview Answers

## Which backend service do you know best? What did it do, and how much traffic or data went through it?

Verizon’s B2B ordering platform: Spring Reactive, GraphQL, Cassandra; SMB Enrollment, Add-A-Line, OCR ScanID, and PEGA integrations; North Star migration with CXP Aggregate, Domain, and CJCM layers. A bulk order engine processes 10,000+ orders per payload via event-driven microservices (RabbitMQ, Netflix OSS). Mission-critical B2B systems run at 99.99% uptime.

---

## How many other teams depended on it?

Multi-vendor teams were mentored on the platform. Secure, reusable internal API libraries were adopted across cross-functional organizational lines.

---

## What was the hardest problem you fixed in it?

Integrated New Relic for proactive order-dropout tracking and sustained reliability on high-volume bulk processing (10K+ orders per payload), with focus on reactive services, observability, and incident debugging.

---

## What is a design decision you made that other teams had to follow? How many teams?

North Star architecture: reactive microservices, GraphQL, and Cassandra across CXP Aggregate, Domain, and CJCM layers. Enterprise architecture board standards for API security and quality bars on production AI and ordering platforms, including Akamai, 42Crunch, and APIGEE on the internet-facing path with minimal legacy impact. Published internal API libraries adopted across cross-functional organizational lines.

---

## What did you choose to buy or adopt instead of building? How did it turn out?

RabbitMQ and Netflix OSS for the bulk order engine. PEGA for case management (pause & resume) and related integrations. LangGraph for the SRE Triaging Service. Claude Code, GitHub Copilot, and MCP Agent Companion for org-wide planning, tests, and code review. Akamai, 42Crunch, and APIGEE for API security. New Relic for observability and order-dropout tracking. Oracle ATG for retail flex flow, which cut in-store POS transaction times.

---

## What have you written down that other engineers still follow today?

Enterprise architecture board guidance on API security and quality bars for production AI and ordering. Internal API libraries used cross-functionally. GenAI evaluation rubrics covering groundedness, citations, instruction-following, and unsafe actions.

---

## What did you personally write code for in the last three months?

Agentic Universe (React, FastAPI, MCP, PostgreSQL) with human-in-the-loop agent validation. SRE Triaging Service (FastAPI, LangGraph) for multi-agent RCA using OpenSearch logs, Git changes, Jira issues, and Kubernetes change windows. MCP Agent Companion and related work across Java, Python, and TypeScript. GenAI evaluation pipelines that score agent traces and LLM outputs.

---

## When did you last fix a live production problem yourself, and what was it?

Production support centers on B2B ordering at 99.99% uptime: reactive services, observability, and incident debugging, including order-dropout issues tracked via New Relic.

---

## Roughly how much of your week is spent writing or reviewing code?

Daily evaluation of AI-generated code and plans; code review through Claude Code, GitHub Copilot, and MCP Agent Companion alongside hands-on FastAPI, LangGraph, and platform delivery work.

---

## What is the AI part of a system you built yourself? Describe what it does, step by step.

**SRE Triaging Service:** A FastAPI service runs LangGraph multi-agent workflows for incident RCA. Agents correlate OpenSearch logs, Git changes, Jira issues, and Kubernetes change windows into evidence-backed RCA narratives. Engineers review multi-agent outputs with citations to logs, Git, Jira, and K8s context.

**Agentic Universe:** Self-service multi-agent hosting with visual workflow builders, run history, and human-in-the-loop validation on React, FastAPI, MCP, and PostgreSQL.

---

## How many people or requests used it, and how often?

Org-wide rollout of Claude Code, GitHub Copilot, and MCP-based agent tooling across multi-stack teams.

---

## What did it get wrong, and what did you change?

GenAI work includes scoring agent traces and LLM outputs for groundedness, citations, instruction-following, and unsafe actions. Academic work uses verify-and-correct loops and prompt evaluation harnesses (zero/few-shot, self-critique, JSON function-calling, CoT/ToT/ReAct, strategy router).

---

## Which parts did you write, and which did someone else write?

Lead GenAI roadmap and delivery for Agentic Universe and SRE Triaging Service; org-wide AI tooling and architecture board governance.

---

## Which AI tools do you use at work, and what for?

Claude Code, GitHub Copilot, and MCP Agent Companion for planning, tests, and code review in Java, Python, and TypeScript. Daily evaluation of AI-generated code and plans from Claude Code, Copilot, and MCP agents. Production agent work uses LangGraph and MCP.

---

## What do you never let an AI assistant write?

Production changes follow architecture board API security and quality bars; human-in-the-loop validation on agents; evaluation of unsafe actions and instruction-following before trusting outputs.

---

## Tell me about a time AI-written code caused a problem. What changed afterwards?

Evaluation focus on AI-generated code and plans: debugging complex workflows, explaining trade-offs, and rubric-based scoring for correctness, performance, and clarity.

---

## Have you got other engineers working this way? What changed for them?

Drove org-wide adoption of Claude Code, GitHub Copilot, and MCP Agent Companion across multi-stack teams for planning, testing, and code reviews.

---

## What stops one of your AI features from doing something it should not?

Human-in-the-loop agent validation on Agentic Universe. Scoring unsafe actions in GenAI evaluation. API security and quality bars for production AI on the enterprise architecture board.

---

## What happens if the AI asks for information the user is not allowed to see?

API security and quality bars on production AI and ordering platforms; secured internet-facing path with Akamai, 42Crunch, and APIGEE.

---

## Has an AI feature of yours ever done something unintended? What did you change?

Ongoing agent trace and LLM output scoring for groundedness, citations, instruction-following, and unsafe actions.

---

## How do you decide which AI model to use for which job?

M.Tech work uses prompt evaluation harnesses and a strategy router across zero/few-shot, self-critique, JSON function-calling, and CoT/ToT/ReAct; models include QLoRA fine-tuned GPT-2 Medium, Flan-T5, BART, and DistilBERT+LoRA in project contexts.

---

## An AI feature gives a customer a wrong answer. How do you find out what happened?

Score agent traces and LLM outputs on groundedness, citations, instruction-following, and unsafe actions.

---

## How do you know a change has not made an AI feature worse?

Rubric-based LLM scoring, prompt evaluation harnesses, and repeated GenAI evaluation on groundedness, citations, instruction-following, and unsafe actions.

---

## What do you measure on an AI feature that you would not measure on an ordinary service?

Groundedness, citations, instruction-following, and unsafe actions on agent traces and LLM outputs.
