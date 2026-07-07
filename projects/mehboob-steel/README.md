# Mehboob Steel — WhatsApp Business Assistant

A WhatsApp-native assistant for a steel trading business in Pakistan, built on top of a **~14,000-contact** customer base. The owner runs his entire workflow through WhatsApp: he photographs business cards to auto-capture contacts, dictates reminders by voice, and receives a scheduled morning digest of what's due — all in Roman Urdu, the language he actually uses. The bot turns an unstructured pile of 16k+ Google Contacts rows and paper business cards into a clean, deduplicated CRM with a conversational front end.

## What It Does

- **Business card capture** — snap a photo of a card, vision models extract `{name, company, phone, role, ...}`, phones are normalized, deduped against the existing database, and inserted or merged.
- **Voice-dictated reminders** — the owner speaks a reminder in Urdu ("Ali ke liye pipe delivery, teen din baad yaad dilana"); NLU extracts the task, due date, and contact, then confirms in Roman Urdu.
- **Daily digest** — a scheduler runs every morning (Asia/Karachi) and delivers all pending reminders due that day, with sent-state tracking to prevent double delivery.
- **Contact database** — a CSV import pipeline profiled, cleaned, and deduplicated a raw Google Contacts export into a production contacts table.

## Tech Stack

| Layer | Choice |
|---|---|
| Messaging | Meta WhatsApp Cloud API (webhooks, media download) |
| Backend | Python (FastAPI-style webhook service) |
| LLM / Vision | Claude Haiku with Sonnet escalation, via OpenRouter |
| Transcription | Device-side Whisper keyboard (primary), Whisper API (fallback) |
| Database | Supabase (Postgres) — isolated project, RLS enabled |
| Rate limiting | slowapi (per-endpoint request throttling) |
| Observability | Langfuse tracing with PII redaction |
| Scheduling | Daily cron-style job (Asia/Karachi timezone) |

## Architecture

```
                          ┌──────────────────────────────┐
  Owner's WhatsApp ─────▶ │  Meta WhatsApp Cloud API      │
  (text / voice / image)  │  (HMAC-SHA256 signed webhook) │
                          └──────────────┬───────────────┘
                                         │
                                         ▼
                          ┌──────────────────────────────┐
                          │  Webhook Service              │
                          │  • signature verification     │
                          │  • sender allowlist           │
                          │  • rate limit (slowapi)       │
                          └──────────────┬───────────────┘
                                         │  route by message type
                 ┌───────────────────────┼───────────────────────┐
                 ▼                       ▼                       ▼
        ┌────────────────┐     ┌────────────────┐      ┌────────────────┐
        │  Text handler  │     │  Audio handler │      │  Card handler  │
        │  commands or   │     │  Whisper API   │      │  Vision:       │
        │  NLU reminder  │     │  → same NLU    │      │  Haiku→Sonnet  │
        │  extraction    │     │  extraction    │      │  escalation    │
        └───────┬────────┘     └───────┬────────┘      └───────┬────────┘
                │                      │                       │
                └──────────┬───────────┘                       ▼
                           ▼                        ┌────────────────────┐
                ┌────────────────────┐              │ Normalize phones   │
                │  reminders table   │              │ → dedup vs 14K     │
                │  pending / sent /  │              │ contacts → insert  │
                │  done              │              │ or merge           │
                └─────────┬──────────┘              └─────────┬──────────┘
                          │                                   │
                          ▼                                   ▼
                ┌────────────────────┐              ┌────────────────────┐
                │  08:00 scheduler   │              │  contacts table    │
                │  (Asia/Karachi)    │              │  (RLS on, service  │
                │  due today → send  │              │  role access only) │
                │  digest, mark sent │              └────────────────────┘
                └────────────────────┘
                          │
                          ▼
            Roman Urdu reply to owner via WhatsApp
```

## Key Engineering Decisions

### Consent-gated messaging at 14K-contact scale
The bot sits on a ~14,000-contact database, which makes Meta policy compliance the single biggest design constraint. Decisions that follow from it:

- **Strict sender allowlist.** Only the business owner's number can command the bot. Unauthorized senders get *silence* — no reply at all — so the bot never initiates or acknowledges contact with the wider database without an explicit, approved flow.
- **24-hour window discipline.** Free-form replies are only sent inside Meta's 24-hour customer-service window. Anything outside it (like the morning reminder digest) must go through a **Meta-approved utility template**; the scheduler was deliberately shipped in a "mark as sent, don't actually send" state until template approval lands, rather than risking policy violations by improvising delivery.
- **Rate limiting** on the webhook (100 req/min via slowapi) plus planned per-contact rate limits, so no bug or replay can turn the contact base into a spam vector.
- **Webhook authenticity** enforced via HMAC-SHA256 signature verification on every inbound event.

### Device-side transcription over API transcription
The primary voice flow was redesigned so the owner dictates via his phone's on-device Whisper keyboard, sending *text* to the bot. This removed a paid API dependency and a whole failure mode (audio download → upload → transcribe) from the hot path. Server-side Whisper API transcription remains as a fallback for raw audio files.

### Bilingual by design — Roman Urdu replies
All bot responses are in Roman Urdu (Urdu written in Latin script), matching how the owner actually types and reads on WhatsApp: "Naya contact save", "Yaad rahega: …", "Kaun sa?". Input handling is bilingual — commands and dictated reminders arrive as Urdu/English mixed text and are parsed by NLU rather than rigid command syntax.

### Cost-tiered vision extraction
Business card extraction runs Claude Haiku first and escalates to Sonnet only when the cheap model's output fails validation — keeping per-card cost minimal while preserving accuracy on messy, real-world cards.

### Ask, don't guess
When a reminder references a contact name that matches multiple people in the database, the bot asks "Kaun sa?" (which one?) instead of auto-picking. With 14K contacts, a wrong silent match means a reminder attached to the wrong customer — clarification was chosen over convenience.

### Privacy posture
- Phone numbers are masked in all logs (`+92••••••1234` style).
- LLM observability traces redact names and transcripts by default (`TRACE_PII=false`), and card image bytes are never sent to the tracing backend.
- The contacts export CSV and all secrets are excluded from version control; the database uses RLS with server-only service-role access.

### Data import pipeline
A profiling script analyzed the raw 16,426-row Google Contacts export before any writes: 95.3% usable phone numbers, 974 duplicates collapsed on normalized phone as the unique key, yielding 14,680 clean unique contacts. Profiling-before-importing meant zero junk rows landed in production.

## Status

Foundation complete: card extraction, text/voice reminders, contact import, scheduler, and security hardening are shipped. Next: Meta Cloud API production connection, approved utility template for out-of-window digest delivery, and a multi-match contact picker.
