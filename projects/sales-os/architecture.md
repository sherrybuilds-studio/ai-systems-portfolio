# Sales OS — Architecture

Three phases, built in order of value: first a lead machine, then compliant outreach, then a live call copilot. Every stage is a small module with a single job; the orchestrators wire them together.

## Pipeline Diagram

```
Phase 1 — Sales Weapon
  "restaurants in <city district>"
    │
    ▼
  Discovery module ──── Google Places API (New) Text Search + Details
    │                   httpx with retry, ~$0.07 per lead
    ▼
  Normalizer ─── raw Places record → clean lead dict
    │
    ▼
  Quality scorer ─── Playwright mobile viewport (375px), 0-100 score
    │                concurrency capped at 2, defect codes per broken signal
    ▼
  Preview generator ─── GLM-5.2 → report card + redesign mockup HTML
    │                   in BOTH German and English (4 files per prospect)
    ▼
  Pipeline orchestrator ─── discover → score → preview → CRM → Telegram
    │                       --dry-run flag skips CRM writes + Telegram
    ▼
  CRM layer ─── Supabase lead upsert + stage machine
    │
    ▼
  Telegram digest ─── top prospects, worst website first

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

### Phase 1 — Lead Discovery and Website Preview

- **Discovery.** Queries the Google Places API (New) with a natural-language niche search ("restaurants in ..."), then fetches details per result. Built on httpx with retry logic; cost works out to roughly $0.07 per lead.
- **Normalizer.** Flattens the verbose Places response into a compact lead dict the rest of the pipeline consumes.
- **Quality scorer.** Loads each prospect's website in Playwright at a 375px mobile viewport and produces a 0–100 score, with an explicit defect code for every broken signal (HTTP-only, missing viewport meta, and similar). Browser concurrency is capped at 2 to protect co-hosted services on constrained hardware.
- **Preview generator.** For low-scoring sites, GLM-5.2 produces a report card plus a redesign mockup — in both German and English, four HTML files per prospect.
- **Orchestrator.** Runs the full chain (discover → score → preview → CRM upsert → Telegram digest) and supports a `--dry-run` mode that skips all external writes.
- **CRM layer.** Upserts leads into Supabase and drives the stage machine. Degrades gracefully: with no credentials configured it doesn't crash — local disk becomes the record.
- **Telegram digest.** Delivers the top prospects to the operator, sorted worst-website-first, since the worst site is the strongest pitch.

### Phase 2 — WhatsApp Outreach (Human-in-the-Loop)

- **Outreach orchestrator.** Collects work: consented leads that finished Phase 1, follow-ups that are due, and (optionally) consented lost leads for reactivation.
- **Drafter.** Generates bilingual first-contact, follow-up, and reactivation drafts with GLM-5.2. Its consent check is a hard gate — it raises an exception for any lead without recorded consent (German UWG §7 compliance enforced in code, not in policy).
- **Telegram approval loop.** Every draft lands in Telegram with a token; the operator replies `/send <token>` (optionally choosing the language) or `/skip <token>`. There is no auto-send code path.
- **WhatsApp channel.** A from-scratch Meta Cloud API client: free-form text inside the 24-hour session window, pre-approved template fallback outside it, HMAC-verified webhooks, and template management.
- **Inbox.** Inbound replies advance the lead's stage, get logged to the CRM, and ping the operator on Telegram. No auto-responder — a human always answers.

### Phase 3 — Live Call Copilot

- **ASR client.** Streams call audio to Deepgram Nova-3 over WebSocket in multilingual mode, handling German/English code-switching in real time.
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
