---
name: backend-test-first
description: >-
  End-to-end procedure for behavior-changing development (refresh base → create
  feature → existing tests green → generate all missing goldens (when existing
  behavior changes) → desired red → implement → green → tester checklist). Read
  before implementing, for bug fixes, security, performance, and refactoring alike.
disable-model-invocation: false
---

# Test-first, end-to-end (backend-test-first)

Changes that alter existing behavior (bug fixes, security fixes, performance improvements, refactoring, etc.) must **never be implemented in any other order**. The endpoint is the tester-facing checklist md. Never stop halfway.

- Rule summary: `rules/03-test-before-implementation.md`
- How to cut branches: `skills/git-repo-operations/SKILL.md`
- Test execution and failure analysis: `skills/snapshot-test-debugging/SKILL.md`

This skill is the **authoritative procedure and completion criteria**. For real command names (A5), the values settled in SETUP-CHECKLIST take precedence; for day-to-day execution commands, the test runbook (duplicated from `templates/test-runbook-template.md`) is authoritative.

---

## Fixed procedure (implementing in any other order is forbidden)

### 0. Task plan

- Write the impact scope, existing tests, failure predictions, and seed requirements into the plan md (the SETUP-CHECKLIST A4 directory)
- Present the edit plan and change files only after explicit user approval (`skills/edit-plan-approval/SKILL.md`. Never edit on request phrases alone)

### 1. Refresh the base

```bash
git fetch origin
git checkout <base branch>
git pull origin <base branch>
```

Record the base SHA in the plan md.

### 2. Development branch creation

```bash
git checkout -b feature/<ticket>-<short-name>
```

- **Without tracking** (`git checkout -b feature/... origin/<protected branch>` is forbidden)
- Never push directly to protected branches. Never use or push to leftover branches (`skills/git-repo-operations/SKILL.md`)

### 3. When fixtures are required (before implementation code, verified separately)

- Add only seeds/schema completion (do not change production logic yet)
- **Run the tests and confirm all green** (follow the "Branching the DB reset mode (physical snapshot available / not)" section of `skills/snapshot-test-debugging/SKILL.md`)
- Red → investigate the cause. If the diff is fixture-caused and harmless, re-record goldens → rerun to green. Stop on unpredicted results
- Never combine fixture work and logic implementation in one step

### 4. Pre-implementation tests (mandatory gate)

- Run the **existing** tests / related suites in scope and confirm all green
- "It was green before", "it passed in another task" are not confirmations. Write the commands, counts, and pass/fail into the plan
- The scope is **every existing suite** tied to the functions, APIs, seeds, and markers you change. Running only the new desired tests is not enough
- If even one is red, do not implement. Leave the failure logs, identify the cause via `git log` / `git blame`, and report (never mix your own task fixes in)

### 5. Full generation and testing of missing goldens (conditional)

**Mandatory (changes to existing processing)**: query tuning, handler/CRUD SQL rewrites, changes that could alter response shapes — anything where **regression to existing results is possible**. If cases are missing for affected targets → create **all missing ones** via the case-generation/recording procedure (settled in A5). Record current behavior (bugs included) as is. Humans never hand-edit goldens. After generation, run that suite and confirm green.

**Skippable (confirmation-query additions etc.)**: changes that touch no existing handlers, existing CRUD SQL, or response assembly — e.g., only adding a separate authorization-check SELECT to a map — where regression of normal operations is unlikely. In that case, skipping full golden generation for the scope is allowed.

- When skipping, state **the skip reason** (what you are not touching) in the plan md. Never skip silently
- Even when skipping, steps "4. Pre-implementation tests", "6. Desired red", "8. After implementation", and "9. Tester checklist" are never skipped
- If you touch or could break targets that already have goldens, they are not skippable (run that suite in step 4)

### 6. Desired tests → red before implementation

- Write the failure prediction into the plan, add expected-behavior tests, and **always make them fail (red) before implementing**
- Do not implement until the failure reason matches the prediction. Stop and report on mismatch
- Never skip the existing suites by running only desired tests

### 7. Implementation

- Only within the approved edit plan
- Prefer design and flow correctness over a small diff. No "while we're at it" fixes

### 8. Green and regression after implementation

- Desired green
- Re-record only the goldens that **changed as predicted**. Everything else unchanged
- **Confirm all tests green**
- Append the counts to the plan

### 9. Tester checklist (endpoint)

- Create the tester-facing checklist md (human test items). Format follows SETUP-CHECKLIST B2 (default: md + spreadsheet tsv)
- Include: background, **a summary of the restrictions/changes made (to catch miswiring)**, prerequisites, reference/update operation tables, out of scope, revision history
- When using a tsv: UTF-8, tab-separated, fixed header row, judgment column empty (for the tester). **The md is authoritative**; sync the tsv whenever the md changes
- Link only the tester md from the implementation plan md. Never link the tsv
- Omission allowed only when the user explicitly said it is unnecessary. Never finish "done" by skipping silently

### 10. commit / push

- **Human manual** (SETUP-CHECKLIST B5). Never run `git commit` / `git push` without instruction
- Push only new feature branches. Protected branches forbidden

---

## Prohibited

- Skipping or reordering steps 0–9 (except the conditional skip in 5)
- Implementing an existing-behavior change without generating the affected goldens (mandatory case in 5)
- Skipping 5 without a reason in the plan
- Implementing without confirming desired red
- Moving on after adding fixtures without confirming the full suite
- Declaring "task done" without the tester md (only when the user explicitly opted out)
- Saying "guaranteed to work", "no regression", "no problem" without test evidence
- Verifying multiple fixes in one shot (1 fix = 1 verification)

---

## Handling of pre-existing bugs

- In code full of existing bugs and no spec sheets, **keep the buggy behavior as is** and fix it as golden. Do not fix on your own initiative
- Bug fixes require business-logic and spec confirmation, so split them into a **separate task**. File open questions into the SETUP-CHECKLIST B6 destination
- When fixing multiple problems in one place, first write a combined work plan (targets, order, failure predictions) into the plan md and get approval

---

## Exceptions (only with the reason stated and user approval)

- Documentation-only changes / test-code-only additions
- Targets the test infrastructure cannot cover (headless batch jobs, external integrations, infrastructure settings): present substitute evidence and get approval. Never skip silently
