# Execute mode — the completion protocol

Loaded when `docs/test-roadmap.md` already exists — the path every run after
the first takes. This is the run that must be quiet about anything the
developer already settled: it reads `## Decisions` once, per phase asks
exactly the one question the completion model requires, and otherwise gets on
with writing tests.

## The execute-mode loop

Six steps, in order:

1. **Read `## Decisions` from the roadmap. Never re-ask anything recorded
   there.** The chosen test framework/runner, suite strategy, phase ordering,
   the weak-test rewrite decision, the test-organization default, the per-tier
   run + coverage commands — all of it was already decided during build mode's
   Stage 3/Stage 5 and is written down for exactly this reason. Re-asking it
   here is not caution, it is the resumability requirement failing through the
   front door. Place the phase's tests according to its `Tier:` and the recorded
   organization, and use the recorded per-tier command when running or covering
   that tier.
2. **Select the candidate phase per the *Completion model* protocol, below.**
   Exactly one phase at a time — the protocol's questioning fires for the
   candidate phase only, never as a batch sweep across every phase in the
   roadmap.
3. **Write the tests for that phase.** This is the only code this step
   writes — see *What execute mode writes*, below.
4. **Run `references/break-it-check.md`.** This gate is mandatory. **No
   phase latches without it** — not a fast path for a phase that "obviously"
   passes, not a phase where the developer is confident the tests are
   right. Skipping this step is skipping the entire mechanism that
   distinguishes this skill's suite from `assert result is not None` a green
   exit code would have waved through.
5. **Write `Landed: YYYY-MM-DD <sha> (<operator>)` and commit on the current
   working branch.** This line is only ever written after step 4 passes — it
   is the gate's own output, not a separate bookkeeping step that could drift
   out of sync with it.
6. **Surface run instructions for the tests just landed.** See *Run
   instructions*, below.

### Run instructions

The developer running this skill may not know the codebase, the test runner, or
even the language. So when a phase's tests are ready, do not leave them to
reconstruct how to run anything. Print a short block, drawn verbatim from the
`## Decisions` per-tier commands (never invented on the spot), giving:

- **The new tests** — the command that runs *just this phase's* tests.
- **This tier with coverage** — the recorded coverage command for the phase's
  `Tier:`, so the developer can see what these tests cover. Where that tier has
  no coverage tool, say so explicitly rather than omitting the line.
- **The whole suite** — the command that runs every test, so they can confirm
  nothing else broke.

A worked shape:

```
Phase 3 (integration) tests are in and committed. To run them:

  just these tests:   pytest tests/integration/billing/
  with coverage:      pytest --cov=billing -m integration
  the full suite:     pytest
```

Keep it to the commands and one label each — this is an aide-mémoire for someone
unfamiliar with the repo, not documentation. The commands are the recorded ones;
if a command has drifted (the runner changed since build mode), that surfaces
here as a failed run the developer can see, not as silent wrong advice.

### What execute mode writes

Execute mode writes **test code only**. It never writes production code, and
it needs no path-based fence to enforce that — the property that matters
here, that mutation never touches the developer's tree, is enforced by
`break-it-check` running its bug injection in a throwaway `git worktree`
(Inviolate #2). Nothing in this mode's own steps stands between it and
production code, so there is nothing here for a path fence to guard.

**Each phase's tests commit onto the current working branch; the skill creates
no per-phase branch.** The suite accumulates in place — as phases land, the
test directory on the branch the developer is already on fills up, and the
`Landed:` SHA points at a commit that branch can actually reach. This is the
literal reading of `break-it-check`'s "commit on the developer's real tree"
(step 6). The skill deliberately does **not** spin each phase onto its own
branch and leave it unmerged: unmerged local branches do not survive a fresh
clone — the very failure the completion model rejects branch-merge detection
over — so putting the test *files* there would strand the suite off the
working branch and break resumability (requirement 2). Per-phase review is not
lost: each phase is its own commit, reviewable with `git show <Landed-sha>`,
and `agentic-review` runs against the accumulated working branch before it is
merged upstream.

### Keep the test run clean

The tests this mode writes must produce a **clean run** — no spurious warnings
or stray output to STDOUT/STDERR. Two reasons, both about the developer: noise
buries the signal (a run that prints twenty lines of framework chatter for one
real failure trains people to skim past it), and a suite that always warns
teaches the developer to ignore warnings — at which point the one warning that
*was* a real bug scrolls past unnoticed.

So, in order of preference:

1. **Treat meaningful output as behavior — capture and assert it.** If the code
   under test emits a deprecation warning, a log line, or a message that *matters*
   (it should fire, or it should *not* fire), that is behavior worth pinning:
   capture it and assert on it, the same as any return value. A warning you
   assert on is no longer noise — it's a test.
2. **Suppress noise you can't assert, at the test boundary.** Third-party
   chatter you can't control and don't need to pin gets silenced narrowly in the
   test setup — not by muting all output globally, but scoped so that a *new*,
   unexpected warning still stands out.
3. **Never let the tests you write add their own noise** — no leftover debug
   prints, no `warn`/`console.log` scaffolding shipped in the committed test.

If clean output genuinely isn't achievable for a phase (the code is
noisy by construction and the noise can't be captured or scoped), say so plainly
to the developer rather than shipping a suite that cries wolf.

### Units too hard to test

Some units resist testing so completely that neither a real characterization
test nor a `scaffold`-mock test (see the ledger in `build-test-roadmap.md`) can
be written honestly. When that happens — and **only as a last resort, after both
a real test and a scaffold-mock have been judged infeasible, not merely
inconvenient** — write a **skipped stub test** that names the obstacle, rather
than silently leaving the unit out of the suite. An untested unit that is simply
absent looks like one nobody needed to test; a skipped stub that says *why* keeps
the gap visible.

**Surface it through the framework's own skip mechanism, with a reason** —
`pytest.skip(reason=…)`, `@Disabled("…")`, `t.Skip("…")`, whatever the detected
framework provides — so the message rides with the test and shows in the run
summary and CI, the same standard-tooling channel *Keep the test run clean* uses
for meaningful output. The reason states in plain language that the unit is hard
to test and what makes it so, e.g. *"skipped — builds its own DB handle in the
constructor; untestable until that dependency is injected."* Only where the
detected framework has no machine-surfaced skip reason does this fall back to a
single one-line STDERR statement — never a wall of prints.

A skipped stub is a **gap-marker, not coverage**:

- It **never goes through `break-it-check`** — nothing is asserted, so there is
  nothing to mutate.
- It **never counts toward a phase's `Landed:` line.** A phase latches on its
  real, bug-catching tests; a skip latches nothing.
- **A phase made only of skipped stubs catches no bug and is dropped as theater
  (Inviolate #5), not landed.** Skips ride *alongside* real coverage; they never
  constitute a phase.
- The obstacle is also recorded as `scaffold`-class debt in the ledger /
  `docs/test-suite-analysis.md`, naming the refactor (usually a dependency-
  injection seam) that would make the unit testable and retire the skip.

When the phase's tests land, tell the developer in plain words how many units
were too hard to test and why, so the skips are a decision they can see rather
than a silent omission.

## Why TDD is not bundled

`superpowers:test-driven-development` is deliberately **not** loaded by
execute mode, even though this mode is, superficially, "write a test, run it."

For a characterization phase — the case this skill exists for — the code
already works. The test step 3 writes therefore passes on its first run *by
construction*: there is no bug yet to make it fail, because none has been
injected. Verified at `test-driven-development/SKILL.md:126`:

> **Test passes?** You're testing existing behavior. Fix test.

Applied literally to a characterization test, that instruction is not merely
inapplicable — it is destructive. The test passing on its first run is
correct behavior here, not a signal that the test is wrong, and an agent
following that rule would rewrite a correct characterization test until it
fails for no reason, or invent a reason to make it fail. Either way it
corrupts the one thing this phase was supposed to produce: an honest pin on
current behavior.

The red phase a characterization test actually needs does not come from
authorship — writing the test a second, "more red," way — it comes from
**mutation**: inject a real bug and confirm the existing test catches it.
That is exactly what `break-it-check` (step 4, above) provides, and it is why
this mode reaches for that gate instead of for TDD's red-green-refactor loop.

## Completion model

### Problem being solved

A hand-written status label goes stale the moment work is merged outside the
agent session — another developer lands the branch, or the same developer
merges it between sessions. A fresh session with no memory of the previous
one then reads the stale label and asks the human whether already-finished
work is finished, which is exactly the churn requirement 2 (resumability)
exists to prevent.

An earlier draft solved this by inferring completion from a `Produces:` path:
path present on disk → treat the phase as landed; path absent → treat that as
proof the phase never started, and execute it silently. That is repo state
the skill does not control, and it fails in both directions:

- **Toward re-churn:** a directory renamed in a reorg, or a fresh clone of a
  repo whose layout changed since, makes landed work look absent, so the
  skill silently re-executes it.
- **Toward silent skip:** two phases sharing a directory — any "group by
  tier" test-organization layout can produce this — makes an unstarted
  phase's path look present, so the inference finds the *other* phase's
  output and reports the unstarted one as landed. That is the one direction
  the governing principle below forbids outright.

### Governing principle

**No signal auto-marks anything done, and no absence is ever proof of
not-done** (Inviolate #4). Signals decide only whether to ask, and supply the
evidence attached to the asking. Being wrong toward "ask with evidence" costs
one keystroke. Being wrong toward "auto-mark done" silently skips real work —
the one outcome the whole design is built to keep from happening quietly.

The original bug behind the `Produces:`-path draft was never that the agent
asked the human something. It was that it asked **empty-handed** — and, just as
bad for a developer new to the repo, in the skill's own jargon. The question
must carry its evidence *and* be in plain words (see `references/test-pushback.md
§ Talking to the developer`):

- Bad (empty-handed): *"Phase 3 shows Pending. Is it complete?"*
- Bad (jargon): *"Phase 3 declares `tests/integration/billing/`; its `Landed:`
  line is empty — did it land, or should I execute it?"*
- Good: *"I don't see tests yet for **billing retries** (I'd add them under
  `tests/integration/billing/`). Did someone already write these — maybe on a
  branch called `billing-integration-tests` — or should I write them now?"*

The phase's recorded details supply the evidence; the developer supplies the
answer. Because they may not know the repo's history, tell them where to look if
unsure — *"you can check with `git log tests/integration/billing/`, or see
whether those test files already exist"* — so the question is answerable without
prior knowledge of what has and hasn't been done here.

### Protocol

`Landed:` is the sole completion signal, and this protocol runs for the
**candidate phase only** — never as a sweep across the whole roadmap.

| Step | Where | Action |
|---|---|---|
| 1 | main agent | `Landed:` populated? → done. Stop. No git, no question. |
| 2 | main agent | `Landed:` empty → ask the developer, in plain words (§ Talking to the developer), whether these tests were already written or should be written now — using the phase's recorded details as evidence and pointing them at where to check. On *"write them,"* run the phase (the execute-mode loop, above). On *"already done,"* record it from what they report. |

Step 1 ends the churn permanently for a phase that has already latched: once
`Landed:` is populated, the control flow never re-examines that phase again —
**except** that `Landed:` is human-clearable (see below), which is a human
decision, not something step 1 itself does.

Step 2 asks rather than infers, and **absence of a "yes" means execute** — the
safe direction — never a silent skip. This is the case that fires only for
work that may have landed *outside* an agent session; work the skill lands
itself goes straight from `break-it-check` passing to a written `Landed:`
line, no question asked, because there the skill *is* the authority on
whether it happened.

**`Landed:` is human-clearable; the skill itself never clears it.** A human
who learns a phase's tests were condemned by a later review, or that the gate
was gamed, can clear the line by hand, and the phase re-enters the flow at
step 2 on the next run. No agent inference is ever the thing that un-lands
work — only a human, deliberately, gets that lever.

### Why not branch-merge detection

Considered and rejected as a signal. Verified behavior of
`git branch --merged main`:

| Merge style | Result |
|---|---|
| `--no-ff` merge commit | works |
| Squash merge (common default) | **fails** — branch tip is not an ancestor |
| Rebase merge | **fails** — commits rewritten, new SHAs |
| Branch deleted after merge | **fails** — nothing left to query |
| Fresh clone | **fails** — no local branches exist |

Four of five cases fail, and all four fail **toward "not done"** — the
skill would regenerate work already landed, which is churn, not safety, but
churn that the resumability requirement was written to prevent.

### Why not commit-log scanning, or `Produces:`-absence

Rejected for the same underlying reason: both infer "did this land" from repo
state the skill does not control. Matching a phase's title against commit
messages is string-matching human prose that was never written to be a
machine contract. `Produces:`-absence infers "not started" from a path
layout that renames, that two phases can collide on, and that — in a
colocated-test ecosystem — may never independently exist at all. Git log
answers *when* a thing landed and *which commit* did it well; it answers
*whether* badly. So the skill does not use it as a signal. `Landed:` is the
signal; a human who wants to know when a phase landed runs `git log`
themselves, with the surrounding context to read what it means — context a
blind scan does not have.

### Why not an anchor commit or a `Covers:` field

Considered: record a `generated-at: <sha>` and a per-phase `Covers:` field
naming the production paths it depends on, then diff `<sha>..HEAD` against
`Covers:` on resume to detect a phase planned against code that has since
moved.

Rejected on two grounds:

1. **It fails silent.** A change to a transitive dependency outside the
   listed paths produces an empty intersection, so the skill reports "no
   drift" for a phase that is actually invalidated. Closing that hole would
   require per-language transitive-closure analysis, which contradicts the
   stack-agnostic requirement this design is built to hold.
2. **`Covers:` is not honestly authorable.** Stage 3 produces behaviors and
   phase groupings; it never enumerates the production paths behind them. The
   field would have to be filled in by inference after the fact, failing
   silent in exactly the same direction the mechanism was meant to close.

`break-it-check`'s "no code path implements this `Catches:`" row already
detects the same condition — code a phase depended on has moved or
vanished — terminally and loudly, at gate time, at no authoring cost and with
no separate field to keep in sync.
