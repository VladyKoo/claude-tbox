# worktrees

Central store for git worktrees. Each leaf is a **separate checkout of one
project on its own branch**, so several tasks can be open at once without
stashing or switching branches in the main checkout.

Created by the `tbox` Claude Code plugin:

- `/tbox:worktree <task> [project]` — create a leaf and register it in the
  workspace; moving the session into it is offered at the end, never assumed
- `/tbox:worktree-remove [task] [project]` — remove a leaf, its branch, and its
  workspace entry

## Layout

    <worktrees-root>/<project>/<task>

`<task>` is also the branch name. The branch is created from the project's
**remote default branch** — resolved per repo from `origin/HEAD`, never
hardcoded, since the default branch differs per repo — and fetched
first, so a new worktree starts from the current remote tip.

The leaf name is the project's **directory basename only**, so this namespace is
flat: two repos with the same basename under different roots would collide here.
Rename one before creating the second.

## Names are never reused

Creation refuses if the worktree, the local branch, `origin/<task>`, or the path
on disk already exists — and in that case creates nothing at all. Reusing a
worktree or a branch would mix two tasks onto one branch, and an existing
worktree may hold uncommitted work. No variant name (`<task>-2`) is ever
auto-generated; pick a new one deliberately.

## Which leaves exist right now

This file describes the concept and is written once, on first use. It is
deliberately **not** an inventory and is not updated per worktree — ask git:

    git -C <project-root> worktree list

To find the main checkout a leaf belongs to:

    git -C <path-to-leaf> rev-parse --path-format=absolute --git-common-dir

That prints `<project-root>/.git`; its parent is `<project-root>` above.

## Removing one

Use `/tbox:worktree-remove <task>`. It removes the checkout, its local branch, and
the `.code-workspace` entry, and refuses when there are uncommitted changes or
commits that exist on no remote. The remote branch is never deleted.

**Do not `rm -rf` a leaf.** The branch's only checkout lives here, not in the
main repo, so uncommitted work in it exists nowhere else. The equivalent manual
removal is:

    git -C <project-root> worktree remove <path-to-leaf>

If a leaf was already deleted by hand, clear the stale metadata with:

    git -C <project-root> worktree prune

## Why here, and why hidden

**This directory is deliberate. Do not sweep it as clutter.** Each leaf is a
real working checkout.

- It sits outside every project, so a worktree never nests inside a checkout.
- It is a dot-directory, because a visible, generically-named folder in `$HOME`
  is exactly what cleanup scripts — and agents tidying up — delete.

To relocate the whole store, export `WORKTREES_DIR`; both commands derive their
paths from that one variable.

## VS Code

Leaves are registered as folders in the enclosing `.code-workspace` file, named
`<project> [<task>]` and grouped next to their parent project. VS Code needs the
workspace reopened for an added or removed folder to appear or disappear.
Removing a worktree means removing its entry there too.
