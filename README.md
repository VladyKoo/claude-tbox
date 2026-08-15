# claude-tbox

Personal Claude Code plugin marketplace.

## Install

```
/plugin marketplace add VladyKoo/claude-tbox
/plugin install tbox@tbox
```

Or from a local checkout:

```
/plugin marketplace add ./claude-tbox
/plugin install tbox@tbox
```

If the install summary says `Run /reload-plugins to activate.`, run that.

Repeat in each config dir you use — `~/.claude` and `~/.claude-personal`
(`claude-personal` alias) have separate plugin registries.

## Plugins

### `tbox` — worktree lifecycle + layered PR review

| Command | Purpose |
| --- | --- |
| `/tbox:worktree <task> [project]` | Create a worktree off the remote default branch in the central worktrees folder, register it in the enclosing `.code-workspace`, and offer to enter it |
| `/tbox:worktree-remove [task] [project]` | Remove that worktree, its branch, and its workspace entry |
| `/tbox:cross-review <pr> [--wait] [--regular]` | Review a PR with Codex plus an independent Claude pass |
| `/tbox:cross-review-n <pr> [cycles=3]` | Run cross-review N times with rotating focus lenses, then vote-count the findings |
| `/tbox:audit-review [pr\|file] [--all]` | Audit an existing review — verify each finding against the code, then produce a fix plan |

Typical flow: `/tbox:worktree fix-123` → `/tbox:cross-review 123` → `/tbox:audit-review 123`.

**Dependency:** `/tbox:cross-review` and `/tbox:cross-review-n` drive Codex through
the [`codex`](https://github.com/openai/codex-plugin-cc) plugin, declared in
`dependencies` and allowlisted via `allowCrossMarketplaceDependenciesOn`. It is
installed automatically provided the `openai-codex` marketplace is already
added.

**Worktree location:** `${WORKTREES_DIR:-$HOME/.worktrees}/<project>/<task>`.
Export `WORKTREES_DIR` to relocate.

`${CLAUDE_PLUGIN_ROOT}/templates/worktrees-README.md` is the concept doc that
`/tbox:worktree` seeds into the worktrees root on first use, verbatim. It is plain
data, not a plugin component, so no manifest field references it. Edit it there
and nowhere else — this plugin is the only thing that writes it.

## Release

**Any commit that changes a shipped file bumps the version.** A command, a skill,
an agent, a template — if it lands in the plugin, the same commit bumps `version`
in **both** `plugins/tbox/.claude-plugin/plugin.json` and the marketplace entry in
`.claude-plugin/marketplace.json`. They must agree.

This is not bookkeeping. `/plugin update` compares the manifest version, **not the
commit sha**: with the version unchanged it answers `already at the latest version`
and copies nothing, so the push looks successful and every install silently keeps
running the old file. The plugin cache is version-keyed
(`cache/<marketplace>/<plugin>/<version>/`), which is also how you tell by eye
which copy is live.

Then:

```
claude plugin validate ./plugins/tbox --strict
claude plugin tag --push
```

Consumers pick it up via `/plugin marketplace update tbox` then
`/plugin update tbox@tbox`, and a session restart — commands and skills are read
at startup.
