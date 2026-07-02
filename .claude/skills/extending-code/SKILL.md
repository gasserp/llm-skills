---
name: extending-code
description: Adds features to an existing codebase by copying the house pattern, registering the feature everywhere the codebase expects, and shipping the smallest diff that fully does the job. Use when adding a feature, endpoint, command, flag, field, handler, or capability to code that already exists — "add support for X", "implement Y in this repo", "extend Z".
---

# Extending Code

## Purpose

You are adding a capability to a codebase that already has opinions. The standard:
the finished feature reads as if the original authors wrote it, touches every
registration point the codebase expects, and contains nothing the request didn't
ask for. Consistency with the codebase beats your personal taste — the next
reader knows the house pattern, not yours.

## Core workflow

1. **Find the three most similar existing features.** Before writing anything,
   search for features closest in kind to what you're adding (another endpoint if
   you're adding an endpoint, another CLI subcommand if adding a subcommand).
   Use `grep`/`Glob` for the nouns of the domain and for framework registration
   keywords (`router.`, `@app.route`, `addCommand`, `register`, `subscribe`).
   Three, not one: one example may itself be an outlier; three reveal the pattern.
   *Exit criterion:* you can name three concrete features and the files each one
   lives in.

2. **Copy their shape, not their code.** For each of the three, list: files
   touched, layer boundaries (where validation happens, where business logic
   lives, where persistence is called), naming conventions (file names, function
   names, test names), and error-handling style (exceptions vs result types,
   error codes used). Where the three agree, that is the house pattern — follow
   it even where you'd personally choose differently, because deviation costs
   every future reader a "why is this one different?" investigation.
   *Exit criterion:* a written sketch of your feature's file list and layering
   that mirrors the majority pattern.

3. **Enumerate every registration point.** Features rarely live in one file. Diff
   the *full* set of files a similar feature touched (use
   `git log --follow --name-only -- <file>` or find the commit/PR that added it:
   `git log --oneline --all -- <its main file>`, then `git show --stat <sha>`).
   Build a checklist from what that commit touched. Typical entries:
   - routes / URL maps / command registries
   - dependency injection or wiring modules
   - database migrations and schema files
   - configuration (defaults, env var docs, sample configs)
   - feature flags
   - permissions / authorization rules
   - user-facing docs, CHANGELOG, help text
   - test fixtures and factories
   *Exit criterion:* a checklist where every item is either done or explicitly
   marked not-applicable with a reason.

4. **Wire in at existing seams.** Add your feature through the extension points
   the codebase already exposes (the registry, the interface, the plugin hook,
   the switch that dispatches by type). Do not invent a new abstraction, base
   class, or helper layer to host one feature — an abstraction with one consumer
   is speculation, and speculation is the unrequested. If no seam exists and
   you'd have to modify five call sites, that is a signal to pause and check
   whether a smaller insertion point exists before restructuring (restructuring
   first is `safe-refactoring`, as its own commit).
   *Exit criterion:* your diff adds code at existing extension points; it does
   not add new frameworks-within-the-framework.

5. **Write the smallest diff that FULLY does the job.** "Small" excludes nothing
   required: error paths, input validation, tests, docs updates are part of the
   job, not gold-plating. "Small" excludes the unrequested: drive-by renames,
   reformatting untouched code, refactoring neighbors, extra options "while
   you're in there". If you spot needed cleanup, note it for a separate change.
   *Exit criterion:* every hunk in `git diff` traces to the request or to an
   item on your step-3 checklist.

6. **Verify end to end.** Run the actual feature the way a user would — real CLI
   invocation, real HTTP request, real UI action — including at least one error
   path. Follow `verifying-changes` for the full standard; a feature is not done
   because it compiles and its unit tests pass.
   *Exit criterion:* you have pasted real command output showing the feature
   working and failing gracefully.

## Decision points

- **Three similar features disagree with each other?** Follow the newest one
  *if* git history shows the codebase migrating toward it (newer features use
  it, older ones don't); otherwise follow the majority. Note the ambiguity in
  the PR description so reviewers can correct you cheaply.
- **No similar feature exists at all?** You are not extending, you are
  architecting. Sketch the shape, check `architecture-decisions` if the choice
  is expensive to reverse, and get a human's confirmation on the shape before
  building it out.
- **The house pattern is objectively bad (known bug class, deprecated API)?**
  Still don't fork the pattern silently. Either (a) follow it and file the
  improvement separately, or (b) propose migrating the pattern — for *all*
  usages — as its own change. Never leave the codebase with old-way and new-way
  side by side and no migration plan.
- **Feature needs data model changes?** Migrations come with a rollback path and
  are backward-compatible with the currently deployed code (add column before
  reading it; never drop-and-rename in one step), because deploys and DB changes
  don't land atomically.
- **Request is ambiguous about scope?** Implement the narrowest reading that is
  genuinely useful, and state in the report what you excluded and why. Guessing
  big wastes work; guessing small and saying so costs one round trip.

## Quality bar

- [ ] Three similar features identified; your diff's file layout mirrors theirs.
- [ ] Registration checklist written; every item done or marked N/A with reason.
- [ ] No new abstraction with only one consumer.
- [ ] Error paths handled in the same style the codebase already uses.
- [ ] Tests exist at the same level and in the same location as the sibling
      features' tests (see `writing-tests` for what to cover).
- [ ] Docs/help text/CHANGELOG updated wherever sibling features have entries.
- [ ] Every diff hunk traces to the request; zero drive-by edits.
- [ ] Verified end to end per `verifying-changes`, with output captured.

## Common traps

- **Inventing a new pattern alongside an existing one.** Now the codebase has
  two ways to do the same thing, and every future reader pays to learn which one
  is "real". Correction: copy the incumbent pattern even when yours is nicer;
  propose pattern changes as separate, complete migrations.
- **Registering in some places but not all.** The feature works in your test but
  is missing from config docs, permissions, or the DI graph, and fails in an
  environment you didn't run. Correction: build the step-3 checklist from what a
  sibling feature's *commit* touched, not from memory.
- **"Small diff" used to skip required work.** Shipping happy-path-only and
  calling it minimal. Correction: small excludes the unrequested, never the
  required — error paths, tests, and docs are inside the job's boundary.
- **Generalizing on the first use case.** Adding a plugin system to host one
  plugin. Correction: hard-code the single case at an existing seam; generalize
  on the second or third real consumer, when the axis of variation is known.
- **Modifying shared code without checking other callers.** You bent a helper to
  fit your feature and broke its three other call sites. Correction: `grep` for
  every caller before changing shared code; if behavior must differ, add a
  parameter or a new seam rather than changing the default.

## Escalation

Stop and ask a human when:

- No similar feature exists and the shape you'd invent affects more than the
  files you're adding (new layer, new dependency, new service).
- Fully doing the job requires touching an area the request didn't mention
  (auth model, data migration on a large table, public API contract).
- Two established patterns conflict and git history shows no migration
  direction — a coin flip here becomes precedent.
- The smallest complete diff is far larger than the request implies (a "small
  feature" needing 20+ files) — the estimate mismatch is information the
  requester needs before you proceed.
