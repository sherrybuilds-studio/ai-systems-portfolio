# Fleet Architecture

## Overview

A 50-agent system on a single VPS. One PostgreSQL-backed dispatcher leases
tasks from a queue and spawns agents as headless Claude Code processes.

## Flow

```
Trigger (cron / webhook / Telegram / post-commit)
    │
    ▼
sherry-enqueue <agent> -m "task" --priority 2
    │
    ▼
┌─────────────────────────────────┐
│  PostgreSQL: agent_tasks table   │
│  (queue with dedup + lease)      │
└─────────┬───────────────────────┘
          │ FOR UPDATE SKIP LOCKED
          ▼
┌─────────────────────────────────┐
│  Dispatcher (PM2 process)        │
│  - leases next eligible task     │
│  - spawns: claude -p "<prompt>"  │
│  - captures stdout → result      │
│  - finish() → cost + status      │
└─────────────────────────────────┘
          │
          ▼
    Agent output → Telegram / CRM / Git
```

## Model Routing

- **GLM-5.2** (z-ai/glm-5.2 via OpenRouter) — fleet default. ~$0.001/1M input,
  ~$0.003/1M output. 1M token context. Used for 90% of agent tasks.
- **Claude** (OAuth via Claude Max) — deep reasoning roles only. ~€140/mo flat.

## Teams (6)

1. **Infra & Security** (10) — watchdog, auditor, incident responder, backup verifier
2. **Code Quality** (10) — reviewer, test engineer, refactor surgeon, bug triager
3. **Product Keepers** (5) — per-product maintainers (montari, restaurant, RAG curator)
4. **Launch & Demos** (3) — WhatsApp launcher, demo builder, interior bot builder
5. **Sales & Growth** (7) — outreach drafter, proposal writer, competitor watcher
6. **Research & Content** (8) — fact checker, paper digester, LinkedIn ghostwriter

## Task Chaining

Tasks can chain: `finish()` auto-enqueues `child_task_spec` entries with
`parent_task_id` linking. Example: sentinel-prime detects issue → enqueues
incident-responder → incident-responder finishes → enqueues code-reviewer.

## Cost Control

- Per-task budget cap (`budget_cents`)
- Daily spend counter (`spent_today_cents`)
- Dedup keys prevent duplicate work
- Semantic cache for repeated LLM calls
