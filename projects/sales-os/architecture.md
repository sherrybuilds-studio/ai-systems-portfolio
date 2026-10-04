# Sales OS — Architecture

Three phases, built in order of value: first a lead machine, then compliant outreach, then a live call copilot. Every stage is a small module with a single job; the orchestrators wire them together.

## Pipeline Diagram

```text
Phase 1 — Lead discovery   (python3 pipeline/run.py "<niche> in <district>" [--dry-run])
  "hair salons in <district>"
    │
    ▼
  Discovery module ──── Google Places API (New) Text Search + Details
    │                   httpx with retry
    ▼
  Normalizer ─── raw Places record → clean lead dict
    │
    ▼
  Exposure scorer ─── pure function: no online booking, no listed hours,
    │                 "nobody answers" reviews, owner-run single location;
    │                 medical and dental businesses excluded
    ▼
  Prospects ─── leads at or above the exposure threshold
    │
    ▼
  CRM layer ─── Supabase lead upsert + stage machine   (--dry-run skips it)
    │
    ▼
  Telegram digest ─── top prospects with their signals  (--dry-run skips it)

  Earlier product, still behind --product website:
  Playwright website-quality scorer (375px viewport, 0–100, defect codes)
  → GLM-5.2 report card and redesign mockup in German and English

Phase 2 — WhatsApp Outreach (human-in-the-loop)
  Outreach orchestrator (--draft)
    │  picks up: consented leads in 'previewed' stage
    │  + follow-ups due  + consented 'lost' leads (--reactivate)
    ▼
  Drafter ─── GLM-5.2 bilingual drafts (first / follow-up / reactivation)
    │         RAISES without consent=true (UWG §7 gate in code)
    ▼
  Telegram approval ─── /send <token> [de|en]  ·  /skip <token>
    │                   nothing auto-sends, ever
    ▼
  WhatsApp channel ─── Meta Cloud API
    │  session open?   → free-form text
    │  session closed? → pre-approved template
    ▼
  CRM layer ─── advance stage (previewed → contacted), log conversation
  Inbox ─── inbound replies: stage advance, CRM log, Telegram ping

Phase 3 — Live Call Copilot
  Call assist entrypoint (live audio)
    │
    ▼
  ASR client ─── Deepgram Nova-3 streaming WebSocket
    │            μ-law 16kHz mono, multilingual (DE/EN code-switching)
    ▼
  Session manager ─── journals every segment to a per-call disk file
    │                 (crash-safe: any mutation is recoverable)
    ▼
  Suggestion engine ─── RAG playbook + GLM-5.2 → ≤2 nudges, ≥0.6 confidence
    │                   semantically cached, throttled between passes
    ▼
  Delivery layer ─── Telegram + SSE browser overlay + WhatsApp thread
    │
    ▼ (on call end)
  Session manager ─── bilingual GLM-5.2 summary → CRM call record + note

  Async fallback (no live audio tap):
    voice-note file → Deepgram REST (or local Whisper)
      → transcript → summary → CRM note
```

## Stage Descriptions

### Phase 1 — Lead Discovery and Exposure Scoring

- **Discovery.** Queries the Google Places API (New) with a niche search ("hair salons in ..."), then fetches details per result. Built on httpx with retry logic.
- **Normalizer.** Flattens the verbose Places response into a compact lead dict the rest of the pipeline consumes.
- **Exposure scorer.** A pure function that scores each lead for missed-call exposure from the Places data: no online booking, no listed opening hours, reviews that say nobody picks up, and an owner-run single location. Medical and dental businesses are excluded. Because it needs no network, its gate runs offline on fixtures.
- **CRM layer.** Upserts leads into Supabase and drives the stage machine. Degrades gracefully: with no credentials configured it doesn't crash — local disk becomes the record.
- **Telegram digest.** Delivers the top prospects and the signals behind each score to the operator.
- **Earlier website funnel.** Behind `--product website`: Playwright scores each prospect's website at a 375px mobile viewport (0–100 with a defect code per broken signal, browser concurrency capped at 2), and GLM-5.2 writes a report card and a redesign mockup in German and English.

### Phase 2 — WhatsApp Outreach (Human-in-the-Loop)

- **Outreach orchestrator.** Collects work: consented leads that finished Phase 1, follow-ups that are due, and (optionally) consented lost leads for reactivation.
- **Drafter.** Generates bilingual first-contact, follow-up, and reactivation drafts with GLM-5.2. Its consent check is a hard gate — it raises an exception for any lead without recorded consent (German UWG §7 compliance enforced in code, not in policy).
- **Telegram approval loop.** Every draft lands in Telegram with a token; the operator replies `/send <token>` (optionally choosing the language) or `/skip <token>`. There is no auto-send code path.
- **WhatsApp channel.** A from-scratch Meta Cloud API client: free-form text inside the 24-hour session window, pre-approved template fallback outside it, HMAC-verified webhooks, and template management.
- **Inbox.** Inbound replies advance the lead's stage, get logged to the CRM, and ping the operator on Telegram. No auto-responder — a human always answers.

### Phase 3 — Live Call Copilot

- **ASR client.** Streams call audio to Deepgram Nova-3 over WebSocket in multilingual mode, handling German and English code-switching in real time.
- **Session manager.** Keeps call state in memory and journals every mutation to a per-call file on disk, so a crash mid-call loses nothing. Post-call, it produces a bilingual GLM-5.2 summary and writes the call record and a note to the CRM.
- **Suggestion engine.** Retrieves relevant playbook snippets (sales playbook, objection handling, pricing FAQ collections in ChromaDB) and asks GLM-5.2 for at most 2 nudges above a 0.6 confidence floor. Results are semantically cached and passes are throttled; on any failure it returns an empty list rather than raising — a silent copilot beats a crashing one.
- **Delivery layer.** A FastAPI app that fans suggestions out to Telegram, a live SSE browser overlay (dark UI showing transcript + nudges), and optionally the WhatsApp thread.
- **Voice-note fallback.** When no live audio tap exists, a recorded voice note goes through Deepgram's REST API (or local Whisper) to produce a transcript, summary, and CRM note asynchronously.

## CRM Stage Machine

```
new → previewed → contacted → replied → call_booked → proposal → won
                                                         ↓
                                                        lost
```
