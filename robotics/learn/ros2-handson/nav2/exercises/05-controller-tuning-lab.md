<!-- markdownlint-disable MD033 -->

# Exercise05 — Nav2 Controller Tuning Lab

## Companion exercises for `../06-nav2-local-control-and-cmdvel.md`, `../08-nav2-recoveries-progress-and-goal-checkers.md`, and `../11-nav2-debugging-observability-and-bag-analysis.md`

**Estimated time:** 95 to 120 minutes  
**Prerequisite lessons:** ../05-nav2-global-planning.md, ../06-nav2-local-control-and-cmdvel.md, ../08-nav2-recoveries-progress-and-goal-checkers.md, ../11-nav2-debugging-observability-and-bag-analysis.md

**Mode options:**

- **Simulation:** tune DWB or Regulated Pure Pursuit in a path-following scenario with turns, aisle entry, and final approach.
- **Bag replay and logs:** inspect command topics, odometry, and local costmap behavior without moving hardware.
- **Static analysis:** use the supplied metrics and symptoms to recommend controller and checker changes.

**Validation goal:** by the end of this lab you should be able to tell whether a navigation problem is caused by controller tuning, goal or progress checking, or downstream command handling.

---

## Overview

Controller tuning is where Nav2 teams most often confuse causes:

1. a poor global path gets blamed on DWB or RPP
2. downstream velocity clipping gets blamed on the controller plugin
3. progress checker thresholds get tuned to hide real execution problems
4. goal tolerances are loosened to mask oscillation near docking or staging points

This lab is designed to stop that behavior by forcing you to compare controller intent with actual robot motion.

---

## Section A — Symptom Classification

For each symptom, state the most likely layer to inspect first.

| Symptom | First inspection target | Why |
| --- | --- | --- |
| robot oscillates 0.3 m from the goal | ? | ? |
| robot publishes smooth controller commands but base motion is jerky | ? | ? |
| robot cuts corners too tightly near shelving | ? | ? |
| progress checker aborts during slow docking alignment | ? | ? |

<details><summary>Answer guidance</summary>

Typical mappings:

- near-goal oscillation: controller tuning plus goal checker tolerance and final approach behavior
- smooth controller commands but jerky base motion: velocity smoother, safety layer, or base control path
- corner cutting: controller lookahead or critic balance, but also inspect global path geometry
- false stuck detection during docking: progress checker thresholds may not match the intended low-speed motion mode

</details>

- [ ] Done

---

## Section B — Compare Controller Candidates

Assume you tested `DWB` and `RegulatedPurePursuit` on the same map and path set.

| Metric | `DWB` | `RegulatedPurePursuit` |
| --- | --- | --- |
| average aisle speed | `0.52 m/s` | `0.58 m/s` |
| corner overshoot incidents | `4 / 20 runs` | `2 / 20 runs` |
| near-goal oscillation incidents | `5 / 20 runs` | `3 / 20 runs` |
| docking approach smoothness (subjective) | medium | high |
| interpretability of tuning knobs for the team | medium | high |

### Task B1 — Recommendation Under Constraints

The robot is a differential-drive warehouse AMR that spends most of its time following structured paths and occasionally docking precisely.

**Questions:**

1. Which controller is the better default candidate from this evidence alone?
2. What evidence is still missing before rollout?
3. Why would a team still keep the other controller available?

<details><summary>Answer guidance</summary>

Many learners will prefer Regulated Pure Pursuit from these numbers, but the important part is to justify the decision based on motion quality, operator trust, and tuning maintainability. Missing evidence usually includes final command-chain verification, behavior under dynamic obstacles, and tolerance performance near precise staging points.

</details>

- [ ] Done

---

## Section C — Goal and Progress Checker Tuning

Read the configuration fragment below.

```yaml
controller_server:
  ros__parameters:
    progress_checker_plugin: progress_checker
    goal_checker_plugins: [general_goal_checker]

    progress_checker:
      plugin: nav2_controller::SimpleProgressChecker
      required_movement_radius: 0.25
      movement_time_allowance: 8.0

    general_goal_checker:
      plugin: nav2_controller::SimpleGoalChecker
      xy_goal_tolerance: 0.05
      yaw_goal_tolerance: 0.05
      stateful: true
```

**Questions:**

1. Which values look most likely to create false failure near docking or final alignment?
2. Why can tightening tolerances improve apparent precision while hurting overall mission success?
3. Suggest one safer initial tuning change for the goal checker and one for the progress checker.

<details><summary>Answer guidance</summary>

The likely pressure points are the tight `xy_goal_tolerance`, tight `yaw_goal_tolerance`, and potentially aggressive progress threshold relative to slow alignment motion. A strong answer explains that tolerances must match what the robot can repeatedly achieve, not what looks ideal on paper. Good initial changes are modest, measured, and tied to observable outcomes.

</details>

- [ ] Done

---

## Section D — Command-Path Verification

Use the following evidence.

```text
[controller_server] Selected command: linear.x=0.22 angular.z=0.35
[velocity_smoother] Output command: linear.x=0.08 angular.z=0.10
[base_controller] Command below angular deadband, zeroing angular.z
[odometry] robot yaw changed 0.01 rad over 2.0 s
```

**Questions:**

1. Why would tuning DWB critics or RPP lookahead first be a poor move here?
2. Which component is currently the strongest suspect?
3. What change or experiment would you run before touching controller parameters?

<details><summary>Answer guidance</summary>

The controller is already selecting a meaningful turn command. That command is being heavily attenuated downstream and then zeroed by the base controller. The local controller may still need tuning eventually, but the fastest next step is to validate and adjust the command chain: smoother limits, deadband settings, and base acceptance thresholds.

</details>

- [ ] Done

---

## Section E — Design a Focused Tuning Loop

You have time for only three tuning iterations.

**Task:** define a disciplined loop with one change per iteration.

Each iteration must include:

1. one parameter or parameter group to change
2. one metric from logs or bags
3. one success threshold
4. one reason you would revert the change

Your loop must include at least one check for:

- controller intent versus final base command
- near-goal behavior
- false stuck or progress-check failures

<details><summary>Answer guidance</summary>

High-quality answers stage the work sensibly:

1. first validate the command chain
2. then adjust controller behavior for path tracking
3. only then refine goal or progress checking for final approach behavior

The core lesson is not to tune five knobs at once.

</details>

- [ ] Done

---

## Section F — AMR Production Reflection

Answer briefly but concretely.

**F1.** Why is loosening goal tolerances a dangerous way to "fix" near-goal oscillation for docking workflows?

**F2.** When would you decide that the controller plugin is not the main issue, even if `FollowPath` keeps failing?

**F3.** What three signals would you add to a dashboard so operators can tell the difference between "controller indecision" and "command chain suppression"?

<details><summary>Answer guidance</summary>

Good answers usually mention downstream process quality: docking pose contracts, velocity smoothing, safety clipping, base deadbands, and odometry response. The dashboard signals should include controller output, final command after smoothing or safety filters, and actual measured motion.

</details>

- [ ] Done

---

## Deliverable Template

```text
Controller tested:

Environment:

Symptoms:

Evidence captured:
- controller output:
- final command output:
- odometry response:
- goal/progress checker events:

Tuning iterations:
1.
2.
3.

Best recommendation:

Open risks:
```

---

## Success Criteria

You have completed this lab well if you can:

1. classify motion problems by layer instead of blaming the controller by default
2. compare controller plugins using end-to-end motion quality rather than only RViz aesthetics
3. tune goal and progress checkers without hiding real system problems
4. prove whether a `FollowPath` failure comes from controller logic or from the downstream command path
