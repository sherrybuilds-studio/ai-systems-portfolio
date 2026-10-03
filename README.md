# AI Systems Portfolio

**Shehryar Irfan** · Berlin · [sherrybuilds.com](https://sherrybuilds.com) · [sherry.aiops@gmail.com](mailto:sherry.aiops@gmail.com) · [LinkedIn](https://www.linkedin.com/in/shehryar-irfan-bb5469349)

I build LLM systems for small businesses and run them in production on one Linux server: an AI phone receptionist, a self-healing fleet of 54 Claude Code agents, a job-search pipeline, retrieval assistants on WhatsApp, and a fail-closed validation pipeline for trading strategies.

This repository holds the architecture notes and the dated eval files behind every number I publish. The product code lives in a private monorepo; the public code is listed under [Public code](#public-code).

---

## Systems

| System | What it does | Status | Evidence |
| --- | --- | --- | --- |
| **[AI phone receptionist](./projects/voice-receptionist/)** | Answers calls in German and English, checks availability and books through a FastAPI tool webhook. Says it is an AI (EU AI Act Art. 50) and gives the recording notice (§201 StGB) at the start of every call, then writes a hash-chained evidence record | Deployed on Vapi, demo on request | [12 of 12 golden calls](./evals/2026-09-02-voice-receptionist-eval.json), 2 Sep 2026 |
| **[Agent fleet and self-healer](./architecture/fleet-overview.md)** | 54 Claude Code agents in six teams lease work from a Postgres queue. A self-healer sorts every failed run into a failure class and requeues, skips or escalates it. A lander merges test-only and docs-only agent work once every CI check is green | Running | [2.4% hard failures over 1,000 runs, 9 Jul to 24 Sep](./evals/2026-09-24-fleet-stats.json) · [54 agents defined](./evals/2026-09-26-fleet-agents.json) |
| **[Job pipeline](./projects/cv-job-hunter/)** · [code](https://github.com/sherrybuilds-studio/job-pipeline) | Scores fresh postings every morning and sends a Telegram digest. An on-demand pass pulls company career boards, drops roles that fail hard requirements, checks the posting is still open, then writes a one-page CV and a cover letter that an independent reviewer must clear at zero unsupported claims | Daily at 08:00 UTC | [83 career boards, 2,981 postings in one pass](./evals/2026-09-26-job-pipeline.json), 26 Sep 2026 |
| **[Sales OS](./projects/sales-os/)** | Finds owner-run local businesses on Google Places, scores how many calls each one is likely to miss, and prepares outreach only where consent exists (UWG §7). Nothing sends without human approval | Built, run by hand | [Scorer 10 of 10](./evals/2026-09-02-sales-os-eval.json) · [phase gates](./evals/2026-09-04-sales-os-phase-gates.json) |
| **[WhatsApp product assistant](./projects/commerce-rag-agent/)** · [code](https://github.com/sherrybuilds-studio/commerce-rag-agent) | Answers product questions for a furniture brand from its own catalogue: keyword match first, vector search second, semantic cache at 0.95 cosine | Pilot | Prompt cut 38% (1,118 to 695 tokens per message) when retrieval replaced the full catalogue, [changelog, 27 Apr 2026](https://github.com/sherrybuilds-studio/commerce-rag-agent/blob/main/CHANGELOG.md) |
| **[Restaurant reservation assistant](./projects/restaurant-bot/)** · [code](https://github.com/sherrybuilds-studio/reservation-agent) | Reservations, waitlist, reminders and menu answers over WhatsApp | Built, not deployed | [Retrieval 10 of 10](./evals/2026-09-02-restaurant-bot-eval.json), 2 Sep 2026 · public repo [10 of 10](https://github.com/sherrybuilds-studio/reservation-agent/blob/main/evals/2026-10-01-retrieval-eval.json), 1 Oct 2026, re-run by CI |
| **[Strategy validation pipeline](./projects/trading-pipeline/)** | Crypto strategy research on Freqtrade. A candidate earns paper-trading time only by passing a nine-stage, fail-closed statistical gate: static lookahead scan, data contract, signal sanity, recursion and lookahead analysis, four-fold walk-forward, fee stress at Kraken's published maker rate, Monte Carlo drawdown, deflated Sharpe. Spot, EUR, dry-run only; live capital needs a human-typed confirmation | Research, dry-run only | [12 trials gated, 0 passed, true out-of-sample 729 days](./evals/2026-10-03-trading-gate.json), 3 Oct 2026 |
| **[sherrybuilds.com](https://github.com/sherrybuilds-studio/sherrybuilds.com)** | Next.js portfolio. Its evidence section is generated from dated eval files like the ones here | Live | Released through CI behind a manual approval |

Every linked number comes from a file in [`evals/`](./evals). Each file records its date and the command or query that produced it.

## Architecture

```text
                ┌────────────────── one Ubuntu VPS (Docker + PM2) ───────────────────┐
phone ── Vapi ─▶│ voice receptionist (FastAPI tool webhook) ──▶ Supabase             │
WhatsApp ──────▶│ product and restaurant assistants ──▶ ChromaDB + MiniLM embeddings │
                │ job pipeline (daily cron) ──▶ Telegram digest                      │
                │                                                                    │
                │ dispatcher ──leases──▶ agent_tasks (Postgres) ◀── self-healer      │
                │   ├─ spawns headless Claude Code agents with allowlisted tools     │
                │   └─ agent work ──▶ side branch ──▶ lander (tests and docs only)   │
                └─────────── public traffic only through a Cloudflare tunnel ────────┘
```

- Product LLM calls go through OpenRouter; fleet agents run on Claude.
- The strategy validation pipeline runs on the same server as its own Docker Compose project, outside every path the fleet can touch; the dispatcher refuses tasks that target it.
- The self-healer runs every 30 minutes, a predictive monitor every 10, and no-LLM probes every 5. A health report reaches Telegram at 09:00 Berlin time.
- Details: [fleet](./architecture/fleet-overview.md) · [retrieval stack](./architecture/rag-stack.md) · [eval framework](./architecture/eval-framework.md) · [solutions mapped to modules](./SOLUTIONS.md)

## How the evals work

- Every product has an offline, deterministic gate over a fixed set of golden cases. No live model calls, no secrets.
- The voice, restaurant and portfolio-chat gates run in CI on every push to the monorepo, next to the unit tests of the voice, Sales OS and job apps. A gate below its threshold fails the build.
- A fleet agent (`eval-runner`) re-runs the gates nightly and flags regressions.
- Fleet numbers come from a SQL count over the task table; the method is stored inside each snapshot.

## What I can build next

Each of these reuses parts that already run in the systems above. **None of them is built yet.**

| Agent | What it would do | Built from |
| --- | --- | --- |
| **Quote agent** | Turns an inquiry from a form, e-mail or WhatsApp into a priced quote from the business's own price list and rules. Asks for missing details, and sends only after the owner approves | Catalogue retrieval (product assistant), approval loop (Sales OS), PDF generation (job pipeline) |
| **Speed-to-lead agent** | Replies to a new inbound lead within a minute, asks two or three qualifying questions, books a call and alerts the owner. Contacts only people who reached out first | Booking tools (voice receptionist), WhatsApp session handling and consent gate (Sales OS) |
| **Missed-call text-back** | When a call goes unanswered, texts the caller a booking link on WhatsApp and logs the lead | Voice webhook, WhatsApp channel |
| **Review reply assistant** | Drafts a reply to every new Google review in the business's voice and posts it after approval | Review monitor and reply drafts (reservation-agent); posting needs the Google Business Profile API |
| **No-show reducer** | Reminders 24 hours and 2 hours before an appointment with confirm and cancel replies, plus a waitlist that fills freed slots | Reminders and waitlist (reservation-agent), generalised beyond restaurants |
| **Invoice follow-up agent** | Sends staged, polite reminders for overdue invoices and stops as soon as a payment is recorded | Follow-up stage machine (Sales OS CRM) |
| **Inbox triage agent** | Sorts incoming e-mail into leads, support, invoices and noise, and drafts answers to common questions from the business's own FAQ | Grounded retrieval with refusals (the chat on sherrybuilds.com) |

Want one of these for your business? Write to [sherry.aiops@gmail.com](mailto:sherry.aiops@gmail.com).

## Scope and limits

Every claim above is scoped to what the evidence files cover. Where a system is further along in engineering than in deployment, this is where it says so.

- **Voice receptionist:** runs as one demo tenant on Vapi, with the German and English call flows, booking tools and compliance records in place. Each new business is set up as its own tenant directory with its own prompts and booking configuration.
- **Restaurant assistant:** built and gated in CI, not yet deployed for a restaurant. **Sales OS:** runs by hand, and nothing it prepares is sent without a human approval.
- **Product assistant:** a pilot for one furniture brand. The 38% figure measures prompt size after the switch to retrieval, not a billing total.
- **Strategy validation pipeline:** research in dry-run. It has never placed a real order, and no gated strategy has passed. Every number carries its data label, and backtest results are not a forecast of live performance.
- **Fleet numbers** count hard failures of agent runs on the task queue, not product calls, and come from a SQL count whose method is stored in each snapshot.

## Public code

| Repository | What it is |
| --- | --- |
| [job-pipeline](https://github.com/sherrybuilds-studio/job-pipeline) | The job pipeline, driven by one profile file: career-board sourcing, liveness check, scoring, tailored CV and cover letter with an honesty reviewer, offline tests in CI |
| [reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent) | The restaurant reservation assistant with a fictional demo restaurant, offline tests and a retrieval eval in CI |
| [commerce-rag-agent](https://github.com/sherrybuilds-studio/commerce-rag-agent) | The WhatsApp product assistant: hybrid retrieval, semantic cache, a Meta-signed webhook, offline tests in CI |
| [sherrybuilds.com](https://github.com/sherrybuilds-studio/sherrybuilds.com) | The portfolio site |

The documents in this repository are MIT licensed.
