---
name: architecture-decisions
description: Structures technology and design choices that are expensive to reverse, and records the reasoning in an ADR. Use when choosing a database, framework, queue, service boundary, protocol, or build-vs-buy; when asked "should we use X or Y", "how should we architect this"; or when a change would be costly to undo once data, clients, or teams depend on it.
---

# Architecture Decisions

## Purpose

You face a choice that will be expensive to reverse — a database, a service
boundary, a wire format, a build-vs-buy call. The standard: the decision is made
against measurable criteria, at least two real options were genuinely considered,
and the reasoning is written down where it survives personnel changes. The goal
is not the perfect choice; it is a defensible choice whose costs were seen in
advance and whose reopening conditions are stated.

## Core workflow

1. **Classify reversibility first.** Ask: once this ships, what would undoing it
   cost? A choice is a **two-way door** if reversal is cheap — behind an
   interface, no persisted data format, no external consumers, swappable in
   days (a JSON library, an internal helper's shape, a linter). A choice is a
   **one-way door** if reversal requires data migration, client coordination,
   retraining a team, or rewriting a system boundary (database engine, public
   API contract, service decomposition, cloud provider, core language). Two-way
   doors need a quick decision and a default, not a document — spending a week
   deciding a reversible thing costs more than deciding it wrong. Only one-way
   doors earn the rest of this process.

   Exit criterion: reversibility stated with the concrete reversal cost ("undo
   = migrate 40M rows and re-release two mobile clients"), and two-way doors
   exited here with a decision made today.

2. **Write the problem and constraints before naming any technology.** One
   paragraph: what must this handle (load, data size, latency, consistency
   needs, team size, deadline, budget, compliance)? Put numbers on it, even
   rough ones — "about 50 writes/sec, 10GB/year, one team of four, must ship in
   two months". Why: without stated constraints, every option "works" and the
   loudest preference wins.

   Exit criterion: constraints listed with numbers; anyone could check an
   option against them without you present.

3. **Enumerate at least two real options — including "do nothing / extend what
   we have".** A decision with one option is not a decision, it's a
   rationalization: the write-up becomes advocacy for a foregone conclusion.
   Each option must be one the team could actually execute, described in 2–4
   sentences of how it would work here. Include the incumbent (current system,
   or "no change") as an option whenever one exists — it has the lowest
   migration cost by definition and forces the new thing to justify itself.

   Exit criterion: two or more options, each concrete enough that a reader can
   picture the implementation, none a strawman.

4. **Score options against measurable criteria.** Build a small table. Rows =
   options; columns, at minimum:
   - *Cost of change later:* if this is wrong, what does switching cost in
     weeks and in data/client migration?
   - *Blast radius on failure:* when it breaks at 3am, what else goes down, and
     how many users notice?
   - *Operational load:* who gets paged, what must be learned to run it
     (upgrades, backups, scaling, monitoring), and is that on-call rotation
     staffed for it?
   - *Team familiarity:* how many current team members have run this in
     production (not tutorials — production)? Unfamiliar tech taxes every
     future incident and every future hire.
   Add problem-specific criteria from step 2 (latency, cost, compliance). Fill
   each cell with a fact or estimate, not an adjective — "one engineer has run
   Kafka in prod; nobody has been on-call for it" beats "moderate familiarity".

   Exit criterion: table complete; every cell is checkable by someone else.

5. **Apply the strong defaults, and demand evidence to override them.**
   - *Boring technology wins ties.* Old, widely-deployed tech has documented
     failure modes and a decade of Stack Overflow answers; novel tech fails in
     ways nobody has written down yet. You have a small budget of novelty
     tokens — spend them only where the boring option demonstrably cannot meet
     a stated constraint.
   - *Monolith-first.* Distribution adds failure modes (partial failure,
     network partitions, distributed tracing, versioned deploys) before it adds
     value; split a service out only when a measured constraint — independent
     scaling, team contention on deploys, isolation requirement — demands it.
   - *Buy/adopt before build* for anything outside your core differentiation.
     Auth, payments, search, analytics pipelines are someone's whole company;
     your half-built version gets their year-one bugs with none of their
     roadmap. Build only what makes your product different.
   Overriding a default is allowed — but the ADR must name the constraint from
   step 2 that the default fails.

   Exit criterion: for each default, either the decision follows it or the
   override is justified by a specific, numbered constraint.

6. **Decide, and write the ADR in-repo.** One markdown file,
   `docs/adr/NNNN-short-title.md` (or the repo's existing convention — check
   before inventing one), with sections:
   - *Context:* the problem and constraints from step 2.
   - *Options considered:* from step 3, with the scoring table from step 4.
   - *Decision:* what was chosen, in one sentence.
   - *Consequences:* what this makes easier AND what it makes harder — every
     real decision makes something harder, and an ADR listing only upsides is
     advocacy, not a record.
   - *Revisit triggers:* see step 7.
   Why in-repo: the reasoning must survive personnel changes and be findable
   from the code it governs; a slide deck or chat thread is unfindable in two
   years, and the next team will re-litigate the decision without the facts.

   Exit criterion: ADR committed alongside the code (or handed off for commit),
   readable in under five minutes.

7. **State the revisit triggers as observable conditions.** "Revisit if needed"
   is a non-statement. Write conditions someone can notice: "reopen if write
   volume exceeds 500/sec", "if the vendor's price increase exceeds 30%", "if
   we hire a second team that needs independent deploys", "if p99 latency
   exceeds 200ms for a week". Why: decisions rot silently; explicit triggers
   turn "we should probably rethink this someday" into a checkable alarm.

   Exit criterion: at least one measurable trigger per major assumption the
   decision rests on.

## Decision points

- **Two-way door:** decide within the hour using the defaults in step 5; skip
  the ADR or write three lines in the PR description. Process cost must stay
  below reversal cost.
- **One-way door but deadline is days away:** pick the option with the lowest
  cost-of-change-later (usually the boring/incumbent one) even if it scores
  worse elsewhere — it preserves optionality; write the ADR after shipping and
  set an aggressive revisit trigger.
- **Options tie on the scorecard:** take the one the team knows best; execution
  quality with familiar tech beats theoretical fit with unfamiliar tech.
- **You lack the facts to fill a scoring cell:** time-box a spike (a day, not a
  sprint) to produce that number — a prototype answers "can it do X" faster
  than a debate; then return to the table.
- **The decision constrains other teams (shared API, shared infra):** circulate
  the draft ADR to affected teams before deciding, not after — objections are
  cheap pre-decision and organizationally expensive post-decision.
- **You are extending, not architecting** (a similar example already exists in
  the codebase): stop — this is `extending-code` territory; follow the existing
  pattern instead of re-deciding it.

## Quality bar

- [ ] Reversibility classified with a concrete reversal cost, not a label.
- [ ] Constraints written with numbers before options were named.
- [ ] Two or more executable options, including the incumbent/do-nothing where
      one exists; no strawmen.
- [ ] Scorecard covers cost-of-change, blast radius, operational load, and team
      familiarity; every cell a checkable fact.
- [ ] Each strong default either followed or overridden by a named constraint.
- [ ] ADR in-repo with context, options, decision, consequences — including
      what got harder.
- [ ] Revisit triggers are observable conditions with thresholds.

## Common traps

- **Resume-driven choice.** Picking tech because it is interesting to work with
  or good on a CV optimizes the engineer's career against the system's
  operability. Correction: the scorecard has no "interestingness" column; if an
  option only wins on excitement, it loses.
- **Deciding by analogy to a company whose constraints you don't share.**
  "Google/Netflix does X" — with a thousand engineers, dedicated platform
  teams, and traffic six orders of magnitude beyond yours. Correction: copy
  their constraints-to-solution *reasoning* only if your step-2 constraints
  match; they almost never do.
- **One-option "decisions".** A document that considers only the desired answer
  is a rationalization wearing an ADR's clothes. Correction: if you cannot
  write a second option a smart colleague would defend, you haven't understood
  the problem yet.
- **Consequences section with no downsides.** Every real choice makes something
  harder; omitting it means the next team discovers the cost unwarned.
  Correction: minimum one "this makes X harder" entry, written concretely.
- **Treating the decision as permanent.** Constraints drift; a right choice at
  10 users is wrong at 10 million. Correction: revisit triggers, and actually
  check them when their metrics move.
- **Deciding forever.** Analysis beyond the point of new information is delay
  wearing rigor's clothes. Correction: when the table is full and a spike has
  answered the open question, decide within a day.

## Escalation

- Reversal cost of the leading option exceeds roughly three team-months of
  effort,
  or the decision binds other teams' roadmaps: require sign-off from the
  senior-most affected engineer/architect before committing.
- The scorecard's winner contradicts a strong default and the overriding
  constraint is itself an estimate you're unsure of: get the estimate validated
  (load test, vendor call, prototype) before deciding.
- Two stakeholders read the same scorecard and reach opposite conclusions: the
  criteria weights are the real disagreement — escalate the *weighting*
  question (what matters more, cost or latency?) to whoever owns the outcome.
- Compliance, data-residency, or contractual terms appear anywhere in the
  constraints: involve legal/security before the ADR is marked decided; an
  architecture that violates a contract is not reversible at any engineering
  cost.
