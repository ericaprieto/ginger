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

**Dispatch disambiguation** (confusable pairs):

| If you are torn between... | Rule |
|---|---|
| diff review vs project review | A diff range or "check my changes" → diff mode. "The whole project/codebase" → project mode. |
| trivial fast path vs full pipeline | All of: 1-2 files, no unknowns, under ~30 lines, no new patterns → trivial. Anything else → pipeline. When unsure, pipeline. |
| explore vs implement | A question whose deliverable is understanding (no code changes asked) → explore. A change to make → implement (explore becomes its planning phase). |
| cleanup alone vs cleanup inside implement | Removing unused code as the task itself → cleanup mode. Dead code spotted during a review/pipeline → a review finding, not a mode. |
| solo ad-hoc work vs the framework | One-line edits you already understand skip everything; any multi-step work gets the spine. |

When no mode fits (cross-cutting migrations, multi-repo work, open-ended investigation), design a bespoke plan with the same spine - decompose into verifiable units, record every decision in the plan.md Log with rejected alternatives - and follow it.

## Durable state: plan.md

`.implementation/<feature-name>/plan.md` is the pipeline's single durable state. It survives session death, enables resume, and doubles as the progress tracker when no worker system is available. Format: [plan-schema.md](references/plan-schema.md). Never commit `.implementation/` files (gitignored).

## Execution spine

Every mode follows the same spine:

1. **Plan** — decompose into verifiable units with acceptance criteria (TDD where testable).
2. **Execute** — run tasks; parallel when independent worker isolation exists, serial otherwise.
3. **Verify** — acceptance criteria checked on the real artifact by the orchestrator itself, never by trusting a self-report.
4. **Review** — adversarial pass over the result before done.

## Worker capability detection

**First action of any mode, before any pipeline work.** Do it in the same turn that dispatches the mode: inspect your own available tools, pick the tier, and state it ("ensemble" / "subagents" / "solo") in your first response. Never defer the check behind research or planning. Do not ask the user. If ensemble-tier tools are present, the tier is ensemble - there is no judgment call to make. Match the highest tier whose capabilities are present:

| Tier | Detection | Effect |
|---|---|---|
| **Ensemble** | A tool to spawn workers with per-branch workspace isolation, a persistent task board with dependencies, and messaging between agents (e.g. `team_create`/`team_spawn`) | Read [ensemble-mode.md](references/ensemble-mode.md) and apply it for the whole run - starting now, not after research. Plan review becomes an adversarial debate, execution becomes parallel worktree waves over the board, reviews become cross-examined panels. |
| **Generic subagents** | A spawn/subagent tool exists but no inter-agent messaging and no shared task board | Parallel-capable but weaker: embed all context in each prompt, collect results synchronously, plan.md stays the only tracker, tasks never share files (chain same-file work serially), no cross-agent contracts. Cross-examination degrades to self-adjudication: re-read each finding's evidence yourself and rule. Plan self-review stays mandatory. |
| **Solo** | Neither of the above | Execute tasks serially yourself, update plan.md statuses as the progress tracker, and self-review each plan against the review checklists before implementing. |

Interruption and resume work identically in every tier: statuses and Log in plan.md are the state. Mechanics of the ensemble tools themselves are documented by the ensemble skill in its own repository - load that skill when operating the team tools.

## Steerability

The user may invoke a named principle mid-task by saying its name. On invocation: apply it immediately and name which decision it changed.

| Principle | Rule |
|---|---|
| prove-it-works | Verify on the real artifact, not a proxy or build log. Never declare done because the build passed. |
| fix-root-cause | Trace each symptom to its root cause. Reproduce first on the same surface before fixing. |
| fix-root-cause | Trace each symptom to its root cause. Reproduce first on the same surface before fixing. |
| subtract-first | Delete the obsolete path before building the new one. Smallest change that solves it. |
| attack-the-premise | Two or more fixes failing the same gate → write down their shared premise and question it before writing a third fix. |
| idempotent-ops | Commands, lifecycle steps, and retries converge to the same end state regardless of partial prior runs. |
| separate-shared-state | Concurrent actors writing the same file/branch/key → give each its own workspace, no locks. |
| guard-the-context | Route bulk reads and fan-out to workers; keep summaries in the orchestrator, not raw payloads. |
| never-block-on-human | Reversible work → proceed, present the result, let the human course-correct after. |

**Drift phrases** - when the run drifts, name the symptom; the reply must name which decision the invocation changed:

| Symptom | Invoke |
|---|---|
| It started fixing before reproducing the failure | attack-the-premise (write the premise down; repro is a task, not a step to skip) |
| Success is claimed because the build passed | prove-it-works (show the real output, not the build log) |
| A new requirement got bolted onto the old design | subtract-first (delete the obsolete path first, then design what is left) |
| Two attempts are about to share a branch or worktree | separate-shared-state (each attempt its own workspace, no locks) |
| Two fixes failed the same gate and a third is being written | attack-the-premise (write down the shared premise and question it) |
| It is about to ask the user something it could just run | never-block-on-human (prototype it and present the result) |
| The context is filling with raw tool output | guard-the-context (route bulk reads to workers, keep summaries) |
| A vague finish condition gives the loop nothing to test | prove-it-works (state the exact command that must exit clean) |

## Hard rules

- Worker-tier detection runs before any pipeline work; ensemble tools present → ensemble mode applies from step 0, never deferred behind research or planning.
- plan.md is single-writer (the orchestrator, plus the architect's initial draft and debate revisions in ensemble mode).
- Every worker prompt embeds the context the worker cannot see (gitignored plan.md sections). Never reference a gitignored path in a worker prompt.
- Verify every self-report yourself. Never claim done without running the project's own check commands.
- Never silently commit or push; never open PRs unless asked. Commit at plan-defined boundaries only.
- A question only ever gets an answer. Never take action based on a question, even if the answer seems obvious or implies a fix; if the user wants action, they will say so.
- A worker that fails repeatedly is absorbed by the orchestrator (implement.md, blocked-task absorption) - caps are a handoff, not a halt.
