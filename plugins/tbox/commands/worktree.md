---
description: Create a worktree from the remote default branch in the central worktrees folder and register it in the .code-workspace
argument-hint: <task-name> [project]
allowed-tools: Bash(git *), Bash(ls *), Bash(mkdir *), Read, Edit, Write, EnterWorktree
---

Create an isolated worktree for task `$1`, branched from the project's **remote
default branch**, stored in the central worktrees folder, and registered in the
enclosing `.code-workspace` file.

`$1` is the task name (required). `$2` is an optional project override — when
omitted, the project is whatever repo the session is currently in.

If `$1` is empty, stop and ask for a task name. Don't invent one.

## Step 1 — resolve the project

**Default (no `$2`):** the project is the git repo containing the current
working directory.

```bash
git rev-parse --show-toplevel
```

**If `$2` is given:** treat it as a directory name under the workspace root
(see Step 2), e.g. `my-app`, `my-service`. Match it case-insensitively
against the directory names there, or against a `"name"`/`"path"` in the
workspace file. If it matches nothing, list the candidates and stop.

Then validate the resolved repo root:

- It must be a git work tree (`git -C <root> rev-parse --is-inside-work-tree`).
- It must **not** be `$HOME`. The home directory is itself a git repo on this
  machine, so a cwd outside any project resolves there — that is never the
  intended target. If the root is `$HOME`, stop and say the session isn't
  inside a project.
- It must not already be a worktree (`git -C <root> rev-parse
  --git-common-dir` differs from `.git`). If the session is already in a
  worktree, resolve the **main** checkout from the common dir and use that as
  the project.

Record `PROJECT_ROOT` and `PROJECT` (its basename).

## Step 2 — resolve the worktrees root and the .code-workspace file

`WORKTREES_DIR` is the single source of truth for where worktrees live. It is
the same for every project and every workspace:

```bash
WORKTREES_DIR="${WORKTREES_DIR:-$HOME/.worktrees}"
```

**To relocate worktrees, change only that line** (or export
`WORKTREES_DIR`) — every other step derives from it. Don't create it
yet; Step 5 does that, after the collision gate has passed.

Two deliberate properties, don't undo them:

- It sits outside every project, so worktrees never nest inside a checkout.
- It is a **dot-directory**. A visible, generically-named folder in `$HOME` is
  the kind of thing cleanup scripts, and agents tidying up, sweep away.

Now find the workspace file to register into. Walk up from `PROJECT_ROOT`, at
most 4 levels, and take the first directory containing a `*.code-workspace`
file:

```bash
d=<PROJECT_ROOT>; for i in 1 2 3 4; do d=$(dirname "$d"); ls "$d"/*.code-workspace 2>/dev/null && break; done
```

That directory is `WORKSPACE_ROOT`; the file found is `WORKSPACE_FILE`. If
several `.code-workspace` files exist, prefer one whose name matches the
workspace root's basename, otherwise ask which to use.

If none is found, continue anyway — create the worktree, skip Step 6 (register),
and say so in the report.

## Step 3 — refuse if the name is already taken

**Run this gate before anything is created, fetched, or written.** Every check
below is read-only. If any of them hits, the command makes no changes at all —
it stops and asks for a different task name.

Never reuse an existing worktree or branch, and never auto-generate a variant
name like `$1-2`. An existing worktree may hold uncommitted work, and an
existing branch may hold commits; reusing either would mix two tasks together
on one branch, and that is what this gate exists to prevent.

Check all four, so the report can list every conflict at once rather than
surfacing them one at a time:

```bash
# 1. worktree already registered for this path or branch
git -C <PROJECT_ROOT> worktree list --porcelain
# 2. local branch of this name
git -C <PROJECT_ROOT> rev-parse --verify --quiet "refs/heads/$1"
# 3. remote branch of this name (no fetch needed)
git -C <PROJECT_ROOT> ls-remote --heads origin "$1"
# 4. path occupied on disk but not registered as a worktree (stale leftover)
ls -d "<WORKTREES_DIR>/<PROJECT>/$1"
```

If any check hits, **stop and ask the user for a different task name.** Report
which of these it was, in these terms:

- *worktree exists* — give its path and branch, and whether it has uncommitted
  changes (`git -C <path> status --porcelain`), since that says how much
  unsaved work is sitting there.
- *local branch exists* — give its last commit subject and date, so it's
  recognizable as either abandoned or live work.
- *remote branch exists* — say so plainly. Branching `$1` from the default
  branch while `origin/$1` already exists sets up a confusing divergence and
  risks clobbering someone else's work on push.
- *stale directory* — the path exists but no worktree is registered. This one
  is genuinely inert, so also mention it can be cleared with `rm -rf <path> &&
  git -C <PROJECT_ROOT> worktree prune` if the user recognizes it as junk.

Suggest a couple of concrete alternative names based on what `$1` was, but do
not pick one yourself. Wait for the user.

Only when all four checks are clear, continue.

## Step 4 — determine the remote base branch

Never hardcode `master`. The default branch differs per repo in this workspace
(`master`, `main`, and `dev` are all in use), so resolve it:

```bash
git -C <PROJECT_ROOT> symbolic-ref --short refs/remotes/origin/HEAD
```

That yields e.g. `origin/dev` — strip the `origin/` prefix to get `BASE`.

Fallbacks, in order:

1. If the ref is missing but an `origin` remote exists, populate it with
   `git -C <PROJECT_ROOT> remote set-head origin -a`, then re-read.
2. If that fails, probe `git -C <PROJECT_ROOT> ls-remote --heads origin` and
   pick the first of `master`, `main`, `dev` that exists.
3. If there is no `origin` remote at all, fall back to the local default
   branch and **say so in the report** — the worktree will not be from a
   remote in that case.

Then fetch, so the worktree is branched from the current remote tip rather
than a stale local ref:

```bash
git -C <PROJECT_ROOT> fetch origin <BASE>
```

Base ref for the worktree is `origin/<BASE>` (or the local branch under
fallback 3).

## Step 5 — create the worktree

Target path: `<WORKTREES_DIR>/<PROJECT>/$1`
Branch name: `$1`

Step 3 has already established that neither the path nor the branch exists, so
create both fresh. Always pass `-b`: this command only ever creates a new
branch, never checks out an existing one.

First ensure the root exists:

```bash
mkdir -p "<WORKTREES_DIR>/<PROJECT>"
```

Then, **only if `<WORKTREES_DIR>/README.md` is absent**, seed it. That file tells
a human or an agent poking around that the directory is deliberate, and it
describes the concept only — the layout, the branching rule, the name-reuse
refusal, how to remove one safely. It is **not an inventory**: never add the
worktree you just created to it, and never update it on later runs.

The canonical seed text ships with this plugin at
`${CLAUDE_PLUGIN_ROOT}/templates/worktrees-README.md` — read it from there and
write it verbatim. That file is the single source for this document; do not
paraphrase it or reconstruct it from memory.

Never overwrite an existing README — it may have been extended by hand, and the
seed is the minimum, not the truth.

Then create the worktree:

```bash
git -C <PROJECT_ROOT> worktree add -b "$1" "<WORKTREES_DIR>/<PROJECT>/$1" "origin/<BASE>"
```

If this fails despite the gate passing, do **not** retry with a different flag,
a different name, or `--force`. Report the error and stop — a failure here
means the world changed since Step 3, and forcing past it is how work gets
lost.

Then switch the session into it with the native tool — this is what makes the
rest of the session operate inside the worktree:

```
EnterWorktree({ path: "<WORKTREES_DIR>/<PROJECT>/$1" })
```

Pass `path`, never `name`. `name` would place the worktree under the project's
own `.claude/worktrees/`, which defeats the central-folder goal.

## Step 6 — register it in the .code-workspace

Read `WORKSPACE_FILE` and add one entry to the `"folders"` array:

```json
{
  "name": "<PROJECT> [$1]",
  "path": "<WORKTREES_DIR>/<PROJECT>/$1"
}
```

Rules:

- Write the **absolute, fully expanded** path — e.g.
  `/Users/you/.worktrees/my-app/my-task`. Do not write `~` or
  `$HOME`; VS Code does not expand either one in this file. Relative paths
  resolve against the file's own directory, and `WORKTREES_DIR` is outside
  every workspace root, so relative would be a brittle `../../..` chain.
  Some entries already in these files *are* relative — leave them as they
  are, don't convert them. `/tbox:worktree-remove` resolves both forms before
  matching, so the two conventions coexist safely.
- Reuse the project's existing display `"name"` from the file if it has an
  entry, so the worktree reads as belonging to it.
- Insert immediately after the project's own entry and after any worktree
  entries already grouped under it, keeping worktrees adjacent to their parent.
  If the project has no entry in the file, append to the end of the array.
- If an entry with this exact `"path"` already exists, leave the file alone.
- Preserve the file's existing formatting, including its trailing commas — it
  is JSONC, and VS Code accepts them. Don't reformat unrelated lines.

## Step 7 — report

State plainly:

- the worktree path and branch name
- which base ref it was branched from (e.g. `origin/dev`), and whether that
  came from `origin/HEAD` or a fallback
- that it is registered in the workspace file as `<PROJECT> [$1]`, and that
  VS Code needs the workspace reopened for the new folder to appear

Then stop. Do not begin work in the worktree, and do not offer to.
