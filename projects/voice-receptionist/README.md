# Frontkept: AI Phone Assistant

Frontkept is my AI phone assistant for Berlin estate agents and property managers. I founded it in October 2026, and it is new.
It answers in German, with English in testing, says in its first sentence that it is an AI, records no audio and keeps a text record of each call.
Status: built, run on demand. **Demo on request** via [sherrybuilds.com](https://sherrybuilds.com).

> The source code is private. This page describes the design and links the dated evidence. It contains no prompts, configuration or code.

---

## The problem

Owner-run offices lose calls that nobody answers. Frontkept picks up, says in its first sentence that it is an AI, records no audio and keeps a text record of each call.

## Architecture (as of 18 Nov 2026)

```text
 web caller ── signed web call ──▶ Clara (ElevenLabs Agents) ──▶ her tool server (FastAPI, mine)
 test line  ── SIP ──────────────▶ Paul  (OpenAI Realtime)   ──▶ his own server process (mine)

 one code base for both · no audio recorded · a text record of each call
```

**One code base, two processes.** Clara and Paul are built from one code base, and each runs on its own server process of mine. Clara runs on ElevenLabs Agents and takes signed web calls only; she has no phone number, and her tools run on a FastAPI tool server. Paul runs on OpenAI Realtime on a test line connected by SIP, in his own server process. Booking and passing a request on to an office are not in use yet.

**History.** The first version ran on a self-hosted speech pipeline, then on Vapi from July 2026. The current two voices replaced Vapi in autumn 2026, and the dialogue rules and guardrails carried over.

## What each call's record shows

- The assistant's opening words are checked for the AI disclosure, on the provider's transcript. The check is logged in a SHA-256 hash-chained journal (one chain per day), since 23 Aug 2026. Each record stores the previous record's SHA-256 and its own. There is no signature, so a rewritten file would not show.
- No audio is recorded; the server keeps a text record of the call.

## Quality gate

- Each new version passes a call simulator, the test suite and a separate review before I switch it on.
- A deterministic outcome rubric grades golden call transcripts, including ones with known defects, with no model and no network. It runs in CI.
- Earlier version, dated: on 2 Sep 2026 the rubric judged 12 of 12 golden calls correctly (gate at least 80%). That graded the earlier version of the assistant, on Vapi, before the move to ElevenLabs ([file](../../evals/2026-09-02-voice-receptionist-eval.json)).

## Stack

ElevenLabs Agents (Clara) · OpenAI Realtime (Paul) · SIP (Paul's test line) · Python · FastAPI · GitHub Actions

## Status and limits

- Built, run on demand; demo on request. Frontkept is new, and the line is not public.
- German first; English is in testing.
- Booking and passing a request on to an office are not in use yet. Missed-call text-back and review replies are not built.

Contact: [sherry.aiops@gmail.com](mailto:sherry.aiops@gmail.com) · [sherrybuilds.com](https://sherrybuilds.com)
