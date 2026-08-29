---
description: Cross-review a GitHub PR with Codex plus an independent Claude pass, in the current master-based worktree; auto-posts the comment when a verified blocker/major finding survives (suppress with --no-post)
argument-hint: '<pr-number-or-url> [--wait] [--regular] [--no-post] [extra focus text]'
allowed-tools: Read, Write, Glob, Grep, Agent, Bash(git:*), Bash(gh:*), Bash(node:*), Bash(ls:*), Bash(sort:*), Bash(tail:*), Bash(rm .cross-review-context.md)
---

Cross-review a GitHub PR with Codex and with an independent Claude pass.

Raw arguments: `$ARGUMENTS`

## Argument parsing

The PR number is the first argument that is either a bare number or a GitHub PR
URL (extract the trailing number from `.../pull/<n>`). `--wait`, `--regular`,
`--no-post`, and any remaining text may appear in any order; the remaining text is
extra review focus. If no PR number can be extracted from any argument, stop and ask for it.
Below, `<pr>` means the extracted PR number.

`--regular` selects Codex's standard `review` verb instead of the default
`adversarial-review`. Adversarial is deliberately biased toward finding problems
(high recall, more noise); regular is a balanced pass with fewer speculative
findings. Below, `<review-verb>` means whichever was selected.

## Assumptions about the workspace

- The current directory is a git worktree that was created **from master**. The PR
  branch is NOT checked out yet.
- Other worktrees for this repo may exist and may already have branches checked out.
- Never touch or disturb any other worktree.

## Constraints

- **Review only.** Do not fix anything, do not apply patches, do not commit,
  do not push, do not approve or request changes.
- **Posting is gated.** Step 7 drafts a comment; step 8 posts it automatically only
  when the mechanical severity rule there is met and `--no-post` was not passed.
  Outside that rule, posting requires the user to explicitly ask for it in a later
  message — drafting is not permission to post.
- Do not offer to implement the findings. Just report them.

## Step 1 — refuse to clobber uncommitted work

Run `git status --short --untracked-files=all`. If there are changes to **tracked**
files (staged or unstaged), stop and tell the user what is uncommitted — do not check
anything out. Untracked files do not block a detached checkout: list them for
awareness and continue. If a leftover `.cross-review-context.md` from a prior run
exists, delete it now.

## Step 2 — bring the PR into this worktree

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

## Step 3 — read what the PR claims to do, and build the context bundle

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

Write a 2-4 sentence summary of the PR's **stated intent**, and note any existing
review comments or concerns already raised. This step needs network and runs on your
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

The calling contract for `codex-companion.mjs` is owned by the plugin's
`codex:codex-cli-runtime` skill; if the invocation below fails in an unexpected way,
consult that skill rather than guessing at flags.

```bash
node "<plugin-path>/scripts/codex-companion.mjs" <review-verb> \
  --base "origin/<baseRefName from step 3>" \
  "Before reviewing, read .cross-review-context.md at the repo root — it contains \
the PR's stated intent, prior review discussion, and this repo's review conventions. \
Judge the diff against those. Do not re-raise concerns the prior discussion already \
resolved. Consult CLAUDE.md and the nearest CLAUDE.md to each changed file for \
conventions, and check whether a concern is already covered by an existing test or \
guarded by surrounding code before flagging it. <one-sentence stated intent>"
```

Execution mode:
- If the arguments contain `--wait`, run it in the foreground **after** dispatching
  the subagent in 4b, so the two still overlap. Set the Bash timeout to its maximum
  (600000 ms) — a review routinely outlives the default timeout, and a killed
  foreground run wastes the whole pass.
- Otherwise run it via `Bash(..., run_in_background: true)` and tell the user it
  started. Background commands re-invoke you when they exit, so do **not** poll in a
  loop — once notified of completion, fetch the output with
  `node "<plugin-path>/scripts/codex-companion.mjs" result`. Use
  `node "<plugin-path>/scripts/codex-companion.mjs" status` only as a manual fallback
  if no completion notification arrives.

The context bundle plus the focus text is how Codex judges the diff against what it
claims to do — and against this repo's actual conventions — not just against itself.

### 4b — independent Claude pass, as a subagent

Do NOT review the diff in this context: by now you have read the PR discussion and
will soon see Codex's output, so a pass done here is not independent. Dispatch a
`general-purpose` subagent instead — it starts with a clean context and, running in
the background, finishes without ever seeing Codex's findings.

Send it a prompt with exactly this information and nothing more:

- The worktree path, the detached head SHA, and the base ref, with the instruction
  to review `git diff origin/<baseRefName>...HEAD` in that directory.
- The path to `.cross-review-context.md`, to be read first — the same context Codex
  gets, so the two passes differ in judgment, not in information.
- Ground rules: **read-only** — no edits, no commits, no checkouts, no network, and
  do not invoke anything under `~/.claude/plugins` (that is the other reviewer).
- Required output: a list of findings, each with `file:line`, severity
  (blocker/major/minor/nit), a one-paragraph description, and the concrete failure
  case — plus a list of PR claims it verified as holding. If the diff is clean, say
  so plainly. Its final message is raw review data, not prose for the user.

## Step 5 — collect both results

Wait for both the Codex run and the subagent to complete (their completion
notifications arrive on their own — do not busy-poll either one). Do not read
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

### The session digest — target ~20 lines, hard cap 30

Print exactly these five things and nothing else. No verbatim reviewer output, no
per-finding prose, no claims-verified list, no fenced copy of the draft.

1. **Context line** — `<short-sha>` vs `<baseRef>`, `+/−` and file count, and which
   Codex mode ran (adversarial or regular) so the noise level is legible.
2. **Bottom line** — one or two sentences: does anything block merge, and what is
   the single most important thing to act on.
3. **Findings table** — one row per finding, agreed findings first:

   | # | Finding | Codex | Claude | Verdict |
   |---|---------|-------|--------|---------|

   Short title, each reviewer's severity (`—` if they missed it), short-phrase
   verdict. This table is the digest's payload — a user should scan it and know
   what happened without opening anything.
4. **Disagreements** — one line each, only where the reviewers conflict or where you
   overrode a severity, naming which call is right. If they agreed throughout, say
   "no disagreements" and move on.
5. **Worktree state** — path and detached SHA, one line.

If both reviewers came back clean, say so in two sentences and skip the table.

## Step 7 — write the PR comment draft

Write the draft to `<scratchpad>/cross-review-pr-<pr>.md` so it survives and can be
posted later without regenerating it.

**Do not print the draft.** It is long by design — full finding sections, failure
cases, and the claims-verified list — and echoing it into the session is exactly the
bulk the step 6 digest exists to avoid. The user reads it in the file, or on the PR
once it is posted. Write it, then decide posting in step 8.

Start the draft file with an HTML comment recording exactly what was reviewed, so a
later "post it" can be staleness-checked:

```markdown
<!-- cross-review: pr=<pr> head=<full-40-char-sha> base=<baseRefName> -->
```

Take the full SHA from `git rev-parse HEAD` — step 2 reported only the short form.

Draft shape — this is the long-form artifact, and it is read by people who did not
run the review, so it carries the full reasoning the session digest omitted.
Optimize for scannability, not brevity: a reader should get the verdict in ten
seconds from the table and be able to jump straight to any finding's detail.

```markdown
# Cross-review: Codex + Claude

Reviewed `<short-sha>` against `<baseRef>` (<+/-, files>). Two independent passes —
OpenAI Codex (<adversarial or regular, per the selected verb>) and Claude — with
disagreements surfaced rather than merged.

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
