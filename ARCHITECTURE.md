# ARCHITECTURE.md

## Overview

```
opencode ──► ~/.config/opencode/skills/ginger      (symlink → skills/ginger)
                   │
                   └─ ginger/               SKILL.md (modes, spine, principles)
                        └ references/       per-mode detail, loaded on demand

worker detection ── team tools present? ──► references/ensemble-mode.md
                            │                      │
                            └─ no ──► solo execution; plan.md is the tracker

opencode-ensemble (separate repo) ── owns the team tool mechanics
```

## Technology Stack

| Layer | Tool | Purpose |
|-------|------|---------|
| Host | any coding agent; opencode v2 preferred | Loads the skill from the symlinked directory |
| Orchestration | opencode-ensemble (external, separate repo) | `team_*` tools: spawning, task board, messaging, worktree merges |
| File access | built-ins + bash | read/edit/write/glob/grep; plain git commands |
| Format | Markdown + YAML frontmatter | Skills are instructions, not code |

## File Structure

```
skills/ginger/
  SKILL.md                 # Mode dispatch, execution spine, worker detection, steering principles, hard rules
  references/
    plan-schema.md         # plan.md format spec (shared by all modes; durable state contract)
    implement.md           # implement/plan mode: resume check, trivial gate, planning, plan review, execution, report
    review.md              # review mode: diff review tracks, project review chunks, output formats
    explore.md             # explore mode: 4-phase parallel survey → module map → focus trace → report
    cleanup.md             # cleanup mode: language-agnostic dead-code removal with usage evidence
    ensemble-mode.md       # What ensemble availability changes per stage; points to the ensemble repo for tool mechanics
AGENTS.md                  # Repo rules, conventions, pitfalls
ARCHITECTURE.md            # This file
README.md                  # Layout, modes, setup
```

## Key Design Decisions

- **Markdown-only**: skills are SKILL.md files with YAML frontmatter, not code. Agents load and follow them as instructions. A prior custom-tools plugin was removed; anything an agent needs beyond built-ins comes from bash or the ensemble plugin.
- **Modes as commands**: the Agent Skills spec has no command field, so modes are first-class sections dispatched from SKILL.md. A named mode overrides routing; ambiguous requests start at implement.
- **Progressive disclosure** (agentskills.io spec): SKILL.md stays lean (metadata + dispatch + spine + principles); heavy per-mode detail lives in `references/` with relative paths, one level deep, loaded only when the mode is active.
- **Plan as a single markdown file**: plan data lives in `.implementation/<feature-name>/plan.md` (gitignored), not SQLite. The orchestrator is the single writer (the architect writes only the initial draft and its debate revisions), so a parseable markdown file replaces multi-writer state. plan.md doubles as the progress tracker in solo mode - interruption and resume need no worker system.
- **Capability-based, not tool-based**: the core speaks in capabilities (isolated workers, parallel subtasks, progress tracking). A 3-tier worker check at session start (ensemble tools / generic subagents / solo) selects how stages run; the stage sequence never changes. Harness-specific tool mechanics live in the ensemble repo; this repo owns the protocol (stages, gates, what ensemble mode adds).
- **Behavior parity**: ensemble mode preserves the original pipeline exactly - adversarial plan debate (REVISE/REBUT → ACCEPT/ESCALATE), worker board with waves and dependencies, verification-before-merge, DISCOVERIES, retry/respawn caps with absorption, reviewer cross-examination. Solo and generic-subagent modes preserve the same stage sequence with self-review and serial execution, and state what replaces each dropped gate.
- **Steering principles**: named rules the user can invoke mid-task; the agent applies the rule and names which decision it changed.

## Configuration

`skills/ginger` is symlinked to `~/.config/opencode/skills/ginger`. The opencode-ensemble skill/plugin is installed separately (its own repo); ensemble mode is detected by the presence of its team tools.
