# Project-specific premises (append-target template)

## How to use

- Duplicate this file as the project's premises file (e.g., `project-premises.md`).
- Append circumstances unique to the project that debugging/implementation must assume **here**. Never create a new numbered rule file per item.
- When to append: when the same failure happens three times (`rules/00-governance-core.md`, the "6. Escalation of repeated failures" section).
- Propose the addition to the user before writing. Do not write one-off or temporary matters.
- Structural premises (the top objective, repository layout, performance constraints) belong to the core rules and the SETUP-CHECKLIST A1 compass. Here you write **traps hit repeatedly during debugging and implementation**.

## What to write

- Environment differences (schemas, settings, versions that do not match across environments)
- Periodic outage schedules of external dependencies (things to suspect whenever failures happen)
- Execution-environment quirks (sandboxes, permissions, network namespaces)
- Definitions of the "correct state" of local environments (easily misjudged things)
- Past repeated mix-ups and the rules preventing them

## Entry format (1 entry = 1 heading)

Write headings as complete statements of the phenomenon (so they are searchable later). Structure the body as follows.

### (Heading example) Never conclude "the local DB is down" from a failed connection

- Facts: (what happens; e.g., an agent-side connection failure is not evidence the host service is down)
- Verification order: (1. Does the user's terminal show a success record? 2. Read the app log's statuses and exceptions 3. If still unclear, the user's terminal result is authoritative)
- Prohibited: (forbidden judgments/operations; unauthorized recovery operations, wrong assertions to the user, etc.)
- Authoritative: (real name of the md to consult, if any)

## Initial examples (SETUP-CHECKLIST A7; rewrite for the actual environment)

- "Schemas may differ per development environment. When basing judgments on schemas, state which environment you confirmed in"
- "A connection failure from the sandbox may be an agent-side restriction. If the user's terminal has a success record, treat the service as running"
- "Production source contains abandoned and half-built remains. Never assume something 'is in use' without checking its call sites"
