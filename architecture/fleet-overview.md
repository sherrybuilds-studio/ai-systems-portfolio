# Fleet Architecture

## Overview

A single-server fleet of **54 agents** in six teams ([catalog snapshot](../evals/2026-09-26-fleet-agents.json)).
A Postgres-backed dispatcher leases tasks from a queue and runs each agent as a headless Claude Code
process with an explicit tool allowlist.

Verified numbers (database snapshot, 2026-09-24): **2.4% hard failures over the 1,000 runs from
9 July to 24 September** (803 done, 173 skipped, 24 failed; [snapshot](../evals/2026-09-24-fleet-stats.json)).
Before the self-healer shipped on 2026-08-22, 43% of all runs (471 of 1,088) were failed or stale;
that backlog drained to zero the same day.

## Flow

```text
Trigger (cron, webhook, Telegram, post-commit, self-healer event)
    │
    ▼
sherry-enqueue <agent> -m "task" --priority 2 --dedup-key <key>
    │
    ▼
┌──────────────────────────────────┐
│  PostgreSQL: agent_tasks         │
│  queue with dedup keys + leases  │
└─────────┬────────────────────────┘
          │ FOR UPDATE SKIP LOCKED
          ▼
┌──────────────────────────────────┐
│  Dispatcher (PM2 process)        │
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

## The six teams

| Team | Agents |
| --- | --- |
| Infrastructure and security (14) | sentinel-prime · security-auditor · incident-responder · backup-verifier · disk-ram-warden · cert-dns-checker · dependency-bumper · log-summarizer · secret-scanner · secret-guard · fleet-janitor · deploy-monitor · stripe-billing-guard · cost-warden |
| Code quality (9) | code-reviewer · test-engineer · eval-runner · refactor-surgeon · bug-triager · perf-profiler · docs-keeper · merge-captain · migration-foreman |
| Product delivery (9) | montari-keeper · restaurant-keeper · jobhunt-keeper · rag-curator · prompt-tuner · conversation-qa · whatsapp-launcher · interior-bot-builder · demo-builder |
| Growth and sales (8) | lead-qualifier · outreach-drafter · proposal-writer · pipeline-analyst · competitor-watcher · market-scanner · testimonial-collector · pricing-strategist |
| Research and learning (7) | morning-researcher · claude-tracker · fact-checker · paper-digester · tool-scout · uni-assistant · idea-curator |
| Content and brand (7) | content-strategist · linkedin-ghostwriter · case-study-writer · portfolio-keeper · video-script-writer · brand-voice-guard · weekly-narrator |

Each agent is one catalog entry: role, tool allowlist, trigger, model tier, turn limit and budget.
An agent runs only when a trigger gives it real work.

## Models and cost

- Fleet agents run on Claude: Haiku for most jobs, Opus for the few that need deeper reasoning.
- Product LLM calls (retrieval assistants, Sales OS drafts and scoring) run on GLM-5.2 through OpenRouter.
- A cost ledger prices every fleet run from its token usage at list price. A daily cap stops new work when
  it is reached, with a soft warning at 80%.
- Dedup keys stop the same work from being queued twice; per-task budgets flag runs that overspend.

## Self-healing loop (since 2026-08-22)

Every 30 minutes the healer classifies failed runs (transient platform error, tool denied, timeout,
real bug), applies a policy (requeue with cooldown, skip stale periodic work, escalate), and remediates
from an allowlisted registry: restart a container or bounce a PM2 process, never the dispatcher itself.
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
