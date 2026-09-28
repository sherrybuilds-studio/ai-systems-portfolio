# Eval Framework

## Principle

Every AI feature has an automated gate. A change that drops a product below its threshold does not
merge: the gate runs in CI and fails the build.

## Pattern

```text
1. Fix N known cases with the expected outcome (golden calls, gold questions, fixture leads)
2. Run the feature against every case, offline: no live model, no network, no secrets
3. Count the cases that behave as specified
4. Gate: pass rate at or above the threshold → exit 0, else exit 1 and the merge is blocked
```

Offline and deterministic matters: the same commit always gets the same score, so a drop is a real
regression, not model noise.

## Current gates (dated results in [`evals/`](../evals))

| Product | What the gate checks | Result | Threshold | File |
| --- | --- | --- | --- | --- |
| Voice receptionist | 12 golden call transcripts scored by a deterministic rubric: booking outcome, number grounding, AI disclosure present, recording-notice handling | 12 of 12 | 80% | [2026-09-02](../evals/2026-09-02-voice-receptionist-eval.json) |
| Restaurant assistant | 10 gold questions against a freshly rebuilt ChromaDB index with hybrid search | 10 of 10, average retrieval score 0.646 | 100% | [2026-09-02](../evals/2026-09-02-restaurant-bot-eval.json) |
| Sales OS, exposure scorer | 10 fixture leads (5 high exposure, 5 low) plus a check that medical and dental businesses are excluded | 10 of 10 | 80% | [2026-09-02](../evals/2026-09-02-sales-os-eval.json) |
| Sales OS, outreach | HMAC verification, inbound parsing, the 24-hour session window, template bodies, the stage machine and the UWG §7 consent gate, all mocked | 20 of 21 | 80% | [2026-09-04](../evals/2026-09-04-sales-os-phase-gates.json) |
| Sales OS, call copilot | ASR parsing, playbook retrieval with fake embeddings, suggestion caps, session recovery | 14 of 14 | 80% | [2026-09-04](../evals/2026-09-04-sales-os-phase-gates.json) |

The voice and restaurant gates run in the monorepo's CI on every push, together with the portfolio-chat
gate: 12 cases covering answers with citations, refusals of out-of-scope questions, and prompt-injection
blocking, plus a grep that fails on any retracted number.

## Beyond the product gates

- **Job pipeline:** every CV and cover letter passes an independent reviewer that must find zero
  unsupported claims before anything is sent, and an output lint.
- **Agent fleet:** agent work reaches main only through the lander, which requires every CI check to
  be green on the exact commit it merges.

## Why it matters

When the product model changes (for example Claude to GLM-5.2), the gates catch regressions before
they reach a customer. A model that scores below the threshold does not ship.
