---
description: Cross-review an OpenSpec change with Codex plus an independent Claude pass — validates it, verifies every claim it makes about the code, and reports. Review only; writes nothing to the change.
argument-hint: '[change-id] [--wait] [extra focus text]'
allowed-tools: Read, Write, Glob, Grep, Agent, Bash(openspec:*), Bash(git:*), Bash(node:*), Bash(ls:*), Bash(sort:*), Bash(tail:*), Bash(rm .openspec-review-context.md)
---

Cross-review an OpenSpec change with Codex and with an independent Claude pass.

Raw arguments: `$ARGUMENTS`

The question this command answers is not "is this well written" but **"is this
true, and does it hold together"** — does the change say things about the code
that are actually so, does every promise survive from proposal to task, and does
the code contain anything the design failed to account for.

**This runs before any code is written and before any PR exists.** The review
subject is the change directory and the current state of the repository it
describes — there is no diff, no PR, and no branch comparison anywhere in this
command. Do not look for one, do not ask for a PR number, and do not fall back to
reviewing uncommitted work. If the user wants a diff reviewed, that is
`/tbox:cross-review`.

## Argument parsing

The change id is the first argument that matches a directory under
`openspec/changes/`. `--wait` may appear in any order; all remaining text is
extra review focus. Below, `<id>` means the resolved change id.

If no change id is given, run `openspec list --json` and pick the most recent
active change. Name the one you picked and continue — do not ask. If the id
given matches nothing, run `openspec list` and stop, showing the available ids.

## Constraints

- **Review only.** Never edit `proposal.md`, `design.md`, `tasks.md`, or any
  delta spec. Never tick a task checkbox. Never run `openspec archive`, and never
  sync deltas into main specs.
- **No git state changes, and no diff as an input.** No checkout, no branch, no
  commit, no stash, no base comparison. Whether the change files are tracked,
  untracked, or half-staged is irrelevant and is never a finding — this normally
  runs on a working tree where the change is brand new and nothing is committed
  yet. Git is read once, in step 1, only to stamp which tree the verdicts
  describe.
- Do not offer to implement the findings or to rewrite the documents. Report.

## Step 1 — deterministic gate

If a leftover `.openspec-review-context.md` from an interrupted run exists,
delete it now — a run that died before step 6 leaves a bundle describing a
different change, and both reviewers read it as gospel.

These run on your side, first, because both reviewers need the results and
neither should burn a pass rediscovering a broken structure.

```bash
openspec validate <id> --strict --type change --no-interactive
openspec show <id> --json --type change --no-interactive
ls -R openspec/changes/<id>
git rev-parse --short HEAD
git status --porcelain -- . ':(exclude)openspec/changes/<id>'
ls -d ~/.claude/plugins/cache/openai-codex/codex/*/ | sort -V | tail -1
```

`--type change --no-interactive` on both `openspec` calls: a change id often
resembles the spec it targets, and on that collision the CLI stops for an
interactive pick that no tool call can answer. `ls -R`, not a `specs/*` glob — an
unmatched glob aborts the whole command in zsh, so a change with no `specs/`
directory would read as a broken gate rather than an empty delta set.

**Codex plugin path.** The last command resolves it here, before anything is
written, because it is a precondition like the others. If it prints nothing the
Codex plugin is not installed — stop, tell the user, and suggest `/codex:setup`.
Stopping at this point leaves nothing behind; the step-2 bundle does not exist
yet. Carry the printed path forward as a **literal string** — shell variables do
not survive between Bash calls.

**Tree state.** Both reviewers read the working tree, not a commit, so record
what that tree is. The `git status` line deliberately excludes the change directory:
the change docs being uncommitted is the normal case here and is never a finding.
What matters is whether **source** files are dirty, because axes A and B verify
the design's claims against the code as it currently stands.

- Clean → the report's verdicts are anchored to `HEAD` and stay valid until the
  code moves.
- Dirty → say so, and name the dirty paths that the change also touches. Review
  the working tree anyway (it is what the author is building on), but two things
  follow: the provenance stamp is marked dirty, and any axis-B finding against a
  file with uncommitted edits is reported as provisional, since half-written code
  reads as a gap the design failed to account for when it may simply be in flight.

This is the only place git is consulted, and it qualifies the report rather than
feeding the review. Nothing here diffs, compares against a base, or reads a PR.

From `openspec show <id> --json`, list every delta: its target spec and its
operation (ADDED / MODIFIED / REMOVED).

Then, **for each MODIFIED or REMOVED delta**, check the thing validation does not
check: does `openspec/specs/<target>/spec.md` exist, and does the delta's
requirement heading match a requirement that is actually in it? A MODIFIED delta
whose base spec is absent, or whose requirement heading has drifted from the
synced text, will validate cleanly and then fail to apply. Read both and compare
the requirement headings directly.

Record for the bundle: the validate result, the delta inventory, and the
base-spec check per MODIFIED delta.

## Step 2 — build the context bundle

Codex's sandbox blocks writes and network but **not** command execution, and it
reads this worktree directly. So the bundle is not a transport for anything Codex
could fetch itself — do not paste the change documents or `CLAUDE.md` into it.
It carries exactly the three things that cannot arrive any other way.

Write `.openspec-review-context.md` at the repo root:

1. **The review contract** — the six axes from step 3, verbatim. This is the
   reason the file exists. The axes are not in the repo, and they cannot go in
   the Codex prompt: they are full of backticks, which a double-quoted shell
   string executes as command substitution, and single-quoting breaks on the
   apostrophes in the prose. A file sidesteps the quoting entirely.
2. **Gate results** — step 1's output: validate, delta inventory, base-spec
   check, and the tree-state line. Command output, and the base-spec check is
   reasoning Claude did rather than a command anyone can re-run.
3. **House rules** — a short distillation of the repo-root `CLAUDE.md`
   invariants that bear on the surfaces this change touches, plus the paths of
   the nearest `CLAUDE.md` and the OpenSpec workflow doc. This is curation, not
   transport: a large routing `CLAUDE.md` read cold sends a reviewer to the
   wrong reference. Name the paths and let both reviewers read them.
4. **Extra focus** — any focus text from the arguments.

Both reviewers read this same file, so they differ in judgment, not information.
Keep it curated — a dump dilutes it. Deleted in step 6.

## Step 3 — the review contract

Both reviewers get these six axes, in this order. The order is the yield order:
the first three find real defects, the last three find soft ones.

**A. Code claims are true.** Every file path, line number, symbol name, and
statement about current behavior in `design.md` gets checked against the tree.
Open the file and read it. A design that is confidently wrong about the code
produces tasks that fail on contact, and this is the single highest-yield axis.

**B. The code contains what the design missed.** Read the cited code for what the
change did *not* say. Existing handlers on the same element, mount order, guards
that already fire, event-phase and ordering races, state that a neighbouring
component resets. This is where a plan collides with reality, and it is not
findable from the documents alone.

**C. Promises survive end to end.** Trace each promise in `proposal.md` forward:
is there a design decision that delivers it, a spec requirement that states it,
and a task that builds it? Then trace back: is any spec requirement unowned by a
task? Flag promises that quietly shrank between documents, and any requirement
the tasks cannot satisfy.

**D. Requirements are true as written.** Each SHALL is testable and falsifiable.
Scenarios cover the negative and edge cases, not only the happy path. Hunt for
**unstated exceptions** — a real path in the code where the requirement is false
and no scenario admits it. Those are the defects that ship.

**E. Tasks are executable.** Ordering and gating are coherent, and a gate says
what it gates and why. Verification tasks name real commands, suites, and tags
that exist in this repo. Flag any task that verifies something unreachable, or
whose test would pass vacuously.

**F. Scope and structure.** Non-goals are honest rather than convenient. Risks
name the mitigation. Open questions are genuinely open, not decisions in
disguise.

## Step 4 — launch both reviews in parallel

### 4a — Codex

Use the plugin path resolved in step 1, as a literal string.

Use the `task` verb, not `review`. An OpenSpec change has no diff to review — its
files are usually untracked, and `review --base` would see nothing. Omit
`--write`: `task` is read-only by default and must stay that way here.

```bash
node "<plugin-path>/scripts/codex-companion.mjs" task \
  "Read .openspec-review-context.md at the repo root first — it carries the six \
review axes, the gate results, and this repo's conventions. \
Review the change at openspec/changes/<id>/ against those axes, hardest first. \
Read the actual source files the design cites and verify each claim; a claim you \
could not verify is a finding. Do not edit any file. Do not rewrite the documents. \
Report findings as: axis letter, file:line in the change, severity \
(blocker/major/minor/nit), what is wrong, and the concrete consequence if it ships \
unchanged. <one-sentence summary of what the change is for>"
```

Execution mode — there are two different backgrounding mechanisms here and
mixing them silently breaks the run:

- With `--wait`: call it in the foreground, **after** dispatching 4b, so the two
  still overlap.
- Otherwise call the same foreground invocation through
  `Bash(..., run_in_background: true)`. The harness re-invokes you when it exits
  and that job's stdout **is** the review — do not poll, and do not call `result`
  on the happy path.

Never pass the companion's own `--background` flag. It returns a job id
immediately and exits, so the Bash job completes in a second with no review in it
and no completion signal worth waiting for. `result` and `status` are fallbacks
for exactly that case — if the stdout comes back as a job id, fetch the review
with `node "<plugin-path>/scripts/codex-companion.mjs" result`.

### 4b — independent Claude pass, as a subagent

Do NOT review the change in this context. You have read the gate output and will
soon read Codex's findings, so a pass done here is not independent. Dispatch a
`general-purpose` subagent with a clean context.

Give it exactly this and nothing more:

- The repo path, and the change directory `openspec/changes/<id>/` to review.
- The path to `.openspec-review-context.md`, to read first.
- Ground rules: **read-only** — no edits to any file, no git state changes, no
  network, and do not invoke anything under `~/.claude/plugins` (that is the
  other reviewer).
- The instruction that carries the weight: **verify, do not summarize.** Open
  every source file the design cites and check the claim against it. Then read
  that code for what the design did not account for. Findings that come only from
  reading the change documents are the weakest kind.
- Required output: findings, each with axis letter, `file:line`, severity, one
  paragraph, and the concrete consequence — plus a list of the change's code
  claims it checked and found **true**. If the change is sound, say so plainly.
  Its final message is raw review data, not prose for the user.

## Step 5 — collect both

Wait for both to complete; their notifications arrive on their own. Do not read
either result until both are in hand, and form no findings of your own in the
meantime. Your role here is synthesis and verification, not a third review.

## Step 6 — report

First delete the bundle so it cannot leak into later work:

```bash
rm .openspec-review-context.md
```

### Do the full synthesis; print almost none of it

Read both results, reconcile them, and verify the contested calls yourself. That
analysis is the substance of this command — but it belongs in the **step 7 report
file**, not in the session. Never paste either reviewer's output verbatim.

**Dismissals require evidence.** Before calling a finding "not an issue", read the
code and cite the specific guard, contract, or scenario (`file:line`) that makes
it wrong. "Seems unlikely" is not a dismissal — without the citation, downgrade to
"unverified, leaning disagree" and say so. Same bar in reverse: do not confirm a
finding you have not traced yourself. **Rank by what costs a rewrite or an
intermittent bug**, not by how confidently it was argued.

Disagreement is the point. Do not average the two reviewers, and do not drop a
finding because you disagree with it — every finding survives into the report with
its original severity.

### The session digest — target ~20 lines, hard cap 30

Print exactly these and nothing else.

1. **Gate line** — `<id>`, validate result, delta count with operations, the
   base-spec check outcome for each MODIFIED delta, and the tree state
   (`clean at <sha>` or `dirty: <n> source files`). If dirty, add one line naming
   which axis-B findings are provisional as a result.
2. **Bottom line** — one or two sentences: is this change safe to implement, and
   what is the single most important thing to settle first.
3. **Findings table** — agreed findings first:

   | # | Finding | Axis | Codex | Claude | Verdict |
   |---|---------|------|-------|--------|---------|

4. **Disagreements** — one line each, naming which call is right. If none, say
   "no disagreements" and move on.
5. **Safe to start** — name the sections of `tasks.md` that are unblocked by the
   findings, so the user can begin there rather than waiting on the whole review.

If both reviewers came back clean, say so in two sentences and skip the table.

## Step 7 — write the report

Write to `<scratchpad>/openspec-review-<id>.md`. Do **not** print it — it is long
by design, and echoing it is exactly the bulk the digest exists to avoid.

Never write it into `openspec/changes/<id>/`. The change directory is the
deliverable and gets archived; a review artifact inside it becomes a file someone
has to remember to delete.

Open the file with a provenance comment so a later re-read knows what was judged:

```markdown
<!-- openspec-review: change=<id> validated=<pass|fail> tree=<short-sha><+dirty if source files are uncommitted> -->
```

`tree` is step 1's `git rev-parse --short HEAD`, suffixed `+dirty` when source
files were uncommitted. Recorded for one reason: axes A and B verify the change
against the source as it stands right now, so those verdicts expire when the code
moves — and on a dirty tree they describe a state that is no commit at all. It is
provenance on the report, not an input to the review; nothing in this command
reads a diff or compares against a base.

Report shape:

```markdown
# OpenSpec cross-review: <id>

`openspec validate <id> --strict` — <result>. <n> deltas: <list with operations>.
Two independent passes — OpenAI Codex and Claude — with disagreements surfaced
rather than merged.

**Bottom line:** <is it safe to implement, and what must be settled first>.

| # | Finding | Axis | Codex | Claude | Verdict |
|---|---------|------|-------|--------|---------|
| 1 | <short title> | B | major | agrees | **settle before implementing** |

---

### 1. <Finding as a sentence> — both reviewers / Codex only / Claude only

📍 `openspec/changes/<id>/design.md` · code: `path/to/file.ts:42`

One short paragraph: what is wrong.

**Consequence:** what happens if it ships unchanged — the failing task, the false
requirement, the bug that reaches a user.

**Fix (cheap/involved):** the concrete correction.

### 2. <Next finding — same structure>

---

### Minor

- `<file>` — doc-hygiene items, ranked below anything that costs a rewrite.

### Verified as holding

- One bullet per code claim that was independently checked and held up. Name the
  claim and the `file:line` that confirms it.

### Traceability

- Promises in `proposal.md` with no design decision, spec requirement, or task.
- Spec requirements no task builds.
- "All accounted for" if nothing is missing.

---
🤖 OpenSpec cross-review by [Claude Code](https://claude.com/claude-code) + OpenAI Codex
```

Rules for the report:
- Every finding cites both the change file and, where it is a code claim, the
  `file:line` that settles it. No vague "the design could be clearer".
- One `###` section per finding, numbered to match the table. Short labeled
  paragraphs — never one multi-clause bullet carrying claim, rebuttal, and fix.
- Keep each reviewer's original severity in the table; resolve in the Verdict
  column. Do not average.
- The **Verified as holding** section is not padding — it is what tells the user
  which parts of the change they can build on without re-checking.
- Omit empty sections. No praise preamble.

After writing it, close with exactly this one line, last in your reply:

> Report saved to `<path>`

`<path>` is the **full absolute path**. Never abbreviate it with `...`, never
shorten the scratchpad prefix, never substitute `~`. The digest is terse
everywhere else; this path is the one thing the user copies into a shell.

Then stop. Do not offer to fix the change, do not summarize what you would change,
and do not ask a question that a bare "yes" could turn into an edit.
