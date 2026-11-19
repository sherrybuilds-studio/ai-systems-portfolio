# AI Systems Portfolio

**Muhammad Shehryar** · Berlin · [sherrybuilds.com](https://sherrybuilds.com) · [sherry.aiops@gmail.com](mailto:sherry.aiops@gmail.com) · [LinkedIn](https://www.linkedin.com/in/shehryar-irfan-bb5469349)

Frontkept founder since Oct 2026 · B.Sc. Computer Science, Arden University Berlin, Feb 2025 to Feb 2028, second year completed

I design, build and operate LLM systems for small businesses. The portfolio covers Frontkept, my AI phone assistant for Berlin estate agents and property managers, an autonomous agent fleet with a self-healer, a job-search pipeline, retrieval assistants on WhatsApp and a fail-closed validation pipeline for trading strategies.

This repository holds the architecture notes and the dated evaluation files behind every number I publish. The product code is kept in a private monorepo, and the public code is listed under [Public code](#public-code).

---

## Systems

| System | What it does | Status | Evidence |
| --- | --- | --- | --- |
| **[Frontkept, AI phone assistant](./projects/voice-receptionist/)** | For Berlin estate agents and property managers. Clara and Paul are built from one code base, and each runs on its own server process of mine: Clara runs on ElevenLabs Agents and takes signed web calls, and Paul runs on OpenAI Realtime on a test line connected by SIP. It says in its first sentence that it is an AI, answers in German, with English in testing, and records no audio. Each call's AI-disclosure check is logged in a SHA-256 hash-chained journal (one chain per day), since 23 Aug 2026. Frontkept is new | Built, run on demand; demo on request | Each new version passes a call simulator, the test suite and a separate review before I switch it on |
| **[Agent fleet and self-healer](./architecture/fleet-overview.md)** | An autonomous agent fleet of headless Claude Code agents, in six role groups, leases work from a Postgres queue. A self-healer sorts every failed run into a failure class and requeues, skips or escalates it. A lander merges test-only and docs-only agent work once every CI check is green. Beside it, I work with ten parallel Claude Code lanes, each with a fixed role, a security lane that reads every change, and my approval before anything is released | Running | [6,598 runs (6,438 tasks) since 10 Jun 2026](./evals/2026-11-18-fleet-runs.json), as of 18 Nov 2026 |
| **[Job pipeline](./projects/cv-job-hunter/)** · [code](https://github.com/sherrybuilds-studio/job-pipeline) | Scores fresh postings and sends a Telegram digest. A sourcing pass pulls company career boards, drops roles that fail hard requirements, checks the posting is still open, then writes a one-page CV and a cover letter that an independent AI reviewer must clear at zero unsupported claims | Built, run on demand (daily until 13 Nov 2026) | [83 career boards, 2,981 postings in one pass](./evals/2026-09-26-job-pipeline.json), 26 Sep 2026 |
| **[Sales OS](./projects/sales-os/)** | Finds owner-run local businesses on Google Places, scores how many calls each one is likely to miss, and prepares outreach only where consent exists. Nothing sends without human approval | Built, run by hand | [Scorer 10 of 10](./evals/2026-09-02-sales-os-eval.json) · [phase gates](./evals/2026-09-04-sales-os-phase-gates.json) |
| **[WhatsApp product assistant](./projects/commerce-rag-agent/)** | Answers product questions for a furniture brand from its own catalogue: keyword match first, vector search second, semantic cache at 0.95 cosine | Pilot, paused since 10 Oct 2026 | Prompt cut 38% (1,118 to 695 tokens per message) when retrieval replaced the full catalogue, measured 27 Apr 2026 (code private) |
| **[Restaurant reservation assistant](./projects/restaurant-bot/)** · [code](https://github.com/sherrybuilds-studio/reservation-agent) | Reservations, waitlist, reminders and menu answers over WhatsApp | Built, not deployed | [Retrieval 10 of 10](./evals/2026-09-02-restaurant-bot-eval.json), 2 Sep 2026 · public repo [10 of 10](https://github.com/sherrybuilds-studio/reservation-agent/blob/main/evals/2026-10-01-retrieval-eval.json), 1 Oct 2026, re-run by CI |
| **[Strategy validation pipeline](./projects/trading-pipeline/)** | Crypto strategy research on Freqtrade. A candidate earns paper-trading time only by passing a nine-stage, fail-closed statistical gate: static lookahead scan, data contract, signal sanity, recursion and lookahead analysis, four-fold walk-forward, fee stress at Kraken's published maker rate, Monte Carlo drawdown and deflated Sharpe. Spot trading in euros, dry run only | Research, dry-run only | [12 trials gated, 0 passed, true out-of-sample 729 days](./evals/2026-10-03-trading-gate.json), 3 Oct 2026 |
| **[sherrybuilds.com](https://sherrybuilds.com)** | Next.js portfolio. Its evidence section is generated from dated eval files like the ones here; the code is private | Online | Released through CI behind a manual approval |

Activity: about 2,400 commits in my code since May 2026, built with AI agents under my review ([method](./evals/2026-11-18-fleet-runs.json)).

Every linked number comes from a file in [`evals/`](./evals), and each file records its date and how it was produced. The 38% is recorded in the WhatsApp assistant's changelog, which is private.

## Architecture

```text
                ┌──────────────────────── single-server stack ───────────────────────┐
web + SIP ─────▶│ phone assistant: Clara and Paul, one code base, two processes      │
WhatsApp ──────▶│ product and restaurant assistants ──▶ ChromaDB + MiniLM embeddings │
                │ job pipeline (on demand) ──▶ Telegram digest                       │
                │                                                                    │
                │ dispatcher ──leases──▶ task queue (Postgres) ◀── self-healer       │
                │   ├─ spawns headless Claude Code agents with allowlisted tools     │
                │   └─ agent work ──▶ side branch ──▶ lander (tests and docs only)   │
                └────────────────────────────────────────────────────────────────────┘
```

- Product LLM calls go through OpenRouter, and the fleet agents run on Claude.
- The strategy validation pipeline is isolated from the agent fleet. The dispatcher refuses any task that targets it.
- The self-healer runs every 30 minutes, a predictive monitor every 10 and rule-based health probes every 5. A health report is sent to Telegram at 09:00 Berlin time.
- Details: [fleet](./architecture/fleet-overview.md) · [retrieval stack](./architecture/rag-stack.md) · [eval framework](./architecture/eval-framework.md) · [solutions mapped to modules](./SOLUTIONS.md)

## How the evals work

- Every product has an offline, deterministic gate over a fixed set of golden cases, with no live model calls and no secrets.
- The voice, restaurant and portfolio-chat gates run in the monorepo's CI, next to the unit tests of the voice, Sales OS and job apps. A gate below its threshold fails the build.
- A fleet agent (`eval-runner`) re-runs the gates nightly and flags regressions.
- Fleet numbers come from a SQL count over the task table, and the method is stored inside each snapshot.

## What I can build next

Each of these reuses parts that already exist in the systems above. **None of them is built yet.**

| Agent | What it would do | Built from |
| --- | --- | --- |
| **Quote agent** | Turns an inquiry from a form, e-mail or WhatsApp into a priced quote from the business's own price list and rules. Asks for missing details, and sends only after the owner approves | Catalogue retrieval (product assistant), approval loop (Sales OS), PDF generation (job pipeline) |
| **Speed-to-lead agent** | Replies to a new inbound lead within a minute, asks two or three qualifying questions, books a call and alerts the owner. Contacts only people who reached out first | Booking tools (voice receptionist), WhatsApp session handling and consent gate (Sales OS) |
| **Missed-call text-back** | When a call goes unanswered, texts the caller a booking link on WhatsApp and logs the lead | Voice webhook, WhatsApp channel |
| **Review reply assistant** | Drafts a reply to every new Google review in the business's voice and posts it after approval | Review monitor and reply drafts (reservation-agent); posting needs the Google Business Profile API |
| **No-show reducer** | Reminders 24 hours and 2 hours before an appointment with confirm and cancel replies, plus a waitlist that fills freed slots | Reminders and waitlist (reservation-agent), generalised beyond restaurants |
| **Invoice follow-up agent** | Sends staged, polite reminders for overdue invoices and stops as soon as a payment is recorded | Follow-up stage machine (Sales OS CRM) |
| **Inbox triage agent** | Sorts incoming e-mail into leads, support, invoices and noise, and drafts answers to common questions from the business's own FAQ | Grounded retrieval with refusals (the chat on sherrybuilds.com) |

To discuss one of these for your business, contact [sherry.aiops@gmail.com](mailto:sherry.aiops@gmail.com).

## Public code

| Repository | What it is |
| --- | --- |
| [job-pipeline](https://github.com/sherrybuilds-studio/job-pipeline) | The job pipeline, driven by one profile file: career-board sourcing, liveness check, scoring, tailored CV and cover letter with an AI honesty reviewer, offline tests in CI |
| [reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent) | The restaurant reservation assistant with a fictional demo restaurant, offline tests and a retrieval eval in CI |

The documents in this repository are MIT licensed.
