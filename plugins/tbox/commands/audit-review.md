---
description: Audit a code review — verify every finding against the real code, then produce a fix plan. Investigation only; writes no code.
argument-hint: '[pr-number | file-path | @file] [--all] [--round N]'
allowed-tools: Read, Write, Glob, Grep, Agent, Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh api graphql:*), Bash(gh api user:*), Bash(git log:*), Bash(git show:*), Bash(git diff:*), Bash(git rev-parse:*), Bash(git status:*), Bash(ls:*), Bash(rg:*)
---

You are auditing a code review, not performing one.

Raw arguments: `$ARGUMENTS`

## INPUT

The review is provided as ONE of the following. Detect which:

  (a) A file path or @-mention → read it.
  (b) PR comments → fetch them.
      - Top-level: `gh pr view <N> --comments`
      - Inline threads with resolution + staleness state:
        ```
        gh api graphql -f query='
          query($owner:String!, $repo:String!, $number:Int!) {
            repository(owner:$owner, name:$repo) {
              pullRequest(number:$number) {
                reviewThreads(first:100) {
                  nodes {
                    isResolved isOutdated path line originalLine
                    comments(first:20) { nodes { author{login} body path line } }
                  }
                }
              }
            }
          }' -F owner=<owner> -F repo=<repo> -F number=<N>
        ```
      - Get `<owner>`/`<repo>` by parsing the URL from
        `gh pr view <N> --json url`.
      - If no PR number is given, use the PR for the current branch
        (`gh pr view --json number,headRefName`).

Codebase: this repository.

## ANCHOR — do this first

Before reading any finding, establish and report what you are verifying
against:

- `git rev-parse --short HEAD` — the SHA every verdict below refers to.
- In PR mode, `gh pr view <N> --json headRefOid` — the PR's head.
- If the two differ, say so in one line BEFORE Phase 1 and name the risk:
  verdicts of ALREADY FIXED or STALE may reflect the branch mismatch rather
  than the code. Verify against local HEAD regardless, but flag it.
- If `git status --porcelain` is non-empty, note that uncommitted changes are
  in play — a finding may read as fixed by work that is not yet committed.

This becomes the digest's opening line: `Verified at <sha> (PR head <sha>) — <clean|N uncommitted files>`.

## ROUNDS

This command is normally run several times against the same PR: the reviewer
leaves a few comments, I fix them and reply, and ask for another pass. Do not
re-audit settled ground each round.

In file mode (input (a)) there are no threads: treat every finding as ACTIVE,
skip the classification below, and the round is 1 unless `--round N` says
otherwise.

Classify every thread first (needs my login: `gh api user --jq .login`):

- **ACTIVE** — the most recent comment in the thread is from someone other
  than me. The reviewer spoke last and is waiting.
- **SETTLED** — my reply is the most recent comment, or the thread is
  resolved.

Then:

- Audit ACTIVE threads in full (Phase 1 + Phase 2).
- Do NOT re-audit SETTLED threads. Spot-check each in one line — does the fix
  still hold at current HEAD? — and list them under "Carried forward". A
  rebase, merge, or later commit can silently drop a fix that was already made
  and replied to, and nobody re-reads round 1 by round 4. Any regression is
  promoted to a full ACTIVE finding.
- `--all` in the arguments overrides this: re-audit every thread from scratch.

Number findings with the round: `R<n>-F<m>`.

- If `--round N` is in the arguments, `<n>` is N. Use it verbatim; do not
  second-guess it.
- Otherwise infer `<n>` by counting distinct review passes on the PR (review
  submissions / comment batches), and state the inferred number in one line so
  I can correct it at the normalize checkpoint.

This keeps IDs stable enough to refer back to across cycles.

## NORMALIZE

Reviews arrive unstructured. First, extract discrete findings and assign each
an ID (`R<n>-F<m>`). Split a paragraph that raises three separate issues into
three findings. Drop pure praise and pure style opinions with no claim
attached — list them once under "Non-findings" and move on.

Merge duplicates: when two reviewers (commonly a bot and a human) raise the
same claim about the same code, that is ONE finding with two sources listed —
not two findings and not two fixes.

For PR comments, capture per finding: author, ACTIVE or SETTLED, whether the
thread is resolved, whether GitHub marks it outdated, and the file:line the
comment is anchored to.

**STOP after printing the normalized list.** Wait for me to confirm or correct
it before starting Phase 1. Do not verify anything in the same turn.

## RULES

1. Treat every finding as an UNVERIFIED CLAIM. It may be wrong, outdated,
   based on hallucinated code, or already fixed.
2. Verify by reading the actual code. Never confirm from the review's
   description alone.
3. Cite evidence as `path/to/file.ext:LINE` for every verdict. No line
   reference → verdict is UNVERIFIABLE.
4. Do not modify any source, test, or config file. Investigation only. The one
   file you may write is the report under OUTPUT.
5. PR comment specifics:
   - The anchored line is where the comment was written, against the diff
     AT THAT TIME. `isOutdated: true` means the anchored hunk has changed —
     treat it as a staleness signal, not a verdict, and always verify against
     current HEAD.
   - A resolved thread is a hint, not proof. Verify whether the code
     actually changed.
   - Weight human-authored comments higher than bot/AI-authored ones as a
     prior, but still verify both. Note the author on each verdict.

## PHASE 1 — VERIFICATION

Do the full verification, and write it to the report file — not to the session.
The per-finding sections below are the long form; the session gets only the
digest defined under OUTPUT. Never paste review text or code excerpts into the
session: the report preserves everything, so nothing is lost by not printing
it twice.

For each finding:

  ### R<n>-F<m>: <short title>
  - Source: <file | PR comment(s) by @author, ACTIVE|SETTLED, resolved: y/n, outdated: y/n>
  - Claim: <one line>
  - Evidence: <file:line + 1-3 line excerpt of the real code>
  - Verdict: <one of the seven below>
  - Reasoning: <why, based on what you actually read>
  - Real severity: critical | high | medium | low | nitpick
    (your own assessment — ignore any severity the review assigned)

Verdicts are mutually exclusive. Apply in this order, first match wins:

- **UNVERIFIABLE** — you cannot cite a `file:line`, or the claim is too vague
  to map onto code. Never use this as a polite way to avoid looking.
- **STALE** — the code the comment anchored to has moved or no longer exists,
  so the claim no longer maps to anything. Say where it went.
- **ALREADY FIXED** — the claim WAS true, and the current code is correct.
  Cite the line that fixes it.
- **NOT APPLICABLE** — the code is real and unchanged, but the claim doesn't
  apply: outside this PR's scope, or the behaviour is intentional. Say which.
- **FALSE** — the code does not do what the claim says. Cite the line that
  disproves it.
- **PARTIALLY CORRECT** — the underlying problem is real but the claim
  misstates it (wrong cause, wrong severity, wrong location). State the
  corrected version — that corrected version is what Phase 2 fixes.
- **CONFIRMED** — the claim is accurate as written.

Then: summary table (ID | Title | Source | Verdict | Real severity),
"Carried forward" (SETTLED threads spot-checked, with any regressions called
out), and "Findings not in the review" — issues you hit while verifying these
findings, not a fresh review pass of your own.

## PHASE 2 — FIX PLAN

Only for CONFIRMED or PARTIALLY CORRECT. Order by real severity, then by
blast radius (fixes others depend on go first).

  ### Fix <i>: <title>  [addresses R<n>-F<m>, R<n>-F<k>]
  - Files to change: <list>
  - Approach: <2-4 sentences, concrete — name functions/types>
  - Diff sketch: <a few before/after lines, not the full patch>
  - Risk: <what could break; flag public API, schema, behaviour changes>
  - Verification: <test to add/run, or manual check>
  - Effort: S | M | L

`<i>` is a plain 1-based fix sequence number — not the round number `<n>` used
in finding IDs.

End the report with:
- "Recommended order" — numbered sequence and why.
- "Deliberately not fixing" — confirmed findings to skip, with reason.
- "Replies to post" — one line per ACTIVE thread (accepted / rejected + why),
  written for the reviewer to read, since this is what unblocks the next
  round. Draft only; do not post. Skip SETTLED threads.
- Open questions you need answered before starting.

## OUTPUT — write the report, print the digest

### The report file

Write the complete Phase 1 + Phase 2 output to:

  `<scratchpad>/audit-review-<target>-r<n>.md`

where `<target>` is `pr-<number>` or the review file's basename without its
extension, and `<n>` is
the round. One file per round, so round 3 sits beside round 2 and the two can
be diffed — that history is how a regression across rounds becomes visible.

Start the file with a header recording what was audited:

```markdown
<!-- audit-review: target=<pr-or-file> head=<full-40-char-sha> round=<n> -->
```

**Do not print the report.** It is long by design — evidence excerpts, per
finding reasoning, diff sketches — and echoing it into the session is exactly
the bulk the digest exists to avoid.

### The session digest — target ~15 lines, hard cap 25

Print exactly these and nothing else. No per-finding prose, no evidence
excerpts, no fenced copy of the report.

1. **Context line** — `Verified at <sha> (PR head <sha>) — <clean|N uncommitted
   files>`, round `<n>`, and `<a> active / <s> settled` threads.
2. **Bottom line** — one or two sentences: what actually needs fixing, and how
   much of the review survived verification.
3. **Findings table** — one row per finding, CONFIRMED first:

   | ID | Finding | Source | Verdict | Severity |
   |----|---------|--------|---------|----------|

   This table is the digest's payload — I should scan it and know what happened
   without opening the file. It is exempt from the line cap: print one row per
   finding no matter how many there are; the cap governs everything around it.
4. **Carried forward** — one line, only if a SETTLED thread regressed. If none
   did, say "no regressions" and move on.
5. **Replies to post** — the one-liners for ACTIVE threads verbatim, since this
   is what I paste into GitHub to unblock the next round. If there are more
   than 5 active threads, print the count and leave them in the file instead.

Omit any section that is empty. If every finding came back FALSE or ALREADY
FIXED, say so in two sentences and skip the table.

Close with exactly this one line, last in your reply:

> Report saved to `<path>`

`<path>` is the **full absolute path** — never abbreviated with `...`, never
shortened to `~`. The digest is terse everywhere else, but this path is the one
thing I copy into a shell or an editor.

Nothing follows that line. Do not offer to implement the fixes — I know this
command does not fix anything, and the reminder is noise on every run.

Then stop. Do not write code until I explicitly approve in a later message.
