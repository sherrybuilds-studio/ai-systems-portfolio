# AI Systems Portfolio

Shehryar Irfan · Berlin · [sherrybuilds.com](https://sherrybuilds.com) · [sherry.aiops@gmail.com](mailto:sherry.aiops@gmail.com)

I build LLM systems and run them on one Linux server: a voice receptionist, an agent fleet that heals its own failures, a job-search pipeline, and retrieval assistants.
This repo has the architecture notes and the dated eval files behind every number I publish.
The product code lives in a private monorepo. The public code repos are linked at the bottom.

---

## Systems

| System | State | Evidence |
|---|---|---|
| **AI phone receptionist**: Vapi voice agent (German and English), FastAPI tool webhook for availability and bookings, AI disclosure (EU AI Act Art. 50) and recording consent (§201 StGB) at the start of every call, hash-chained evidence record per call | Deployed. Demo on request | [12/12 golden calls, 2026-09-02](./evals/2026-09-02-voice-receptionist-eval.json) |
| **Agent fleet and self-healer**: Postgres-leased dispatcher that spawns Claude Code agents. 54 agents defined, 36 enabled. The self-healer sorts each failure into a failure class and then requeues, skips or escalates it | Running | [1,000 runs since 2026-07-09, 2.4% hard failures (snapshot 2026-09-24)](./evals/2026-09-24-fleet-stats.json) · [31 distinct agents run since 2026-07-09 (2026-09-26)](./evals/2026-09-26-fleet-agents.json) |
| **Job pipeline**: daily scrape and rule-based scoring, plus an on-demand sourcing pass over company career boards. For each role it builds a tailored one-page CV and a cover letter that an independent reviewer checks | Daily cron at 08:00 UTC | [Run counts and sourcing totals, 2026-09-26](./evals/2026-09-26-job-pipeline.json) |
| **Sales OS**: finds local businesses on Google Places, scores missed-call exposure, drafts outreach only when consent exists (UWG §7). Nothing sends without approval | Tested on one live run | [Scorer 10/10, 2026-09-02](./evals/2026-09-02-sales-os-eval.json) · [Live run: 20 places, 5 prospects, 2026-08-25](./evals/2026-08-25-phase1-live.json) · [Phase gates, 2026-09-04](./evals/2026-09-04-sales-os-phase-gates.json) |
| **WhatsApp product assistant** for a furniture brand: hybrid keyword and vector retrieval, semantic cache at 0.95 cosine | Pilot | Prompt cut 38% (1,118 to 695 tokens per message) after retrieval replaced the full catalogue in the prompt, per the [changelog, 2026-04-27](https://github.com/sherrybuilds-studio/commerce-rag-agent/blob/main/CHANGELOG.md) |
| **Restaurant reservation assistant**: bookings, waitlist, reminders, menu retrieval | Built, not deployed | [Retrieval 10/10, 2026-09-02](./evals/2026-09-02-restaurant-bot-eval.json) |
| **WhatsApp assistant for a steel trading business**: contact lookup and reminders in Roman Urdu | Built, waiting for the Meta connection | none yet |
| **This site**: [sherrybuilds.com](https://github.com/sherrybuilds-studio/sherrybuilds.com), Next.js, evidence section generated from these eval files | Live | First automated release 2026-09-02 |

Every linked number comes from a file in [`/evals`](./evals). Each file records the date and the command or query that produced it.

## Architecture

```
                 ┌──────────────── one Ubuntu VPS (Docker + PM2) ────────────────┐
 phone ─ Vapi ──▶│ voice-receptionist (FastAPI tool webhook) ─┐                  │
 WhatsApp ──────▶│ product / restaurant assistants ───────────┼─▶ ChromaDB       │
                 │                                            └─▶ Postgres / Supabase
                 │ job-hunter (PM2 cron 08:00 UTC) ──▶ Telegram digest           │
                 │                                                               │
                 │ dispatcher ──leases──▶ agent_tasks (Postgres) ◀── self-healer │
                 │      └─ spawns Claude Code agents (allowlisted tools)         │
                 └──── public traffic only through a Cloudflare tunnel ─────────┘
```

- Product LLM calls go through OpenRouter. Fleet agents run on Claude.
- The self-healer runs every 30 minutes. It also runs six no-LLM script probes every 5 minutes, eight silent-failure probes, and event triggers for the sentinel and deploy-monitor agents.
- More detail: [`architecture/fleet-overview.md`](./architecture/fleet-overview.md), [`architecture/rag-stack.md`](./architecture/rag-stack.md), [`architecture/eval-framework.md`](./architecture/eval-framework.md). Product-to-module map: [`SOLUTIONS.md`](./SOLUTIONS.md).

## How the evals work

- Each product has an offline, deterministic gate with a fixed set of golden cases. No live LLM calls are needed.
- The voice gate and the restaurant gate run in CI on every push to the monorepo.
- A fleet agent (`eval-runner`) re-runs the gates on a nightly schedule and flags regressions.
- The fleet numbers come from a SQL count over the task table. The method is stored inside each snapshot file.

## Using this repo

This repo holds documents only, with no runnable service. [`docker-compose.example.yml`](./docker-compose.example.yml) is a template for the shared stack (Postgres, embeddings service, gateway) with every value left as an env var placeholder.

## Limits

- The restaurant assistant is not deployed. Sales OS has had one live discovery run.
- The WhatsApp product assistant is a pilot. The 38% figure measures prompt size after the switch to retrieval, not a billing total.
- Not built yet: missed-call text-back, Google review replies, a multi-tenant voice setup.

## Public code

[reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent) · [commerce-rag-agent](https://github.com/sherrybuilds-studio/commerce-rag-agent) · [sherrybuilds.com](https://github.com/sherrybuilds-studio/sherrybuilds.com)

MIT licensed. Contact: [sherry.aiops@gmail.com](mailto:sherry.aiops@gmail.com) · [LinkedIn](https://www.linkedin.com/in/shehryar-irfan-bb5469349)
