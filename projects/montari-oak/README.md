# Montari Oak AI — WhatsApp Sales Bot with RAG

An AI sales assistant for a luxury furniture brand in Lahore, running entirely over WhatsApp. It answers product questions from a RAG-backed catalogue (prices, wood types, finishes, lead times), holds a bilingual Urdu/English conversation in a respectful brand voice, and feeds a lead-generation pipeline that finds and scores high-value prospects automatically. Grounding is strict: the bot only quotes prices that exist in the vector store — it never invents them.

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python |
| Messaging | Meta WhatsApp Cloud API (Graph API webhooks) |
| Webhook server | FastAPI |
| Vector store | ChromaDB (persistent) |
| Embeddings | `all-MiniLM-L6-v2` (sentence-transformers) |
| LLM | Claude 3.5 Haiku via OpenRouter |
| Chunking | LangChain `RecursiveCharacterTextSplitter` (512 chars, 100 overlap) |
| CRM / storage | Supabase (leads), Google Sheets (outreach list) |
| Observability | Langfuse (per-agent traces and cost tagging) |
| Automation | n8n (scheduled lead runs, error alerting) |
| Eval / CI | 10-question eval gate (must score ≥ 80% to merge), ruff + pytest in CI |

## Architecture

```
Customer (WhatsApp)
        │
        ▼
Meta WhatsApp Cloud API ──▶ webhook (FastAPI, HMAC-verified)
        │
        ▼
   Bot core
        │
        ├─ 1. Semantic cache lookup (MiniLM embedding,
        │      ≥95% cosine similarity → return cached answer)
        │
        ├─ 2. Cache miss → hybrid retrieval
        │      ├─ keyword search  (SKUs, wood/material names, exact terms)
        │      └─ semantic search (ChromaDB vectors)
        │      keyword hits ranked first, semantic fills the rest
        │
        ├─ 3. LLM call (Claude 3.5 Haiku via OpenRouter)
        │      system prompt enforces tone + grounding rules
        │
        └─ 4. Cache the new answer (TTL + LRU eviction)
        │
        ▼
Reply via Meta Graph API  (max 4 lines per message)
```

A separate lead-generation pipeline scrapes public property portals for luxury listings, scores them (see [scorer.md](./scorer.md)), and writes qualified leads to Supabase and Google Sheets for outreach.

## Key Engineering Decisions

### Hybrid search over pure vector search
Pure semantic search missed exact-match queries — SKU codes like `MOK-001` and material names embed poorly. The retriever runs a keyword pass over structured product fields (id, name, category, wood, finish options) *and* a semantic pass over ChromaDB, with keyword hits taking priority. This fix alone pushed retrieval past the 80% CI gate (10/10 gold questions at build time, June 2026; token cost per message fell 38%, 1,118 → 695).

### Semantic cache before every LLM call
Customers ask the same questions in slightly different words. Incoming queries are embedded and compared against cached Q&A pairs; at ≥95% similarity the cached answer is returned with zero LLM cost. The cache has a 7-day TTL and LRU eviction (500 entries) so stale prices age out.

### Bilingual tone as a hard rule, not a vibe
The system prompt encodes Urdu register explicitly: **"Jee" instead of "haan"**, **"Aap" instead of "tum"** — the respectful forms a luxury brand would use with a customer. Treating tone as testable prompt constraints (rather than "be polite") keeps the voice consistent across model swaps.

### Grounded answers only
The prompt forbids hallucinated prices: if a figure isn't in the retrieved ChromaDB context, the bot doesn't quote it. Product data lives in a single JSON source of truth that the indexer chunks (512/100) into per-chunk vectors carrying product id, category, and price metadata.

### WhatsApp-native brevity
Replies are capped at 4 lines. Long answers get ignored on WhatsApp; short ones get read.

### Eval as a merge gate
A 10-question eval suite runs in CI and exits non-zero below 80%, so retrieval or prompt regressions block the merge. A model bake-off on the same gate is how Claude 3.5 Haiku was chosen (better score at lower cost than the newer candidate); swapping models is a one-line config change.

### Operational hardening
HMAC signature verification on the webhook, input sanitization, rate limiting, fail-fast settings validation at boot (missing API key = refuse to start), and secret-scanning pre-commit hooks.
