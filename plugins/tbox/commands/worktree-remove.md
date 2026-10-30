---
description: Remove a worktree created by /tbox:worktree — the worktree itself, its branch, and its .code-workspace entry
argument-hint: '[task-name] [project]'
allowed-tools: Bash(git *), Bash(ls *), Bash(cd *), Read, Edit, ExitWorktree
---

Remove a worktree previously created by `/tbox:worktree`: the worktree checkout,
its local branch, and its entry in the enclosing `.code-workspace` file.

`$1` is the task name (optional) — when omitted, the target is the worktree
the session is currently inside. `$2` is an optional project override — when
omitted, the project is whatever repo the session is currently in.

## Step 1 — resolve the project and the target worktree

Find where the session is:

```bash
git rev-parse --show-toplevel
git rev-parse --git-common-dir
```

**Resolve the project (`PROJECT_ROOT`):** if the session is inside a worktree
(`--git-common-dir` differs from `.git`), the main checkout is the parent of
the common dir — use that. Otherwise the current repo root is the project. If
`$2` is given, instead match it case-insensitively against directory names
under the workspace root (see Step 2) and use that repo. `PROJECT_ROOT` must
never be `$HOME` and must never itself be a worktree.

**Resolve the target worktree (`WT_PATH`):**

- **No `$1`:** the session must currently be inside a worktree; the target is
  that worktree's root. If the session is in the main checkout instead, list
  the project's worktrees (`git -C <PROJECT_ROOT> worktree list`) and stop —
  ask which one to remove. Never guess.
- **With `$1`:** look for it in `git -C <PROJECT_ROOT> worktree list
  --porcelain` — match a worktree whose path ends in `/$1` or whose branch is
  `$1`. Prefer paths under the central folder
  (`${WORKTREES_DIR:-$HOME/.worktrees}/<PROJECT>/$1`). If nothing
  matches, show the worktree list and stop. If several match, list them and
  ask.

Never target the main checkout. If `WT_PATH` resolves to `PROJECT_ROOT`,
stop — this command removes worktrees only.

Record `WT_PATH`, `WT_BRANCH` (may be `(detached)`), and `TASK` (the leaf
directory name of `WT_PATH`).

**Detached HEAD is normal here, and the branch still exists.** `/tbox:worktree`
creates the worktree *on* a branch named after `TASK`; a later review flow may
run `git checkout --detach` inside it (`/tbox:cross-review` before 1.9.0 did), which leaves the
branch behind while `worktree list` reports `(detached)`. So when `WT_BRANCH`
is `(detached)`, resolve the owning branch by name:

```bash
git -C <PROJECT_ROOT> rev-parse --verify --quiet "refs/heads/<TASK>"
```

If that ref exists and is not checked out in another worktree, set
`WT_BRANCH` to `<TASK>` and treat it as this worktree's branch for Steps 3 and
5. Only if no such ref exists is there genuinely no branch to remove.

## Step 2 — find the .code-workspace file

Walk up from `PROJECT_ROOT`, at most 4 levels, and take the first directory
containing a `*.code-workspace` file:

```bash
d=<PROJECT_ROOT>; for i in 1 2 3 4; do d=$(dirname "$d"); ls "$d"/*.code-workspace 2>/dev/null && break; done
```

If several exist, prefer one whose name matches its directory's basename,
otherwise ask. If none is found, continue — skip Step 6 and say so in the
report.

## Step 3 — safety gate: refuse to destroy work silently

**Run all checks before anything is removed.** Every check is read-only.

```bash
# 1. uncommitted changes (staged, unstaged, untracked)
git -C <WT_PATH> status --porcelain
# 2. branch commits not on the remote default branch (skip if detached HEAD)
git -C <PROJECT_ROOT> log --oneline "origin/<BASE>..<WT_BRANCH>" 2>/dev/null
# 3. does the branch exist on the remote?
git -C <PROJECT_ROOT> ls-remote --heads origin "<WT_BRANCH>"
```

`<BASE>` is the remote default branch from
`git -C <PROJECT_ROOT> symbolic-ref --short refs/remotes/origin/HEAD`
(strip `origin/`).

Interpret the results:

- **Uncommitted changes** — show the `status --porcelain` output and STOP.
  Do not remove anything. The user must explicitly confirm they want the
  changes destroyed before you proceed with `git worktree remove --force`.
- **Unpushed/unmerged commits** (check 2 non-empty AND check 3 empty) — list
  the commits and STOP. Ask whether to push the branch first, keep the branch
  but remove the worktree, or discard everything. Wait for the answer.
- **Branch pushed to remote, or detached HEAD, or no extra commits** — safe;
  continue without asking.

Never pass `--force` to `worktree remove` or `-D` to `branch` except as the
user's explicit, per-run answer to one of these stops. Prior approval in an
earlier run does not carry over.

## Step 4 — leave the worktree if the session is inside it

Skip this step entirely if the session's cwd is not under `WT_PATH`.

Otherwise move the session out before Step 5 deletes the directory underneath
it. Two mechanisms, in order:

```
ExitWorktree({ action: "keep" })
```

Always `keep`, never `remove` — Step 5 does the removal, and `remove` is
refused anyway for worktrees entered by `path`, which is how `/tbox:worktree`
enters them.

**A no-op result here is expected, not a failure.** `ExitWorktree` only knows
about worktrees that `EnterWorktree` entered *in this same session*, and that
state is per-session — it does not survive a restart. So whenever `/tbox:worktree`
ran in an earlier session (or the session simply started with its cwd inside
the worktree), this call reports "no active EnterWorktree session to exit" and
changes nothing. That is the common case. Do not report it as an error, do not
retry it, and do not stop.

Then, regardless of what the tool returned, move the shell out explicitly:

```bash
cd <PROJECT_ROOT>
```

This is what actually guarantees the session isn't sitting in a directory
Step 5 is about to delete. Without it, later `Bash` calls fail with a stale
working directory.

## Step 5 — remove the worktree and its branch

```bash
git -C <PROJECT_ROOT> worktree remove "<WT_PATH>"
```

This deletes the directory and unregisters it in one step. If it fails
because of untracked/modified files that Step 3 already surfaced and the user
already approved discarding, retry once with `--force`; for any other error,
report it and stop — do not improvise flags.

Then delete the local branch, if one exists — including the `<TASK>` branch
recovered in Step 1 when HEAD was detached. Skip only when Step 1 found no
matching ref at all:

```bash
git -C <PROJECT_ROOT> branch -d "<WT_BRANCH>"
```

Use lowercase `-d`. If git refuses because the branch is not merged, that
contradicts Step 3's gate only if the user chose "discard everything" — in
that case use `-D`; otherwise keep the branch and say so in the report. Never
touch the remote branch — deleting `origin/<branch>` is out of scope.

Finish with a prune so nothing stale lingers:

```bash
git -C <PROJECT_ROOT> worktree prune
```

## Step 6 — deregister from the .code-workspace

Read the workspace file and remove the `"folders"` entry for this worktree.
Match on path, not name — and compare **resolved** paths, never raw strings.
Entries may be absolute or relative, and both forms are in active use:

- absolute — compare directly to `WT_PATH`.
- relative — resolves against the directory holding the workspace file, so
  `../../../.worktrees/my-app/review-1123` in
  `~/Desktop/Projects/team/team.code-workspace` is
  `~/.worktrees/my-app/review-1123`.

Normalize every entry that way, then remove the one resolving to `WT_PATH`.
String equality alone would skip the relative entries and silently orphan
them in the file after the checkout is gone.

Remove the whole object, including its trailing comma, keeping the file's
existing JSONC formatting. If no entry matches, leave the file alone and note
it in the report.

## Step 7 — report

State plainly:

- what was removed: worktree path, branch (deleted / kept / was detached),
  workspace entry (removed / not found / no workspace file)
- anything intentionally left behind and why (e.g. branch kept because
  unmerged, remote branch untouched)
- that VS Code needs the workspace reopened for the folder to disappear

Then stop. Do not start new work in the main checkout, and do not offer to.
