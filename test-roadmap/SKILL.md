---
name: test-roadmap
description: >
  Analyzes a repository and any existing test suite, grades existing tests for
  weakness, classifies mocks, emits a phased roadmap for building a test suite
  that catches real regressions, then executes those phases one at a time. Use
  when planning or building a test suite, assessing whether existing tests are
  worth anything, adding tests to a legacy codebase, or when the user mentions
  test coverage, test strategy, or weak tests.
compatibility: Requires git
---

# test-roadmap

This file is the router. It stays dumb on purpose: one check, one branch,
nothing else. All the substance — grading, planning, writing tests, bug
injection — lives in `references/` and loads only once routing has picked a
mode.

## Route

```
docs/test-roadmap.md exists?  →  load references/execute-test-roadmap.md
                       absent  →  load references/build-test-roadmap.md
```

That is the entire routing logic — one file existence check, two branches.
`docs/test-roadmap.md` is the roadmap this skill itself writes at the end of
build mode, so its presence is exactly the signal that a previous run already
did Detect/Grade/Plan/Critique/Write and there is a phased plan to execute
against. Its absence means this is either the first run against this repo, or
a run after that file was deleted — either way, build it.

If it ever grows a third condition, that is a signal something has been put
in the wrong place — take it back to `build-test-roadmap.md` or
`execute-test-roadmap.md`, not to this file.

This router never loads `references/break-it-check.md`,
`references/test-pushback.md`, or `references/test-theater.md` directly.
Those three are loaded by whichever of the two mode files needs them, at the
point in their own protocol that needs them — not from here.

## Before routing: confirm you're in a repo

`compatibility: Requires git` above means this skill needs a working git
checkout — build mode fans out grading subagents against the tree as it
stands, and execute mode's `break-it-check` gate runs bug injection in a
disposable `git worktree`. If the current directory isn't inside a git repo,
say so and stop before loading either mode file.

## Entering build mode

Before build mode can plan anything, it needs to know what it's planning
for. The first thing it does — Stage 1, Detect — is identify the stack from
manifests and config rather than assumption (`package.json`,
`pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, and so on: examples, not
a closed table), then determine how tests are invoked and what test files
already exist. The skill must never hardcode a language; where it needs a
per-ecosystem fact, it looks for the signal in the repo instead of consulting
a built-in list.

The full five-stage protocol — Detect, Grade, Plan, Critique, Write — lives
in `references/build-test-roadmap.md`. Load it now if
`docs/test-roadmap.md` is absent.

## Entering execute mode

Load `references/execute-test-roadmap.md` now if `docs/test-roadmap.md`
exists. It reads that file's `## Decisions` section once, selects the next
phase per the completion protocol, and gets on with writing tests — it does
not re-detect or re-ask anything build mode already settled.
