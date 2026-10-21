# Work log timeline (auto-sync skeleton)

## Principles

- **Humans never hand-write this file.** AI appends and keeps it in sync at each step's completion.
- Purpose: leave the history of AI-driven steps (causal chains) in a form resolvable uniquely, without guessing — so later sessions and other maintainers can restore the context.
- Keep the same structure every time (`rules/00-governance-core.md`, the "5. Topology stability of documents" section).
- Write evidence. Never write bare assertions.

## Skeleton (duplicate and use)

### Task header

- Task name:
- Branch:
- Base SHA:
- Related plan md:
- Status: investigating / awaiting plan approval / implementing / awaiting acceptance / done

### Timeline

| Date/time | Step | What was done | Evidence (commands, result summaries, referenced md, relevant log lines) | Files touched | What happens next (causality) |
|---|---|---|---|---|---|
| | | | | | |

### Causal chain (only when needed)

- Write, with references and in chronological order, which step caused which problem and led to which fix.

### Current state

- Which step we are on now, and what the next gate is.

## Update rules

1. Append one timeline line every time a step completes (never batch-write later).
2. If the procedure changes, fix the plan md first, then continue the work; append to the log after that.
3. Before producing a response, confirm this log has been updated (`rules/02-response-self-check.md`, checklist item 13).
4. Write with unique references (file paths, full headings, commands). No ambiguous abbreviations.

## Endpoint processing (after acceptance)

1. Record the human acceptance/testing completion in the header status.
2. **Reflect the confirmed procedures and results into the relevant plan md.** Do not leave detailed trial-and-error logs in the plan md (archive them to a destination when needed: a gitignored work-log directory, e.g., `work-logs/`).
3. Update the related md files whose content changed (`rules/01-workflow-gates.md`, the "4. Post-acceptance md sync (endpoint processing)" section). Code updates and md updates come as a set.
4. For work that needed a tester checklist md (human test items), confirm it exists.
5. Move low-reuse finished deliverables (tester checklists, etc.) under `finished/` (e.g., `finished/testing-plan/`). When moving, search every md referencing the old location and sync the related md without fail (never skimp on the time to maintain unique references).
