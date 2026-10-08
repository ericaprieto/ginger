# AGENTS.md

## Doc Map

| Doc | Owns |
|-----|------|
| AGENTS.md | Repo rules, conventions |
| ARCHITECTURE.md | File structure, design decisions |
| README.md | Skill layout, setup, symlinks |
| skills/ginger/SKILL.md | The framework: modes, spine, principles, worker detection |
| skills/ginger/references/*.md | Per-mode detail; plan-schema is the plan.md format spec |

## Repo rules

1. This repo contains **only the ginger skill** (SKILL.md + references). Nothing else - do not reintroduce a plugin, tools, or a package without asking.
2. **NEVER** change the commit author or self-attribute in the commit body.
3. **NEVER** commit secrets or keys.
4. **ALWAYS** open PRs as draft.
5. **NEVER** add comments to code unless asked.
6. Implementation directory path: always `.implementation/<feature-name>/` (gitignored).
7. Avoid em dashes in PR descriptions.
8. This repo is the source of truth for the skill. `skills/` is symlinked to `~/.config/opencode/skills`. Always edit files in this repo - never through the symlink target.

## Skill authoring style

- SKILL.md stays lean (progressive disclosure): metadata triggers routing; heavy detail lives in `references/` with relative paths, one level deep.
- Dense tables/lists, no prose paragraphs. No em dashes in new text.
- Tool-agnostic: reference opencode built-ins and bash; ensemble specifics live in the opencode-ensemble skill (its own repo); this repo describes what ensemble mode changes, not how to operate its tools.
