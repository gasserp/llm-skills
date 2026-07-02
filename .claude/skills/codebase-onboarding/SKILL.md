---
name: codebase-onboarding
description: Builds a fast, verified mental model of an unfamiliar repository before changing anything. Use when starting work in a repo you don't know, when asked to "get familiar with", "understand", or "explore" a codebase, before the first feature or fix in a new project, or when you catch yourself reading files at random without a goal.
---

# Codebase Onboarding

## Purpose

You are about to change code in a repository you do not know. The standard: within
a time-boxed session (30–60 minutes for most repos), produce a written mental
model accurate enough that you can predict where a given behavior lives — and you
verify that prediction before trusting the model. Depth on one execution path
beats breadth across many files, because breadth-first reading produces
familiarity with names, not understanding of flow.

## Core workflow

1. **Time-boxed recon (10–15 min, hard stop).** Answer five questions with
   commands, not by reading source top to bottom:
   - *What is it?* Read README.md, then the manifest (`package.json`, `pyproject.toml`,
     `go.mod`, `Cargo.toml`, `pom.xml`). The manifest's dependencies tell you the
     tech stack faster than any doc: a web framework, an ORM, a queue client each
     imply a whole architecture.
   - *Where does it start?* Find entry points: `main` functions, `bin/` scripts,
     `scripts`/`entry_points` in the manifest, `Dockerfile` `CMD`, route
     registration files. `grep -rn "def main\|func main\|if __name__" --include="*.py" --include="*.go"`
     or check the manifest's declared entry.
   - *How do I build and test?* Find the commands (`Makefile`, `package.json`
     scripts, `justfile`, CI config in `.github/workflows/`). Run the test suite
     now — a green baseline is your safety net for every later change, and a red
     baseline tells you which failures are pre-existing, not yours.
   - *What is the shape?* `ls` the top two directory levels. Note the layering
     the directory names imply (`handlers/ → services/ → repositories/`, or
     `cmd/ → internal/ → pkg/`).
   - *Where is config?* Env files, `config/` dirs, settings modules, feature
     flags. Config locations reveal what the system considers variable.

   Exit criterion: you can state in one paragraph what the system does, how it
   starts, and how to run its tests — and the tests have actually run.

2. **Trace ONE real operation end to end (15–25 min).** Pick a concrete,
   representative operation — one HTTP request, one CLI command, one message
   consumed — ideally the one closest to the change you'll make. Follow it from
   entry point to side effect (DB write, response, file), reading every file on
   the path and nothing off it. Use `grep` for the route/command string to find
   the entry, then follow calls downward. Do not detour into interesting-looking
   files; note them and move on. Why one path: a single vertical trace shows you
   the layering, the error-handling style, the dependency-injection mechanism,
   and the naming conventions all at once — everything a horizontal skim shows
   you none of.

   Exit criterion: you can narrate the full path ("request enters at X, router Y
   dispatches to handler Z, which calls service A, which uses repository B to
   write table C") without looking at the code.

3. **Inventory the unwritten conventions (5–10 min).** From the traced path plus
   2–3 quick greps, write down:
   - *Naming:* file, class, and function patterns (`UserService` vs `user_svc`?
     test files as `test_*.py` or `*.spec.ts`?).
   - *Error handling:* exceptions vs error returns vs Result types? Where are
     errors translated to user-facing responses? Grep for `raise`, `throw`,
     `errors.New`, or the error middleware.
   - *Layering rules:* which layers import which. A handler that queries the DB
     directly is either the convention or a violation — check three handlers to
     know which.
   - *Registration:* how new things get wired in (decorator, config file, DI
     container, explicit list). Find where an existing route/command/plugin is
     registered; yours goes in the same place.

   Exit criterion: each of the four items above has a concrete answer with a file
   path as evidence.

4. **Write the map down (5 min).** Produce a short artifact (scratch notes or a
   comment in your working notes, NOT a committed file unless asked): the layers,
   the 5–10 key modules and their jobs, where things register, the build/test
   commands, and the conventions inventory. Why write it: an unwritten model
   decays within the hour, and writing exposes gaps you'd otherwise paper over.

   Exit criterion: the map exists and fits on one screen.

5. **Verify the model by prediction.** Pick a behavior you have NOT read the code
   for ("where is authentication checked?", "where does the retry logic live?").
   Predict the file from your map, then check. If wrong, your map has a hole —
   trace a second path through the mispredicted area and fix the map. Why: an
   unverified model feels identical to a verified one until it costs you a wrong
   change.

   Exit criterion: at least one prediction confirmed, or the map corrected after
   a miss.

6. **Find the three most similar examples.** Before writing anything new, locate
   the three existing things most like what you'll build; grep for the nearest
   domain noun or the registration pattern found in step 3. See `extending-code`
   step 1 for the full procedure and the why.

   Exit criterion: three named examples, with file paths, for the change you
   are about to make.

## Decision points

- **Repo under ~5k lines:** skip step 1's directory-shape and config bullets;
  still run the test suite and note the build/test commands, then read the
  entry point and go straight to the trace. Recon overhead exceeds its value,
  but the test baseline is required regardless of size.
- **Monorepo or >100k lines:** scope everything to the one package/service you'll
  touch. Onboard to the subsystem, not the world; treat other packages as
  external dependencies.
- **No tests or tests won't run:** note it, budget extra verification time, and
  read `writing-tests` before changing anything — you'll need a characterization
  test as your safety net.
- **Docs contradict code:** trust the code, and lower your trust in all other
  docs in the repo accordingly. Code executes; docs drift.
- **The traced path hits generated code, metaprogramming, or reflection:** find
  the generator/registry instead of reading generated output; the source of
  truth is the input to the generator.
- **You'll only make a one-line, well-specified fix in a file you've been given:**
  do steps 1 (build/test commands only) and 6; a full trace is over-investment.

## Quality bar

- [ ] Test suite has been run; you know the baseline (green, or which tests are
      pre-existing failures).
- [ ] One real operation traced entry-to-side-effect and narratable from memory.
- [ ] Conventions inventory has file-path evidence for naming, error handling,
      layering, and registration.
- [ ] Written map exists and fits on one screen.
- [ ] At least one where-does-X-live prediction made and checked.
- [ ] Three most-similar examples identified for the change you're about to make.
- [ ] Total time spent is inside the time box; onboarding did not become the task.

## Common traps

- **Breadth-first reading for hours.** Skimming 40 files produces the *illusion*
  of understanding — you recognize names but cannot predict behavior. Correction:
  after the 15-minute recon, all reading must serve the single trace or a
  specific question.
- **Reading documentation instead of running code.** Docs describe intent; only
  the running system shows truth. Correction: run the tests and, if possible, the
  app itself within the first 15 minutes.
- **Trusting one example as "the convention".** The one file you happened to open
  may be the legacy outlier. Correction: three examples minimum before you imitate
  a pattern.
- **Skipping the write-down because "I've got it".** You don't; retrieval without
  notes fails within the hour and you'll re-read the same files. Correction: the
  map is mandatory output, not optional hygiene.
- **Onboarding forever to avoid starting.** Perfect understanding is not the exit
  criterion; a *verified-once* model is. Correction: when the prediction check
  passes, stop and start the change.
- **Ignoring the test/build setup until change time.** Discovering a broken build
  after you've edited five files conflates your changes with pre-existing rot.
  Correction: green (or characterized-red) baseline before the first edit.

## Escalation

- Build or test suite cannot be made to run after 30 minutes of setup attempts:
  ask a human for the incantation instead of burning the session — setup lore is
  often tribal.
- Two consecutive prediction checks fail after correcting the map: the
  architecture is non-obvious (heavy indirection, event-driven, code-gen); ask
  for a 10-minute walkthrough or an architecture doc.
- The operation you must change has no similar existing example anywhere in the
  repo: you're not extending, you're architecting — read
  `architecture-decisions` and confirm the approach with a human first.
- The repo contains multiple candidate implementations of the same concern (two
  auth systems, two ORMs): ask which is current before building on either;
  guessing wrong means building on the deprecated one.
