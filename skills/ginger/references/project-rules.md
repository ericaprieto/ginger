# Project Review Skills and Rules

Discover the project's own review-related skills, rules, and tools, and fold them into the active mode. Harness-agnostic: these locations are conventions, not requirements - check each that exists, ignore the rest. If nothing is found, proceed as if this section does not exist. Never stall on absent project instructions.

## Discovery (one pass only)

| Source | What it owns | How to use |
|---|---|---|
| `AGENTS.md`, `CONTRIBUTING.md`, project `README.md` review/standards sections | Review standards, severity language, style rules to check | Add as review dimensions; keep the mode's fixed output format |
| Harness convention folders: `.claude/`, `.opencode/`, `.agents/`, `.cursor/`, `.aider*`, `.windsurf/` (commands, rules, agents, or instructions files inside them) | Deeper per-domain checklists (framework conventions, migration rules, custom review prompts) | Load and apply as an additional track or planning constraint |
| Project-provided tools (`Makefile` review/lint targets, lint configs, security scanners, CI checks) | Mechanical checks | Run them; their output feeds the findings aggregation |

1. Scan file names first - they follow each harness's own layout. Load only files whose name or frontmatter marks review/checklist/lint intent.
2. `rg -il 'review|checklist'` for project-local review docs, one pass.
3. Record what was found (and that nothing was found) in the Log or Research section.

## Usage by mode

| Mode | Use |
|---|---|
| implement (architect research) | Discovered rules become Shared constraints and coding standards in plan.md; a project review skill names where review criteria come from in Step 7 |
| review | Rules refine findings - they do not replace the output format. Findings still land in the Critical / Recommendations / Suggestions buckets; map the project's own severity names onto the closest ginger severity. Conflicts: project rules win on style and convention findings; ginger's severity calibration (concrete evidence for CRITICAL/HIGH) always applies |
| cleanup | Project lint/dead-code configs supply candidate lists and check commands alongside the survey |
| explore | Report project-specific review rules as part of the overview so later pipelines find them |
