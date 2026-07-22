# test-suite-roadmap — Design

**Date:** 2026-07-22
**Status:** Approved design, not yet implemented
**Format:** Agent Skill per https://agentskills.io/specification

## Purpose

Analyze any repository — in any language, with or without an existing test suite —
and emit a phased roadmap for building out a test suite that actually catches
regressions.

The skill is **plan-only**. It writes no tests and executes no phases.

Motivating source: https://curtispoe.org/articles/watching-claude-sonnet-outperform-opus

The load-bearing claims from that article:

- Three tiers: unit, integration, end-to-end.
- On legacy code: *"you're not trying to fix the bugs; you're trying to nail down
  current behavior."*
- Weak tests are the enemy: *"`foo is not Null` often isn't a real test."*
- The acceptance criterion for the whole suite: *"the test suite, if built
  correctly, should fail loudly when you fix bugs."*

## Non-goals

- Writing tests. A separate execution skill or session does that.
- Executing roadmap phases.
- Reviewing code diffs. `agentic-review` covers that, and it needs a diff this
  skill does not produce.
- Choosing a test framework for the user. The skill detects and recommends;
  the human decides.

## Deliverable shape

```
test-suite-roadmap/
├── SKILL.md                # detection, the five stages, phase format
└── references/
    └── test-theater.md     # weak-test catalog; loaded during grading and critique
```

Frontmatter:

```yaml
name: test-suite-roadmap
description: >
  Analyzes a repository and any existing test suite, grades existing tests for
  weakness, classifies mocks, and emits a phased roadmap for building a test
  suite that catches real regressions. Use when planning a test suite, assessing
  whether existing tests are worth anything, adding tests to a legacy codebase,
  or when the user mentions test coverage, test strategy, or weak tests.
compatibility: Requires git
```

`SKILL.md` stays under 500 lines. `test-theater.md` is the only bundled reference
that earns its own file; the grading-subagent brief and the phase format stay
inline in `SKILL.md`. If either grows past roughly a page it moves out.

## The five stages

### Stage 1 — Detect (main agent)

Identify the stack from manifests and config rather than assumption:
`package.json`, `pyproject.toml` / `setup.cfg`, `go.mod`, `Cargo.toml`,
`Gemfile`, `pom.xml` / `build.gradle`, `composer.json`, `cpanfile` / `Makefile.PL`,
`*.csproj`, `mix.exs`, and so on. Then determine how tests are invoked and what
test files already exist.

The skill must never hardcode a language. Where it needs a per-ecosystem fact it
looks for the signal in the repo rather than consulting a built-in table.

### Stage 2 — Grade (fan-out subagents)

**Skipped entirely when no tests exist.** Greenfield goes straight to Stage 3.

Dispatch subagents partitioned by test tier or directory. Each subagent reads its
partition against `references/test-theater.md` and returns a compact structured
verdict — never pasted test source, never diffs. Rationale: a legacy suite is
thousands of lines the main agent must not absorb.

Each subagent returns, per weak test found: file, line, pattern name, one-sentence
statement of why it fails to catch regressions, and a suggested replacement.
Plus, per mock encountered, a classification (see Mock ledger below).

### Stage 3 — Plan (main agent)

Identify behaviors worth testing that are not tested. Group into phases across
the three tiers. Build the mock ledger. Draft phases in the format below.

For legacy code, phases pin **current** behavior. Suspected-wrong behavior is
recorded as a note on the phase, not fixed. Fixing bugs while characterizing a
legacy system compounds two hard problems into one.

### Stage 4 — Critique (main agent, inlined)

A pushback-style adversarial pass against the skill's *own* draft plan. Findings
are fixed inline; no report is written.

The governing rule:

> **Every phase must name the bug it would catch.** If a phase cannot answer
> *"what breakage makes these tests go red?"*, it is not a phase — it is coverage
> theater. Rewrite it or drop it.

Supporting rules:

1. No phase's success criterion may reduce to *"the command exits 0."* Verified
   failure modes: `go test ./...` with no `_test.go` files exits 0;
   `jest --passWithNoTests` exits 0; a fully-skipped file exits 0 in every
   runner examined. Exit status is not evidence.
2. Every mock is classified, and every `scaffold` mock names the refactor that
   retires it.
3. Legacy phases characterize current behavior; they do not fix it.

If a `pushback` skill is available in the environment, running it additionally is
strictly additive and should be offered. It is not required — this skill is
self-contained by design, so it remains useful to anyone on any agent.

### Stage 5 — Write (main agent)

Two artifacts:

- `docs/test-roadmap.md` — the phases.
- `docs/test-suite-analysis.md` — grading verdicts and the mock ledger.

`docs/test-roadmap.md` is always the target. The skill never writes into an
existing `docs/roadmap.md`, avoiding phase-numbering collisions and clobbering of
hand-written entries.

## Mock ledger

Every mock, existing or proposed, is classified:

| Class | Meaning |
|---|---|
| `boundary` | Permanent and correct. A real external edge: network, clock, randomness, payment gateway, third-party API, filesystem where relevant. |
| `scaffold` | Exists only because the code resists testing. Test debt. **Must name the refactor that retires it.** |

The distinction is the point. A `scaffold` mock silently promoted to permanent is
how a codebase's testability problems become invisible. Recording the retiring
refactor makes the debt legible.

Known gap, accepted: a `scaffold` mock's retirement has no automatic completion
signal. Whether "the refactor that retires this mock" happened is not observable
by any check in this design. It is tracked for humans, not machines.

## Phase format

```markdown
## Phase 3: Billing retry & dunning integration tests

Catches: a retry exhausting without transitioning the account to dunning;
         a partial refund double-crediting.
Produces: tests/integration/billing/
Branch:   billing-integration-tests
Landed:
```

- **`Catches:`** — the anti-theater gate from Stage 4, preserved in the artifact so
  a later reader can audit whether the phase was honest.
- **`Produces:`** — paths the phase creates. Declared at directory granularity so
  renames within the directory do not regress the signal.
- **`Branch:`** — a human-readable breadcrumb. **Not a machine signal.**
- **`Landed:`** — empty until confirmed, then `YYYY-MM-DD <sha>`.

Each phase's definition-of-done includes a recommendation to run `agentic-review`
on the branch before merging, if available.

## Completion model

### Problem being solved

A hand-written status label goes stale the moment work is merged outside the
agent session. A fresh session reads the stale label and asks the human whether
already-finished work is finished.

Verified root cause, against the roadmap skill this was originally meant to feed:
its next-phase selection is *"the first phase whose section does not have a
`<!-- plan: … -->` comment"* — **status is never consulted by the control flow.**
It is written and read by nothing. The churn is not a control-flow step; it is an
agent pulling the whole roadmap into context, seeing a `Pending` label, and
helpfully asking. A field with no reader, sitting in context, being wrong.

That skill also marks the previous phase `Done` by inference — *"it must have been
completed if we're moving on"* — with no evidence. That is a defect in that skill,
out of scope here, and the reason this design has no inferred status at all.

### Governing principle

**No signal auto-marks anything done.** Signals decide whether to ask, and supply
the evidence attached to the asking. Being wrong toward "ask with evidence" costs
one keystroke. Being wrong toward "auto-mark done" silently skips real work.

The original bug was never that the agent asked. It was that it asked
empty-handed:

- Bad: *"Phase 3 shows Pending. Is it complete?"*
- Good: *"Phase 3 declares `tests/integration/billing/` — present, 11 files, added
  in `a1b2c3d` (3 days ago). Marking landed; say otherwise."*

The second is answered by not replying.

### Protocol

For the candidate phase only:

| Step | Where | Action |
|---|---|---|
| 1 | main agent | `Landed:` populated? → done. Stop. No git, no question. |
| 2 | main agent | Paths absent? → phase genuinely not started. Proceed to plan it. No question. |
| 3 | **subagent** | Paths present, not latched → corroborate, then latch unless the human objects. |

Step 1 is what ends the churn permanently: the question is asked once per phase in
the project's lifetime, then never again. The latch is monotonic — a later
restructure cannot un-land a completed phase, which is what happens when
`agentic-review` condemns a phase's tests and they get rewritten in new locations.

Critically, the latch differs from the status field it replaces: status was
written by *inference*; `Landed:` is written only on *observed evidence*.

### Corroboration subagent

Dispatched only in step 3. Because the diff never enters main context, the
subagent can read it in full:

```
git log --diff-filter=A --format='%h %ad %s' -1 -- <paths>   # the commit that ADDED the paths
git show --stat <sha>                                        # cheap: files and line counts
git show <sha>                                               # full diff, on ambiguity
```

`--diff-filter=A` finds the *landing* commit rather than the most recent unrelated
touch.

The subagent returns one line, e.g.
`landed: true — added in a1b2c3d (2026-07-19), 11 files, +840 lines under tests/integration/billing/`

Two constraints written into its brief:

1. Return a verdict, never paste the diff.
2. It cannot ask the human. On ambiguity it returns `inconclusive` with a reason,
   and the main agent surfaces that rather than guessing.

### Why not branch-merge detection

Considered and rejected as the primary signal. Verified behavior:

| Merge style | `git branch --merged main` |
|---|---|
| `--no-ff` merge commit | works |
| Squash merge (common default) | **fails** — branch tip is not an ancestor |
| Rebase merge | **fails** — commits rewritten, new SHAs |
| Branch deleted after merge | **fails** — nothing left to query |
| Fresh clone | **fails** — no local branches exist |

Four of five cases fail, and all fail *toward "not done"* — regenerating the exact
churn being eliminated. Path existence queries tree state, which is
merge-strategy independent.

### Why not commit-log scanning

Also rejected. Matching phase titles against commit messages is string-matching
human prose: a phase that landed 40 commits ago falls outside any window; real
messages say `Add dunning retry tests` rather than `Phase 3: …`; squash collapses
to a PR title that may match nothing. Git log answers *when* and *which commit*
well, and *whether* badly — so it is used as the corroborator in step 3, never as
the primary signal.

## Accepted limitations

1. A file existing does not mean the tests inside it are good. Path existence
   answers "did the work land," not "is it any good." Grading answers the latter,
   separately.
2. `scaffold` mock retirement has no automatic completion signal.
3. A phase that produces no new files — a pure refactor, or strengthening tests
   in place — cannot use `Produces:`. Such a phase must be authored with a
   `Produces:` path that its work genuinely creates, or accept a one-time human
   confirmation.
4. The skill does not bootstrap the scaffolding other roadmap tooling may expect
   (`CLAUDE.md` sections, `docs/plans/`, `docs/roadmap-decisions/`). It is
   self-contained and assumes none of it.

## Decision log

| Decision | Chosen | Rejected |
|---|---|---|
| Scope | Plan only | Plan + execute; plan + execute + weak-test gate |
| Existing tests | Grade them; fold findings into the roadmap | Baseline only; separate audit skill |
| Format | Agent Skill (agentskills.io) | Kiro power |
| Dependencies | Self-contained, pushback-style critique inlined | Hard-depend on roadmap + pushback; self-contained with no inlined critique |
| Grading | Fan out by tier/directory | Single-pass sampling; conditional fan-out |
| `agentic-review` | Recommended per phase, not baked in | Baked into the skill |
| Roadmap path | Always `docs/test-roadmap.md` | Always `docs/roadmap.md`; conditional merge |
| Completion signal | `Produces:` + git corroboration + `Landed:` latch | Runnable `Done-when:` predicate; branch-merge detection; commit-log scanning; status field |
| Corroboration | Subagent, verdict-only | Main agent reads git output |

A `/pushback` review of the completion-model options produced the findings that
killed the `Done-when:` shell predicate: no consumer executed it, exit-0 is not
evidence, shell commands in checked-in markdown are an injection surface, and the
greenfield case has no runner to invoke. Those findings are reflected above.
