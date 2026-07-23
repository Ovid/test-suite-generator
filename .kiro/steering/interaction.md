---
inclusion: always
---

# Interaction Rules

## Presenting questions and options (hard rule)
When presenting the user with a question or a set of options:

1. Present ONE question or decision at a time. Never batch multiple open
   decisions into a single message. Wait for the user's answer before moving on.
2. For each option, give concise pros and cons. Be honest about tradeoffs; do
   not steer by omission.
3. Always add one extra option: "Run a subagent to perform an adversarial
   pushback against these options." Choosing it dispatches a subagent to
   critique the option set itself — surfacing missing options, hidden
   assumptions, and flaws in the framing before the user commits.

This rule applies to every phase of work in this project: brainstorming, design,
planning, and implementation.

## Why
Batching decisions hides tradeoffs and invites rubber-stamping. One-at-a-time,
pros-and-cons, plus an adversarial escape hatch keeps the human genuinely in the
loop and catches bad framing early. This mirrors PAAD's pushback methodology: if
English is the new programming language, pushback is code review.
