# Fleet Architecture

## Overview

An autonomous agent fleet on a single server, grouped by role.
A Postgres-backed dispatcher leases tasks from a queue and runs each agent as a headless Claude Code
process with an explicit tool allowlist.

Runs (queue table, read on 18 Nov 2026): **6,598 runs (attempts) and 6,438 tasks since 10 Jun 2026**
([count](../evals/2026-11-18-fleet-runs.json)).
Before the self-healer shipped on 2026-08-22, 43% of all runs (471 of 1,088) were failed or stale;
that backlog drained to zero the same day ([snapshot](../evals/2026-08-25-fleet-stats.json)).

Beside the fleet, I work with parallel Claude Code lanes, each with a fixed role, a security lane that
reads every change, and my approval before anything is released.

## Flow

```text
Trigger (cron, webhook, Telegram, post-commit, self-healer event)
    │
    ▼
sherry-enqueue <agent> -m "task" --priority 2 --dedup-key <key>
    │
    ▼
┌──────────────────────────────────┐
│  PostgreSQL task queue           │
│  queue with dedup keys + leases  │
└─────────┬────────────────────────┘
          │ FOR UPDATE SKIP LOCKED
          ▼
┌──────────────────────────────────┐
│  Dispatcher (long-running)       │
│  - leases the next eligible task │
│  - spawns claude -p "<prompt>"   │
│  - records cost and status       │
│  - commits authored files to a   │
│    side branch, never to main    │
└─────────┬────────────────────────┘
          │
          ▼
Agent output → Telegram digest, CRM, or a branch for the lander
```

## Roles

The agents are grouped by role: infrastructure and security, code quality, product delivery, growth and
sales, research and learning, content and brand.

Each agent is one catalog entry: role, tool allowlist, trigger, model tier, turn limit and budget.
An agent runs only when a trigger gives it real work.

## Models and cost

- Fleet agents run on Claude: Haiku for most jobs, Opus for the few that need deeper reasoning.
- Product LLM calls (retrieval assistants, Sales OS drafts and scoring) run through OpenRouter.
- A cost ledger prices every fleet run from its token usage at list price. A daily cap stops new work when
  it is reached; past 80% of it, only priority-1 work is leased.
- Dedup keys stop the same work from being queued twice; per-task budgets flag runs that overspend.

## Self-healing loop (since 2026-08-22)

Every 30 minutes the healer classifies failed runs (transient platform error, tool denied, timeout,
real bug), applies a policy (requeue with cooldown, skip stale periodic work, escalate), and remediates
from an allowlisted registry: restart a container or a service, never the dispatcher itself.
A daily cap of 12 side-effect remediations is the kill switch; past it, the healer only escalates to
Telegram. A predictive monitor runs every 10 minutes, no-LLM probes every 5, and a health report
reaches Telegram every morning at 09:00.

## Safety boundaries

- Every agent runs with an explicit `--allowedTools` list, the only privilege boundary, and every prompt
  treats fetched content as data, never as instructions.
- One writer per git repository at a time, and a memory gate that holds new work when free memory runs low.
- The dispatcher commits only the files a task wrote, to `agent/<name>/<task-id>`, with trailers that tie
  the commit to its agent and task. Nothing an agent writes lands on main directly.
- **The lander** (since 2026-09-28) is the only path from an agent branch to main, and it takes only
  test-only and docs-only changes: provenance trailers on every commit, no protected paths, size caps,
  a secret scan, test code that stays offline, and every CI check green on the exact head commit.
  Everything else becomes a pull request for human review. A red CI run on main pauses the lander.

## Task chaining

`finish()` can enqueue child tasks with a `parent_task_id` link. Example: sentinel-prime detects an issue,
enqueues incident-responder, whose finding enqueues code-reviewer; code-reviewer hands untested logic to
test-engineer.
