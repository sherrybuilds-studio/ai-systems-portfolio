# Solutions → Modules

The productized systems I sell, mapped to the code that actually implements
them. Where a module lives in the private platform monorepo it says so;
where public code exists it is linked. Nothing here is aspirational: if a
system is not built yet, the row says **not yet built** and names the
nearest existing code.

| # | Solution | What it does for the business | Implemented by | Status (2026-08-25) |
|---|---|---|---|---|
| 1 | **AI phone receptionist** | Answers every call in German or English, checks availability, books the appointment, discloses that it's an AI (EU AI Act Art. 50), asks consent before recording (§201 StGB), writes tamper-evident evidence per call | `apps/voice-receptionist/vapi/server.py` (tool webhook: `check_availability`, `book_reservation`), `vapi/compliance.py`, `qa/rubric.py` + `tests/eval_phase7.py` (12 golden calls) | **Live** — 43 real calls, 12/12 gate |
| 2 | **Lead discovery & scoring** | Finds owner-run businesses on Google Places and ranks them by missed-call exposure (no online booking, no listed hours, reviews saying nobody answers) | `apps/sales-os/discovery/places.py`, `discovery/exposure.py`, `tests/eval_exposure.py` | Live — first run 2026-08-25, 20 salons → 5 prospects |
| 3 | **Outreach (human-approved)** | Bilingual first-touch drafts; every message goes to Telegram for `/send` or `/skip`; consent gate (UWG §7) in code, nothing auto-sends | `apps/sales-os/outreach/drafter.py`, `pipeline/run_outreach.py`, `channels/whatsapp.py` | Built; awaiting Meta business verification for sends |
| 4 | **Follow-ups** | Stage-aware follow-up drafts (contacted → replied → call booked → proposal), due-list from the CRM | `apps/sales-os/pipeline/crm.py` (`follow_ups_due`, stage machine), `outreach/drafter.py` (`draft_followup`) | Built; eval 17/17 (mocked) |
| 5 | **Re-engagement** | Wakes cold or lost leads with a fresh, consent-safe hook | `apps/sales-os/outreach/drafter.py` (`draft_reactivation`), `pipeline/crm.py` | Built |
| 6 | **CRM** | Supabase-backed lead store: stages, conversations log, consent flag, offline journal fallback | `apps/sales-os/pipeline/crm.py`, `channels/inbox.py`, `dashboard/app.py` (local cockpit) | Built; `sales_leads` migration pending in the current project |
| 7 | **Reminders** | Voice-dictated reminders in Urdu, NLU-extracted task/date/contact, morning digest with sent-state tracking | Mehboob Steel client build (`agents/` — private), plus `apps/restaurant-bot/reservations/reminders.py` for booking reminders | Built (client); Meta template approval pending for out-of-window sends |
| 8 | **Campaigns / broadcasts** | Segmented WhatsApp broadcasts to an opted-in customer list | `apps/restaurant-bot/automations/broadcast.py` (public: [reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent)) | Built |
| 9 | **Missed-call text-back** | Text a caller back automatically when a call goes unanswered | **Not yet built.** Nearest code: the voice receptionist (which removes the missed call instead) and `channels/whatsapp.py` session texts | Planned |
| 10 | **Review automation** | Monitor and respond to Google reviews | **Not yet built.** Reading reviews exists (`discovery/places.py` pulls them; `exposure.py` scans for "nobody answers"); *replying* needs the Google Business Profile API with owner OAuth | Planned |

## Shared platform underneath all of them

- `packages/sherry-core` — config (`CoreSettings`), one validated LLM client with retries, semantic cache, Telegram client with backoff, graph engine.
- Agent fleet — Postgres-leased dispatcher, 17 enabled agents, self-healer, cost-truth ledger (`services/dispatcher`, `services/self-healer`).
- Eval gates in CI — every product has an offline gate; a change that drops a gate below its threshold does not merge (`architecture/eval-framework.md`).
