# RAG Stack

## Components

- **ChromaDB** — persistent vector store (one collection per product)
- **Embeddings service** — one shared MiniLM service (all-MiniLM-L6-v2) for every product
- **Semantic cache** — answers a repeated question without a model call when cosine similarity clears the threshold (0.92 to 0.95, per product)

## Hybrid Search (commerce agent pattern)

```text
User query
    │
    ├──► Keyword search (exact SKU / material name matches)
    │         │
    │         ▼  ranked first
    ├──► Semantic search (ChromaDB vector similarity)
    │         │
    │         ▼  fills remaining slots
    └──► Merged results → LLM context → response
```

Keyword results rank first (exact matches are high-confidence).
Semantic results fill the remaining context window.

## Semantic Cache

```text
Incoming query
    │
    ▼
Embed query → compare to cache keys (cosine similarity)
    │
    ├──► ≥ threshold (0.95)? → return cached answer (zero LLM cost)
    │
    └──► < threshold? → full RAG pipeline → cache the result
```

In the commerce agent the cache holds entries for 7 days, capped at 500 with least-recently-used eviction, so stale prices age out.

## Playbook RAG (Sales OS)

Three ChromaDB collections, queried in parallel and merged by distance:
- `sales_playbook` — call flow, discovery questions, closing moves
- `objection_handling` — named objections + counters
- `pricing_faq` — what the setup covers, timelines, payment terms

Bilingual seed docs: every entry is German + English in one text block,
so retrieval works whichever language the prospect is speaking.
