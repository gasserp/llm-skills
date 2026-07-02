---
name: skill-authoring
description: Adds, tests, and maintains skills in this library so they transfer senior judgment to juniors and smaller models. Use when asked to "write a skill", "add to the skill library", "capture this lesson", "document what we learned", when the same mistake or judgment call has recurred, or when updating or retiring an existing SKILL.md.
---

# Skill Authoring

## Purpose

You are adding to or maintaining this library. The standard: a skill is a
decision procedure a fresh session can execute mechanically to a senior-quality
outcome — not an essay about the topic. The test is 2am utility: when something
is on fire and nobody senior is awake, does this file tell the reader exactly
what to check, in what order, and when to stop? Write for 2am, not for the
conference talk.

## Core workflow

1. **Verify the skill earns its place.** A skill earns a directory when the
   same judgment call has been made — or botched — more than once, by you or by
   others. One incident is an anecdote; two is a pattern worth encoding. Write
   from real scars: the concrete task where the judgment was needed, what the
   naive approach did wrong, what the experienced approach checked. Speculative
   skills ("might be useful someday") encode guesses, and guesses read exactly
   like knowledge until someone follows one off a cliff. Also check the library
   map in `.claude/skills/README.md`: if an existing skill covers 70% of the
   territory, extend it (step 7) instead of fragmenting the library.

   Exit criterion: you can name at least two real occasions this skill would
   have changed the outcome, and no existing skill substantially covers it.

2. **Harvest with the extraction question.** After any completed task, ask:
   *"What did I know at the end that a checklist could have told me at the
   start?"* The answers — the check you wish you'd run first, the assumption
   that cost an hour, the file you didn't know to look at — are the skill's raw
   material. Collect them as concrete, past-tense facts ("the bug was in the
   caller, not the diff") before generalizing, because generalizing first
   discards the specificity that makes advice checkable.

   Exit criterion: a scratch list of specific lessons, each traceable to a real
   moment in a real task.

3. **Draft inside the binding conventions.** Read
   `.claude/skills/README.md` and follow it exactly —
   conventions are what make skills composable and loadable, and every
   deviation taxes every future reader:
   - File at `.claude/skills/<name>/SKILL.md`; `name` kebab-case, matching the
     directory.
   - Six body sections in order: Purpose, Core workflow (numbered steps, each
     with what/how/exit criterion), Decision points (explicit if-X-then-A
     branches), Quality bar (verifiable checklist), Common traps (failure mode
     + correction), Escalation (stop-and-ask triggers, not vibes).
   - Style: imperative voice; every heuristic carries its one-sentence *why*;
     banned phrases banned ("be careful", "use best practices", "as
     appropriate").
   - 100–250 lines. Overflow (long command references, worked examples, tool
     specifics) goes to `<name>/references/*.md`, linked from the SKILL.md and
     loaded only when needed — the main file must stay skimmable because it is
     loaded whole, every time.
   - Add the new skill to the README's library map table.

   Exit criterion: draft passes a section-by-section diff against the README's
   conventions, and `wc -l` is within 100–250.

4. **Engineer the description as a retrieval key.** The `description`
   frontmatter is how a model or user finds the skill — it is matched against
   what they *say*, not against what the skill is *about*. Write it in third
   person: one clause on what the skill does, then "Use when ..." listing the
   trigger situations and the literal words a requester would use ("review this
   PR", "should we use X or Y", "split this up"). Test it: write three requests
   a user would plausibly type; if the description shares no distinctive words
   with them, the skill will never load, and an unloadable skill is dead
   weight.

   Exit criterion: three plausible user phrasings each share trigger words with
   the description.

5. **Convert every instruction into a decision procedure.** Audit the draft
   line by line with one question: *could a reader act on this without me in
   the room?* "Be careful with concurrency" fails — careful *how*? Rewrite as
   the check: "for every read-then-write on shared state, name the lock or
   transaction that guards it; if you can't, it races." Each heuristic needs
   its why in one sentence, because a rule without its reason cannot be applied
   to the situation the author didn't foresee — and unforeseen situations are
   the only ones that matter. Numbers beat adjectives ("under 15 minutes", "at
   least three examples") because a Sonnet-class model follows thresholds
   mechanically but interprets adjectives optimistically.

   Exit criterion: no sentence survives that says what to want without saying
   what to check.

6. **Test on a fresh session with a real task.** Give the skill to a session
   that shares no context with you — a new agent or a colleague who wasn't in
   the incident — on a real task the skill claims to cover, and watch where it
   stumbles: the step it misreads, the branch it can't tell applies, the term
   it doesn't know, the exit criterion it can't evaluate. The stumbles ARE the
   edit list — not evidence the reader is weak. Why fresh: you cannot review
   your own skill for missing context, because your head silently supplies
   everything the file omits. Fix the stumbles and re-test if the fixes were
   structural.

   Exit criterion: one fresh-session run completed; every stumble either fixed
   in the skill or consciously accepted with a reason.

7. **Maintain: update beats appending, and retirement is an outcome.** When a
   new lesson arrives for an existing skill, fold it into the step or trap
   where a reader would need it — do not bolt a new section onto the end.
   Appended skills grow into contradictory scrolls, and length is a tax on
   every future load: a 400-line skill is skipped precisely when it's needed
   most. If folding pushes the file past 250 lines, demote the least-load-
   bearing detail to `references/`. And retire (delete, plus remove from the
   README map) any skill whose advice the tooling has absorbed — a lint rule,
   a CI gate, or a platform feature that now enforces the check automatically.
   A skill restating what a machine already enforces is pure load-tax.

   Exit criterion: after any update, the file is still within 100–250 lines,
   internally consistent read end-to-end, and the README map still accurate.

## Decision points

- **Lesson is one sentence** ("always run X before Y in this repo"): it's a
  CLAUDE.md / project-notes line, not a skill; a directory per sentence
  fragments the library.
- **Lesson is project-specific** (this repo's deploy quirk): project docs, not
  this library — library skills must transfer across codebases.
- **Two skills keep getting loaded together and cross-reference each other
  heavily:** merge them; one coherent procedure beats two half-procedures the
  reader must interleave.
- **A skill needs deep tool-specific detail** (flags, API sequences): main
  file carries the decision procedure; `references/<topic>.md` carries the
  detail, linked at the step that needs it.
- **You disagree with an existing skill's guidance:** test your correction on
  a real case first, then edit the skill with the why updated — do not add a
  contradicting note beside the old advice; two contradicting instructions
  read as zero instructions.
- **Asked to write a skill for something you haven't done:** decline or do the
  task first; step 1's scar requirement is not waivable, because plausible-
  sounding speculation is this library's most dangerous failure mode.

## Quality bar

- [ ] Backed by at least two real occurrences, named.
- [ ] Conforms to README conventions: frontmatter, six sections in order,
      style rules, 100–250 lines.
- [ ] Description written in third person with literal trigger words; three
      plausible user phrasings would match it.
- [ ] Every instruction is a decision procedure: a check, a threshold, or a
      branch — never an attitude.
- [ ] Every heuristic carries its one-sentence why.
- [ ] Every workflow step has an exit criterion a stranger can evaluate.
- [ ] Tested on a fresh session against a real task; stumbles folded back in.
- [ ] Library map in README updated (added, or removed on retirement).

## Common traps

- **Writing the conference talk instead of the 2am page.** Inspiring prose
  ("embrace simplicity", "think holistically") demos well and helps nobody at
  incident time. Correction: for each sentence ask "what does the reader *do*
  differently?"; if the answer is "feel motivated", delete it.
- **Encoding speculation in the voice of experience.** Guessed advice is
  indistinguishable from earned advice on the page, and readers extend it full
  trust. Correction: the two-scars rule from step 1; no scar, no skill.
- **Appending instead of folding.** Each bolted-on "Addendum" makes the skill
  longer and less coherent, until it's skipped entirely. Correction: new
  lessons go into the existing step or trap where the reader needs them; the
  line count must not creep monotonically.
- **Testing on yourself.** The author's head fills every gap in the text, so
  author-testing always passes. Correction: fresh session, zero shared
  context, real task — non-negotiable before calling a skill done.
- **Description written for the librarian, not the searcher.** "Encapsulates
  methodological guidance for evaluative processes" matches nothing anyone
  types. Correction: write the words a stressed user actually says ("my PR",
  "it's slow", "prod is down") into the description.
- **Hoarding retired skills "just in case".** Stale advice that tooling now
  contradicts actively misleads. Correction: delete on retirement; git history
  is the archive.

## Escalation

- Two fresh-session tests stumble on the same step after a rewrite: the
  procedure itself may be wrong, not the wording — take it back to whoever
  owns the underlying practice before a third wording pass.
- A proposed skill would change or contradict the binding conventions in
  README.md: the conventions govern all skills, so changing them is a library-
  wide decision — raise it with the library owner; do not fork the format
  unilaterally.
- Two existing skills give conflicting guidance for the same situation and you
  cannot determine which is right from real cases: flag both to the library
  owner rather than picking a winner by taste; readers are currently getting
  contradictory instructions.
- A skill's advice caused a bad outcome in the wild: treat it like a
  production bug — fix or retire the skill the same day, because every day it
  stands, another session may follow it.
