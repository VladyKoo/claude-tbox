---
description: Cross-review a GitHub PR — or, with --local, the uncommitted working tree — with Codex plus an independent Claude pass; auto-posts the PR comment when a verified blocker/major finding survives (suppress with --no-post)
argument-hint: '<pr-number-or-url> | --local [--wait] [--no-post] [--codex-model <id>] [--claude-model fable|opus|sonnet] [extra focus text]'
allowed-tools: Read, Write, Glob, Grep, Agent, Bash(git:*), Bash(gh:*), Bash(node:*), Bash(ls:*), Bash(sort:*), Bash(tail:*), Bash(rm .cross-review-context.md)
---

Cross-review a GitHub PR — or the local uncommitted working tree — with Codex and
with an independent Claude pass.

Raw arguments: `$ARGUMENTS`

## Argument parsing

Two modes, chosen by the arguments:

- **PR mode** (default): the PR number is the first argument that is either a bare
  number or a GitHub PR URL (extract the trailing number from `.../pull/<n>`).
  Below, `<pr>` means the extracted PR number.
- **Local mode**: `--local` is present. The subject is the working tree of the
  current directory — staged, unstaged, and untracked changes relative to `HEAD`.
  Nothing is fetched or checked out, and nothing is ever posted.

`--wait`, `--no-post`, `--codex-model <id>`, `--claude-model <name>`, and any
remaining text may appear in any order; the remaining text is extra review focus.
The token after `--codex-model` or `--claude-model` is its value, never focus text.
`--claude-model` must be one of `fable`, `opus`, `sonnet` — anything else: stop and
say so. If neither `--local` nor a PR number is present, stop and ask for one. If
both are present, stop and ask which was meant.

Below, steps marked **PR mode only** are skipped in local mode; each such step
says what local mode does instead.

Codex always runs `adversarial-review`: it is deliberately biased toward finding
problems (high recall, more noise), and it is the only Codex verb that accepts a
prompt — the plain `review` verb rejects focus text, so it could not be pointed at
the context bundle.

## Model selection

Record which models actually ran — the digest, the draft, and the table headers
name the real reviewers, never the intended ones. The Codex model is the
`--codex-model` value, else `model` from `~/.codex/config.toml` (read it once in
4a); the companion's output does not print it.

| Reviewer | Model | Override |
|---|---|---|
| Claude subagent | `fable` | `--claude-model fable\|opus\|sonnet` |
| Codex | the default from `~/.codex/config.toml` (no `--model` passed) | `--codex-model <id>` |

**Fable unavailable → ask, do not switch.** Dispatch the Claude subagent (4b)
**before** launching Codex (4a), and check the dispatch result first. If the
`Agent` call itself fails (model unavailable, usage limit) before the subagent does
any work, stop and tell the user in two lines: what failed, and that the pass can
run on `opus` instead. Wait for an explicit yes, then re-dispatch the same prompt
with `model: "opus"` and continue with 4a. On "no", stop the run — nothing has been
launched yet. A subagent that ran and returned findings — even thin ones — is a
result, not a failure.

**Codex quota exhausted → second Claude pass.** Codex's usage limit is per account,
not per model, so another Codex model would fail the same way. When the companion
fails before any review output with a usage or rate limit (`usage_limit_exceeded`,
"rate limit", "limit reached", or a reset time), do not retry Codex. Dispatch a
second Claude subagent with the same 4b prompt on a **different** Claude model —
`opus` when the first pass is on `fable` or `sonnet`, `sonnet` when it is on
`opus` — and tell the user in one line, with the reset time if the error gave one.
The second subagent starts from a clean context, so the two passes stay
independent. If that second dispatch also fails, do not ask: continue with the
single Claude pass and label the run single-reviewer everywhere. Any other Codex
failure is reported, not worked around.

## Assumptions about the workspace

- PR mode: the current directory is a git worktree. The PR branch is NOT checked
  out yet.
- Local mode: the current directory is the repo or worktree holding the uncommitted
  changes. It may be on any branch.
- Other worktrees for this repo may exist and may already have branches checked out.
- Never touch or disturb any other worktree.

## Constraints

- **Review only.** Do not fix anything, do not apply patches, do not commit,
  do not push, do not approve or request changes.
- **Posting is gated.** Step 7 drafts a comment; step 8 posts it automatically only
  when the mechanical severity rule there is met and `--no-post` was not passed.
  Outside that rule, posting requires the user to explicitly ask for it in a later
  message — drafting is not permission to post. In local mode there is no PR, so
  nothing is ever posted.
- Do not offer to implement the findings. Just report them.

## Step 1 — check the working tree

Run `git status --short --untracked-files=all`. If a leftover
`.cross-review-context.md` from a prior run exists, delete it now.

**PR mode:** if there are changes to **tracked** files (staged or unstaged), stop and
tell the user what is uncommitted — do not check anything out. Untracked files do not
block a detached checkout: list them for awareness and continue.

**Local mode:** the opposite check — if the status is empty, there is nothing to
review; stop and say so. Otherwise report the counts of staged, unstaged, and
untracked files, plus `git rev-parse --short HEAD` and the current branch. Untracked
files are part of the review. Never stage, stash, or otherwise touch the changes.

## Step 2 — bring the PR into this worktree (PR mode only)

Local mode: skip — the changes are already here.

```bash
git fetch origin pull/<pr>/head
git checkout --detach FETCH_HEAD
```

Use `--detach` deliberately: `gh pr checkout` creates a local branch, and git refuses
if that branch is live in another worktree. Detached HEAD sidesteps that entirely.

If the fetch fails, the PR number is probably wrong — stop and report it. If the
checkout fails because an untracked file would be overwritten, stop and show the
colliding paths — never delete or move the user's files to force the checkout.

Report the resulting commit SHA and subject.

## Step 3 — read what the change claims to do, and build the context bundle

**Local mode** has no PR to read, so skip every `gh` and `git fetch` call in this
step and build the bundle from what is available locally: the **Stated intent**
section is the user's extra focus text if any, otherwise the literal line
`Not stated — judge the diff on its own terms and flag where the intent is unclear.`
**Prior review discussion** and **Linked issue** are `none`. **House rules** and
**Extra focus** are built exactly as below. Then continue at step 4.

**PR mode:**

```bash
gh pr view <pr> --json title,body,baseRefName,headRefName,additions,deletions,changedFiles
gh pr view <pr> --comments
gh api "repos/{owner}/{repo}/pulls/<pr>/comments" --paginate
```

The `gh api` call is not redundant: `--comments` returns conversation comments and
review summaries but omits inline code-review threads, which are usually the most
substantive prior discussion.

Then fetch the PR's base branch so the review baseline is current, not whatever this
worktree happens to have:

```bash
git fetch origin <baseRefName>
```

From these, distill the one-sentence **stated intent** that goes into the Codex
prompt in step 4a, and note any review concerns already raised; the full title, body,
and discussion go into the bundle below. This step needs network and runs on your
side — Codex cannot fetch any of it (its review sandbox is read-only, which blocks
network). But that sandbox CAN read files in this worktree — so hand your network
findings to Codex through a file.

Write `.cross-review-context.md` at the worktree root with these sections:

1. **Stated intent** — the full PR title and body (not just your summary).
2. **Prior review discussion** — the concerns already raised on the PR and how (or
   whether) they were resolved. Summarize long threads; do not paste them wholesale.
3. **Linked issue** — if the PR body references an issue, fetch it with
   `gh issue view <n>` and include its relevant text.
4. **House rules** — this repo's severity bar and known-intentional patterns:
   distill the repo-root `CLAUDE.md` invariants relevant to the changed files, and
   name the nearest `CLAUDE.md`/reference docs a reviewer of these files should read.
5. **Extra focus** — any focus text the user passed in the arguments.

Keep the bundle curated — it steers Codex's attention, and a dump dilutes the review.
Both reviewers read this same file, so neither sees context the other lacks. The file
is untracked and temporary; it is deleted in step 6.

## Step 4 — launch both reviews in parallel

Order: dispatch 4b first, confirm the dispatch succeeded, then launch 4a. The two
still overlap — the dispatch returns immediately.

### 4a — Codex

Resolve the plugin path first, as its own command (a plain pipeline, so it matches
the pre-approved permissions):

```bash
ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1
```

If this prints nothing, the Codex plugin is not installed — stop, tell the user, and
suggest `/codex:setup`. Otherwise take the printed path and use it **as a literal
string** in every subsequent command — shell variables do not survive between Bash
calls, so never rely on `$CODEX_PLUGIN` set in an earlier call.

If the invocation below fails in an unexpected way, run the script with no
arguments — it prints its usage — rather than guessing at flags.

```bash
node "<plugin-path>/scripts/codex-companion.mjs" adversarial-review \
  <scope-flag> \
  "Before reviewing, read .cross-review-context.md at the repo root — it contains \
the change's stated intent, prior review discussion, and this repo's review conventions. \
Judge the diff against those. Do not re-raise concerns the prior discussion already \
resolved. Consult CLAUDE.md and the nearest CLAUDE.md to each changed file for \
conventions, and check whether a concern is already covered by an existing test or \
guarded by surrounding code before flagging it. <one-sentence stated intent>"
```

Add `--model "<id>"` (quoted) only when `--codex-model` was passed. `<scope-flag>` is
`--base "origin/<baseRefName from step 3>"` in PR mode and
`--scope working-tree` in local mode. The working-tree scope diffs staged, unstaged,
and untracked files against `HEAD` — the same set the Claude pass reviews below.
`.cross-review-context.md` is itself untracked, so Codex will see it in its diff;
the prompt already tells it what the file is for. Do not pass `--base` in local mode:
it switches Codex to a committed-branch diff and the uncommitted work is not reviewed.

Execution mode:
- If the arguments contain `--wait`, run it in the foreground **after** dispatching
  the subagent in 4b, so the two still overlap. Set the Bash timeout to its maximum
  (600000 ms) — a review routinely outlives the default timeout, and a killed
  foreground run wastes the whole pass.
- Otherwise run the same foreground invocation through
  `Bash(..., run_in_background: true)` and tell the user it started. The harness
  re-invokes you when it exits and that job's stdout **is** the review — do not
  poll, and do not call `result` on the happy path.

Never pass the companion's own `--background` flag. It returns a job id immediately
and exits, so the Bash job completes in a second with no review in it. `result` and
`status` are fallbacks for exactly that case — if the stdout comes back as a job id,
fetch the review with `node "<plugin-path>/scripts/codex-companion.mjs" result`.

The context bundle plus the focus text is how Codex judges the diff against what it
claims to do — and against this repo's actual conventions — not just against itself.

### 4b — independent Claude pass, as a subagent

Do NOT review the diff in this context: by now you have read the PR discussion and
will soon see Codex's output, so a pass done here is not independent. Dispatch a
`general-purpose` subagent instead, with `model` set per Model selection — it
starts with a clean context and, running in the background, finishes without ever
seeing Codex's findings.

Send it a prompt with exactly this information and nothing more:

- The worktree path and what to diff. PR mode: the detached head SHA and the base
  ref, with the instruction to review `git diff origin/<baseRefName>...HEAD` in that
  directory. Local mode: the HEAD SHA, with the instruction to review `git diff HEAD`
  (staged plus unstaged) and to read in full every file listed by
  `git ls-files --others --exclude-standard`, except `.cross-review-context.md`.
- The path to `.cross-review-context.md`, to be read first — the same context Codex
  gets, so the two passes differ in judgment, not in information.
- Ground rules: **read-only** — no edits, no commits, no checkouts, no staging or
  stashing, no network, and do not invoke anything under `~/.claude/plugins` (that
  is the other reviewer).
- Required output: a list of findings, each with `file:line`, severity
  (blocker/major/minor/nit), a one-paragraph description, and the concrete failure
  case — plus a list of PR claims it verified as holding. If the diff is clean, say
  so plainly. Its final message is raw review data, not prose for the user.

## Step 5 — collect both results

Wait for both reviewers to complete — the Codex run and the subagent, or, when
Codex failed on quota, the two subagents (their completion notifications arrive on
their own — do not busy-poll either one). Do not read
either result until both are in hand, and form no conclusions of your own in the
meantime — in this command your role is synthesis and verification, not a third
review.

## Step 6 — report

First, delete the temporary context bundle so it cannot leak into later work in this
worktree:

```bash
rm .cross-review-context.md
```

### Do the full synthesis; print almost none of it

Read both results in full, reconcile them, and verify the contested calls against
the code. That analysis is the substance of this command — but it belongs in the
**step 7 draft file**, not in the session. The terminal gets a scannable digest;
the draft carries the long form. Never paste either reviewer's output verbatim into
the session: the draft preserves every finding, so nothing is lost by not printing
it twice.

**Dismissals require evidence, not opinion.** Before calling either reviewer's
finding "not an issue" or "not worth fixing", verify it against the code: read the
actual path the finding describes and cite the specific guard, caller contract, or
existing test (`file:line`) that makes the failure impossible or the concern already
covered. "Seems unlikely" or "in practice this won't happen" without a citation is
not a dismissal — if you cannot produce the evidence, downgrade your position to
"unverified, leaning disagree" and say so. The same bar applies in reverse: do not
confirm a finding you have not traced yourself.

Disagreement is the point of this command. Do not smooth it over, do not average the
two opinions, and do not drop a finding just because you disagree with it — every
finding either reviewer raised survives into the draft with its original severity.

### The session digest

Print exactly these five things and nothing else. No verbatim reviewer output, no
per-finding prose, no claims-verified list, no fenced copy of the draft.

1. **Context line** — PR mode: `<short-sha>` vs `<baseRef>`, `+/−` and file count.
   Local mode: `working tree at <short-sha> (<branch>)`, with the staged/unstaged/
   untracked counts. Both: the reviewers that actually ran, e.g.
   `Codex gpt-6-sol + Claude fable`, or `Codex unavailable (quota, resets <time>)
   — Claude fable + Claude opus`.
2. **Bottom line** — one or two sentences: does anything block merge, and what is
   the single most important thing to act on.
3. **Findings table** — one row per finding, agreed findings first:

   | # | Finding | Codex | Claude | Verdict |
   |---|---------|-------|--------|---------|

   In the Claude-only fallback the columns are the two Claude models instead.
   Short title, each reviewer's severity (`—` if they missed it), short-phrase
   verdict. This table is the digest's payload — a user should scan it and know
   what happened without opening anything.
4. **Disagreements** — one line each, only where the reviewers conflict or where you
   overrode a severity, naming which call is right. If they agreed throughout, say
   "no disagreements" and move on.
5. **Worktree state** — path and detached SHA (PR mode) or path, branch, and HEAD
   SHA (local mode), one line.

If both reviewers came back clean, say so in two sentences and skip the table.

## Step 7 — write the review draft

Write the draft to `<scratchpad>/cross-review-pr-<pr>.md` (PR mode) or
`<scratchpad>/cross-review-local-<repo-dir-name>-<short-sha>.md` (local mode) so it
survives and can be posted, or re-read after fixes, without regenerating it.

**Do not print the draft.** It is long by design — full finding sections, failure
cases, and the claims-verified list — and echoing it into the session is exactly the
bulk the step 6 digest exists to avoid. The user reads it in the file, or on the PR
once it is posted. Write it, then decide posting in step 8.

Start the draft file with an HTML comment recording exactly what was reviewed, so a
later "post it" can be staleness-checked:

```markdown
<!-- cross-review: pr=<pr> head=<full-40-char-sha> base=<baseRefName> -->
```

In local mode the header is instead:

```markdown
<!-- cross-review: local head=<full-40-char-sha> branch=<branch> tree=<git-status-hash> -->
```

where `<git-status-hash>` is the output of `git diff HEAD | git hash-object --stdin`,
so a later reader can tell whether the tracked changes have moved since the review.

Take the full SHA from `git rev-parse HEAD` — earlier steps reported only the short form.

Draft shape — this is the long-form artifact, and it is read by people who did not
run the review, so it carries the full reasoning the session digest omitted.
Optimize for scannability, not brevity: a reader should get the verdict in ten
seconds from the table and be able to jump straight to any finding's detail.

```markdown
# Cross-review: Codex + Claude

Reviewed `<short-sha>` against `<baseRef>` (<+/-, files>). Two independent passes —
OpenAI Codex `<codex-model>` (adversarial) and Claude `<claude-model>` — with
disagreements surfaced rather than merged.
<!-- Claude-only fallback: intro becomes "OpenAI Codex was unavailable (<reason>);
two independent Claude passes — `<model A>` and `<model B>` — with disagreements
surfaced rather than merged." Then replace EVERY "Codex" / "Claude" label in the
draft — the H1, the bottom line, the table columns, the "Codex only / Claude only"
section suffixes, the "Claude's read:" label, and both footers (drop "+ OpenAI
Codex") — with the two model names, e.g. "Claude fable" / "Claude opus". -->
<!-- local mode: "Reviewed the uncommitted working tree at `<short-sha>` (<branch>;
<n> staged, <n> unstaged, <n> untracked files)." — keep the rest of the sentence. -->
<!-- local mode: rename "Claims verified as holding" to "Intent verified as
holding", or omit it when no intent was stated. -->
<!-- local mode: replace the last line with
"🤖 Cross-review by Claude Code + OpenAI Codex — local working tree, not posted." -->

**Bottom line:** <Codex's verdict> vs. <Claude's verdict>, in one or two sentences —
what (if anything) must change before merge, and what is merely worth doing.

| # | Finding | Codex | Claude | Verdict |
|---|---------|-------|--------|---------|
| 1 | <short title> | high | agrees | **fix before merge** |
| 2 | <short title> | high | disagrees | <one-phrase resolution> |

---

### 1. <Finding title as a sentence> — both reviewers / Codex only / Claude only

📍 `path/to/file.ts:42`

One short paragraph: what the issue is.

**Failure case:** the concrete sequence that goes wrong (or **Mechanism:** /
**Claude's read:** when the section is about a disagreement).

**Fix (cheap/involved):** the concrete change that resolves it.

### 2. <Next finding — same structure>

...

---

### Not blocking

- `path/to/file.ts:99` — minor note, nit, or a follow-up that depends on a fix above.

### Claims verified as holding

- One bullet per PR claim that was independently checked and held up.

---
🤖 Cross-review by [Claude Code](https://claude.com/claude-code) + OpenAI Codex
```

Rules for the draft:
- Every finding cites `file:line`. No vague "consider refactoring the handler".
- **Readability over density.** One `###` section per finding, numbered to match the
  summary table. Inside a section use short labeled paragraphs (**Failure case:**,
  **Mechanism:**, **Claude's read:**, **Fix:**) — never a single multi-clause bullet
  that packs the claim, the rebuttal, the severity call, and the fix into one sentence.
- Order the table and sections by confidence: agreed findings first — those are the
  ones worth acting on.
- Disagreements stay visible: keep each reviewer's original severity in the table,
  state the opposing read in the section body, and give a one-phrase resolution in the
  Verdict column. Do not average the two opinions.
- Omit any section that is empty (including the table if there are ≤1 findings).
  Do not pad.
- If both reviewers came back clean, say so in two sentences and skip the sections.
- Attribute honestly. Do not present a Codex finding as Claude's, or vice versa.
- No praise padding, no "great work overall" preamble.

## Step 8 — post or hold

**Local mode always holds** — there is no PR to post to. Skip the rule below, close
with the one-line "Draft saved" form, and never suggest committing or opening a PR
to make the draft postable.

Posting is decided by a mechanical rule, not by taste. **Auto-post** the draft iff
all of these hold:

- `--no-post` was NOT in the arguments;
- at least one finding survived step 6 synthesis at **blocker or major** severity;
- that finding is either one **both reviewers raised**, or one **you confirmed
  against the code** in step 6 (with the `file:line` evidence the dismissal rule
  demands). A major raised by a single reviewer that you could not verify does not
  qualify — adversarial mode is deliberately noisy, and an unverified solo major is
  exactly its noise profile.

Everything below the bar — clean runs, minor/nit-only runs, unverified solo
majors — is **held**: the draft stays in the file and posting waits for an explicit
instruction.

**If posting:** run `gh pr comment <pr> --body-file <path>`. No staleness check is
needed — the review just ran against the current head. Then close with exactly these
two lines, the last thing in your reply, immediately after the step 6 digest:

> Posted to PR #<pr>: <comment-url>
> Draft saved to `<path>`

**If holding:** close with exactly this one line instead:

> Draft saved to `<path>`

In both cases `<path>` is the **full absolute path**. Never abbreviate it with
`...`, never shorten the scratchpad prefix, never substitute `~`. The digest is
terse everywhere else, but this path is the one thing the user copies into a shell
or an editor — a truncated path is unusable.

Nothing follows those lines. When holding, do not explain why the threshold was not
met beyond one short clause if it would otherwise be surprising, do not spell out
the posting command, and do not ask a yes/no question that could be read as
auto-posting on a bare "yes" — wait for an explicit instruction to post.

## When the user later says to post a held draft

This runs only on an explicit instruction in a later message — never as a retry of
the step 8 decision.

1. Read the `<!-- cross-review: ... -->` header from the draft file.
2. Check the PR hasn't moved since the review:
   ```bash
   gh pr view <pr> --json headRefOid
   ```
3. If `headRefOid` still equals the recorded `head` SHA, post with
   `gh pr comment <pr> --body-file <path>`.
4. If it differs, do NOT post. Tell the user the PR has new commits since the review
   (show both SHAs) and offer either to re-run the cross-review against the new head,
   or — only if they explicitly insist — to post the stale draft with a first line
   noting which commit it reviewed.
