<p align="center">
  <img src="https://img.shields.io/badge/Reasoning_Audit-%2497-1a1a2e?style=for-the-badge&labelColor=0f0f1e" alt="Reasoning Audit, $97">
  <img src="https://img.shields.io/badge/Turnaround-48_hours-d97757?style=for-the-badge&labelColor=0f0f1e" alt="48 hour turnaround">
  <img src="https://img.shields.io/badge/Refund-If_no_fix_in_14d-2d6a4f?style=for-the-badge&labelColor=0f0f1e" alt="Refund policy">
</p>

# Reasoning Audit

> Your AI agent fails silently. Find out why in **48 hours, $97**.

Send your agent prompts, code, and one example failure. You get back a 20 to 30 minute Loom walkthrough plus a 2-page Markdown report identifying your top three failure modes with concrete fixes.

**Refund if no fix lands inside 14 days.**

📩 [**Book the audit, $97**](mailto:rain.singlesource@gmail.com?subject=Reasoning%20Audit%20booking&body=Hi%20Rainier%2C%20I%27d%20like%20to%20book%20the%20audit.%20Please%20send%20payment%20instructions%20and%20the%20intake%20checklist.)

5 spots per week. Async, no calls.

---

## How it runs

```mermaid
flowchart LR
    A[Buyer emails<br/>rain.singlesource@gmail.com]:::buyer --> B[Pay $97]:::pay
    B --> C[Send intake<br/>prompts + 1 failure]:::intake
    C --> D[I review<br/>48 hours async]:::work
    D --> E[Loom + Report<br/>+ Contract stub]:::deliver
    E --> F{Fix lands<br/>in 14 days?}:::check
    F -->|Yes| G[Done]:::win
    F -->|No| H[Full refund]:::refund

    classDef buyer fill:#1a1a2e,color:#fff,stroke:#1a1a2e
    classDef pay fill:#d97757,color:#fff,stroke:#d97757
    classDef intake fill:#1a1a2e,color:#fff,stroke:#1a1a2e
    classDef work fill:#1a1a2e,color:#fff,stroke:#1a1a2e
    classDef deliver fill:#1a1a2e,color:#fff,stroke:#1a1a2e
    classDef check fill:#fafaf7,color:#1a1a2e,stroke:#1a1a2e
    classDef win fill:#2d6a4f,color:#fff,stroke:#2d6a4f
    classDef refund fill:#9d0208,color:#fff,stroke:#9d0208
```

---

## Built by

Rainier Potgieter, maintainer of [LOGIC.md](https://github.com/SingularityAI-Dev/logic-md): open-source declarative reasoning layer for AI agents (325 tests, 95.9% branch coverage on the compiler, MIT licensed, validated through Modular9 in production).

## Who this is for

- **You ship LLM agents.** They pass your evals and break in production.
- **You suspect contract drift** but nobody on the team has formalised the reasoning DAG.
- **You don't have a senior** who has shipped ten of these and seen the failure modes recur.

## What you get

### 1. Loom walkthrough (20 to 30 min)
I review your agent prompts, code, and example failures. I name the failure modes on camera so you can hand the recording to your team.

### 2. Markdown report (1 to 2 pages)
Top three failure modes ranked. Each comes with a fix recommendation and code or contract examples you can apply this week.

### 3. LOGIC.md contract stub (if applicable)
If your stack benefits, I write a reasoning contract that makes the top failure mode structurally impossible.

---

## Common failure modes I catch

```mermaid
flowchart TD
    Root([5 most common agent failure modes]):::root

    Root --> F1[Contract drift<br/>Behaviour drifts from prompt's stated contract]:::fail
    Root --> F2[Silent tool-call failures<br/>Tool returns error, agent treats as success]:::fail
    Root --> F3[Retry-loop blow-ups<br/>Cost spikes from poorly-bounded retries]:::fail
    Root --> F4[Planning-without-doing<br/>Agent describes the task instead of executing]:::fail
    Root --> F5[Eval-prod divergence<br/>Passes test fixtures, breaks on real input]:::fail

    F1 --> Fix1[Fix: pre/post assertions on contract boundary]:::fix
    F2 --> Fix2[Fix: structured retry policy + outcome assertion]:::fix
    F3 --> Fix3[Fix: bounded retry with exponential backoff]:::fix
    F4 --> Fix4[Fix: execution mandate + output contract]:::fix
    F5 --> Fix5[Fix: trajectory eval, not just final output]:::fix

    classDef root fill:#1a1a2e,color:#fff,stroke:#1a1a2e
    classDef fail fill:#9d0208,color:#fff,stroke:#9d0208
    classDef fix fill:#2d6a4f,color:#fff,stroke:#2d6a4f
```

---

## Market signal (May 2026)

| Signal | Source |
|---|---|
| 32% of teams cite quality as the top barrier for agents in production | Datadog, State of AI Engineering 2026 |
| Final-output evals miss 20-40% of agent failures vs. trajectory eval | LangChain, State of Agent Engineering 2026 |
| 8.4M rate-limit errors logged across LLM call spans, March 2026 alone | Datadog State of AI Engineering 2026 |
| Mastra (TS-first agents) hit 150k weekly npm downloads, $13M seed | Public reporting |

If your team is in those numbers, this audit is for you.

---

## How it runs (the details)

1. Email to book. Receive intake checklist within 5 minutes.
2. You send: agent prompts, one or two example failures, repo link or relevant code excerpt.
3. I deliver inside 48 hours of intake. Async, no calls. NDA on request.

## FAQ

**What if my framework is exotic?**
I cover vanilla OpenAI/Anthropic API, LangChain, LangGraph, LlamaIndex, CrewAI, AutoGen, Mastra, Vercel AI SDK, and custom orchestration. If your stack is genuinely too far outside that surface and I cannot help, full refund inside 24 hours of intake.

**Refund policy?**
Full refund if no fix recommendation lands inside 14 days of payment. Email and I refund.

**Will you sign an NDA?**
Yes. Send yours with the intake.

**Why $97? Why not more?**
Below the friction threshold for a senior engineer's approval ceiling. Launch price. Goes up once I have testimonials.

---

## Book

Email **rain.singlesource@gmail.com** with subject "Reasoning Audit booking" and I send payment instructions plus the intake checklist.

5 spots per week. When the week's spots fill, listing closes until next Monday.
