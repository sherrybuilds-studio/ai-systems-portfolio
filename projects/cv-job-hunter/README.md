# CV Job Hunter

An automated daily job-hunting pipeline for the Berlin AI/tech market. It scrapes multiple job boards, scores every listing against a candidate profile with fast rule-based matching, enriches promising leads with Playwright-driven contact discovery, generates tailored cover letters with Claude, and delivers a morning Telegram digest — fully hands-off from scrape to application-ready shortlist.

> **Status (2026-08-25):** ran daily on cron; last run **2026-08-20** (105 scored matches on file). **Parked on 2026-08-23** to focus on the voice receptionist. Public code: [job-hunt-ai](https://github.com/sherrybuilds-studio/job-hunt-ai).

## Tech Stack

- **Python** — pipeline stages as small, single-purpose modules
- **Job sources** — Adzuna API, Arbeitnow API, Firecrawl (for boards without an API)
- **Playwright** — headless-browser enrichment for contact discovery and page-level filtering
- **Claude (via OpenRouter)** — cover-letter generation only; matching stays LLM-free
- **Supabase (Postgres)** — application tracking with insert/dedup/mark-applied lifecycle
- **Telegram Bot API** — daily digest delivery on a morning cron schedule

## Architecture

```
                    ┌──────────────────────────────┐
                    │           SOURCES            │
                    │  Adzuna API   Arbeitnow API  │
                    │  Firecrawl (scraped boards)  │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
 profile.py ┐          ┌─────────────────────┐
 (PROFILE   ├─────────►│  scraper.py         │  normalize listings
  dict —    │          └─────────┬───────────┘
  single    │                    ▼
  source of │          ┌─────────────────────┐
  truth)    ├─────────►│  matcher.py         │  rule-based weighted
            │          │  score 0–105 pts    │  scoring — instant, free
            │          └─────────┬───────────┘
            │                    ▼
            │          ┌─────────────────────┐
            │          │  enrich.py          │  Playwright contact
            │          │  (Playwright)       │  enrichment + hard-reject
            │          └─────────┬───────────┘  filters (role level,
            │                    ▼               parked domains)
            │          ┌─────────────────────┐
            └─────────►│  writer.py          │  Claude cover letter
                       │  (LLM step)         │  per job, ≤300 words
                       └─────────┬───────────┘
                                 ▼
                       ┌─────────────────────┐
                       │  tracker.py         │  Supabase applications
                       │                     │  table — insert / dedup /
                       └─────────┬───────────┘  mark-applied
                                 ▼
                       ┌─────────────────────┐
                       │  notifier.py        │  Telegram digest:
                       │                     │  hot matches + top 5
                       └─────────────────────┘  of the day
```

## Key Engineering Decisions

### Rule-based scoring instead of an LLM
Matching uses deterministic weighted scoring (max 105 points) against a candidate profile — no model call per listing. This makes scoring **instant, free, and reproducible**: hundreds of listings can be ranked daily at zero marginal cost, and score changes are always explainable. The LLM is reserved for the one task where it earns its cost — writing the cover letter.

### Playwright enrichment with hard-reject filters
Listings that survive scoring go through a headless-browser enrichment pass that visits company pages to discover contact information. The same pass applies **hard-reject filters** — role-level mismatches and low-quality signals like parked/placeholder domains are dropped before any LLM spend or human attention. Fuzzy substring matching handles messy data, such as company info embedded inside the name field. This enrichment module was solid enough to be extracted and reused as a website-quality scorer in a separate sales pipeline.

### Deduplication at the tracking layer
The tracker owns the insert/dedup/mark-applied lifecycle against the database, so the same job seen across multiple boards (or across days) never produces duplicate cover letters or repeated digest entries. Dedup lives in one place rather than being re-implemented per source.

### Single source of truth for the candidate profile
All stages read from one `PROFILE` dict in `profile.py`. Scoring weights, cover-letter personalization, and filters stay in sync automatically — updating skills or preferences is a one-file change.

### Cheap-first pipeline ordering
Stages are ordered by cost: free API scraping → free rule-based scoring → browser enrichment → paid LLM writing. Each stage shrinks the set before the next, more expensive stage runs, keeping daily operating cost near zero.

### Push, not pull
Results arrive as a scheduled morning Telegram digest (hot matches above threshold plus the day's top five), turning job hunting from an active chore into a review-and-apply routine.
