# Test-first procedure before implementation (summary)

Changes that alter existing behavior (bug fixes, security fixes, performance improvements, refactoring) are started only following the authoritative procedure below.

- Authoritative end-to-end procedure: `skills/backend-test-first/SKILL.md`, the "Fixed procedure (implementing in any other order is forbidden)" section
- Fill-in template: `templates/desired-red-gate-checklist.md`
- Test execution and failure analysis: `skills/snapshot-test-debugging/SKILL.md`

## 0. Scope and exceptions

- Scope: every change that alters existing behavior
- Exceptions (only with the reason stated and human approval): documentation-only changes / test-code-only additions / targets the test infrastructure cannot cover (headless batch jobs, external integrations, infrastructure settings — present substitute evidence and get approval)

## 1. Pre-implementation gate

- Actually run the **existing** tests in scope and confirm all green before changing source. Running is confirming; "it was green before" is not a confirmation
- Write the executed commands, counts, pass/fail, and base SHA into the plan md. If even one test is red, do not start implementing; identify the cause from the commit history and report
- Never say "guaranteed to work", "no regression", or "no problem" without test evidence
