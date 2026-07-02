---
name: writing-tests
description: Designs tests that pin observable behavior at the right level, with one reason to fail each and diagnostic failure output. Use when writing or reviewing tests, deciding unit vs integration vs end-to-end, choosing what to mock, adding coverage for a change, or asked "is this tested well enough".
---

# Writing Tests

## Purpose

You are building the safety net that lets this code be changed by strangers.
The standard: tests pin the contract (what callers can observe), survive any
refactor that preserves behavior, and when one fails, its name and message tell
you what broke without opening a debugger. A test that fails for reasons other
than a real behavior change trains people to delete tests.

## Core workflow

1. **Write down the contract before any test code.** For the unit under test,
   list its promises as observable statements: given input X, returns/raises/
   emits/persists Y. Include the error promises ("raises ValidationError on
   negative amount"), not just success. If you can't state a promise without
   naming a private function or internal field, you're describing implementation,
   not contract — and tests of implementation break on every refactor while the
   behavior is still correct, which is the exact inversion of a test's job.
   *Exit criterion:* a promise list phrased entirely in terms visible to a
   caller or user.

2. **Choose the level by the failure mode you're guarding.**
   - *Unit* — logic branches: calculations, parsing, state machines, edge-case
     handling. Fast, precise blame, run thousands cheaply.
   - *Integration* — wiring and contracts between components: does the query
     match the schema, does the handler decode what the client encodes, does DI
     actually construct the graph. Bugs here are invisible to unit tests because
     each side passes against its own assumptions.
   - *End-to-end* — the few flows whose breakage is existential (signup,
     checkout, deploy). Keep this set small: e2e tests are slow, flaky-prone,
     and give vague blame, so each one must earn its place by guarding a flow
     you cannot afford to discover broken in production.
   Match each promise from step 1 to the cheapest level that can catch its
   realistic failure. Do not re-test pure logic through e2e, and do not "unit
   test" wiring by mocking both sides — that verifies your mocks agree with each
   other, nothing more.
   *Exit criterion:* every promise assigned a level, with the bulk at unit.

3. **Run the edge-case checklist against every input.** For each parameter,
   collection, and external response the code consumes, walk this list and
   either write the case or note why it's impossible:
   - **empty** (empty string, empty list, null/None, missing key)
   - **exactly one** (loops and pluralization break differently at n=1)
   - **many** (enough to expose pagination, batching, O(n²))
   - **boundary values** (0, -1, max size, max int, off-by-one at limits)
   - **invalid input** (wrong type, malformed, out of range — assert the
     *specific* error, not just "throws")
   - **duplicate** (same item twice; same request retried)
   - **concurrent / repeated calls** (idempotency, shared-state races)
   This list exists because these are where real bugs cluster; "it works" tests
   re-verify the case the author already ran by hand.
   *Exit criterion:* checklist walked for each input; every skipped item has a
   stated reason.

4. **Write each test with one reason to fail and a name that states the
   expected behavior.** Name pattern: `<unit> <behavior> <condition>` —
   `returns_empty_list_when_no_matches`, `rejects_transfer_when_balance_insufficient`.
   Never `test2`, `test_edge_cases`, `test_process` — when that fails in CI six
   months from now, the name is the first (often only) thing the reader sees.
   One logical assertion per test: multiple unrelated assertions mean the first
   failure masks the rest and the test's name can't be honest about what it
   checks. Several assertions on one result object count as one logical
   assertion.
   *Exit criterion:* reading only the test names reproduces your promise list
   from step 1.

5. **Structure as arrange-act-assert with visible intent.** Setup that matters
   to the behavior stays visible in the test (`user = make_user(balance=0)`);
   setup that doesn't goes in builders/fixtures with safe defaults. A reader
   must see *why this input produces this output* without chasing five
   fixtures — magic values pulled from a distant fixture make the test
   unfalsifiable by inspection. Exactly one "act" line.
   *Exit criterion:* each test readable top-to-bottom as given/when/then.

6. **Mock only what you own or what is expensive/nondeterministic.** Legitimate
   mock targets: the clock, randomness, the network, third-party paid APIs,
   interfaces *you* defined at your own boundaries. Do not mock the standard
   library, your own value objects, or cheap in-process collaborators — every
   mock hard-codes an assumption about the real thing, and a mock that drifts
   from the real dependency makes tests pass while production fails. For
   dependencies you don't own, wrap them in an interface you do own, mock that,
   and back it with at least one contract/integration test against the real
   thing (or its official fake) so the wrapper's assumptions get checked.
   *Exit criterion:* every mock is on an owned interface or a
   nondeterministic/expensive boundary, and each such boundary has one real
   integration check somewhere in the suite.

7. **Make failures diagnostic, then prove each test can fail.** Use assertions
   that print expected vs actual (`assertEqual`, matcher libraries), add
   messages where the values alone won't explain ("balance after refund"), and
   avoid `assert result` on complex objects. Then, for each new test, break the
   code it guards (invert the condition, return early) and confirm the test
   fails *with a readable message* — a test you've never seen fail may be
   asserting nothing (wrong fixture, unawaited async, assertion after early
   return).
   *Exit criterion:* every new test observed red once, with output that names
   what broke.

## Decision points

- **Behavior spans a DB query or external schema?** Integration test against a
  real instance (testcontainers, local docker, in-memory only if it's the same
  engine). Mocked-out persistence tests can't catch the top persistence bug:
  query/schema mismatch.
- **Nondeterminism (time, random, ordering)?** Inject the source (clock
  parameter, seeded RNG) rather than sleeping or retrying — flaky tests get
  ignored, and an ignored suite is a deleted suite with extra CI cost.
- **Bug fix?** First write the test that reproduces the bug and fails on the
  old code; then fix; the test is the regression guard and the proof the fix
  addresses the actual bug and not a lookalike.
- **Testing code with terrible seams (statics, globals, God objects)?**
  Characterization tests through the outermost callable boundary first (see
  `safe-refactoring`), then refactor seams in, then write proper unit tests.
- **Table-driven vs individual tests?** Table-driven when cases share one
  behavior with varied data (parser inputs). Individual when behaviors differ —
  a 40-row table asserting different *kinds* of things has forty reasons to fail
  and one name.

## Quality bar

- [ ] Test names alone reconstruct the contract; no `test1`/`misc` names.
- [ ] Every promise, including error promises, has a test at the cheapest
      adequate level.
- [ ] Edge-case checklist (empty / one / many / boundary / invalid / duplicate /
      concurrent-or-repeated) applied to every input, gaps justified.
- [ ] No test asserts on private state, call order of internals, or exact log
      strings, unless that IS the public contract.
- [ ] Mocks confined to owned interfaces and expensive/nondeterministic
      boundaries; each mocked boundary has a real integration check.
- [ ] Every new test seen failing once, with a message that names the breakage.
- [ ] Suite is deterministic: no sleeps, no wall-clock, no order dependence.

## Common traps

- **Chasing coverage percentage instead of failure-mode coverage.** 95% line
  coverage with assertion-free tests, or lines "covered" by a test that would
  pass anyway, protects nothing — coverage measures execution, not verification.
  Correction: use coverage to *find untested branches*, never as the goal; the
  goal is that each realistic failure mode flips at least one test red.
- **Asserting internals.** `expect(service._cache.size).toBe(1)` or verifying a
  private method was called: breaks on every refactor while behavior is intact,
  teaching the team that red tests are noise. Correction: assert what a caller
  observes; if you can't, the design lacks an observable seam — fix the seam.
- **Mocking the thing under test's neighbors until nothing real runs.** The
  test passes because the mocks agree with each other. Correction: count what's
  real in the test; if the answer is "only glue", promote it to an integration
  test or delete it.
- **One giant test per feature.** Fails at the first assertion, hides the other
  nine, and its name can't say what it checks. Correction: split by promise;
  shared setup goes in a builder, not by fusing tests.
- **Copying the implementation into the expectation.** Computing the expected
  value by running the same algorithm the code runs — the test passes even when
  both are wrong. Correction: hand-compute expectations, or use known-good
  values from a spec or a trusted independent source.

## Escalation

Stop and ask a human when:

- The contract itself is undocumented and ambiguous — pinning the wrong promise
  makes the suite enforce a bug (relates to `safe-refactoring`
  characterization).
- Adequate testing needs infrastructure that doesn't exist in CI (real
  third-party sandbox, hardware) — the gap is a risk decision, not yours alone.
- You find an existing test that is wrong but green — other code may rely on
  the wrong behavior; changing it is a behavior change, not a test fix.
- Guarding the failure mode properly requires a design change (no seam, global
  state) whose cost exceeds the change you were asked to make.
