# Restaurant Reservation Assistant

A conversational assistant for restaurants. Guests book tables, join a waitlist, get reminders and ask
menu questions in their own language. Every menu answer is grounded in the restaurant's own data through
retrieval, so the bot never invents a dish, a price or an allergen.

> **Status (2026-09-28):** built and tested, not deployed. Retrieval gate 10 of 10
> ([eval, 2026-09-02](../../evals/2026-09-02-restaurant-bot-eval.json)).
> Public code: [reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent), the WhatsApp
> side with a demo restaurant and its eval.

## What it does

- **Reservations**: a multi-turn booking flow that extracts date, time and party size, checks
  availability and confirms.
- **Waitlist**: when a slot frees up, the next guest is offered it with a short confirm window.
- **Reminders**: 24 hours and 2 hours before the booking; the guest confirms or cancels by replying.
- **Menu and FAQ**: grounded answers from a vector index built from the restaurant's `menu.json`.
- **Owner tools**: new-review alerts, reply drafts the owner approves (never auto-posted), daily and
  weekly reports on Telegram, and broadcasts to guests who opted in.
- **Languages**: German by default, or the guest's language.

## Tech stack

| Layer | Technology |
| --- | --- |
| Language | Python 3 |
| Messaging | WhatsApp Cloud API webhook (FastAPI, HMAC-verified); a Telegram front end with long polling in the private version |
| LLM | Claude (Haiku class) through OpenRouter |
| Retrieval | ChromaDB with hybrid search (semantic plus keyword), MiniLM embeddings |
| Data | Supabase (reservations, customers, waitlist, logs) |
| Config | Pydantic settings, validated at boot |
| CI | Retrieval gate on every push |

## Architecture

```text
                        ┌──────────────────────────────┐
  Guest on WhatsApp ───▶│ webhook (FastAPI)            │
  or Telegram      ────▶│ signature check, rate limit, │
                        │ prompt-injection filter      │
                        └──────────────┬───────────────┘
                                       │
                          ┌────────────▼────────────┐
                          │     assistant core      │
                          │ intent rules first,     │
                          │ session memory (TTL,    │
                          │ LRU), language per guest│
                          └──────┬───────────┬──────┘
                                 │           │
                     ┌───────────▼──┐   ┌────▼────────────┐
                     │ retrieval    │   │ LLM (Claude     │
                     │ ChromaDB,    │   │ via OpenRouter) │
                     │ hybrid over  │   │ grounded reply  │
                     │ menu index   │   └─────────────────┘
                     └──────┬───────┘
                            │
                   ┌────────▼────────┐        ┌──────────────────────────┐
                   │   menu.json     │        │ Supabase: reservations,  │
                   │ one file per    │        │ waitlist, reminders      │
                   │ restaurant      │        └──────────────────────────┘
                   └─────────────────┘
```

## Key engineering decisions

**Rules before the model.** Intent detection for bookings runs as plain rules first, so a clear
"table for 4 tomorrow at 7" never costs a model call and never gets misread.

**Grounding from a single source of truth.** Each restaurant's menu, policies and FAQ live in one
`menu.json`; the index is rebuilt from it deterministically. Onboarding a new restaurant is a data task,
not a code change, and CI can rebuild the exact same index.

**Eval gate in CI.** Every push rebuilds the index and runs 10 gold questions (dishes, allergens, prices,
vegan and vegetarian options, hours, group policy, location, one in German) with no secrets and no model
calls. A regression in retrieval fails the build.

**Bounded memory by design.** The in-memory session stores (conversations, pending reservations, language
per guest) once grew without limit. They now have a 24-hour TTL with LRU eviction capped at 500 sessions,
and the process supervisor enforces a hard memory ceiling as a last line of defence.

**Fail-fast configuration.** Settings are validated at boot: a missing API key stops the process with a
clear error instead of surfacing as a silent `None` on the first model call.

**Long polling for Telegram.** The Telegram front end pulls updates instead of exposing an endpoint: no
inbound port and no public attack surface for that channel.

**Drafts, not posts.** Review replies are drafted for the owner to approve. Posting to Google needs the
Business Profile API with owner OAuth, which is not built.

## Business model

Offered as a one-time setup plus a monthly retainer. Because a restaurant is configuration (menu data and
settings) on a shared codebase, each new restaurant adds little engineering work.
