---
inclusion: always
---

# Product: test-suite-generator (Kiro Power)

## What this project is
A Kiro Power that helps developers build (or strengthen) a test suite for an
existing codebase, in any programming language. It analyzes the repository, then
guides the developer through a phased plan to build a regression safety net:
unit tests, integration tests, and a small set of critical-path end-to-end tests.

## Who it's for
Developers working on legacy or greenfield codebases who need a trustworthy
safety net before refactoring or adding features with AI assistance.

## The problem it solves
Refactoring without a safety net is miserable and dangerous. AI accelerates code
changes, which accelerates technical debt; a high-quality test suite is the brake
pedal. But AI readily writes weak tests that pass without actually testing
anything, giving false confidence. This power exists to prevent that.

## Guiding principles (north star)
These are stable commitments. Implementation mechanics may evolve, but these do not.

- Lock down current behavior, don't fix bugs. The suite's job is to catch
  regressions, not to make the code correct. Building it will surface bugs;
  track them, don't fix them mid-build.
- Three layers, kept independently runnable: unit, integration, and a few
  "must never fail" critical-path e2e tests. Mixing them yields a noisy signal.
- Coverage is a floor, not the goal. "Exercised" is not "tested." A handful of
  e2e tests can report high coverage without meaningfully testing anything.
- Weak tests are the enemy. `assert foo is not null` is usually not a real test.
  Guard against tautological assertions, missing assertions, and tests that pass
  even when the code is broken.
- Distinguish observed behavior from intended behavior. Characterization tests
  capture what the code does today (bugs included). They must be marked so that
  when a bug is later fixed, the failing test is understood as expected, then
  promoted to intended.
- Incremental and context-aware. Plan the whole suite, but execute one phase per
  fresh session to protect the context window and avoid slop.
- Technology-agnostic. The power provides methodology and decision frameworks
  and relies on the agent's knowledge of each ecosystem's tooling.

## What this project is NOT
- Not a bug-fixing tool.
- Not a coverage-number chaser.
- Not a one-shot "generate all the tests" button.

## Open design questions (not yet decided — do not silently lock these in)
- Coverage floor vs mutation score as the primary quality gate.
- The exact observed-vs-intended marking mechanism (naming convention vs sidecar
  registry vs framework tags) and the observed-to-intended promotion lifecycle.
- Whether to hard-require paad's #pushback or degrade gracefully without it.
- Standalone roadmap document vs Kiro-native Spec system.
