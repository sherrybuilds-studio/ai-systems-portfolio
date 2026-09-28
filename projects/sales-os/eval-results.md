# Sales OS — Eval Results

Every phase ships with an automated eval that acts as a merge gate: a model swap or code change must
score **80% or more** on the relevant gate before it lands. Results below come from the dated files in
[`evals/`](../../evals).

| Gate | What it gates | Score | Date |
| --- | --- | --- | --- |
| Exposure scorer | Phase 1, missed-call exposure, offline | **10 of 10** | [2026-09-02](../../evals/2026-09-02-sales-os-eval.json) |
| Phase 2, outreach | WhatsApp outreach and compliance, mocked | **20 of 21** | [2026-09-04](../../evals/2026-09-04-sales-os-phase-gates.json) |
| Phase 3, copilot | Live call copilot, mocked | **14 of 14** | [2026-09-04](../../evals/2026-09-04-sales-os-phase-gates.json) |

## Gate 1 — Exposure scorer (10 of 10)

Ten fixture leads with ground-truth labels: five with high missed-call exposure (no online booking, no
listed hours, "nobody answers" reviews, owner-run) and five with low exposure. A separate check confirms
that medical and dental businesses are excluded before scoring. The scorer is a pure function, so the gate
needs no network and gives the same result on every run.

## Gate 2 — Phase 2 outreach (20 of 21)

Fully mocked (no network), covering the correctness and compliance surface of outreach:

- **HMAC webhook verification**: signature validation on inbound Meta webhooks.
- **Inbound and status message parsing**: replies and delivery receipts decode correctly.
- **24-hour session window**: free-form versus template routing flips at the window boundary.
- **Template body formatting**: pre-approved template payloads are built correctly.
- **CRM stage machine**: lead stages advance only along legal transitions.
- **Follow-up scheduling**: due follow-ups are computed correctly.
- **Consent gate (UWG §7)**: the drafter *must raise* for any lead without recorded consent. The
  compliance rule is asserted as a test, so it cannot silently regress.

One case did not pass in the 2026-09-04 run; at 95% the gate still clears its 80% bar.

## Gate 3 — Phase 3 copilot (14 of 14)

Fully mocked (ChromaDB on a temporary path with deterministic fake embeddings; no live speech-to-text),
covering the live-call loop end to end:

- **ASR message parsing and transcript assembly**: streaming events assemble into a coherent transcript.
- **Playbook indexing and retrieval**: the collections index and return the expected context.
- **Suggestion parsing, filtering and caps**: at most two suggestions, each at 0.6 confidence or more.
- **Session lifecycle and persistence fallback**: sessions journal to disk and recover after a
  simulated crash.
- **Delivery formatting**: suggestion payloads render correctly for each channel.

## Why evals are merge gates

The riskiest parts of the pipeline are the hardest to review by eye: a scorer that decides who gets
contacted, a legally constrained outreach path, and a real-time suggestion loop. Encoding each one's
contract as an executable eval means a model upgrade, prompt change or refactor either proves it still
holds the line, or it does not merge.
