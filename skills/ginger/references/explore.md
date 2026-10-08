# Explore Mode

Systematically explore a medium-to-large codebase using parallel workers. Produces a structured report: project overview, module map, dependency hotspots, key abstractions, and data flows. For small codebases (under ~20 files), explore manually with `ls`/`find`/`grep` - worker overhead is not worth it.

Given a `{CODEBASE PATH}` (defaults to the current working directory) and an optional `{FOCUS}` (a question, area, or feature to investigate):

## Phase structure

Parallel within a phase, sequential across phases: Phase 2 needs Phase 1's results to know which modules to explore.

### Phase 1 - Survey (parallel workers)

Spawn three workers simultaneously, each returning a short summary (under 300 words):

**Worker 1 - Project overview**
1. Read README.md if it exists
2. Read AGENTS.md, ARCHITECTURE.md, or DESIGN.md if they exist
3. Estimate scale and language mix: `find . -type f -name '*.<ext>' | wc -l` per language, `wc -l` totals

Return: project name and purpose (1-2 sentences); tech stack (language, framework, key dependencies); entry points; test runner and scripts; LOC breakdown by language.

**Worker 2 - Directory structure**
1. `tree -L 3` or `find . -maxdepth 3 -type d`
2. For each top-level source directory (not config, not deps), one level deeper

Return: purpose of each top-level directory (1 line each); which directories are source vs config vs docs vs tests; any monorepo/workspace structure detected.

**Worker 3 - Dependency hotspots**
1. Scan imports with `rg` (adapt patterns per language: `^import `, `^from `, `#include`, `use `, `require`); build an import graph (file → imported files)
2. Compute: most-imported files (hubs), files with the most imports (complexity), circular dependencies, files not imported anywhere

Return: top 10 hubs with importer counts; top 10 complexity files; circular deps found; orphan files (note: dynamic loading makes static analysis miss edges - say so when the project uses it).

**After all three report**, synthesize. Shut down the survey workers.

### Phase 2 - Map key modules (parallel workers)

Identify the 3-5 most important source directories. Spawn one worker per directory, all in parallel. Each:

1. `tree <module> -L 2` (or `find`)
2. Build the module's internal import graph with `rg`
3. For the 3-5 most connected files, inspect their structure (grep for declarations, `sed`/`head` for section boundaries) - never read whole large files
4. If an outline reveals important types/interfaces/classes, read only those sections (with offset/limit)

Return: module purpose (1-2 sentences); key files and what each does (1 line each); main abstractions and their relationships; how the module connects to the rest (imports from / exports to); notable patterns (factory, middleware chain, plugin system).

**After all report**, shut down the module mappers.

### Phase 3 - Focused investigation (only if `{FOCUS}` was provided)

Trace the focus area with 1-2 workers:

1. Find definitions related to the focus (`rg` definition patterns)
2. Trace how the files connect (import graph from Phase 2)
3. Find usages and call sites (`rg`)
4. Read the key files (structure first, then specific sections)

Return: where the relevant code lives (files and line ranges); data/control flow for the area; key types and functions; non-obvious connections or gotchas.

### Phase 4 - Write report

Output directly to the user:

```markdown
# Codebase Exploration: {project name}

## Overview
{1 paragraph: what this project is, its tech stack, its scale}

## Architecture
{Module map: what each major directory/module does and how they connect}

## Key Files
{Table: file path | role | fan-in | fan-out}
{~10 most structurally important files with a 1-line description each}

## Dependency Structure
{Hotspots, circular deps, notable patterns}

## Abstractions
{Key types, interfaces, and patterns that define the codebase's vocabulary}

## {FOCUS area} (if provided)
{Findings from Phase 3}

## Entry Points for Further Exploration
{3-5 suggested starting points with specific file paths and what you would learn from each}
```

## Worker rules

- All explorers are read-only (no worktree isolation needed); spawned one at a time by the lead. Workers are leaf-only - never spawn subagents or delegate.
- **Use the right tool for the job:** `rg` for content search, `find`/`tree` for structure, import scanning for dependency analysis, structural inspection before full reads.
- **Stay within context budgets.** Each explorer returns under 300 words; the lead synthesizes.
- **Do not read every file.** Structure first, then targeted sections.
- If a worker goes quiet, one status ping; still silent → shut it down force and respawn once with the same prompt.
