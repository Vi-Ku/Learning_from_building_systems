# 11 — Nav2 Debugging, Observability, and Bag Analysis

## How to debug AMR navigation incidents from symptoms, logs, TF, topics, and rosbag evidence instead of random parameter changes

**Prerequisite:** 10-nav2-parameters-launch-and-plugin-extension.md, 07-nav2-localization-odom-amcl-ekf.md, 03-nav2-bt-navigator-and-bt-xml.md
**Unlocks:** Faster root-cause isolation, stronger incident response habits, cleaner use of logs and bag replay, better separation between localization, costmap, planner, controller, and BT-policy failures

---

## Why Should I Care? (Context)

When a production AMR fails in the field, the worst debugging pattern is:

1. restart everything
2. change two parameters
3. re-run once
4. declare it fixed because the robot moved
5. wait for the incident to come back next shift

Nav2 incidents usually sit at the intersection of multiple surfaces:

- lifecycle and startup state
- TF freshness and frame ownership
- localization quality
- costmap observations
- planner validity
- controller execution quality
- BT retry and recovery policy
- mission-layer expectations

If you cannot observe those surfaces cleanly, you do not really know what failed.

This lesson is about building a disciplined debugging workflow for AMR navigation systems.

---

## PART 1 — OBSERVABILITY BEFORE TUNING

---

## 1.1 Most Nav2 "Tuning" Problems Are Actually Visibility Problems

Teams often say:

- the planner is bad
- the controller is oscillating
- Nav2 is flaky

But what they really mean is:

- we do not know which server failed first
- we do not know whether TF was valid at failure time
- we do not know whether the robot was localized correctly
- we do not know what costmaps believed

If you cannot answer those four questions, tuning is premature.

---

## 1.2 A Production Debugging Rule

Before changing parameters, capture evidence from at least these categories:

| Surface | Questions to answer |
| --- | --- |
| lifecycle | were the nodes actually active and healthy? |
| TF | did required transforms exist, with sane timestamps? |
| localization | was pose estimate believable enough for navigation? |
| topics and actions | was the command and feedback flow alive? |
| logs | which component emitted the first meaningful failure? |
| bag data | can the incident be replayed or inspected offline? |

This turns vague failure reports into something diagnosable.

---

## 1.3 Start With the Symptom, Not With a Favorite Tool

Real incident statements are usually symptom-level:

- robot spins near a rack and never finishes the goal
- robot says no valid path although aisle looks open
- robot reaches near the dock and then aborts
- robot keeps clearing costmaps until mission timeout

Those are starting points, not diagnoses.

Your job is to translate the symptom into a structured investigation.

---

## PART 2 — A STRUCTURED NAV2 INCIDENT TRIAGE FLOW

---

## 2.1 First Decide Whether the Failure Is Startup, Runtime, or Intermittent

These three classes demand different debugging moves.

| Failure class | Typical signs | Best first move |
| --- | --- | --- |
| startup failure | bringup never stabilizes, actions unavailable, nodes inactive | inspect lifecycle, launch, params, missing dependencies |
| runtime failure | system starts cleanly but fails during navigation | inspect logs, TF, costmaps, action feedback, controller state |
| intermittent field failure | same stack sometimes works, sometimes does not | capture bags, timestamps, environment conditions, repeatability clues |

Do not debug all three with the same mental model.

---

## 2.2 Ask: What Was the First Broken Contract?

Useful Nav2 debugging often reduces to this question:

```text
what contract failed first?
```

Possible contracts:

- transform contract: required frames unavailable or stale
- localization contract: pose estimate inconsistent with reality
- perception contract: obstacle data missing, stale, or polluted
- planning contract: no valid path under current map and constraints
- control contract: commands issued but motion not achieved
- behavior policy contract: retries or recoveries mismatch the real failure story

If you skip straight to the last visible symptom, you often miss the earliest broken contract.

---

## 2.3 Keep a Fixed Triage Order

A practical order for field incidents:

1. confirm goal and mission context
2. check lifecycle and node liveness
3. inspect TF tree and timestamps
4. validate localization against reality
5. inspect global and local costmap story
6. inspect planner outputs and controller behavior
7. inspect BT recoveries and retry loops
8. only then change configuration or code

This avoids the common trap of tuning the controller when the real issue is stale TF.

---

## PART 3 — LOGS: WHAT TO READ AND HOW TO READ THEM

---

## 3.1 Logs Should Be Read as a Timeline, Not as Isolated Errors

Engineers often grab the loudest line in the logs and stop there.

That is dangerous because later components may simply be reporting downstream fallout.

Read logs as a sequence:

1. what was the last normal event?
2. which server reported trouble first?
3. what did recovery logic do next?
4. what finally caused abort or timeout?

The first meaningful anomaly matters more than the last dramatic one.

---

## 3.2 Useful Log Categories During Nav2 Incidents

Look for evidence around:

- lifecycle transitions and activation failures
- missing transforms or TF timeout messages
- costmap sensor update or clearing anomalies
- planner no-path or invalid-path messages
- controller progress-checker or goal-checker outcomes
- BT recovery loop entries and retry exhaustion

One useful discipline is to annotate each log clue by subsystem ownership before discussing fixes.

---

## 3.3 Avoid Reading Logs Like Confirmation Bias Fuel

Bad pattern:

1. operator says robot was stuck
2. engineer assumes controller issue
3. engineer scans logs until they find one controller warning
4. localization drift evidence gets ignored

Good pattern:

1. collect all subsystem anomalies in time order
2. rank them by how upstream they are
3. pick the earliest plausible root-cause candidate

That is how you avoid cargo-cult retuning.

---

## PART 4 — TF INSPECTION: THE FASTEST WAY TO CATCH INVISIBLE BREAKAGE

---

## 4.1 TF Problems Commonly Masquerade as Planner or Controller Problems

If these transforms are wrong, stale, or inconsistently timestamped, Nav2 behavior will degrade in confusing ways:

- `map -> odom`
- `odom -> base_link`
- sensor frame relationships to `base_link`

Symptoms may look like:

- planner says no path unexpectedly
- local costmap appears shifted relative to reality
- robot oscillates because local frame alignment is bad
- recoveries repeat because the robot state estimate is incoherent

---

## 4.2 What to Inspect in TF

During an incident, inspect:

1. whether required frames exist at all
2. whether timestamps are fresh enough for current processing
3. whether transforms jump, reset, or drift unexpectedly
4. whether frame ownership is consistent with your localization design

Useful questions:

- is AMCL or EKF publishing the transform you think it is?
- did simulation and hardware use different frame conventions?
- did namespacing or remapping point Nav2 at the wrong tree?

---

## 4.3 Typical TF Failure Patterns in AMRs

| Symptom | Likely TF issue |
| --- | --- |
| robot footprint looks offset in RViz | wrong static transform or base frame assumption |
| costmap obstacles appear late or displaced | sensor frame transform stale or incorrect |
| localization appears to jump after startup | competing publishers or reset behavior |
| robot moves physically but Nav2 thinks progress is poor | odom or localization inconsistency |

Treat TF as first-class production telemetry, not background infrastructure.

---

## PART 5 — TOPIC AND ACTION INSPECTION

---

## 5.1 Topic Liveness Does Not Mean Semantic Health

A topic existing is not enough.

You still need to ask:

- is data arriving at the expected rate?
- are timestamps sane?
- is the frame ID correct?
- does the data tell a believable physical story?

This matters especially for:

- odometry
- localization pose outputs
- laser or depth observations
- costmap publications
- velocity commands
- action feedback and status

---

## 5.2 Action Debugging Is Mission Debugging Too

For navigation actions, inspect:

- whether goals are accepted cleanly
- whether feedback progresses in a believable way
- whether cancel or preemption events happened
- whether the result failure code matches the observed behavior

In AMR systems, navigation failures can be misreported if the mission layer times out or cancels first.

Do not assume Nav2 owned the final abort just because the robot stopped moving.

---

## 5.3 `cmd_vel` Inspection Must Be Interpreted Carefully

Seeing velocity commands tells you only that Nav2 attempted control output.

It does not prove:

- the base executed them correctly
- the commands survived downstream safety gating
- deadband or saturation did not flatten them
- localization reflected the resulting motion accurately

Always compare command intent with actual motion evidence.

---

## PART 6 — ROSBAG AS THE SOURCE OF TRUTH

---

## 6.1 Why Bags Matter

Field failures are often impossible to reason about from memory or screenshots.

Bags let you inspect:

- exact timestamps
- message order
- transform availability
- action feedback over time
- localization drift and recovery loops

Without a bag, teams often end up debugging stories rather than evidence.

---

## 6.2 What to Capture for Nav2 Incidents

A good incident bag should usually include:

- TF and TF static
- odometry and localization outputs
- scan or perception topics feeding costmaps
- global and local costmap publications if feasible
- action goal, feedback, and result topics
- `cmd_vel` and any post-safety gated velocity topic
- diagnostics or mission state topics that explain context

Capture enough context to reconstruct the failure story, not just the last 10 seconds of motion.

---

## 6.3 Bag Review Questions

When reviewing a bag, answer in order:

1. what goal was active?
2. where did the robot believe it was?
3. what did the maps and costmaps believe about the environment?
4. did the planner have a valid route?
5. what commands did the controller produce?
6. what recovery policy was triggered?
7. what evidence supports the claimed root cause?

If you cannot answer those from the bag, the capture was incomplete.

---

## 6.4 Bag Replay Is Not Reality, But It Is Still Powerful

Bag replay helps isolate software-side behavior, but it has limits:

- actuator faults may not reproduce
- safety PLC gating may not be modeled
- real traffic and moving humans are not recreated automatically
- timing on overloaded production hardware may differ from replay hosts

Use replay to narrow hypotheses, then confirm in the right environment.

---

## PART 7 — COSTMAP AND RVIZ INSPECTION

---

## 7.1 Costmaps Explain Many "Planner" Incidents

If the planner says no path, inspect whether the path was truly impossible under the current costmap state.

Common realities:

- inflated corridor became too narrow for the configured footprint
- stale obstacle blocked the only aisle
- unknown-space policy prevented routing
- local costmap made valid motion look unsafe near the robot

Many planner incidents are really world-model incidents.

---

## 7.2 RViz Is a Reasoning Tool, Not Just a Pretty Dashboard

Use RViz to compare:

- robot pose vs reality
- footprint vs aisle geometry
- planned path vs local obstacles
- sensor observations vs costmap occupancy
- goal location vs operationally correct staging point

If the visual story is inconsistent, stop tuning and explain the inconsistency first.

---

## 7.3 Snapshot the Evidence, Not Just the Screen

For recurring incidents, retain:

- RViz screenshots with frame overlays visible
- bag timestamp range
- log timestamp range
- mission identifier and goal details
- environmental notes such as blocked aisle, pallet, or dock occupancy

That turns a one-off troubleshooting session into reusable operational knowledge.

---

## PART 8 — A PRACTICAL NAV2 INCIDENT PLAYBOOK

---

## 8.1 Minimal Debugging Checklist

Use this checklist before anyone proposes a fix:

1. record incident time and mission context
2. save logs for the time window
3. confirm lifecycle state of Nav2 nodes
4. inspect TF health and frame freshness
5. verify localization against map reality
6. inspect costmap and planner story
7. inspect controller output and real motion
8. review recovery loop behavior
9. extract bag evidence for offline analysis
10. state the root-cause hypothesis and the evidence that supports it

This checklist is deliberately boring. That is why it works.

---

## 8.2 Map Common Symptoms to First Checks

| Symptom | Best first checks |
| --- | --- |
| robot immediately rejects goal | lifecycle, action server availability, frame mismatch |
| robot says no path in open map | costmap occupancy, footprint, unknown-space policy, TF |
| robot plans but barely moves | controller outputs, progress checker, base execution, odom |
| robot nears goal then oscillates or aborts | localization precision, goal checker, controller tuning, docking semantics |
| repeated recoveries with no progress | wrong failure story, stale costmap, localization issue, blocked aisle policy |

This is where debugging becomes operationally efficient.

---

## 8.3 Write the Incident Summary Like an Engineer, Not Like a Witness

Bad summary:

```text
Nav2 failed near the dock and seemed confused.
```

Better summary:

```text
Robot accepted NavigateToPose at 14:03:12.
Global planning succeeded, but local costmap retained a stale obstacle at dock approach.
Controller produced bounded angular commands with poor linear progress.
Progress checker triggered three recoveries and abort followed at 14:03:46.
TF remained valid. Localization error stayed within expected tolerance.
Primary root cause hypothesis: stale local obstacle persistence near dock staging zone.
```

That level of precision changes the quality of the next discussion.

---

## PART 9 — CASE STUDIES

---

## 9.1 Incident: "Planner Is Broken"

Observed symptom:

- robot reports no valid path in a visually open aisle

Structured findings:

1. lifecycle healthy
2. TF healthy
3. localization reasonable
4. global costmap shows inflated blockage caused by an old obstacle source
5. planner correctly refuses the path under that map state

Root cause class:

- perception or costmap observability problem, not planner algorithm failure

---

## 9.2 Incident: "Controller Is Oscillating"

Observed symptom:

- robot wiggles near a final pose and eventually aborts

Structured findings:

1. goal is effectively a docking-like alignment problem
2. localization noise near the station is worse than assumed
3. goal checker tolerance and final-approach expectations conflict
4. generic navigation control is being asked to finish a docking workflow it does not own

Root cause class:

- ownership and operational contract problem, not just controller tuning

---

## 9.3 Incident: "Nav2 Randomly Fails on One Robot"

Observed symptom:

- same site works on two AMRs but one repeatedly gets stuck

Structured findings:

1. parameters appear shared, but one robot has different drivetrain deadband
2. `cmd_vel` shows small commands that do not translate into real motion
3. odometry reports weak progress and progress checker fires

Root cause class:

- robot-specific execution and tuning mismatch hidden inside supposedly shared configuration

---

## PART 10 — WHAT GOOD LOOKS LIKE

---

## 10.1 Mature Teams Build Navigation Debugging Into Operations

Strong teams do not wait for a severe incident to think about observability.

They already have:

- known-good bag capture procedures
- subsystem-oriented logging conventions
- RViz layouts for fast inspection
- TF health checks in bringup validation
- post-incident templates that separate symptom, evidence, hypothesis, and fix

That is operational maturity, not extra polish.

---

## 10.2 Final Mental Model

Nav2 debugging is not about finding the one magical line in the logs.

It is about reconstructing a chain:

```text
goal intent
    -> lifecycle readiness
    -> TF and localization validity
    -> world model correctness
    -> planning outcome
    -> control execution
    -> recovery policy
    -> mission consequence
```

When you can walk that chain without hand-waving, you can own AMR navigation incidents under pressure.

---

## Quick Recap

- debug in a fixed order before tuning
- use logs as a timeline, not as isolated error fragments
- treat TF as a first-class production dependency
- inspect action, topic, and `cmd_vel` data semantically, not just for liveness
- use bags to reconstruct evidence and replay hypotheses offline
- separate root cause from downstream fallout

---

## Next Lesson

Continue to 12-nav2-amr-failure-patterns-and-capstone.md
