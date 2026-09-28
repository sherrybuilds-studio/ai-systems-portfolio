# Sales OS

**A compliance-first lead pipeline for the AI phone receptionist.**

Sales OS finds owner-run local businesses on Google Places and ranks them by **missed-call exposure**: no
online booking, no listed opening hours, reviews saying nobody picks up, one owner running one location.
Outreach is prepared only for businesses that have given consent, every message waits for a human to
approve it, and a live call copilot suggests answers during sales calls. Every phase is gated by an
automated eval before any change merges.

> **Status (2026-09-28):** built and tested, run by hand. Gates: exposure scorer 10 of 10
> ([2026-09-02](../../evals/2026-09-02-sales-os-eval.json)); outreach 20 of 21 and copilot 14 of 14
> ([2026-09-04](../../evals/2026-09-04-sales-os-phase-gates.json)). The code is private.

## Was Sales OS macht (kurz auf Deutsch)

Sales OS findet inhabergeführte Betriebe auf Google Places und bewertet, wie viele Anrufe dort
voraussichtlich unbeantwortet bleiben. Nachrichten werden nur für Betriebe mit Einwilligung vorbereitet
(UWG §7), und keine Nachricht wird ohne Freigabe durch einen Menschen verschickt.

## What it does

1. **Discovery**: a niche search on Google Places (for example salons in one district), then details per
   business. Medical and dental businesses are excluded by design.
2. **Exposure scoring**: a pure function scores each business on the signals above and keeps the ones
   above a threshold as prospects.
3. **Consent-first outreach**: bilingual drafts (German and English) only for leads with recorded
   consent (UWG §7). Each draft goes to Telegram, where the operator sends it with `/send` or drops it
   with `/skip`.
4. **Follow-ups and re-engagement**: a CRM stage machine computes which follow-ups are due and drafts them
   the same way.
5. **Live call copilot**: streaming speech-to-text during a sales call, retrieval over a sales playbook,
   and at most two short suggestions at a time in a browser overlay; a bilingual summary lands in the CRM
   after the call.

## Tech stack

- **Python**: small single-purpose modules, wired by orchestrators
- **Google Places API (New)**: Text Search and Place Details for discovery (httpx with retry)
- **GLM-5.2 through OpenRouter**: drafts, call suggestions, summaries
- **Supabase (Postgres)**: CRM tables for leads, conversations and calls
- **Telegram Bot API**: pipeline digests and the human approval loop
- **Meta WhatsApp Cloud API**: templates, session messages, HMAC-verified webhooks, built from scratch
- **Deepgram Nova-3**: streaming speech-to-text over WebSocket, German and English in the same call
- **ChromaDB**: playbook index (sales playbook, objection handling, pricing FAQ)
- **FastAPI with server-sent events**: the live browser overlay for the copilot
- **Playwright**: the earlier website-quality scorer, still available behind a flag

## Architecture

```text
Phase 1 — Lead discovery
  "hair salons in <district>"
        │
        ▼
  Discovery ──── Google Places Text Search + Details
        │
        ▼
  Exposure scorer ─── no online booking · no listed hours · "nobody answers" reviews · owner-run
        │             (medical and dental excluded)
        ▼
  Orchestrator ─── discover → score → CRM upsert → Telegram digest   (--dry-run skips all writes)

Phase 2 — WhatsApp outreach (human in the loop)
  Consented leads → GLM-5.2 bilingual drafts → Telegram approval (/send or /skip)
  → Meta Cloud API → CRM stage advance + inbound inbox

Phase 3 — Live call copilot
  Call audio → Deepgram streaming ASR → crash-safe session journal
  → playbook retrieval + GLM-5.2 → at most 2 suggestions above 0.6 confidence
  → Telegram + browser overlay → bilingual summary after the call → CRM
```

See [architecture.md](architecture.md) for the stage-by-stage breakdown.

## Key engineering decisions

- **Consent enforced in code.** German UWG §7 requires prior express consent for electronic advertising.
  The drafter raises an exception for any lead without `consent=true` in the CRM. No cold-messaging tool
  exists in the codebase.
- **Nothing sends by itself.** Every outbound message is a draft that a human approves in Telegram; no
  code path reaches the send functions without that approval.
- **A deterministic scorer.** Exposure scoring is a pure function over the Places data, so every score is
  explainable and the gate runs offline on fixtures.
- **A silent copilot beats a crashing one.** The suggestion engine never raises: any failure returns an
  empty list. Suggestions are capped, throttled and semantically cached so a repeated objection does not
  buy a repeated model call mid-call.
- **Crash-safe call sessions.** Every session change is journaled to disk, so a crash mid-call loses
  nothing.
- **Graceful degradation.** Missing CRM credentials never crash the pipeline; results fall back to disk as
  the record.
- **The 24-hour window.** Free-form WhatsApp messages go out only inside Meta's 24-hour session window;
  outside it, the system uses a pre-approved template.

## Eval gates

| Gate | What it checks | Result |
| --- | --- | --- |
| Exposure scorer | 10 fixture leads, 5 high and 5 low exposure, plus the medical and dental exclusion | **10 of 10** |
| Phase 2, outreach | HMAC, session windows, the stage machine, the consent gate (mocked) | **20 of 21** |
| Phase 3, copilot | ASR parsing, playbook retrieval, suggestion caps, sessions (mocked) | **14 of 14** |

Threshold to merge: 80%. Details in [eval-results.md](eval-results.md).
