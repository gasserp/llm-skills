# Skill Library

This library encodes senior engineering judgment as executable procedure. It exists
so that engineers early in their careers — and smaller, cheaper AI models — can
debug, extend, validate, and advance projects at the standard of a distinguished
engineer without one in the room.

Every skill is a decision procedure, not an essay. If a skill tells you to "be
careful" without telling you what to check, it is broken — file an issue or fix it
via the `skill-authoring` skill.

## Library map

**Understand & diagnose**

| Skill | Reach for it when |
|---|---|
| `codebase-onboarding` | You are about to work in a repo you don't know |
| `systematic-debugging` | Something is broken and the cause is not obvious |
| `performance-tuning` | Something is slow, or you're asked to make it faster |
| `incident-response` | Production is down or degraded *right now* |

**Change & validate**

| Skill | Reach for it when |
|---|---|
| `extending-code` | Adding a feature or capability to an existing codebase |
| `safe-refactoring` | Restructuring code without changing behavior |
| `writing-tests` | Deciding what to test, at what level, and how |
| `verifying-changes` | Before you claim any change is done |

**Judge & advance**

| Skill | Reach for it when |
|---|---|
| `reviewing-code` | Reviewing a diff — yours or someone else's |
| `architecture-decisions` | A choice will be expensive to reverse |
| `agent-orchestration` | Work is too big for one session; splitting across agents |
| `skill-authoring` | Adding or improving a skill in this library |

## Conventions (binding for every skill)

Structure: one directory per skill, `.claude/skills/<name>/SKILL.md`, flat — no
nesting. Extra depth goes in `<name>/references/*.md`, linked from the SKILL.md,
loaded only when needed.

Frontmatter: `name` (kebab-case, must match the directory) and `description`.
The description is the retrieval key: third person, starts with what the skill
does, then "Use when ..." with the concrete trigger situations and words a user
or model would actually say. No marketing language.

Body structure, in this order:

1. **Purpose** — one short paragraph: the situation, and the standard this skill
   holds you to.
2. **Core workflow** — numbered steps. Each step says what to do, how to do it
   (commands, artifacts), and its exit criterion — how you know the step is
   done. Write it as a plain `Exit criterion:` line (no italics) so the format
   is uniform across skills.
3. **Decision points** — explicit branching criteria ("if X, do A; if Y, do B").
   A junior reader must never have to guess which branch applies.
4. **Quality bar** — a checklist that defines "done at senior standard". These are
   verifiable statements, not aspirations.
5. **Common traps** — the specific failure modes of this activity, each with its
   correction. These carry the most tacit knowledge; do not pad them.
6. **Escalation** — the conditions under which you stop and ask a human, stated
   as triggers, not vibes.

Style rules:

- Imperative voice. Concrete commands and examples over abstractions.
- Every heuristic includes its *why* in one sentence, so it transfers to
  situations the author didn't foresee.
- Banned phrases: "be careful", "use best practices", "as appropriate",
  "consider whether" without the criteria to consider *by*.
- Length: 100–250 lines per SKILL.md. If you need more, move depth to
  `references/` — the main file must stay loadable and skimmable.
- Cross-reference related skills by name so readers can chain them.

Audience calibration: assume the reader is competent with tools (can run
commands, read code, write tests) but does not yet have judgment (does not know
when to stop, what to distrust, or what "good" looks like). Skills supply the
judgment; they must not waste lines teaching tool basics.
