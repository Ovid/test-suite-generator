# test-roadmap Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Author the `test-roadmap` Agent Skill — six markdown files that plan and execute a regression-catching test suite for any repo — exactly as specified in the design.

**Architecture:** The deliverable is prose, not code: a router (`SKILL.md`) plus five on-demand reference files. There is no build system, no dependency manifest, no runtime. "Implementation" means authoring each file from its governing spec sections so the assembled skill obeys the design's inviolate decisions. The single load-bearing *executable* mechanism — the disposable-worktree bug-injection lifecycle — is verified against real `git` before the file that documents it is written.

**Tech Stack:** Markdown (Agent Skill format per https://agentskills.io/specification). `git` worktree commands referenced by `break-it-check`. No language runtime, no test framework, no dependencies.

## Global Constraints

Copied verbatim from the spec; every task's acceptance implicitly includes these.

- **Source of truth:** `docs/superpowers/specs/2026-07-22-test-suite-roadmap-design.md`. Author each file *from* its named spec sections; do not invent behavior the spec does not state.
- **Per-file budget:** each file stays **under 500 lines**.
- **Never hardcode a language.** Where a per-ecosystem fact is needed, detect the signal in the repo — never consult a built-in table.
- **Deliverable path:** the skill lives at repo root as `test-roadmap/` (matches the spec's *Deliverable shape*).
- **Roadmap output path is load-bearing and not configurable:** `docs/test-roadmap.md` (its existence is the router's signal). Full grading sink is `docs/test-suite-analysis.md`.
- **The six Inviolate decisions** (CLAUDE.md) hold across every file: (1) the suite pins current behavior, even wrong behavior; (2) production code is never permanently modified — injection happens in a throwaway worktree; (3) exit status and coverage % are never evidence a test is good; (4) no signal auto-marks anything done; (5) every phase names the bug it would catch; (6) the skill is self-contained (may recommend other skills, never require them).
- **Progressive disclosure:** references load on demand. The router loads exactly one mode file; mode files load the leaves they need. A resume run's context cost = router + `execute-test-roadmap.md` + `break-it-check.md` only.

---

## File Structure

```
test-roadmap/
├── SKILL.md                       # Task 7 — router + frontmatter + Stage 1 detection
└── references/
    ├── build-test-roadmap.md      # Task 5 — the five stages; owns phase format + ## Decisions authoring
    ├── execute-test-roadmap.md    # Task 6 — completion protocol, writes tests, latches Landed:
    ├── break-it-check.md          # Task 4 — mandatory per-phase gate (git worktree lifecycle)
    ├── test-pushback.md           # Task 2 — adversarial critique + approach-menu format
    └── test-theater.md            # Task 1 — weak-test catalog
```

**Build order is dependency order** (leaves first, router last): theater → pushback → *(verify git)* → break-it-check → build → execute → SKILL → integration check. Each task produces one self-contained file (or one verification) a reviewer can accept or reject on its own.

**Cross-file ownership (locked here to prevent drift):**
- The **phase format** (`Catches:`/`Produces:`/`Branch:`/`Landed:`) and the `## Decisions` section are *authored* by `build-test-roadmap.md` (Stage 5 writes them) and *consumed* by `execute-test-roadmap.md`. The two files MUST use identical field names.
- The **approach-menu format** is defined once in `test-pushback.md § Presenting approaches`; both mode files point at it — they never restate it.
- `break-it-check.md` is consumed only by `execute-test-roadmap.md`. The router never loads it directly.

---

## Task 1: `references/test-theater.md` — weak-test catalog

**Files:**
- Create: `test-roadmap/references/test-theater.md`

**Interfaces:**
- Consumes: nothing (leaf).
- Produces: a named catalog of weak-test patterns and a per-item verdict shape that Stage 2 subagents cite. Grading subagents return, per weak test: `file`, `line`, `pattern name`, one-sentence *why it fails to catch regressions*, suggested replacement. This file defines the pattern-name vocabulary those verdicts draw from.

**Governing spec sections:** *Purpose* (load-bearing claims — `foo is not Null` isn't a real test), Stage 2 (*Grade*), the *test-double & fixture ledger* (`boundary`/`scaffold`/`data`).

- [ ] **Step 1: Author the catalog.** Write `test-roadmap/references/test-theater.md` cataloging the weak-test patterns a grading subagent must recognize. At minimum, name and describe: assertion-free tests (`assert result is not None`, snapshot-only with no meaningful invariant); tautological assertions (asserting a mock returns what it was told to return); tests that assert on exit status or coverage rather than behavior; over-mocked tests where the mock *is* the behavior under test; happy-path-only tests that never exercise the boundary the phase names. For each pattern: a one-line detector cue and a one-line "what a real replacement asserts instead." Keep it a reference catalog, not a procedure.

- [ ] **Step 2: Acceptance check (read the file against this list).** Confirm all hold:
  - Every pattern names *what breakage it would let through* (ties to Inviolate #5).
  - The file states explicitly that a passing command and a covered line are **never** evidence a test is good (Inviolate #3).
  - No language is hardcoded — patterns are described behaviorally, with examples optionally drawn from multiple ecosystems (Global Constraint).
  - File is under 500 lines.

- [ ] **Step 3: Commit.**
```bash
git add test-roadmap/references/test-theater.md
git commit -m "test-roadmap: add test-theater weak-test catalog"
```

---

## Task 2: `references/test-pushback.md` — critique mode + approach-menu format

**Files:**
- Create: `test-roadmap/references/test-pushback.md`

**Interfaces:**
- Consumes: nothing (leaf).
- Produces: two named, referenceable sections that both mode files point at:
  1. **`§ Presenting approaches`** — the approach-menu format (each option with pros/cons; a recommendation + reason; an option to dispatch a subagent for adversarial pushback against the presented options).
  2. **mode `critique-plan`** — the adversarial pass Stage 4 invokes against the draft plan.

**Governing spec sections:** Stage 4 (*Critique*), *Approaches vs findings*, CLAUDE.md *Presenting options*.

- [ ] **Step 1: Author `§ Presenting approaches`.** Write the menu format verbatim to the design's *Presenting options* contract: every menu contains each option with **pros and cons**; a **recommendation** with its **reason**; and a final option to **dispatch a subagent for an adversarial pushback review against the presented options**. State the "one decision at a time, unbatched" rule. This section is the single definition both mode files reference — write it to be pointed at, not inlined.

- [ ] **Step 2: Author mode `critique-plan`.** Write the adversarial critique of the skill's *own* draft plan. The governing rule, stated prominently: **every phase must name the bug it would catch** — a phase that cannot answer *"what breakage makes these tests go red?"* is coverage theater; rewrite it or drop it. Supporting rules: (1) no success criterion may reduce to "the command exits 0" — cite the verified exit-0 failure modes (`go test ./...` with no `_test.go`; `jest --passWithNoTests`; a fully-skipped file); exit status and coverage % are not evidence; (2) every test double and fixture is classified, and every `scaffold` mock names the refactor that retires it (deferred per the ledger); (3) legacy phases characterize current behavior, they do not fix it. State that findings are fixed **inline**; no report is written. Note that an external `pushback` skill, if present, is strictly-additive and offered — never required (Inviolate #6).

- [ ] **Step 3: Acceptance check.** Confirm:
  - `§ Presenting approaches` is a standalone section a one-line pointer can target (no dependence on surrounding prose).
  - The "name the bug it would catch" rule is present and phrased as a gate, not a suggestion.
  - The exit-0 examples are the verified ones from the spec (not invented).
  - Self-containment: external skills are recommended, never required.
  - File is under 500 lines.

- [ ] **Step 4: Commit.**
```bash
git add test-roadmap/references/test-pushback.md
git commit -m "test-roadmap: add test-pushback critique + approach-menu format"
```

---

## Task 3: Verify the `break-it-check` git-worktree lifecycle against real git

**Files:**
- Create (throwaway, not committed): a scratch repo under the session scratchpad.

**Interfaces:**
- Consumes: nothing.
- Produces: confirmation (or correction) of the exact `git` commands `break-it-check.md` will document. This de-risks the one load-bearing *executable* mechanism before Task 4 writes prose around it. The spec's *Verification notes* reproduced most of this on another machine; this task re-confirms it on the target machine's git 2.50.1 and closes the one claim the spec left unreproduced (worktree checkout excludes gitignored artifacts).

**Governing spec sections:** *break-it-check § Protocol*, *Why a disposable worktree*, *Verification notes*.

- [ ] **Step 1: Build a scratch repo exercising the full lifecycle.** In the session scratchpad, script (do not commit into the project) a repo that reproduces every git behavior the gate depends on:
```bash
set -euo pipefail
SB="$SCRATCH/wt-verify"; rm -rf "$SB"; mkdir -p "$SB"; cd "$SB"
git init -q && git config user.email t@t && git config user.name t
mkdir src && printf 'fn lt(a:i32,b:i32)->bool{a<b}\n' > src/lib.rs
printf 'target/\n' > .gitignore && mkdir -p target && echo built > target/artifact
git add -A && git commit -qm init
# (1) add a worktree at HEAD under the fixed .git-internal prefix
git worktree add -q .git/test-roadmap-worktrees/phase1 HEAD
# (2) checkout must NOT contain gitignored artifacts  [the unreproduced claim]
test ! -e .git/test-roadmap-worktrees/phase1/target/artifact && echo "OK: gitignored excluded"
# (3) copy a test in, then mutate a tracked file -> worktree is unclean by construction
echo 'test' >> .git/test-roadmap-worktrees/phase1/newtest.rs
sed -i '' 's/a<b/a<=b/' .git/test-roadmap-worktrees/phase1/src/lib.rs
# (4) bare remove refuses the unclean worktree
git worktree remove .git/test-roadmap-worktrees/phase1 2>&1 | grep -q 'use --force' && echo "OK: bare remove refused"
# (5) --force removes it
git worktree remove --force .git/test-roadmap-worktrees/phase1 && echo "OK: --force removed"
# (6) prune reclaims a record whose checkout dir was deleted out from under git
git worktree add -q .git/test-roadmap-worktrees/phase2 HEAD
rm -rf .git/test-roadmap-worktrees/phase2
git worktree prune && ! git worktree list --porcelain | grep -q phase2 && echo "OK: prune reclaimed strand"
```
- [ ] **Step 2: Run it and record the outputs.**
Run the script. Expected: all six `OK:` lines print, exit 0.
If any line fails (git version drift), note the exact behavior — Task 4 must document what *this* git does, not what the spec assumed.

- [ ] **Step 3: No commit.** This is scratch verification; nothing enters the project tree. Record the results in the Task 4 authoring notes.

---

## Task 4: `references/break-it-check.md` — the mandatory gate

**Files:**
- Create: `test-roadmap/references/break-it-check.md`

**Interfaces:**
- Consumes: the git behaviors confirmed in Task 3; the phase's `Catches:` line (defined in Task 5) as its behavior input.
- Produces: the gate protocol `execute-test-roadmap.md` (Task 6) invokes at step 4. Discharges *"the test suite should fail loudly when you fix bugs."*

**Governing spec sections:** the entire `## break-it-check` section — *Protocol* (6 steps), *Constrained mutation*, *Failure table*, *Why a disposable worktree*.

- [ ] **Step 1: Author the protocol (6 steps, in order).** Transcribe the spec's protocol precisely, because order is load-bearing:
  1. **Sweep, then create one worktree per phase.** `git worktree prune`, then enumerate `git worktree list --porcelain`, **filter to paths under `.git/test-roadmap-worktrees/` and force-remove only those** (the filter is the whole guarantee a developer's hand-made worktree is never touched). Then `git worktree add .git/test-roadmap-worktrees/<phase> HEAD`. State that the worktree is reused across the baseline run and every mutation in the phase (amortizes the undiscoverable dependency-install cost).
  2. **Copy the phase's new test files into the worktree, *then* mutate.** State why order is load-bearing (colocated-test ecosystems: the test file *is* the production file; mutate-first silently overwrites the mutation). Cite Rust `#[cfg(test)] mod tests` as the reproduced case.
  3. **Baseline run** — phase's own tests, unmutated, must pass; confirm stable across two runs.
  4. **Per behavior on `Catches:`, inject one mutation via a blind mutator subagent** — subagent gets the `Catches:` text, the production file, the worktree path, the operator list, but **not the test**; returns one hunk; main agent applies the **first** hunk, runs the phase's own tests, checks for red.
  5. **`git worktree remove --force`** — `--force` required (worktree unclean by construction), confirmed in Task 3.
  6. **Commit only after the check passes**, on the developer's real tree; echo operator name(s) to transcript and onto the `Landed:` line.

- [ ] **Step 2: Author *Constrained mutation*.** The named operator set (flip a comparison; off-by-one a boundary; negate a condition; alter a constant; drop a state transition), the subagent declaring the operator **by name**, and the **explicit bans**: no early `return`/`raise`/`throw` at function entry, no whole-function-body removal (sledgehammers that make *any* test go red, letting theater pass). State the re-roll rule: if the first hunk leaves the test green, that is the *theater* row — rewrite the test, do **not** re-roll the mutation.

- [ ] **Step 3: Author the *Failure table*.** All five rows verbatim in meaning: baseline-red-but-green-in-full-suite → order dependence, surface, don't latch; baseline-red-and-red-in-full-suite → mischaracterization, rewrite test; baseline unstable → flaky, surface; no code path implements the behavior → phase invalidated, re-plan; mutation applied but test green → theater, rewrite test, don't re-roll. State that "run the test" means the **phase's own tests, not the whole suite.**

- [ ] **Step 4: Author *Why a disposable worktree*.** The sweep-on-entry is the **guarantee**; on-exit `remove` (step 5) is only the fast path a crash can skip. State: developer's tree is never touched, so Inviolate #2 holds regardless; the leak closed is worktree strands, closed on next entry not current exit.

- [ ] **Step 5: Acceptance check.** Confirm:
  - The gate is stated as **mandatory** — no phase latches without it.
  - The force-remove is **filter-to-prefix**, never a blanket "remove all worktrees."
  - Every git command matches what Task 3 confirmed on this machine (correct any drift).
  - The blind-mutator never sees the test; the **first** hunk is used; no re-roll.
  - Injection is described as happening only in the disposable worktree (Inviolate #2).
  - File is under 500 lines.

- [ ] **Step 6: Commit.**
```bash
git add test-roadmap/references/break-it-check.md
git commit -m "test-roadmap: add break-it-check gate"
```

---

## Task 5: `references/build-test-roadmap.md` — the five stages

**Files:**
- Create: `test-roadmap/references/build-test-roadmap.md`

**Interfaces:**
- Consumes: `test-theater.md` (Stage 2 grading), `test-pushback.md § Presenting approaches` (approach menus) and mode `critique-plan` (Stage 4).
- Produces: the **canonical phase format** and the **`## Decisions` section format**, both consumed by Task 6. Emits `docs/test-roadmap.md` and `docs/test-suite-analysis.md`.

**Governing spec sections:** *Build mode: the five stages* (Stages 1–5), *Phase format*, *Approaches vs findings*, *Test-double & fixture ledger*, *Test-data strategy*, *Suite-health preconditions*.

- [ ] **Step 1: Author Stage 1 — Detect (main agent).** Identify the stack from manifests (list the examples: `package.json`, `pyproject.toml`/`setup.cfg`, `go.mod`, `Cargo.toml`, `Gemfile`, `pom.xml`/`build.gradle`, `composer.json`, `cpanfile`/`Makefile.PL`, `*.csproj`, `mix.exs`). Detect three things from repo signals only: (1) how tests are invoked + what test files exist; (2) whether a coverage tool is available (read-only, Stage 3 gap-finding, never a quality signal; fall back to agent judgment and say so if none); (3) whether a test-data mechanism exists (fixtures/factories/seeds/testcontainers/`conftest.py`/`factories/`/`fixtures/`). State: never hardcode a language; Stage 1's output is a discovery, not a safety boundary (no path fence anywhere).

- [ ] **Step 2: Author Stage 2 — Grade (fan-out subagents).** **Skipped entirely when no tests exist** (greenfield → Stage 3). Dispatch subagents partitioned by tier/directory, each reading its partition against `test-theater.md`, returning a compact structured verdict — **never pasted test source, never diffs**. Per weak test: file, line, pattern name, one-sentence why, suggested replacement; per mock/fixture: a ledger classification. State the cardinality containment: full list → `docs/test-suite-analysis.md`; main context receives **counts + top-K by severity only**.

- [ ] **Step 3: Author Stage 3 — Plan (main agent).** Identify untested behaviors worth testing; group into phases across the three tiers (unit/integration/e2e); build the ledger; draft phases. Two Stage 1 inputs: coverage for gap-finding only (never quality); test-data mechanism — a phase needing data against a repo with no mechanism gets a **surfaced note** (*"requires a test-data mechanism this repo lacks"*), not a silent plan. Legacy phases pin **current** behavior; suspected-wrong behavior is a note, not a fix (Inviolate #1).

- [ ] **Step 4: Author Stage 4 — Critique.** Invoke `test-pushback.md` mode `critique-plan` against the draft plan; findings fixed inline, no report. Restate the governing rule: every phase names the bug it would catch (Inviolate #5).

- [ ] **Step 5: Author Stage 5 — Write, plus the phase format and `## Decisions`.** Two artifacts: `docs/test-roadmap.md` (the `## Decisions` section, then the phases) and `docs/test-suite-analysis.md` (full verdicts + ledger — the sink for anything unbounded). State that `docs/test-roadmap.md` is always the target, never an existing `docs/roadmap.md`, and its path is load-bearing (the router's signal). **Define the canonical phase format here** (this is the definition Task 6 consumes):
```markdown
## Phase 3: Billing retry & dunning integration tests

Catches: a retry exhausting without transitioning the account to dunning;
         a partial refund double-crediting.
Produces: tests/integration/billing/
Branch:   billing-integration-tests
Landed:
```
  with the field semantics: `Catches:` is the anti-theater gate + `break-it-check`'s behavior input (names behavior, not paths); `Produces:` is documentation only, **not** a completion signal; `Branch:` is a human breadcrumb, not a machine signal; `Landed:` is empty until the gate passes, then `YYYY-MM-DD <sha> (<operator>)`, human-clearable, never cleared by the skill. Author the **ledger** (`boundary`/`scaffold`/`data`) and the **approaches vs findings / recorded defaults** rules: ~4–6 approaches per run go to unbatched menus (via `test-pushback § Presenting approaches`); test organization is a **recorded default** (mirror source tree / convention), not a menu item; findings go to the ledger; `scaffold` mocks are **batched** (retiring refactor named when that refactor is planned); **silence defaults to `scaffold`** (asymmetric harm). Record decisions in `## Decisions` so execute mode never re-asks.

- [ ] **Step 6: Acceptance check.** Confirm:
  - Stage 2 is skipped for greenfield, and its full output never lands in main context (counts + top-K only).
  - Coverage is gap-finding only, never a quality gate (Inviolate #3).
  - The phase format matches the spec exactly, field-for-field (Task 6 depends on this).
  - `docs/test-roadmap.md` path is stated as load-bearing and non-configurable.
  - The approach menu is referenced from `test-pushback`, not restated here.
  - No language hardcoded; the manifest list is examples, not a closed table.
  - File is under 500 lines.

- [ ] **Step 7: Commit.**
```bash
git add test-roadmap/references/build-test-roadmap.md
git commit -m "test-roadmap: add build mode (five stages)"
```

---

## Task 6: `references/execute-test-roadmap.md` — completion protocol

**Files:**
- Create: `test-roadmap/references/execute-test-roadmap.md`

**Interfaces:**
- Consumes: the phase format + `## Decisions` format (Task 5); `break-it-check.md` (Task 4); `test-pushback.md § Presenting approaches` (Task 2). Field names MUST match Task 5 exactly (`Catches:`/`Produces:`/`Branch:`/`Landed:`).
- Produces: the per-resume execution path — the path every run after the first takes.

**Governing spec sections:** *Execute mode*, *Why TDD is not bundled*, *Completion model* (Problem / Governing principle / Protocol / Why-not sections).

- [ ] **Step 1: Author the execute-mode loop.** Five steps: (1) read `## Decisions` from the roadmap, never re-ask anything recorded there; (2) select the candidate phase per the *Completion model* protocol; (3) write the tests for that phase; (4) run `break-it-check.md` — **mandatory, no phase latches without it**; (5) write `Landed: YYYY-MM-DD <sha> (<operator>)` and commit. State: execute mode writes test code, never production code, and needs no path fence because injection is in a throwaway worktree.

- [ ] **Step 2: Author *Why TDD is not bundled*.** State that `superpowers:test-driven-development` is deliberately not loaded: for a characterization phase the code already works, so the test passes on first run by construction; TDD's *"Test passes? → Fix test"* rule (verified at `test-driven-development/SKILL.md:126`) is destructive here. The red phase characterization needs is **mutation** (`break-it-check`), not authorship.

- [ ] **Step 3: Author the *Completion model*.** State the governing principle: **no signal auto-marks anything done, and no absence is ever proof of not-done** (Inviolate #4). The protocol, `Landed:` as sole signal, candidate phase only:
  - Step 1 (main agent): `Landed:` populated? → done, stop, no git, no question.
  - Step 2 (main agent): `Landed:` empty → surface the phase block (`Catches:`/`Produces:`/`Branch:`) and ask the human *"did this land, or should I execute it?"* On "execute," run the phase; on "it landed," write the `Landed:` line from what the human reports. Absence of a "yes" means **execute**, never silent skip.
  State that `Landed:` is human-clearable but the skill never clears it (so no agent inference can un-land work). Include the *why-not* rejections briefly: not branch-merge detection (4 of 5 cases fail toward "not done" — cite the squash/rebase/deleted-branch/fresh-clone table), not commit-log scanning, not `Produces:`-absence, not an anchor commit / `Covers:` field (the gate's "no code path implements this `Catches:`" row already detects staleness terminally).

- [ ] **Step 4: Acceptance check.** Confirm:
  - Phase-format field names are **identical** to Task 5 (`Catches:`/`Produces:`/`Branch:`/`Landed:`) — no `clearLayers`/`clearFullLayers` drift.
  - The `break-it-check` gate is stated as mandatory (no latch without it).
  - Execute mode never writes production code and never clears `Landed:`.
  - The completion protocol asks *with the phase block as evidence*, and empty ⇒ execute (safe direction), never silent skip (Inviolate #4).
  - TDD-not-bundled cites the verified line reference.
  - File is under 500 lines.

- [ ] **Step 5: Commit.**
```bash
git add test-roadmap/references/execute-test-roadmap.md
git commit -m "test-roadmap: add execute mode + completion model"
```

---

## Task 7: `SKILL.md` — router + frontmatter + Stage 1 detection entry

**Files:**
- Create: `test-roadmap/SKILL.md`

**Interfaces:**
- Consumes: `build-test-roadmap.md` (Task 5), `execute-test-roadmap.md` (Task 6) — routes to exactly one.
- Produces: the skill entry point. Stays dumb: router + detection only.

**Governing spec sections:** *Deliverable shape* (frontmatter), *The router*, Stage 1 (*Detect*).

- [ ] **Step 1: Confirm the frontmatter schema against agentskills.io.** Before writing, fetch `https://agentskills.io/specification` and confirm the required SKILL.md frontmatter fields (at minimum `name`, `description`). This prevents shipping a skill that fails to load. If the spec's asserted `compatibility: Requires git` field is not part of the schema, keep git-requirement info in the description/body instead and note the correction.

- [ ] **Step 2: Author the frontmatter.** Use the spec's values verbatim (adjusting only if Step 1 found a schema mismatch):
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

- [ ] **Step 3: Author the router + Stage 1 entry.** The entire routing logic is two conditions:
```
docs/test-roadmap.md exists?  →  load references/execute-test-roadmap.md
                       absent  →  load references/build-test-roadmap.md
```
  State that the router never loads `break-it-check.md`, `test-pushback.md`, or `test-theater.md` directly (the two mode files do), and that a third routing condition is a signal something is in the wrong place. Include the minimal Stage 1 detection framing needed for the main agent to enter build mode correctly (detect stack/tests before routing decisions), keeping detail in `build-test-roadmap.md`.

- [ ] **Step 4: Acceptance check.** Confirm:
  - Frontmatter validates against the agentskills.io schema confirmed in Step 1.
  - The router is exactly two branches keyed on `docs/test-roadmap.md` existence — nothing more.
  - SKILL.md loads neither leaf file directly.
  - File is under 500 lines (it should be well under).

- [ ] **Step 5: Commit.**
```bash
git add test-roadmap/SKILL.md
git commit -m "test-roadmap: add SKILL.md router"
```

---

## Task 8: Whole-skill integration check

**Files:**
- Modify (only if the check finds defects): any of the six files.

**Interfaces:**
- Consumes: all six files.
- Produces: a consistent, spec-conformant skill.

- [ ] **Step 1: Cross-reference resolution.** Confirm every file path a file points at actually exists at that path, and every `§`-section pointer (e.g. `test-pushback § Presenting approaches`) resolves to a real heading. Fix any dangling reference.

- [ ] **Step 2: Phase-format consistency.** Grep the four field names across `build-test-roadmap.md` and `execute-test-roadmap.md`; confirm they are spelled and cased identically in both, and match the phase-format block. Fix drift.

- [ ] **Step 3: Inviolate-decisions pass.** Re-read the six files against the six Inviolate decisions (Global Constraints). Confirm none is contradicted — in particular that no file describes mutating the developer's tree, no file treats exit-status/coverage as evidence, and no file requires another skill.

- [ ] **Step 4: Budget + language check.** Confirm each file is under 500 lines (`wc -l test-roadmap/SKILL.md test-roadmap/references/*.md`) and that no file hardcodes a language table where a repo signal is specified instead.

- [ ] **Step 5: Spec-coverage sweep.** Walk each section of the design doc; confirm each maps to content in some file. List and close any gap.

- [ ] **Step 6: Commit any fixes.**
```bash
git add test-roadmap/
git commit -m "test-roadmap: integration fixes (cross-refs, field consistency)"
```

---

## Self-Review (plan author, against the spec)

- **Spec coverage:** Purpose/acceptance criterion → Task 4 (break-it-check enforces it) + Task 5 Stage 4. Non-goals → constrain each task; no task adds a diff-review/bug-fix/optimizer path. Deliverable shape + frontmatter → Task 7. Router → Task 7. Five stages → Task 5. Execute mode → Task 6. break-it-check (protocol/mutation/failure table/worktree rationale) → Tasks 3–4. Approaches vs findings, ledger, test-data strategy, suite-health preconditions → Task 5. Phase format → Task 5 (owner) + Task 6 (consumer). Completion model + all why-nots → Task 6. Accepted limitations → inherent (documented in-file, no code). Verification notes → Task 3 re-confirms the executable ones. All spec sections mapped.
- **Placeholder scan:** No "TBD"/"handle edge cases"/"write tests for the above." Content steps name the exact sections and claims to author; the spec is the transcription source (DRY — the plan maps and locks acceptance, it does not re-transcribe 500 lines of settled prose into itself).
- **Type/name consistency:** The four phase-format field names (`Catches:`/`Produces:`/`Branch:`/`Landed:`) are defined once (Task 5), consumed once (Task 6), and cross-checked (Task 8 Step 2). File paths are identical everywhere. `§ Presenting approaches` named identically in Tasks 2, 5, 6.
- **Adaptation note:** TDD red-green steps are intentionally replaced by spec-conformance acceptance checks for the five prose files, per CLAUDE.md's override and the design's own anti-theater ethos; the single executable mechanism (git worktree lifecycle) keeps a real runnable check (Task 3).
