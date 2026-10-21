---
name: git-repo-operations
description: >-
  Safety rules for git operations (branch creation, push, PR, protected-branch
  judgment). Use before running git checkout / branch / push / merge / PR, or
  whenever unsure which branches are protected or where pushing is allowed. The
  branch table with real names from SETUP-CHECKLIST A6 is authoritative.
disable-model-invocation: false
---

# Git repository operation safety rules

## 0. Branch table (replace the sample with your real names)

The table below is a sample. **The authoritative version is the one whose sample names were replaced with this project's real names via SETUP-CHECKLIST A6 (Protected-branch table).** Never operate while the real names are unsettled.

| Branch (sample) | Purpose |
|---|---|
| `main` | **Production**. Directly connected to deployments |
| `staging` | **Verification. Daily feature base** |
| `develop` / `test` | Only when needed |
| Leftover names like `master` | Not current. Do not use, do not push |

**Prohibited (never repeat this mix-up):**

- Never use leftover branches (`master` / `new-master`, etc.) as the base for production, verification, or features, or as a PR base.
- Even if same-named branches remain on the remote, they are **not current**. Even if `origin/HEAD` points to `master`, production is the real name in the table.
- Even if md files or skills contain old branch names, the real-name table in this skill is authoritative. When unsure, check the CI workflow definitions (`on.push.branches` in `.github/workflows/*.yml`).

## 1. Basics of repository and workspace structure

- Always `cd` into the target repository directory before git operations (never `git init` / `commit` / `push` at the workspace root). If `.git/` exists at the root it was created by mistake — propose deleting it after user confirmation.
- Do not propose committing things that belong to **no repository**: config files, secrets, shared documentation, etc. Check the repository's `.gitignore`.
- If a documentation-only repository exists, note its operation (whether feature branches are cut, or always main) in the SETUP-CHECKLIST A6 table.

## 2. Protected branches (direct push absolutely forbidden)

- Direct pushes to the production and verification branches in the table (sample: `main` / `staging` / `develop` / `test`) are absolutely forbidden, regardless of the reason. Force pushes obviously, but also normal pushes are forbidden.
- Never push directly to leftover branches either (their existence alone means they are not current).
- Merges happen **only via GitHub PRs**. The agent never merges or pushes to protected branches directly.
- Which branches are deployment-connected is confirmed from the CI workflow definitions (and reflected into the table).

## 3. Branch creation rules

- **Before cutting a new branch, always pull the base branch to the latest remote state.** Branching from a stale local base causes diffs and conflicts later.
- The daily base is the "feature base" branch of the table (sample: `staging`). Never branch from the production branch or leftover branches (unless B4 adopted trunk based).

```bash
git checkout staging
git pull origin staging
git checkout -b feature/xxx
```

- ❌ Forbidden: `git checkout -b feature/xxx origin/<protected branch>` (upstream gets set to the protected branch and causes accidental pushes). Always create without tracking.
- You may `git fetch` before pulling to check the remote, but **never skip the pull itself before branching**.

## 4. push, commit, and merge rules

- **Push only new branches**: `git push -u origin feature/xxx` pushes to a new remote branch.
- **Always stop right before `git commit` / `git push` / `git push -u origin`, present the commands to the user, and ask for execution.** Request phrases ("go ahead", "implement it") are not commit/push permissions. No exception for feature or new branches (unless SETUP-CHECKLIST B5 opted into conditional permission).
- Never commit secrets (real `.env` values, credentials, raw personal data). Before committing, run `git status` and visually confirm nothing secret is included.
- Destructive operations (`git reset --hard`, force push, history rewrite) only on the user's explicit instruction, after presenting the target and impact in an edit plan.

## 5. State verification after work

After cross-branch operations (after hearing a merge completed, after cherry-picks, etc.), never proceed on assumptions — verify the actual state before the next task.

```bash
git status --short --branch
git log --oneline -3
```

## 6. Porting between protected branches (cherry-pick) procedure

Procedure for porting a fix already merged into the verification branch over to the production branch (sample: `staging` → `main`).

Daily flow: feature → PR → verification branch → for production, create a production-bound branch, cherry-pick the same fix, etc. → PR → production branch → auto deploy.

### 6-1. Prerequisites and prohibitions

- Production and verification histories diverge daily. **Never decide the porting target from a whole-branch `git diff`.** Identify the exact commits (or content) to port.
- Never port into leftover branches, never pull from them.
- Before deciding, read this skill's "0. Branch table" and the authoritative deployment-environment md if one exists (doc-first).

### 6-2. Identifying the commits to port (never trust `git cherry` blindly)

`git cherry` can show "not yet applied" for content already merged under a different SHA (due to squash/rebase). Always verify by content.

```bash
git cherry -v main staging
git grep -n "<distinctive string of the change>" main
git ls-tree -r --name-only main
```

- If the change is already in the production branch under a different SHA, **do not apply it twice** (exclude it).
- Narrow the target to the actually-new commits, starting from the appropriate base commit.

### 6-3. Branch creation and cherry-pick

```bash
git checkout main
git pull origin main
git checkout -b feature/apply-xxx-to-main
git cherry-pick -x <c1> <c2> ... <cN>
```

- On conflicts, **never auto-resolve** — report the affected files and ask for the user's judgment.
- An interrupted state can be detected by the presence of `.git/CHERRY_PICK_HEAD`.

### 6-4. Sanity verification (mandatory before pushing)

- Confirm no conflict markers remain (search for `<<<<<<< ` etc.).
- Confirm important files exist (test infrastructure, migrations, config).
- **Confirm green with marker-tagged tests etc.** (do not forget the leftover-process check in `skills/snapshot-test-debugging/SKILL.md`).

### 6-5. push / PR are manual by the human

- Always stop right before pushing or creating a PR and present the commands (see "4. push, commit, and merge rules").
- Call out explicitly that the **PR base is the production branch (deployment-connected)**.
- If migrations are included, also state their application status per environment (idempotent? already applied?).
