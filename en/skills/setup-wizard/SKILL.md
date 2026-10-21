---
name: setup-wizard
description: >-
  Interactive setup wizard for adopting agi-blueprint into a project. Fills
  the Section A mandatory items and Section B choice items of SETUP-CHECKLIST.md
  through questions and suggestions, and writes them into the files. When applying
  to an existing project, it analyzes the source code and documentation, creates a
  structured md, and suggests the required items. Use right after adoption, in the
  first session, or when revisiting settings.
disable-model-invocation: false
---

# Setup wizard

## Purpose

The wizard fills the inventory items of `SETUP-CHECKLIST.md` by asking questions interactively. The human does not hand-edit the checklist — answering the questions and suggestions completes the setup.

## Mode decision (do this first)

- **Existing-project mode**: applying this template to a project that already has source code and documentation. → Run the analysis phase in step 1
- **New-project mode**: a project starting from scratch. Nothing to analyze. → Skip step 1 and go straight to asking in step 2

## Step 1: Analyze the existing project (existing-project mode only)

Analyzing the existing source code and documentation can estimate most SETUP-CHECKLIST items with high confidence. Do not fill by guessing — **suggest values with the analysis as evidence**.

### Analysis targets and what they reveal

| Analysis target | Reveals | Mapped SETUP-CHECKLIST item |
|---|---|---|
| Documentation directories (listing under `docs/` etc.), README | Where the authoritative documentation lives, the project's stated purpose | A1, A3 |
| CI config (`on.push.branches` and test jobs in `.github/workflows/*.yml` etc.) | Real protected-branch names, test commands | A6, A5 |
| Test config (`pytest.ini`, `package.json`, etc.) and test directories | Test framework, markers, execution commands, presence of golden concept | A5, B3 |
| `git branch -r` and remote state | Real branch names (production/staging/leftover; trust the workflows config, not `origin/HEAD`) | A6 |
| DB usage (connection settings, migrations, schema files) | Whether MySQL-related skills are needed | C1 |
| Existing runbooks, incident records, config differences | Candidates for project-specific premises | A7 |

### Structuring the analysis result

- Structure the analysis into an md following `templates/adoption-analysis-template.md` (analysis date, real names of files read, candidate values with confidence and evidence per item).
- Creating the md is a file change, so follow `skills/edit-plan-approval/SKILL.md` and get the edit plan approved before creating it.
- Mark unresolvable items as "unknown" and ask about them in step 2. Never write guesses as facts.

## Step 2: Suggestions and questions for Section A (never start operations with these unfilled)

- Present the Section A items in batches of 2–4. Attach candidate values (with confidence) from the analysis, or choices, to each item.
- Candidates are suggestions, not decisions. Adopt them only after the user confirms. For items you cannot infer, just ask.
- Do not move to the next step until every Section A item is settled.

## Step 3: Writing (edit-plan approval required)

- Present the collected values as an edit plan — a list of "which file, which section gets which value" — and get explicit approval before writing. Answers and requests are not approvals.
- Write targets:
  - A1 → `rules/00-governance-core.md`, the "0. Compass (top objective)" section
  - A3, A4 → `rules/01-workflow-gates.md`, the "2. doc-first" section, and `skills/backend-test-first/SKILL.md`, the "0. Task plan" section
  - A5 → the real names into `rules/03-test-before-implementation.md`, `skills/backend-test-first/SKILL.md`, `skills/snapshot-test-debugging/SKILL.md`
  - A6 → `skills/git-repo-operations/SKILL.md`, the "0. Branch table (replace the sample with your real names)" section
  - A7 → create the premises file duplicated from `templates/project-premises-template.md`
- After writing, re-read the files to confirm no placeholder is left behind.
- If the tree was placed in split locations, rewrite the inter-file references (relative paths of `rules/`, `skills/`, `templates/`) to the adopted real paths (syncing related md).

## Step 4: Section B choice questions (customization)

- Ask Section B in one batch and reflect the choices into the target files (again, edit-plan approval required).

## Step 5: Section C deletion proposals

- Enumerate the skills and sections matching Section C and propose deletions. Deletions also require approval. Never delete silently.
- When deleting, also remove references to the deleted targets from the remaining files.

## Step 6: Completion report

- Output a list of: filled items, unfilled items, files written, skills deleted, and the md files to read next.
- If any mandatory Section A item remains unfilled, prompt for completion before operations start.
