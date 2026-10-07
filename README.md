# LLM Integration Bot

Design patterns for an LLM support bot that helps developers integrate an API.

Design notes from building and operating an LLM-powered technical support bot for
partners integrating against a logistics **Open API** (order creation, tracking,
shipping labels, webhooks). Written from the product-owner seat: what the bot had to
do, how the pieces fit, and what I would do differently.

Everything here is generalised. No company names, IDs, credentials or internal
documents — only the reusable patterns.

## The problem

External developers integrating with an API hit the same wall over and over:
signature mismatches, wrong parameter formats, webhook misconfiguration, "which
endpoint do I use for X". Each question used to land on a PM or engineer via chat,
got answered ad hoc, and the answer evaporated.

Goals for the bot:

1. Answer the common 80% from the official API docs — **without inventing links or
   error codes**.
2. Collect the *right* diagnostic info (headers, request/response body, signing
   code) before trying to answer.
3. Know when to stop and hand off to a human, with a summary engineers can act on.
4. Get smarter over time without a prompt rewrite every week.
5. Serve two very different audiences from one brain: **external partners** (public
   messenger) and **internal ops/eng** (company chat).

## The five patterns

| # | Pattern | One-liner |
|---|---------|-----------|
| 1 | [Sentinel contract between LLM and code](docs/architecture.md#1-the-sentinel-contract) | The agent's reply starts with a fixed token (`[ADDITIONAL DETAILS REQUIRED]`) when it needs more info; deterministic code branches on it. |
| 2 | [Deterministic loop around a non-deterministic agent](docs/architecture.md#2-the-conversation-loop) | Ask → answer → "did this help?" → retry once → escalate. The LLM never owns the control flow. |
| 3 | [Classifier → router for the internal channel](docs/architecture.md#3-classifier--router-internal-channel) | Form-shaped requests (credential / webhook setup) bypass the LLM entirely; only free-form questions reach it. |
| 4 | [Two-layer knowledge: static docs + learnt KB](docs/architecture.md#4-knowledge-layering) | Official docs are the floor; a write-enabled "learnt" KB captures resolved cases and is searched first. |
| 5 | [Log every turn to a flat table](docs/architecture.md#5-interaction-logging) | Classification, timestamps, "more info needed" flag, satisfaction, escalation — the analytics come for free. |

## Repo map

```
README.md                         ← you are here
LICENSE                           ← CC BY 4.0
docs/architecture.md              ← the patterns in detail, with diagrams and pseudocode
docs/lessons-learned.md           ← what broke, what I'd change, review checklist
prompts/support-agent.template.md ← the agent system prompt, generalised with placeholders
```

## Stack (generalised)

- An internal **agent platform** that hosts LLM agents, lets you bind knowledge bases
  and Python tools to them, and chains agents in a graph ("multi-agent").
- **Rule-driven agents**: plain Python with an SDK (`invoke_tool`, `invoke_agent`)
  used for anything that must be deterministic.
- Two chat surfaces: a public messenger (partners) and the company's internal chat
  (ops / engineering).
- A spreadsheet as the interaction log — deliberately low-tech.

Model choices at the time: a frontier model for the answering agent; a small, cheap
model for the classifier and for rule-driven agents (which barely use the LLM).

## Status

The bot ran in production for partner support. These notes reflect the design as of
late 2026 plus the improvement review in `docs/lessons-learned.md`. Built together
with the platform and engineering teams; I owned the product, prompts and
escalation design.

## License

[CC BY 4.0](LICENSE). Reuse, adapt and share freely — just credit this repo.
