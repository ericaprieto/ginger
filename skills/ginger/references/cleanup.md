# Cleanup Mode

Find and remove dead code with search tools alone, then delete only what survives per-item verification. Language-agnostic; no external analysis tools. When uncertain, flag and skip: a false "unused" is worse than leftover code.

**Read-before-delete discipline: never delete an item without recording its usage evidence first** (where it is defined, what references it, which dynamic-use caveats were checked and ruled out).

> **[ensemble]** Steps 1-2 (survey + verify) parallelize well: one worker per candidate group, each returning per-item evidence tables; the orchestrator adjudicates ambiguity and performs removals. Worker rules: [ensemble-mode.md](ensemble-mode.md).

## 1. Survey (inventory candidates)

Identify the project's check commands first (`README.md`, `AGENTS.md`, `Makefile`, `package.json` scripts, `pyproject.toml`, CI config): tests, typecheck, build, lint.

Inventory three kinds of candidates:

| Candidate type | How to find |
|---|---|
| Exported functions/types/classes/constants | `rg` definition patterns per language (table below) |
| Files never imported | `rg -l '<filename-without-extension>'` across the repo; zero import/require hits |
| Config entries never read | Extract declared keys (`package.json` scripts/bin, `Makefile` targets, entry points, route tables, env var names) and search for their consumption |

Definition pattern examples:

| Language family | Example definition patterns |
|---|---|
| JS/TS | `export (function\|const\|class\|interface\|type)`, `module.exports` |
| Python | `^def `, `^class ` |
| Go | `^func `, `^type ` |
| Rust | `pub fn`, `pub struct`, `pub enum`, `pub trait` |
| Java/C# | `public .*(class\|interface\|void\|[A-Z]\w+)` |

Skip up front: generated files, vendored code, entry points (`main`, `index`, CLI handlers), and anything referenced from config files.

## 2. Verify (usage evidence per candidate)

For each candidate, count real usages across the repo, word-boundary aware, excluding the file where it is defined:

```sh
rg -w '<symbol>' --glob '!<defining-file>'
```

Distinguish definition-only matches from usage matches:

- Definition-only: the declaration line itself, re-export barrel lines (`export * from`, `from x import y` re-exports)
- Usage: call sites, references, assignments, passing as a value

Classify each match; only real usage matches count as consumers. Narrow noise with `rg -w` and call-site patterns (`symbol(`, `: symbol`, `symbol =`).

Before flagging anything as dead, check for dynamic or indirect use:

| Risk | Check |
|---|---|
| String-based access | `rg` the symbol name inside quotes; route maps, DI registries, command tables |
| Reflection / metaprogramming | framework config, attributes/tags/annotations referencing the name |
| Framework conventions | routes, lifecycle hooks, handlers, migrations wired by naming convention or declared in configs |
| External consumers | published package, `package.json` `exports`/`main`, or other repos importing the export |
| Test-only usage | consumers only under test dirs: report as test-only, do not silently remove or keep |
| Docs references | `rg` in README, docs/, examples; documented API stays |

If any check is ambiguous, flag the item and skip it.

## 3. Remove

Use `read` then `edit` for each confirmed item:

- Delete the declaration plus any blank line it leaves behind
- If the symbol was the only import from a module, remove the whole import line
- If it was one of several imports, remove just that named import
- If an entire file is dead, delete it and clean up references to its path

## 4. Verify again

Run the project's own check commands (tests, typecheck, lint):

1. New failures pointing at a removed item → restore it: `git checkout -- <file>` for uncommitted edits, or re-add the declaration; move the item to the skipped list
2. Pre-existing failures (present before removal) are not yours to fix here; note them in the report

## 5. Report

Summarize with per-item evidence:

```
Removed: N items (E functions, T types, F files, C config entries) across F files

Per item:
| Item | Defined at | Referenced by | Caveats ruled out | Status |
|------|-----------|---------------|-------------------|--------|
| {symbol} | {file:line} | none found (rg -w across repo) | string access, reflection, external, docs | removed |
| {symbol} | {file:line} | tests only | reflection, external | skipped (test-only) |

Verification: {check commands run, results}
Skipped: S items (flagged as ambiguous, with reason)
```

## Gotchas

- **Entry points are never "unused"**: main files, plugin registrations, CLI routes, and anything referenced from config files.
- **Tests count as consumers**: a symbol used only by tests is "test-only", not dead; report it explicitly.
- **Public APIs of published packages**: anything exported for external consumers stays.
- **Languages with implicit usage** (serialization, ORMs, template engines): grep cannot see these consumers; be conservative.
- **Never trust a single search**: name collisions mean a hit may be a different symbol; verify the hit is the same type/scope before counting it.
- **Batch removals in small groups**: remove, re-check, then continue, so a breakage maps to one small change.
