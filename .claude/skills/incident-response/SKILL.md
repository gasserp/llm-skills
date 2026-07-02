---
name: incident-response
description: Restores service when production is down or degraded right now, then drives learning afterward. Use when users are impacted at this moment — outage, error-rate spike, latency spike, data pipeline stalled — when someone says "production is down", "we're getting paged", "rollback?", or when deciding between mitigating and debugging live. For non-urgent bugs, use systematic-debugging instead.
---

# Incident Response

## Purpose

Production is down or degraded, users are impacted, and the clock is running. The
standard: restore service by the fastest safe mitigation before diagnosing root
cause, keep a timestamped record as you act, communicate on a fixed cadence, and
convert the incident into prevention through a blameless postmortem with owned,
dated action items. Every minute spent understanding-while-down is a minute of
user pain you chose.

## Core workflow

1. **Confirm impact and open the timeline (first 2 minutes).** Verify the alert
   is real user impact, not a broken monitor: check the user-facing signal (error
   rate, success rate, latency dashboard) directly. Immediately start a timeline
   document — a running log of `HH:MM — observation or action` entries. Append to
   it after every step for the rest of the incident. Why now: memory will not
   survive the adrenaline, and the postmortem, the comms, and any handoff all
   depend on this record.

   Exit criterion: impact confirmed with a named metric and its current bad
   value; timeline doc exists with its first entries.

2. **Ask "what changed?" — the most recent change is guilty until proven
   innocent.** Pull the last few hours of: deploys, config changes, feature-flag
   flips, infra changes, dependency/vendor status, traffic anomalies. The
   overwhelming majority of incidents are triggered by a change someone made.
   Check the deploy log FIRST, before dashboards seduce you into analysis.

   Exit criterion: list of candidate changes with timestamps, ordered
   most-recent-first, each mapped against the failure start time.

3. **Mitigate before you diagnose.** Pick the fastest action that stops user
   pain; understanding why comes later. In rough order of preference:
   - *Rollback* the suspect deploy — fastest and safest when a recent deploy
     correlates with onset.
   - *Feature-flag off* the suspect feature — near-instant, smallest blast radius.
   - *Failover* to a healthy replica/region/instance.
   - *Scale up / shed load* (add capacity, rate-limit, serve degraded mode) when
     the cause is load, not a change.
   Announce the action in the incident channel BEFORE executing ("rolling back
   api to v141, ETA 4 min"), execute, timestamp it. Why announce first: parallel
   uncoordinated actions by multiple responders is how one incident becomes two.

   Exit criterion: exactly one mitigation in flight, announced and timestamped.

4. **Observe the effect before making a second change.** After the mitigation
   lands, watch the impact metric for one full propagation cycle (deploy time +
   cache TTL + metric lag — know this number for your system; assume 5–15 minutes
   if you don't). Do not stack a second change on top of an unobserved first one:
   overlapping changes make it impossible to attribute recovery or new breakage,
   and can compound the outage.

   Exit criterion: metric verdict recorded — improved, unchanged, or worse — with
   timestamp. If unchanged/worse: revert the mitigation if it carries risk,
   return to step 2's list, take the next candidate.

5. **Communicate on a fixed cadence.** From the first confirmation, post updates
   every 15–30 minutes (pick one interval and say it), each containing exactly:
   what is known (impact, scope, suspected trigger), what is being done, and when
   the next update comes. Post on schedule even if the update is "no change since
   last update; still rolling back" — silence reads as abandonment and generates
   the status-ping interruptions that slow the actual response. If responders
   number more than two, one person handles comms so the others can work.

   Exit criterion per update: all three elements present, next-update time stated.

6. **Verify recovery with the SAME signal that showed the failure.** Recovery is
   declared only when the metric from step 1 — the same dashboard, same query,
   same threshold — returns to normal and stays there for one full propagation
   cycle. Not a proxy: "deploy succeeded", "pods are healthy", "I loaded the page
   once" are all consistent with users still failing. Timestamp the declaration,
   post the final update, downgrade the incident.

   Exit criterion: original impact metric nominal and stable; recovery announced.

7. **Root-cause AFTER recovery, then hold a blameless postmortem.** Now — with
   users unaffected — switch to `systematic-debugging` using the timeline as your
   evidence base to find the true root cause (the rollback told you *what*
   triggered it, not *why* it broke). Within a few days, run a postmortem that
   produces: the timeline, the root cause and contributing factors, what went
   well/poorly in the response, and action items where EVERY item has a named
   owner and a due date. Blameless means naming system gaps ("no canary on this
   service") not people ("X pushed a bad change") — because blame teaches people
   to hide information, and hidden information causes the next incident. An
   action item without an owner and date is a wish; unowned wishes are how the
   same incident recurs quarterly.

   Exit criterion: postmortem written; every action item has owner + date; the
   rolled-back change is not re-landed until the root cause is fixed.

## Decision points

- **Rollback vs roll-forward:** Roll back when a recent change correlates with
  onset and rollback is available — it is the known-good state and needs no new
  code under pressure. Roll forward (fix + deploy) only when: rollback is
  impossible (irreversible migration, data written in a new format), rollback
  re-opens a worse issue (the change was itself an emergency fix), or the forward
  fix is truly trivial AND your pipeline ships it faster than a rollback. When
  in doubt, roll back — writing new code during an outage means writing it at
  your worst.
- **Irreversible migration blocking rollback:** roll back the application code
  while keeping the schema (most migrations are backward-compatible for one
  version); if not compatible, that's a roll-forward situation — and a postmortem
  action item ("expand-contract migrations") for later.
- **Partial mitigation vs waiting for the full fix:** take the degraded mode NOW
  if it stops the worst impact — serve cached/stale data, disable the broken
  feature while the rest works, rate-limit heavy endpoints. 80% service in 5
  minutes beats 100% in 2 hours; you can pursue the full fix from a position of
  reduced pain. Announce the degradation explicitly so support isn't blindsided.
- **Cause is an external vendor/provider:** you can't fix them — mitigate around
  them (failover to secondary, cached responses, graceful degradation), open a
  ticket with them, and communicate the dependency to your users. Do not spend
  the incident debugging their black box.
- **No recent change and no obvious trigger:** suspect gradual exhaustion
  (disk, memory leak, cert expiry, ID overflow, connection-pool saturation) or
  traffic shift. Check resource saturation dashboards and expiry dates before
  reading code; restarts/failover legitimately buy time here while you find the
  leak.
- **Severity triage:** total outage or data loss risk → all-hands, page whoever
  owns the suspect change, comms every 15 min. Partial degradation → smaller
  response, 30-min cadence. Data-corruption suspicion → additionally STOP writes
  or snapshot state before mitigating, because mitigation that keeps writing can
  convert a recoverable incident into permanent loss.

## Quality bar

- [ ] First mitigation attempt began before any root-cause investigation deeper
      than "what changed recently".
- [ ] Timeline has a timestamped entry for every observation, decision, and
      action, written during the incident, not reconstructed.
- [ ] Only one change in flight at any moment; each change's effect observed and
      recorded before the next began.
- [ ] Every comms update contained known/doing/next-update-time and shipped on
      the stated cadence.
- [ ] Recovery declared on the original failing metric, stable for a full
      propagation cycle — not on a proxy.
- [ ] Root-cause analysis completed post-recovery (`systematic-debugging`), and
      the reverted change stays out until the cause is fixed.
- [ ] Postmortem exists, is blameless, and every action item has a named owner
      and a due date.

## Common traps

- **Debugging root cause while users are still down.** The puzzle is more
  interesting than the rollback, so responders read code while the error rate
  holds at 40%. Correction: mitigation first is a hard rule; the mystery will
  still be there — nicely preserved in your timeline — after service is restored.
- **Stacking changes.** A second fix launched before the first one's effect is
  visible; now recovery (or worsening) is unattributable. Correction: one change,
  one observation window, one recorded verdict, then the next change.
- **"It can't be my change."** The deploy that correlates with onset gets waved
  off because it "only touched logging". Correction: recency is evidence;
  innocence is proven by rollback (or by demonstrated non-correlation), not by
  assertion.
- **Verifying recovery by proxy.** "Deploy went green" while users still 500 —
  the incident quietly continues and re-pages an hour later. Correction: the
  metric that opened the incident is the only signal that can close it.
- **Going silent while heads-down.** Stakeholders escalate, ping responders, and
  add load precisely because they hear nothing. Correction: scheduled updates are
  part of the response, not overhead; "no change" is a valid update.
- **No timeline until afterward.** The postmortem gets reconstructed from fuzzy
  memory and Slack archaeology; half the lessons are lost. Correction: append to
  the timeline in real time — one line per event costs seconds.
- **Postmortem without owners and dates.** Everyone agrees on the action items;
  nobody does them; the incident recurs. Correction: no item leaves the meeting
  without a name and a date, and someone tracks them to completion.

## Escalation

- Impact metric not improving 15 minutes after your best mitigation, or you've
  exhausted the candidate-change list: page the service owner / senior on-call
  now — the cost of waking someone is trivially smaller than extended downtime.
- Any suspicion of data corruption or data loss: stop mitigations that write,
  snapshot current state, and escalate immediately — recovery decisions with
  permanent consequences need senior sign-off.
- Any sign of a security breach (unexplained access, defacement, data
  exfiltration): switch to the security incident process and involve security
  on-call — evidence preservation now conflicts with normal restart-and-restore
  instincts.
- The mitigation itself is high-risk (region failover, restoring from backup,
  manual production data edits): get a second qualified person to review the
  exact command before execution — irreversible actions under adrenaline are how
  incidents become disasters.
- Customer-facing legal/contractual impact (SLA breach, regulatory reporting
  clock): loop in management during the incident, not after — some obligations
  have deadlines measured in hours.
