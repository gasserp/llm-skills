# llm-skills

A knowledge-transfer skill library for [Claude Code](https://code.claude.com) and
compatible agents. It encodes the working methods of a senior engineer — how to
debug, extend, validate, review, and make decisions — as explicit decision
procedures, so that junior/mid-level engineers and smaller AI models can carry
projects forward at that standard without the senior engineer in the room.

## What's inside

Twelve skills under [`.claude/skills/`](.claude/skills/), in three clusters:

- **Understand & diagnose** — `codebase-onboarding`, `systematic-debugging`,
  `performance-tuning`, `incident-response`
- **Change & validate** — `extending-code`, `safe-refactoring`, `writing-tests`,
  `verifying-changes`
- **Judge & advance** — `reviewing-code`, `architecture-decisions`,
  `agent-orchestration`, `skill-authoring`

The library map, binding format conventions, and quality bar live in
[`.claude/skills/README.md`](.claude/skills/README.md).

## Design principles

- **Decision procedures, not essays.** Every skill is numbered steps with exit
  criteria, explicit branch conditions, a verifiable done-checklist, named
  failure modes with corrections, and escalation triggers.
- **Judgment, not tool basics.** The reader is assumed competent with tools but
  not yet calibrated on when to stop, what to distrust, or what "good" looks like.
- **Written for weaker executors.** A Sonnet-class model following a skill
  mechanically should land at a senior-quality outcome.
- **Self-advancing.** The `skill-authoring` skill defines how the library grows
  from real experience, and `agent-orchestration` defines how big work is split
  across cheap sessions with review gates — so the library improves itself over time.

## Using it

Clone this repo (or copy `.claude/skills/` into your project). Claude Code
discovers skills automatically; invoke one explicitly with `/<skill-name>` or let
the agent select it from the `description` triggers. Each skill is a single
`SKILL.md`, with optional deeper material under `references/`.

## Contributing

Follow `skill-authoring`. In short: skills are written from real scars, tested on
a fresh session before merging, and reviewed against the conventions in
`.claude/skills/README.md`.
