---
name: git-btw
description: Perform a side cleanup or small improvement ("by the way…" task) in an isolated git worktree on a fresh branch cut from the default branch (main/master), then push it and open a draft PR/MR via gh or glab. Use when the user wants a refactor/cleanup/fix handled separately from their current work-in-progress so their dirty tree stays untouched.
argument-hint: <guidance for the side task, may reference code in the current dirty tree>
---

# /git-btw — side task in an isolated worktree

The user is in the middle of something else. Their working tree is likely dirty and they may **keep editing files while you work**. Your job: take the guidance in `$ARGUMENTS`, do that one task on a clean branch cut from the **default branch** (never the current branch), in a **separate git worktree**, and finish with a **draft** PR/MR. Never modify anything in the user's active working tree.

## Phase 1 — Snapshot FIRST (before any analysis or worktree setup)

The active tree is a moving target, so capture everything you might need immediately:

1. `ROOT=$(git rev-parse --show-toplevel)` — abort with a clear message if not in a git repo.
2. `SNAP=$(mktemp -d /tmp/git-btw-snap.XXXXXX)`
3. Save the current dirty state:
   - `git -C "$ROOT" status --porcelain=v1 > "$SNAP/status.txt"`
   - `git -C "$ROOT" diff > "$SNAP/unstaged.patch"`
   - `git -C "$ROOT" diff --staged > "$SNAP/staged.patch"`
   - `git -C "$ROOT" rev-parse HEAD > "$SNAP/head.txt"`
4. Read the guidance and identify which files it references (explicitly by path, or implicitly — "that helper I just added", "the config parser", etc.). Use the status/diff output plus quick greps to resolve implicit references. Copy each referenced file **as it exists right now in the working tree** into `$SNAP/files/` preserving relative paths (`cp` — include untracked files; they may exist only in the dirty tree).
5. From this point on, treat `$SNAP` as the source of truth for anything from the user's tree. Do not re-read files from the active tree later — they may have changed.

## Phase 2 — Scope the task

Determine what the guidance actually asks for, and which parts of the dirty state are relevant:

- The user's dirty tree contains changes for **their** main task plus possibly the code the guidance refers to. Extract only what's relevant to the BTW task; ignore all unrelated modifications — they must not leak into the new branch.
- If the relevant code exists only as uncommitted changes (e.g., "extract the helper I just wrote into utils"), carry over the **minimal** hunks/files needed from `$SNAP`, not whole unrelated diffs.
- If the guidance is about committed code, work from the default branch's version directly and use `$SNAP` only to understand what the user meant.

## Phase 3 — Create the worktree from the default branch

1. `git -C "$ROOT" fetch origin` (best effort — continue on failure, e.g. offline).
2. Detect the default branch:
   - `git -C "$ROOT" symbolic-ref --short refs/remotes/origin/HEAD` → strip the `origin/` prefix.
   - If unset, try `git -C "$ROOT" remote show origin | sed -n 's/.*HEAD branch: //p'`; failing that, check whether `origin/main` or `origin/master` exists; with no remote at all, fall back to local `main`/`master`.
3. Pick a branch name `btw/<short-kebab-slug-from-guidance>`; if it already exists, append a numeric suffix.
4. `WT=$(mktemp -d /tmp/git-btw-wt.XXXXXX)` then
   `git -C "$ROOT" worktree add "$WT/repo" -b <branch> origin/<default>` (use the local default branch if there is no remote).

All subsequent edits, builds, and git commands happen inside `$WT/repo` only.

## Phase 4 — Do the task

- Implement exactly what the guidance asks — this is a focused side task, not an invitation to improve everything nearby.
- **Do not write new tests unless the guidance explicitly asks for them.** Updating existing tests that your change breaks is fine and expected.
- If the project has a fast, obvious check (lint, typecheck, affected existing tests), run it in the worktree to verify the change compiles/passes. Skip long-running suites.

## Phase 5 — Commit, push, draft PR/MR

1. Commit in the worktree with a concise message describing the cleanup (conventional style if the repo uses it — check `git log --oneline -10`). Prefer "chore" over "refactor" action word unless it's a notable change in internal design.
2. `git -C "$WT/repo" push -u origin <branch>`.
3. Choose the CLI from the remote URL (`git remote get-url origin`):
   - contains `github.com` (or `gh auth status` recognizes the host) → `gh pr create --draft --base <default> --title "..." --body "..."`
   - contains `gitlab` (gitlab.com or self-hosted) → `glab mr create --draft --target-branch <default> --title "..." --description "..." --yes`
   - Otherwise: report the pushed branch and let the user open the PR/MR themselves.
1. Title/body: summarize what changed and why; do not reference user's in-flight work. Include the Claude Code attribution footer per the usual convention.
2. If `gh`/`glab` is missing or unauthenticated, don't loop on retries — report the pushed branch name and the exact command the user can run.

## Phase 6 — Clean up and report

1. `git -C "$ROOT" worktree remove "$WT/repo"` (use `--force` only if the worktree is fully committed and pushed) and delete `$WT` and `$SNAP`.
2. Report to the user: branch name, one-line summary of the change, the draft PR/MR URL, and confirmation that their working tree was not touched.

## Hard rules

- Never edit, stage, stash, or clean anything in the user's active working tree.
- Never base the branch on the current branch or on HEAD — always the default branch.
- Unrelated dirty changes must never appear in the BTW branch's diff.
- The PR/MR must be created as a **draft**.
