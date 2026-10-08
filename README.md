# agent-toolkit

The ginger skill: an agentic engineering framework for coding agents - implement features and fixes, review code, explore codebases, and clean dead code through a plan-driven pipeline with durable state. Harness-agnostic; opencode is the primary host, and parallel work is supercharged by opencode-ensemble when available.

## Layout

| Path | Purpose |
|---|---|
| `skills/ginger/SKILL.md` | The framework: mode dispatch, execution spine, worker detection, steering principles |
| `skills/ginger/references/` | Per-mode detail and the plan.md format spec |
| `AGENTS.md`, `ARCHITECTURE.md` | Repo rules and design decisions |

## Modes

| Mode | When to use |
|---|---|
| `implement` | Implement a feature or fix a bug: trivial fast path, or a full plan-driven pipeline. Resumes an unfinished `plan.md` |
| `plan` | Write a plan.md only (decompose without executing) |
| `review` | Review a diff or a whole codebase |
| `explore` | Explore an unfamiliar codebase |
| `cleanup` | Find and remove dead code (language-agnostic) |

## Setup

Symlink the skill into the opencode config directory. Setup is machine-local and not stored in git, so repeat it on each machine. The commands below assume a fresh configuration directory and no existing links; resolve any existing files or symlinks before running them.

| In this repo | Symlink target |
|---|---|
| `skills/` | `~/.config/opencode/skills` |

```bash
mkdir -p ~/.config/opencode
ln -s "$PWD/skills" ~/.config/opencode/skills
```

Install the [opencode-ensemble](https://codeberg.org/visualglitch91/opencode-ensemble) skill/plugin separately for the `team_*` tools. Without it, the framework runs single-agent: plan.md is the progress tracker, tasks execute serially, and reviews are self-applied checklists.

Always edit the skill in this repo, never through the symlink target.

## Ensemble supercharge

When the `team_*` tools are detected, the framework switches to ensemble mode (see `skills/ginger/references/ensemble-mode.md`):

- Plan review becomes an adversarial critic debate (REVISE/REBUT → ACCEPT/ESCALATE).
- Tasks become a parallel board executed in worktree-isolated waves.
- Reviews become reviewer panels with a cross-examination round.

Without ensemble, the same stages run solo; behavior differs, quality gates do not.

## Models

Models and providers are configured per machine in `~/.config/opencode/opencode.jsonc`. Keep API keys and tokens there only, never in this repo.
