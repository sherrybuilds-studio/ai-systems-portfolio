# Sales OS

**An end-to-end, compliance-first sales pipeline for a done-for-you website service.**

Sales OS discovers owner-run local businesses via the Google Places API and ranks them by **missed-call exposure** — no online booking, no listed hours, reviews saying nobody picks up, owner-operated — to sell the AI phone receptionist (retargeted 2026-08-25; the original website-quality funnel with Playwright scoring and GLM-5.2 bilingual report cards is still available behind a flag). Later phases add human-approved WhatsApp outreach and a live call copilot with real-time ASR and RAG-backed suggestions — all gated by automated evals that must pass before any merge.

## Tech Stack

- **Python** — ~3,700 lines across 30 modules, plus one HTML overlay
- **Google Places API (New)** — Text Search + Details for lead discovery (httpx with retry, ~$0.07/lead)
- **Playwright** — headless website quality scoring at a 375px mobile viewport
- **GLM-5.2** — all LLM calls: preview cards, outreach drafts, call suggestions, summaries
- **Supabase (Postgres)** — CRM tables for leads, conversations, and calls
- **Telegram Bot API** — pipeline digests and the human approval loop
- **Meta WhatsApp Cloud API** — templates, session messages, HMAC-verified webhooks (built from scratch)
- **Deepgram Nova-3** — streaming ASR over WebSocket (multilingual DE/EN code-switching)
- **ChromaDB** — RAG playbook index (sales playbook, objection handling, pricing FAQ)
- **FastAPI + SSE** — live browser overlay for the call copilot

## Architecture Flow

```
Phase 1 — Lead Machine
  "restaurants in <city district>"
        │
        ▼
  Discovery ──── Google Places Text Search + Details
        │
        ▼
  Normalizer ─── Places record → lead dict
        │
        ▼
  Quality Scorer ─── Playwright, mobile viewport, 0-100 score + defect codes
        │
        ▼
  Preview Generator ─── GLM-5.2 → report card + redesign mockup (DE + EN)
        │
        ▼
  Orchestrator ─── discover → score → preview → CRM upsert → Telegram digest
                   (worst website first; --dry-run skips all writes)

Phase 2 — WhatsApp Outreach (human-in-the-loop)
  Consented leads → GLM-5.2 bilingual drafts → Telegram approval
  (/send or /skip) → Meta Cloud API → CRM stage advance + inbound inbox

Phase 3 — Live Call Copilot
  Call audio → Deepgram streaming ASR → crash-safe session journal
  → RAG playbook + GLM-5.2 → ≤2 high-confidence nudges
  → Telegram + SSE browser overlay → post-call bilingual summary → CRM
```

See [architecture.md](architecture.md) for the full stage-by-stage breakdown.

## Key Engineering Decisions

- **Eval gates before merge.** Three automated eval suites (website scorer, outreach, copilot) must pass at ≥80% before any model or code change merges. All three sat at 100% when measured (2026-07-07); the new offline exposure-scorer gate is 10/10 (2026-08-25). First live run 2026-08-25: 20 Berlin salons → 5 prospects for ~$0.83 in Places calls, 0 cold messages drafted (UWG §7 — consent first). See [eval-results.md](eval-results.md).
- **Consent-gated outreach, enforced in code.** German UWG §7 requires prior express consent for unsolicited B2B electronic advertising. The drafter *raises an exception* if a lead lacks `consent=true` in the CRM — compliance is a code path, not a policy doc. No cold-blast tooling exists in the codebase by design.
- **Nothing auto-sends, ever.** Every outbound WhatsApp message is a draft delivered to Telegram for explicit human approval via a `/send <token>` command. No code path can reach the send functions without prior approval.
- **Mobile-first scoring model.** Websites are scored 0–100 at a 375px viewport with explicit defect codes per broken signal (HTTP-only, missing viewport meta, etc.) — because that's how the prospect's customers actually see their site.
- **Bounded concurrency.** Playwright browser contexts are capped at 2 to keep the pipeline from degrading co-hosted services on a memory-constrained box.
- **Silent copilot beats a crashing one.** The live-call suggestion engine never raises — any failure returns an empty suggestion list. Suggestions are hard-capped (max 2, ≥0.6 confidence), throttled, and semantically cached so repeat objections don't buy repeat LLM calls mid-call.
- **Crash-safe call sessions.** Every session mutation is journaled to disk, so a mid-call crash can recover the full transcript and state.
- **Graceful degradation.** Missing CRM credentials never crash the pipeline — results are persisted to disk as the fallback record of truth.
- **24h session window tracking.** WhatsApp free-form messages are only sent inside Meta's 24-hour window; outside it, the system falls back to a pre-approved template.

## Eval Results

**100% on all three gates** (threshold to merge: ≥80%):

| Gate | Scope | Result |
|------|-------|--------|
| Website scorer | 10 known sites, healthy vs. broken classification | **10/10** |
| Phase 2 — outreach | HMAC, session windows, stage machine, consent gate | **17/17** |
| Phase 3 — copilot | ASR parsing, RAG retrieval, suggestion caps, sessions | **14/14** |

Full breakdown in [eval-results.md](eval-results.md).
