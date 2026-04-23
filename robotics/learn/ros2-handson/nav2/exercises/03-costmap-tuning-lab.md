<!-- markdownlint-disable MD033 -->

# Exercise03 — Nav2 Costmap Tuning Lab

## Companion exercises for `../04-nav2-costmaps-and-layers.md`, `../07-nav2-localization-odom-amcl-ekf.md`, and `../11-nav2-debugging-observability-and-bag-analysis.md`

**Estimated time:** 90 to 110 minutes  
**Prerequisite lessons:** ../04-nav2-costmaps-and-layers.md, ../07-nav2-localization-odom-amcl-ekf.md, ../11-nav2-debugging-observability-and-bag-analysis.md

**Mode options:**

- **Simulation:** tune a global and local costmap in a corridor map and compare behavior in RViz.
- **Bag replay:** replay sensor or TF data and inspect how the costmaps evolve over time.
- **Static analysis:** use the supplied map geometry, YAML snippets, and incident prompts to practice reasoned tuning without live hardware.

**Validation goal:** you should finish with a defensible explanation of how footprint, inflation, obstacle layers, and timing together determine whether an AMR sees the world accurately enough to navigate.

---

## Overview

Costmap tuning is where many Nav2 teams start changing values without understanding the geometry they are changing.

This lab forces you to reason from:

1. aisle width versus footprint versus inflation radius
2. global route quality versus local obstacle realism
3. stale observations versus genuinely blocked space
4. simulation convenience versus production safety margins

You can do the whole lab in simulation, or you can complete it entirely as a structured analysis exercise.

---

## Section A — Corridor Passability Math

Assume the following environment and robot:

- aisle width: `1.40 m`
- robot physical width: `0.58 m`
- recommended operational lateral clearance target: `0.20 m` per side
- map resolution: `0.05 m`

### Task A1 — Feasibility Estimate

**Questions:**

1. What is the approximate free clearance left in the aisle before inflation is applied?
2. Why can a physically passable aisle still become non-passable in the global costmap?
3. What would make the answer different for the local costmap?

<details><summary>Answer guidance</summary>

Raw geometric leftover width is roughly:

```text
1.40 - 0.58 = 0.82 m total spare width
0.41 m on each side before inflation or localization uncertainty
```

But costmaps also encode inflation, discretization, and sometimes localization or control uncertainty. Once you subtract those margins, a corridor that is physically passable can look operationally blocked to the planner.

The local costmap may differ because it often has a rolling window, more dynamic obstacle content, and different tuning priorities than the global costmap.

</details>

- [ ] Done

---

### Task A2 — Inflation Radius Sweep

For each candidate `inflation_radius`, mark whether you expect the global planner to consider the aisle comfortably passable, marginal, or effectively blocked.

| `inflation_radius` | Prediction | Why |
| --- | --- | --- |
| `0.10` m | ? | ? |
| `0.20` m | ? | ? |
| `0.35` m | ? | ? |
| `0.50` m | ? | ? |

<details><summary>Answer guidance</summary>

Your exact labels can vary, but the reasoning should show that as inflation expands wall influence inward, the planner's usable corridor shrinks. A value near or above half the remaining free half-width becomes dangerous because inflated zones overlap or leave too little comfortable track width for the planner and controller.

</details>

- [ ] Done

---

## Section B — YAML Critique and Tuning Proposal

Read this simplified configuration and propose changes.

```yaml
global_costmap:
  global_frame: map
  robot_base_frame: base_link
  resolution: 0.05
  footprint: "[[0.36, 0.29], [0.36, -0.29], [-0.36, -0.29], [-0.36, 0.29]]"
  plugins: [static_layer, obstacle_layer, inflation_layer]
  inflation_layer:
    plugin: nav2_costmap_2d::InflationLayer
    inflation_radius: 0.50
    cost_scaling_factor: 2.0

local_costmap:
  global_frame: odom
  robot_base_frame: base_link
  rolling_window: true
  width: 3.0
  height: 3.0
  resolution: 0.05
  plugins: [obstacle_layer, inflation_layer]
  obstacle_layer:
    observation_sources: scan
    scan:
      topic: /scan
      clearing: false
      marking: true
  inflation_layer:
    plugin: nav2_costmap_2d::InflationLayer
    inflation_radius: 0.45
```

**Questions:**

1. What two values look most likely to create narrow-aisle trouble?
2. Why is `clearing: false` such a dangerous setting in a dynamic warehouse?
3. Which parameter would you tune first for the global costmap, and which one for the local costmap?

<details><summary>Answer guidance</summary>

Likely concerns:

- the inflation radii are quite large relative to the corridor geometry
- disabling clearing in the local obstacle layer means transient obstacles can linger indefinitely as ghost occupancy

A strong answer explains the separation of concerns:

- **Global costmap first:** set inflation to reflect strategic route safety without closing valid aisles
- **Local costmap first:** restore correct observation clearing and then tune inflation for short-horizon maneuvering

</details>

- [ ] Done

---

## Section C — Stale Observation Diagnosis

Use this evidence from a bag replay or thought experiment.

```text
[costmap_2d] Message Filter dropping message: frame 'laser' at time 245.200 for reason 'the timestamp on the message is earlier than all the data in the transform cache'
[local_costmap.local_costmap] Received 0 valid observations in last 3.0s
[controller_server] FollowPath: FAILURE
[controller_server] Robot footprint appears blocked in local costmap
```

**Questions:**

1. Why can the robot appear blocked even if no real obstacle is present?
2. Is the best first fix costmap inflation, obstacle clearing, or TF and timing? Explain.
3. Name one command or one visualization that would help prove your diagnosis.

<details><summary>Answer guidance</summary>

If message filtering rejects sensor data because of timestamp or TF mismatch, the local costmap can become stale or fail to clear previously marked space. The first fix is not inflation tuning; it is proving TF and timing correctness so observation ingestion becomes trustworthy again.

Useful validation examples:

- inspect TF timing with `tf2_echo`
- visualize local costmap and sensor points together in RViz
- replay the bag under simulated time and compare whether drops disappear

</details>

- [ ] Done

---

## Section D — Design a Tuning Experiment

You have 30 minutes to improve navigation through a narrow aisle in simulation without hardware changes.

**Task:** propose a three-run experiment plan.

For each run, specify:

1. what parameter or parameter group you will change
2. what metric you will measure
3. what outcome would cause you to keep or reject the change

Use at least one metric from each category:

- path feasibility or no-path rate
- route quality or clearance
- local execution stability

<details><summary>Answer guidance</summary>

The best answers use disciplined changes rather than editing five settings at once. For example:

1. reduce global inflation slightly and measure no-path rate and route clearance
2. enable or fix local clearing behavior, then measure phantom obstacle persistence
3. adjust local inflation only after observation flow is trustworthy, then measure hesitation or near-collision behavior in the aisle

</details>

- [ ] Done

---

## Section E — AMR Production Judgment

Answer in 3 to 5 sentences each.

**E1.** Why is a successful path in simulation not enough to approve costmap tuning for a production AMR?

**E2.** If operations complain that the robot is too conservative, what evidence would you demand before shrinking inflation or footprint margins?

**E3.** When should you suspect localization drift rather than costmap tuning, even if the symptom is "no path found"?

<details><summary>Answer guidance</summary>

Strong answers mention real sensor timing, localization drift, pose uncertainty, pallet overhangs, and the difference between strategic map geometry and dynamic floor behavior. A good answer also notes that shrinking safety margins to improve throughput can silently move the risk into controller tracking and docking safety.

</details>

- [ ] Done

---

## Deliverable Template

```text
Environment:
Simulation / bag replay / static analysis

Geometry assumptions:

Baseline parameters:

Observed symptoms:

Tuning changes tested:
1.
2.
3.

Best final recommendation:

Remaining risks:
```

---

## Success Criteria

You have completed this lab well if you can:

1. reason quantitatively about aisle passability instead of treating inflation as guesswork
2. separate stale-observation incidents from geometry-tuning incidents
3. explain why the local and global costmaps should not always share identical tuning priorities
4. defend a safe, evidence-backed costmap change for an AMR environment
