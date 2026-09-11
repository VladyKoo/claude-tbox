---
description: Resolve merge/rebase conflicts semantically — reconstruct intent on both sides, preserve behavior from both, then sweep for silent semantic breakage git never flagged.
argument-hint: '[target-branch, only needed when no merge/rebase is in flight]'
disable-model-invocation: true
---

# Semantic merge

Your job is **not** to make conflict markers go away. It is to produce a tree
where the *intent* of both sides survives and the code is still correct. A
resolution that compiles, passes review, and silently drops one side's fix is
the worst possible outcome — worse than leaving the conflict unresolved.

Every claim you make about "what this side was doing" is unverified until you
cite the commit or line that supports it.

---

## Phase 0 — Sides and polarity (read-only)

`git status --porcelain=v2 --branch` and `git diff --name-only --diff-filter=U`.
Then **derive the two sides from what git recorded — do not guess them:**

```bash
if   [ -f .git/MERGE_HEAD ];        then op=merge;       onto=HEAD; landing=MERGE_HEAD
elif [ -f .git/CHERRY_PICK_HEAD ];  then op=cherry-pick; onto=HEAD; landing=CHERRY_PICK_HEAD
elif [ -f .git/REVERT_HEAD ];       then op=revert;      onto=HEAD; landing=REVERT_HEAD
elif [ -d .git/rebase-merge ] || [ -d .git/rebase-apply ]; then
     op=rebase; landing=REBASE_HEAD
     onto=$(cat .git/rebase-merge/onto 2>/dev/null || cat .git/rebase-apply/onto 2>/dev/null)
fi
base=$(git merge-base "$onto" "$landing")
case "$op" in rebase) mine=$landing; other=$onto ;; *) mine=$onto; other=$landing ;; esac
```

`<<<<<<< ours` is **always** `$onto`; `>>>>>>> theirs` is **always** `$landing`.
The markers are not the trap — **which side is yours** is:

| operation | my work | the other branch |
|---|---|---|
| rebase | `$landing` — the commit being replayed | `$onto` — what you are rebasing onto |
| merge, cherry-pick, revert | `$onto` — your checked-out branch | `$landing` — the incoming commit |

Use `$mine` / `$other` from here on; never "ours"/"theirs". Two caveats:
mid-rebase `$mine` is one commit, not your branch — `.git/rebase-merge/orig-head`
is the original tip; and for a cherry-pick the true base is `$landing^` if it
disagrees with the merge-base.

Report the inventory: operation, step N of M, `$mine`/`$other`/`$base` short
SHAs, conflicted paths. Stop and confirm if the state is ambiguous or you are
mid-rebase on a step you did not start. If `rerere.enabled` is set, some hunks
may already be resolved from a past resolution you never saw — those files are
unreviewed, not done.

**If no operation is in flight** there is nothing to resolve. Report the
intended target — `$1` if given (qualify a bare name against the remote; the
remote-tracking ref beats a stale local branch), else
`git symbolic-ref --short refs/remotes/<remote>/HEAD` — and whether HEAD is
behind it, then **ask** before starting anything. Never hardcode `origin/main`,
and never assume the remote is `origin`: in a fork workflow `origin` is the
fork and `upstream` is the real target.

---

## Phase 1 — Understand (READ ONLY — no edits until the triage exists)

Per conflicted file, learn why each side changed:

```bash
git log --oneline -n 10 "$base".."$other" -- <file>   # what the other branch did
git log --oneline -n 10 "$base".."$mine"  -- <file>   # what my side did
git log -p        -n 3  "$base".."$other" -- <file>   # the actual diffs
git show :1:<file>  # base   — what both sides started from
git show :2:<file>  # = $onto
git show :3:<file>  # = $landing
```

The `:1:/:2:/:3:` reads are safe — they touch nothing.
`git checkout --conflict=zdiff3 -- <file>` (diff3 if git < 2.35) renders the
base inline and is nicer to read, but **discards every edit already made to
that file**: never use it where hunks have been hand-resolved. Do not reach for
`git config merge.conflictStyle` instead — it affects only *future* conflicts,
so it does nothing to markers already in the tree, and it mutates the user's
repo config beyond this run.

Read the **whole file**, not just the hunks: a conflict at line 40 is often
only resolvable by knowing what happened at line 300. Note any issue, PR or
ticket the commit messages cite — the intent is usually stated there in a line.

Then classify every hunk:

- **Independent** — the two sides touched different things that happen to sit
  in the same region: two added imports, two new switch cases, two new config
  keys. **This is the common case**; the resolution is the union. One line of
  justification, no table.
- **Overlapping or mutually exclusive** — both sides changed the same constant,
  conditional, signature or algorithm. Write these up:

  | Base | what the code did before either side touched it |
  |---|---|
  | My intent | one sentence + SHA |
  | The other side's intent | one sentence + SHA |
  | Correct combined behavior | one sentence — or "cannot combine" |

  **Mutually exclusive means you stop.** If both sides changed the same thing to
  different ends, you cannot know which is right: present both with your reading
  of each and ask.

Above ~15 conflicted hunks, do not attempt one pass — report the triage, then
work file-by-file. Never resolve a hunk by picking whichever side makes the
build pass; that is a textual fix wearing a semantic costume.

---

## Phase 2 — Resolve

By file type:

- **Lock files** (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`,
  `Cargo.lock`, `poetry.lock`, `go.sum`, `Gemfile.lock`) — resolve the
  *manifest* first (union of additions; higher version on overlap unless a
  commit message shows one side pinned lower deliberately), then take either
  side of the lock to clear markers and **regenerate** with the real package
  manager. Never hand-edit a lock file.
- **Migrations / schema** — do not resolve. Order is load-bearing and a wrong
  guess corrupts data. Report and ask.
- **Generated / vendored** (`*.pb.go`, `dist/`, `*.generated.*`, snapshots) —
  resolve the source, regenerate the artifact. Never merge generated output.
- **Structural** (imports, method lists, struct fields, test blocks) — try
  `mergiraf solve <file>` if installed; it merges on the syntax tree and beats
  you at pure structure. Fall through to manual analysis if markers remain.
- **Stacked-PR duplicates** — empty base section, both sides saying the same
  thing modulo whitespace. Not a real conflict; keep either side and say so. If
  they are merely *similar*, they are not duplicates — show both and ask.
- **Everything else** — write the combined-behavior version from the triage.

---

## Phase 3 — Semantic sweep (the part that matters)

**Git flags textual overlap. The dangerous breakage is in files that merged
cleanly.** If the other branch renamed a function, changed a signature,
tightened a type or renamed a config key, and your side added callers of the
old form, git merges both without complaint and the result is syntactically
valid and behaviorally wrong. Do this every time, even when Phase 2 was trivial.

1. **What did the other branch change structurally?**
   ```bash
   git diff --stat "$base".."$other"
   git diff "$base".."$other" -- '<source globs>'
   ```
   Extract: renamed or removed symbols, changed signatures, parameter order or
   defaults, return types, error-handling contracts, config keys, env var
   names, moved or deleted modules, DB column names.
2. **What did my side add or call?** `git diff --name-only "$base".."$mine"`
3. **Cross-reference.** For each symbol the other branch changed:
   ```bash
   rg -n '<old_symbol>' -g '!node_modules' -g '!dist'
   ```
   Every hit in code my side added or modified is a silent semantic conflict.
   Fix it to the other branch's new contract and report it.

   Mid-rebase the tree is partial — the conflicting commit is applied, but
   commits later in the sequence are not, so their callers are invisible. Also
   search `git grep -n '<old_symbol>' "$(cat .git/rebase-merge/orig-head)"`, and
   re-run the whole sweep after the final step.
4. **Invariants.** Two implementations of the same thing, one from each side?
   Two config blocks setting one key? A flag both sides defined differently? A
   test asserting behavior the other side deliberately changed? Duplication is
   the tell that you took a union where you needed a synthesis.

---

## Phase 4 — Verify and continue

Stop at the first failure.

1. Nothing unresolved: `git diff --name-only --diff-filter=U` empty,
   `git diff --check` clean, and
   `rg -n '^(<<<<<<< |\|\|\|\|\|\|\||>>>>>>> )' -- <files you touched>`.
   Scope that `rg` — a bare `=======` matches ordinary markdown.
2. Build / typecheck, then lint on changed files.
3. **Run each side's tests by name**, not just the suite you thought of. Tests
   the other branch added are the specification for what it intended; if they
   fail you dropped its change. Same for mine.
   ```bash
   git diff --name-only "$base".."$other" -- '*test*' '*spec*' '*_test.*'
   git diff --name-only "$base".."$mine"  -- '*test*' '*spec*' '*_test.*'
   ```
4. A failing test means your intent reconstruction was wrong. Go back to Phase 1
   for the hunk it covers.

Then continue, suppressing the editor so it cannot hang:

```bash
git add -- <resolved files>
GIT_EDITOR=true git rebase --continue     # or: git commit --no-edit
                                          # or: GIT_EDITOR=true git cherry-pick --continue
```

**Loop back to Phase 0** — the next rebase step has its own conflicts. When the
operation finishes, report.

---

## Phase 5 — Report

Not "resolved N conflicts." Per non-trivial hunk (independent ones collapse to
a line per file):

```
<file>:<line>  [independent | synthesized | picked-side | asked-user]
  mine:   <intent>  (<sha>)
  other:  <intent>  (<sha>)
  result: <what the merged code now does>
  dropped: <anything from either side NOT in the result, and why>
```

Then every Phase 3 adaptation made to non-conflicted files, and a closing line
naming whatever you are less than confident about. A non-empty `dropped:` you
did not ask about first is a bug in your process, not a note.

---

## Hard rules

- No edits before the Phase 1 triage exists.
- State which side is `$mine` before touching a hunk. In a rebase, mine is
  `theirs`.
- Never `--ours` / `--theirs` wholesale on a source file — that is choosing a
  side without reading it.
- Never delete a marker block without a stated intent for both sides.
- Never destroy an in-progress manual resolution for a prettier rendering.
- Never edit a test to make it pass.
- Never touch files outside the conflicted set plus the Phase 3 adaptations.
- When genuinely ambiguous, stop and ask. Asking costs a minute; a silently
  dropped fix costs a production incident, and nobody catches it in review
  because the diff looks clean.
