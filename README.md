# claude-tbox

A small Claude Code plugin marketplace: PR and OpenSpec review that pairs Codex
with an independent Claude pass, plus a couple of review and merge helpers.

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

If you run Claude Code with more than one config dir (`CLAUDE_CONFIG_DIR`),
install in each — they have separate plugin registries.

### Requirements

- [`gh`](https://cli.github.com/), authenticated — PR mode reads and comments on PRs.
- The [`codex`](https://github.com/openai/codex-plugin-cc) plugin and a working
  Codex CLI login — see **Dependency** below.
- [OpenSpec](https://github.com/Fission-AI/OpenSpec) CLI on `PATH`, for
  `/tbox:openspec-review` only.

### Before you run it

- **It can post to your PR.** `/tbox:cross-review` and `/tbox:cross-review-n`
  in PR mode post a review comment on their own when a verified blocker/major
  finding survives. Pass `--no-post` to keep everything local. Branch mode (no PR
  number) never posts.
- **It spends quota.** Every run uses both Codex and Claude; `cross-review-n`
  multiplies that by the number of cycles.

## Plugins

### `tbox` — layered review + merge help

| Command | Purpose |
| --- | --- |
| `/tbox:cross-review [pr] [--wait] [--no-post] [--codex-model <id>] [--claude-model <name>]` | Review a PR — or, with no PR number, the current branch's commits plus uncommitted work — with Codex plus an independent Claude pass, in a temporary detached worktree it removes afterwards; your checkout is never touched, so it runs from `main` or mid-work. Branch mode never auto-posts. Claude pass on `fable` (asks before switching to `opus` if unavailable), Codex on its configured default model; runs a second Claude pass instead when the Codex quota is exhausted |
| `/tbox:cross-review-n [pr] [cycles=3] [--no-post] [--codex-model <id>] [--claude-model <name>]` | Run cross-review N times with rotating focus lenses, then vote-count the findings; same model selection as cross-review, carried across cycles |
| `/tbox:audit-review [pr\|file] [--all]` | Audit an existing review — verify each finding against the code and decide what to fix, skip, push back on, or ask about |
| `/tbox:openspec-review [change-id] [--wait] [--codex-model <id>] [--claude-model <name>]` | Cross-review an OpenSpec change with Codex plus an independent Claude pass — validate it, verify every claim it makes about the code, and report; same model selection as cross-review |
| `/tbox:smart-merge [target-branch]` | Resolve merge/rebase conflicts semantically — reconstruct both sides' intent, then sweep the cleanly-merged files for breakage git never flagged |

Typical flow: `/tbox:cross-review 123` from any checkout → fix → reply → repeat. On your own branch, `/tbox:cross-review` with no number reviews it before or after the PR exists.

`/tbox:audit-review` earns its keep when the review came from someone else — a human
reviewer, Codex, Copilot, CodeRabbit — and you want each finding verified against the
code before you act on it. Across passes it tells settled threads from active ones:
settled threads are spot-checked for regressions, active ones get full verification and a
drafted reply.

For spec-driven work: write the change, then `/tbox:openspec-review <change-id>` before
implementing it. It is review-only — it never edits the change, and writes its report to
the scratchpad rather than into the change directory, which gets archived.

`/tbox:smart-merge` is for the conflicts where a clean-looking resolution is the
failure mode: it reconstructs what each side meant before editing anything, and its
Phase 4 sweep looks for callers your branch added against signatures the target branch
changed — breakage that merges without a single conflict marker.

**Dependency:** `/tbox:cross-review`, `/tbox:cross-review-n`, and `/tbox:openspec-review` drive Codex through
the [`codex`](https://github.com/openai/codex-plugin-cc) plugin, declared in
`dependencies` and allowlisted via `allowCrossMarketplaceDependenciesOn`. It is
installed automatically provided the `openai-codex` marketplace is already
added.

## Release

**Any commit that changes a shipped file bumps the version.** A command, a skill,
an agent — if it lands in the plugin, the same commit bumps `version`
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

## License

MIT — see [LICENSE](LICENSE).
