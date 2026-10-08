# Implement Mode

Implement a feature or fix a bug from a description. Resumes unfinished pipelines. No branch or PR creation unless asked.

Worker mode (ensemble vs solo) changes only HOW stages run - see the `ensemble-mode.md` hooks below. The stage sequence never changes.

## Step 0 - Resume check

Before anything else, look for an unfinished pipeline:

1. Find `.implementation/*/plan.md` files.
2. If one exists with `Meta.status: in_progress`:
   - If it matches this request (or the user asked to resume), read it and jump to **Step 6 (Execute)** - rebuild the ready set from pending tasks and continue. The Log section tells you what already happened.
   - If multiple unfinished plans exist, ask the user which one.
3. No unfinished plan → continue.

## Step 1 - Read project docs

Read each file that exists (skip missing ones): `AGENTS.md`, `ARCHITECTURE.md`, `DESIGN.md`, `README.md`.

## Step 2 - Assess: trivial fast path or full pipeline?

**Trivial fast path** - use when ALL of the following are true:
- 1-2 files affected
- No unknowns
- Likely under 30 lines of change
- No new patterns, no architectural decisions

For trivial changes: make the change directly, run tests, commit, report. No plan.md, no ceremony. Done.

**Full pipeline** - everything else. Continue.

## Step 3 - Set up

Record the current HEAD: `git rev-parse HEAD` - this becomes `start_hash` in plan.md.

> **[ensemble]** `team_create` with the feature name; architect worker gets no worktree isolation because plan.md is gitignored - see [ensemble-mode.md](ensemble-mode.md) Team lifecycle.

## Step 4 - Plan (research + decompose)

> **[ensemble]** Spawn an architect worker with the plan-research contract below as its prompt body; the architect writes the plan.md draft. Spawn mechanics: [ensemble-mode.md](ensemble-mode.md).
> **[solo]** Research the codebase yourself and write the plan.md draft directly.

Research protocol (whoever executes it - architect worker or the orchestrator):

1. Read the plan format spec: [plan-schema.md](plan-schema.md). The plan must parse cleanly under it.
2. Record the execution start hash (Step 3).
3. Understand the request: restate the goal, the finish condition, and constraints in one sentence each; identify the affected area.
4. Research affected areas: read the relevant files, trace call chains, find where the change lands. Verify claims against actual code - trust code, not comments. Report findings with file:line references.
5. Identify patterns and constraints: existing patterns to follow, hard constraints, coding standards (project docs), relevant files.
6. Decompose into tasks: a task is one atomic unit of work. If a task contains "and" or spans more than one file area, split it. Order dependencies (what must exist before what). Assign each task a role (`research` / `implement` / `review`), files, `depends:`, and a phase label.
7. Acceptance criteria with TDD sub-items per [plan-schema.md](plan-schema.md) (red/green/refactor; state the runnable check).
8. Write `.implementation/<name>/plan.md` following the template in plan-schema.md exactly. Meta.status `in_progress`, Meta.team `none` (the orchestrator fills it in ensemble mode).
9. Report: feature name, plan path, task count by phase, open questions, which tasks can run in parallel first.

Spawn-failure handling (ensemble): if the architect wedges, lacks write capability, or idles, shut it down force and respawn under a fresh name; only if the replacement still cannot write plan.md, surface that as a blocker - never substitute a chat-message breakdown.

**Validate before proceeding:** read plan.md and validate it against plan-schema.md (required Meta fields, task fields, statuses, dependencies). If it deviates, send the specific problems back to the architect (ensemble) or fix them (solo). Do not proceed on an invalid plan. Advisory refinement: if any phase group exceeds 8 tasks, consider splitting; do not blindly split intentional groups.

## Step 5 - Plan review

The plan review always happens. Worker availability decides the form:

> **[ensemble]** Adversarial critic debate: one Simplifier critic (plus a Risk Auditor when the plan touches more than 5 files or is architectural). Findings cycle through REVISE/REBUT to the architect, then ACCEPT/ESCALATE back, 3-round cap, the orchestrator adjudicates escalations. Full protocol: [ensemble-mode.md](ensemble-mode.md) Plan debate.
> **[solo]** Self-review pass. Apply BOTH critic protocols below to your own plan, honestly, before writing any code:

**Simplifier pass** - what would dramatically less look like?
1. Understand (do not strawman) - acknowledge what is good.
2. Find the load-bearing assumption - what MUST be true for this to work.
3. Three heresies: Lazy (solve by doing dramatically less), Weird (solution from a parallel universe), Nuclear (delete the problem entirely).
4. The uncomfortable question - one question that might change everything.
5. Spot-check 2-3 load-bearing research claims from the Research section against the actual code - open each file:line and confirm it says what the plan claims.

**Risk Auditor pass** - what breaks during execution?
1. Load-bearing assumptions that are unverified.
2. Hidden coupling the plan touches without naming: shared state, callers, importers, ordering, migrations.
3. Unverifiable acceptance criteria - criteria no test, command, or inspection could prove. Rewrite them.
4. The uncomfortable question.

Classify each finding: **MUST RETHINK** (the flaw will likely cause implementation failure - revise the plan) or **CONSIDER** (a tradeoff worth knowing - log it). Log every finding and resolution in plan.md's Log. Then validate the format once more and proceed.

## Step 6 - Execute

Run tasks in dependency order. Compute the ready set: pending tasks whose `depends:` are all `done`.

> **[ensemble]** Derive the worker board, spawn one builder per ready task in parallel (worktree isolation), verify acceptance yourself on merge, mark done. Full protocol including stall handling, retry caps, hook-blocked commits, worktree propagation, absorption, adaptation checks, and builder contracts: [ensemble-mode.md](ensemble-mode.md) Execution.

**[solo]** For each ready task (repeat until none remain):

1. Mark the task `in_progress` + Log line.
2. Assemble the working context from the current plan.md per plan-schema.md Prompt assembly (research verbatim, description, files, shared constraints, acceptance).
3. Implement it yourself: write code only in the listed files, follow the spec precisely, run tests. No improvisation - if the plan is unclear, mark the task `blocked` with a `blockers:` line instead of guessing.
4. Verify every acceptance criterion yourself on the real artifact - run the checks, read the files, inspect the diff. Never trust a build log as proof.
5. Met → mark `done` + `result:` line. Not met → fix and re-verify; after two honest failures, mark `failed` with reasons in the Log, run the attack-the-premise check, then absorb: revise or simplify the task in plan.md and retry once more.
6. If `commit_boundary: yes`, commit with a descriptive message.
7. Adaptation check after every failure or discovery: re-examine not-yet-done tasks; if any rests on invalidated assumptions, revise it in plan.md (description, files, acceptance) + Log line `T{n} revised: <reason>`.

Blocked-task absorption (both modes): exhausted caps are a handoff, not a halt. The orchestrator implements the blocked task itself in its own tree (solo path), mining the failed attempts' Log entries as context. Only if absorption also fails after an honest attempt does the task stay `blocked` - the one legitimate blocked state, reported in Step 8.

## Step 7 - Code review

When all tasks are done:

1. Read `start_hash` from plan.md Meta. Compute the pipeline's diff: `git log --oneline <start_hash>...HEAD` and `git diff <start_hash>...HEAD`.
2. Run the `review` mode (review.md) in **diff mode**, scoped to `<start_hash>...HEAD` - do not let it auto-detect the base.
3. For every confirmed top-bucket finding (CRITICAL, MUST FIX): append a task to plan.md (next free T number, `phase: p-review Code review fixes`, `role: implement`, acceptance criteria from the finding) and run Step 6 for the fix tasks.
4. Mid-bucket findings (SHOULD FIX, HIGH) and lower: report to the user with the disposition - do not auto-fix; fix them only if the user confirms.

## Step 8 - Report and cleanup

After review and any fix waves:

1. Set `Meta.status: done` in plan.md (or `blocked` if tasks remain blocked).
2. Report: files changed, test results, task summary (statuses from plan.md), CONSIDER findings, blockers encountered.
3. Clean up the worker team if one is active ([ensemble-mode.md](ensemble-mode.md) Team lifecycle).

## Hard rules

- **plan.md is single-writer**: the orchestrator and (during the debate, ensemble mode) the architect. Teammates never write plan.md otherwise.
- **Hand-assemble every worker prompt** from the current plan.md per plan-schema.md, embedding Research and the relevant Shared sections verbatim. Validate the plan before each spawn.
- **Verify acceptance yourself** - never trust a worker's self-report.
- **Worktree workers cannot see plan.md** (gitignored). All their context flows through the embedded prompt. Never reference plan.md by path in a worker prompt.
- **Commit at commit boundaries only**, after verification. No branch creation, no PR creation.
- **Never commit `.implementation/` files.**
- **Blocked tasks isolate**: one blocked task halts only its dependent subgraph. Keep other work running.
- **Adjudicate worker conflicts yourself** - weigh both sides, rule, and log the adjudication with reasoning in plan.md's Log. Only a genuine ambiguity about the user's goal stops the pipeline to ask.
