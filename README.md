# SherryBuilds — AI Engineering Portfolio

Shehryar Irfan · CS student in Berlin · AI automation engineer

I build production AI systems that replace manual workflows — WhatsApp bots,
lead pipelines, agent fleets, and call copilots. Everything is eval-gated at
≥80% before merge. No vibes, no hype — just working systems with test results.

## Projects

- **Sales OS** — Lead discovery → website scoring → bilingual preview cards → WhatsApp outreach → live call copilot. 3 phases, 3 eval gates at 100%.
- **Montari Oak** — Luxury furniture WhatsApp bot with hybrid RAG (keyword + semantic search) and a semantic cache. Lead scoring pipeline with 4-tier qualification.
- **Restaurant Bot** — WhatsApp automation for restaurant order-taking and customer interaction.
- **CV Job Hunter** — Job board scraping + Playwright website enrichment + LLM scoring.
- **Mehboob Steel** — Steel trading WhatsApp bot serving 14,000+ contacts with Roman Urdu NLU.

## Architecture

- **50-agent fleet** on a single VPS, orchestrated by a PostgreSQL-backed dispatcher
- **GLM-5.2 routing** — cost-efficient model for the fleet, Claude for deep reasoning
- **Eval-gated development** — every product has automated tests; CI blocks merges below 80%
- **Bilingual by design** — German + English for the Berlin market

## Tech Stack

Python · Playwright · ChromaDB · Supabase · Meta WhatsApp Cloud API · Deepgram ·
OpenRouter · Docker · PM2 · FastAPI · httpx

## Links

- Portfolio: [sherrybuilds.com](https://sherrybuilds.com)
- Email: codewithsherry1@gmail.com

---

*Code is private. Architecture is open. DM for demos.*
