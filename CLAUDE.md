# CLAUDE.md

Guidance for Claude Code working in this repo. See `README.md` for what the
plugins do and how to install them.

## The one rule that bites

**Every commit that changes a shipped file must bump the version, in the same
commit.** Both manifests, kept in agreement:

- `plugins/tbox/.claude-plugin/plugin.json` → `version`
- `.claude-plugin/marketplace.json` → the `tbox` entry's `version`

Shipped files are everything under `plugins/tbox/` — `commands/`, `skills/`,
`agents/`, `templates/`. Editing prose in a command file counts; that *is* the
product here.

Forget it and the failure is silent, not loud: `/plugin update` compares the
manifest version rather than the commit sha, so it reports `already at the latest
version`, copies nothing, and every install keeps running the old file behind a
push that looked fine. Never diagnose "the change didn't take" by re-editing the
command — check the version first.

Verify a release actually landed by reading the version-keyed cache, not the repo:

```bash
ls ~/.claude-personal/plugins/cache/tbox/tbox/          # which versions exist
grep -c "<a phrase from the change>" ~/.claude-personal/plugins/cache/tbox/tbox/<version>/commands/<cmd>.md
```

`~/.claude` and `~/.claude-personal` have separate plugin registries, so a change
can be live in one and stale in the other.

## Editing commands

Command files are instructions to an agent, not documentation for a human. Keep
them short and imperative. Rationale earns its place only where it stops a future
edit from undoing the instruction — one or two lines, not a paragraph.

Some lines are load-bearing verbatim: `/tbox:worktree` ends with the exact line
`Enter the worktree?` because Claude Code's prompt suggestion discards anything
longer or written in Claude's voice. Where a file says to keep wording verbatim,
keep it.

A change to a command's behavior usually needs matching edits in two other
places: the command table in `README.md`, and — for `/tbox:worktree` —
`plugins/tbox/templates/worktrees-README.md`, which describes the commands to
whoever finds the worktrees folder.
