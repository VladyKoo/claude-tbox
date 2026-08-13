---
description: Run cross-review N times with rotating focus lenses, then combine all cycles into one vote-counted verdict
argument-hint: '<pr-number-or-url> [cycles=3] [--regular] [extra focus text]'
allowed-tools: Read, Write, Glob, Grep, Agent, Bash(git:*), Bash(gh:*), Bash(node:*), Bash(ls:*), Bash(sort:*), Bash(tail:*), Bash(rm .cross-review-context.md)
---

Run the `/tbox:cross-review` procedure multiple times over the same PR, each cycle
with a different review lens, then merge every cycle's findings into a single
vote-counted report. Built for huge PRs where one pass has too much surface to
cover reliably.

**This command reuses the base procedure.** Read
`${CLAUDE_PLUGIN_ROOT}/commands/cross-review.md` now — its steps, constraints, and calling
contracts (Codex invocation, subagent prompt shape, draft format) all apply
here except where this file explicitly overrides them. Do not re-derive that
mechanics from memory.

Raw arguments: `$ARGUMENTS`

## Argument parsing

Same as the base command, plus: the first bare number that is **not** the PR
number (and is ≤ 10) is the cycle count. Default is **3**; hard cap 5 — more
than that is cost without coverage. `--regular` and extra focus text pass
through to every cycle. `--wait` is meaningless here (cycles are always
sequential with Codex in the foreground) — ignore it if present.

## Inherited constraints — restated because they matter more here

- Review only; never post to the PR; never fix anything.
- Never touch other worktrees.
- **Cycle independence is the whole point.** No cycle's reviewers may see any
  earlier cycle's findings. The Claude pass is always a fresh subagent; the
  main session must not paste earlier results into any prompt.
- **The main session stays lean.** Raw reviewer output goes into per-cycle
  files immediately (step C below) and is analyzed once, at the end. Do not
  synthesize, verify, or editorialize between cycles.

## Focus lenses

Cycle *i* uses lens *i* (wrap around if cycles > 5):

1. **Correctness & logic** — wrong behavior, broken invariants, off-by-one,
   state machine edges, concurrency/ordering.
2. **Security & tenancy** — authz, tenant isolation, injection, secrets,
   trust of client-supplied identity.
3. **Error handling & edge cases** — failure paths, partial-failure recovery,
   null/empty/oversized inputs, timeouts, idempotency.
4. **Tests & contracts** — coverage of the changed behavior, API/wire contract
   shifts, migration safety, backward compatibility.
5. **Free-roam adversarial** — no lens; find whatever the others missed.

The lens is **appended to the reviewer prompts**, phrased as
"This cycle, weight your attention toward: <lens>. Report anything you find
regardless of category." It steers attention; it must not forbid off-lens
findings. The lens does NOT go into `.cross-review-context.md` — the bundle
stays identical across cycles so cycles differ in attention, not information.

## Setup — once, not per cycle

Run the base command's steps 1–3 exactly once: refuse dirty tracked files,
fetch and detach onto the PR head, read the PR discussion, fetch the base
branch, and write `.cross-review-context.md` (with the user's extra focus text
in its section 5, as usual). Also resolve the Codex plugin path once (base
step 4a's `ls -d ... | sort -V | tail -1`) and reuse the literal path in every
cycle. Record `<pr>`, `<head-sha>`, `<baseRefName>`, `<review-verb>`.

The context bundle stays on disk for the whole run and is deleted only in the
Combine step.

## Cycle loop — for i in 1..N, strictly sequential

Codex's companion runtime holds a single result slot, so cycles cannot
overlap. Within a cycle the two reviewers do overlap:

**A.** Dispatch the independent Claude subagent (base step 4b prompt, plus the
lens sentence) in the background.

**B.** Run Codex in the **foreground** (base step 4a invocation with the lens
sentence appended to its prompt argument). Foreground is deliberate: it
serializes the cycles and its completion naturally paces the loop.

**C.** When both are done, immediately write each raw result to its own file —
verbatim, no analysis, no commentary in the session beyond one status line
("cycle 2/3 done, N findings raw"):

- `<scratchpad>/cross-review-pr-<pr>-cycle-<i>-codex.md`
- `<scratchpad>/cross-review-pr-<pr>-cycle-<i>-claude.md`

Each file starts with a header line recording the cycle number and lens.
Then start the next cycle. Do not read the files back until all cycles finish.

If a cycle fails outright (Codex error, subagent death), record the failure in
that cycle's file, tell the user in one line, and continue with the remaining
cycles — a lost cycle degrades the vote counts, it does not abort the run.

## Combine — after the last cycle

First delete `.cross-review-context.md`.

Now read every per-cycle file and merge:

1. **Dedup by locus.** Two findings are the same finding if they point at the
   same `file:line` region (±ε) and describe the same failure mode — judge by
   substance, not wording.
2. **Vote-count each merged finding.** A "vote" is one reviewer in one cycle
   raising it — max possible is 2×N. Record votes as e.g. `4/6` and list which
   cycles/reviewers raised it. High votes ≈ real; a single vote from one
   adversarial Codex pass is the noisiest possible signal.
3. **Verify the contested middle.** Findings with unanimous or near-unanimous
   votes need only a sanity check; single-vote findings and reviewer
   disagreements get the base command's evidence bar — trace the code, cite
   the guard or test that dismisses it, or mark it "unverified, leaning
   agree/disagree". Do not silently drop even 1-vote findings; they survive
   into the draft with their vote count as the caveat.
4. **Severity:** take the highest severity any reviewer assigned, then note in
   the verdict if you overrode it downward with evidence.

## Report

Base command's step 6 digest shape, with two changes:

- The findings table gains a **Votes** column (`4/6`) and is sorted by votes
  descending, then severity.
- The context line states the cycle count and lens list, so the reader knows
  what 6 votes means.

Same hard cap: ~20 lines, 30 max. No per-cycle recaps, no verbatim output.

## Draft — one combined PR comment

Base command's step 7, with these deltas:

- File: `<scratchpad>/cross-review-pr-<pr>-combined.md`
- Header comment additionally records `cycles=<N>`:
  `<!-- cross-review: pr=<pr> head=<sha> base=<base> cycles=<N> -->`
- The intro sentence says N independent dual-reviewer cycles were run with
  rotating focus lenses; the table carries the Votes column.
- Per-finding sections note which cycles raised it only when that is
  informative (e.g. "found only by the security lens cycle").
- Everything else — file:line citations, disagreement handling, honest
  attribution, no padding, the closing "Draft saved to `<absolute path>`"
  line, and the do-not-post rule — is inherited unchanged.

The per-cycle raw files are left in the scratchpad for audit; mention their
common prefix once in the digest's worktree-state line.

## Posting later

Identical to the base command's staleness-checked posting flow, using the
combined draft file.
