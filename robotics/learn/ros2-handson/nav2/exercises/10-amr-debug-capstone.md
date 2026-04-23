<!-- markdownlint-disable MD033 -->

# Exercise10 — AMR Debug Capstone

## Companion exercises for `../07-nav2-localization-odom-amcl-ekf.md`, `../08-nav2-recoveries-progress-and-goal-checkers.md`, `../09-nav2-waypoints-docking-zones-and-missions.md`, `../10-nav2-parameters-launch-and-plugin-extension.md`, `../11-nav2-debugging-observability-and-bag-analysis.md`, and `../12-nav2-amr-failure-patterns-and-capstone.md`

**Estimated time:** 120 to 150 minutes  
**Prerequisite lessons:** ../07-nav2-localization-odom-amcl-ekf.md, ../08-nav2-recoveries-progress-and-goal-checkers.md, ../09-nav2-waypoints-docking-zones-and-missions.md, ../10-nav2-parameters-launch-and-plugin-extension.md, ../11-nav2-debugging-observability-and-bag-analysis.md, ../12-nav2-amr-failure-patterns-and-capstone.md

**Mode options:**

- **Simulation:** reproduce parts of the incident in a Nav2-enabled warehouse world and compare observed symptoms to the evidence packet.
- **Bag replay and logs:** treat the supplied evidence as the primary source of truth and produce a structured root-cause analysis.
- **Design review:** complete the capstone as a production incident exercise with no hardware dependency.

**Validation goal:** by the end of this capstone you should be able to separate localization, costmap, controller, recovery-policy, zone-policy, and mission-layer mistakes in one coherent incident review instead of blaming "navigation" as a single black box.

---

## Scenario

You are the on-call engineer for a warehouse AMR fleet.

At 09:14, robot `amr-07` was executing this mission:

1. leave outbound staging
2. travel through packing zone corridor
3. stop at pick station `P-12`
4. continue to charger after pick confirmation

Operations report:

- the robot slowed appropriately near the packing zone
- then hesitated, backed up once, turned sharply, and aborted
- the dashboard labeled the incident `navigation_failed`
- the mission system immediately reassigned the order to another robot

Your job is to produce a defensible diagnosis from incomplete evidence.

---

## Evidence Packet

```text
[mission_manager] Dispatch mission pick_P12_then_charge to amr-07
[zone_manager] Entered speed zone packing_corridor, max_speed=0.25
[bt_navigator] Begin NavigateToPose to P-12 approach waypoint
[controller_server] Selected command: linear.x=0.18 angular.z=0.22
[velocity_smoother] Output command: linear.x=0.10 angular.z=0.08
[local_costmap.local_costmap] Message Filter dropping message: frame 'laser' at time 551.420 for reason 'the timestamp on the message is earlier than all the data in the transform cache'
[controller_server] No valid control command found
[behavior_server] Executing backup behavior
[amcl] Pose covariance increased to x=0.36 y=0.40 yaw=0.31
[bt_navigator] Recovery node returned SUCCESS
[planner_server] ComputePathToPose failed: no valid path found
[mission_manager] Navigation result=FAILED for waypoint P-12 approach
[mission_manager] Auto-diverting robot to charge mission
```

Supplemental notes:

- RViz screenshot from the incident shows the robot slightly rotated relative to the mapped corridor.
- A replay clip suggests intermittent human traffic was present in the packing corridor.
- The forklift keepout zone was not violated.
- The battery was at `38%`, well above the normal charge-divert threshold.

---

## Section A — First-Pass Triage

Answer in order, without skipping ahead to fixes.

1. What are the first three competing hypotheses you would consider?
2. Which single log line most strongly changes your investigation priority?
3. Why is `navigation_failed` an insufficient incident label here?

<details><summary>Answer guidance</summary>

Reasonable first hypotheses include:

- localization or TF timing degradation causing observation drops and controller invalidation
- a genuine transient corridor obstruction interacting with the recovery policy
- mission-layer misclassification or mis-escalation after a bounded Nav2 failure

The dropped-message line and covariance increase should strongly influence prioritization because they suggest state trust issues, not just route inconvenience.

</details>

- [ ] Done

---

## Section B — Evidence Classification Table

Classify each evidence item.

| Evidence item | Best category | Why it matters |
| --- | --- | --- |
| speed zone entry | ? | ? |
| smoothed command lower than controller intent | ? | ? |
| local costmap dropped laser message | ? | ? |
| backup behavior success | ? | ? |
| planner later reports no valid path | ? | ? |
| mission auto-diverts to charger at 38% battery | ? | ? |

Possible categories: spatial policy, command-chain behavior, TF or timing issue, recovery outcome, path feasibility symptom, mission-layer policy error.

<details><summary>Answer guidance</summary>

This table should help you avoid flattening the entire incident into one cause. Several entries are likely downstream effects or policy context rather than root cause.

</details>

- [ ] Done

---

## Section C — Root Cause Analysis

### Task C1 — Build the Causal Chain

Write a causal chain using this structure:

```text
Trigger or degraded condition -> immediate Nav2 symptom -> recovery behavior -> downstream planning symptom -> mission-layer decision
```

Then answer:

1. Which part of the chain currently has the strongest evidence?
2. Which part is still only a hypothesis?
3. What extra evidence would most reduce uncertainty?

<details><summary>Answer guidance</summary>

High-quality answers usually treat TF or localization degradation as an upstream candidate, controller invalidation and backup as middle symptoms, and the charge diversion as a possibly separate mission-policy mistake.

</details>

- [ ] Done

---

### Task C2 — Distinguish Root Cause From Bad Outcome Policy

**Questions:**

1. Is the automatic diversion to charge mission justified by the evidence given?
2. Why is that question separate from diagnosing the original navigation issue?
3. What two policy contracts appear weak even if Nav2 was partly at fault?

<details><summary>Answer guidance</summary>

The battery note suggests the diversion policy may be unjustified. That is separate because mission policy can still be wrong even when a real navigation issue occurred. Common weak contracts here are failure classification and post-failure mission escalation rules.

</details>

- [ ] Done

---

## Section D — Focused Validation Plan

You have 45 minutes and no hardware access.

Design a validation plan with exactly four steps. It must include:

1. one TF or timing check
2. one localization-confidence check
3. one command-path check
4. one mission-policy review step

For each step, specify:

- what evidence you inspect
- what result would support your leading hypothesis
- what result would falsify it

<details><summary>Answer guidance</summary>

The goal is disciplined narrowing. Avoid broad fishing expeditions. Each step should be able to disconfirm part of your current story.

</details>

- [ ] Done

---

## Section E — Operator and Dashboard Redesign

The current operator ticket only says `navigation_failed`.

### Task E1 — Replace It With a Better Incident Summary

Write a replacement incident summary of 5 to 7 lines that separates:

- possible localization or TF health issue
- bounded recovery execution
- downstream path failure
- questionable mission escalation

Then answer:

1. What labels should the dashboard have emitted instead?
2. Which two timeline events should be highlighted for review by default?

<details><summary>Answer guidance</summary>

Strong answers create a layered narrative such as: `packing_zone_hold_or_state_degradation`, `recovery_executed`, `planner_no_path_after_recovery`, `mission_policy_diverted_to_charge`. The point is truthful decomposition.

</details>

- [ ] Done

---

## Section F — Corrective Action Proposal

Propose one corrective action in each category below.

| Category | Corrective action | Why |
| --- | --- | --- |
| observability | ? | ? |
| localization or TF robustness | ? | ? |
| recovery policy | ? | ? |
| mission-layer failure handling | ? | ? |

<details><summary>Answer guidance</summary>

The best answers avoid magical one-line fixes. They improve evidence quality, harden the likely weak contract, and reduce the chance of the mission system making a second bad decision after the first fault.

</details>

- [ ] Done

---

## Section G — Final RCA Writeup

Write a concise root-cause analysis using this template.

```text
Incident summary:

Most likely upstream fault:

Evidence supporting it:
-
-
-

Secondary contributing factors:
-
-

Uncertain or missing evidence:
-

Why the dashboard label was misleading:

Recommended next validations:
1.
2.
3.
```

<details><summary>Answer guidance</summary>

Strong writeups show hierarchy: upstream fault, downstream Nav2 symptoms, and mission-policy mistakes are separated rather than blended.

</details>

- [ ] Done

---

## Section H — Senior Reflection

Answer briefly but concretely.

**H1.** Why do AMR incidents become expensive when teams use `navigation_failed` as a catch-all label?

**H2.** What would you teach a new on-call engineer to check first in incidents like this?

**H3.** If you could add one mandatory artifact to every Nav2 incident package, what would it be?

<details><summary>Answer guidance</summary>

Strong answers mention time-to-diagnosis, bad escalation decisions, and the value of one compact artifact that joins TF health, localization confidence, and mission context.

</details>

- [ ] Done

---

## Deliverable Template

```text
Incident ID:

Leading hypothesis:

Evidence table:
- localization or TF:
- command chain:
- recovery sequence:
- planner symptom:
- mission policy:

Causal chain:

Most likely root cause:

Secondary contributing factors:

Corrective actions:
1.
2.
3.
4.

Recommended dashboard labels:

Open questions:
```

---

## Success Criteria

You have completed this capstone well if you can:

1. decompose one AMR incident across localization, command path, recovery behavior, planner outcome, and mission policy
2. state which evidence is strongest and which conclusions remain provisional
3. design a focused validation plan that could falsify your current diagnosis without hardware access
4. write an incident summary that operations, robotics engineers, and mission-software engineers can all act on
