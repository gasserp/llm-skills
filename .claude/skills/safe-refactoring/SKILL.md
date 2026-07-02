---
name: safe-refactoring
description: Restructures code without changing behavior, using characterization tests, small mechanical steps, and a commit at every green state. Use when asked to refactor, clean up, restructure, extract, rename, deduplicate, or modernize code, or when a feature change requires reshaping code first.
---

# Safe Refactoring

## Purpose

You are changing the structure of code while proving its behavior did not
change. The standard: a reviewer can verify "no behavior change" by inspection
of each commit, every intermediate state is green, and any breakage bisects to
one small step. Refactoring that might have changed behavior is not refactoring
— it is an untested rewrite.

## Core workflow

1. **Separate refactor from behavior change — always, at the commit level.** If
   the task includes both (it usually does), split it before starting: which
   commits will be pure structure, which will be behavior. A mixed commit makes
   both halves unreviewable — the reviewer can't verify equivalence of the
   refactor *or* see the behavior change clearly, so they must re-derive both
   from scratch.
   *Exit criterion:* a written plan listing each commit and labeling it
   `refactor` or `behavior`.

2. **Measure the safety net.** Run the existing tests for the code you'll touch
   and check what they actually pin down: run coverage
   (`pytest --cov=<module>`, `go test -cover`, `npx jest --coverage`) or, if
   coverage tooling is unavailable, deliberately break the code
   (invert a condition) and confirm a test fails. A suite that stays green when
   the code is wrong protects nothing.
   *Exit criterion:* you know which behaviors of the target code are pinned by
   tests and which are not.

3. **Write characterization tests where coverage is thin.** For each unpinned
   behavior you're about to move: call the code with representative inputs,
   record what it *currently* returns or does, and assert exactly that —
   including current bugs and oddities. You are pinning "what it does", not
   "what it should do"; fixing a bug you find is a behavior change and goes in
   a separate, later commit, because a refactor that also fixes bugs can no
   longer be verified by equivalence. Commit the characterization tests first,
   as their own `test:` commit.
   *Exit criterion:* every code path you will move fails a test if its output
   changes.

4. **Refactor in small mechanical steps.** Use named, reversible moves — rename,
   extract function/variable, inline, move to another file, replace conditional
   with polymorphism, change signature with all call sites — one move (or one
   tight cluster of the same move) at a time. Prefer IDE/language-server refactor
   commands and codemods (`gofmt -r`, `comby`, `ast-grep`, IDE rename) over hand
   edits, because mechanical tools don't get tired mid-file. Big-bang rewrites
   lose the thread of equivalence: once everything is different at once, neither
   you nor the reviewer can say why it should behave the same.
   *Exit criterion per step:* code compiles, full relevant test suite passes.

5. **Commit at every green state.** After each passing step:
   `git add -A && git commit -m "refactor: <mechanical move>"`. Small green
   commits mean `git bisect` pins any later breakage to one move, and `git
   revert` of one step never entangles others. If a step goes red and the fix
   isn't obvious within a few minutes, `git checkout .` back to green and take a
   smaller step — debugging a broken intermediate state is how refactors turn
   into rewrites.
   *Exit criterion:* `git log` reads as a sequence of single mechanical moves,
   each green.

6. **Keep a rollback path.** Work on a branch; never force-push over the
   pre-refactor state; if the refactor changes anything observable at runtime
   (serialization, ordering, timing, wire formats), keep the old path deletable
   in one revert. Before declaring done, run the code for real, not just its
   tests — see `verifying-changes` — because tests only pin what someone thought
   to pin.
   *Exit criterion:* you can state the one-command rollback (`git revert <range>`
   or branch deletion) and you have run the real flow once.

## Decision points

- **Refactor-then-change vs change-then-refactor?** Refactor first when the
  current structure makes the change awkward ("make the change easy, then make
  the easy change") — this is the default. Change first when the fix is urgent
  (production bug) and the refactor would delay it: ship the minimal fix, then
  refactor. Never interleave them in one commit either way.
- **Is a rewrite justified over incremental refactoring?** Only when *all* of
  these hold: (a) the current behavior is not worth preserving or is precisely
  specified elsewhere (spec, protocol, golden files); (b) incremental steps are
  impossible because the core abstraction is wrong everywhere at once; (c) the
  old and new implementations can run side by side behind a flag or be compared
  on the same inputs (shadow traffic, golden-file diff) before cutover. If you
  can't compare old and new on real inputs, you don't have a rewrite plan, you
  have a hope.
- **Found a bug while characterizing?** Pin it as-is with a test named
  `characterization: <odd behavior>` plus a code comment or issue link. Fix it
  after the refactor lands, as a `fix:` commit that flips that one test.
- **Refactor touches a public API or serialized format?** That is a behavior
  change to consumers regardless of your intent. Treat it as one: deprecation
  path, versioning, or explicit sign-off — not a silent rename.
- **Tests are so entangled with implementation that every step breaks them?**
  The tests assert internals, not contract. First rewrite the worst offenders
  against observable behavior (see `writing-tests`), commit that, then refactor.

## Quality bar

- [ ] Every commit is purely `refactor` or purely `behavior`/`fix`/`test`,
      stated in its message.
- [ ] Characterization tests existed (or coverage was verified adequate) before
      the first structural change.
- [ ] Each commit is one mechanical move and was green when committed.
- [ ] Full test suite green at the end, with zero test assertions weakened or
      deleted to get there (a weakened assertion is a hidden behavior change).
- [ ] Diff of observable behavior is empty: same outputs, same errors, same
      public API, same serialized formats.
- [ ] Rollback path stated in the PR description.
- [ ] Real flow exercised once post-refactor per `verifying-changes`.

## Common traps

- **"While I'm here" scope creep.** You came to extract a function and also
  renamed twelve variables, reordered imports, and fixed a typo in a string the
  UI displays. Each addition grows the surface a reviewer must verify as
  behavior-neutral. Correction: keep a `LATER.md` or issue list for everything
  you notice; touch only what the planned moves require.
- **Refactoring code you have no reason to touch.** Restructuring for taste, in
  code with no upcoming change and no defect history, spends review time and
  bisect noise for zero delivered value — and every refactor carries nonzero
  regression risk. Correction: refactor in service of a concrete next change,
  a recurring bug source, or a measured comprehension cost; otherwise leave it.
- **Fixing bugs mid-refactor.** The diff now changes behavior, so "no behavior
  change" is false and the reviewer must audit everything. Correction: pin the
  bug in a characterization test, finish the refactor, fix it in its own commit.
- **Trusting a green suite that pins nothing.** Tests passed before and after,
  but they never asserted the behavior you moved. Correction: step 2's
  break-the-code check — if inverting a condition doesn't fail a test, the
  suite's green light is decoration.
- **Marathon red states.** "It'll compile again once I finish moving
  everything" — three hours later you're debugging a state no one can reason
  about. Correction: if you've been red for more than ~15 minutes, revert to
  the last green commit and re-plan smaller steps.

## Escalation

Stop and ask a human when:

- The rewrite criteria in Decision points arguably apply — a rewrite is an
  architecture decision (`architecture-decisions`), not a solo judgment call.
- Characterizing reveals behavior that looks load-bearing *and* wrong (callers
  may depend on the bug) — changing or preserving it is a product decision.
- The refactor cannot avoid touching a public contract, wire format, or stored
  data, and no deprecation policy exists.
- Required characterization would need infrastructure you can't run (prod-only
  dependencies, licensed services) — proceeding without a net must be an
  explicit, human-approved risk.
