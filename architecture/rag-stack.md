# RAG Stack

## Components

- **ChromaDB** — persistent vector store (one collection per product)
- **Embeddings service** — shared MiniLM service on port 7010 (all-MiniLM-L6-v2)
- **Semantic cache** — cosine similarity threshold (0.92-0.95) to avoid repeat LLM calls

## Hybrid Search (Montari Oak pattern)

```
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

```
Incoming query
    │
    ▼
Embed query → compare to cache keys (cosine similarity)
    │
    ├──► ≥ threshold (0.95)? → return cached answer (zero LLM cost)
    │
    └──► < threshold? → full RAG pipeline → cache the result
```

Saves ~40% of LLM calls on repeated question patterns.

## Playbook RAG (Sales OS)

Three ChromaDB collections, queried in parallel and merged by distance:
- `sales_playbook` — call flow, discovery questions, closing moves
- `objection_handling` — named objections + counters
- `pricing_faq` — what €500 covers, timelines, payment terms

Bilingual seed docs: every entry is German + English in one text block,
so retrieval works whichever language the prospect is speaking.
