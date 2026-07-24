# Changelog

All notable changes to the **test-roadmap** skill are recorded here. The format
follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). The skill is
alpha and not yet versioned; until a first release, entries accumulate under
Unreleased.

## [Unreleased]

### Added

- **Phase progress readout.** Every phase the skill names is shown as "Phase X of
  Y", with "N done, M to go" on the forward-looking prompts, so you can always see
  how much of the plan is left rather than being told only about the current step.
- **Findings log** (`paad/test-roadmap/test-roadmap-findings.md`). While pinning
  your code's current behavior, the skill records the concrete bugs it notices —
  each paired with the test that pins it, so you know which test will go red when
  you fix it. Only verified, actionable findings are logged; vague hunches are
  dropped, and your production code is never changed.
- **Clean-run gate.** Before a phase is done, the skill runs your whole suite on
  your branch the way you would — once normally, once under coverage (coverage
  runs surface problems a plain run hides) — and won't finish a phase while it is
  failing or noisy. Everything it fixes is test-side; production code is never
  patched to quiet a warning.
- **Working-branch guard.** The skill refuses to build or execute on your primary
  branch (`main`, `master`, `trunk`, and the like) and offers to create a working
  branch first, so a half-built suite never lands on your main line.
- **Production-untouched check.** Before marking a phase done, the skill verifies
  from git that the phase changed only test files and its own docs — no existing
  code modified, deleted, or moved. If it finds any production change, it stops and
  shows you, rather than quietly latching. A build/config change (a test
  dependency, a coverage tool) is surfaced for your OK rather than assumed.

### Changed

- **Generated files moved from `docs/` to `paad/test-roadmap/`.** Keeps your
  `docs/` uncluttered and puts the skill's output where PAAD tooling expects it.
  **Breaking for a repo mid-build:** move an existing `docs/test-roadmap.md`,
  `docs/test-suite-analysis.md`, and `docs/test-roadmap-findings.md` into
  `paad/test-roadmap/`, or the next run will not find the roadmap and will rebuild
  the plan from scratch.
