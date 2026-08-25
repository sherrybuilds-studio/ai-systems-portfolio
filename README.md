# SherryBuilds — AI Engineering Portfolio

Shehryar Irfan · CS student in Berlin · AI automation engineer

I build production AI systems that replace manual workflows — an AI phone
receptionist that answers real calls, a self-healing agent fleet, RAG bots,
lead pipelines. Everything is eval-gated at ≥80% before merge, and every
number below is dated and verifiable. No vibes — working systems with test results.

**Call the live receptionist: +1 650 479 7535** — German or English, books
appointments, tells you it's an AI in the first sentence (EU AI Act Art. 50)
and asks before recording (§201 StGB). 43 real calls logged so far.

## Projects

- **Voice Receptionist** — Vapi-based AI phone receptionist (DE/EN) with a FastAPI tool webhook for availability + booking, per-call compliance evidence (Art. 50 disclosure, §201 consent, sha256-chained), and a 12/12 golden-call outcome gate (2026-08-25). Live.
- **Agent Fleet** — 17 enabled agents on a Postgres-leased dispatcher with a self-healer (classify → policy → remediate, daily kill-switch) and a cost-truth ledger. 520 runs since 2026-07-09 at 0.8% hard failures; a 43% failed-or-stale backlog drained to zero on 2026-08-22. See [architecture/fleet-overview.md](architecture/fleet-overview.md).
- **Sales OS** — Lead discovery → missed-call exposure scoring → bilingual outreach drafts (human-approved) → live call copilot. First live run 2026-08-25. Eval gates: exposure 10/10, outreach 17/17, copilot 14/14.
- **Montari Oak** — Luxury furniture WhatsApp bot with hybrid RAG (keyword + semantic search) and a semantic cache. Lead scoring pipeline with 4-tier qualification.
- **Restaurant Bot** — Telegram RAG assistant for orders, reservations and menu questions. Retrieval eval 10/10 (2026-08-25); service currently offline.
- **CV Job Hunter** — Daily job-board pipeline: scrape → score → cover letters → Telegram digest. Last run 2026-08-20, parked 2026-08-23.
- **Mehboob Steel** — Steel trading WhatsApp bot serving 14,000+ contacts with Roman Urdu NLU.

## Architecture

- **Agent fleet** on a single VPS — 17 enabled agents (54 cataloged), orchestrated by a PostgreSQL-backed dispatcher with a self-healing loop
- **Model routing** — fleet agents run on Claude Haiku 4.5; product LLM calls (RAG bots, Sales OS) run on GLM-5.2 via OpenRouter; every call cost-traced in Langfuse
- **Eval-gated development** — every product has automated tests; CI blocks merges below 80%
- **Bilingual by design** — German + English for the Berlin market

## Tech Stack

Python · FastAPI · Vapi · ChromaDB · Supabase · PostgreSQL · Meta WhatsApp Cloud API ·
Playwright · OpenRouter · Langfuse · Docker · PM2 · httpx

## Links

- Portfolio: [sherrybuilds.com](https://sherrybuilds.com)
- Which product does what, mapped to real modules: [SOLUTIONS.md](SOLUTIONS.md)
- Public code: [reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent) · [job-pipeline](https://github.com/sherrybuilds-studio/job-pipeline) · [commerce-rag-agent](https://github.com/sherrybuilds-studio/commerce-rag-agent)
- Email: codewithsherry1@gmail.com

---

*The platform monorepo is private; architecture is open. Every metric here is dated and traceable — ask for the evidence.*
