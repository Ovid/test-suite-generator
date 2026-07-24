# test-roadmap

**Lock down what your code does today, so you know exactly what breaks when you change it tomorrow.**

`test-roadmap` is an [Agent Skill](https://agentskills.io/specification) that looks at any
repository — any language, tests or none — and builds a test suite that catches real
regressions. It plans the suite in phases, then writes those tests one phase at a time, across
as many sessions as it takes. One command: `/test-roadmap`, on day 1 and on day 90.

**Warning**: Alpha code. This repo is a testing ground and, if successful, will
likely be merged into [PAAD](https://github.com/Ovid/paad). [Read this post
for more background](https://curtispoe.org/articles/watching-claude-sonnet-outperform-opus).

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
- **It runs your suite the way you will — and won't call a phase done while it's noisy.** Before
  a phase lands, test-roadmap runs your whole suite on your branch, once normally and once under
  coverage — coverage runs shake out problems a plain run hides. Failing tests, stray warnings
  like `Wide character in print`, anything that would make you distrust the run: it fixes what's
  its own to fix and surfaces the rest, and it never edits your code to quiet things down.
- **It never touches your real code.** That bug injection happens in a throwaway git worktree,
  discarded the moment it's done. Your working tree, your database, and your config files are
  never at risk.
- **Pick it up anywhere.** Resumable across unrelated commits, squash merges, fresh clones, and
  sessions that remember nothing about the last one.
- **You finish with a bug list you didn't start with.** While pinning your code's behavior,
  test-roadmap spots where the code contradicts itself — a function that ignores its own
  docstring, a check that lets through what it claims to reject — and logs each one, with the
  exact test that proves it. Only concrete, actionable bugs make the list; vague hunches don't.
  It never fixes them — that's your call — but the day you start, you're working from a real
  to-do list, not a blank page.

## Using it

The whole workflow is one command, run again each session — there's nothing else to
remember:

```
/test-roadmap
```

It runs on a working branch, never your main line. Start it on `main` (or `master`, `trunk`,
whatever yours is called) and it stops and offers to make a branch first, so your primary
branch never fills up with half-built tests.

- **The first run**, when there's no roadmap yet, reads your repo, grades any tests you
  already have, and drafts a phased plan. Along the way it asks you to settle the few
  choices that are genuinely yours — test framework, whether to start from unit or
  end-to-end, what order the phases go in. Nothing gets built without your say-so.
- **Every run after that** is the *same* command. It reads the roadmap, finds the next
  phase you haven't built, writes its tests, and proves them — by breaking your code in
  a throwaway copy and confirming the test screams — before marking the phase done. It
  works across unrelated commits, fresh clones, and sessions that remember nothing about
  the last one.

The roadmap file it writes (`paad/test-roadmap/test-roadmap.md`) is the memory; `/test-roadmap` is
the only verb. You never have to recall a second command or which step you were on.

## Status

**Alpha.** The skill is built and installable — it's the Markdown files under
[`test-roadmap/`](test-roadmap/) — but it's early, unpackaged, and not yet published to a
skills registry, so expect rough edges and install it by hand for now.

The design behind it is hardened through multiple rounds of adversarial review, with a
decision log that records not just what was chosen but what was rejected and why:

- **The design:** [`docs/superpowers/specs/2026-07-22-test-suite-roadmap-design.md`](docs/superpowers/specs/2026-07-22-test-suite-roadmap-design.md)

## License

MIT © Curtis "Ovid" Poe
