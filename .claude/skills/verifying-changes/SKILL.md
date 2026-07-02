---
name: verifying-changes
description: Defines and executes the verification a change needs before anyone claims it is done — running the real flow, error paths, and neighbors, and reporting with evidence. Use before saying "done", "fixed", "implemented", or "should work now"; when asked to verify, validate, or double-check a change; or when writing a completion report or PR description.
---

# Verifying Changes

## Purpose

You are about to claim a change works. The standard: "done" means you *observed*
it working end to end — not "it compiles", not "unit tests are green". Tests
verify what you thought to test; running the real flow verifies what you didn't
think of. A claim without evidence is a hope, and hopes filed as reports destroy
trust and cost others a debugging session.

## Core workflow

1. **List what could now be broken.** Before running anything, write down:
   (a) the flows the change is supposed to affect, (b) the error paths in or
   near the changed code, (c) the neighbors — everything that shares the code
   you touched. Find neighbors mechanically: `grep` for callers of every
   function you changed, importers of every module
   (`grep -rn "import <module>\|from <module>"`), and other users of any config
   key, schema, or shared helper you modified. Verification driven by "what did
   I intend" misses exactly the breakage you didn't intend.
   Exit criterion: a checklist of flows: affected, error, neighbor.

2. **Build and run from a clean, production-like state.** Rebuild the artifact
   that actually runs (`npm run build`, `make`, container image), clear caches
   that could serve stale code (bundler cache, `__pycache__`, hot-reload state,
   old service instances still running — check with `ps`/`docker ps`), and use
   data that resembles reality rather than a hand-seeded row that happens to
   dodge the bug. Verifying a stale build verifies the *previous* version of
   your change.
   Exit criterion: you can state which artifact ran and why it must contain
   your change (fresh build timestamp, image digest, restarted process).

3. **Exercise the ACTUAL affected flow at the layer the user experiences.**
   Not the function in a REPL — the real entry point:
   - CLI change → run the real command a user would type, from a clean shell.
   - API change → real HTTP request (`curl -i ...`) against a running server;
     check status code, headers, and body, not just "no exception".
   - UI change → perform the real click/keystroke path; screenshot or DOM-check
     the result.
   - Library change → a consumer-style snippet importing the built package the
     way users do, not test-internal imports.
   - Config/infra change → apply it where it runs (staging), and confirm the
     system picked it up (log line, effective-config endpoint).
   Calling the inner function skips parsing, routing, serialization, auth, and
   wiring — which is where integration bugs live.
   Exit criterion: real invocation executed; actual output captured.

4. **Verify error paths, not just the happy path.** Most production breakage
   lives in paths nobody ran. From your step-1 list, trigger at least: one
   invalid input, one missing/unavailable dependency if the change touches one
   (kill the connection, point at a bad URL), and one permission-denied case if
   the change touches auth. Confirm the failure is the *designed* failure —
   correct status/message, no stack trace to the user, no half-written state
   left behind.
   Exit criterion: each relevant error path observed producing its intended
   failure, output captured.

5. **Regression-check the neighbors.** Run the flows from step 1(c): the other
   callers of what you changed, at whatever layer is cheapest but real. Also
   run the existing test suite for the touched modules — it is necessary
   (cheap, broad) but not sufficient (only pins what someone thought to pin).
   Exit criterion: every neighbor flow on the list ran; suite green.

6. **Report with evidence, and never imply more than you did.** The report
   contains the actual commands and their actual output (trimmed to the
   relevant lines), one per claim:

   ```
   $ curl -is localhost:8080/api/orders/42 | head -5
   HTTP/1.1 404 Not Found
   {"error":"order not found","id":"42"}
   ```

   "Should work" is not a report. If something could not be run (no prod
   access, missing credentials, environment won't build), say precisely:
   "Verified: unit + integration tests, real CLI run of X and Y. NOT verified:
   behavior against the real S3 bucket — no credentials in this environment."
   Stating the gap lets the reader cover it; hiding it converts your gap into
   their outage.
   Exit criterion: every claim in the report is backed by pasted output or
   explicitly labeled unverified.

## Decision points

Scale verification to blast radius — proportionality, not ceremony:

- **Docs/comment-only change:** render or preview it (broken markdown and dead
  links are the failure mode); confirm no code is touched (`git diff --stat`).
  Steps 3–5 don't apply.
- **Single-module logic change:** full workflow, neighbor check can be the
  module's test suite plus one real invocation of the main flow.
- **Shared helper / cross-cutting change:** full workflow with emphasis on
  step 5 — enumerate ALL callers; run each caller's primary flow or its
  integration tests.
- **Schema migration / data change:** everything above, plus: run the migration
  on a realistic-size copy (timing and locks), verify rollback actually
  restores, verify old code runs against the new schema (deploys overlap).
- **Dependency bump:** run the app's real flows, not just the suite — the
  breakage mode is changed behavior inside the dependency that no local test
  asserts. Read the dependency's changelog for the jumped range.
- **Cannot run the flow at all (no env, no creds)?** Do everything runnable,
  then escalate per below — do not substitute confidence for observation.

## Quality bar

- [ ] The affected flow ran at the user-experienced layer, on a fresh build,
      and the output is in the report.
- [ ] At least one error path per touched failure domain observed producing its
      designed failure.
- [ ] Neighbors enumerated by grep, not memory; each one's flow or integration
      tests ran.
- [ ] Full relevant test suite green — as a floor, not the claim.
- [ ] Every claim in the report is backed by pasted command + output.
- [ ] Anything unverified is explicitly listed as unverified, with the reason.
- [ ] Verification state matched production shape: fresh artifacts, no leftover
      processes, realistic data.

## Common traps

- **Green CI as a proxy for working software.** CI proves the suite passes,
  and the suite only encodes past imagination; the bug you just wrote is by
  definition the one nobody imagined. Correction: CI is the floor; the real
  flow is the claim.
- **Verifying a state that doesn't match production.** Stale build, cached
  bundle, still-running old process, seeded DB row shaped to survive the bug,
  `DEBUG=true` masking the error handler. You verified an environment, not the
  change. Correction: step 2 — rebuild, restart, verify the artifact identity
  before trusting any observation from it.
- **Happy-path-only verification.** The demo works; the 4xx handler you also
  touched now returns a stack trace. Correction: error paths are on the step-1
  checklist before you run anything, so they can't be forgotten in the relief
  of a working happy path.
- **"Should work" reporting.** Reasoning from the diff instead of observing the
  system — the phrase marks the exact spot where observation was skipped.
  Correction: ban the phrase from reports; each claim gets output or an
  "unverified" label.
- **Testing the inner function and claiming the feature.** The function is
  correct; the route registration you forgot means users get 404. Correction:
  step 3 — enter through the same door the user does.
- **Declaring done at first success.** One green run after five failed
  attempts, with leftover state from the attempts possibly propping it up.
  Correction: reset to clean state and reproduce the success once more; done
  is repeatable, not lucky.

## Escalation

Stop and report (with the verified/unverified split) instead of claiming done
when:

- The affected flow cannot be exercised in your environment (prod-only
  service, missing credentials, hardware) — the human must decide whether to
  accept the risk or provide access.
- Verification reveals behavior that is wrong but pre-existing — fixing it is
  scope expansion (see `extending-code`); silently shipping atop it hides a
  known defect.
- The realistic-data requirement can't be met (migration verified only on toy
  data) — timing/locking on real volume is a deploy risk the owner must accept.
- Neighbor enumeration turns up more callers than the change was scoped for —
  the change's blast radius exceeds its review.
