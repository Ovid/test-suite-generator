# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

`test-roadmap`, an Agent Skill (per https://agentskills.io/specification) that analyzes a
repository, grades any existing test suite, emits a phased roadmap for building a test suite
that catches real regressions, then executes those phases across multiple sessions.

The repository holds two things that must stay in sync:

1. **The skill itself** — `test-roadmap/`, made of Markdown: `SKILL.md` (the router) plus
   `references/` (the two mode files, the `break-it-check` anti-theater gate, the
   `test-theater` weak-test catalog, and `test-pushback` — the menu format, the
   plain-language rule, and the critique pass). The skill *is* these files; this is the
   implementation. It's alpha.
2. **The design** — `docs/superpowers/specs/2026-07-22-test-suite-roadmap-design.md`, the
   source of truth for *why* the skill is shaped the way it is.

```
test-roadmap/
  SKILL.md
  references/  build-test-roadmap.md, execute-test-roadmap.md,
               break-it-check.md, test-pushback.md, test-theater.md
docs/superpowers/specs/2026-07-22-test-suite-roadmap-design.md
```

There is no build system, test suite, linter, or dependency manifest, and the skill needs
none — it's Markdown. Do not go looking for one, and do not scaffold one unless asked.

## Working on the skill and its design

The skill files and the spec **must stay consistent**: a behavior change lands in
`test-roadmap/`, and its rationale — including what was rejected and why — lands in the
spec's `## Decision log`. Don't change one and leave the other stale.

`docs/superpowers/specs/2026-07-22-test-suite-roadmap-design.md` is the source of truth. Its
`**Status:**` line states where the design stands; read it before assuming the document is settled.

The design has been through multiple adversarial review rounds, and its `## Decision log` records
what was chosen *and what was rejected, with reasons*. Before proposing a change, check whether it
is already in the rejected column — several attractive-sounding ideas (a `Covers:` field, a
path-based write-fence, branch-merge detection, commit-log scanning, a status field, bundling TDD)
were considered and killed on specific grounds. Reversing one requires engaging with the recorded
reason, not restating the original proposal.

The `## Verification notes` section separates claims that were reproduced from claims that were
only asserted. Preserve that distinction when editing. An unverified claim presented as verified is
treated here as a defect.

Two requirements govern the design and constrain nearly every decision in it:

1. **Dead simple to invoke** — one command, `/test-roadmap`, on day 1 and day 90. A constraint on
   invocation, not on interaction.
2. **Resumable** — across unrelated commits, squash merges, fresh clones, and sessions with no
   memory of the previous one.

A change that compromises either is a change to the design's premises, not a detail.

## Inviolate decisions

These are settled. Do not re-open, re-litigate, or quietly erode them. If one genuinely needs to
change, say so explicitly and argue it as a premise change — never route around it.

1. **The suite pins current behavior, even wrong behavior.** Characterizing a legacy system and
   fixing its bugs are two hard problems; the skill does one. Suspected-wrong behavior is recorded
   as a note on the phase. Bugs get fixed later, by developers, watching these tests break — that
   is the point of building the suite, not a step within it.
2. **Production code is never permanently modified.** `break-it-check`'s bug injection happens in a
   throwaway git worktree, never in the developer's working tree.
3. **Exit status and coverage percentage are never evidence that a test is good.** A passing command
   and a covered line both survive a test that asserts nothing. Coverage is legitimate for finding
   *untested* behavior; it can never certify a test that exists.
4. **No signal auto-marks anything done.** Signals decide whether to ask and supply the evidence
   attached to the asking. Being wrong toward "ask with evidence" costs a keystroke; being wrong
   toward "auto-mark done" silently skips real work.
5. **Every phase names the bug it would catch.** A phase that cannot answer "what breakage makes
   these tests go red?" is coverage theater and gets rewritten or dropped.
6. **The skill is self-contained.** It may recommend other skills when present, and must never
   require them.

The design's `## Decision log` holds the rest — settled, but revisable on new evidence. This list
is not.

## Always look for the simplest solution that will work

Find the smallest mechanism that actually closes the hole, then stop. Two mechanisms guarding the
same failure are usually one mechanism and one piece of theater — apply the design's own Stage 4
rule to its machinery and ask what breakage each one catches that the others don't. If it can't
answer, cut it.

This is a rule about the solution, never about the analysis. Understand the problem completely
first: trace what the change touches and what it interacts with, *then* take the smallest fix. A
small change in the wrong place is not simplicity, it is a second bug.

Deletion counts as progress. A revision that removes a section and closes the same hole is a better
revision than one that adds a section.

## Presenting options

Whenever you present me with options — design forks, approaches, anything where I have to choose —
the response must contain:

- each option with **pros and cons**;
- a **recommendation** with the **justification** for it;
- a final option to **dispatch a subagent for an adversarial pushback review against the presented
  options**.

This applies to ordinary conversation, not just to documents. It is also the format the design
itself mandates for its approach menus, so a violation here is inconsistent with the artifact.

**One decision at a time.** When feasible, ask for exactly one decision, then stop. Do not bundle
a second question ("...and do you also want X?") onto the one being asked — it splits attention and
degrades the answer to both. Resolve the first decision, act on it, and only then raise the next.
A follow-up that is fully determined by the first answer is not a second decision — just do it.
