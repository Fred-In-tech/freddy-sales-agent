# Architecture — Freddy Sales Agent

Deeper notes on how the agent is actually put together. Paths reference the private codebase.

## 1. Identity as configuration

Freddy's behavior is defined entirely in markdown directives loaded on session start:

| File | Role |
|---|---|
| `SOUL.md` | Who the agent is — a sales operator, explicitly *not* a generic chatbot |
| `AGENTS.md` | Mission, responsibilities, priority order |
| `TOOLS.md` | The approved tool layer and the mandatory pipeline order |
| `HEARTBEAT.md` | The cron cadence and what each job may and may not do |
| `MEMORY.md` | Operational state, including the `OUTREACH_MODE` flag |

This split matters: changing the agent's judgment, tools, or cadence are three different edits with three different blast radii. The runtime (OpenClaw) treats tools as callable functions; skills teach when and how to use them.

## 2. The mandatory SDR pipeline

`TOOLS.md` encodes a five-stage order that may not be skipped:

1. **Discover** — Tavily search for companies in the focus vertical
2. **Qualify** — Firecrawl + a structured-reasoning skill scores Strong/Medium/Weak; Weak is dropped before any spend
3. **Enrich** — Hunter.io finds the decision-maker and verifies the email
4. **CRM entry** — the lead lands in ClickUp with status, source, and all data
5. **Outreach prep** — draft from cold-outreach frameworks + the outreach playbook, referencing specific research findings; a ClickUp follow-up task is created with a date

The invariant: **no unverified email ever enters the CRM, and no outreach exists without a logged lead and a follow-up task.** A complementary OpenStreetMap skill (`local-business-finder`, a small Python script against free OSM data) sources local businesses and flags the ones with no website — the highest-fit prospects for a media company.

## 3. Grounding: the knowledge base

The KB is 8 markdown files (~244KB) generated from the production website codebase, whose `lib/constants.js` is declared the single source of truth. It was adversarially fact-checked before deployment — 34 corrections — and is indexed by the runtime's memory search.

- `kb-index.md` is read first on every inbound email and routes deeper questions.
- `kb-grounding-rule.md` is a hard rule: search the KB before answering anything about services, pricing, process, coverage, or terms. Never answer from model knowledge.
- The mandated fallback is the literal phrase **"NOT IN KB"** — the agent is told what to say when it doesn't know, which is the piece most grounding schemes forget.
- `kb-gaps.md` enumerates what the KB deliberately does not cover, read before promising anything.

Result observed in practice: invented prices stopped. Grounding beat prompting.

## 4. The approval gate

`outreach-guardrails` is classified non-negotiable and overrides any instruction to send more or faster:

- **REVIEW mode (current):** drafts are posted to the owner's Telegram, numbered (`Draft 1 → contact, company`). Replies of `send`, `send all`, or `send 1,3` control exactly what goes out via Composio Gmail.
- **AUTO-SEND** exists but is gated behind deliverability work (SPF+DKIM are verified; DMARC hardening and warmup are prerequisites) and an explicit flag flip in `MEMORY.md`.
- Send caps, a suppression list, and unsubscribe compliance apply in both modes.

## 5. Cron in REVIEW mode

Three OpenClaw cron jobs (Mon–Fri, America/Chicago): `freddy-morning-prospecting` (9:30), `freddy-followups` (14:00), `freddy-eod-report` (17:30). Each runs the pipeline, drafts, and reports to Telegram — none can send email while `OUTREACH_MODE = REVIEW`. An early operational lesson: the jobs originally depended on a local model running on a laptop, which meant the laptop had to be awake at trigger time; that dependency drove the move to hosted inference.

## 6. Model journey and the compaction-loop incident

Local Qwen (llama.cpp) → Kimi K2.6 → NVIDIA-served GLM-5.2.

The defining bug: replies of 6–11 minutes with no errors. Root cause — the local model had an **8K context window** and the assembled bootstrap prompt measured **8.8K tokens**. Every turn overflowed, triggered compaction, and the compacted state re-overflowed: an infinite compaction loop before any useful work. The fix was to measure the prompt against the window and trim, not to tune prompts blindly. Corollary rules from the same era: a hosted catalog can list retired models (404 = retired, 401/403 = credential), and replacements get chosen by a scripted latency/tone test battery.

## 7. Fleet context

Freddy is one of three agents on the same host, each in its own Docker container (`docker-compose.yml`, restart unless-stopped, volume-mounted identity/knowledge/logs). Nana (CEO orchestrator) directs and quality-gates; Jade runs the same motion for the studio sub-brand. Because Telegram bots cannot see other bots' messages, agent-to-agent traffic uses a file message bus — per-agent inbox/outbox/processed directories, a 4-second watcher service for instant delivery, cron sweeps as backup, and dedupe state. The full loop (orchestrator → agent → orchestrator → owner Telegram) is verified.

Secrets (API keys, gateway config) live in environment files and a runtime config that are never checked into any public artifact, including this one.
