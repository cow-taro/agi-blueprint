# Safe AI automation design guide (SAFE-AUTOMATION)

Positioning: the safety-design appendix of this template. It implements, at the operational level, "humans own the responsibility" from `rules/01-workflow-gates.md`, the "0. The three absolute rules" section. It is **not** loaded constantly by the agent — it is a reference document humans read at adoption time to decide their own scope. The principles derive from the author's actual operation (as of 2026-10-16). The subject matter is the **basics of safe AI use** — an extension of safe computer practice.

## 1. Work you never let AI do directly stays forbidden — no exceptions

- Never open exceptions to work decided as "absolutely never", for reasons like "busy", "urgent", or "it's a trivial operation". **Exceptions become permanent.**
- Each adopting project defines its own list in the Section 8 checklist. Typical examples: direct modification/deletion of production data, issuing/changing credentials, administrator-level operations, billing/cancellation operations, operations on shared resources affecting other projects.

## 2. Privileged work goes through exactly two routes

Privileged work (the operations AI must not do directly) is performed only via one of these routes:

- **Route (a) personal copy-paste execution**: AI only presents the command. The user personally verifies it, copies and pastes it into their own terminal, and executes it themselves.
- **Route (b) via a web terminal (or similar) that requires two-factor authentication**: safer. The route itself is protected by authentication.

Inconvenient or not, never break these two routes.

## 3. When not in use, keep it physically unreachable

- Access routes to production etc. (bastion servers and the like) are **stopped or network-restricted** whenever you are not working. The final responsibility for stopping lies with the human. As a safeguard against forgetting, a periodic auto-kill (scheduled shutdown), e.g., every Sunday, is effective.
- Maintain, as the steady state, a condition where production environments are unreachable unless the route is started through two-factor authentication.
- Avoid handling data involving personal information or production environments as much as possible. When it must be handled, go strictly through the two-factor-authentication gate and weigh the leak risk.
- Only information that could not enable a direct attack even if leaked may remain in an archive outside repository management, covered by ignore settings. Personal information, credentials, and data leading to production access are **deleted**, not archived.
- Purpose: **keep a state where, even if your computer is hacked, leaking personal information or attacking production is impossible.**

## 4. Source code is protected by PR review approval

- Source changes and releases go only through protected branches plus pull-request review approval (consistent with the operation in `skills/git-repo-operations/SKILL.md`). AI never pushes to or merges protected branches directly.

## 5. Limit the blast radius to yourself

- The goal is to confine any damage, should an accident or breach happen, as much as possible to yourself (and that project). **How far to go is up to each developer.**
- Hobby projects: with **keys of narrowly limited permission** that cannot affect other projects, AI may sometimes be trusted even to deploy to production.
- Professional work: never grant the computer **admin privileges including production changes**, and never leave those credentials on the computer.

## 6. Implementation example (the author's environment, as of 2026-10-16)

- Linux inside a development-only VM: all commands are allowed, except `su`-type commands (apart from some resource restarts), git operations, and file edits outside the workspace.
- Secret files such as `.env` are protected with rooignore (the agent's ignore settings, e.g., `.rooignore`).

## 7. Safety aspects of model selection

- If you insist on complete information control, consider **running the standard-quantization GLM 5.3 locally**.
- **Never change from the quantization used in training to a lower one — it demonstrably increases hallucination.**
- Prefer AI providers that seem least likely to lower quantization for cost reasons.

## 8. Adoption checklist (fill in per adopting project)

| Item | Entry |
|---|---|
| List of work AI must never do directly | |
| Privileged-work route ((a) personal copy-paste / (b) 2FA-required web terminal / both) | |
| Production access route (bastion etc.) and the stop/network-restriction means when idle | |
| Type of two-factor authentication required to start it | |
| PR review approval operation (protected branches, reviewers) | |
| Project category (hobby / professional) and key permission-separation policy | |
| Secret-file protection (rooignore etc.) | |
| Handling of sensitive data (avoid-by-default, gate routing, delete/archive criteria) | |
| Whether a local LLM is needed (complete information-control requirement) | |
