---
inclusion: always
---

# Tech & Tooling

## Power type
Knowledge Base Power — pure documentation/methodology, no MCP server, no mcp.json.

## Format
- Markdown only. Frontmatter in POWER.md and in steering files.
- No build step and no runtime dependencies for the power itself.

## Technology-agnostic stance
The power must not hardcode one language's tooling. It provides decision
frameworks and relies on the agent to identify, per ecosystem:
- test runner (pytest, JUnit, Jest/Vitest, go test, RSpec, cargo test, ...)
- coverage tool
- mutation-testing tool where one exists (Stryker, PITest, mutmut, cosmic-ray, ...)
Always confirm what the target repo actually uses before assuming.

## Composition with other powers
This power complements PAAD. Where PAAD is installed, recommend running
#pushback on generated plans and #agentic-review on written tests. Degrade
gracefully when PAAD is not present.

## Testing the power itself
Install from this directory and exercise each skill against sample repositories
in multiple languages.
