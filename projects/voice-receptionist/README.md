# AI Phone Receptionist

A phone agent that answers a small business's calls in German or English, qualifies the caller, books through the business's own systems, and hands the owner a complete record of every call.
It says it is an AI in its first sentence, gives a recording notice when a call is recorded, and writes tamper-evident evidence of both for every call.
Deployed on Vapi with a demo tenant (a Berlin estate agency). **Demo on request** via [sherrybuilds.com](https://sherrybuilds.com).

> The source code is private: the product is available for licensing. This page describes the design and links the dated evidence. It contains no prompts, configuration or code.

---

## The problem

Owner-run businesses lose bookings to phones nobody answers. An AI receptionist fixes that only if a German business can legally use it: since 2 August 2026 the EU AI Act (Art. 50) requires an AI agent to disclose that it is a machine at the first interaction, and §201 StGB makes recording a call without notice a criminal offence. A receptionist that can't *prove* both, call by call, is a liability rather than a product.

## Architecture

```
 caller ──phone──▶ Vapi (telephony, speech-to-text, LLM, text-to-speech, barge-in)
                     │
                     │  tool calls during the call            end-of-call report
                     ▼                                        ▼
            ┌──────────────────── FastAPI webhook (shared-secret auth, fail-closed) ────────────────────┐
            │  tools: availability · booking · lead capture        │  compliance evidence record        │
            │  every tool call runs under a hard timeout           │  outcome-QA rubric (deterministic) │
            └──────────────┬───────────────────────────────────────┴───────────────┬───────────────────┘
                           ▼                                                       ▼
          business backend (Supabase: availability,           append-only evidence journal (hash chain)
          bookings, one lead row per call)                    + call record for daily QA grading
                           │
                           └──▶ Telegram: the owner gets each lead and any failure, immediately
```

**Failure discipline.** A dropped backend must not drop a call. If a tool is slow, the caller hears a short "one moment" line. If it fails, the caller gets a callback promise and never an error. The request is written to a local journal first, and the owner is alerted. Lead capture follows the same rule: the local record is written before the database, so an outage loses nothing, and a replay pushes the backlog later. Re-delivered call reports never create duplicate leads.

**One webhook, many businesses.** Each business is a tenant: its own assistant, number, knowledge and booking backend behind the same webhook. The earlier self-hosted build (telephony, speech pipeline, barge-in state machine) was replaced by Vapi in July 2026. The dialogue flows, triage rules and guardrails carried over.

## Compliance by design

| Requirement | How it is met | How it is proven |
|---|---|---|
| **EU AI Act Art. 50**: disclose being an AI at first interaction | The assistant's opening line states it is an AI assistant | Each call's evidence record checks the **first assistant utterance** for a disclosure and flags the call non-compliant if it is missing. A missing disclosure is never passed silently |
| **§201 StGB**: no recording without notice | When a call is recorded, the opening includes a recording notice | The record stores whether a recording exists and whether the notice was spoken. A recorded call without a notice is flagged as a §201 gap |
| **GDPR**: transcription is personal data | Only the fields the business needs are extracted per call | No secret, token or credential ever enters a record |
| **Tamper evidence** | Every record is chained to the previous one: it stores the previous record's SHA-256 and its own SHA-256 over the canonical JSON | Altering or deleting any past record breaks the chain from that point on. The journal is append-only |
| **"What did the caller actually hear?"** | Each record carries a fingerprint of the exact assistant configuration used for the call | A disputed call can be tied to the assistant configuration that was live at that moment |

## Quality gate

Green dashboards lie: a call can "complete" while the assistant invents a listing or skips the legal disclosure. So each call is graded on what it **did**:

- A **deterministic outcome rubric** checks every call on eight points: AI disclosure, recording notice where required, phone numbers read back digit by digit, details confirmed back to the caller, no invented facts, no leaked instructions, staying in its role, and no action the caller did not ask for. It uses no model and no network.
- **Golden calls**: a fixed set of transcripts, including ones with known defects, must be judged correctly.
- The gate runs in **CI on every push**, and a fleet agent grades the day's real calls on top of it.

| Result | Evidence |
|---|---|
| **12 of 12 golden calls judged correctly** (gate ≥ 80%), offline and deterministic | [`evals/2026-09-02-voice-receptionist-eval.json`](../../evals/2026-09-02-voice-receptionist-eval.json) — 2 Sep 2026 |

## Stack

Vapi · Deepgram (speech-to-text) · ElevenLabs (text-to-speech) · Claude · Python / FastAPI · Supabase (Postgres) · Telegram Bot API · GitHub Actions

## Status and limits

- **Live** on Vapi for one demo tenant. Demo calls on request, since the public demo number was taken off the site on 2026-09-10.
- Tenant templates exist for an estate agency, a dental practice, a salon and a restaurant. Only the estate-agency demo is live, and no paying business has been onboarded yet.
- Missed-call text-back and Google review replies are not built.

Contact: [sherry.aiops@gmail.com](mailto:sherry.aiops@gmail.com) · [sherrybuilds.com](https://sherrybuilds.com)
