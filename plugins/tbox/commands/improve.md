---
description: Critically review and improve a target artifact — code, prompt, skill, doc, or config — preserving its intent and behavior
argument-hint: '[file-path | @file] [focus, constraints, or desired outcome]'
---

Critically review and improve a target artifact. Anything in the arguments
beyond the target itself is context: focus areas, constraints, priorities, or
the outcome wanted. Follow that context; it narrows the review, it does not
license ignoring defects you find along the way.

Raw arguments: `$ARGUMENTS`

## Resolve the target

In order, first match wins:

1. A file path or `@`-mention in the arguments → that file (or directory).
2. An artifact produced or discussed earlier in this conversation → that.
3. Uncommitted changes in the repo (`git status --porcelain` non-empty) → the
   changed files.

If none of these applies, or two of them plausibly compete, ask which one is
meant — one short question, then wait. Do not guess and start editing.

State the resolved target in one line before doing anything else.

## Understand

Before proposing any change, establish:

- **Purpose** — what the artifact is for and who consumes it (a reader, a
  runtime, an LLM, a CI system). The consumer decides what "better" means.
- **Contract** — the externally observable behavior, interface, or meaning
  that must survive. For code, that includes callers: grep for usages before
  assuming something is safe to change.
- **Existing qualities worth keeping** — deliberate choices are not defects.
  If something looks odd, check for a reason (a comment, a caller, git log)
  before treating it as a flaw.

## Find issues

Review the whole artifact, not just anything the arguments pointed at. Adapt
the criteria to the artifact type — correctness and edge cases for code;
ambiguity and contradictions for prompts and skills; structure and redundancy
for docs — and look for: correctness problems, brittle assumptions, missing
edge cases, contradictions, duplication, unnecessary complexity, weak
structure, and opportunities for real simplification.

Classify each finding:

- **Defect** — wrong, ambiguous, or broken as-is.
- **Improvement** — works, but meaningfully better is available.
- **Taste** — stylistic preference with no concrete payoff. Do not act on
  these; list them once at the end or drop them.

**Print the findings list — ranked by impact, one line each with its class —
before making any edit.** This is the review's spine; the edits must trace
back to it. If the list is empty, say the artifact is sound, name the one or
two things you checked hardest, and stop — do not invent changes to justify
the invocation.

## Apply

Fix defects and apply improvements, respecting the contract:

- Preserve externally observable behavior and meaning unless the arguments
  explicitly ask to change them.
- No scope creep: related changes are in-bounds only when they directly
  support a finding. Unrelated cleanup, reformatting, and rewording that
  merely makes the result *different* are out.
- Prefer the simple, clear fix over the clever one. If a finding needs a new
  abstraction, it must remove more complexity than it adds.
- If a defect is real but the right fix is risky or architectural, report it
  under "not fixed" instead of half-fixing it.

Edit files in place. Only when the target is conversation text with no file
does the improved version go inline in the reply.

## Verify

- Re-read the finished version against the original purpose: no lost
  behavior, no new ambiguity, no regression the edits introduced.
- For code: find and run the project's relevant checks (tests, type check,
  lint — whatever the repo actually uses) scoped to what changed. Report the
  actual result; a failed check means the work is not done.
- For prompts/skills/docs: re-read as the intended consumer would, checking
  each original obligation still holds.

## Report

Keep it proportional — a small improvement gets a few sentences. Otherwise:

1. **Bottom line** — one or two sentences: what state the artifact was in and
   what changed.
2. **Findings acted on** — the ranked list, each with what was done.
3. **Not fixed** — real issues deliberately left, with the reason.
4. **Verification** — which checks ran and their results.

Do not paste the full improved artifact into the reply when it was written to
a file; the diff is visible to the user already.
