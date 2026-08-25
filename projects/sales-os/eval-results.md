# Sales OS — Eval Results

Every phase ships with an automated eval suite that acts as a merge gate: any model swap or code change must score **≥80%** on the relevant suite before it lands. The three original suites passed at **100%** when measured (2026-07-07); the exposure-scorer gate added with the voice retarget is **10/10** (2026-08-25, offline, deterministic).

| Suite | What it gates | Score |
|-------|---------------|-------|
| Exposure scorer eval (2026-08-25) | Phase 1 — missed-call exposure, offline | **10/10 (100%)** |
| Website scorer eval (2026-07-07) | Phase 1 legacy — website quality | **10/10 (100%)** |
| Phase 2 eval | WhatsApp outreach + compliance | **17/17 (100%)** |
| Phase 3 eval | Live call copilot | **14/14 (100%)** |

## Gate 1 — Website Scorer Eval (10/10)

Classification eval against 10 known websites with ground-truth labels:

- **5 healthy sites** (Google, Wikipedia, Stripe, GitHub, Apple) — must classify as healthy.
- **5 broken sites** exhibiting known defects (HTTP-only, missing viewport meta, and similar) — must classify as broken with the correct defect signals.

Runs against the live web through the real Playwright scoring path. Pass bar is ≥80% classification accuracy; current result is 10/10.

This gate exists because the scorer is the pipeline's judgment call — if it can't reliably separate a healthy site from a broken one, every downstream preview and pitch is built on noise. It also pins the LLM: the model powering the pipeline cannot be changed unless this eval still passes.

## Gate 2 — Phase 2 Outreach Eval (17/17)

Fully mocked (no network), covering the outreach system's correctness and compliance surface:

- **HMAC webhook verification** — signature validation on inbound Meta webhooks.
- **Inbound and status message parsing** — replies and delivery receipts decode correctly.
- **24-hour session window tracking** — free-form vs. template routing flips at the window boundary.
- **Template body formatting** — pre-approved template payloads are built correctly.
- **CRM stage machine** — lead stages advance only along legal transitions.
- **Follow-up scheduling** — due follow-ups are computed correctly.
- **Consent gate (UWG §7)** — the drafter *must raise* for any lead without recorded consent. The compliance rule is asserted as a test, so it cannot silently regress.

Result: 17/17.

## Gate 3 — Phase 3 Copilot Eval (14/14)

Fully mocked (ChromaDB on a temp path with deterministic fake embeddings; no live ASR), covering the live-call loop end to end:

- **ASR message parsing and transcript assembly** — streaming ASR events assemble into a coherent transcript.
- **Playbook indexing and retrieval** — RAG collections index and return the expected context for a query.
- **Suggestion parsing, filtering, and caps** — the max-2-suggestions and ≥0.6-confidence rules are enforced.
- **Session lifecycle and persistence fallback** — sessions journal to disk and recover after a simulated crash.
- **Delivery formatting** — suggestion payloads render correctly for each delivery channel.

Result: 14/14.

## Why Evals as Merge Gates

The pipeline's riskiest components are exactly the ones that are hardest to review by eye: an LLM-backed scorer, a legally constrained outreach path, and a real-time suggestion loop. Encoding each one's contract as an executable eval means a model upgrade, prompt tweak, or refactor either proves it still holds the line — or it doesn't merge.
