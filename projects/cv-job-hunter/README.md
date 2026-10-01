# Job Pipeline

A job-search pipeline that finds roles fitting a candidate profile and prepares everything needed to
apply: a one-page CV tailored to the posting and a cover letter that an independent reviewer has checked.
Nothing is sent automatically; a person sends every application.

> **Status (2026-09-28):** runs daily at 08:00 UTC, 308 to 383 postings per run over 23 to 26 September,
> plus an on-demand sourcing pass over **83 company career boards** (2,981 postings on 26 September).
> [Evidence](../../evals/2026-09-26-job-pipeline.json). Public code:
> [job-pipeline](https://github.com/sherrybuilds-studio/job-pipeline), with a fictional example profile.

## Two passes and a kit builder

**Daily run.** Scrapes job boards, drops roles that fail a hard requirement, scores the rest with rules
(no model call), prepares a tailored CV for each strong match and sends a Telegram digest. A small graph decides between
researching, widening the search and reporting "no jobs today".

**Sourcing pass, on demand.** Pulls company career boards directly (Greenhouse, Ashby, Lever, Personio,
SmartRecruiters, Join), then applies its gates in order: hard requirements, role type, field, location,
already applied, and finally a liveness check on the employer's own posting page. The result is a report
of ready and weak roles.

**Application kits.** For each ready role: a tailored one-page CV, a cover-letter draft, an independent
review, at most one revision, and a final honesty gate. A kit is cleared only at zero unsupported claims.

## Architecture

```text
DAILY RUN (cron, 08:00 UTC)
  job boards ──▶ hard requirements ──▶ rule-based score (no LLM)
                                           │
               ┌───────────────────────────┼──────────────────────────┐
               ▼                           ▼                          ▼
         strong match               weak match                   no match
       tailored CV            widen the search, retry         "no jobs today"
               └──────────────▶ Telegram digest ◀─────────────────────┘

SOURCING PASS (on demand)
  83 career boards + public boards
    ──▶ hard requirements ──▶ role type ──▶ field ──▶ location
    ──▶ already applied? ──▶ employer page still open? ──▶ READY or WEAK report

KIT (per READY role, never sends)
  tailored CV ──▶ letter draft ──▶ independent reviewer ──▶ ≤ 1 revision ──▶ honesty gate
    ──▶ lint on the rendered PDFs ──▶ cleared kit ──▶ a person sends it
    ──▶ send gate: posting live in the last 24 h, lint clean ──▶ follow-up draft on day 4
```

## Key engineering decisions

### Rule-based scoring instead of a model

Matching uses deterministic weighted scoring against the profile, with no model call per posting. Hundreds
of postings are ranked every day at no model cost, and every score can be explained. The model is
reserved for the one job where it earns its cost: writing.

### Hard requirements before scoring

Roles the candidate cannot honestly apply to (a required language level, a required degree, the wrong
employment type) are dropped before scoring, each with a logged reason. A high score can never
override a hard requirement.

### The employer's own page decides

Aggregator listings go stale. Before anything is written, the pipeline resolves each role to the
employer's own posting page and checks that it is still open. The send gate checks again: a send is
stamped only if the page was live within the last 24 hours.

### An independent reviewer at zero unsupported claims

The reviewer reads the letter in a fresh context with the posting fenced off as untrusted data, so
instructions hidden in a job ad cannot steer it. Any claim the profile does not back is an unsupported
claim, and one is enough to block the kit.

### Lint what the recruiter receives

The final lint reads the text layer of the rendered PDFs, not the draft: placeholders, empty buzzwords,
banned claims and formatting slips mark the kit for a human fix.

### A person sends every application

The pipeline prepares; it never sends. Follow-ups are drafts too.
