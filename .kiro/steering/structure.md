---
inclusion: always
---

# Project Structure & Power Anatomy

## This repository produces a Kiro Power
The deliverable is a Knowledge Base Power (no MCP server). Layout:

    POWER.md              # Frontmatter + overview + index of steering skills
    steering/             # One markdown file per on-demand "skill"
      <skill>.md

## POWER.md frontmatter
Use these fields: name, displayName, description, keywords, author. (paad-power
also uses version and iconURL, which are acceptable.) Do not invent fields like
tags, repository, or license.

## Steering skills (each a manual steering file)
Skills are loaded on demand via `#skill-name`. Keep each skill focused and
self-describing. POWER.md lists them with a one-line description each, matching
paad-power's style.

## Conventions
- Keep POWER.md lean; push detailed workflows into steering/ files.
- Prefer a single power with multiple steering files over splitting into
  multiple powers (per Kiro's power-builder guidance).
- Write for an agent audience: concrete steps, decision tables, and checklists
  over prose.
- Any generated artifacts (analysis reports, roadmap docs, plans) must have
  defined output paths so runs are reproducible.

## Reference model
The installed paad-power (POWER.md + steering/ with one file per skill) is the
structural template to follow.
