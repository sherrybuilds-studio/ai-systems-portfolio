# Lead Scoring Model

The lead-generation pipeline scrapes public property portals for luxury real-estate listings (the target customer for high-end furniture: someone who just bought an expensive, unfurnished home). Every raw listing is scored 0–100 across four weighted signals, filtered, tiered, and sorted so the sales team always contacts the best leads first.

## Signals and Weights

| Signal | Max points | Rationale |
|---|---|---|
| Property price | 40 | Budget proxy — the strongest predictor of furniture spend |
| Location prestige | 40 | Premium areas correlate with luxury-brand fit |
| Listing freshness | 20 | New buyers furnish soon after purchase; stale listings go cold |
| Property type | 10 | Houses/villas need far more furniture than apartments |

Maximum total: 100 (price and location dominate by design — a fresh cheap flat should never outrank a week-old villa).

### Price (max 40)

| Property value (PKR) | Points |
|---|---|
| ≥ 10 crore | 40 |
| ≥ 5 crore | 30 |
| ≥ 3 crore | 20 |
| below 3 crore | 5 |

### Location (max 40)

Areas are bucketed into three prestige tiers (matched against listing location *and* title, since scraped location fields are unreliable):

| Tier | Example areas | Points |
|---|---|---|
| Tier 1 — premium | DHA/Defence, Gulberg, Cantt | 40 |
| Tier 2 — good | Bahria, Model Town, Garden Town, Johar Town | 25 |
| Tier 3 — standard | Iqbal Town, Township, Faisal Town, … | 15 |
| Unrecognized | — | 10 |

### Freshness (max 20)

| Listing age | Points |
|---|---|
| < 24 hours | 20 |
| < 3 days | 15 |
| < 7 days | 10 |
| older / unparseable date | 5 |

### Property type (max 10)

Keyword match on the listing title:

| Match | Points |
|---|---|
| House / villa / farmhouse / bungalow / kothi | 10 |
| No type keyword found | 7 |
| Apartment / flat / studio | 5 |

Unknown types score *between* houses and apartments deliberately — absence of evidence shouldn't punish a lead below a known apartment.

## Score → Tier → Action

| Total score | Tier | Action |
|---|---|---|
| ≥ 80 | ULTRA LUXURY | Contact first — top of the outreach queue |
| 60–79 | LUXURY | Contact — strong fit |
| 40–59 | STANDARD | Contact if capacity allows |
| < 40 | LOW | **Dropped** — never reaches the outreach list |

Pipeline behavior:

1. Every scraped listing is scored and enriched with a full `score_breakdown` (per-signal points), so a human can audit *why* a lead ranked where it did.
2. Leads below the minimum score (default 40) are filtered out entirely.
3. Survivors are sorted descending by score and written to the CRM and outreach sheet with `outreach_status: pending`.

## Design Notes

- **Deterministic, not ML.** Weights are hand-tuned business rules — transparent, auditable, and adjustable in one file. With low lead volume, a learned model would be overkill and unexplainable to the client.
- **Breakdown over black box.** Persisting per-signal points with each lead made weight-tuning conversations with the client concrete ("location is over-weighted for our niche") instead of anecdotal.
- **Freshness fails safe.** A malformed scrape date scores the minimum rather than crashing the batch.
