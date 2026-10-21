# SETUP-CHECKLIST (inventory of initial settings)

Authoritative list of the items to enter and adjust when adopting this template into a project.

- If you let AI load `skills/setup-wizard/SKILL.md`, it asks and suggests through these items in this order and fills them in.
- When applying to an existing project, the setup-wizard analyzes the source code and documentation, structures the result into `templates/adoption-analysis-template.md`, and suggests the Section A values from it.
- Completion check: open each item's "write target" and confirm the placeholder has been replaced with a real value.
- Never start operating with any Section A item left unfilled.

---

## A. Items you must enter first (mandatory)

### A1. Top objective (compass)

- Write target: `rules/00-governance-core.md`, the "0. Compass (top objective)" section
- Format: 1–5 lines. State the purpose of this project using real names (product name, target business)
- Example: "Provide a business system that integrates with several external systems, stably and with the features it needs"
- Every judgment returns here. The direction of all implementation, investigation, and fixes is decided from "what is needed to fulfill this purpose"

### A2. Adoption language

- Choice: English (this `en/` tree) / Japanese (`../ja/` tree)
- Put only the adopted tree into the project. Installing both doubles the references and causes mix-ups

### A3. Authoritative documentation directory

- Write target: `rules/01-workflow-gates.md`, the "2. doc-first" section
- Format: real path of the directory that holds the md files for specs, procedures, and confirmations
- Example: `docs/forAI/` and `docs/`

### A4. Where to put task plan md and tester checklist md

- Write target: `rules/01-workflow-gates.md`, the "2. doc-first" section, and `skills/backend-test-first/SKILL.md`, the "0. Task plan" section
- Example: `docs/plans/`

### A5. Real names of the test infrastructure

- Write target: `rules/03-test-before-implementation.md`, `skills/backend-test-first/SKILL.md`, `skills/snapshot-test-debugging/SKILL.md`
- Format: test framework name, execution commands, how suites are split (markers etc.), whether golden files (files that fix the current behavior as expected values) exist
- If you do not introduce automated tests, delete the relevant skills per C2
- Additional write target: fill the guaranteed-to-succeed, as-is commands into a test runbook duplicated from `templates/test-runbook-template.md`. Treat the runbook as a living authoritative document — append and revise it whenever the environment changes, commands change, or something fails

### A6. Protected-branch table

- Write target: `skills/git-repo-operations/SKILL.md`, the "0. Branch table (replace the sample with your real names)" section
- Format: real names for production, staging, feature base, and leftover branches
- Example: production `main`, staging and feature base `staging`, leftover `master`

### A7. Initial entry of project-specific premises

- Write target: a premises file duplicated from `templates/project-premises-template.md`
- Format: environment differences, outage schedules of external dependencies, sandbox/permission quirks, and other traps likely to be hit repeatedly

---

## B. Items to customize or choose per project (recommended)

### B1. Mode-name mapping

- Write target: `rules/01-workflow-gates.md`, the "5. Mode selection and risk" section
- Remap the table to the real mode names of your agent tool (Ask / Plan / Debug / Agent, etc.)

### B2. Tester deliverable format

- Write target: `skills/backend-test-first/SKILL.md`, the "9. Tester checklist (endpoint)" section
- Choices: md + spreadsheet tsv (default) / md only / no tester operation (not recommended: it removes the human acceptance gate)

### B3. Adopting golden (current-behavior fixture) operation

- Write target: `skills/backend-test-first/SKILL.md`, the "Fixed procedure (implementing in any other order is forbidden)" section
- If your test infrastructure has no golden concept, remap the relevant steps to that infrastructure

### B4. Git strategy

- Write target: `skills/git-repo-operations/SKILL.md`, the "3. Branch creation rules" section
- Choices: a staging branch as the feature base (default) / trunk based (branch features directly from the production branch), etc.

### B5. commit / push operation

- Write target: `rules/01-workflow-gates.md`, the "3. Approval before edits" section, and `skills/git-repo-operations/SKILL.md`, the "4. push, commit, and merge rules" section
- Default is "AI never executes; always manual by the human". If you allow conditional execution, document the conditions explicitly

### B6. Where to file open questions and issues

- Write target: `skills/backend-test-first/SKILL.md`, the "Handling of pre-existing bugs" section
- Write the real name of the list or issue tracker where spec-confirmation findings and known problems are filed

---

## C. Items you may delete when unnecessary

When deleting, also remove the references to the deleted targets from the remaining files (`skills/setup-wizard/SKILL.md` proposes deletions; every deletion requires human approval).

### C1. Projects that do not use a database

- Deletion candidates: `skills/query-tuning/SKILL.md`, `skills/mysql-performance-diagnosis/SKILL.md`, `skills/fast-snapshot-db-reset/SKILL.md`

### C2. Projects without (or not introducing) an automated test infrastructure

- Deletion candidates: `skills/backend-test-first/SKILL.md`, `skills/snapshot-test-debugging/SKILL.md`, `templates/desired-red-gate-checklist.md`
- Even then, keep the summary in `rules/03-test-before-implementation.md`, remapped to substitute evidence procedures (manual verification, presenting real logs)

### C3. Operations where AI never touches git

- Deletion candidate: `skills/git-repo-operations/SKILL.md`

### C4. When the physical snapshot uses another method

- Rewrite candidates: the XtraBackup-specific sections of `skills/fast-snapshot-db-reset/SKILL.md` (setup, troubleshooting), replaced with the adopted alternative (e.g., incremental application of logical dumps)
