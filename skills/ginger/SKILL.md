---
name: ginger
description: Agentic engineering framework: implement features and fixes, review code, explore codebases, and clean dead code through a plan-driven pipeline with durable state. Modes act as commands - implement (plan, decompose, execute, verify, review), review, explore, cleanup. Works with any harness; parallel work supercharged by opencode-ensemble when available. Triggers on "implement", "fix", "build a feature", "review code", "explore codebase", "clean up dead code", "plan this work".
---

# Ginger

A mode-driven engineering framework. Each mode is a command: pick it, follow its reference. All modes share one durable state (plan.md) and one execution spine (plan → execute → verify → review).

## Mode dispatch

| Mode | When to use | Reference |
|---|---|---|
| `implement` | A feature to build or a bug to fix; resumes an unfinished pipeline | [implement.md](references/implement.md) |
| `review` | Review a diff or a whole codebase | [review.md](references/review.md) |
| `explore` | Onboard onto or investigate an unfamiliar codebase | [explore.md](references/explore.md) |
| `cleanup` | Find and remove dead code | [cleanup.md](references/cleanup.md) |
| `plan` | Write a plan.md only (decompose without executing) | [implement.md](references/implement.md) steps 0-4, stop before step 5 |

A named mode overrides routing. Without an explicit mode, infer from the request; ambiguous or multi-part requests start at `implement`.

## Durable state: plan.md

`.implementation/<feature-name>/plan.md` is the pipeline's single durable state. It survives session death, enables resume, and doubles as the progress tracker when no worker system is available. Format: [plan-schema.md](references/plan-schema.md). Never commit `.implementation/` files (gitignored).

## Execution spine

Every mode follows the same spine:

1. **Plan** — decompose into verifiable units with acceptance criteria (TDD where testable).
2. **Execute** — run tasks; parallel when independent worker isolation exists, serial otherwise.
3. **Verify** — acceptance criteria checked on the real artifact by the orchestrator itself, never by trusting a self-report.
4. **Review** — adversarial pass over the result before done.

## Worker capability detection

Check once per session by inspecting your own available tools - do not ask the user. Match the highest tier whose capabilities are present:

| Tier | Detection | Effect |
|---|---|---|
| **Ensemble** | A tool to spawn workers with per-branch workspace isolation, a persistent task board with dependencies, and messaging between agents (e.g. `team_create`/`team_spawn`) | Read [ensemble-mode.md](references/ensemble-mode.md) and apply it for the whole run. Plan review becomes an adversarial debate, execution becomes parallel worktree waves over the board, reviews become cross-examined panels. |
| **Generic subagents** | A spawn/subagent tool exists but no inter-agent messaging and no shared task board | Parallel-capable but weaker: embed all context in each prompt, collect results synchronously, plan.md stays the only tracker, tasks never share files (chain same-file work serially), no cross-agent contracts. Cross-examination degrades to self-adjudication: re-read each finding's evidence yourself and rule. Plan self-review stays mandatory. |
| **Solo** | Neither of the above | Execute tasks serially yourself, update plan.md statuses as the progress tracker, and self-review each plan against the review checklists before implementing. |

Interruption and resume work identically in every tier: statuses and Log in plan.md are the state. Mechanics of the ensemble tools themselves are documented by the ensemble skill in its own repository - load that skill when operating the team tools.

## Steerability

The user may invoke a named principle mid-task by saying its name. On invocation: apply it immediately and name which decision it changed.

| Principle | Rule |
|---|---|
| prove-it-works | Verify on the real artifact, not a proxy or build log. Never declare done because the build passed. |
| subtract-first | Delete the obsolete path before building the new one. Smallest change that solves it. |
| attack-the-premise | Two or more fixes failing the same gate → write down their shared premise and question it before writing a third fix. |
| idempotent-ops | Commands, lifecycle steps, and retries converge to the same end state regardless of partial prior runs. |
| separate-shared-state | Concurrent actors writing the same file/branch/key → give each its own workspace, no locks. |
| guard-the-context | Route bulk reads and fan-out to workers; keep summaries in the orchestrator, not raw payloads. |
| never-block-on-human | Reversible work → proceed, present the result, let the human course-correct after. |

## Hard rules

- plan.md is single-writer (the orchestrator, plus the architect's initial draft and debate revisions in ensemble mode).
- Every worker prompt embeds the context the worker cannot see (gitignored plan.md sections). Never reference a gitignored path in a worker prompt.
- Verify every self-report yourself. Never claim done without running the project's own check commands.
- Never silently commit or push; never open PRs unless asked. Commit at plan-defined boundaries only.
- A question only ever gets an answer. Never take action based on a question, even if the answer seems obvious or implies a fix; if the user wants action, they will say so.
- A worker that fails repeatedly is absorbed by the orchestrator (implement.md, blocked-task absorption) - caps are a handoff, not a halt.
