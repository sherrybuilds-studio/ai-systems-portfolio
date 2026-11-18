# Frontkept: AI Phone Assistant

Frontkept is my AI phone assistant for Berlin estate agents and property managers. I founded it in October 2026, and it is new.
It answers in German, with English in testing, says in its first sentence that it is an AI, notes each caller's request, and records no audio.
Status: pilot. **Demo on request** via [sherrybuilds.com](https://sherrybuilds.com).

> The source code is private. This page describes the design and links the dated evidence. It contains no prompts, configuration or code.

---

## The problem

Owner-run offices lose calls that nobody answers. An AI assistant on the phone has to say at the first interaction that it is a machine (EU AI Act Art. 50, from 2 August 2026), and recording a call without notice is a criminal offence in Germany (§201 StGB). Frontkept says that it is an AI in its first sentence and records no audio.

## Architecture (as of 18 Nov 2026)

```text
 web caller ── signed web call ──▶ Clara (ElevenLabs Agents) ──┐
 test line  ── SIP ──────────────▶ Paul  (OpenAI Realtime)  ───┤  tool calls during the call
                                                               ▼
                       ┌───────────── FastAPI tool server (one, my own) ─────────────┐
                       │  the tools the assistant calls, each under a hard timeout    │
                       │  a text record of each call (no audio)                       │
                       │  the AI-disclosure check, logged in a daily SHA-256 chain    │
                       └──────────────────────────────────────────────────────────────┘
```

**Two voices, one server.** Clara runs on ElevenLabs Agents and takes signed web calls only; she has no phone number. Paul runs on OpenAI Realtime on a test line connected by SIP. Both call the same FastAPI tool server. Booking and passing a request on to an office are not in use yet.

**History.** The first version ran on a self-hosted speech pipeline, then on Vapi from July 2026. The current two voices replaced Vapi in autumn 2026, and the dialogue rules and guardrails carried over.

## How the rules are met

| Rule | How it is met | What the record shows |
|---|---|---|
| **EU AI Act Art. 50**: say it is an AI at the first interaction | The opening line says it is an AI assistant | Each call's record checks the assistant's opening words, on the provider's transcript, for the disclosure, and flags the call if it is missing |
| **§201 StGB**: no recording without notice | No audio is recorded | The server keeps a text record of the call, no audio |
| **Hash chain** | Each record stores the previous record's SHA-256 and its own; the chain restarts every UTC day | A changed record breaks that day's chain. There is no signature, so a rewritten file would not show |

Each call's AI-disclosure check is logged in this SHA-256 hash-chained journal (one chain per day), since 23 Aug 2026.

## Quality gate

- Each new version passes a call simulator, the test suite and a separate review before I switch it on.
- A deterministic outcome rubric grades golden call transcripts, including ones with known defects, with no model and no network. It runs in CI.
- Earlier version, dated: on 2 Sep 2026 the rubric judged 12 of 12 golden calls correctly (gate at least 80%). That graded the earlier version of the assistant, on Vapi, before the move to ElevenLabs ([file](../../evals/2026-09-02-voice-receptionist-eval.json)).

## Stack

ElevenLabs Agents (Clara) · OpenAI Realtime (Paul) · SIP (Paul's test line) · Python · FastAPI · GitHub Actions

## Status and limits

- Pilot, demo on request. The line is not public.
- German first; English is in testing.
- Booking and passing a request on to an office are not in use yet. Missed-call text-back and review replies are not built.

Contact: [sherry.aiops@gmail.com](mailto:sherry.aiops@gmail.com) · [sherrybuilds.com](https://sherrybuilds.com)
