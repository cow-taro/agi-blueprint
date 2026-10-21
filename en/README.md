# English Guide — agi-blueprint/en

English version of the generic AI governance framework. `../ja/` contains the identical Japanese version. **Adopt exactly one language tree per project.**

## Structure and how files are loaded

| Type | Location | Loaded | What goes here |
|---|---|---|---|
| rules | `rules/` | Always (applied every turn) | Only high-frequency rules that apply every time |
| skills | `skills/` | On demand (for the relevant work) | Procedures and know-how that apply only occasionally |
| templates | `templates/` | On demand (copy and fill) | Fill-in templates |

The appendix at the tree root, `SAFE-AUTOMATION.md` (safe AI automation design guide), is not loaded constantly by the agent — it is a reference document humans read at adoption time to decide their own scope.

**Do not put occasionally-applied content into rules.** Keep the always-on context minimal to protect the model's attention.

## File list and roles

### rules (always loaded)

| File | Role |
|---|---|
| `rules/00-governance-core.md` | Authoritative top-level rule. Compass (top objective) fill-in frame, unique references, no raw code dumps, document topology stability, escalation of repeated failures |
| `rules/01-workflow-gates.md` | Separation of planning and execution, evidence-based planning, approval before edits, doc-first, post-acceptance md sync, mode selection, cost and autonomy |
| `rules/02-response-self-check.md` | Pre-response self-check (feedback processing). Fix rule violations before producing the output |
| `rules/03-test-before-implementation.md` | Summary of the test-first procedure (details delegated to skills) |

### skills (loaded on demand)

| File | Role |
|---|---|
| `skills/setup-wizard/SKILL.md` | Interactive setup wizard. Asks the items of `SETUP-CHECKLIST.md` and writes the answers into the files |
| `skills/ask-when-unclear/SKILL.md` | Ask back instead of guessing when instructions are ambiguous, conflicting, or incomplete |
| `skills/edit-plan-approval/SKILL.md` | Authoritative procedure for edit plan → human approval → execution |
| `skills/git-repo-operations/SKILL.md` | Git safety rules (protected branches, branch creation, stop before commit/push, cherry-pick porting) |
| `skills/backend-test-first/SKILL.md` | End-to-end procedure for behavior-changing development (pre-implementation gate → desired red → implementation → green → tester checklist) |
| `skills/snapshot-test-debugging/SKILL.md` | Correct test execution and analysis of unexpected failures (never suppress output, evidence first) |
| `skills/query-tuning/SKILL.md` | Policy for speeding up slow MySQL queries (splitting huge JOINs, avoiding full view materialization, authorization-boundary verification) |
| `skills/mysql-performance-diagnosis/SKILL.md` | Diagnosis via EXPLAIN / performance_schema, index design principles (server config changes: AI proposes, humans execute) |
| `skills/fast-snapshot-db-reset/SKILL.md` | Introduction and usage sample of a fast DB reset via XtraBackup physical snapshots |

### templates (copy and use)

| File | Role |
|---|---|
| `templates/desired-red-gate-checklist.md` | Fill-in self-check template for the pre-implementation / failure-prediction gate |
| `templates/work-log-skeleton.md` | Skeleton of a work log that AI keeps syncing automatically (humans never hand-write it) |
| `templates/project-premises-template.md` | Where to append project-specific premises (copy and operate) |
| `templates/adoption-analysis-template.md` | Template for the structured analysis md produced when applying this blueprint to an existing project (used by the setup-wizard analysis mode) |
| `templates/test-runbook-template.md` | Living runbook holding environment-specific guaranteed commands and the strict execution procedure (leftover-process kill, conflict avoidance) |

### Appendix (reference document — not something to duplicate)

| File | Role |
|---|---|
| `SAFE-AUTOMATION.md` | Design guide for safe AI automation. Absolutely-forbidden work, the two privileged-work routes, keeping the bastion stopped, blast-radius limitation, quantization warnings, and the adoption checklist |

## Adoption steps

1. Place the contents of this `en/` tree into the project. With Roo / Claude style agent tools, put `rules/` into the rules directory and `skills/` into the skills directory. If your tool has no rule mechanism, pin the contents of `rules/` to the **top** of the system prompt (rules at the top keep being referenced even when the context grows, preventing rule forgetting).
2. Open `SETUP-CHECKLIST.md` and fill in the mandatory Section A items. If you let AI load `skills/setup-wizard/SKILL.md`, the setup completes just by answering its questions and suggestions.
3. Adjust Section B to the project's circumstances and delete the skills and sections that fall under Section C.
4. Duplicate `templates/project-premises-template.md` as the premises file and start operating.

## Extending during operation (eliminating repeated failures)

- When the same failure happens three times: append it to the premises file if it is a project-specific circumstance, or turn it into a new skill if it is a reusable general procedure (see `rules/00-governance-core.md`, the "6. Escalation of repeated failures" section).
- Propose the premises-file addition to the user before making it. Do not skill-ize one-off, temporary, or situation-dependent judgments.
- Do not create new numbered rule files on an ad-hoc basis.
