# Test runbook (living authoritative document)

How to use: duplicate this file, fill in the settled commands and the strict execution procedure for your development environment, and **keep appending and revising it while operating**. Tests run many times, and results break easily from environment differences, leftover processes, and conflicts with concurrently running tests. That is why only commands that are "**verified to succeed as-is as of the last confirmation date**" belong here. When something breaks or changes, fix this runbook first, then proceed.

## 0. Environment header

- Machine / OS: (e.g., WSL2 Ubuntu 22.04)
- Project root / venv path:
- DB reset mode: physical snapshot introduced / not introduced (see `skills/fast-snapshot-db-reset/SKILL.md`)
- Last verified:

## 1. Settled command table (only commands verified to succeed as-is)

| Purpose | Settled command | Last verified | Notes |
|---|---|---|---|
| Full suite | | | |
| Marker run (e.g., authorization suite) | | | |
| Single test | | | |
| Golden generation / recording | | | |
| DB reset (physical snapshot) | | | |
| DB reset (logical) | | | |

- If a command fails, fix the cause on the spot and reflect it here. Never leave it as "it worked before".
- Never invent ad-hoc commands or scripts not listed here. To add one, first run it, confirm it succeeds, then add it.

## 2. Strict pre-run procedure (every run, in this order)

1. **Check and stop leftover processes**
   ```bash
   ps aux | grep <real name of server / pytest> | grep -v grep
   ```
   - Kill only your own recognized leftovers, **after confirming their content**. Never kill unrelated processes.
2. **Conflict check against concurrently running tests**
   - Confirm nothing else is running that uses the same DB or ports — another pytest, a server, a manual run. If one is running, wait for it to finish or stop only your own before running. **Parallel runs corrupt test results and DB state.**
3. **DB state judgment**: if seeds/dumps/schema changed, reset first (retake the physical snapshot if introduced) before running. If you must measure with reset-invisible changes applied (indexes etc.), run with the reset skipped.
4. **Permission-required commands: actually put the permission field into the tool call arguments before sending** (`skills/snapshot-test-debugging/SKILL.md`, the "Put execution permissions into the arguments for real (pre-send verification, absolute)" section).
5. **Never suppress output** (`2>/dev/null`, `| tail`, `| head`, etc. are forbidden — `skills/snapshot-test-debugging/SKILL.md`, the "0. Never suppress command output (absolute)" section).

## 3. During and after the run

- **One test run at a time.** Never run multiple suites in parallel.
- On unexpected failure, rerun with a full reset (no skip flags) to isolate whether DB state was the cause.
- Record the results (command, counts, pass/fail, elapsed time) into the work log (derived from `templates/work-log-skeleton.md`).
- If you ran with a changed command, **update the settled command table of this runbook first.**

## 4. Maintenance rules for this runbook

- This runbook is the **living authoritative document**. Whenever the environment changes, commands change, or something fails, append and revise it so the settled command table is always true.
- A stale runbook makes every subsequent test run inaccurate. md updates come as a set with code updates (`rules/01-workflow-gates.md`, the "4. Post-acceptance md sync (endpoint processing)" section).
