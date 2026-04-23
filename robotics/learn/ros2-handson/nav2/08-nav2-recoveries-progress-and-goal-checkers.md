# 08 — Nav2 Recoveries, Progress Checkers, and Goal Checkers

## How to design safe retry behavior, detect real stuck conditions, and decide when an AMR should escalate instead of looping

**Prerequisite:** 07-nav2-localization-odom-amcl-ekf.md, 06-nav2-local-control-and-cmdvel.md, 03-nav2-bt-navigator-and-bt-xml.md
**Unlocks:** Safer recovery policy, fewer opaque mission failures, cleaner retry budgets, better differentiation between temporary blockage and real navigation breakdown

---

## Why Should I Care? (Context)

When navigation fails in production, the most important question is not "can the robot try something?"
It is "what should the robot try next, how many times, and when should it stop?"

Bad recovery design creates recognizable AMR failures:

1. robot spins repeatedly in a blocked aisle while traffic backs up
2. robot gives up too early on a temporary human obstruction
3. robot keeps clearing costmaps even though localization is the real problem
4. progress checker declares the robot stuck even though the base is inching forward correctly
5. goal checker reports success at the wrong pose and corrupts downstream task logic

Recoveries and checkers are where technical navigation meets operational policy.

---

## PART 1 — RECOVERIES ARE POLICY, NOT CLEANUP NOISE

---

## 1.1 What Recoveries Actually Mean

A recovery is a deliberate policy decision that says:

```text
the normal plan-follow loop is not succeeding
try a bounded alternative behavior to regain progress or clarity
```

Common Nav2 recovery behaviors include:

- clearing costmaps
- spinning
- backing up
- waiting

Those are not equivalent. Each assumes a different story about why navigation failed.

---

## 1.2 Match the Recovery to the Failure Story

| Suspected failure story | Good first recovery bias | Bad first recovery bias |
| --- | --- | --- |
| temporary human or forklift obstruction | wait, maybe replan | aggressive backup or repeated spin |
| stale or phantom obstacle in costmap | clear relevant costmap, then replan | repeated wait with no model refresh |
| robot nose trapped in tight geometry | short backup, then replan | repeated clear-only loop |
| localization confusion | escalate or relocalize workflow outside default recoveries | endless spin/clear cycles |

If the recovery assumes the wrong story, the robot wastes time and operator trust.

---

## 1.3 Retry Budget Is a Product Decision

The number of retries is not just a technical knob.

More retries mean:

- better resilience to temporary blockage
- more opportunity for success without operator help
- more time spent failing inside shared traffic space

Fewer retries mean:

- faster escalation to mission or fleet logic
- fewer useless loops
- more false aborts in environments with frequent short-lived obstruction

That tradeoff depends on aisle width, human traffic, mission urgency, and operator expectations.

---

## PART 2 — PROGRESS CHECKERS: IS THE ROBOT REALLY MOVING?

---

## 2.1 What a Progress Checker Is Trying to Prove

A progress checker asks a simple but operationally critical question:

```text
has the robot made enough real progress within enough time to justify continuing?
```

That usually boils down to thresholds such as:

- minimum movement radius
- allowed time without sufficient movement

Those values sound simple, but they are heavily dependent on the robot and its motion stack.

---

## 2.2 False Stuck Detection Is Common

A progress checker can fire even when the controller is behaving rationally.

Typical causes:

- velocity smoother reduces motion too much
- base deadband prevents tiny commands from producing measurable progress
- robot is intentionally inching during docking or narrow-aisle alignment
- localization noise hides real low-speed movement

If the checker says "stuck," prove whether the robot was actually unable to move or only unable to satisfy the chosen threshold.

---

## 2.3 Progress Thresholds Must Match Motion Mode

The same thresholds rarely fit all scenarios.

| Scenario | Risk if threshold too strict | Risk if threshold too loose |
| --- | --- | --- |
| normal aisle following | false stuck detection | slow recognition of real failures |
| dense human environment | constant needless recoveries | robot waits too long in congestion |
| docking or final alignment | aborts during valid inching motion | masks real low-speed deadlock |

This is why high-quality AMR systems often treat docking and general navigation differently at the mission layer.

---

## PART 3 — GOAL CHECKERS: WHEN ARE YOU REALLY "THERE"?

---

## 3.1 Goal Reached Is a Contract, Not a Feeling

A goal checker determines whether the robot has satisfied positional and angular tolerances strongly enough to report success.

This matters because success triggers downstream actions:

- task completion
- manipulator handoff
- docking continuation
- fleet scheduling updates

If success is declared too early, the robot may be operationally in the wrong place even though Nav2 says done.

---

## 3.2 Position and Yaw Tolerances Are Operational Choices

Loose tolerances can help throughput in coarse navigation tasks.

Tight tolerances matter for:

- docking
- pickup/dropoff staging
- narrow station approach
- handoff to another subsystem that assumes precise pose

Bad pattern:

1. team sees goal oscillation
2. they loosen tolerances to make the issue disappear
3. docking or staging later becomes unreliable

That is not fixing the root cause. It is shifting the failure downstream.

---

## 3.3 Stateful Goal Checking Can Be Useful and Dangerous

Some goal-checking behavior uses state to avoid flapping once tolerances are satisfied.

That can help stability, but it can also hide situations where the robot briefly enters tolerance and then drifts away.

Use it intentionally, especially if a downstream workflow assumes precise final pose.

---

## PART 4 — DEFAULT RECOVERIES AND WHEN THEY MAKE SENSE

---

## 4.1 Clear Costmap

This makes sense when the world model may be stale or polluted.

Good use cases:

- phantom obstacle from observation lag
- old obstacle trail after a pallet moved
- local costmap not matching visible reality

Bad use cases:

- persistent real obstacle
- wrong global localization
- base cannot execute commands

If costmap clearing works repeatedly, do not celebrate. Ask why the stale obstacle keeps returning.

---

## 4.2 Wait

Waiting is underused in human-heavy or forklift-heavy environments.

It is often the safest first move when:

- obstruction is temporary
- backing up creates more conflict
- spinning expands risk envelope near shelves or pedestrians

Waiting is bad when the robot is geometrically trapped or the world model itself is wrong.

---

## 4.3 Back Up

Backing up is useful when the robot needs space to replan or re-enter a corridor.

It is risky when:

- rear perception is weak
- shared-traffic policy forbids blind retreat
- the robot is near a station or handoff zone

In production AMRs, backup distance and conditions are policy decisions, not just defaults.

---

## 4.4 Spin

Spin can help with sensor coverage and local environment refresh.

It can also be a throughput killer in narrow aisles.

If the robot repeatedly spins in a place where turning radius is operationally awkward, the policy likely belongs at the BT or mission layer, not in more retries.

---

## PART 5 — RECOVERY DESIGN FOR COMMON AMR INCIDENTS

---

## 5.1 Blocked Aisle by Temporary Traffic

Recommended bias:

1. wait briefly
2. re-evaluate and maybe replan
3. escalate if blocked beyond policy budget

Why:

- spinning and backing often worsen shared-space behavior
- repeated aggressive recoveries reduce throughput for everyone

---

## 5.2 Phantom Obstacle in Local Costmap

Recommended bias:

1. clear local costmap
2. retry local follow
3. if repeated, investigate sensor or observation-source health

Do not let costmap clear become a permanent band-aid for perception integration defects.

---

## 5.3 Nose Trapped Near Shelf Corner

Recommended bias:

1. short backup
2. replan
3. retry with bounded count

This is the kind of incident where backup makes more sense than waiting because geometry, not traffic, is the main issue.

---

## 5.4 Docking or Staging Failure

Recommended bias:

1. avoid generic repeated recoveries
2. decide whether the issue is approach geometry, localization precision, or station-specific logic
3. escalate to a docking-specific routine or mission layer if needed

Default recoveries are often too generic for precise station work.

---

## PART 6 — CHECKER TUNING WITHOUT LYING TO YOURSELF

---

## 6.1 Do Not Use Goal Tolerance to Hide Localization Noise

If the robot cannot settle because pose estimate is noisy, loosening the goal checker may only move the failure to docking, manipulation, or task completion.

Treat loose tolerances as a task-level decision, not a universal fix.

---

## 6.2 Do Not Use Progress Thresholds to Hide Base Problems

If the base ignores small commands, increasing movement time allowance may reduce false aborts but it does not repair the command-chain mismatch.

First prove that commanded low-speed motion is physically executable.

---

## 6.3 Validate Checkers on Three Separate Cases

Always validate on:

1. normal aisle motion
2. temporary obstruction
3. final approach or docking-like low-speed motion

If one threshold set only works in one case, document the limitation instead of pretending it is universal.

---

## PART 7 — ESCALATION TO THE MISSION OR FLEET LAYER

---

## 7.1 Not Every Failure Should Be Solved Inside Nav2

Escalate when:

- blockage persists beyond local retry budget
- another robot or human policy needs to change the route
- docking requires a product-specific decision tree
- operator assistance is required

Nav2 should not carry all mission semantics alone.

---

## 7.2 Good Escalation Signals

Useful signals to expose upward:

- reason for abort or failure type
- number of retries consumed
- whether progress checker or goal checker was the trigger
- whether recoveries changed the world model or only retried movement

This is what lets the mission layer make informed next-step decisions.

---

## 7.3 Senior Interview Version

Strong answers explain that recoveries are:

- bounded
- scenario-specific
- safe for the environment
- observable from logs and metrics
- connected to mission escalation rather than endless local looping

That is what distinguishes production AMR policy from demo navigation.

---

## Next Lesson

Continue to 09-nav2-waypoints-docking-zones-and-missions.md.
That lesson explains how waypoint flows, docking, costmap filters, and higher-level mission logic sit above these local retry and success contracts.
