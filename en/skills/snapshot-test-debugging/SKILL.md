---
name: snapshot-test-debugging
description: >-
  The correct way to run snapshot/golden tests and the analysis procedure when
  tests or performance improvements fail unexpectedly. Use whenever running tests,
  starting servers, verifying speed, or wondering "why did this fail". Rules to
  prevent blind guesswork fixes.
disable-model-invocation: false
---

# Test execution and failure analysis (snapshot-test-debugging)

For work touching test execution, server startup, DB reset, or golden recording, **always** confirm this procedure before acting. Never invent ad-hoc commands or verification scripts not found here (real names come from SETUP-CHECKLIST A5).

Mandatory reading: the test runbook duplicated from `templates/test-runbook-template.md`. The runbook is authoritative for exact commands; if it is stale, fix it before running.

## 0. Never suppress command output (absolute)

- **Never truncate results with `2>/dev/null` / `>/dev/null` / `| tail` / `| head` / `| grep`.**
- Always emit **all** stdout/stderr of tests, DB resets, DB operations, etc. If long, read the terminal/log files the tool writes. Never thin via pipes.
- Never create "I can't see progress" or "it looks frozen" states yourself. Never use execution methods that hide the situation by discarding output.
- Only when output is truly too large for human reading, reduce it with the user's instruction. The AI never suppresses output merely because it is large.

## 1. Before implementation: run the existing tests in scope and confirm green

Before changing a single line of source, actually run the **existing** tests in scope and confirm all green. The authoritative rule is `rules/03-test-before-implementation.md`, the "1. Pre-implementation gate" section.

## 2. Things you must do before running

### Leftover process check

Before **every** test run, check whether test servers or previously started processes remain from an earlier abnormal exit.

```bash
ps aux | grep <server-process-name> | grep -v grep
```

- If it matches expectations (only what you just started), all good.
- If an unrecognized process with an old start time remains, **confirm its content before any automatic kill** — never kill unrelated processes by mistake.

### Put execution permissions into the arguments for real (pre-send verification, absolute)

In sandboxed environments, some commands require **granted permissions**: loopback connections, port inspection, commands involving `sudo`, etc.

- For such commands, actually put the permission field into **the tool call arguments (top-level field of the argument JSON)**. Not into the command string, not into your prose.
- **Even if you say you "included it", it may be missing from the actual arguments.** Before sending, re-read the assembled arguments and confirm the permission field is really there. Never resend without checking.
- Without the permission, shell commands may behave as if running in **a different network namespace**, making outside processes (including leftovers) "invisible" — misjudging the port as free and silently starting a new process. Test results obtained in that state are untrustworthy.
- **Diagnostic order on failure**: when you see permission-related errors (`sudo` with `no new privileges`, etc.) or local connection failures, **check your own arguments before suspecting the environment, the agent, or MySQL**. If the permission field was missing, rerun the same command once with permissions.
- **When the same command succeeded minutes ago but now fails, the cause is often your own execution form (permission-field omission), not the environment.** Never repeat a full-suite run without permissions, wasting the entire run and cost.

### Branching the DB reset mode (physical snapshot available / not)

The mode is determined by configuration (introduction state of `skills/fast-snapshot-db-reset/SKILL.md`; default is the logical load). **Some development machines do not have the physical snapshot. If not introduced, the conventional logical load is fine.**

| Environment | Full-suite runs with reset |
|---|---|
| Physical snapshot introduced | After dump/seed/schema changes, retake the snapshot first, then **always run via the physical snapshot**. Never flee to the logical load just because "the snapshot is old" |
| Not introduced | Conventional logical load (dump + seed) as before. No prepare needed |

- Prohibited on introduced machines: running the full suite via the logical load "because the snapshot is old". The snapshot exists precisely to avoid the logical load every time.
- The physical-snapshot reset does not re-apply seeds (included in the snapshot). If the snapshot is old, retake it.
- The only exception for returning to logical load (even on introduced machines): recovery after a copy-back failure (the "6-1. Recovery on failure (logical reset)" section of `skills/fast-snapshot-db-reset/SKILL.md`).

### When testing with changes the reset does not reproduce

The DB reset restores "dump + seeds from `tests/db/seed/`" (or equivalent). **Changes not reproduced by that process** (indexes added via migrations, fixtures not in seeds, manually inserted data) **disappear on reset**.

- When measuring/verifying with such changes applied, always run with the reset skipped (environment variable etc., as settled in A5). This applies to all testing of reset-invisible changes, not just indexes.
- Index performance comparison procedure example: (1) measure without the index → (2) apply the migration → (3) measure again without reset.

### When you cannot tell the baseline, re-measure both

If you cannot be sure which log represents "the original state", never fix one log as the baseline by guesswork. **Re-measure both conditions (e.g., index present/absent) on the spot.** Re-measuring is faster and more certain than reasoning over old logs.

## 3. When something unexpected happens, investigate evidence before judging

When the unexpected occurs — tests not running, no output, abnormally slow — obey the following.

- **Never decide the cause by guessing.** No "probably the code", "probably the operation". First investigate concrete evidence: the actual error messages, server logs, real command output, the command itself as executed.
- **Never open every possibly-related file and declare a cause.** Examine only what the evidence points to. Even when suspecting an unedited file, first produce evidence that the file is really involved.
- Never start fixing before the evidence is complete. Act only on the facts the evidence shows.

## 4. Creating goldens for code containing pre-existing bugs

### Step A: Record the current buggy behavior as golden

Do not fix the bug before recording. **Record the current (buggy) behavior as golden.** This lets the tests detect "the response changed" after the fix.

### Step B: Document the "failure prediction" before implementation (mandatory)

After recording the tests, and before changing a single line of source, write into the target plan md (fill-in template: `templates/desired-red-gate-checklist.md`):

1. **Tests that must fail (changes expected)**: test name/condition, current behavior (bug state), expected behavior after the fix
2. **Tests that must not change**: other endpoints calling the same function, other parameter patterns — with the reasoning why they will not change
3. **When the implementation approach branches**: organize in advance "if approach A, X changes; if approach B, Y changes"

### Step C: Verification after implementation

- Only the predicted "tests that must fail" are failing
- All "must not change" tests pass
- After confirming both, re-record only the failed tests
- If failures differ from the prediction, **stop fixing immediately**, report the difference, and ask for instructions

## 5. Failure analysis procedure (most important)

### Never be misled by the framework's debug dumps

Most test frameworks, on failure, output a huge debug dump (enumerating all variables/response fields) **before** the actual error. **This is not the cause of the failure.** The real cause is at the **end** of the output — the error section (`Errors:`, `TestFailError`, status code mismatches, Python tracebacks, etc.). When analyzing a failure, find the error section first. Do not spend time on the dump.

```bash
grep -A 30 "^Errors:" <output file>
```

### Look at server-side logs, not test logs

- The test's own log contains only the test results and the framework's failure messages.
- **The application-side real log** contains per-query execution times and actual tracebacks of exceptions inside the server. Judge both performance verification ("did it really get faster") and cause investigation ("why a 500") from this log.

### Blind guesswork fixes are forbidden

- Never apply "commonly likely causes" or "general best practices" without checking this specific error.
- Before fixing, identify the real cause in this codebase from the specific error message, traceback, and logs.
- If the cause cannot be identified or multiple possibilities remain, narrow it down with more log checks or reproduction experiments. Never attempt fixes while unidentified.

### One fix, one verification

- Never stack multiple fixes and verify them together. Change one thing, verify that one change.
- If the expected result (speedup, test PASS) is not obtained, **never attempt the next fix on your own judgment** — present the actual error/log as is and ask for instructions.
- For performance verification, compare actual elapsed times, not just PASS/FAIL. Constantly suspect that tests may have run against different code or a different environment (leftover processes, etc.).
