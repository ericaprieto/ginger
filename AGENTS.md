# AGENTS.md

## Doc Map

| Doc | Owns |
|-----|------|
| AGENTS.md | Repo rules, conventions, pitfalls |
| ARCHITECTURE.md | File structure, design decisions |
| README.md | Skill layout, setup, modes, symlinks |
| skills/ginger/SKILL.md | The framework: mode dispatch, execution spine, worker detection, steering principles, hard rules |
| skills/ginger/references/*.md | Per-mode detail; `plan-schema.md` is the plan.md format spec |

## Build & Check Commands

| Action | Command |
|--------|---------|
| Link/path validation | `grep -rn` cross-reference spot checks |
| Frontmatter validation | compare `SKILL.md` frontmatter `name` against the directory name |

This repo is markdown-only. There is no build, test, or typecheck step.

## Hard Rules

1. This repo contains **only the ginger skill** (`skills/ginger/`). Never reintroduce a plugin, custom tools, or a package manifest without asking.
2. Skills must stay **tool-agnostic**: reference opencode built-ins and bash; ensemble tool mechanics live in the opencode-ensemble skill (its own repo). Never reference tools that do not exist in a default opencode install.
3. **NEVER** change the commit author or self-attribute in the commit body.
4. **NEVER** commit secrets or keys.
5. **ALWAYS** open PRs as draft.
6. **NEVER** add comments to code unless asked.
7. Implementation directory path: always `.implementation/<feature-name>/`. Never `.<feature-name>-implementation/` or any variant.
8. This repo is the source of truth for the skill. `skills/ginger` is symlinked to `~/.config/opencode/skills/ginger`. Always edit files in this repo - never through the symlink target.

## Skill Authoring Style

- SKILL.md stays lean (progressive disclosure): metadata triggers routing; heavy detail lives in `references/` with relative paths, one level deep. The skill `name` must match the directory name.
- Dense tables/lists, no prose paragraphs. No em dashes in new text.
- Every markdown link target must resolve to a real file.

## Known Pitfalls

- **Teammates cannot see gitignored files**: plan.md lives under `.implementation/` (gitignored); everything a builder needs must be embedded into its prompt by the orchestrator. Symptom: a builder in a worktree reports "file not found" for a plan.md path.
- **A shut-down teammate's NAME stays registered**: respawn under a fresh name; `team_spawn` with the same name returns "already exists". Symptom: spawn rejection after a shutdown.
- **`team_view navigate:true` hijacks the user's client**: only when explicitly asked.
- **Never claim done without verification**: run the project's own check commands before reporting success. Symptom: "build passed" used as proof of behavior.
- **Skill directories must match frontmatter names**: a renamed folder without a frontmatter update registers the skill twice or not at all. Symptom: two entries for one skill in the harness skill list.
