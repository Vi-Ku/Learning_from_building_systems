<!-- markdownlint-disable MD033 -->

# Exercise02 — Nav2 BT Trace and Recovery Loop

## Companion exercises for `../03-nav2-bt-navigator-and-bt-xml.md`, `../08-nav2-recoveries-progress-and-goal-checkers.md`, and `../11-nav2-debugging-observability-and-bag-analysis.md`

**Estimated time:** 80 to 95 minutes  
**Prerequisite lessons:** ../02-nav2-bringup-lifecycle-actions.md, ../03-nav2-bt-navigator-and-bt-xml.md, ../08-nav2-recoveries-progress-and-goal-checkers.md, ../11-nav2-debugging-observability-and-bag-analysis.md

**Mode options:**

- **Simulation:** run a blocked-goal or obstacle scenario and watch BT status in logs or Groot-style tooling.
- **Log analysis:** use the supplied BT traces and reason about the policy without running a robot.
- **Field RCA:** substitute a real recovery-loop incident and classify each retry stage using the same structure.

**Validation goal:** you should be able to read a BT trace and decide whether the real problem is planning, local execution, stale world modeling, or a bad recovery policy.

---

## Overview

BehaviorTree traces are where senior Nav2 debugging starts to look different from guesswork.

Junior debugging often sounds like this:

1. the robot spun, so spinning must be broken
2. the planner failed, so the planner plugin must be wrong
3. recovery ran too many times, so reduce retries

Senior debugging asks different questions:

1. what node in the tree failed first?
2. what assumption did the recovery subtree make about the failure story?
3. was replanning helpful, neutral, or actively wasting aisle time?
4. when should the BT have escalated instead of looping?

This lab trains that lens.

---

## Section A — Tick-by-Tick Trace Reading

For each scenario, name the first failing contract and recommend the most defensible next investigation.

### Scenario A1 — Infinite Replan, Zero Progress

```text
[bt_navigator] Tick
[bt_navigator]   ComputePathToPose: SUCCESS
[bt_navigator]   FollowPath: RUNNING
[bt_navigator] Tick
[bt_navigator]   FollowPath: FAILURE
[bt_navigator]   GoalUpdated: FAILURE
[bt_navigator]   ClearEntireCostmap: SUCCESS
[bt_navigator]   Spin: SUCCESS
[bt_navigator] Tick
[bt_navigator]   ComputePathToPose: SUCCESS
[bt_navigator]   FollowPath: FAILURE
[controller_server] Progress checker failed: no movement in 10.0s
```

**Questions:**

1. Which contract failed first: planning, control, progress checking, or recovery logic?
2. Why is costmap clearing probably not the right primary fix?
3. What two pieces of evidence would distinguish controller tuning from base deadband or downstream velocity clipping?

<details><summary>Answer guidance</summary>

The planner is succeeding, so the first failure is on the local execution side. The trace points at controller or command-path inability to make progress. Costmap clearing is being used as a generic recovery, but nothing in the evidence says stale obstacles are the main story.

The decisive evidence pair is usually:

- controller output versus final base command topic
- robot pose or odometry change during the same interval

If the controller publishes meaningful commands that never become motion, the problem is likely downstream. If commands themselves are indecisive or oscillatory, controller tuning becomes more likely.

</details>

- [ ] Done

---

### Scenario A2 — Planner Fails, Recovery Works Once, Then Repeats

```text
[bt_navigator]   ComputePathToPose: FAILURE
[planner_server] No valid path found
[bt_navigator]   ClearEntireCostmap: SUCCESS
[bt_navigator]   ComputePathToPose: SUCCESS
[bt_navigator]   FollowPath: RUNNING
[bt_navigator]   FollowPath: FAILURE
[costmap_2d] Observation buffer dropped 18 messages due to stale transforms
[bt_navigator]   ComputePathToPose: FAILURE
```

**Questions:**

1. Why is "planner issue" an incomplete diagnosis here?
2. What changed between the first and second planning attempts?
3. What upstream contract looks weak enough to corrupt both planning and execution over time?

<details><summary>Answer guidance</summary>

This is not just a planner story because the planner briefly succeeds after a recovery, then execution later collapses again. That pattern suggests the world model is unstable rather than the planner algorithm being intrinsically incapable.

The important clue is stale transform handling inside costmap observation processing. TF timing or localization freshness can poison obstacle marking, which then causes path validity to oscillate over time.

</details>

- [ ] Done

---

## Section B — Recovery Policy Critique

Read each policy proposal and decide whether it matches the failure story.

### Proposal B1 — Blocked Aisle With Human Traffic

```xml
<ReactiveFallback name="RecoveryFallback">
  <GoalUpdated/>
  <SequenceWithMemory name="RecoveryActions">
    <Spin spin_dist="3.14"/>
    <BackUp backup_dist="0.50" backup_speed="0.10"/>
    <Wait wait_duration="1.0"/>
  </SequenceWithMemory>
</ReactiveFallback>
```

**Questions:**

1. What assumption is this policy making about the cause of failure?
2. Why might it be poor for a narrow warehouse aisle with pedestrians or forklifts?
3. Reorder the recoveries into a safer first-pass policy and justify the first two steps.

<details><summary>Answer guidance</summary>

This tree assumes the robot is locally trapped and should physically maneuver first. In a human-heavy aisle that can be the wrong story. Spinning enlarges the risk envelope, backing up may worsen traffic conflicts, and a 1-second wait is usually too short to discriminate temporary blockage from a real deadlock.

A stronger first-pass policy is often:

1. short wait
2. replan or clear only the relevant costmap if stale perception is plausible
3. controlled backup only if geometry suggests the robot is nose-trapped
4. spin later, not first

</details>

- [ ] Done

---

### Proposal B2 — Phantom Obstacles Near a Dock

```xml
<SequenceWithMemory name="RecoveryActions">
  <Wait wait_duration="10.0"/>
  <Wait wait_duration="10.0"/>
  <Wait wait_duration="10.0"/>
</SequenceWithMemory>
```

**Questions:**

1. Why is waiting alone a weak response to suspected stale obstacle data?
2. What recovery action would better test the hypothesis that the world model is polluted?
3. What evidence would tell you the real problem is localization rather than costmap dirtiness?

<details><summary>Answer guidance</summary>

If the world model is wrong, waiting does not actively refresh it. A costmap clear or targeted observation-debug step is more aligned with the failure story. If the robot or dock pose is globally wrong, however, repeated clears will not help; you would instead see consistent mismatch between RViz or log-reported pose and the physical staging geometry.

</details>

- [ ] Done

---

## Section C — Build an Incident Timeline

Use the evidence below to write a five-step incident timeline.

```text
12:00:00.010  Goal accepted by NavigateToPose
12:00:00.030  ComputePathToPose SUCCESS, path length 14.2m
12:00:02.400  FollowPath RUNNING
12:00:13.100  FollowPath FAILURE
12:00:13.120  progress_checker: robot pose changed only 0.04m in 10.0s
12:00:13.140  Recovery: ClearLocalCostmap SUCCESS
12:00:13.500  ComputePathToPose SUCCESS, path length 14.0m
12:00:23.600  FollowPath FAILURE
12:00:23.620  progress_checker: robot pose changed only 0.03m in 10.0s
12:00:23.700  Recovery: BackUp SUCCESS
12:00:24.900  ComputePathToPose SUCCESS
12:00:35.100  FollowPath FAILURE
12:00:35.200  Recovery budget exhausted, aborting
```

**Tasks:**

1. Write the earliest plausible root-cause statement.
2. List two alternate hypotheses that still fit the evidence.
3. State which one observation would most efficiently discriminate between them.

<details><summary>Answer guidance</summary>

The earliest plausible root cause is failure to make local progress despite valid global paths. Alternate hypotheses include:

- controller tuning or overly strict progress checker
- downstream `cmd_vel` clipping or base deadband
- local costmap showing a hidden or stale obstacle that makes the controller refuse motion

The best discriminating observation is often a synchronized view of controller command output, final base command, and odometry delta over the same window.

</details>

- [ ] Done

---

## Section D — Design a Better Escalation Rule

You are tuning Nav2 for an AMR that shares aisles with people and pallet traffic. A blocked aisle can be normal for 15 to 30 seconds, but spinning repeatedly in place is operationally unacceptable.

**Questions:**

1. After how many failed local progress cycles should the BT escalate to mission or fleet logic rather than keep retrying internally?
2. What operator-visible event should be emitted when that escalation happens?
3. What information should be attached to that event so another system can make a better decision?

<details><summary>Answer guidance</summary>

There is no single magic number, but a strong answer ties the retry budget to aisle blocking norms, safety, and throughput. A good escalation event should differentiate between "temporarily blocked", "stuck due to local execution", and "global planning unavailable". The payload should include current pose, goal pose, last successful path age, recovery history, and a concise failure classification.

</details>

- [ ] Done

---

## Deliverable Template

```text
Scenario type:
Simulation / logs / production incident

First failing contract:

Evidence for that claim:
- trace lines:
- supporting logs:
- missing evidence:

Best next action:

Recovery or escalation recommendation:
```

---

## Success Criteria

You have completed this lab well if you can:

1. read a BT trace without confusing symptoms and causes
2. explain when replanning is helpful and when it is just churn around a deeper local problem
3. critique recovery ordering using real AMR operating constraints rather than abstract elegance
4. define when the BT should stop retrying and hand control back to a broader mission system
