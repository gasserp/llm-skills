---
name: systematic-debugging
description: Hypothesis-driven root-cause analysis for bugs whose cause is not obvious. Use when something is broken and a first look doesn't explain it, when a test fails intermittently, when asked to "figure out why", "debug", "investigate", or "find the root cause", or when a previous fix attempt didn't work. For live production outages, use incident-response first and return here after mitigation.
---

# Systematic Debugging

## Purpose

Something is broken and the cause is not obvious. The standard: find the root
cause by running the cheapest experiment that discriminates between hypotheses,
one variable at a time, and declare the bug fixed only when the original
reproduction passes AND you can narrate the complete causal chain from defect to
symptom. Anything less is symptom-hiding with extra steps.

## Core workflow

1. **Reproduce first, and make it deterministic.** Before reading or changing any
   code, get a command that fails on demand: fixed random seed, pinned input
   data, pinned versions, single thread if concurrency is suspected. Then shrink
   it — smallest input, fewest steps, fastest runtime — because you will run this
   repro dozens of times and its runtime multiplies your whole session. Why this
   is non-negotiable: a bug you can't reproduce is a bug you can't verify fixed;
   without a repro, "fixed" means "stopped happening for unknown reasons".

   Exit criterion: one command, failing every run (or with a known failure rate
   you can measure, e.g. "fails 8/10 runs"), taking seconds not minutes.

2. **Read the actual error, completely.** Read the full error message and the
   entire stack trace, including the caused-by chain and the first frame in
   *your* code, not just the top frame. Extract every fact: exact exception type,
   the offending value, the file:line. But treat the reported location as where
   the corpse was found, not where the murder happened — a null that crashes in
   the formatter was usually created three layers up. Why: half of "hard" bugs
   are solved by facts already printed in the error that nobody read past line one.

   Exit criterion: you can state the symptom precisely ("`user.email` is None at
   render time for users created via the import path"), not vaguely ("it crashes").

3. **Open a hypothesis ledger.** In scratch notes, keep a running table:

   | # | Hypothesis | Cheapest discriminating test | Result |
   |---|---|---|---|
   | 1 | Cache returns stale entry after TTL change | Disable cache, rerun repro | Still fails → dead |

   Every hypothesis must name the *cheapest* experiment whose outcome differs
   depending on whether the hypothesis is true — a test that passes either way
   discriminates nothing. Why keep the ledger: without it you will re-test dead
   ends an hour later, and under pressure your memory of "what I already ruled
   out" is unreliable.

   Exit criterion: ledger exists; every experiment you run is written down with
   its result before you run the next one.

4. **Binary-search the search space.** Halve, don't inspect linearly:
   - *Regression (used to work)?* `git bisect run <repro-command>` — it's
     automated binary search over history and finds the guilty commit in
     log₂(N) runs. This is usually the single highest-value move available.
     Wrap the repro so it exits 0 on success and 1 on failure (bisect trusts
     exit codes, not output — a repro that merely prints the wrong answer
     bisects to garbage; exit 125 skips an untestable commit).
   - *Data-dependent?* Halve the input until the failing record is isolated.
   - *System too big?* Disable or stub subsystems (cache off, queue drained,
     middleware removed) to halve the suspect surface.
   - *Somewhere in a pipeline?* Check the midpoint's intermediate state to learn
     which half is corrupt.

   Why: halving turns a 1000-suspect space into ~10 experiments; linear reading
   turns it into an afternoon.

   Exit criterion: suspect region is small enough to read exhaustively (one
   function, one commit, one record).

5. **Change one variable at a time.** Each experiment alters exactly one thing
   from the known-failing baseline, and you revert it before the next experiment
   (`git stash`, config flag back). Why: two simultaneous changes that "fix" the
   bug leave you not knowing which mattered — you've traded one unknown for two.

   Exit criterion: every ledger entry names exactly one altered variable, and
   the baseline was restored between entries.

6. **Choose instrumentation vs code-reading deliberately.**
   - *Read the code* when the logic is static and the state is simple: pure
     functions, config resolution, branching logic. Reading is faster and
     side-effect free.
   - *Instrument* (targeted logging, debugger, `print` of the suspect value at
     three points along the path) when state is dynamic, timing-dependent, or
     environment-dependent: concurrency, caches, retries, framework lifecycle,
     "works on my machine". Why: for dynamic state, the code tells you what
     *could* happen; only observation tells you what *did*.
   - When instrumenting, log the suspect value at the boundary between "known
     good" and "known bad" layers — that placement is itself a binary search.

   Exit criterion: for the current hypothesis you can say which mode you chose
   and why, and the observation (or reading) produced a recorded ledger result.

7. **Fix at the root, then verify the whole chain.** Once the ledger converges on
   a cause, write down the causal chain: defect → intermediate corruption →
   observed symptom, with a file:line for each link. Fix the defect (not the
   symptom's location). Then:
   - Re-run the ORIGINAL, un-shrunk reproduction — it must pass.
   - Re-run the full test suite — the fix must not break neighbors.
   - Add a regression test that fails without the fix (see `writing-tests`).

   If you cannot narrate cause → effect end to end, you have hidden a symptom,
   not fixed a bug — the defect is still there and will resurface wearing a
   different stack trace.

   Exit criterion: original repro green, suite green, regression test red-then-
   green, causal chain written in one paragraph.

## Decision points

- **Cannot reproduce at all?** Do not fix blind. Instead: harvest evidence from
  where it did happen (logs, core dump, failing CI artifact, exact env versions)
  and reproduce the *environment* first. If still impossible, add targeted
  instrumentation to production paths and wait for the next occurrence — that is
  a legitimate outcome of a debugging session.
- **Intermittent (fails k out of n runs)?** Measure the baseline rate first, then
  judge experiments statistically: run n times, compare rates. Suspect
  concurrency, time, ordering (hash/dict iteration), or shared state between
  tests; force determinism (single thread, frozen clock, fixed seed) and see
  which flip makes it deterministic — that flip names the cause family.
- **Regression vs never-worked?** If it ever worked, `git bisect` before any code
  reading. If it never worked, bisect is useless; go to input/subsystem halving.
- **Bug in a dependency suspected?** Prove it with a minimal repro *outside* your
  codebase using only the dependency. If it repros: pin/upgrade/workaround and
  file upstream. If it doesn't: the bug is yours; return to the ledger.
- **Production is actively down?** Stop. Switch to `incident-response`: mitigate
  first, and come back to this skill for root cause after service is restored.
- **Heisenbug (vanishes under debugger/logging)?** The instrumentation changed
  timing or memory — that fact itself indicts concurrency or undefined behavior.
  Switch to lower-impact observation (counters, post-hoc log analysis, record
  timestamps only).

## Quality bar

- [ ] Deterministic (or rate-quantified) minimal reproduction existed before the
      first code change.
- [ ] Hypothesis ledger shows every hypothesis tested with a discriminating
      experiment and a recorded result; no experiment repeated.
- [ ] Root cause identified at a specific file:line, not a component ("somewhere
      in caching" is not a root cause).
- [ ] Causal chain defect → symptom written out and each link verifiable.
- [ ] Fix applied at the defect, not at the crash site, unless they coincide.
- [ ] Original reproduction re-run and passing; full suite passing; regression
      test added that fails without the fix.
- [ ] All debugging instrumentation removed from the final diff — except
      instrumentation deliberately shipped under the cannot-reproduce branch
      (see Decision points), which is called out in the report.

## Common traps

- **Shotgun debugging.** Changing several things at once until the symptom stops.
  You now have an untraceable diff and no knowledge of which change mattered — or
  whether the bug is merely masked. Correction: one variable per experiment,
  revert between experiments, ledger entry for each.
- **Fixing without a repro.** "This looks wrong, I'll fix it" ships a plausible
  patch for an unverified cause; the real bug survives. Correction: no fix lands
  without a repro that fails before and passes after.
- **Stopping at the first plausible cause.** Plausible is not proven. The first
  explanation that fits the symptom often shares that property with three other
  explanations. Correction: run one discriminating test that would *fail* if your
  cause were wrong before you write the fix.
- **Trusting the stack trace's top frame.** The trace shows where the invalid
  state was *detected*. Correction: walk backward from the crash asking "who
  produced this value?" until you reach the frame where the state first became
  wrong.
- **Debugging the wrong layer of a flaky test.** Rerunning until green and
  merging leaves a landmine. Correction: a flaky test is a real bug in the test
  or the code; apply the intermittent-bug branch above.
- **Ledger in your head.** After 90 minutes you will re-run experiment #2 and
  contradict your own memory of its result. Correction: write results down at the
  moment you observe them.

## Escalation

- Two hours (or ~10 ledger entries) without narrowing the suspect region: pause
  and present the ledger to a human — the ledger itself makes the handoff cheap,
  and a fresh reader often spots the untested assumption.
- Root cause lands in code you cannot change (third-party service, closed
  dependency, platform): document the causal chain, implement the workaround
  with the smallest surface area that does not patch or monkey-patch the
  dependency's internals, document it with a link to the upstream issue at the
  call site, and escalate the upstream fix with your minimal repro attached.
- Evidence contradicts itself (the same experiment gives different results with
  all known variables pinned): suspect the environment (dirty build cache, two
  processes, wrong binary) — verify you're running what you think you're running,
  and if that checks out, escalate; environment schizophrenia wastes days solo.
- The fix requires changing behavior other code may depend on: stop and read
  `safe-refactoring` and `reviewing-code`; get a second opinion before widening
  the blast radius.
