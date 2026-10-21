---
name: edit-plan-approval
description: >-
  Presents a batch edit plan before any file create/change/delete when the user's
  request would modify the codebase. Use when the user requests implementation,
  fixes, docs updates, refactors, rollbacks, or any work that may touch files.
  A user request is NOT approval; wait for explicit OK after the plan.
disable-model-invocation: false
---

# Edit plan → approval → execution

## Triggers

Follow this skill **before the first Edit** when any of the following applies:

- A request for implementation, fixes, refactoring, documentation updates, or configuration changes
- An investigation concludes that file changes are necessary

## A request is not an approval

When you receive a request:

1. Settle the approach via investigation (read_file / search_files, etc.)
2. Write the **edit plan**
3. Run the integrity check (below) on it
4. Present the plan
5. Wait for the user's explicit **OK / approved / execute it**
6. Only after approval, use the editing tools (apply_diff / write_to_file / search_and_replace / insert_content, etc.)

**No per-file approvals.** 1 request = 1 plan (multiple files may be batched).

## Mandatory self-check before presenting the plan

Right before showing the plan to the user, run the self-check of `rules/02-response-self-check.md` against **the plan text itself**. No item is exempt. Watch especially:

- Did you verify every reference exists? (no line numbers, symbol anchors, bare numbers, or undefined abbreviations)
- No common nouns proper-noun-ized? (the "2. Allowed and forbidden forms of references" section of `rules/00-governance-core.md`)
- Does every factual sentence have evidence you saw this turn? (no unverified or self-contradicting assertions)
- Nothing written twice?
- Are you answering what was asked?
- For new files, names, and numbers: would they fit existing ones, does the name match the content, is the number chosen by importance?

Fix anything flagged before presenting. Never present an unverified plan.

## Edit plan template

```markdown
## Edit plan (awaiting approval)

### Understanding of the request
(1–2 sentences on what the user wants)

### Files to change and approach
| File | Approach |
|---|---|
| (md under docs/plans) | (what changes and how) |
| (source file) | (what changes and how) |

### What will not change
- (out-of-scope files and behaviors)

### Pre-implementation green (mandatory for plans that change source)
- Existing tests in scope: (files / markers / commands)
- Executed?: If not, run them **before** editing source. Write the results (counts, pass/fail, base SHA) into the plan
- If already red: do not implement. Identify the cause from commit history and report first

### Execution order
1. ...
2. ...

### Points to confirm
- (if any; otherwise "none")

### Integrity check (resolvable without guessing?)
- Every reference in this plan resolves uniquely. No ambiguous anchors, no slips

---
Upon approval, I will execute exactly this scope.
```

## Prohibited

- File edits before presenting the plan
- Presenting plans violating integrity (ambiguous anchors, slips, guess-required abbreviations)
- Treating request phrases ("go ahead", "create it") as approvals
- Changing files outside the plan
- `git commit` / `git push` without the user's explicit instruction

## After approval

- Execute exactly as planned. Report facts only (never declare "no regression" or "done" without test evidence)
- If a direction change becomes necessary mid-work → stop and present a **new edit plan**
