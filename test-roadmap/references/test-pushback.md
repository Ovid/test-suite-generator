# test-pushback — critique mode + approach-menu format

This file holds two independent, referenceable pieces. Both mode files
(`build-test-roadmap.md`, `execute-test-roadmap.md`) point at them by name
rather than inlining them, so the format and the gate stay defined once:

- **`§ Presenting approaches`** — the menu format used whenever the skill
  puts a genuine fork in front of the developer.
- **mode `critique-plan`** — the adversarial pass Stage 4 runs against the
  skill's *own* draft plan before Stage 5 writes it.

Neither section depends on the other, and neither depends on surrounding
prose in this file — either can be pointed at on its own.

---

## § Presenting approaches

Use this format only for **approaches** — genuine forks where the answer is
taste or is expensive to reverse (test framework/runner, suite strategy,
phase ordering, whether to rewrite weak tests grading found). Do not use it
for **findings** — verdicts derived from evidence, such as a weak-test
classification or a mock classification. A finding is a fact, not a choice;
asking a human to vote on a fact is the wrong tool. Findings are fixed
inline or recorded to the ledger, not put on a menu.

**Assume the developer does not know this repository.** The person answering
may have opened it for the first time this session — legacy and unfamiliar
code is the case the skill exists for. So every menu must be answerable with
no prior knowledge of the codebase: state each option in plain terms, and
make the recommendation strong enough to follow blind, justified by what
*this repo actually is* — its detected stack, its existing tests, its
structure — not by generic preference. A developer who knows nothing about
the code should be able to take the recommended option and be right.

Every menu, presented unbatched, contains:

- **Each option**, with **pros and cons** stated for it — in terms a
  newcomer to this codebase can weigh, not insider shorthand.
- A **recommendation**, with the **reason** grounded in what was detected in
  *this* repo — not a bare pick and not a generic default.
- A final **deep dive** option: **dispatch a subagent to run an adversarial
  pushback against the presented options** — challenging the menu itself and
  surfacing any better option it missed, before committing to any one on it.

**One decision at a time, unbatched.** A menu asks for exactly one decision.
Do not bundle a second question onto it ("...and while we're at it, do you
also want X?") — it splits the developer's attention and degrades the
answer to both. Resolve the decision in front of them, act on it, and only
then raise the next one. A follow-up that is fully determined by the answer
just given is not a second decision — it does not need its own menu, just do
it.

If an external `pushback` skill is available in the environment, offer it
as an addition to the adversarial-review option above — never as a
replacement for it and never as a requirement to proceed. This skill is
self-contained by design (Inviolate #6): the menu format works with or
without that skill present, on any agent.

---

## mode `critique-plan`

An adversarial pass against the plan Stage 3 just drafted — the skill
critiquing its own work product before committing to it. Findings from this
pass are **fixed inline, in the draft, on the spot**. No separate report is
written; there is nothing here to hand off, only phases to rewrite or drop
before Stage 5 writes them to disk.

This is not the approach-menu format above — nothing in this mode asks the
developer to choose anything. It is the plan's author, one pass later,
reading the draft adversarially.

### The governing rule

> **Every phase must name the bug it would catch.** If a phase cannot
> answer *"what breakage makes these tests go red?"*, it is not a phase —
> it is coverage theater. Rewrite it or drop it.

This is a gate, not a suggestion. A phase that fails it does not proceed to
Stage 5 in its current form — it gets a `Catches:` line that names an
actual breakage, or it does not survive the draft. Walk every phase in the
draft and ask the question explicitly; do not let a phase through on the
strength of a plausible-sounding title. "Billing retry & dunning
integration tests" is not, on its own, an answer — *"a retry exhausting
without transitioning the account to dunning; a partial refund
double-crediting"* is (see *Phase format*'s worked example). If the answer
you write down is vague enough that it would survive being attached to any
phase in any codebase, it has not actually named a bug — push until it
names the specific breakage this phase's tests would turn red for.

### Supporting rules

**1. No success criterion may reduce to "the command exits 0."** A green
exit code and a covered line are never evidence a test is good — a test
that asserts nothing produces both, forever, including against code that is
obviously broken. This is verified tool behavior, not a hypothetical:

- `go test ./...` exits 0 against a package with no `_test.go` files.
- `jest --passWithNoTests` exits 0 with zero tests collected.
- A fully-skipped test file exits 0 in every runner examined.

Check every phase's stated success criterion against this. If a phase's
definition of done is satisfied by any of the three cases above — or by
anything with the same shape, a run that can complete having asserted
nothing — rewrite the criterion to name the specific assertion that would
have to hold, tied to the bug the phase names it would catch. Coverage
percentage is not evidence either, for the same reason: it is legitimate
only for finding code with no test at all, never for certifying that a
covered line is well-tested.

**2. Every test double and fixture is classified, and every `scaffold`
mock names the refactor that retires it.** Walk the ledger entries the
draft plan touches — new mocks and fixtures a phase would introduce, and
existing ones a phase's tests would exercise. Each must carry one of the
three classes: `boundary` (a genuine external edge — network, clock,
randomness, a third-party API), `scaffold` (exists only because the
surrounding code resists testing — test debt), or `data` (constructed test
data that is permanent and correct — a seeded account, a factory-built
order). An unclassified double is a gap in the draft, not a detail to leave
for later. A `scaffold` mock additionally names the refactor that will
retire it; that refactor is **deferred** — named now, actually done when
that refactor is actually planned, not during this pass. When in doubt
between `boundary` and `scaffold`, or when a double goes unexamined,
default to `scaffold`: the harm is asymmetric, since a `scaffold` silently
promoted to permanent hides test debt, while a `boundary` mistakenly
recorded as `scaffold` only leaves a note someone later deletes.

**3. Legacy phases characterize current behavior; they do not fix it.**
Where a phase's tests would pin behavior of existing code, check that
every assertion the phase proposes matches what the code *currently does*,
not what it *should* do. If the pass turns up behavior that looks wrong,
that is a note on the phase — recorded as a comment, not turned into a
different assertion or a code change. Characterizing a legacy system and
fixing its bugs are two different hard problems; this pass exists to
police the first, and to stop the second from sneaking in disguised as a
"more correct" assertion.

### After the pass

Every phase that survives can answer the governing rule's question in one
sentence, has every double and fixture it touches classified, and — where
it characterizes legacy code — pins current behavior only. Phases that
can't be fixed to clear this bar are dropped from the draft, not carried
forward with a note to "revisit."

If a `pushback` skill is available in the environment, running it in
addition to this pass is strictly additive and should be offered — it is
not required. This skill remains self-contained (Inviolate #6): the gate
above is complete on its own, on any agent, whether or not that skill is
present.
