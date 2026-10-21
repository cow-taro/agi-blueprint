# Pre-adoption analysis report (adoption-analysis)

How to use: created in `skills/setup-wizard/SKILL.md`'s "Step 1: Analyze the existing project (existing-project mode only)" when applying this template to an existing project. It structures the analysis of source code and documentation, mapped to SETUP-CHECKLIST items. **This file is evidence for suggestions, not decided values.** The adoption judgment belongs to the user.

## 0. Analysis metadata

- Analysis date/time:
- Analyzed repository / directories:
- Files read (real names; the evidence of this analysis):

## 1. Purpose candidate (SETUP-CHECKLIST A1)

- Candidate value: (expressions extracted from README/docs)
- Confidence: high / medium / low (unknown)
- Evidence: (which statement in which file)

## 2. Authoritative documentation directory candidates (A3, A4)

- Candidate values:
- Evidence:
- Missing pieces: (e.g., no place for task plan md or tester checklist md)

## 3. Test infrastructure candidates (A5, B3)

- Test framework: (real name confirmed from `pytest.ini`, `package.json`, etc.)
- Execution command candidates:
- Markers / how suites are split:
- Presence of the golden (current-behavior fixture) concept:
- Evidence:

## 4. Branch name candidates (A6)

- Production candidate:
- Verification / feature base candidates:
- Leftover branch candidates:
- Evidence: (`git branch -r` results and `on.push.branches` in `.github/workflows/*.yml`. **Never trust `origin/HEAD`**)

## 5. Project-specific premise candidates (A7)

- List environment differences, periodic outages, execution-environment quirks, and past mix-up candidates (with evidence; "none detected" when empty)

## 6. DB usage (material for the C1 judgment)

- Whether MySQL / other DBs are used, with evidence (connection settings, migrations, schema files)

## 7. Recommended actions (items to confirm with the user)

- Items adoptable as is:
- Items with 2+ candidates requiring a choice:
- Items the analysis could not resolve (routed to step 2 questions):
- Section C deletion candidates:
