# Restaurant Bot AI

A conversational AI assistant for restaurants, delivered over Telegram. It handles food orders, table reservations, and menu/FAQ questions in multiple languages, answering from the restaurant's own menu data via retrieval-augmented generation (RAG) — so responses stay grounded in what the restaurant actually offers. Built as a productized service: one codebase, configured per restaurant, designed to run 24/7 under process supervision.

> **Status (2026-08-25):** retrieval eval 10/10 on a fresh index today; the Telegram service is currently **offline** while the voice receptionist is the live product. Public code: [reservation-agent](https://github.com/sherrybuilds-studio/reservation-agent).

## What It Does

- **Orders** — guests browse and order in natural language; the bot resolves items against the menu index
- **Reservations** — multi-turn reservation flow with pending-state tracking and confirmation
- **Menu & FAQ** — grounded answers from a vector index built from the restaurant's `menu.json`
- **Multilingual** — per-session language detection and memory, so each guest is answered in their own language

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3 |
| Messaging | Telegram Bot API (long polling) |
| LLM | Claude (Haiku-class) via OpenRouter |
| Retrieval | ChromaDB with hybrid search |
| Config | Pydantic settings (fail-fast validation) |
| Runtime | PM2 process supervision, Docker image available |
| CI | Automated retrieval-eval gate on every push/PR |

## Architecture

```
                        ┌──────────────────────────────┐
  Guest on Telegram     │        Bot Process           │
 ───────────────────►   │  (long polling — no inbound  │
   messages / replies   │   port, no public webhook)   │
                        └──────────────┬───────────────┘
                                       │
                          ┌────────────▼────────────┐
                          │       AI Agent Core     │
                          │  session memory (TTL +  │
                          │  LRU) · response cache  │
                          │  · language tracking    │
                          └──────┬───────────┬──────┘
                                 │           │
                     ┌───────────▼──┐   ┌────▼────────────┐
                     │  RAG Layer   │   │   LLM (Claude   │
                     │  ChromaDB    │   │  via OpenRouter)│
                     │  hybrid      │   │  intent, reply  │
                     │  search over │   │  generation     │
                     │  menu index  │   └─────────────────┘
                     └──────┬───────┘
                            │
                   ┌────────▼────────┐
                   │   menu.json     │
                   │ (per-restaurant │
                   │  source data)   │
                   └─────────────────┘
```

Flow: a guest message arrives via long polling → the agent loads session state (conversation history, pending reservation, language) → relevant menu chunks are retrieved from the vector index → the LLM composes a grounded reply → the response goes back through a hardened Telegram client (retry/backoff, rate-limit handling, message-length truncation).

## Key Engineering Decisions

**Long polling over webhooks.** The bot pulls updates instead of exposing an HTTP endpoint. No inbound port, no public attack surface, and the Docker image needs only an outbound connection — which simplifies deployment for a service sold to small businesses.

**RAG grounding from a single source of truth.** Each restaurant's menu lives in one `menu.json` file; the ChromaDB index is rebuilt from it deterministically. That makes onboarding a new restaurant a data task, not a code task, and means the eval suite can rebuild the exact same index in CI.

**Eval gate in CI.** Every push runs a retrieval-only evaluation (10 scenario checks, no secrets required) that rebuilds the index and scores the bot's grounding. A regression in retrieval quality fails the build before it reaches production. Current score: 10/10.

**Bounded memory by design.** In-memory session stores (conversations, pending reservations, per-session language) originally grew without limit and leaked ~700 MB. Fixed with a 24-hour TTL plus LRU eviction capped at 500 sessions, periodic garbage collection, and a hard memory cap at the process supervisor level as a last line of defense.

**Fail-fast configuration.** Settings are validated with Pydantic at boot. A missing API key crashes immediately with a clear error instead of the previous behavior — a silent `None` that only surfaced on the first LLM call.

**Shared core library.** Telegram delivery is delegated to a shared internal client that handles exponential backoff, rate-limit (429) responses, message truncation at the platform's 4096-character limit, and a dry-run mode for testing without a live token — so every bot in the fleet gets the same hardened transport for free.

**Pragmatic runtime.** The bot runs under PM2 with an auto-restart memory ceiling; a Dockerfile exists for containerized deployment (vector store mounted as a volume) when the shared embeddings service cut-over completes.

## Business Model

Productized AI service for restaurants: one-time setup fee plus monthly retainer. The multi-tenant-by-configuration design (per-restaurant menu data + settings, shared codebase) keeps marginal onboarding cost near zero.
