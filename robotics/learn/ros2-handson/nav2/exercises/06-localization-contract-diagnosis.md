<!-- markdownlint-disable MD033 -->

# Exercise06 — Nav2 Localization Contract Diagnosis

## Companion exercises for `../07-nav2-localization-odom-amcl-ekf.md`, `../11-nav2-debugging-observability-and-bag-analysis.md`, and `../12-nav2-amr-failure-patterns-and-capstone.md`

**Estimated time:** 95 to 120 minutes  
**Prerequisite lessons:** ../04-nav2-costmaps-and-layers.md, ../07-nav2-localization-odom-amcl-ekf.md, ../11-nav2-debugging-observability-and-bag-analysis.md, ../12-nav2-amr-failure-patterns-and-capstone.md

**Mode options:**

- **Simulation:** run Nav2 with AMCL or fused odometry in a mapped environment and inspect transform quality during navigation.
- **Bag replay:** replay TF, odometry, AMCL, IMU, and scan data to diagnose localization failure without moving hardware.
- **Incident review:** use the supplied logs, topic snapshots, and parameter fragments as an RCA exercise suitable for production design review.

**Validation goal:** by the end of this lab you should be able to prove whether navigation symptoms come from the localization contract itself, from downstream Nav2 consumers of that contract, or from operator assumptions that skipped the evidence.

---

## Overview

Localization failures are expensive because they rarely look like localization failures at first.

In AMR operations they often surface as:

1. controllers oscillating on an apparently valid path
2. costmaps looking inconsistent with the real aisle geometry
3. recoveries triggering even though the floor looks clear
4. operators blaming planner or controller plugins when the pose estimate is already broken

This lab treats localization as a contract with clear consumers: TF, costmaps, planners, controllers, and mission logic. Your job is to diagnose where that contract is violated and how the violation propagates.

---

## Section A — Define the Localization Contract

### Task A1 — Name the Contract Clauses

For each clause below, explain why Nav2 depends on it.

| Contract clause | Why Nav2 depends on it |
| --- | --- |
| `map -> odom` is globally meaningful | ? |
| `odom -> base_link` is smooth and continuous | ? |
| transform timestamps are current enough for costmap and controller use | ? |
| pose estimate covariance stays within operational expectations | ? |
| laser or sensor observations align with the believed pose | ? |

<details><summary>Answer guidance</summary>

Strong answers distinguish between global correctness and local smoothness:

- `map -> odom` anchors the robot in the shared map so planning and zone semantics make sense globally.
- `odom -> base_link` must stay smooth so local control is stable even while global localization is updated.
- stale timestamps break message filters and cause costmap or controller consumers to work from old geometry.
- covariance is an operational confidence signal; low-quality localization should change mission policy, not just silently continue.
- sensor alignment proves the pose estimate is consistent with the actual environment.

</details>

- [ ] Done

---

### Task A2 — Symptom-to-Clause Mapping

Map each symptom to the contract clause you would inspect first.

| Symptom | First clause to inspect | Why |
| --- | --- | --- |
| global plan passes through shelving that looks occupied in RViz | ? | ? |
| controller output alternates left and right in a straight aisle | ? | ? |
| local costmap repeatedly drops sensor observations | ? | ? |
| robot reaches staging locations smoothly but is globally offset by 0.7 m | ? | ? |

<details><summary>Answer guidance</summary>

Typical mapping:

- incorrect global path through real obstacles: suspect global pose alignment or map frame correctness first
- left-right controller indecision in a straight aisle: suspect pose jitter or frame inconsistency before changing controller critics
- dropped observations: inspect transform timing and message-filter compatibility first
- smooth local motion but globally shifted robot: suspect `map -> odom` or localization alignment, not local odometry smoothness

</details>

- [ ] Done

---

## Section B — Evidence Packet Triage

Use a live system, bag replay, or the evidence below.

```text
$ ros2 topic echo /amcl_pose --once
pose:
  pose:
    position: {x: 18.42, y: 6.31, z: 0.0}
    orientation: {z: 0.71, w: 0.70}
  covariance: [0.45, 0.0, 0.0, 0.0, 0.0, 0.0,
               0.0, 0.42, 0.0, 0.0, 0.0, 0.0,
               0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
               0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
               0.0, 0.0, 0.0, 0.0, 0.0, 0.0,
               0.0, 0.0, 0.0, 0.0, 0.0, 0.38]

$ ros2 run tf2_ros tf2_echo map base_link
At time 430.200
- Translation: [18.44, 6.26, 0.000]
- Rotation: in Quaternion [0.000, 0.000, 0.706, 0.708]
- Average rate: 3.8
- Buffer length: 0.9

[local_costmap.local_costmap] Message Filter dropping message: frame 'laser' at time 429.740 for reason 'the timestamp on the message is earlier than all the data in the transform cache'
[controller_server] No valid control command found
[bt_navigator] NavigateToPose aborted
```

**Questions:**

1. Which evidence item is the strongest sign that localization is not healthy enough for confident navigation?
2. Why is `NavigateToPose` aborting not the primary diagnosis?
3. What three checks would you run next before changing Nav2 tuning parameters?

<details><summary>Answer guidance</summary>

Good answers usually combine two signals:

- high covariance suggests weak localization confidence
- TF echo rate and buffer length are suspiciously poor for a healthy runtime consumer stack
- dropped observations indicate TF and timing misalignment, which makes costmaps and control unreliable

The action abort is downstream fallout. The next checks should focus on TF publication rate, simulated or system time alignment, and whether AMCL or EKF is publishing consistent transforms during the replay window.

</details>

- [ ] Done

---

## Section C — Drift Versus Jump Diagnosis

Read the two incident summaries and classify them.

### Incident 1

```text
At shift start the robot pose aligns with the map.
After 11 minutes of aisle driving, the robot appears 0.6 m into the rack face in RViz.
Odometry remains smooth. No sudden TF discontinuity is observed.
Global plans become increasingly unrealistic.
```

### Incident 2

```text
The robot looks aligned during bag replay.
When AMCL receives a scan burst after a turn, the map pose snaps by 0.4 m.
The controller briefly commands a strong corrective turn.
Mission logic reports a navigation failure immediately after the pose jump.
```

**Questions:**

1. Which incident looks more like drift, and which looks more like a pose jump?
2. Why do those two failure modes create different Nav2 symptoms?
3. What mitigation or monitoring differs between them?

<details><summary>Answer guidance</summary>

Incident 1 is classic drift: global correctness degrades gradually while local motion stays smooth. Incident 2 is a pose jump: the map-frame belief changes abruptly and destabilizes downstream consumers.

Mitigations differ:

- drift pushes you toward odometry quality, sensor fusion quality, and long-horizon consistency checks
- jumps push you toward AMCL tuning, outlier handling, relocalization thresholds, and mission-layer guards that pause or revalidate after large pose corrections

</details>

- [ ] Done

---

## Section D — Mission Contract Design

You are reviewing a mission node that only checks whether `/navigate_to_pose` exists before dispatching goals.

### Task D1 — Preflight Policy Review

**Questions:**

1. Why is action availability alone an unsafe preflight gate?
2. What localization-specific preflight checks would you add for a warehouse AMR?
3. What operator status should the mission layer expose when those checks fail?

<details><summary>Answer guidance</summary>

Action availability proves almost nothing about pose trustworthiness. A stronger answer includes checks such as:

- localization node healthy and current
- covariance below an agreed threshold for the mission type
- `map -> odom -> base_link` available with sane timestamps
- no recent pose jump or TF discontinuity over a short observation window

Operator-facing status should be explicit, such as `localization_not_trusted`, not a vague `navigation_failed` message.

</details>

- [ ] Done

---

### Task D2 — Recovery Escalation Choice

An operator suggests adding more recovery retries whenever localization quality drops because "sometimes it recovers itself." Decide whether each action below is appropriate.

| Proposed response | Appropriate? | Why |
| --- | --- | --- |
| retry controller recovery three more times | ? | ? |
| slow the robot and request localization revalidation | ? | ? |
| clear costmaps immediately on every covariance spike | ? | ? |
| pause mission execution and ask for a re-localization event if pose jump exceeds threshold | ? | ? |

<details><summary>Answer guidance</summary>

The important distinction is whether the action addresses a localization contract failure or only hides it.

- extra controller recoveries rarely fix bad localization on their own
- slowing and revalidating can be reasonable if the localization issue is transient and bounded
- clearing costmaps is only justified when stale obstacle state is actually implicated, not as a generic localization remedy
- pausing on large pose jumps is often the safest mission behavior in production AMRs

</details>

- [ ] Done

---

## Section E — Design a Replay-First Diagnosis Loop

Define a four-step diagnosis loop that can be run from a bag replay or log package without hardware access.

Your loop must include:

1. one TF verification step
2. one pose-confidence step
3. one downstream Nav2 consumer check
4. one decision rule for whether tuning Nav2 is allowed before fixing localization

<details><summary>Answer guidance</summary>

The strongest answers sequence the work like this:

1. prove TF availability and timing
2. prove pose estimate plausibility and confidence
3. inspect costmap or controller behavior using that pose
4. explicitly block planner or controller tuning if the localization contract is not yet trustworthy

</details>

- [ ] Done

---

## Section F — AMR Production Reflection

Answer briefly but concretely.

**F1.** Why is "the robot is roughly in the right place" not a usable localization standard for narrow-aisle AMRs?

**F2.** Which three dashboard signals would tell you within 15 seconds that Nav2 is failing because of localization quality rather than planner choice?

**F3.** If you had to automate one guardrail for every replay-based incident review, what would it be and why?

<details><summary>Answer guidance</summary>

Strong answers usually mention aisle clearance margins, speed-zone semantics, docking tolerances, and the need to separate confidence from mere availability. Dashboard signals should combine pose confidence, TF freshness, and one downstream symptom such as observation drops or controller invalid-command rate.

</details>

- [ ] Done

---

## Deliverable Template

```text
Mode used:

Localization stack under review:

Evidence captured:
- TF health:
- covariance or confidence:
- observation or costmap symptom:
- navigation symptom:

Primary diagnosis:

Downstream impacts:

Recommended fixes:
1.
2.
3.

Mission-layer guardrails:

Open risks:
```

---

## Success Criteria

You have completed this lab well if you can:

1. define the localization contract in terms Nav2 consumers actually depend on
2. separate drift, jumps, stale TF, and weak confidence instead of calling them all "bad localization"
3. show why many apparent controller or planner failures are only downstream effects of pose problems
4. design mission and operator guardrails that react to localization quality with evidence rather than guesswork
