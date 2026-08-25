# Eval Framework

## Principle

Every AI feature has an automated test gate. A change that drops a product
below its eval threshold (≥80%) must not merge. CI enforces this.

## Pattern

```
1. Define N known test cases (10 for website scorer, 14-17 for pipeline gates)
2. Run the feature against all cases
3. Count correct classifications / passes
4. Gate: pass_rate >= 80% → exit 0, else exit 1 (blocks merge)
```

## Real Examples (dated — see each product's `docs/evals/*.json`)

### Voice Receptionist — outcome QA (2026-08-25)
- 12 golden call transcripts scored by a deterministic rubric: booking
  outcome, number grounding (whole-token match), AI disclosure present,
  recording-consent handling
- Offline, no LLM in the loop; wired into `make eval` and CI
- Result: 12/12 (100%)

### Sales OS — missed-call exposure scorer (2026-08-25)
- 10 fixture leads (5 high exposure, 5 low) + a medical/dental exclusion check
- Pure function, no network; gate ≥80%
- Result: 10/10 (100%)

### Restaurant Bot — RAG retrieval (2026-08-25)
- 10 gold-standard questions against a freshly rebuilt ChromaDB index, hybrid search
- No LLM calls, no secrets; CI rebuilds the index and blocks the merge below 10/10
- Result: 10/10 (100%), average retrieval score 0.646

### Sales OS — Phase 1: Website Scorer (2026-07-07)
- 10 known websites: 5 healthy (Google, Wikipedia, Stripe, GitHub, Apple)
- 5 broken (HTTP-only, no viewport meta, etc.)
- Gate: classification accuracy ≥ 80%
- Result: 10/10 (100%)

### Sales OS — Phase 2: WhatsApp Outreach (2026-07-07)
- 17 tests: HMAC verification, inbound parsing, 24h session window,
  template body formatting, stage machine, UWG §7 consent gate
- Fully mocked (no network)
- Result: 17/17 (100%)

### Sales OS — Phase 3: Call Copilot (2026-07-07)
- 14 tests: ASR parsing, playbook indexing/retrieval (fake embeddings),
  suggestion filtering/caps, session lifecycle, delivery formatting
- Fully mocked
- Result: 14/14 (100%)

## Why It Matters

When switching from one LLM to another (e.g., Claude → GLM-5.2), the eval
gate catches regressions before they reach production. A model that scores
below 80% on the eval doesn't ship — period.
