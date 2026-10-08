# Plan Format

All modes produce and consume a single markdown file: `plan.md`. It is the only durable state of a pipeline - the worker board (when a worker system is active) is a disposable runtime view, but `plan.md` survives session death and enables resume.

## File location

`.implementation/<feature-name>/plan.md`

**NEVER commit `.implementation/` files.** They are ephemeral agent workspace. If the repo lacks a `.gitignore` entry for `.implementation/`, add one.

Because the file is gitignored, isolated workers (worktree workers in ensemble mode) cannot see it. All context they need (research, patterns, constraints, acceptance criteria) is **embedded into their prompt** by the orchestrator at assembly time (see Prompt assembly). Never reference plan.md by path in a worker prompt.

## Single-writer rule

Only the orchestrator edits plan.md - with two exceptions in ensemble mode: the architect worker writes the initial draft and its debate revisions, and critics read it (never write) during the debate as the orchestrator directs. The orchestrator makes every other edit, including refinement, debate deadlock settlement, and mid-pipeline revisions. Workers never write plan.md; worktree workers cannot even see it. Workers report results to the orchestrator, and the orchestrator records status changes and log lines.

## Mid-pipeline revisions

The orchestrator may revise not-yet-done tasks (pending or in_progress) when discoveries, retries, conflicts, or review findings invalidate a task's assumptions. Every revision gets a Log line: `[YYYY-MM-DD HH:MM] T{n} revised: <reason>`. Revised tasks keep their task IDs and statuses - a revision changes content, not identity. Downstream workers receive the updated context automatically: the orchestrator assembles each prompt from the current plan.md at spawn time. A worker already running a revised task is messaged the revised criteria immediately (ensemble mode) - never left working from stale context.

Task-result messages may carry a **DISCOVERIES** section - findings that invalidate plan assumptions, in the reporting worker's own task or in other tasks. The orchestrator records these in the Log.

## Format

Strict enough to parse by regex. The orchestrator validates the file against this spec before every spawn and fixes any deviation immediately.

```markdown
# Plan: <feature-name>

## Meta
- feature: <feature-name>
- goal: <one sentence>
- start_hash: <git commit hash recorded before any planning edits>
- status: in_progress
- team: <worker team name, or none>

## Research
<Findings with file:line references. Keep under ~150 lines - this section
is embedded into every worker prompt.>

### <Topic>
- **Location**: `src/auth.ts:42-67`
- **What it does**: <one sentence>
- **Key detail**: <specific value or behavior>

## Open Questions
- <question> — <resolution when decided>

## Shared
### Patterns
- <pattern to follow>
### Constraints
- <hard constraint>
### Coding standards
- <standard name> — <path or reference>
### Relevant files
- `<path>` — <what it is>

## Tasks
### T1 — <task name>
- status: pending
- phase: p1 <Phase name>
- role: implement
- files: src/a.ts, src/b.ts
- depends: none
- commit_boundary: no
- description: <what to do — one paragraph max>
- acceptance:
  - [ ] <criterion 1>
  - [ ] <criterion 2>
- result: <filled by orchestrator after verification>

### T2 — <task name>
- status: pending
- phase: p2 <Phase name>
- role: implement
- files: src/c.ts
- depends: T1
- commit_boundary: yes
- description: <what to do>
- acceptance:
  - [ ] <criterion>

## Log
- [YYYY-MM-DD HH:MM] T1 spawned to worker <name> (board <id>)
- [YYYY-MM-DD HH:MM] T1 done: <one-line outcome>
- [YYYY-MM-DD HH:MM] Critic MUST RETHINK: <finding> — revised T4
- [YYYY-MM-DD HH:MM] Contract agreed between workers: <decision>
- [YYYY-MM-DD HH:MM] Decision: <what was chosen> — <why> — rejected: <alternative(s) considered>
```

## Decision entries

Significant decisions (design choice, contract between workers, mid-pipeline revision, premise rejection after failed fixes) get a `Decision:` log line with the rationale and the rejected alternatives. A resume session reads these instead of re-deriving or re-litigating them; recording what was rejected is as important as recording what was chosen. Routine events (spawn, done, status change) stay plain one-liners.

## Field reference

| Section | Field | Required | Notes |
|---|---|---|---|
| Meta | feature | yes | kebab-case, max 5 words |
| Meta | goal | yes | one sentence |
| Meta | start_hash | yes | scopes code review to pipeline commits: `git diff <start_hash>...HEAD` |
| Meta | status | yes | `in_progress` or `done` or `blocked` (pipeline-level, not task-level) |
| Meta | team | yes | current worker team name; `none` when running solo |
| Tasks | status | yes | `pending`, `in_progress`, `done`, `failed`, `blocked` |
| Tasks | phase | no | label like `p1 Diagnose`; grouping + commit boundary marker |
| Tasks | role | yes | `research`, `implement`, `review` (default `implement` if omitted) |
| Tasks | files | yes | comma-separated paths, or `none` for pure research tasks |
| Tasks | depends | yes | comma-separated task IDs, or `none` |
| Tasks | commit_boundary | no | `yes` on the last task of each phase group; the orchestrator commits after verifying it |
| Tasks | description | yes | what to do; continue on the next line if needed (indented) |
| Tasks | acceptance | yes | checkbox list; every task needs at least one criterion; where testable, state the runnable check (command or test name), not prose |
| Tasks | result | no | one-line outcome, filled by the orchestrator after verifying |
| Tasks | blockers | no | why the task is blocked/failed |

## Task status lifecycle

```
pending -> in_progress -> done
                        -> failed (blockers line required)
                        -> blocked (blockers line required)
blocked -> in_progress (retry)
failed -> in_progress (retry)
done -> terminal
```

The orchestrator transitions statuses as it assigns and verifies - never the workers.

## Task roles

| Role | May write code? | Used for |
|---|---|---|
| `research` | No | Plan-task investigation, reproduction, diagnosis, verification |
| `implement` | Yes (listed files only) | Creating/modifying code, fixing bugs |
| `review` | No | Adversarial review of plans or code |

Role contracts (one-liners the orchestrator prepends to every assembled prompt):

- **research**: "You are a code researcher. Dig deep into the codebase, trace call chains, and report findings with file:line references. Trust code, not comments. Do not write or edit code. Return findings only."
- **implement**: "You are an implementor. Follow the spec precisely. Write code only in the listed files. Run tests. No improvisation -- if the plan is unclear, report a blocker instead of guessing."
- **review**: "You are an adversarial reviewer. Find the load-bearing assumption that must be true for this to work. Provide three heresies (Lazy: do dramatically less, Weird: parallel-universe solution, Nuclear: delete the problem), one uncomfortable question, and your actual take. Do not write code."

## TDD: Red-Green-Refactor

Implementation tasks (`role: implement`) must carry acceptance criteria with explicit TDD sub-items:

1. Test exists for the behavior
2. Test fails before the implementation (red)
3. Test passes after the implementation (green)
4. Test still passes after any refactor

For bug-fix tasks, the first criterion is "test reproduces the bug" instead of the generic "test fails before."

**Exception:** tasks with no testable behavior (config files, type definitions, scaffolding, docs) should say "no testable behavior -- verify by inspection" instead of forcing a fake test.

Where a criterion has a cheap runnable check, state the exact command (or test name) in the criterion so the orchestrator reruns the same check the worker did - never trust a build log as proof.

The orchestrator verifies each acceptance criterion itself after the worker reports - never trust self-reports.

## Phase labels

Phase is a label on tasks, not a status-bearing entity. Phases exist for:

- **Planning clarity**: the architect groups tasks into phases during decomposition
- **Commit boundaries**: `commit_boundary: yes` on the last task of each phase group
- **Progress overview**: the orchestrator reads the file and groups by phase

Advisory limit: if any phase group exceeds 8 tasks, consider splitting into smaller phases or breaking large tasks up. Intentionally large groups are fine - this is advisory, not enforced.

## ID conventions

Task IDs are `T` + sequential number (`T1`, `T2`, `T3`), assigned by the architect and never reused. Review-fix tasks appended mid-pipeline continue the sequence.

## Task sizing

A task that touches one large file (1000+ lines) with many scattered edits will fail in a single pass. Split by logical change group, not by file:

| Edit density | Guidance |
|---|---|
| 1-5 edits in a file | One task is fine |
| 6-15 edits in a file | One task, but list the specific sections/functions in the description |
| 16+ edits in a file | Split into multiple tasks, each scoped to a section or function group |

## Prompt assembly

The orchestrator assembles every worker prompt by hand from the current plan.md.

**Embed the Research section verbatim in every worker prompt.** Isolated workers cannot read the gitignored plan.md, so copy the full section, never a summary. Embed the Shared subsections the task needs (patterns, constraints, coding standards, relevant-file annotations). Never reference plan.md by path in a worker prompt.

Assembled prompt contents, in order:

1. Role contract (research/implement/review, one-liner from Role contracts)
2. Task name, feature name, phase label
3. **Research section embedded verbatim** (isolated workers cannot read the gitignored file)
4. Task description (goal)
5. Files with relevant-file annotations
6. Patterns, constraints, coding standards from Shared
7. Acceptance criteria verbatim
8. Rules verbatim (only listed files, no unrequested features, report as a task result, message the orchestrator when blocked)

Assemble each prompt at spawn time from the file as it then stands; this is how revised content reaches downstream workers after mid-pipeline revisions.

The orchestrator validates the plan itself before every spawn: required Meta fields per the Field reference, every task carrying status, role, files, depends, description, acceptance, every `depends:` referencing an existing task ID, and statuses following the lifecycle table. Fix deviations before spawning: while the architect is alive (ensemble mode), message it the specific problems to fix; after the debate, fix the format yourself.

## Resuming

A pipeline is resumable if and only if plan.md exists on disk with `Meta.status: in_progress` and pending or blocked tasks. The orchestrator reads the file, rebuilds the worker board for unfinished tasks (ensemble mode) or continues executing them itself (solo), and resumes. The Log section tells the new session what already happened.

## Writing style rules

Plan.md is read by agents and parsed by regex - write it clean:

1. Plain English prose in all fields.
2. No markdown tables inside task fields.
3. No nested quotes in descriptions - describe values in prose.
4. Keep Research under ~150 lines; it is embedded in every worker prompt.
