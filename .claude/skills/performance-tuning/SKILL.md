---
name: performance-tuning
description: Measurement-first optimization of slow code or systems. Use when something is slow, when asked to "make it faster", "optimize", "reduce latency", "cut memory", or "improve throughput", when a performance SLO/budget is missed, or before accepting a change justified only by claimed performance benefits.
---

# Performance Tuning

## Purpose

Something is slow, or you've been asked to make it faster. The standard: every
optimization is justified by a measured baseline, targeted at a profiled hotspot,
and kept only if the number moved — and the work stops the moment an explicit,
pre-agreed target is met. Optimization without measurement is superstition that
costs maintainability.

## Core workflow

1. **Define the metric and the numeric target before touching code.** Write down:
   which operation, which metric (p50/p95/p99 latency, throughput in req/s or
   items/s, peak memory, cost), the current value, and the target value ("checkout
   p95 from 2.1s to <500ms under 100 concurrent users"). If the requester gives no
   target, propose one and get agreement. Why: without a target you cannot know
   when you're done, and "as fast as possible" licenses infinite gold-plating.

   Exit criterion: metric + baseline number + target number written down and
   agreed.

2. **Build a repeatable measurement harness.** A single command that produces the
   metric with benchmark hygiene:
   - *Warm up* before measuring — JIT compilation, connection pools, and page
     caches make the first runs unrepresentative.
   - *Multiple runs* — report median and spread, never a single run; if variance
     exceeds ~10% of the effect you hope to create, fix the noise (isolate the
     machine, longer runs) before proceeding, because you can't detect a signal
     smaller than your noise.
   - *Realistic data sizes and shapes* — an O(n²) hotspot is invisible on 100 rows
     and fatal on 100k; use production-scale data or a documented scale-down.
   - *Watch for caching making re-runs lie* — a second run served from cache
     measures the cache, not your code. Clear caches between runs or measure
     steady-state deliberately, and know which one your target refers to.

   Exit criterion: same command run three times gives numbers within acceptable
   variance, at realistic scale.

3. **Profile to locate the hotspot — before forming any opinion.** Run a profiler
   on the measured operation (`py-spy`/`cProfile`, `perf`, `pprof`, Chrome
   DevTools, `EXPLAIN ANALYZE` for queries; flame graphs for CPU, allocation
   profilers for memory, distributed traces for cross-service latency). Identify
   where the time actually goes as percentages. Why this precedes thinking:
   engineers' guesses about hotspots are wrong more often than right — decades of
   profiling literature and every senior's scar tissue agree — and a wrong guess
   costs days of optimizing the innocent.

   Exit criterion: a ranked list "component X: 62%, Y: 21%, Z: 8%" backed by
   profiler output, not intuition.

4. **Apply Amdahl reasoning to pick the target.** Optimizing code that is 2% of
   runtime caps your total win at 2% — even if you make it infinitely fast.
   Compute the ceiling for each candidate: eliminating a 62% component at best
   yields a 2.6× speedup; halving it yields ~1.45×. Only attack components whose
   ceiling can reach your target; if no single component can, you need a
   structural change (step 5's algorithmic branch, or architecture), not tuning.

   Exit criterion: chosen target's best-case win, computed, is ≥ the gap to the
   target.

5. **Fix algorithmic complexity before constant factors.** Within the hotspot,
   check for complexity bugs first: N+1 queries, O(n²) nested loops over growing
   data, repeated recomputation of invariants, unindexed scans, accidental
   quadratic string building. Why first: a complexity fix's payoff grows with
   data size while micro-optimizations yield fixed small percentages — and a
   micro-optimized O(n²) is still doomed at 10× scale. Only after complexity is
   right, spend on constant factors (allocation reduction, batching, vectorization).

6. **Change one thing, re-measure, keep or revert.** For each optimization: apply
   it alone, run the harness, compare against baseline with variance in mind.
   Keep the change only if the number moved beyond noise; otherwise revert it
   even if it "should" help — unmeasurable improvements are pure complexity cost.
   Record each attempt (change → delta) so you don't retry reverted ideas. Run
   the functional test suite after each kept change: a fast wrong answer is worse
   than a slow right one.

   Exit criterion per change: delta recorded, decision (keep/revert) made, tests
   green.

7. **Stop at the target.** When the harness shows the target met at realistic
   load, stop — even if more wins are visible. Past the target you are trading
   readability, flexibility, and review time for a number nobody asked for.
   Record the final numbers, the harness command, and the changes kept; hand the
   diff to `verifying-changes` and `reviewing-code`.

## Decision points

- **Latency vs throughput vs memory — diagnose which you actually have; each
  needs different tools and fixes:**
  - *Latency* (one operation too slow): trace/profile the critical path; fix via
    removing serial round-trips, caching, indexes, cutting work off the path.
    Parallelism helps only the parallelizable fraction (Amdahl again).
  - *Throughput* (system can't keep up): find the bottleneck resource
    (utilization metrics, queue depths); fix via batching, pooling, horizontal
    scaling, backpressure. Note: batching often *raises* per-item latency —
    check which metric your target names before trading one for the other.
  - *Memory* (OOM, GC pressure, swap): allocation profiler / heap dump; fix via
    streaming instead of materializing, smaller data representations, bounded
    caches. GC pressure frequently masquerades as a CPU/latency problem — check
    GC time share when a latency profile looks flat and diffuse.
- **Caching vs algorithm vs batching vs parallelism — pick by cause:**
  - Same expensive result computed repeatedly with tolerable staleness → *cache*;
    but a cache adds an invalidation bug class, so prefer an algorithmic fix when
    one exists at similar cost. Define eviction and staleness bounds at design
    time, not after the first stale-read bug.
  - Cost grows faster than input size → *algorithm/data-structure* change; no
    cache saves you from O(n²) at scale.
  - Many small operations each paying fixed overhead (network, syscall, txn) →
    *batching*; the win is amortizing overhead, so measure overhead share first.
  - CPU-bound, work divisible, cores idle → *parallelism*; last resort because it
    imports the concurrency bug class, and useless if the bottleneck is I/O or a
    lock.
- **Hotspot is in code you don't own (library, DB engine, network)?** Change your
  usage pattern (fewer calls, better query, bulk API) rather than forking; only
  then consider replacing the dependency — see `architecture-decisions`.
- **Profile is flat (no hotspot >10%)?** Death by a thousand cuts. Look for a
  pervasive cost (allocation churn, logging, serialization, chatty I/O) with an
  allocation or syscall profiler, or reconsider the architecture; per-site tuning
  cannot beat a flat profile.
- **Production slow but benchmark fast?** Your harness is unrealistic — diff the
  environments (data volume, concurrency, cold caches, network) and fix the
  harness before fixing code; otherwise you're optimizing a fiction.

## Quality bar

- [ ] Baseline and target recorded as numbers, with the exact measurement command.
- [ ] Profiler evidence identifies the hotspot; no change justified by intuition
      alone.
- [ ] Amdahl ceiling computed for the chosen target before work started.
- [ ] Each kept change has a recorded before/after delta exceeding measurement
      noise; each non-moving change was reverted.
- [ ] Functional tests pass after every kept change.
- [ ] Final measurement at realistic data size and load meets the target.
- [ ] Work stopped at the target; remaining ideas listed, not implemented.
- [ ] Any added cache has documented invalidation and staleness bounds.

## Common traps

- **Optimizing from intuition.** The guessed hotspot is usually innocent, and the
  "obviously slow" code often measures at 1%. Correction: no opinion until the
  profiler has spoken.
- **Benchmarking the cache.** Re-running the benchmark hits warm caches and
  reports a speedup your users will never see. Correction: decide cold vs
  steady-state explicitly and control cache state between runs.
- **Single-run comparisons.** One run before, one after, 7% "improvement" — well
  inside noise. Correction: multiple runs, compare medians, require the delta to
  exceed observed variance.
- **Micro-optimizing inside a complexity bug.** Shaving 20% off each iteration of
  an O(n²) loop loses to removing the nesting. Correction: complexity first,
  constants second.
- **Toy-sized test data.** Everything is fast on 100 rows. Correction: production
  scale or a justified, documented scale-down.
- **Not knowing when to stop.** Without a target, optimization continues until
  the code is unmaintainable. Correction: numeric target agreed up front; stop on
  contact.
- **Breaking correctness silently.** Reordered, cached, or parallelized code that
  returns subtly wrong answers. Correction: full functional suite after every
  kept change, plus an output-equivalence check on the optimized path.

## Escalation

- Meeting the target requires an architecture change (new datastore, service
  split, queue introduction): stop tuning and open `architecture-decisions` with
  your profile data attached — that decision is expensive to reverse.
- Target unreachable even with the largest hotspot eliminated (Amdahl ceiling <
  target): report the math to the requester; the target, the workload, or the
  architecture must change, and that's their call.
- Optimization would trade correctness or durability (weaker consistency,
  fire-and-forget writes, staleness beyond stated bounds): surface the trade-off
  to a human; never make it unilaterally.
- Measurements are irreproducible after controlling environment, warm-up, and
  data (variance still swamps the effect): escalate with your harness — shared
  infrastructure noise may need a dedicated environment you can't provision
  yourself.
