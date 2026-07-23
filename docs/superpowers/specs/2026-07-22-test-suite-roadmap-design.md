# test-roadmap — Design

**Date:** 2026-07-22
**Last revised:** 2026-07-23
**Status:** Approved design, revised after a fourth pushback review (overthinking pass); later revised from trial feedback (zero-repo-knowledge menu premise; build-mode commits its artifacts; phases commit onto the current working branch, no per-phase branch; `Tier:` field + per-tier run/coverage commands instead of a layout mandate; run instructions surfaced on completion; plain-language rule for all developer-facing output; clean test-output discipline)
**Format:** Agent Skill per https://agentskills.io/specification

## Purpose

Analyze any repository — in any language, with or without an existing test suite —
emit a phased roadmap for building a test suite that actually catches regressions,
then execute those phases one at a time across as many sessions as it takes.

Two requirements govern every decision below:

1. **Dead simple to invoke.** One command, `/test-roadmap`, on day 1 and on day 90.
   The developer never has to remember which skill to run next. This is a
   constraint on *invocation*, not on interaction — being consulted is fine.
2. **Resumable.** Picking the work back up must work after other developers have
   pushed unrelated commits, after a squash merge, in a fresh clone, and from a
   session with no memory of the previous one.

Motivating source: https://curtispoe.org/articles/watching-claude-sonnet-outperform-opus

The load-bearing claims from that article:

- Three tiers: unit, integration, end-to-end — and they must stay distinguishable
  and independently coverage-runnable (see *The tiers must be independently
  runnable*).
- On legacy code: *"you're not trying to fix the bugs; you're trying to nail down
  current behavior."*
- Weak tests are the enemy: *"`foo is not Null` often isn't a real test."*
- The acceptance criterion for the whole suite: *"the test suite, if built
  correctly, should fail loudly when you fix bugs."*

That last claim is the acceptance criterion for the *skill*, not just the suite it
produces. An earlier draft of this design stated it and then never enforced it.
`break-it-check` (below) is the stage that enforces it.

## Non-goals

- Reviewing code diffs. `agentic-review` covers that, and it needs a diff this
  skill does not produce.
- Choosing a test framework for the user. The skill detects and recommends; the
  human decides.
- Fixing bugs. The suite pins **current** behavior, correct or not.
- Optimizing the suite as a running artifact — speed, parallelization, fixture
  performance. The skill *names* the suite-health properties its own gate depends
  on (isolation, determinism) and surfaces violations, but it is not a test-suite
  optimizer. See *Suite-health preconditions*.
- Bootstrapping scaffolding for other roadmap tooling (`CLAUDE.md` sections,
  `docs/plans/`, `docs/roadmap-decisions/`). This skill is self-contained and
  assumes none of it.

## Deliverable shape

```
test-roadmap/
├── SKILL.md                       # router + detection only; stays dumb
└── references/
    ├── build-test-roadmap.md      # the five stages
    ├── execute-test-roadmap.md    # completion protocol, writes tests, latches
    ├── break-it-check.md          # mandatory per-phase gate
    ├── test-pushback.md           # adversarial critique + approach-menu format
    └── test-theater.md            # weak-test catalog
```

Frontmatter:

```yaml
name: test-roadmap
description: >
  Analyzes a repository and any existing test suite, grades existing tests for
  weakness, classifies mocks, emits a phased roadmap for building a test suite
  that catches real regressions, then executes those phases one at a time. Use
  when planning or building a test suite, assessing whether existing tests are
  worth anything, adding tests to a legacy codebase, or when the user mentions
  test coverage, test strategy, or weak tests.
compatibility: Requires git
```

Per-file budget: each file stays under 500 lines. References load on demand
(agentskills.io progressive disclosure: *"The agent follows the instructions,
optionally executing bundled code or loading referenced files as needed"*), so the
context cost of a resume run is the router plus `execute-test-roadmap.md` plus
`break-it-check.md` — never the build stages, never the weak-test catalog unless
grading is actually running.

### Why one skill rather than two

An earlier plan was two skills: one to build the roadmap, one to execute it. That
fails requirement 1 — the developer has to know which to run and when — and it
turns the `Landed:` contract into a cross-package protocol, which is where this
class of design rots. The context saving that motivated the split is fully
delivered by on-demand references instead.

### The router

`SKILL.md` does one thing:

```
docs/test-roadmap.md exists?  →  load references/execute-test-roadmap.md
                       absent  →  load references/build-test-roadmap.md
```

That is the entire routing logic. If it ever grows a third condition, that is a
signal something has been put in the wrong place. The router never loads
`break-it-check.md`, `test-pushback.md`, or `test-theater.md` directly — the two
mode files do.

## Build mode: the five stages

### Stage 1 — Detect (main agent)

Identify the stack from manifests and config rather than assumption:
`package.json`, `pyproject.toml` / `setup.cfg`, `go.mod`, `Cargo.toml`,
`Gemfile`, `pom.xml` / `build.gradle`, `composer.json`, `cpanfile` / `Makefile.PL`,
`*.csproj`, `mix.exs`, and so on. Then determine how tests are invoked and what
test files already exist.

Detect **four** things, all from repo signals rather than a built-in table:

1. How tests are invoked, and what test files exist.
2. Whether a **coverage tool** is available (used read-only in Stage 3 for
   gap-finding — never as a quality signal). Where none is detectable, Stage 3
   falls back to agent judgment and says so.
3. Whether a **test-data mechanism** exists — fixtures, factories, seed scripts,
   testcontainers, a `conftest.py`, a `factories/` or `fixtures/` directory. This
   is an input to Stage 3: an integration or e2e phase that needs constructed data
   cannot be planned honestly against a repo that has no way to construct it. See
   *Test-data strategy*.
4. **How the repo separates test tiers, and the per-tier run + coverage
   command.** The tiers must end up distinguishable and independently
   coverage-runnable (see *The tiers must be independently runnable*). The
   mechanism is a *selector* the ecosystem already expresses and it is not
   universal — a path-selected directory only where the ecosystem uses an
   external test tree, otherwise a tag/marker, a build target, or a label.
   Detect which, and derive per tier the command that runs just that tier with
   coverage. Do not assume the selector is a directory.

The skill must never hardcode a language. Where it needs a per-ecosystem fact it
looks for the signal in the repo rather than consulting a built-in table.

**Stage 1's output is a discovery, not a boundary.** It tells you what tests exist;
it does not exhaustively partition the repo into "test" and "production." Nothing
downstream treats it as a safety fence — bug injection happens in a disposable
worktree (see `break-it-check`), so no path-based fence is needed anywhere.

### Stage 2 — Grade (fan-out subagents)

**Skipped entirely when no tests exist.** Greenfield goes straight to Stage 3.

Dispatch subagents partitioned by test tier or directory. Each subagent reads its
partition against `references/test-theater.md` and returns a compact structured
verdict — never pasted test source, never diffs. Rationale: a legacy suite is
thousands of lines the main agent must not absorb.

Each subagent returns, per weak test found: file, line, pattern name, one-sentence
statement of why it fails to catch regressions, and a suggested replacement. Plus,
per mock or fixture encountered, a classification (see *the ledger*).

**The number of instances is unbounded by design, and the design contains it
rather than merely asserting it is unbounded.** Per-item verdicts are bounded in
size (structured, no pasted source), but *cardinality* is not — a legacy suite may
yield hundreds. Therefore:

- The full verdict list is written to `docs/test-suite-analysis.md`, not returned
  to main context.
- Main context receives **counts and the top-K by severity only**. A 400-item
  grading result must never land in the resume context budget.

An earlier draft bounded per-item size and left cardinality unbounded into main
context, which is the same defect wearing a different hat.

### Stage 3 — Plan (main agent)

Identify behaviors worth testing that are not tested. Group into phases across the
three tiers. Build the ledger. Draft phases in the format below.

Two inputs from Stage 1 feed this stage:

- **Coverage, for gap-finding only.** Where a coverage tool was detected, run it
  read-only to locate behaviors with *no test at all*. Coverage answers "what is
  untested," which it does well; it never answers "is this test any good," which
  it cannot (a line is covered by a test that asserts nothing). This is an
  inviolate distinction — coverage percentage is never evidence a test is worth
  keeping. Where no tool exists, this is agent judgment reading source, and the
  plan says so.
- **Test-data mechanism.** If an integration or e2e phase needs constructed data
  and Stage 1 found no mechanism to construct it, the phase carries a surfaced note
  — *"requires a test-data mechanism this repo lacks"* — rather than being planned
  as if executable. Discovering un-runnability at plan time is the point of having
  a plan stage; discovering it at the gate is the expensive place.

For legacy code, phases pin **current** behavior. Suspected-wrong behavior is
recorded as a note on the phase, not fixed. Fixing bugs while characterizing a
legacy system compounds two hard problems into one.

### Stage 4 — Critique (main agent, via `test-pushback.md` mode `critique-plan`)

An adversarial pass against the skill's *own* draft plan. Findings are fixed
inline; no report is written.

The governing rule:

> **Every phase must name the bug it would catch.** If a phase cannot answer
> *"what breakage makes these tests go red?"*, it is not a phase — it is coverage
> theater. Rewrite it or drop it.

Supporting rules:

1. No phase's success criterion may reduce to *"the command exits 0."* Verified
   failure modes: `go test ./...` with no `_test.go` files exits 0;
   `jest --passWithNoTests` exits 0; a fully-skipped file exits 0 in every runner
   examined. Exit status is not evidence — and neither is coverage percentage.
2. Every test double and fixture is classified, and every `scaffold` mock names the
   refactor that retires it (deferred per *the ledger*).
3. Legacy phases characterize current behavior; they do not fix it.

If a `pushback` skill is available in the environment, running it additionally is
strictly additive and should be offered. It is not required — this skill is
self-contained by design, so it remains useful to anyone on any agent.

### Stage 5 — Write (main agent)

Two artifacts:

- `docs/test-roadmap.md` — the `## Decisions` section, then the phases.
- `docs/test-suite-analysis.md` — full grading verdicts and the ledger. This
  is the sink for anything unbounded (Stage 2's full list, the ledger); main
  context never carries it.

`docs/test-roadmap.md` is always the target. The skill never writes into an
existing `docs/roadmap.md`, avoiding phase-numbering collisions and clobbering of
hand-written entries. Its existence is also the router's signal, so its path is
load-bearing and not configurable.

## Execute mode

Loaded when `docs/test-roadmap.md` exists. This is the path every run after the
first takes, so it must be quiet about anything the developer has already settled.

1. Read `## Decisions` from the roadmap. Do not re-ask anything recorded there.
   Place the phase's tests per its `Tier:` and the recorded organization.
2. Select the candidate phase per the *Completion model* protocol.
3. Write the tests for that phase.
4. Run `break-it-check.md`. **This gate is mandatory. No phase latches without it.**
5. Write `Landed: YYYY-MM-DD <sha> (<operator>)` and commit.
6. **Surface run instructions** for the tests just landed — drawn verbatim from
   the `## Decisions` per-tier commands: how to run *just this phase's* tests, this
   tier *with coverage*, and *the whole suite*. The developer may not know the
   repo, the runner, or the language, so leaving them to reconstruct the commands
   fails the same zero-repo-knowledge premise the menus assume. Recorded commands,
   not invented ones — a command that has drifted surfaces as a failed run the
   developer sees, not as silent wrong advice.

Execute mode writes test code. It does not write production code — and it never
needs a path-based rule to enforce that, because bug injection happens in a
throwaway worktree that is discarded, never in the developer's tree.

**The tests it writes produce a clean run** — no spurious warnings or stray
STDOUT/STDERR. Noise buries the signal, and a suite that always warns trains the
developer to ignore warnings, so the one warning that was a real bug scrolls past
unnoticed. Order of preference: (1) if the output *matters* — a warning that
should or shouldn't fire — treat it as behavior and **capture and assert it**, at
which point it's a test, not noise; (2) suppress uncontrollable third-party
chatter narrowly at the test boundary, scoped so a *new* warning still stands
out; (3) never let the written tests add their own debug prints. Where a clean
run genuinely isn't achievable, say so plainly rather than shipping a suite that
cries wolf.

### Why TDD is not bundled

`superpowers:test-driven-development` is deliberately **not** loaded by execute
mode. Verified at `test-driven-development/SKILL.md:126`:

> **Test passes?** You're testing existing behavior. Fix test.

For a characterization phase that instruction is not merely inapplicable, it is
destructive: the code already works, so the test passes on first run *by
construction*, and TDD would order the agent to rewrite the test until it goes
red. The red phase that characterization actually needs is mutation, not
authorship — which is what `break-it-check` provides.

## break-it-check

The gate that discharges *"the test suite should fail loudly when you fix bugs."*
For each behavior named on the phase's `Catches:` line: inject that bug into a
**disposable copy** of the production code, confirm the new test goes red, discard
the copy.

The mechanism is adapted from `verification-before-completion/SKILL.md:84-88`
(verified):

```
Write → Run (pass) → Inject bug → Run (MUST FAIL) → Discard copy → Run (pass)
```

with two inversions from that skill: there is no fix to revert — the bug is
**injected** into working code — and the injection happens in a throwaway `git
worktree`, so nothing in the developer's tree is ever at risk.

### Protocol

1. **Sweep, then create one worktree per phase.** First reclaim any strand from a
   crashed or aborted prior run: `git worktree prune` (which only reclaims records
   whose checkout dir is already gone — it cannot touch a live worktree), then
   enumerate `git worktree list --porcelain`, **filter to paths under
   `.git/test-roadmap-worktrees/`, and force-remove only those.** The removal is
   filter-to-prefix, never a blanket "remove all worktrees" — that filter is the whole
   guarantee that a developer's hand-made worktree (which lives outside the prefix) is
   never in the set. Then `git worktree add .git/test-roadmap-worktrees/<phase> HEAD`.
   The prefix is fixed and inside `.git`, so the sweep can identify its own worktrees
   and the checkout never lands in the working tree. The worktree is reused across the baseline run and
   every mutation in the phase, so the (ecosystem-specific, undiscoverable)
   dependency-install cost is paid once per phase, not once per mutation.
2. **Copy the phase's new test files into the worktree, *then* mutate.** Order is
   load-bearing. In colocated-test ecosystems (Rust `#[cfg(test)] mod tests`, Zig
   `test` blocks, D `unittest`, Rust/Python doctests, Elixir `doctest`) the test
   *is* the production file; mutate first and the copy silently overwrites the
   mutation, the test stays green, and the gate misreads it as theater. Only the
   Rust case was reproduced — see *Verification notes* — but one is sufficient.
3. **Baseline run.** Run the phase's own tests, unmutated, in the worktree. They
   **must pass**. Also confirm the result is *stable across two runs*. See the
   failure table for what each way of failing this step means.
4. **Per behavior on `Catches:`, inject one mutation via a blind mutator
   subagent.** The subagent receives the `Catches:` behavior text, the production
   file, the worktree path, and the operator list — but **not the test**. It
   returns one hunk. The main agent applies the *first* hunk returned, runs the
   phase's own tests, and checks for red. A subagent that cannot see the test
   cannot tune the mutation to the test — which severs the root problem: the same
   agent otherwise authors both the test and the mutation meant to indict it.
5. **`git worktree remove --force`.** `--force` is required, not optional: the
   worktree is unclean by construction — the copied-in test is untracked, the
   injected mutation is a tracked-file modification — and git refuses to remove an
   unclean worktree without it (verified). Nothing to restore, nothing to stage, no
   fence. This is only the fast path; step 1's sweep is the guarantee (see below).
6. **Commit only after the check passes,** on the developer's real tree. The
   operator name(s) used are echoed to the transcript and recorded on the `Landed:`
   line as provenance.

**"Run the test" means the phase's own tests, not the whole suite.** Mutating
production code *should* break unrelated tests — that is expected, not signal — so
there is no reason to run the suite here, and a legacy suite's pre-existing
failures stay out of frame.

### Constrained mutation

The mutation is not free-form. It is a single change drawn from a **named operator
set**, and the subagent must declare which operator by name:

- flip a comparison (`<` ↔ `<=`, `==` ↔ `!=`)
- off-by-one a boundary
- negate a condition
- alter a constant
- drop a state transition

**Explicitly banned:** an early `return` / `raise` / `throw` at function entry, and
removal of a whole function body. These are sledgehammers: they make *any* test go
red, including `assert result is not None`, so they let a theater test pass the
gate. Naming the banned shapes closes the hole that "keep the mutation small" left
open — an agent can rationalize "small," but "no early return at function entry" is
a concrete line. Declaring the operator by name makes a dishonest mutation an
affirmative written claim rather than a silent choice.

If the first hunk leaves the test green, that is the *theater* row below — rewrite
the test, do **not** re-roll the mutation. Re-rolling would let the main agent
shop for a convenient hunk, reintroducing the identity the blind mutator exists to
break.

### Failure table

| Outcome | Meaning | Action |
|---|---|---|
| Baseline red (test fails unmutated, in isolation) — but green in a full-suite run | The test has an **order dependency**, not a mischaracterization: it relies on state another test sets up. The isolated gate cannot judge it. | **Surface it. The phase cannot use the isolated gate as-is.** Do not latch, do not "fix" the code. |
| Baseline red (test fails unmutated) — and also red in a full-suite run | The test does not describe current behavior — **mischaracterization**. | **Rewrite the test. Do not fix the code, do not latch.** |
| Baseline result unstable across two runs | The test is **flaky** — its verdict is noise, so the gate cannot judge it. | **Surface it. Do not latch.** |
| No code path implements this `Catches:` behavior (no valid site to inject) | The phase was planned against code that has since moved or vanished. | **Phase invalidated — re-plan it.** Do not latch. |
| Mutation applied, test stayed green | The test does not catch what the phase claims. | **Theater — rewrite the test.** Do not latch. Do not re-roll the mutation. |

The order-dependence and flaky rows exist because the isolated gate silently
*depends* on the phase's tests being isolable and deterministic. Naming those
preconditions where the gate touches them is the whole of this skill's engagement
with suite health (see *Suite-health preconditions*); it detects the violations, it
does not repair the suite.

The "no code path" row is the design's staleness detector, and it is sufficient:
a bug cannot be injected into a code path that moved or vanished, and no phase
latches without passing the gate. See *Why not an anchor commit*.

### Why a disposable worktree, not an in-tree mutation

An earlier draft mutated the developer's working tree and protected it with an
elaborate write-fence: a whole-tree `git status --porcelain` snapshot compared
before and after each mutation, plus a `git add`-before-`git checkout` staging
dance to survive `git checkout`'s restore-from-index behavior. Roughly forty lines
of protocol existed solely to make an in-tree mutation survivable — and it still
had a window: a crash, context exhaustion, or a `--watch` runner mid-mutation could
leave altered production code on disk, caught only on the *next* run, after the
developer already had it.

The worktree deletes the entire problem. The mutated copy is disposable, so there
is no invariant to protect: no snapshot comparison, no staging, no path-based
fence, and the colocated-test hazard (which broke every path fence, because in
Rust/Zig/D the test file *is* under `src/`) becomes irrelevant — you mutate a copy
you are about to throw away.

The same crash the write-fence could not survive can still strand the *worktree
itself* — the disposable checkout, plus its `.git/worktrees/` record — on any exit
that skips step 5: a failure-table short-circuit, context exhaustion, a `--watch`
runner, a hard kill. That is hygiene, not safety — the developer's tree is never
touched, so inviolate #2 holds regardless — but the leak is closed the way the
write-fence's window was *not*: on the next entry, never the current exit. Step 1's
sweep (`git worktree prune` plus a force-remove of test-roadmap's own prefix) runs on
the one path a crash cannot skip, so a strand from a dead session dies at the start of
the next one. On-exit removal (step 5) is only the fast path; the sweep is the
guarantee, because no on-exit cleanup survives the crash this design already refused
to trust once.

The cost, stated honestly: a fresh worktree has no `node_modules`, no `target/`,
no venv, no `_build`. Some ecosystems need a full install before tests run, which
can be minutes. The skill cannot know the install command stack-agnostically, so
the mitigation is structural — one worktree per phase, reused across all mutations
(protocol step 1) — not a built-in table.

## Approaches vs findings

The two are handled differently, and conflating them was a live error in an
earlier draft.

**Approaches** — genuine forks where the answer is taste or is expensive to
reverse. There are roughly four to six across a whole run: test framework/runner,
suite strategy (unit-first bottom-up / e2e-first top-down / risk-first), phase
ordering, and whether to rewrite the weak tests grading found.

Every approach is presented as a menu, unbatched, and every menu contains:

- each option, with **pros and cons**;
- a **recommendation** and the **reason** for it;
- a final **deep dive** option to **dispatch a subagent for an adversarial
  pushback review against the presented options** — challenging the menu itself
  and surfacing any better option it missed.

Every menu assumes the developer **does not know this repository** — legacy and
unfamiliar code is the case the skill exists for, so the person answering may be
seeing the repo for the first time. The options and the recommendation are
therefore written to be answerable with no prior knowledge of the codebase: the
recommendation must be strong enough to follow blind and justified by what *this
repo actually is* (detected stack, existing tests, structure), not by generic
preference. This premise governs every question the skill asks, not only these
menus.

**Everything the skill says to the developer is in plain language.** The
developer knows nothing about the skill's internals, so its working vocabulary —
`break-it-check`, "the gate," "latch," "mutation," "the ledger",
`boundary`/`scaffold`, "theater," `Landed:`, "build/execute mode" — stays inside
the reference files and is *translated* whenever it reaches the developer: name
what you're doing in ordinary words, report findings as what they mean for the
tests, and gloss any artifact term on first use. The deep-dive option, for
instance, is *presented* as "have a second, skeptical pass challenge these
options and look for a better one," not as "dispatch an adversarial-pushback
subagent." Single-sourced in `test-pushback.md § Talking to the developer`; the
mode files and `break-it-check.md` point at it.

That format is specified once in `test-pushback.md § Presenting approaches`; both
mode files point at it, so the router stays dumb.

**Not on the menu — recorded defaults.** Some things look like choices but have a
dominant default and are cheap to reverse, so they are *decided* and recorded in
`## Decisions`, not voted on:

- **Test organization** — a *detected fact* about how the repo already separates
  tiers, not a taste choice, so it is recorded, not menued (menuing it would breach
  the 4–6 ceiling requirement 1 leans on, for a decision that isn't the developer's
  to make). The organization must leave the tiers distinguishable and
  independently coverage-runnable (see *The tiers must be independently runnable*),
  but the *form* is whatever the ecosystem expresses — a tier directory only where
  it uses an external, path-selected test tree, a tag/marker/build-target/label
  elsewhere. An earlier version argued against a "group by tier" layout because two
  phases sharing a tier directory would manufacture a false-landed collision; that
  reasoning is now spent — the completion model was rewritten so `Landed:` is the
  sole signal, nothing infers completion from a directory's contents, so two phases
  in one tier directory each latch via their own `Landed:` line and collide on
  nothing. Tier grouping is safe; it just isn't a *menu* item, because it is
  detected, not chosen.

**Findings** — verdicts derived from evidence: weak-test classifications, mock
classifications. These are not approaches. Asking a human to choose between
`boundary` and `scaffold` is asking them to vote on a fact. Findings go to the
ledger, consistent with Stage 4's treatment (*findings are fixed inline; no report
is written*).

**`scaffold` mocks are batched, not surfaced one-by-one.** An earlier draft
surfaced every `scaffold` mock individually so its retiring refactor could be
named. On a legacy codebase — the case that generates the most `scaffold` mocks —
that is dozens of serial decisions in one sitting, which fails requirement 1. The
mocks are recorded to the ledger en bloc; the retiring refactor for each is named
*when that refactor is actually planned*, which is both less overwhelming and a
better moment to name it than during a grading fan-out.

**Silence defaults to `scaffold`.** The harm is asymmetric — a `scaffold` silently
promoted to `boundary` hides test debt permanently, whereas a `boundary`
mistakenly recorded as `scaffold` merely leaves a note someone deletes. Defaulting
this direction makes inattention safe, which is a mechanism rather than an appeal
to how carefully a human will read a list.

**Decisions are recorded** in a `## Decisions` section at the top of
`docs/test-roadmap.md`. Execute mode reads it and never re-asks. Without this,
menus fire again on every resume and requirement 2 fails through the front door.
Recorded at minimum: the test framework/runner, suite strategy, phase ordering,
the weak-test rewrite decision, the test-organization default, and the per-tier
run + coverage commands (see next).

## The tiers must be independently runnable

A requirement from the motivating source, made explicit: the suite must leave the
three tiers (1) **distinguishable** — a developer new to the repo can tell which
tier a test belongs to — and (2) **independently coverage-runnable** — coverage
can be run over just one tier.

Distinguishability is carried by the `Tier:` field on every phase (and by the
on-disk layout where the ecosystem already separates tiers there). Independent
coverage is carried by **recording, per tier, the command that runs it with
coverage** — because per-tier coverage is *not* a universal "point the coverage
tool at a directory" operation:

- Coverage instruments the **source under test**, never the test files, so
  pointing a coverage tool at a directory of tests does not scope coverage to a
  tier. What scopes it is the tier's *selector* applied to the coverage run.
- That selector is a directory only in ecosystems with external, path-selected
  test trees (Python, PHP, Ruby, Perl). Elsewhere it is a tag (Go `-tags`,
  `pytest -m`), a build target (`cargo test --lib`/`--test`, a `.csproj`, a Jest
  project), or a label (CTest `-L`).
- In colocated-test ecosystems the tier directory **cannot exist**: Go `_test.go`
  files must live in the package under test, and Rust unit tests are
  `#[cfg(test)]` modules compiler-bound inside `src/`. Both reproduced by
  construction — see *Verification notes*.

So the skill records the *commands*, not a layout. The only invariant that holds
across every ecosystem is "there is a command that runs exactly one tier with
coverage," and its selector is detected in Stage 1, never assumed. This is why an
earlier layout-mandate framing (tier directories as the default shape) was
dropped: it hardcodes one selector — the path — as if universal, which is the
per-ecosystem-table failure the stack-agnostic charter forbids, and it is outright
impossible in Go and Rust.

## Test-double & fixture ledger

Every test double and fixture, existing or proposed, is classified. "Ledger" below
is shorthand for this section.

| Class | Meaning |
|---|---|
| `boundary` | Permanent and correct. A real external edge: network, clock, randomness, payment gateway, third-party API, filesystem where relevant. |
| `scaffold` | Exists only because the code resists testing. Test debt. **Names the refactor that retires it** — deferred to when that refactor is planned. |
| `data` | Constructed test data that is permanent and correct: a seeded account, a factory-built order, a fixture record. Neither an external edge (`boundary`) nor debt a refactor retires (`scaffold`). |

The `boundary`/`scaffold` distinction is the point: a `scaffold` mock silently
promoted to permanent is how a codebase's testability problems become invisible.
Recording the retiring refactor makes the debt legible.

`data` exists because test data is a third thing the two-class ledger could not
name — a seeded account is not an external boundary and not scaffolding a refactor
removes. Without this class, integration and e2e phases had no vocabulary for their
own inputs, which is why an earlier draft never noticed it needed a test-data
strategy at all.

Known gap, accepted: a `scaffold` mock's retirement has no automatic completion
signal. Whether "the refactor that retires this mock" happened is not observable
by any check in this design. It is tracked for humans, not machines.

## Test-data strategy

Integration and e2e tests — two of the three tiers this design commits to — cannot
exist without constructed data: an account, in a state, with a history. Where that
data comes from is a real decision, and three parts of the design silently assumed
it was already solved until this review:

- **Stage 1 detects** the existing mechanism (fixtures, factories, seeds,
  testcontainers) from repo signals.
- **Stage 3 consumes it.** A phase needing data against a repo with no mechanism
  gets a surfaced note, not a silent plan.
- **`break-it-check` runs it.** An integration test's data setup runs in the
  throwaway worktree too; if it needs a database the worktree lacks, that surfaces
  at the gate — which is why Stage 3 flagging it first matters.

The `data` ledger class (above) is the vocabulary. The inspiration is the fixture
discipline in *The Zen of Test Suites* — *"fine-grained, well-documented fixtures
which are easy to create and easy to clean up"* — taking the problem it names while
deliberately **not** taking its transaction-rollback prescription.

## Suite-health preconditions

The isolated gate depends on two properties of the phase's tests that nothing else
in the design establishes: **isolation** (they pass run alone, not only as part of
the full suite) and **determinism** (their verdict is stable). The two new rows in
the failure table detect violations of each and surface them.

This is the *entire* extent of the skill's engagement with the suite as a running
artifact. Speed, parallelization, warning censuses, and fixture performance — the
rest of what *The Zen of Test Suites* covers — are a different tool and a non-goal.
The skill names the preconditions its own gate needs and stops there; it does not
become a test-suite optimizer.

## Phase format

```markdown
## Phase 3: Billing retry & dunning integration tests

Tier:     integration
Catches:  a retry exhausting without transitioning the account to dunning;
          a partial refund double-crediting.
Produces: tests/integration/billing/
Branch:   billing-integration-tests
Landed:
```

- **`Tier:`** — exactly one of `unit`, `integration`, `e2e`. Stage 3 already groups
  behaviors by tier, so this is honestly authorable at plan time (unlike the
  rejected `Covers:` field). It makes which tier a phase belongs to explicit in the
  roadmap — the distinguishability half of the tier requirement, independent of
  where files land — and selects which recorded per-tier run/coverage command
  applies. See *The tiers must be independently runnable*.
- **`Catches:`** — the anti-theater gate from Stage 4, preserved in the artifact so
  a later reader can audit whether the phase was honest. It is also
  `break-it-check`'s input: it names the *behavior* to mutate. It deliberately
  does not name paths — a path is neither necessary (the behavior locates the
  code) nor sufficient (directory granularity cannot pick a line to mutate).
- **`Produces:`** — paths the phase creates, **as documentation only**. It is no
  longer a completion signal (see *Completion model* — that inference was removed).
  A later reader uses it to find the phase's output; the control flow does not.
- **`Branch:`** — a human-readable breadcrumb recording the working branch the
  phase's tests were committed on. **Not a machine signal, and not a per-phase
  branch the skill creates.** Execute mode commits each phase onto the current
  working branch; the suite accumulates there. The skill deliberately does not
  spin each phase onto its own unmerged branch — unmerged local branches do not
  survive a fresh clone (the same failure that sank branch-merge detection as a
  completion signal), so putting the test *files* there would strand the suite
  off the working branch and break requirement 2. Per-phase review survives
  anyway: each phase is its own commit (`git show <Landed sha>`), and
  `agentic-review` runs against the accumulated branch before it merges upstream.
- **`Landed:`** — empty until `break-it-check` passes, then
  `YYYY-MM-DD <sha> (<operator>)`. The operator name is gate provenance: a later
  reader can judge whether the gate was honest without opening any other file. It
  is self-reported, so it records a claim rather than proving one — worth having
  only because a named claim is costlier to fake than a silent one.
  **`Landed:` is human-clearable; the skill itself never clears it.** See
  *Completion model*.

Each phase's definition-of-done includes a recommendation to run `agentic-review`
on the accumulated working branch before it is merged upstream, if available. It is not bundled: it requires a fresh
session with no substantive history, so invoking it from inside execute mode would
be inert.

## Completion model

### Problem being solved

A hand-written status label goes stale the moment work is merged outside the agent
session. A fresh session reads the stale label and asks the human whether
already-finished work is finished.

An earlier draft solved this with a `Produces:`-path signal: path present →
corroborate; path *absent* → treat as proof the phase never started, and execute it
silently. That inference is the same class of error the design elsewhere rejects
(branch-merge detection, commit-log scanning): it infers "did this land" from repo
state the skill does **not** control. It fails both directions —

- **toward re-churn:** a directory renamed in a reorg, or a fresh clone of a
  repo whose layout changed, makes landed work look absent, so the skill silently
  re-executes it; and
- **toward silent skip:** two phases sharing a directory (any "group by tier"
  layout) makes an unstarted phase's path look present, so corroboration can find
  the *other* phase's commit and report it landed — skipping real work, the one
  direction the governing principle forbids.

The skill already owns a ledger that the control flow already reads: the `Landed:`
line. The path apparatus existed only to reconstruct, from paths the skill does not
own, information the ledger already holds. So it is removed.

### Governing principle

**No signal auto-marks anything done, and no absence is ever proof of not-done.**
Signals decide whether to ask and supply the evidence attached to the asking. Being
wrong toward "ask with evidence" costs one keystroke. Being wrong toward "auto-mark
done" silently skips real work; being wrong toward "silently re-execute" wastes it.
Only the first of those two is a safety violation, but the design avoids both where
it cheaply can.

The original bug was never that the agent asked. It was that it asked
empty-handed — and a jargon-laden question fails the same way for a developer new
to the repo. The question must carry its evidence *and* be in plain words
(*Talking to the developer*):

- Bad (empty-handed): *"Phase 3 shows Pending. Is it complete?"*
- Bad (jargon): *"Phase 3 declares `tests/integration/billing/`; its `Landed:`
  line is empty — did it land, or should I execute it?"*
- Good: *"I don't see tests yet for **billing retries** (I'd add them under
  `tests/integration/billing/`). Did someone already write these — maybe on a
  branch called `billing-integration-tests` — or should I write them now?"*

The phase's recorded details supply the evidence; the developer supplies the answer.

### Protocol

`Landed:` is the sole completion signal. For the candidate phase only:

| Step | Where | Action |
|---|---|---|
| 1 | main agent | `Landed:` populated? → done. Stop. No git, no question. |
| 2 | main agent | `Landed:` empty → ask the developer, in plain words (*Talking to the developer*), whether these tests were already written or should be written now — using the phase's recorded details as evidence and pointing them at where to check. On *"write them,"* run the phase. On *"already done,"* record it from what they report. |

Step 1 ends the churn permanently: once latched, the phase is never re-examined by
the control flow — *except* that `Landed:` is human-clearable. The skill never
clears it (so no agent *inference* can un-land work — the property the old
"monotonic latch" was protecting), but a human who learns a phase's tests were
condemned by `agentic-review`, or gamed the gate, can clear the line by hand and
the phase re-enters the flow. This removes the design's only terminal, unfalsifiable
state without handing the agent a lever to un-land its own work.

Step 2 asks rather than infers. The phase block already on screen *is* the evidence
the asking needs — the original bug was asking empty-handed (a bare status label),
not asking at all. The developer who ran `/test-roadmap` is in the loop and answers
in one word; if they don't recall, they can check `git log` themselves, with the
context to read it that a blind corroborator lacked. Absence of a "yes" means
**execute** (the common greenfield case), never silent skip — the safe direction.

Work this skill does itself latches directly after `break-it-check`; step 2 fires
only for work that may have landed *outside* an agent session — the one case an
in-loop human answers for free.

### Why not branch-merge detection

Considered and rejected as a signal. Verified behavior:

| Merge style | `git branch --merged main` |
|---|---|
| `--no-ff` merge commit | works |
| Squash merge (common default) | **fails** — branch tip is not an ancestor |
| Rebase merge | **fails** — commits rewritten, new SHAs |
| Branch deleted after merge | **fails** — nothing left to query |
| Fresh clone | **fails** — no local branches exist |

Four of five cases fail, and all fail *toward "not done"* — regenerating churn.

### Why not commit-log scanning, or `Produces:`-absence

All rejected as *primary* signals for the same reason: they infer completion from
state the skill does not control. Matching phase titles against commit messages is
string-matching human prose. `Produces:`-absence infers not-started from a path
layout that renames, collides, and (in colocated ecosystems) never exists at all.
Git log answers *when* and *which commit* well and *whether* badly — so the skill
does not use it as a signal at all. `Landed:` is the signal; a human who wants to
know when a phase landed runs `git log` themselves, with the context to judge what
it means.

### Why not an anchor commit or a `Covers:` field

Proposed and rejected: record `generated-at: <sha>` plus a per-phase `Covers:` field
naming production paths, then diff `<sha>..HEAD` against `Covers:` on resume to
detect a phase planned against moved code.

Rejected on two grounds:

1. **It fails silent.** A change to a transitive dependency outside the listed
   paths yields an empty intersection, so the skill reports "no drift" for an
   invalidated phase. Closing the hole needs per-language transitive-closure
   analysis, contradicting the stack-agnostic requirement.
2. **`Covers:` is not honestly authorable.** Stage 3 produces behaviors and phase
   groupings; it never enumerates production paths. The field would be filled by
   inference, failing silent in the same direction.

`break-it-check`'s "no code path implements this `Catches:`" row already detects the
same condition, terminally and loudly, minutes later and at no authoring cost.

## Accepted limitations

1. `Landed:` answers "did the work land," not "is it any good." Grading answers the
   latter for existing tests; `break-it-check` answers it for phases this skill
   executes itself. A human-cleared `Landed:` line is the recovery path when the
   answer to "is it good" turns out to be no.
2. `scaffold` mock retirement has no automatic completion signal.
3. Coverage-based gap-finding depends on a detectable coverage tool. Where none
   exists, Stage 3 falls back to agent judgment and says so.
4. The skill detects suite-health precondition *violations* (order dependence,
   flakiness) but does not repair them; it surfaces them for the human.
5. One `test-roadmap` run per repo at a time. Two concurrent runs share the
   `.git/test-roadmap-worktrees/` prefix, so one's startup sweep can force-remove the
   other's in-flight worktree and spoil its gate. Consistent with requirement 1
   (single developer, one command, *sequential* resume), and bounded — the blast
   radius is the skill's own scratch, never the developer's other worktrees or
   production code.

## Decision log

| Decision | Chosen | Rejected |
|---|---|---|
| Scope | Plan **and** execute | Plan only; plan + execute + weak-test gate |
| Packaging | One skill, thin router, on-demand references | Two skills (build + execute); depend on the `roadmap` skill |
| Existing tests | Grade them; fold findings into the roadmap | Baseline only; separate audit skill |
| Format | Agent Skill (agentskills.io) | Kiro power |
| Dependencies | Self-contained, pushback-style critique inlined | Hard-depend on roadmap + pushback |
| Grading | Fan out by tier/directory | Single-pass sampling; conditional fan-out |
| Grading volume | Full list → `docs/test-suite-analysis.md`; counts + top-K → main context | Full list into main context |
| `superpowers:test-driven-development` | Not bundled | Bundled into execute mode |
| `paad:agentic-review` | Recommended per phase, not bundled | Bundled into the skill |
| `superpowers:verification-before-completion` | Adapted into `break-it-check.md` | Inlined as a two-sentence rule |
| Suite acceptance criterion | Enforced by `break-it-check` | Stated in the design and never enforced |
| Bug injection location | Throwaway `git worktree` copy | In the developer's working tree behind a write-fence |
| Write-fence | None needed — the mutated copy is disposable | Whole-tree `git status --porcelain` snapshot; `git add`-then-`checkout` staging; path fences |
| Worktree cleanup | Sweep-on-entry (`prune` + force-remove own prefix) as the guarantee; `--force` remove as the fast path | On-exit `git worktree remove` as the guarantee (a crash skips it — the write-fence's flaw again); bare `remove` (fails on the always-dirty worktree) |
| Mutation constraint | Named operator set; early-return/whole-body-removal banned; operator declared by name | Free-form agent-invented mutation |
| Mutation authorship | Blind mutator subagent (never sees the test); first hunk used | Same agent authors test and mutation; cross-behavior green-check; record diff to file |
| Coverage | Stage 3 gap-finding only, read-only, never a quality signal | Coverage as a test-quality gate; mutation-only with no gap-finding |
| Gate preconditions | Name isolation + determinism where the gate needs them; surface violations | A suite-health subsystem; ignore them |
| Test data | Detect mechanism in Stage 1, feed Stage 3, add `data` ledger class | Put test-data strategy on the approach menu; scope it out entirely |
| Completion signal | `Landed:` sole signal, human-clearable, ask the human on empty | `Produces:` as signal; branch-merge; commit-log; status field; monotonic terminal latch; git-corroboration subagent |
| Out-of-session landing | Ask the in-loop human (the phase block is the evidence) | Git-corroboration subagent (`--diff-filter=A`, verdict-only, injection-hardened) |
| `Landed:` latch | Human-clearable, gate provenance on the line | Monotonic/terminal; explicit `redo` command |
| Staleness detection | `break-it-check`'s "no code path implements this `Catches:`" row | `generated-at:` anchor sha + per-phase `Covers:` |
| Approach menus | Every approach (~4-6 per run), unbatched, always offering adversarial pushback | Two fixed gates only; menus on request only |
| Test organization | Recorded default in `## Decisions`; detected, not menued | An approach-menu item |
| Tier distinguishability + per-tier coverage | `Tier:` field per phase + detect the tier *selector* and record per-tier run+coverage *commands* | Mandate `unit/ integration/ e2e/` directories (impossible in Go/Rust; assumes path-selector universally) |
| Run instructions on completion | Execute mode surfaces how to run this phase's tests, the full suite, and per-tier coverage when a phase lands | Assume the developer knows the repo's commands and leave them to find them |
| Talking to the developer | Plain language everywhere; internal vocabulary (`break-it-check`, "gate", `boundary`/`scaffold`, `Landed:`) translated on the way out; single-sourced in `test-pushback.md` | Let internal codenames and field labels reach the developer as-is |
| Test output cleanliness | Tests produce a clean run; meaningful output is captured and asserted, uncontrollable noise suppressed narrowly | Let tests emit warnings/chatter (buries signal; trains the developer to ignore warnings that may hide bugs) |
| Finding menus | None — ledger; `scaffold` batched, retire-refactor named when planned | Menu per finding; `scaffold` surfaced individually up front |
| Menu audience | Every menu assumes zero repo knowledge; recommendation must be followable blind and grounded in this repo's detected facts | Assume the developer knows the codebase; generic-preference recommendations |
| Build-mode artifacts | Stage 5 commits `docs/test-roadmap.md` + `docs/test-suite-analysis.md` | Leave them uncommitted (a fresh clone loses the roadmap → silent rebuild, breaking requirement 2) |
| Phase integration | Commit each phase onto the current working branch; suite accumulates in place | Skill spins each phase onto its own branch left unmerged (strands the suite off the working branch, dies on fresh clone — same flaw as branch-merge detection) |
| Rubber-stamp safety | Asymmetric default: silence → `scaffold` | Argue that batching preserves attention |
| Decision persistence | `## Decisions` section in the roadmap | Re-ask on each resume |
| Units too hard to test | Skipped stub naming the obstacle, surfaced via the framework's native skip-with-reason (one-line STDERR only where none exists); last resort after real test + scaffold-mock both infeasible; a gap-marker — never through `break-it-check`, never toward `Landed:`, never a phase on its own; recorded as `scaffold`-class debt | A new 4th ledger class (`scaffold` already covers code that resists testing); STDERR print as the primary channel (fights the clean-run rule — a skip-with-reason is the standard-tooling channel that rule endorses); silently omitting the unit (an absent test looks like one nobody needed); counting skips toward completion or mutating them (a skip asserts nothing, so it can neither latch a phase nor pass the gate) |
| Approach-menu rendering | Menu substance in a prose reply (one decision, pro+con per option, recommendation, deep-dive last); answer collected by **one blocking single-question picker** (options as labels, deep-dive included) so *one at a time* and *wait* are tool-enforced. Compact reflex restated at the point of use in build mode; full format single-sourced in `§ Presenting approaches` | Point only at `§` in the on-demand reference file (field run rendered via the harness picker: batched two decisions as tabs, dropped pros/cons and the deep-dive option); reword `§` (a micro-test showed the wording already yields compliance when in context — clarity was not the gap, salience was); **pure prose menu with an instructed "then wait"** (waiting-by-instruction is the same soft-rule fragility class as the salience miss — a bare prose menu does not stop the run); **picker carrying the whole menu** (the original field failure — batches, drops three of four parts) |

Four `/pushback` reviews produced this table. The first killed a `Done-when:` shell
predicate (no consumer executed it, exit-0 is not evidence, shell in checked-in
markdown is an injection surface). The second, four adversarial rounds, killed the
`Covers:` field (fails silent, not authorable), the path-based write-fence (fails
open in colocated-test ecosystems, would have destroyed the tests it wrote), TDD in
execute mode (destructive for characterization), and the batching-preserves-attention
argument (replaced by the asymmetric default). The third produced the seven findings
in this revision: no green baseline for the gate; unconstrained mutation letting a
sledgehammer certify theater; an unfalsifiable `Landed:` latch; `Produces:`-absence
as a same-class inference error that both re-churns and silent-skips; unbounded N
leaking into context and menus; the gate's unnamed isolation/determinism
preconditions; and the absent test-data strategy. Two of that round's own
recommendations were reversed by its adversarial subagents — a cross-behavior
mutation check (measures coupling, not honesty; voided by editing the `Catches:`
line) and a per-`Produces` completion fix (the skill should own the ledger, not
infer from paths).

The fourth was an *overthinking* pass — the first review aimed at subtraction rather
than hole-plugging, since the prior three had only ever added mechanism. It cut the
completion model's git-corroboration subagent: the developer who ran `/test-roadmap`
is in the loop and answers *"did this land?"* for free, so the subagent's
`--diff-filter=A` archaeology and its commit-message injection surface were machinery
serving one edge case (out-of-session landing), and their removal deleted two accepted
limitations outright. It also renamed the "mock ledger" to the "test-double & fixture
ledger": an adversarial subagent showed that cutting the `data` class — the tempting
deletion — would strand the per-item classification Stages 2 and 4 mandate, reopening
the misfiling hole `data` closed, so the fix was the category error's actual size (one
heading), not the class.

## Verification notes

Recorded because several decisions rest on claims about tool behavior, and an
unverified claim that looks verified is how a design acquires a defect that survives
review.

**Reproduced directly, in a scratch repository:**

- Colocated test/production files: a Rust `#[cfg(test)] mod tests` block lives inside
  `src/foo.rs`. This is why `break-it-check` step 2 copies the test into the
  worktree *before* mutating — the test file and the mutation target are the same
  file.
- `git worktree remove` refuses an unclean worktree — `fatal: '<path>' contains
  modified or untracked files, use --force to delete it` — so `break-it-check` step 5
  must pass `--force`, because the worktree is unclean by construction (copied-in test
  is untracked, injected mutation is a tracked-file modification). `--force` succeeds.
- `git worktree prune` reclaims a stale `.git/worktrees/` record after the checkout
  dir is deleted out from under git (`gitdir file points to non-existent location`).
  This is what makes step 1's sweep self-healing after a crash.
- **Per-tier coverage is not a directory-path operation in Go or Rust** (scratch
  repos `gomod`, `rustcrate`; Go 1.26.5, Rust/cargo 1.97.1). Reproduced: (a) a Go
  unit test moved out of its package into a `unit/` dir fails to compile —
  `name add not exported by package calc`; (b) `go test -tags=integration` selects
  the integration tier while both tiers share a directory, and coverage scope is
  set by `-coverpkg` (source packages), reporting on `calc/calc.go`, never on a
  test dir; (c) a Rust unit test calling a private `helper` compiles inline via
  `#[cfg(test)]` but fails from `tests/foo.rs` — `error[E0603]: function 'helper'
  is private`; (d) `cargo test --lib` vs `--test foo` selects the tier by build
  target, and instrumented coverage (`-C instrument-coverage` + `xcrun llvm-cov`)
  reports on `src/lib.rs` for both, differing only by which build-target binary
  ran. Together these reproduce that "per-tier coverage = point at a tier
  directory" is false-to-impossible outside path-selected ecosystems — the basis
  for recording per-tier *commands* rather than mandating a layout.

**Discipline rules pressure-tested against subagents (2026-07-23, RED-GREEN per
`superpowers:writing-skills`):** three of the skill's discipline claims were run as
single-turn decision scenarios, each with a no-guidance control (RED) and a
with-skill arm (GREEN), 3–4 reps each:

- **The `break-it-check` gate** (verify by mutation before landing) — control 8/8
  chose to verify, including a hardened arm where authority explicitly said "green
  is green, don't break your own code" and it was framed as non-team-practice.
- **Characterize, don't fix** (pin current behavior; don't fix a bug spotted while
  writing the test) — control 4/4 pinned current behavior and filed the bug
  separately, rejecting both the fix and the aspirational assertion.
- **Don't re-roll the mutation** (a test that survives a boundary-targeted mutation
  is theater; rewrite the test, don't shop for a redder hunk) — control 3/3 rewrote
  the test.

Result: **every no-guidance control already complied**, including the two claims
(characterize-don't-fix, don't-re-roll) chosen because they run *against* model
priors — so no reproducible RED, and therefore no skill edit was justified. The
GREEN arms all complied *and* quoted the load-bearing sentence verbatim, which
verifies the rules are legible and discoverable when read (and that the inline
restatement of Decision-log rationale is what agents actually cite). What this does
**not** establish: these are the easiest possible conditions (single model family —
Opus; fresh context; temptation stated cleanly; virtuous option offered as an
explicit choice). It says nothing about long-session drift, or about the skill's
mechanism/structure (worktree, blind mutator, ledger, completion protocol,
resumability) — none of which a decision-snapshot can reach. A separate
end-to-end execution on a real repo is the method for those; it has not been run.

A withdrawn hypothesis from the same session: the concern that the `description`'s
workflow summary would cause agents to skip reading the skill body did not
reproduce — a head-to-head against a triggers-only variant had both invoking the
body 2/2, so the description was left unchanged.

**Reasoned, not reproduced — the approach-menu salience fix (2026-07-23):** a
field run rendered an approach menu through the harness's structured picker,
which batched two decisions as tabs and showed no pros/cons or deep-dive option
— violating `§ Presenting approaches` on three counts. A control-vs-treatment
micro-test (4 + 5 reps) found the *current* `§` wording already yields fully
compliant prose menus when the section is in context (both arms 100%), so the
wording was not the gap. The remaining explanation is **salience**: `§` lives in
an on-demand reference file and was not effectively in context at the menu
moment (deep in a 60-module analysis), and/or the live picker's pull outweighed
diluted guidance. The fix — a compact reflex restated at the point of use in
`build-test-roadmap.md` — targets that, but its mechanism (context dilution,
live-tool pull) is **not reproducible in a subagent snapshot** by construction;
the honest test is an end-to-end run, not yet performed. Filed here as reasoned,
not verified.

The reflex was then revised (same session) from "prose menu, never a picker" to a
**hybrid**: the menu's substance stays in prose (pros/cons, recommendation,
deep-dive), but the answer is collected by one blocking single-question picker.
Reason: a bare prose menu does not *wait* — the agent must remember to stop, which
is the same soft-rule fragility as the salience miss it was meant to fix — whereas
a one-question picker blocks until answered, making *wait for the answer* and *one
at a time* tool-enforced rather than instructed. Also reasoned, not reproduced:
subagents run autonomously to completion and cannot exhibit waiting, so a snapshot
can neither reproduce the no-wait failure nor validate the fix — end-to-end only.

**Read directly in the installed skills:**

- `test-driven-development/SKILL.md:126` — *"**Test passes?** You're testing
  existing behavior. Fix test."*
- `verification-before-completion/SKILL.md:84-88` — the revert-and-confirm-red
  sequence.

**Asserted but not reproduced here:**

- Colocated test/production files in Zig, D, Elixir, and Python/Rust doctests. Only
  the Rust `#[cfg(test)] mod tests` shape was reproduced; one such ecosystem is
  sufficient to force the copy-before-mutate ordering.
- `git worktree add <tmp> HEAD` produces a checkout without gitignored artifacts
  (`node_modules`, `target/`, venv), hence the per-phase dependency-install cost the
  one-worktree-per-phase reuse is designed to amortize. Reasoned from how git
  tracks ignored files, not reproduced.
- `agentic-review`'s fresh-session precondition (reported by a subagent). The
  decision not to bundle it predates this reason.
- Java's tier selector (Surefire runs `*Test`, Failsafe runs `*IT`; JaCoCo binds
  coverage per plugin execution, not by directory) and the other non-path selectors
  in the per-language survey (.NET projects, Jest projects, CTest labels, Elixir
  tags, Swift targets). Standard, documented tool behavior; not reproduced here.
  Go and Rust — the two ecosystems where a tier directory is *impossible*, not
  merely non-idiomatic — were reproduced above, which is what the mechanism claim
  most needed.

**No longer relied upon** (the in-tree write-fence they supported was removed):
`git checkout -- <path>` restoring from the index, and `git diff --quiet` exiting 0
with untracked files present. Both were reproduced for the earlier draft; the
worktree approach makes neither load-bearing.
