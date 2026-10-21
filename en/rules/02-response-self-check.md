# Pre-response self-check (feedback processing)

## 0. When to run

Always run it after generating an answer and **before presenting it**. Fix violations, then output.

## 1. Checklist items

1. **Gaps or excess** — does it cover everything the user asked for?
2. **Errors** — are procedures, configuration values, and UI notation correct?
3. **Missed documentation references** — did you drop or overwrite authoritative values/procedures already in an existing md?
4. **doc-first** — were commands, procedures, values, and specs checked from existing md, not source code? (the "2. doc-first" section of `rules/01-workflow-gates.md`)
5. **Project-rule violations** — no CLI presentations for GUI-only operations? no destructive operations without approval?
6. **Test-first** — did you actually run the existing tests in scope and confirm green? If skipped, did you state the reason and substitute evidence? (`rules/03-test-before-implementation.md`)
7. **References exist** — count every file/section/rule/skill reference and confirm each resolves to a real name or full heading. Rewrite line numbers, symbol anchors, and code words.
8. **Proper-noun-ized common nouns and unsupported assertions** — no names unique only after guessing the context? Does every factual sentence have evidence seen this turn? Write guesses as guesses.
9. **Nothing written twice** — no pairs repeating the same content in different words? Delete one.
10. **Answering the question asked** — re-read the last utterance; confirm you answer that question.
11. **New files, names, numbers** — does it fit an existing file? Does the name match the content? Numbers chosen by importance?
12. **Not thinning documents** — did fixes/rearrangements silently discard existing definitions, notes, or links? If you drop something, get approval in the edit plan.
13. **Deliverables synced into md** — work reflected into the work md and related md files? Tester checklist md omitted only when the user explicitly said so.

## 2. What to do on violations

- Fix flagged items before outputting. When fixing after a correction, also check for slips (the banned phrasing must not survive in disguise). If unsure, state so and ask. Never output while unverified.

## 3. Why omission is forbidden

- Responses get slower, but the first-pass approval rate rises and approval rounds shrink; total cost (time + tokens) improves. "The more you rush, the more you enforce the self-check."
