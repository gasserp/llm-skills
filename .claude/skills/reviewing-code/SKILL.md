---
name: reviewing-code
description: Reviews a diff to senior standard — correctness first, then design, then style — with feedback the author can learn from. Use when asked to "review this PR/diff/change", "look over my code", "give feedback on this branch", before approving any pull request, or when self-reviewing your own change before requesting review.
---

# Reviewing Code

## Purpose

You are reviewing a diff — someone else's or your own. The standard: approve only
code you would be comfortable being paged for at 3am. A review that catches a
naming nitpick but misses a logic bug is a failed review, whatever its comment
count. Order of concerns is binding: correctness first, then design, then style —
because a beautifully named function that corrupts data is worse than an ugly one
that works, and reviewer attention is finite.

## Core workflow

1. **Load context before reading a single changed line.** Read the PR/commit
   description, the linked issue, and answer: what is this change *supposed* to
   do, and how will you know it does it? Then open the changed files in full, not
   just the diff hunks — most bugs live in the interaction between the diff and
   the unchanged code around it (a new early return that skips cleanup written
   twenty lines below; a new caller that violates an invariant the function's
   other callers uphold). Why: a diff viewer shows you what changed; it hides
   what the change breaks.

   Exit criterion: you can state the change's intent in one sentence, and you
   have the surrounding code of every hunk on screen, not just the hunk.

2. **Pass one — what the code DOES (correctness).** Read the diff top to bottom
   and execute it in your head with real values: a typical input, an empty
   input, the largest plausible input, a malicious input. Trace each changed
   function with at least one concrete value end to end ("userId=42, list is
   empty, so line 30 returns... None, and the caller at line 80 does `.id` on
   it"). Do not evaluate design or style yet; a wrong-but-elegant diff must be
   caught as wrong first.

   Exit criterion: for each changed function, you have traced one concrete
   execution and either confirmed the result or written a blocking comment.

3. **Pass two — what the code FORGETS (omissions).** Bugs of omission are
   invisible in pass one because absent code has no lines to read. Walk this
   checklist explicitly against the diff:
   - *Error handling:* what happens when each new call fails — network error, DB
     conflict, file missing, timeout? Is failure swallowed, logged, retried, or
     propagated — and is that the same choice the surrounding code makes?
   - *Edge cases:* empty collection, null/None, zero, negative, unicode,
     duplicate entries, boundary of any `<` vs `<=`.
   - *Concurrency:* can two requests/threads run this at once? Check-then-act
     sequences (read, decide, write) race unless guarded; new shared mutable
     state needs a lock, a transaction, or an ownership argument.
   - *Resource cleanup:* every open/acquire/subscribe needs its close/release/
     unsubscribe on ALL paths — including the new error paths this diff adds.
   - *Security:* new inputs reaching queries (injection), templates (XSS), file
     paths (traversal), or shell calls; new endpoints missing the authz check
     their siblings have; secrets or PII in logs.
   - *Backwards compatibility:* changed function signatures, API responses,
     serialized formats, or DB schemas — who are the existing callers/consumers,
     and does old data still load? A new non-nullable column needs a migration
     story for existing rows, not just for new ones.

   Exit criterion: each of the six items has an explicit verdict — "not
   applicable", "handled at line N", or a comment filed.

4. **Check the tests test the change.** Read the new/modified tests and ask: if
   I reverted the production change, would these tests fail? A test that passes
   with the change reverted tests nothing — it usually asserts on mocks, on
   setup, or on values the change never touches. Also check coverage of the
   failure paths from step 3: a diff that adds error handling with no test
   forcing that error has untested error handling. If the diff has no tests,
   apply `writing-tests` criteria: behavior changes need tests; pure renames may
   not. Hand deeper "does this change actually work" questions to
   `verifying-changes`.

   Exit criterion: you can name the specific assertion that would fail if the
   core change were reverted — or you've filed a blocking comment that none
   exists.

5. **Check against the codebase's existing patterns.** Find the diff's siblings
   — the two or three existing things most like what it adds — and compare.
   Does the new route/handler/job/flag register everywhere its siblings do
   (router table, DI container, config schema, docs, metrics, feature-flag
   cleanup list)? Grep for one sibling's name and diff the set of files it
   appears in against the set this change touches; a missing file is usually a
   missing registration. Why: half-integrated code compiles, passes its own
   tests, and then silently never runs in production.

   Exit criterion: sibling comparison done; any registration gap filed as
   blocking.

6. **Now — and only now — design and style.** Design: wrong layer, duplicated
   logic that an existing helper covers, an abstraction that will not survive
   the next obvious requirement. Style: naming, formatting, comment quality —
   and only where a linter doesn't already own it. Keep proportion: if steps
   2–5 produced blocking findings, cap style comments at the few that matter,
   so the author's attention lands on the bugs.

7. **Write the feedback.** For every comment:
   - Mark it **blocking**, **non-blocking**, or **question** — the author must
     never have to guess what stands between them and merge.
   - For blocking items, state the concrete failure scenario: "with input X,
     this does Y" — e.g. "if `items` is empty, line 30 returns None and line 80
     throws AttributeError". An unfalsifiable objection ("this feels fragile")
     is a question, not a block.
   - State the *why* behind the rule, in one sentence, so the author learns the
     rule and not just this fix.
   - Verdict: approve only if you'd be comfortable being paged for this code;
     request changes if any blocking item stands; say explicitly which comments
     are take-it-or-leave-it.

   Exit criterion: every comment is labeled, every blocking comment has a
   failure scenario, and the overall verdict is stated.

## Decision points

- **Diff is your own (self-review before requesting review):** run the same
  passes, but write the findings as fixes, not comments — and re-run the review
  after fixing, because fixes introduce bugs at the same rate as features.
- **Diff over ~400 lines:** review commit-by-commit if the commits are coherent;
  otherwise ask the author to split it. Review quality degrades sharply with
  size, and "LGTM" on a 2000-line diff is a rubber stamp, not a review.
- **Pure rename/move/format diff:** verify it is actually pure — `git diff -w`
  and a search for any logic token change — then skip passes 2–3. A "pure
  refactor" with one hidden logic change is the classic smuggling vector.
- **Generated code or vendored dependency updates:** review the generator input
  or the changelog/lockfile delta, not the output line by line.
- **You cannot tell whether a line is a bug without domain knowledge:** file it
  as a **question** with your specific uncertainty ("does the API guarantee
  order here?"), not as a block and not as silence.
- **Urgent hotfix under incident pressure:** narrow scope to pass one, pass two's
  error-handling item, and rollback safety; file the rest as a follow-up issue.
  Cross-reference `incident-response`.

## Quality bar

- [ ] Change intent stated in one sentence before reading the diff.
- [ ] Every hunk read with its surrounding unchanged code, not in isolation.
- [ ] Each changed function traced with at least one concrete value.
- [ ] All six omission categories given an explicit verdict.
- [ ] Named the assertion that fails if the change is reverted (or blocked on
      its absence).
- [ ] Sibling-pattern comparison performed; registration gaps checked.
- [ ] Every comment labeled blocking / non-blocking / question.
- [ ] Every blocking comment states a concrete failure scenario and its why.
- [ ] Verdict passes the paging test: you would take the 3am page for this code.

## Common traps

- **Reviewing only the changed lines.** Most bugs live where the diff meets the
  unchanged code — the caller that isn't updated, the invariant established
  elsewhere. Correction: open full files; for every changed function, read at
  least one caller.
- **Style comments on a diff with a logic bug.** Ten naming nitpicks signal
  thoroughness while the data-corruption bug ships. Correction: enforce the
  pass order; do not write a style comment until passes 1–3 are clean or filed.
- **Trusting green CI as correctness.** CI proves the tests pass, not that the
  tests test the change. Correction: step 4's revert question, every time.
- **Reading code as prose instead of executing it.** Prose-reading confirms the
  code matches its comments; only value-tracing finds off-by-ones and
  null-flows. Correction: pick concrete inputs and follow them.
- **Vague blocks ("this seems risky").** Unfalsifiable objections teach nothing
  and breed resentment. Correction: name input X and outcome Y, or downgrade to
  a question.
- **Approving to be nice, or blocking to show rigor.** Both substitute social
  signaling for judgment. Correction: the paging test is the only approval
  criterion.

## Escalation

- The diff touches authn/authz, cryptography, payment, or data-deletion paths
  and you are not confident in the threat model: request a security-owner
  review rather than approving on general correctness.
- You find a bug that likely exists in production already (the diff copies it
  from existing code): file it as its own issue immediately; do not let it hide
  inside a PR comment thread.
- Third round of review on the same PR with new blocking findings each round:
  stop reviewing asynchronously and pair with the author — the change likely
  needs redesign, which `architecture-decisions` or `extending-code` handles
  better than review ping-pong.
- The change has no description and its intent cannot be inferred in five
  minutes: return it for a description before reviewing. Reviewing intent-less
  diffs means guessing the spec, and a review against a guessed spec is void.
