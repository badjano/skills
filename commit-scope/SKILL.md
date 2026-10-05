---
name: commit-scope
description: >
  Apply when staging files, finishing any code change, preparing a PR, handling a
  commit-and-push / diff-tab commit action, or when the user asks to commit or rejects
  a change. Auto-commit every completed change on feature/* (never on develop); keep
  staging scoped to the change intent; on user rejection (“didn’t like it” / “didn’t
  see any change”), reset those commits and discard the changes. Squash-on-merge means
  commit count is unimportant — prefer many small commits over holding uncommitted work.
---

# Commit scope

## Policy (branch + frequency)

PRs squash on merge, so **commit count does not matter**. Prefer **many small commits** over leaving work uncommitted.

1. **Commit every completed change**, no matter how small (after each discrete edit/fix/feature slice the agent finishes).
2. **AI may commit without waiting for the user to ask**, but **only** on a `feature/*` branch.
3. **Never commit on `develop`** (also never on `main` / `master`).
4. If the current branch is not `feature/*`, **create or switch to a `feature/*` branch first**, then commit. Do not commit while still on a protected branch.
5. Keep the staging gate below: each commit still contains only files for **that** change’s intent — do not blanket-add unrelated WIP.

## Hard gate (do this before every `git add` / commit)

**Never stage first and sort later.** Before any stage or commit:

1. **Confirm branch:** `git branch --show-current` must match `feature/*`. If not, create/switch to `feature/<short-name>` before staging.
2. **Name the commit intent** in one sentence (what this commit is *for*). Infer from the task just finished — not from “everything dirty in the tree.”
3. **List all candidate changes:** `git status` + `git diff --stat` (and untracked). Treat the full working tree as *suspect*, not as the commit.
4. **Classify every path** (or tight path group) as one of:
   - **INCLUDE** — required to implement or correctly ship that intent
   - **EXCLUDE** — unrelated, experimental, accidental, generated, or belonging to another intent
   - **SPLIT** — belongs in a *different* commit (same branch OK; do not mix)
5. **Justify INCLUDEs briefly** (why this file is needed for *this* commit). If you cannot say why in one line, **EXCLUDE** it.
6. **Stage only INCLUDE paths** with explicit `git add <path>…`. Do **not** use `git add .`, `git add -A`, or `git commit -a` unless the user explicitly ordered “commit everything.”
7. **Re-check staged set:** `git diff --cached --stat` / `--name-only`. If anything outside the INCLUDE list is staged, unstage it before committing.
8. **Tell the user** what was committed (short hash + message) and what was left unstaged — especially when the dirty tree is much larger than the commit.

If a UI / “diff-tab commit-and-push” / automation suggests committing the whole branch diff, **still run this gate**. The button does not expand scope.

## What “important for that commit” means

A file is INCLUDE only if **removing it from the commit would break, omit, or misrepresent** the stated change.

| Usually INCLUDE | Usually EXCLUDE |
|-----------------|-----------------|
| Source that implements the feature/fix | Prefabs/materials/meshes touched only while experimenting or profiling |
| Asmdef / project refs required to compile that code | Mass Synty / vendor / TreePack / Addressables churn from A/B tests |
| Tests or fixtures that validate *this* change | Unrelated WIP on the same branch |
| Config/assets the feature **reads or requires** at runtime | Formatting-only sweeps, drive-by renames |
| `.meta` **only** for new assets you are actually adding | Orphan `.meta`, deleted generated junk, `ProfilerCaptures`, local caches |
| | Docs/`*.md` unless the task was documentation |
| | `learnings.md` / preference logs unless asked |
| | Temporary debug, profiler probes, timing menus, investigative logs |

**Unity / art-heavy trees:** A long `git status` of foliage prefabs, mats, and scenes after a perf session is **noise by default**. Only stage the scripts/config that encode the real product change (e.g. quality tiers), not every asset that was toggled during measurement — unless the user explicitly wants those authored asset results in the commit.

## Out of scope by default (do not mix)

- Documentation not requested for this task
- Editor/tooling config and formatting-only sweeps
- Preference / meta logs
- Generated or accidental artifacts
- **Temporary debug / profiler / timing / measurement scaffolding** — do **not** stage unless the user **directly** says to include it. A bare “commit” / “commit the PR” / diff-tab action is **not** permission

## Mixed local changes

- **One logical change per commit** (`feat` / `fix` / `refactor` / `perf` / …).
- Never bundle unrelated features or tasks.
- Accidental touch → leave unstaged or **SPLIT** into another commit.
- **Micro-commits are fine** (same intent split across tiny steps is OK because squash-on-merge). Do not hold finished work uncommitted to “batch” commits.

## Analysis checklist (copy mentally each commit)

```
Branch: feature/<name> (not develop/main/master)
Intent: <one sentence>
INCLUDE:
  - path — why
EXCLUDE / leave unstaged:
  - path or group — why
SPLIT (optional separate commit):
  - path — other intent
Staged verification: matches INCLUDE only? yes/no
```

## User rejection → reset and discard

When the user says they **didn’t like it at all**, **didn’t see any change**, or otherwise wants the agent’s last work fully undone:

1. Identify the commit(s) created for that rejected work (prefer commits authored in this session on the current `feature/*` branch).
2. **Reset those commits and discard the changes** — e.g. `git reset --hard <sha-before-rejected-work>` (or `HEAD~N` when N is exactly that rejected slice).
3. Confirm with `git status` / `git log -1` that the branch is back and the working tree is clean of that work.
4. Do **not** reset on `develop` / `main` / `master`. Do **not** rewrite history that has already been merged.
5. If the rejected commits were already pushed to the feature remote, warn once and only force-push that `feature/*` branch if the user explicitly confirms.

## GitHub PR size warnings

Before pushing or opening/updating a PR, estimate files vs base (`git diff --name-only <base>...HEAD` plus still-to-include paths):

| Threshold | Action |
|-----------|--------|
| **~200+ files** | Soft warn — consider splitting / dropping noise |
| **~300+ files** | Strong warn — review UX often degrades |
| **~1000+ files** | Hard warn — single-file mode limits |
| **~3000+ files** | Stop and escalate — split the work |

Do not silently grow a PR past these; ask whether to split or proceed.

## Commit message

- Matches the **single** logical scope (not a laundry list of excluded work).
- **Conventional Commits:** `type: description` — e.g. `feat:`, `fix:`, `refactor:`, `chore:`, `docs:`, `perf:`.
- Pass the message via HEREDOC (PowerShell-safe equivalent on Windows). Follow the workspace git safety protocol (no config edits, no `--no-verify` unless asked, no amend except the documented amend cases).

## Verification

- Do not commit until the task’s required verification (compile / tests / logs for that change) has been done when project rules require it.
- A clean compile of *included* code does not justify staging unrelated dirty assets.
