# Ensemble Mode

What changes when opencode-ensemble's `team_*` tools are available. Ensemble is not just "workers exist" - it is a qualitatively stronger system with three capabilities generic subagent tooling lacks:

1. **Messaging between agents.** Workers talk to each other (builder-to-builder contracts, reviewers cross-examining each other's findings) and to the lead without going through the orchestrator's context. Conflict adjudication, contract negotiation, and cross-examination depend on this.
2. **A persistent task board with dependencies.** `team_tasks_add` with `depends_on` mirrors plan.md's task graph; the board holds IDs, claims, and completion state while workers run. plan.md stays the durable source of truth; the board is the disposable runtime view.
3. **Per-branch worktree isolation.** Each implement worker gets its own branch; `team_merge` carries only committed branch refs back into the lead's tree. This is what makes parallel execution safe - and what makes the gitignored-file embedding rules load-bearing.

Tool mechanics - exact spawn syntax, board APIs, merge commands, model selection, prompt recipes - are documented by the opencode-ensemble skill in its own repository. Load that skill when operating the team tools; this file defines the protocol and quality gates only.

If only generic spawn-and-collect subagent tooling exists (no messaging, no board), use the generic-subagent tier in SKILL.md instead: no cross-agent contracts, no cross-exam loops, plan.md as the only tracker, serial same-file work.

## Team lifecycle

| Stage | Rule |
|---|---|
| Create | One team per pipeline (or reuse the active pipeline's team for review). Review-only runs may create a scratch team. |
| Spawn | One at a time. Workers are leaf-only: never spawn subagents, nested sessions, or delegate work. The lead alone manages and spawns. |
| Roles | `research`/`review` workers: no worktree (read-only). `implement` workers: worktree isolation. Plan-research architect: no worktree (writes the gitignored plan.md). |
| Model | Per spawn: pick the best-fit model preserving the session's provider and model prefix; exact ID from the models listing. No fit → stop and ask rather than switching providers. |
| Report | Workers report via task-result messages. Message-driven - never poll a status board. On ANY wake-up (a bare notification, a user "continue", a stall flag), call `team_results`/`team_status` once before concluding there is nothing new - a delivered message may be empty in the transcript while its full content sits in the results store. Never end a turn idle while any worker may have completed unreported work. |
| Merge | Merge each worker's branch, inspect the diff, then verify acceptance. Never merge without reading the result and the diff. |
| Shutdown | Shut down workers as their tasks complete; never leave idle workers. |
| Cleanup | Clean the team at the end. Purge actions need human approval. |

Operational gotchas that change outcomes: a shut-down worker's NAME stays registered (respawn under a fresh name); merged work lands UNSTAGED in the lead's tree (commit additive files mid-chain so later worktrees see them); `team_view navigate:true` hijacks the user's client; worktree workers cannot see gitignored files (all context flows through the embedded prompt).

## Plan debate (implement step 5)

The plan review becomes an adversarial debate between critics and the architect.

- **Critics**: one Simplifier always; add a Risk Auditor when the plan touches more than 5 files or is architectural. Both read plan.md (no worktree) and apply their protocols from implement.md Step 5, returning MUST RETHINK or CONSIDER findings.
- **Loop**:
  1. Label every finding with its critic (Simplifier-F1, Risk Auditor-F2, ...). No MUST RETHINK findings → log the round and converge.
  2. Send ALL MUST RETHINK findings to the architect at once. Per finding the architect must respond **REVISE** (edit plan.md and report what changed) or **REBUT** (reasoned rejection; silent dismissal is not allowed).
  3. Send each critic the architect's responses plus the plan path to re-read. Each returns **ACCEPT** or **ESCALATE** per finding.
  4. The lead adjudicates escalations only. An ACCEPT or an adjudication closes a finding - never relitigate a closed finding.
  5. Disputed findings loop back to step 2. A critic raising a NEW flaw on a revised plan counts as a new round. After 3 unresolved rounds, the lead edits plan.md itself to settle the dispute and ends the debate.
- Log every round in plan.md's Log: findings by critic, REVISE/REBUT responses, ACCEPT/ESCALATE verdicts, adjudications. CONSIDER findings are logged, never blocking.
- On convergence: re-validate the format, set `Meta.team` to the team name, shut down the architect and critics.

## Execution (implement step 6)

Single-worker tasks become a parallel board:

1. **Derive the board**: for every pending/in_progress plan.md task, add a board task with the same content and `depends_on` mirroring `depends:`. Record the mapping in plan.md's Log (`T3 mapped to board <id>`). The board is a disposable runtime view; plan.md is the source of truth.
2. **Waves**: spawn a builder per ready task, all in parallel (one spawn call at a time; workers run concurrently), with worktree isolation and the hand-assembled prompt (plan-schema.md Prompt assembly) plus: commit on your branch when done; report **DISCOVERIES** (anything contradicting the plan's assumptions - wrong file, changed API, hidden dependency, flawed criterion - never silently adapt around them). Tasks that touch the same files must be chained via `depends:` so they never run as parallel worktrees - same-file parallelism is a merge-conflict generator, not a speedup.
3. **Verification gate**: verify acceptance criteria yourself - run the tests, read the files, inspect the diff (worktree paths are visible without merging). Met → shut down the builder → merge → inspect → mark done + Log → complete the board task → if `commit_boundary: yes`, commit the merged work with a descriptive message. Not met → send the gap; a task revised since spawn gets a freshly assembled prompt; max 2 retries, then absorb. Workers already running a task that was revised mid-run get messaged the revised criteria immediately.
4. **Stalls**: on a stall flag, one status ping; a second flag → shut down force → respawn fresh with the same board task (same 2-respawn cap). Weigh flags against context: long silent commands trip flags while healthy. When in doubt, ping with a de-risking directive (commit finished files, then report) instead of killing.
5. **Hook-blocked commits mid-chain**: when project-wide pre-commit checks fire, a serial chain's first committable state may be its last task. Detect this at plan time: if tasks deliberately leave known-red references for later tasks (a field removal whose callers come later), the first committable state is the LAST task of the chain and no mid-chain task carries `commit_boundary: yes`. When a builder's commit fails the hook on exactly the predicted known-red set, that is an expected event: the builder stops and reports the red list for the lead's ruling - never `--no-verify`, never improvising around hooks. Fold the chain into ONE builder worktree; the builder commits once when the tree goes green - that commit is the plan's atomic boundary. Never collect uncommitted work by copying or stashing: only committed branch refs merge.
6. **Merge conflicts** between parallel branches: resolve yourself; for non-trivial resolutions, message both builders for intent before choosing.
7. **Builder contracts**: builders may message each other to negotiate shared interfaces; require them to report every agreed contract to the lead, recorded in plan.md's Log.
8. **Conflict adjudication**: teammates disagree → the lead weighs both sides, rules, logs the reasoning. Only genuine ambiguity about the user's goal stops the pipeline to ask.
9. **Risky work**: require the worker's plan approval before it edits when the task touches auth, payments, migrations, data deletion, concurrency, or security.

## Review (implement step 7; review mode)

Reviewers are spawned as read-only teammates (no worktree) into the active team or a scratch review team:

- Diff mode: parallel tracks (dead code, specialist, conditional security) become parallel reviewer teammates.
- Project mode: one reviewer per lexical chunk (cap 6), plus cross-cutting reviewers (dead code, test coverage, architecture).
- **Cross-examination round**: after findings return, merge all findings into one numbered list labeled by reviewer, send it to every reviewer, and require **CONFIRM** or **DISPUTE** per finding with reasoning. One round only - no debate loops. The lead adjudicates remaining disputes (an upheld dispute counts as confirmed). Single reviewer → skip the round. In project mode, cap the cross-exam list: send only CRITICAL and HIGH findings plus the top 20 by severity to every reviewer (low-severity findings are self-evident and do not need cross-exam).
- Reviewers return structured findings (JSON arrays); dedupe keeping the highest severity. Never auto-fix.

## Fallback

If team tools exist but a spawn consistently fails (wrong model, no capability), do not degrade the pipeline silently: fix the spawn (fresh name, re-verified model) up to 2 attempts, then absorb the task solo per implement.md. A degraded mode with silent quality loss is worse than solo mode.
