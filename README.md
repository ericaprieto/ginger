# ginger

An agentic engineering framework: a single skill that gives a coding agent a disciplined way to build, review, explore, and clean up code - with a written plan, verifiable acceptance criteria, and adversarial review built in.

## Why

AI coding agents fail in predictable ways: they fix symptoms instead of root causes, declare victory because the build passed, bolt new requirements onto old designs, and lose context between sessions. Ginger encodes the counter-disciplines - reproduce first, verify on the real artifact, name the data shape before writing logic, keep decisions in a durable log - into a workflow the agent follows instead of having to remember.

The key idea is **one durable plan file** (`.implementation/<feature>/plan.md`): research, constraints, tasks with acceptance criteria, and every decision with its rejected alternatives live in one gitignored file. If the session dies mid-pipeline, a new session reads that file and resumes exactly where things stopped. Nothing lives only in the chat transcript.

## What it gives you

| Without ginger | With ginger |
|---|---|
| Fixes applied before the failure is reproduced | Bug plans start with a reproduce-and-diagnose phase |
| "Done" means the build passed | Every task's acceptance states a runnable check; the orchestrator reruns it |
| Big-bang changes, unreviewed | Work decomposed into small verifiable units; adversarial plan review before any code |
| Session death loses the work | The plan file is the resume point - statuses, log, decisions survive |
| Parallel agents colliding on the same files | Same-file work is chained; parallel work runs in isolated worktrees |

## Modes

You talk to the agent in one of five modes (a mode is just a named section of the skill - say it, or describe the task and let the skill route):

- **implement** - build a feature or fix a bug. Small changes are done directly; everything else goes through the pipeline: research the codebase, write the plan, review the plan adversarially, execute, verify, review the result.
- **plan** - decompose work into a plan.md without executing. Useful for scoping before committing.
- **review** - review a diff or an entire codebase and report findings with evidence. Never auto-fixes.
- **explore** - onboard onto or investigate an unfamiliar codebase; produces a structured report.
- **cleanup** - find and remove dead code, with per-item usage evidence before anything is deleted.

## How to use

1. Install the skill (see Setup below).
2. Talk to your agent in goals, not steps: "add a --json flag to this command; text output stays byte-identical; verify both". Ginger sequences the rest.
3. If the agent drifts mid-task (starts fixing before reproducing, calls a build log proof), invoke a principle by name: `prove-it-works`, `fix-root-cause`, `subtract-first`. The agent applies it and names the decision it changed.
4. Interrupted or coming back the next day? Say "resume" - the agent picks up from plan.md.

## Works with any harness

The core needs nothing but file tools and bash: plans, serial execution, self-review, resume all work with a single agent. Two optional upgrades are detected automatically:

- **opencode-ensemble** (recommended, with opencode): plan review becomes a multi-critic adversarial debate, tasks run in parallel across isolated worktrees, and reviews become panels that cross-examine each other. Install it from its own repo.
- **Generic subagent tooling** (other harnesses): parallel spawns where the harness supports them, with the plan file as the only tracker.

## Setup

Symlink the skill into your opencode config:

```bash
mkdir -p ~/.config/opencode/skills
ln -s "$PWD/skills/ginger" ~/.config/opencode/skills/ginger
```

(For other harnesses, follow its skill-loading convention - the skill directory is self-contained.)

See `ARCHITECTURE.md` for how it works internally; see `AGENTS.md` for repo rules if you want to modify the skill.
