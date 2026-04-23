<!-- markdownlint-disable MD033 -->

# Exercise01 — Nav2 Bringup and Lifecycle Lab

## Companion exercises for `../01-nav2-system-architecture.md` and `../02-nav2-bringup-lifecycle-actions.md`

**Estimated time:** 75 to 90 minutes  
**Prerequisite lessons:** ../01-nav2-system-architecture.md, ../02-nav2-bringup-lifecycle-actions.md, ../../03-nav2-architecture.md

**Mode options:**

- **Simulation:** run Nav2 in Gazebo or another simulator and inspect lifecycle state transitions live.
- **Bag and log analysis:** if you cannot run a robot or simulator, treat the provided command outputs and log snippets as your incident evidence and work the lab as an RCA drill.
- **Production shadowing:** if you already have AMR logs from your workplace, substitute them where the lab asks for evidence capture.

**Validation goal:** by the end of this lab you should be able to prove whether Nav2 is merely started, properly configured, or actually ready to accept `NavigateToPose` goals without hand-waving.

---

## Overview

This lab is about the most common false positive in Nav2 operations: "the stack is up" when the stack is only half up.

In production AMRs, lifecycle mistakes create expensive confusion:

1. planner and controller processes exist, but one server never reaches `ACTIVE`
2. mission code sends a goal before `bt_navigator` can coordinate the rest of the stack
3. a node configures successfully but cannot do useful work because TF, map, or sensor dependencies are still missing
4. an operator restarts random nodes instead of proving which lifecycle contract failed first

The exercises below force you to use state evidence, action availability, and startup order to diagnose bringup correctly.

---

## Section A — Lifecycle State Mapping

For each prompt, write the answer before expanding the guidance.

**A1.** Explain the operational difference between these statements:

1. "The process is running"
2. "The lifecycle node is `INACTIVE`"
3. "The lifecycle node is `ACTIVE`"
4. "The navigation system is healthy"

<details><summary>Answer guidance</summary>

Your answer should separate four different claims:

- **Process running:** the executable exists in the OS and shows up in `ros2 node list`, but this says nothing about whether publishers, action servers, costmaps, or TF dependencies are usable.
- **`INACTIVE`:** the node completed configuration and allocated resources, but it is not yet performing full runtime work. For Nav2 this often means the server exists but is not actually servicing normal requests.
- **`ACTIVE`:** the node is ready for normal runtime behavior, such as accepting actions and publishing operational outputs.
- **System healthy:** every required server reached `ACTIVE`, their dependencies are valid, and cross-node contracts such as TF, map, costmaps, and action availability all work together.

The key point: lifecycle state is necessary but not sufficient. A green lifecycle table can still hide stale TF, empty costmaps, or broken localization.

</details>

- [ ] Done

---

**A2.** Fill in the table below for the main Nav2 bringup sequence.

| Component | Why it must be ready before downstream nodes depend on it | Symptom if missing or late |
| --- | --- | --- |
| map server or localization | ? | ? |
| global and local costmaps | ? | ? |
| planner server | ? | ? |
| controller server | ? | ? |
| bt navigator | ? | ? |
| lifecycle manager | ? | ? |

<details><summary>Answer guidance</summary>

A strong answer should mention that the lifecycle manager sequences transitions, costmaps depend on map and sensor or TF correctness, planner depends on a usable global costmap, controller depends on a usable local costmap and path-following dependencies, and `bt_navigator` depends on the lower servers being alive because it only orchestrates them.

Typical failure wording:

- missing map or localization: planner cannot produce valid routes
- missing costmap health: planner or controller reach `ACTIVE` but behave uselessly
- missing planner or controller: `NavigateToPose` goal may be accepted by the top-level action path but abort later when the BT cannot complete the required subtree
- missing lifecycle manager: activation order is manual or inconsistent, producing half-started systems

</details>

- [ ] Done

---

## Section B — Hands-On Bringup Drill

Use either a live Nav2 simulation or the evidence blocks below.

### Task B1 — Prove Bringup Health With a Minimal Evidence Packet

Collect or reason about these five artifacts:

1. `ros2 lifecycle nodes`
2. lifecycle state of planner, controller, and bt navigator
3. whether `navigate_to_pose` action is discoverable
4. one TF sanity check for `map -> odom -> base_link`
5. one log line that proves activation completed in order

If running live, use commands like:

```bash
ros2 lifecycle nodes
ros2 lifecycle get /planner_server
ros2 lifecycle get /controller_server
ros2 lifecycle get /bt_navigator
ros2 action list | grep navigate
ros2 run tf2_ros tf2_echo map base_link
```

If you cannot run Nav2, use this evidence packet instead:

```text
$ ros2 lifecycle get /planner_server
Node /planner_server has current state: active [3]

$ ros2 lifecycle get /controller_server
Node /controller_server has current state: active [3]

$ ros2 lifecycle get /bt_navigator
Node /bt_navigator has current state: inactive [2]

$ ros2 action list
/compute_path_to_pose
/follow_path

[lifecycle_manager_navigation] Activating planner_server
[lifecycle_manager_navigation] Activating controller_server
[lifecycle_manager_navigation] Timed out waiting for bt_navigator bond
```

**Questions:**

1. Is this stack healthy enough for mission software to send a navigation goal?
2. Which single piece of evidence is the fastest disqualifier?
3. What would you check next before blaming launch files broadly?

<details><summary>Answer guidance</summary>

1. No. The top-level stack is not ready because `bt_navigator` is still `INACTIVE` and the action list does not show `/navigate_to_pose`.
2. The fastest disqualifier is the missing top-level navigation action or the inactive `bt_navigator`. Either one proves the stack cannot yet service a normal navigation request.
3. Check why `bt_navigator` failed to activate: missing bond, missing dependency, parameter or plugin load failure, or a downstream server the navigator depends on not being fully usable.

</details>

- [ ] Done

---

### Task B2 — Startup Failure Triage

Read the following startup log and write a three-step triage plan.

```text
[lifecycle_manager_navigation] Configuring planner_server
[planner_server] Created global costmap
[planner_server] Activating
[controller_server] Activating
[controller_server] Unable to get transform from base_link to map
[controller_server] Timed out waiting for transform after 1.00s
[lifecycle_manager_navigation] Failed to bring up node: controller_server
[bt_navigator] Waiting on external lifecycle transitions to activate
```

**Questions:**

1. Which contract failed first: lifecycle, TF, costmap, planner, or controller?
2. Why is `bt_navigator` not the root cause even though it is still waiting?
3. Name one likely fix in each category: launch/config, TF/localization, and operator workflow.

<details><summary>Answer guidance</summary>

The first broken contract is TF availability for the controller. The lifecycle manager is doing its job; it attempted ordered activation and stopped when the controller could not satisfy a prerequisite. `bt_navigator` is downstream fallout because it should not activate before the required lower servers are healthy.

Possible fixes:

- **Launch/config:** ensure localization or static transform publishers are started before controller activation.
- **TF/localization:** verify `map`, `odom`, and `base_link` ownership and timestamps; ensure AMCL or EKF is actually publishing the required transforms.
- **Operator workflow:** never send test goals until transform readiness has been proven with a TF query.

</details>

- [ ] Done

---

## Section C — Action Contract Check

This section connects lifecycle readiness to the action surface your mission layer depends on.

### Task C1 — Goal Acceptance vs Bringup Health

Suppose a custom mission node retries `NavigateToPose` every 500 ms until a goal is accepted.

**Questions:**

1. Why is this retry pattern dangerous during bringup?
2. What preflight checks should the mission node perform before sending the first goal?
3. What status should the mission layer expose to operators instead of repeatedly saying "navigation failed"?

<details><summary>Answer guidance</summary>

Good answers should mention:

- repeated goals hide startup problems and create noisy logs
- goal rejection during bringup is not the same class of problem as runtime navigation failure
- the mission layer should gate on action availability, lifecycle health, and optionally localization readiness before issuing goals
- operator-facing status should be something like "navigation stack not ready" or "bringup incomplete" rather than a misleading execution failure label

</details>

- [ ] Done

---

### Task C2 — Write a Startup Readiness Checklist

Write a six-line checklist that an on-call engineer can use before saying Nav2 is ready for a shift.

Your checklist must include:

- lifecycle state
- action surface
- TF correctness
- map or localization availability
- command chain or controller health
- one evidence artifact to archive

<details><summary>Example answer structure</summary>

An acceptable checklist looks like:

1. planner, controller, behavior server, and bt navigator all report `ACTIVE`
2. `/navigate_to_pose` and supporting actions appear in `ros2 action list`
3. `map -> odom -> base_link` exists with sane timestamps
4. localization or map server output matches expected environment state
5. controller server and local costmap show no immediate startup warnings
6. store one bringup log excerpt or screenshot for the incident record

</details>

- [ ] Done

---

## Section D — AMR Production Reflection

Answer briefly but concretely.

**D1.** In a warehouse AMR, why is an incomplete startup more dangerous than an obvious crash?

**D2.** Which signals would you surface on an operator dashboard so that "half-started Nav2" is visible within 10 seconds?

**D3.** If you could automate only one bringup gate in CI or in a pre-shift validation script, what would it be and why?

<details><summary>Answer guidance</summary>

Good answers usually mention that obvious crashes trigger intervention quickly, while half-started systems waste time and can create unsafe or misleading mission behavior. Operator signals should expose lifecycle state, top-level action availability, TF health, and recent startup warnings. A strong automation choice is a readiness probe that validates lifecycle state plus action availability plus one TF check, because that catches the most expensive false positives early.

</details>

- [ ] Done

---

## Deliverable Template

Use this structure for your lab write-up.

```text
Bringup environment:
Simulation / bag replay / log-only / production shadow

Evidence captured:
- lifecycle states:
- actions visible:
- TF check:
- startup logs:

Root cause summary:

Immediate remediation:

Prevention for future bringup:
```

---

## Success Criteria

You have completed this lab well if you can:

1. reject the statement "the stack is up" unless you have lifecycle and action evidence
2. identify whether a startup incident is really a TF, lifecycle, or dependency-order problem
3. explain why `bt_navigator` often reports the symptom while another server holds the root cause
4. write an operator-facing readiness check that distinguishes startup health from runtime failure
