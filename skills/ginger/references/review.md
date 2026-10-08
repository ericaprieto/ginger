# Review Mode

Comprehensive code review against project rules. Two modes: **diff review** (default, reviews git changes) and **project review** (the entire codebase in lexical chunks). "Review the project" / "the whole codebase" triggers project mode; everything else triggers diff mode.

> **[ensemble]** Reviews run as parallel reviewer teammates (no worktree isolation - read-only) into the active pipeline team or a scratch team. Reviewer panels and the cross-examination round: [ensemble-mode.md](ensemble-mode.md) Review.
> **[solo]** Run the same tracks yourself in sequence, applying each reviewer prompt's checklist to the diff or chunk. Report in the same output formats.

Project-specific review skills and rules (harness convention folders, project docs, review tools) are discovered once and folded into the reviewer prompts as an additional track: [project-rules.md](project-rules.md).

Workers (teammates) are leaf-only: never spawn subagents or delegate work. Only the lead manages and spawns.

## Diff review

### 1. Determine what to review

If a diff range was passed (e.g. the implement pipeline passes `<start_hash>...HEAD`), use it - do not auto-detect.

Otherwise:

```sh
git rev-parse --abbrev-ref HEAD@{upstream} 2>/dev/null | sed 's/[^/]*\///' || echo "main"
```

- Uncommitted changes present: `git diff HEAD`
- Otherwise: `git diff <base>...HEAD`

Also grab: `git log --oneline <base>...HEAD` and any relevant `AGENTS.md`.

### 2. Run the review tracks

**Track A - Dead code.** For each changed file: list symbols it defines/exports (`rg` for export/definition keyword patterns matching the language: `export`, `module.exports`, `pub fn`, `func`, `def`, `class`); count real usages repo-wide with `rg` excluding the definition site; check dynamic references (string-based access, reflection, test fixtures, config-driven dispatch, external consumers) before flagging. Only report symbols with zero usages outside their own file, scoped to the changed files. Severity: LOW unless a changed file exports something unused and no other changed file references it - then MEDIUM.

**Track B - Specialist.** First-principles review of the changes:

```
Assess: is the problem clearly solved? Are there correctness, security, or
design concerns? Return findings as MUST FIX, SHOULD FIX, or CONSIDER,
with reasoning.
```

**Track C - Security audit (conditional).** If the diff touches auth, permissions, input parsing, network handling, secrets, file paths, or shell/subprocess invocation. Reject-biased: assume unsafe until shown otherwise. Look for: injection, path traversal, secret exposure, missing authz checks, unsafe deserialization, SSRF, command injection. Findings as CRITICAL / HIGH / MEDIUM with file:line evidence; nothing found → say so explicitly. Skip for diffs that clearly avoid security surfaces.

### 3. Aggregate and report

Deduplicate: same file:line flagged by multiple reviewers → keep the highest severity, note the sources. Disputed file:line assessments → reviewers converge; the lead arbitrates.

Output:

```
## Critical (must fix)
<CRITICAL + MUST FIX findings, file:line, explanation>

## Recommendations (should fix)
<HIGH + SHOULD FIX findings>

## Suggestions (consider)
<MEDIUM/LOW + CONSIDER findings>

## Assessment
<Specialist's verdict: problem clarity, solution quality, tradeoffs, missing pieces>

---
Scope: <worktree changes | N commits ahead of base>
Files: <count> | Reviewers: <list>
```

## Project review

### 1. Survey and partition (the orchestrator does this)

0. Project review skills and rules: run the discovery check ([project-rules.md](project-rules.md)) so its findings shape the reviewer prompts.
1. File counts and LOC by language: `find . -name '*.ext' -not -path '*/node_modules/*' | xargs wc -l` (adapt per language).
2. Directory structure: `tree -L 2` or `find . -maxdepth 2 -type d`.
3. Source file list: include source code files; exclude `node_modules`, `dist`, `build`, `.next`, `coverage`, `vendor`, `.git`, lock files, generated and minified files, config files under ~20 lines. Exclude test files from the main review - they get the test-coverage cross-cutting pass.
4. Partition: ~600-1000 effective LOC per chunk, lexical sort, single files over 1000 LOC get their own chunk. **Cap: 6 chunk reviewers** - merge the smallest adjacent chunks until 6 or fewer.

### 2. Dispatch reviewers

One reviewer per chunk (plus cross-cutting reviewers below), each prompted:

```
You are reviewing a lexical chunk of a project's source code. Perform a
thorough code review covering: correctness, security, error handling, API
design, performance, and maintainability.

Project root: {PROJECT_ROOT}
Project context: {AGENTS.md summary, key constraints from the LOC survey}
Chunk: {CHUNK_NAME} ({CHUNK_LOC} LOC, {CHUNK_FILE_COUNT} files)
Files in this chunk:
{CHUNK_FILE_LIST}

For each file, inspect its structure first (grep for declarations, sed/head
for section boundaries), then read specific sections that look concerning.
Do NOT read every file in full - read only what looks complex or worth
inspecting.

Return findings as a JSON array:
[
  {
    "severity": "CRITICAL" | "HIGH" | "MEDIUM" | "LOW",
    "file": "relative/path.ext",
    "line": 42,
    "category": "correctness" | "security" | "error-handling" | "api-design" | "performance" | "maintainability",
    "message": "Description of the issue",
    "suggestion": "How to fix it"
  }
]

Rules:
- Only report REAL issues you can point to with file:line evidence
- Do NOT report style preferences, missing tests for trivial code, or hypothetical concerns
- CRITICAL = data loss, security vulnerability, crash path, data corruption
- HIGH = incorrect behavior under normal conditions, leaked secrets, race condition
- MEDIUM = poor error handling that could mask bugs, confusing API, unnecessary complexity
- LOW = minor improvements, naming, minor redundancy
```

**Cross-cutting reviewers** (run in parallel with chunk reviewers):

- **Dead code** - per the `cleanup` mode's method: for each file's exported/public symbols, count usages repo-wide (excluding the definition site), then check dynamic references before flagging. Each finding LOW unless in a file with higher-severity findings (then MEDIUM).
- **Test coverage** - reviews excluded test files: missing test files for complex modules, happy-path-only tests, flaky patterns (timers, network calls without mocks). MEDIUM (missing coverage for complex code) or LOW (shallow tests).
- **Reuse** - for each new helper, abstraction, or reusable unit in the diff, search the repo for an existing shared equivalent (sibling modules, shared utilities, project docs, migration or deprecation notes). Hand-rolled code that duplicates one is MEDIUM (HIGH if the project docs mark the existing one as the standard or the hand-rolled approach as deprecated). Logic duplicated across 2+ new call sites with no shared home is MEDIUM.
- **Architecture** - scans imports/includes and cross-file references with `rg`/`grep`, then reviews the dependency structure for: circular dependencies (HIGH), god modules with fan-in > 20 (MEDIUM), leaky abstractions (MEDIUM), layering violations (HIGH).

### 3. Aggregate and report

1. Collect all findings into one list.
2. Overlapping findings from different reviewers → reviewers discuss and converge; the lead arbitrates the rest.
3. Deduplicate (highest severity wins), cluster by file path, rank CRITICAL → HIGH → MEDIUM → LOW.

Output:

```
# Project Review: {project_name}

## Summary
- Files reviewed: {count} ({LOC} LOC)
- Chunks: {count} ({chunk_size_range} LOC each)
- Reviewers: {chunk reviewers} + architecture + dead-code + tests
- Findings: {N} CRITICAL, {N} HIGH, {N} MEDIUM, {N} LOW

## Critical (must fix)
## High (should fix)
## Medium (recommendations)
## Low (suggestions)

## Architecture Notes
## Dead Code
## Test Coverage Gaps

## Files by Issue Density
<file path | CRITICAL | HIGH | MEDIUM | LOW | total, sorted by total desc, top 15>
```

Clean up the scratch review team if one was created.

## Shared rules (both modes)

- **Structured findings**: project-mode chunk reviewers return JSON arrays so the lead can parse and rank programmatically; diff-mode tracks return prose findings in the MUST FIX / SHOULD FIX / CONSIDER buckets shown above.
- **Severity calibration**: CRITICAL/HIGH require concrete evidence (file:line, test case, or invariant violation). MEDIUM/LOW may reference patterns and best practices.
- **Never auto-fix.** Read-only review; report findings only.
- **Respect .gitignore.** Files excluded by git are out of scope.
- **Context budget**: each reviewer returns under 500 words of findings. The lead synthesizes.
- **Reviewers never edit files** - read-only always.
