# SherryBuilds — AI Systems Portfolio

**Shehryar Irfan** · AI Engineer · CS student, Arden University Berlin
Production AI systems, built and operated solo on one VPS — with the eval gates, compliance evidence, and dated test results to prove they work.

📞 **Call the live receptionist: +1 650 479 7535** — English or German. It tells you it's an AI in the first sentence (EU AI Act Art. 50), asks before recording (§201 StGB), captures your inquiry, and can book a viewing — every call leaves hash-chained compliance evidence and lands in the database at hang-up.

---

## Systems

| System | Status | Dated proof |
|---|---|---|
| **AI Phone Receptionist** — bilingual (DE/EN) Vapi agent, FastAPI tool webhook, viewing bookings, per-call Art. 50 + §201 evidence | 🟢 Live — call it | [12/12 golden-call outcome eval · 2026-09-02](./evals/2026-09-02-voice-receptionist-eval.json) |
| **Self-Healing Agent Fleet** — Postgres-leased dispatcher, 11 enabled agents, classify→policy→remediate self-healer, cost-truth ledger | 🟢 Live | [520 runs since Jul 9 · 0.8% hard failures · 471-task backlog cleared to zero](./evals/2026-08-25-fleet-stats.json) |
| **Job Pipeline** — scrape 10 sources → score → tailored CV + cover letter per match → Telegram digest | 🟢 Runs daily | 220–290 postings/run · fully automated |
| **This portfolio site** — Next.js 16, approval-gated CI/CD to GHCR, fail-closed auth on private routes, evidence strip generated from real eval JSONs, grounded chat widget | 🟢 Live | First automated release 2026-09-02 · [source](https://github.com/sherrybuilds-studio/sherrybuilds.com) |
| **Sales OS** — lead discovery → missed-call scoring → consent-gated bilingual outreach (UWG §7) → call copilot | 🟡 Phase 1 live-tested | [20 places → 5 prospects, 2026-08-25](./evals/2026-08-25-phase1-live.json) · [gates 10/10 · 20/21 · 14/14](./evals/2026-09-04-sales-os-phase-gates.json) |
| **Commerce RAG Agent** — WhatsApp sales assistant, hybrid keyword+semantic retrieval, semantic cache (95% cosine) | 🟡 Pilot | **38% token cost cut** (1,118 → 695/msg), measured in production |
| **Restaurant Bot** — Telegram reservations + menu RAG | ⚪ Built, offline | [Retrieval eval 10/10 · 2026-09-02](./evals/2026-09-02-restaurant-bot-eval.json) |
| **Mehboob Steel** — WhatsApp assistant for a steel trader, Roman Urdu NLU, 14k-contact lookup | ⚪ Built, pending Meta connection | — |

Every green row answers traffic today. Every linked metric is a dated eval file in [`/evals`](./evals) — nothing here is estimated.

## Engineering approach

- **Eval-first.** Each product ships with an offline gate (voice 12/12, restaurant 10/10, sales 10/10); CI blocks any merge that drops one. Gates re-run nightly by the fleet's eval-runner agent.
- **Compliance in code, not slides.** AI disclosure (Art. 50), recording consent (§201 StGB), and outreach consent (UWG §7) are enforced in the codebase and leave per-call evidence.
- **Cost is a feature.** A cost-truth ledger caps fleet spend daily from real token usage; the 38% RAG optimization was measured, not guessed.
- **Incidents are documented.** A fleet failure wave was root-caused to a model-routing mismatch and a 471-task backlog cleared to zero — today's hard-failure rate is 0.8% over 520 runs; three secret-leak paths were closed at the source. The debugging stories are part of the portfolio.

## Patterns ready to deploy

The systems above share one module library — RAG retrieval, semantic caching, eval gates, voice/WhatsApp/Telegram channels, consent-gated outreach, approval loops, compliance evidence, self-healing ops. From it, these variants are **days of assembly, not months of building**:

- **Google review responder** — new reviews in, drafted replies out, human-approved before posting; sentiment routing for complaints. *(Reuses: RAG over past replies, the Sales OS approval loop, the restaurant bot's review_log schema.)*
- **Missed-call rescue** — unanswered business calls get an instant AI callback; the caller's inquiry lands in the same lead pipeline as the receptionist. *(Reuses: voice stack, lead capture, Telegram alerting.)*
- **Appointment reminder & no-show prevention** — confirmations and day-before reminders over the caller's channel. *(Reuses: reservation flow, appointments schema, scheduled jobs.)*
- **Inbound email triage** — classify, draft, escalate; nothing sends without approval. *(Reuses: the job pipeline's scoring loop, the outreach drafter's approval gate.)*
- **FAQ / order-status WhatsApp bot** — catalogue or policy RAG with the measured 38%-cheaper retrieval pattern. *(Reuses: commerce agent end to end.)*
- **Lead-list builder** — discover businesses by niche + area, score them, deliver a ranked sheet. *(Reuses: Sales OS Phase 1, live-tested.)*

## Designed, on the roadmap

Specced on top of the same modules — deeper builds, engineered rather than assembled:

- **"Clara calls you" demo widget** — visitor requests a call, gets phoned by the receptionist within seconds. The design covers OTP consent, per-number/per-IP rate limits, calling-hours windows, cost caps and a kill switch — because an outbound widget without them is a harassment tool.
- **Multi-tenant voice** — one receptionist codebase, per-tenant prompts, numbers, and data isolation; the tenant config layer already exists in the voice app.
- **Live call copilot** — real-time ASR → playbook RAG → nudges to a human agent mid-call; Phase 3 of Sales OS, built and gated offline (14/14), awaiting a live pilot.
- **Call QA & coaching loop** — every production call graded on outcome, protocol and truthfulness by a QA agent (running today on the fleet), feeding weekly coaching summaries.
- **Compliance evidence API** — the hash-chained Art. 50/§201 journal, exposed so any business can prove what its AI said and when.

## Architecture

One VPS (Ubuntu, Docker + PM2) runs everything: Caddy gateway → FastAPI services → ChromaDB retrieval → Supabase/Postgres persistence. Fleet agents run on Claude (Haiku); product LLM calls route through OpenRouter. Full picture: [`architecture/fleet-overview.md`](./architecture/fleet-overview.md) · product-to-module map: [`SOLUTIONS.md`](./SOLUTIONS.md)

**Stack:** Python · FastAPI · Claude API · Vapi · Deepgram · ElevenLabs · ChromaDB · PostgreSQL · Supabase · Redis · Docker · PM2 · Caddy · Meta WhatsApp Cloud API · n8n · Next.js

## Links

🌐 [sherrybuilds.com](https://sherrybuilds.com) — live demos and evidence
💻 Public code: [reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent) · [job-pipeline](https://github.com/sherrybuilds-studio/job-pipeline) · [commerce-rag-agent](https://github.com/sherrybuilds-studio/commerce-rag-agent)
💼 [LinkedIn](https://www.linkedin.com/in/shehryar-irfan-bb5469349) · ✉️ [shehryarmughal30@gmail.com](mailto:shehryarmughal30@gmail.com)

*Open to Werkstudent roles in Berlin — available immediately.*
