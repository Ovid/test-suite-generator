# test-roadmap

**Lock down what your code does today, so you know exactly what breaks when you change it tomorrow.**

`test-roadmap` is an [Agent Skill](https://agentskills.io/specification) that looks at any
repository — any language, tests or none — and builds a test suite that catches real
regressions. It plans the suite in phases, then writes those tests one phase at a time, across
as many sessions as it takes. One command: `/test-roadmap`, on day 1 and on day 90.

## Why

High coverage numbers lie. A line can be "covered" by a test that asserts nothing — green
forever, catching nothing. So when you finally refactor the scary part of a legacy codebase,
the suite stays quiet and the regression ships anyway.

test-roadmap builds the suite that *doesn't* stay quiet. It pins your code's **current**
behavior — even the buggy parts, on purpose — so that the day you start fixing things, the
tests break loudly and tell you exactly what you changed.

## What makes it different

- **Every phase names the bug it would catch.** If a phase can't answer *"what breakage makes
  these tests go red?"*, it's coverage theater — and it gets rewritten or dropped. Nothing is
  added just to move a number.
- **It proves each test actually works.** Before a phase counts as done, test-roadmap injects
  the very bug the phase claims to catch and confirms the test goes red. A passing command and
  a covered line are never accepted as proof that a test is any good.
- **It never touches your real code.** That bug injection happens in a throwaway git worktree,
  discarded the moment it's done. Your working tree, your database, and your config files are
  never at risk.
- **Pick it up anywhere.** Resumable across unrelated commits, squash merges, fresh clones, and
  sessions that remember nothing about the last one.

## Status

This is a **design repository**. The skill isn't built yet — what's here is the design,
hardened through four rounds of adversarial review, with a decision log that records not just
what was chosen but what was rejected and why.

- **The design:** [`docs/superpowers/specs/2026-07-22-test-suite-roadmap-design.md`](docs/superpowers/specs/2026-07-22-test-suite-roadmap-design.md)

## License

MIT © Curtis "Ovid" Poe
