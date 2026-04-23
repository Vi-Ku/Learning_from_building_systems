<!-- markdownlint-disable MD033 -->

# Exercise04 — Nav2 Planner Comparison Lab

## Companion exercises for `../05-nav2-global-planning.md`, `../04-nav2-costmaps-and-layers.md`, and `../12-nav2-amr-failure-patterns-and-capstone.md`

**Estimated time:** 85 to 105 minutes  
**Prerequisite lessons:** ../04-nav2-costmaps-and-layers.md, ../05-nav2-global-planning.md, ../12-nav2-amr-failure-patterns-and-capstone.md

**Mode options:**

- **Simulation:** compare planner outputs on the same map and goals using two or more planner plugins.
- **Offline analysis:** inspect path screenshots, metrics, or log excerpts if you cannot run the planners directly.
- **Design review:** treat this as a planner-selection memo for a real AMR product team.

**Validation goal:** finish with a planner recommendation that is grounded in environment constraints, controller followability, and operational risk rather than personal preference.

---

## Overview

This lab exists because many planner discussions collapse into cargo-cult answers like "use the more advanced planner" or "just switch to Smac Hybrid-A*".

A production planner decision should instead weigh:

1. whether the global costmap is trustworthy enough for planner comparisons to mean anything
2. whether the vehicle actually benefits from stronger kinematic realism
3. whether the returned path is easier for the controller to follow in the real environment
4. whether computation and replanning behavior fit your AMR throughput requirements

---

## Section A — Comparison Setup

Assume a differential-drive AMR in a warehouse map with these properties:

- narrow aisles: `1.2 m` to `1.6 m`
- frequent long straight segments
- some 90-degree end-of-aisle turns
- occasional blocked aisle requiring reroute
- moderate CPU budget on the robot computer

You are comparing three planner candidates:

- `NavFn`
- `Smac 2D`
- `Smac Hybrid-A*`

### Task A1 — Hypothesis Table

Fill in the table before looking at any results.

| Planner | What you expect it to do well | What might go wrong |
| --- | --- | --- |
| `NavFn` | ? | ? |
| `Smac 2D` | ? | ? |
| `Smac Hybrid-A*` | ? | ? |

<details><summary>Answer guidance</summary>

Strong answers usually predict:

- `NavFn`: simple baseline behavior and low conceptual overhead, but possibly less path realism
- `Smac 2D`: strong modern grid planning for structured maps, but still dependent on good costmaps
- `Smac Hybrid-A*`: better orientation-aware route realism, but more compute and potentially more tuning burden than the platform actually needs

</details>

- [ ] Done

---

## Section B — Compare Results

Use your own runs or this sample result set.

| Planner | Path length | Min obstacle clearance | Mean planning latency | Controller followability note |
| --- | --- | --- | --- | --- |
| `NavFn` | `18.4 m` | `0.19 m` | `18 ms` | sharp corner entry at aisle exit |
| `Smac 2D` | `18.9 m` | `0.27 m` | `31 ms` | smoother centerline behavior |
| `Smac Hybrid-A*` | `19.8 m` | `0.29 m` | `78 ms` | most realistic turning shape, but longer route |

### Task B1 — Interpret the Tradeoff

**Questions:**

1. Which planner appears best if you optimize only for planning latency?
2. Which planner appears best if you optimize for controller followability in tight aisles?
3. Why might the longest path still be the operationally best path?

<details><summary>Answer guidance</summary>

Latency alone favors `NavFn`. Followability and clearance lean toward `Smac 2D` or `Smac Hybrid-A*`. The longest path can still be best if it reduces oscillation, recoveries, or near-obstacle behavior enough to improve end-to-end mission reliability.

</details>

- [ ] Done

---

### Task B2 — Root-Cause Discipline

Suppose `NavFn` often returns no path in one specific aisle, while `Smac 2D` usually succeeds.

**Questions:**

1. Why is "`NavFn` is bad" still an incomplete conclusion?
2. What costmap or map-quality checks would you run before recommending a planner swap?
3. What evidence would make a planner change defensible anyway?

<details><summary>Answer guidance</summary>

A planner difference may be real, but you still need to prove that the world model is stable and legal. Costmap inflation, unknown-space policy, stale obstacles, or map discretization can all masquerade as planner weakness. A planner change becomes defensible when repeated evidence shows that, under the same trustworthy world model, one planner consistently produces safer or more trackable paths for the robot's operating geometry.

</details>

- [ ] Done

---

## Section C — Planner Selection Memo

Write a short recommendation for this product scenario:

```text
Robot: differential-drive warehouse AMR
Requirement: high throughput, low recovery frequency, no need for car-like reverse maneuvers
Constraint: compute headroom is limited because perception and fleet comms share the same computer
```

Your memo must include:

1. selected planner
2. one rejected alternative and why
3. the biggest residual risk even after your choice
4. one metric you would monitor after rollout

<details><summary>Answer guidance</summary>

Many learners will land on `Smac 2D` as the best balance here, but a strong memo can defend another option if the reasoning is coherent. The important part is explicit tradeoffs: computational budget, route realism, controller compatibility, and failure rate in narrow aisles.

</details>

- [ ] Done

---

## Section D — Replanning Under Change

Now assume a forklift blocks the nominal route halfway through execution.

**Questions:**

1. What planner behavior matters more now than initial path optimality?
2. Why is replanning performance a system property, not only a planner property?
3. What other Nav2 surfaces could make a good planner look bad during this event?

<details><summary>Answer guidance</summary>

The key behavior becomes resilient replanning under an updated costmap. This is not just the planner's job: costmap freshness, TF timing, BT replan cadence, and controller ability to accept updated paths all matter. A planner can look bad if the world model is stale or if the local controller fails on a good reroute.

</details>

- [ ] Done

---

## Section E — AMR Failure Pattern Reflection

Answer each in 3 to 4 sentences.

**E1.** When does a planner with more realistic kinematics become unnecessary complexity for an AMR team?

**E2.** Why should planner evaluation include controller followability instead of judging only the path shape in RViz?

**E3.** If operations report that the robot "chooses weird routes," what evidence would help you decide whether the planner or the costmap representation is the real problem?

<details><summary>Answer guidance</summary>

Strong answers mention compute budget, actual vehicle constraints, and the danger of solving the wrong layer. A path that looks strange in RViz may still be rational under the encoded cost field. That means costmap semantics and inflation must be audited before blaming the planner alone.

</details>

- [ ] Done

---

## Deliverable Template

```text
Environment or dataset:

Planners compared:

Metrics recorded:
- path length:
- clearance:
- latency:
- followability:

Recommendation:

Why not the alternatives:

Post-rollout watch metric:
```

---

## Success Criteria

You have completed this lab well if you can:

1. compare planners without collapsing costmap and controller issues into planner preference
2. justify planner choice for a specific AMR and environment instead of giving a generic answer
3. explain why end-to-end mission quality can favor a path that is not shortest in pure geometry
4. write a planner recommendation memo that a senior robotics team could actually debate
