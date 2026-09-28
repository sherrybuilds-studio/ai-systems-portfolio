# Commerce RAG Agent — WhatsApp Product Assistant

An AI sales assistant for a furniture brand, running over WhatsApp. It answers product questions from the
brand's own catalogue (prices, wood types, finishes, lead times) in a respectful brand voice, and a
separate pipeline finds and scores likely buyers. Grounding is strict: the assistant quotes only prices
that exist in the catalogue.

> **Status (2026-09-28):** pilot. The pilot converses in Urdu and English; the public version,
> [commerce-rag-agent](https://github.com/sherrybuilds-studio/commerce-rag-agent), answers in English and
> German.

## Result

Sending only the retrieved products to the model, instead of the whole catalogue in every prompt, cut the
prompt from **1,118 to 695 tokens per message (38%)**, measured on 2026-04-27
([changelog](https://github.com/sherrybuilds-studio/commerce-rag-agent/blob/main/CHANGELOG.md)). The saving
comes from retrieval, not from the cache, and it measures prompt size, not a billing total.

## Tech stack

| Layer | Technology |
| --- | --- |
| Language | Python |
| Messaging | Meta WhatsApp Cloud API (Graph API webhooks) |
| Webhook server | FastAPI |
| Vector store | ChromaDB (persistent) |
| Embeddings | `all-MiniLM-L6-v2` (sentence-transformers) |
| LLM | Claude 3.5 Haiku through OpenRouter |
| Chunking | LangChain `RecursiveCharacterTextSplitter` (512 characters, 100 overlap) |
| Leads | Supabase, plus a shared sheet for outreach |
| Automation | n8n (scheduled lead runs, error alerts) |
| CI | ruff, gitleaks, catalogue validation and an offline retrieval gate on every push |

## Architecture

```text
Customer (WhatsApp)
        │
        ▼
Meta WhatsApp Cloud API ──▶ webhook (FastAPI: verify-token handshake, rate limit, injection filter)
        │
        ▼
   Assistant core
        │
        ├─ 1. Semantic cache lookup (MiniLM embedding,
        │      cosine similarity 0.95 or more → return the cached answer)
        │
        ├─ 2. Cache miss → hybrid retrieval
        │      ├─ keyword search  (SKUs, wood and material names, exact terms)
        │      └─ semantic search (ChromaDB vectors)
        │      keyword hits rank first, semantic results fill the rest
        │
        ├─ 3. Model call (Claude 3.5 Haiku through OpenRouter)
        │      the system prompt enforces tone and grounding rules
        │
        └─ 4. Cache the new answer (7-day TTL, 500 entries, LRU eviction)
        │
        ▼
Reply through the Graph API (at most 4 lines per message)
```

A separate lead pipeline scores property listings for buyers who are likely to furnish a new home (see
[scorer.md](./scorer.md)) and writes qualified leads to the CRM for outreach.

## Key engineering decisions

### Hybrid search over pure vector search

Pure semantic search missed exact-match queries: SKU codes like `MOK-001` and material names embed
poorly. The retriever runs a keyword pass over structured product fields (id, name, category, wood,
finish options) and a semantic pass over ChromaDB, with keyword hits taking priority.

### Retrieval instead of the whole catalogue

The first version put the full catalogue into every prompt. Retrieving only the relevant products cut
the prompt by 38% (see Result) and keeps the prompt small as the catalogue grows.

### Semantic cache before every model call

Customers ask the same questions in slightly different words. Incoming questions are embedded and
compared with cached answers; at 0.95 cosine similarity or more, the cached answer is returned without a
model call. A 7-day TTL and a 500-entry cap with LRU eviction let stale prices age out.

### Tone as a hard rule, not a vibe

The system prompt encodes the brand's register explicitly, for example the respectful Urdu forms "Jee"
instead of "haan" and "Aap" instead of "tum". Treating tone as testable prompt constraints, rather than
"be polite", keeps the voice consistent across model changes.

### Grounded answers only

The prompt forbids invented prices: if a figure is not in the retrieved context, the assistant does not
quote it. Product data lives in one JSON file that the indexer chunks into vectors carrying the product
id, category and price.

### WhatsApp-native brevity

Replies are capped at 4 lines. Long answers get ignored on WhatsApp; short ones get read.

### Gates that fit the cost

The retrieval gate, catalogue validation, lint and a secret scan run on every push with no paid key. The
full 10-question assistant eval calls the live model, so it runs by hand before a deploy and must score
80% or more.

### Operational hardening

The Meta verify-token handshake, input sanitisation against prompt injection, rate limiting (10 requests
per minute per IP), settings validated at boot (a missing API key stops the process), and secret-scanning
pre-commit hooks.
