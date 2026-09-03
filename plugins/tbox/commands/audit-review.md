---
description: Audit a code review — verify every finding against the real code and decide, per finding, whether to fix it, skip it, push back, or ask. Investigation only; writes no code.
argument-hint: '[pr-number | file-path | @file] [--all]'
disable-model-invocation: true
allowed-tools: Read, Write, Glob, Grep, Agent, Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh api graphql:*), Bash(gh api user:*), Bash(git log:*), Bash(git show:*), Bash(git diff:*), Bash(git rev-parse:*), Bash(git merge-base:*), Bash(git status:*), Bash(ls:*), Bash(rg:*)
---

You are auditing a code review, not performing one. Investigation only: the
one file you may write is the report under REPORT.

Raw arguments: `$ARGUMENTS`

Order of work: ANCHOR → INPUT → NORMALIZE → VERIFY → DECIDE → REPORT.

## ANCHOR

Before reading any finding, record what you are verifying against:

- `git rev-parse HEAD` — every verdict refers to this SHA (full form in the
  report header, short form in the digest).
- PR mode: `gh pr view <N> --json headRefOid`, then
  `git merge-base --is-ancestor <pr-head> HEAD`.
  - Ancestor: local is ahead of the PR. Say so in one line — `fixed` verdicts
    may reflect unpushed work — and continue.
  - Not an ancestor, or the commit is unknown locally: this checkout does not
    contain the PR. Stop and ask me to switch to the matching worktree; do not
    audit a PR against unrelated code.
- `git status --porcelain` non-empty: note that uncommitted changes are in
  play — a finding may read as fixed by work not yet committed.

This becomes the digest's first line:
`Verified at <sha> (PR head <sha>) — <clean|N uncommitted files>`.
In file mode drop the parenthetical.

## INPUT

The review is ONE of:

(a) A file path or @-mention → read it. Every finding is ACTIVE; skip the
    classification below.

(b) PR comments. No PR number → the PR for the current branch
    (`gh pr view --json number`); none there either → ask what to audit and
    stop.
    - Top-level: `gh pr view <N> --comments`
    - Threads, with resolution and staleness:
      ```
      gh api graphql --paginate -f query='
        query($owner:String!, $repo:String!, $number:Int!, $endCursor:String) {
          repository(owner:$owner, name:$repo) {
            pullRequest(number:$number) {
              reviewThreads(first:100, after:$endCursor) {
                pageInfo { hasNextPage endCursor }
                nodes {
                  isResolved isOutdated path line originalLine
                  comments(first:100) { nodes { author{login} body path line } }
                }
              }
            }
          }
        }' -F owner=<owner> -F repo=<repo> -F number=<N>
      ```
      `<owner>`/`<repo>` come from `gh pr view <N> --json url`. `line` is null
      on outdated threads; use `originalLine` for the anchor. Every thread
      counts — never audit a truncated list.

Classify each thread (my login: `gh api user --jq .login`):

- **ACTIVE** — the last comment is not mine. The reviewer is waiting.
- **SETTLED** — my reply is last, or the thread is resolved.

ACTIVE threads get full VERIFY + DECIDE. SETTLED threads get a one-line
spot-check — does the fix still hold at HEAD? — under "Carried forward"; a
rebase or later commit can silently drop a fix nobody re-reads. A regression
is promoted to ACTIVE. `--all` re-audits every thread.

## NORMALIZE

Extract discrete findings, numbered `F<m>`. Split a paragraph raising three
issues into three findings. Merge the same claim about the same code from two
sources (commonly a bot and a human) into ONE finding listing both. Drop pure
praise and pure style opinion with no claim attached; list them once under
"Non-findings".

Per finding, capture: author, ACTIVE|SETTLED, resolved y/n, outdated y/n, and
the anchored file:line.

If nothing survives — the review is all praise or style opinion — say so in one
line, write no report, and stop.

Print the list — one line per finding: ID, short title, author, state. No
comment text. If you split or merged anything, **stop** and wait for me to
confirm before verifying. Otherwise continue.

## VERIFY

Rules:

1. Every finding is an UNVERIFIED CLAIM: it may be wrong, outdated, based on
   hallucinated code, or already fixed. Confirm only by reading the code.
2. Every verdict cites `path/file.ext:LINE` — the code you found while
   verifying, not the review's anchor; a top-level comment with no anchor is
   still verifiable. For missing behaviour, cite the symbol that should have
   it, its callers or tests, and the scope you searched. UNVERIFIABLE only
   when the claim cannot be mapped to code after a real search — never as a
   polite way to avoid looking.
3. `isOutdated` and resolved threads are hints, not verdicts. Verify against
   HEAD. Weight human comments above bot/AI ones as a prior, verify both, and
   name the author on each verdict.

Write the long form to the report, not the session:

  ### F<m>: <short title>
  - Source: <file | @author, ACTIVE|SETTLED, resolved: y/n, outdated: y/n;
    n/a for both in file mode>
  - Claim: <one line>
  - Evidence: <file:line + the smallest excerpt(s) that establish it — more
    than one location when the behaviour spans files>
  - Verdict: <below>
  - Reasoning: <from what you read>
  - Real severity: critical | high | medium | low | nitpick — your own; ignore
    the review's
  - Decision: <from DECIDE> — <one-line reason>

Verdicts, first match wins:

- **UNVERIFIABLE** — could not be mapped to code after a real search.
- **REJECTED (<reason>)** — no fix needed. Reason is one of: `false` (code
  does not do what is claimed — cite the disproving line), `fixed` (was true,
  now correct — cite the fixing line), `stale` (anchored code moved or is gone
  — say where), `n/a` (real and unchanged, but out of scope or intentional —
  say which).
- **PARTIALLY CORRECT** — real problem, misstated claim (cause, severity, or
  location). State the corrected version; that is what DECIDE judges.
- **CONFIRMED** — accurate as written.

Then a summary table (ID | Title | Source | Verdict | Severity | Decision),
"Carried forward", and "Findings not in the review" — issues hit while
verifying, not a fresh review pass; they get a verdict and decision too.

## DECIDE

Every finding gets exactly one decision. This is your call — make it, do not
hedge — and I may override it.

- **FIX** — CONFIRMED or PARTIALLY CORRECT, and either severity ≥ medium or
  the fix is trivial (a few lines, in code this PR already touches). Give the
  fix in one or two sentences naming the function or file; no diff, no plan.
- **SKIP** — CONFIRMED or PARTIALLY CORRECT, but low/nitpick with a cost out
  of proportion, or a pre-existing issue this PR did not make worse. Say why
  in one line; the reply acknowledges it and states the reason.
- **PUSH BACK** — REJECTED. The reply says why, pointing at the evidence.
- **ASK** — UNVERIFIABLE. The reply asks the reviewer for the location, repro,
  or the missing half of the claim.

When several FIX findings touch the same code, note which one to do first.

End the report with "Replies to post": one line per ACTIVE thread, carrying
the decision in the reviewer's terms (agreed and will fix / won't fix because /
disagree because / could you point to). Draft only — do not post.

## REPORT

Write everything above to `<scratchpad>/audit-review-<target>-<sha>.md`, where
`<target>` is `pr-<number>` or the review file's basename, and `<sha>` is the
short HEAD. If the path exists, append `-2`, `-3`, …; never replace it. Start
it with:

```markdown
<!-- audit-review: target=<pr-or-file> head=<full-40-char-sha> -->
```

Do not print the report. Print only this digest — target 15 lines, hard cap
25, the table exempt:

1. Context line from ANCHOR, plus `<a> active / <s> settled`.
2. Bottom line — one or two sentences: how many FIX, how much of the review
   survived.
3. Table, FIX first: `| ID | Finding | Source | Verdict | Severity | Decision |`.
   One row per finding, however many. If every finding was REJECTED, two
   sentences instead of the table.
4. Findings not in the review — count and worst severity, only if any.
5. Carried forward — "no regressions" or the regressed IDs; PR mode with
   SETTLED threads only.
6. Replies to post — verbatim, for pasting into GitHub. More than 5 → print
   the count and leave them in the file.

No per-finding prose, no evidence excerpts, no fenced copy of the report.

Close with exactly this line, last in your reply, `<path>` as the full
absolute path — never `...`, never `~`:

> Report saved to `<path>`

Nothing follows it. Do not offer next steps — fixing, posting replies, or
planning is my decision in a later message. Do not write code until I ask.
