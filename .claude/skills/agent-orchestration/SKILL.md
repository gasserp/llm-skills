---
name: agent-orchestration
description: Splits work across AI subagents or sessions when a task exceeds what one session can hold, with verifiable subtasks and review gates. Use when a task spans many independent parts, when asked to "parallelize", "fan out", "use subagents", or "split this up", when context limits force multiple sessions, or when deciding whether to orchestrate at all.
---

# Agent Orchestration

## Purpose

The task is too big for one focused session, and you are deciding how to split
it across subagents or sequential sessions. The standard: every subtask is
independently verifiable, every prompt carries the context its agent cannot
discover cheaply, and the orchestrator — you — verifies the merged whole rather
than trusting the pieces' self-reports. Orchestration is overhead spent to buy
parallelism and context isolation; spend it only where it pays.

## Core workflow

1. **Decide whether to orchestrate at all.** Estimate the task honestly: can one
   focused session hold it — the files fit in context, the steps are sequential,
   no step must wait on external work? If yes, do NOT orchestrate. Every spawned
   agent starts from zero and must re-derive context you already have (repo
   layout, conventions, the problem statement), and every handoff loses
   information — the subtleties you know but didn't write down do not transfer.
   Orchestrate only when at least one holds: the work exceeds a session's
   context, parts are genuinely independent and parallelism saves wall-clock
   time that matters, or a part needs isolation (a risky experiment in a
   worktree; a cheap model on a mechanical subtask).

   Exit criterion: a stated reason from that list, or the decision to stay in
   one session — recorded either way.

2. **Decompose along verification seams, not topic seams.** Split the work so
   that each subtask is independently checkable: its own files (disjoint from
   other subtasks' files, so merges don't conflict), its own tests or check
   command, and its own done-criterion you can evaluate without re-doing the
   work. The test of a good seam: you can write, in one sentence, how you will
   verify the subtask *before* assigning it — "done when `pytest tests/parser/`
   passes and `mypy src/parser/` is clean". If you cannot state the
   verification, it isn't a subtask yet — decompose further or keep it
   yourself. Why: verifiable subtasks are what let you trust a cheaper/smaller
   model with the piece; the check catches what the model misses.

   Exit criterion: a written list of subtasks, each with owner-files, a check
   command, and a done-criterion.

3. **Map the dependency structure.** Mark which subtasks are independent (can
   run in parallel) and which depend on another's output (must run in sequence).
   Interfaces between dependent subtasks — function signatures, file formats,
   schemas — must be fixed by YOU before spawning, not negotiated between
   agents; two agents inventing the two sides of one interface will invent
   incompatible ones.

   Exit criterion: a two-column plan — parallel batch(es) and sequential
   chain(s) — with shared interfaces pinned in writing.

4. **Write each subagent prompt as a self-contained brief.** The agent knows
   nothing you don't tell it and will guess at anything vague — and it guesses
   with confidence. Every prompt carries:
   - *Context it can't discover cheaply:* the goal of the overall task, the
     relevant repo conventions, decisions already made, the pinned interfaces
     from step 3, absolute paths to the files that matter.
   - *The exact deliverable:* files to produce/modify, behavior to implement,
     with an example of the expected shape where one exists.
   - *Hard constraints:* the only paths it may touch, things it must NOT do
     (no new dependencies, no schema changes, no committing), and its check
     command from step 2.
   - *Required report format:* what to state on completion — files changed,
     check-command output, deviations from the brief, open questions. A
     structured report is what makes step 6's spot-check possible.
   Rule of thumb: if the prompt is under ~10 lines, the agent will spend its
   first third of the session rediscovering what you already knew.

   Exit criterion: each prompt could be executed by a competent stranger with
   no access to you.

5. **Run parallel batches; gate sequential stages with review.** Launch
   independent subtasks concurrently. Between dependent stages, insert a review
   gate: before stage N+1 starts, review stage N's output against its
   done-criterion using `reviewing-code` — run its check command yourself, read
   the diff. Why gates: errors compound downstream — a wrong interface in stage
   one becomes three agents' wasted work by stage three; catching it at the
   gate costs minutes, catching it at integration costs the whole pipeline.

   Exit criterion: no dependent stage started before its predecessor passed its
   gate.

6. **Integrate and verify the merged whole yourself.** Collect the pieces,
   merge, and then verify the *combination*: build the whole, run the full test
   suite, and exercise at least one end-to-end path that crosses subtask
   boundaries — boundary crossings are where independently-correct pieces
   disagree. Spot-check each subagent's self-report against its artifacts: open
   two or three of the files it claims to have changed, re-run its check
   command yourself. Agents report success in the same confident tone whether
   or not they succeeded. Apply `verifying-changes` to the merged result. The
   orchestrator owns the final standard; subagents own pieces — responsibility
   does not delegate.

   Exit criterion: whole-system checks pass, one cross-boundary path exercised,
   at least a sample of self-reports verified against artifacts.

7. **Spend review, not regeneration.** When a subtask's output is flawed, the
   cheap fix is one strong, specific review pass — concrete findings sent back
   to the same agent (its context is warm) — not respawning for another blind
   attempt. Three weak generation passes cost more than one strong review pass
   and converge slower, because regeneration without feedback re-rolls the same
   dice.

## Decision points

- **Task fits one session:** stay in one session. This is the default; the
  burden of proof is on orchestrating.
- **Subtasks are mechanical and verification is airtight** (rename across 200
  files, apply a lint autofix, port a fixed pattern): assign a cheap/fast
  model; the check command carries the quality.
- **Subtask requires judgment or design** (choosing an approach, an API shape):
  keep it in the orchestrator session or assign the strongest model with a
  review gate behind it — design errors are the ones that compound.
- **Two subtasks turn out to need the same file:** merge them into one subtask
  or serialize them; parallel edits to one file guarantee a conflict resolved
  by whoever merges last, i.e. by luck.
- **A subagent reports a blocker or asks a scope question:** answer it
  yourself with a course-correction message; do not let it improvise scope —
  improvised scope is how two subtasks silently overlap.
- **An agent's output fails its gate twice on the same criterion:** stop
  regenerating. Either the brief is wrong (fix the prompt — usually missing
  context) or the seam is wrong (re-decompose); a third identical attempt is
  the definition of the weak-generation trap.
- **Long-running risky experiment alongside stable work:** isolate it in a
  worktree so a failed experiment can be discarded without touching the main
  tree.

## Quality bar

- [ ] Orchestration decision has a stated reason; one-session default was
      considered.
- [ ] Every subtask has disjoint owner-files, a check command, and a
      done-criterion written before assignment.
- [ ] Shared interfaces pinned by the orchestrator before any spawn.
- [ ] Every prompt self-contained: context, deliverable, hard constraints,
      report format.
- [ ] No dependent stage started before its predecessor's review gate passed.
- [ ] Merged whole built, full tests run, one cross-boundary path exercised by
      the orchestrator personally.
- [ ] Self-reports spot-checked against actual artifacts.

## Common traps

- **Fan-out for its own sake.** Five agents on a task one session could hold
  produces five context re-derivations, five report-reading sessions, and one
  integration headache — negative net work. Correction: step 1's burden of
  proof; parallelism must buy something you can name.
- **Vague prompts that make the agent guess scope.** "Improve the error
  handling in the API layer" yields whatever the agent imagines that means,
  confidently. Correction: exact files, exact deliverable, exact check — step
  4's brief format, every time.
- **Trusting self-reports.** "All tests pass and the feature is complete" is
  what agents say; it is weak evidence of what happened. Correction: re-run the
  check commands yourself; open the files; treat reports as claims to verify,
  not results.
- **Skipping gates to save time.** The time saved is borrowed against
  integration, at compound interest — stage-one errors multiply through every
  dependent stage. Correction: gates are the schedule, not overhead on it.
- **Letting agents define shared interfaces.** Two agents will produce two
  incompatible halves of the same seam. Correction: the orchestrator pins
  signatures, schemas, and formats before spawning.
- **Regenerating instead of reviewing.** Re-rolling a failed subtask without
  feedback repeats the failure with variations. Correction: one specific review
  pass to the same warm agent beats three cold retries.
- **Declaring done when the last subagent reports done.** The pieces passing
  their gates is not the whole passing. Correction: run `verifying-changes` on
  the merged result before claiming completion.

## Escalation

- After decomposition, more than ~20% of the work is "integration and glue"
  you can't assign to any subtask: the seams are wrong — stop and re-decompose
  rather than spawning into a bad structure.
- A subtask cannot be given a check command because the codebase has no test
  or build infrastructure for that area: fix the infrastructure first (see
  `writing-tests`) or keep that piece in your own session; unverifiable
  delegation is hope, not orchestration.
- Two rounds of re-decomposition still leave subtasks entangled (shared files,
  circular dependencies): the task may be inherently serial — do it in
  sequential sessions with written handoff notes instead of parallel agents.
- Budget or time consumed exceeds twice the single-session estimate with the
  merge not yet verified: pause, report status honestly to the human, and ask
  whether to continue, simplify scope, or take over manually.
