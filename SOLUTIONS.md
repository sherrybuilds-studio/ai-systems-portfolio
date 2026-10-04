# Solutions → Modules

The systems I offer, mapped to the code that implements them. Where a module lives in the private
monorepo it says so; where public code exists it is linked. Nothing here is aspirational: if a
system is not built yet, the row says **not yet built** and names the nearest existing code.

| # | Solution | What it does for the business | Implemented by | Status (2026-09-28) |
| --- | --- | --- | --- | --- |
| 1 | **[AI phone receptionist](./projects/voice-receptionist/)** | Answers every call in German or English, checks availability, books the appointment, says it is an AI (EU AI Act Art. 50), gives the recording notice (§201 StGB), and writes tamper-evident evidence per call | `apps/voice-receptionist/vapi/server.py` (tool webhook: `check_availability`, `book_reservation`), `vapi/compliance.py`, `qa/rubric.py` + `tests/eval_phase7.py` (12 golden calls) | Deployed for a demo tenant; gate 12 of 12 (2026-09-02) |
| 2 | **Lead discovery and scoring** | Finds owner-run businesses on Google Places and ranks them by missed-call exposure: no online booking, no listed hours, reviews saying nobody answers | `apps/sales-os/discovery/places.py`, `discovery/exposure.py`, `tests/eval_exposure.py` | Built, run by hand; gate 10 of 10 (2026-09-02) |
| 3 | **Outreach, human-approved** | Bilingual first-touch drafts; every message goes to Telegram for `/send` or `/skip`; the consent gate (UWG §7) is code, and nothing sends by itself | `apps/sales-os/outreach/drafter.py`, `pipeline/run_outreach.py`, `channels/whatsapp.py` | Built; sending waits for Meta business verification |
| 4 | **Follow-ups** | Stage-aware follow-up drafts (contacted → replied → call booked → proposal) and a due list from the CRM | `apps/sales-os/pipeline/crm.py` (`follow_ups_due`, stage machine), `outreach/drafter.py` (`draft_followup`) | Built; covered by the phase-2 gate, 20 of 21 (2026-09-04, mocked) |
| 5 | **Re-engagement** | Wakes cold or lost leads with a fresh, consent-safe message | `apps/sales-os/outreach/drafter.py` (`draft_reactivation`), `pipeline/crm.py` | Built |
| 6 | **CRM** | Supabase lead store: stages, conversation log, consent flag, offline journal fallback | `apps/sales-os/pipeline/crm.py`, `channels/inbox.py`, `dashboard/app.py` (local cockpit) | Built; `sales_leads` migration pending in the current project |
| 7 | **Reminders** | Booking reminders with confirm and cancel replies, plus a waitlist that fills freed slots | `apps/restaurant-bot/reservations/reminders.py` (public: [reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent)) | Built; Meta template approval pending for out-of-window sends |
| 8 | **Campaigns and broadcasts** | Segmented WhatsApp broadcasts to an opted-in customer list | `apps/restaurant-bot/automations/broadcast.py` (public: [reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent)) | Built |
| 9 | **Missed-call text-back** | Texts a caller back automatically when a call goes unanswered | **Not yet built.** Nearest code: the voice receptionist (which answers the call instead) and the session texts in `channels/whatsapp.py` | Planned |
| 10 | **Review automation** | Monitors Google reviews and drafts replies | **Not yet built** as a product. Reading reviews exists (`discovery/places.py` pulls them, `exposure.py` scans them), and reservation-agent drafts replies for approval; posting needs the Google Business Profile API with owner OAuth | Planned |
| 11 | **Quote agent** | Turns an inquiry into a priced quote from the business's own price list, asks for missing details, sends after the owner approves | **Not yet built.** Nearest code: catalogue retrieval in [commerce-rag-agent](https://github.com/sherrybuilds-studio/commerce-rag-agent), the Telegram approval loop in `apps/sales-os/pipeline/run_outreach.py`, PDF output in [job-pipeline](https://github.com/sherrybuilds-studio/job-pipeline) | Planned |
| 12 | **Speed-to-lead** | Replies to a new inbound lead within a minute, qualifies it and books a call | **Not yet built.** Nearest code: the booking tools in `apps/voice-receptionist/vapi/server.py` and the WhatsApp session handling in `channels/whatsapp.py` | Planned |

## Shared platform underneath all of them

- `packages/sherry-core`: config (`CoreSettings`), one validated LLM client with retries, semantic cache, Telegram client with backoff, graph engine.
- Agent fleet: Postgres-leased dispatcher with 54 agents in six teams, self-healer, a lander that merges test-only and docs-only agent work after CI passes, and a cost ledger (`services/dispatcher`, `services/self-healer`). See the [fleet overview](./architecture/fleet-overview.md).
- Eval gates in CI: every product has an offline gate, and a change that drops a gate below its threshold does not merge (see the [eval framework](./architecture/eval-framework.md)).
