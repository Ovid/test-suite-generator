# Build mode — the five stages

Loaded when `docs/test-roadmap.md` does not exist yet — the first run against a
repo, or any run after that file was deleted. Five stages, run in order, each
handing a concrete artifact to the next: Detect, Grade, Plan, Critique, Write.

## Stage 1 — Detect (main agent)

Identify the stack from manifests and config rather than assumption —
`package.json`, `pyproject.toml`/`setup.cfg`, `go.mod`, `Cargo.toml`, `Gemfile`,
`pom.xml`/`build.gradle`, `composer.json`, `cpanfile`/`Makefile.PL`, `*.csproj`,
`mix.exs`, and so on. This list is examples, not a closed table — the skill must
never hardcode a language. Where it needs a per-ecosystem fact, it looks for the
signal in the repo rather than consulting a built-in list.

Detect **three** things, all from repo signals:

1. **How tests are invoked, and what test files exist.** Read the manifest's test
   script/target and enumerate the test files it would run.
2. **Whether a coverage tool is available.** Used read-only, later, in Stage 3 for
   gap-finding — **never as a quality signal**. Where none is detectable, Stage 3
   falls back to agent judgment reading source, and says so in the plan rather
   than silently proceeding as if coverage output existed.
3. **Whether a test-data mechanism exists** — fixtures, factories, seed scripts,
   testcontainers, a `conftest.py`, a `factories/` or `fixtures/` directory. This
   feeds Stage 3: an integration or e2e phase that needs constructed data cannot
   be planned honestly against a repo with no way to construct it.

**Stage 1's output is a discovery, not a boundary.** It tells you what tests exist
and how to run them; it does not exhaustively partition the repo into "test" and
"production," and nothing downstream may treat it as a safety fence. There is no
path fence anywhere in this skill — bug injection happens in a disposable
worktree (`references/break-it-check.md`), so no path-based fence is needed to
keep mutation off production code.

## Stage 2 — Grade (fan-out subagents)

**Skipped entirely when no tests exist.** Greenfield goes straight to Stage 3 —
there is nothing to grade.

Where tests exist, dispatch subagents partitioned by test tier or directory.
Each subagent reads its partition against `references/test-theater.md` and
returns a compact **structured verdict** — never pasted test source, never
diffs. A legacy suite is thousands of lines the main agent must not absorb.

Each subagent returns:

- **Per weak test found:** file, line, pattern name (verbatim from
  `test-theater.md`), a one-sentence statement of why it fails to catch
  regressions, and a suggested replacement.
- **Per mock or fixture encountered:** a ledger classification (see *The
  test-double & fixture ledger*, below).

**Cardinality containment.** The number of instances is unbounded by design — a
legacy suite may yield hundreds of weak tests — and the design contains that
rather than merely asserting it away:

- The **full verdict list** is written to `docs/test-suite-analysis.md`, never
  returned to main context.
- **Main context receives counts and the top-K by severity only.** A 400-item
  grading result must never land in the resume context budget. Bounding each
  verdict's size while leaving the list itself unbounded in main context is the
  same defect wearing a different hat — both the per-item size and the list
  length must be bounded before anything reaches the main agent.

## Stage 3 — Plan (main agent)

Identify behaviors worth testing that are not tested. Group them into phases
across the three tiers — unit, integration, e2e. Build the ledger. Draft phases
in the canonical format defined in *Stage 5 — Write*, below.

Two Stage 1 findings feed this stage:

- **Coverage, for gap-finding only.** Where a coverage tool was detected, run it
  read-only to locate behaviors with *no test at all*. Coverage answers "what is
  untested," which it does well; it never answers "is this test any good," which
  it cannot — a line can be covered by a test that asserts nothing. This
  distinction is inviolate: coverage percentage is never evidence a test is
  worth keeping, in this stage or any other. Where no coverage tool exists, this
  is agent judgment reading source, and the plan states that plainly rather than
  presenting judgment as if it were tool output.
- **Test-data mechanism.** If a phase needs constructed data and Stage 1 found
  no mechanism to construct it, the phase carries a **surfaced note** — *"requires
  a test-data mechanism this repo lacks"* — rather than being planned as if it
  were executable as written. Finding un-runnability at plan time is the point
  of having a plan stage; finding it at the gate is the expensive place to find
  it.

**Legacy phases pin current behavior.** Where a phase characterizes existing
code, every assertion it proposes must match what the code *currently does*.
Suspected-wrong behavior is recorded as a note on the phase, never turned into a
different assertion or a code fix — characterizing a legacy system and fixing
its bugs are two hard problems, and this stage does the first one only.

## Stage 4 — Critique

Invoke `references/test-pushback.md` mode `critique-plan` against the draft
plan Stage 3 just produced. Findings are **fixed inline, in the draft, on the
spot** — no report is written; there is nothing to hand off, only phases to
rewrite or drop before Stage 5 writes them to disk.

The governing rule, restated because it is the gate the whole stage exists to
enforce:

> **Every phase must name the bug it would catch.** If a phase cannot answer
> *"what breakage makes these tests go red?"*, it is not a phase — it is
> coverage theater. Rewrite it or drop it.

See `test-pushback.md` for the full pass, including the supporting rules (exit
status and coverage are never evidence; every double and fixture is classified;
legacy phases characterize, they don't fix).

## Stage 5 — Write (main agent)

Two artifacts, written in this order:

- **`docs/test-roadmap.md`** — the `## Decisions` section first, then the
  phases.
- **`docs/test-suite-analysis.md`** — full grading verdicts and the ledger.
  This is the sink for anything unbounded (Stage 2's full list, the ledger in
  full); main context never carries it.

**`docs/test-roadmap.md` is always the target.** The skill never writes into an
existing `docs/roadmap.md` — that would risk phase-numbering collisions and
clobbering hand-written entries that live there for other reasons. This path is
also the router's signal for which mode to load on the next invocation, so it is
**load-bearing and not configurable** — do not rename it, alias it, or write
phases anywhere else.

### The canonical phase format

Every phase, in both build mode's draft and execute mode's updates, takes this
exact shape:

```markdown
## Phase 3: Billing retry & dunning integration tests

Catches: a retry exhausting without transitioning the account to dunning;
         a partial refund double-crediting.
Produces: tests/integration/billing/
Branch:   billing-integration-tests
Landed:
```

Field semantics:

- **`Catches:`** — the anti-theater gate from Stage 4, preserved in the artifact
  so a later reader can audit whether the phase was honest. It is also
  `break-it-check`'s behavior input: it names the *behavior* to mutate, not a
  path. A path is neither necessary (the behavior locates the code) nor
  sufficient (directory granularity cannot pick a line to mutate).
- **`Produces:`** — paths the phase creates, as **documentation only**. It is
  **not** a completion signal. A later reader uses it to find the phase's
  output; no control flow in this skill infers phase status from it.
- **`Branch:`** — a human-readable breadcrumb. **Not a machine signal.**
- **`Landed:`** — empty until `break-it-check` passes for this phase, then set
  to `YYYY-MM-DD <sha> (<operator>)`. **Human-clearable; the skill itself never
  clears it.** A cleared `Landed:` line is a human declaring the phase needs
  redoing — the skill has no mechanism that would ever produce that state on
  its own.

Each phase's definition-of-done includes a recommendation to run `agentic-review` on the branch before merging, if available. This is not bundled into execute mode: `agentic-review` requires a fresh session with no substantive history, so calling it from within the test-roadmap session would be inert.

### The test-double & fixture ledger

Every test double and fixture, existing or proposed, gets one classification:

| Class | Meaning |
|---|---|
| `boundary` | Permanent and correct. A real external edge: network, clock, randomness, a payment gateway, a third-party API, filesystem where relevant. |
| `scaffold` | Exists only because the surrounding code resists testing. Test debt. Names the refactor that retires it — that refactor is *deferred*, named now and actually done only when it is actually planned, not during this pass. |
| `data` | Constructed test data that is permanent and correct — a seeded account, a factory-built order, a fixture record. Neither an external edge (`boundary`) nor debt a refactor retires (`scaffold`). |

An unclassified double is a gap in the plan, not a detail to leave for later.
**When in doubt between `boundary` and `scaffold`, or when a double goes
unexamined, default to `scaffold`.** The harm is asymmetric: a `scaffold`
silently promoted to permanent hides test debt indefinitely, while a `boundary`
mistakenly recorded as `scaffold` only leaves a note a human later deletes.

### Approaches vs findings

Two different things reach the developer during a run, and they are not
handled the same way.

**Approaches** — genuine forks where the answer is taste or is expensive to
reverse. Roughly four to six across a whole run: test framework/runner, suite
strategy (unit-first / e2e-first / risk-first), phase ordering, whether to
rewrite the weak tests Stage 2 found. Every approach goes to an unbatched menu
in the format defined once in `references/test-pushback.md § Presenting
approaches` — that format is not restated here, so both mode files point at the
same definition rather than drifting apart.

**Not on the menu — recorded defaults.** Some things look like a choice but have
a dominant default and are cheap to reverse, so they are decided once and
recorded, not voted on. **Test organization** — mirror the source tree, or
follow ecosystem convention — is one of these: putting it on a menu would blow
the 4–6-approach ceiling and, worse, an operator-chosen "group by tier" layout
would put two phases in the same directory, manufacturing the exact
false-landed directory collision the completion model exists to avoid.

**Findings** — verdicts derived from evidence: a weak-test classification, a
mock classification. These are not approaches; asking a human to vote on a
classification is asking them to vote on a fact. Findings go to the ledger,
consistent with Stage 4's own rule that findings are fixed inline, not
reported.

**`scaffold` mocks are batched, not surfaced one-by-one.** They are recorded to
the ledger en bloc, not raised as individual decisions during grading — on a
legacy codebase, the case that produces the most `scaffold` entries, one-by-one
surfacing would mean dozens of serial decisions in a single sitting. The
retiring refactor for each is named when that refactor is actually planned,
which is both less overwhelming and a better moment to name it than during a
grading fan-out.

**Silence defaults to `scaffold`** — restated here because it governs what gets
written to the ledger during this stage, same reasoning as above: the harm of
under-classifying is asymmetric, so inattention is made safe by construction
rather than relying on a human reading every entry carefully.

### `## Decisions`

Every recorded default and every approach the developer picked is written to a
`## Decisions` section at the top of `docs/test-roadmap.md`, before the first
phase. Execute mode reads this section on every resume and **never re-asks**
anything recorded there — without it, menus would fire again on every run and
the resumability requirement fails through the front door. Record at minimum:
the chosen test framework/runner, the suite strategy, phase ordering, the
weak-test rewrite decision (if grading ran), and the test-organization default.
Keep each entry to the decision and its one-line reason — this section is a
record for a machine to skip past, not a design document.
