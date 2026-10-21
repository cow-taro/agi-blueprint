# Desired red gate checklist (fill-in)

How to use: duplicate this file into the task plan md and fill every item before implementation. Proceeding to implementation with blanks is forbidden (`rules/03-test-before-implementation.md`). AI fills it in; the human verifies.

## 1. Base information

- Branch: (feature name)
- Base branch and refresh confirmation: (date/time of `git pull`)
- Base SHA: (latest commit SHA of the base branch)

## 2. Pre-implementation gate (record of existing test runs)

| Executed suite / command | Count | pass | fail | Executed at |
|---|---|---|---|---|
| | | | | |

- All green?: yes / no
- If red, the cause investigation result (commit history, relevant log lines):
- Whether new goldens (current-behavior fixtures) are needed, and why: (mandatory when touching existing processing; otherwise the reason for skipping)

## 3. Failure prediction (must be documented before implementation)

### 3-1. Tests that must fail (desired red)

| Test name / condition | Current behavior (bug state) | Expected behavior after fix | Predicted failure shape |
|---|---|---|---|
| | | | |

### 3-2. Tests that must not change

| Test name / condition | Reason they will not change (rationale) |
|---|---|
| | |

### 3-3. Branching by implementation approach (when applicable)

- Approach A changes (what) / Approach B changes (what):

## 4. Desired red confirmation (fill in before implementation)

- Executed command:
- Failed tests and the actual failure reasons:
- Match with the failure prediction: matches / **mismatch (on mismatch, stop implementing and report the difference for instructions)**

## 5. Post-implementation confirmation (fill in after implementation)

- Desired green: yes / no
- Re-recorded goldens (only those changed as predicted):
- Full-suite rerun result (counts, pass/fail):
- "Must not change" tests passing:

## 6. Endpoint

- Tester checklist md (human test items) created: done (path: ) / omitted (reason: user explicitly opted out)
