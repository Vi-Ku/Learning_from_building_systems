# 05 — Nav2 Global Planning

## Choosing, interpreting, and debugging global planners for warehouse and factory AMRs

**Prerequisite:** 04-nav2-costmaps-and-layers.md, 01-nav2-system-architecture.md
**Unlocks:** Better planner selection, cleaner root-cause separation, stronger tuning decisions, more credible AMR architecture reviews

---

## Why Should I Care? (Context)

Global planning is where the robot turns "I need to get there" into an actual route through the mapped world.

When this layer goes wrong, teams often make one of three mistakes:

1. they swap planner plugins before proving the costmap is trustworthy
2. they blame the planner for controller-tracking problems
3. they choose a planner based on popularity rather than vehicle and environment constraints

For production AMRs, planner choice is not academic. It affects:

- throughput
- path smoothness
- compute budget
- behavior in narrow aisles
- recovery frequency
- how well the route respects the robot's real kinematics

---

## PART 1 — WHAT THE GLOBAL PLANNER IS RESPONSIBLE FOR

---

## 1.1 The Planner's Job

The global planner computes a route from current pose to goal pose over the **global costmap**.

It answers:

```text
Given this map, this start pose, this goal pose, and this configuration,
what path should the robot try to follow?
```

It does **not** answer:

- can the controller track this perfectly right now?
- is there a moving forklift about to enter the aisle?
- should the mission layer have sent this goal in the first place?

Those belong elsewhere.

---

## 1.2 A Valid Plan Is Not the Same as a Good Plan

Two planners may both return valid paths, but one may be operationally better because it is:

- easier for the controller to track
- safer near obstacles
- more kinematically realistic
- less likely to trigger recoveries in tight spaces

This distinction matters in AMRs because a formally legal path can still be throughput-poor or fragile.

---

## PART 2 — COMMON NAV2 GLOBAL PLANNER OPTIONS

---

## 2.1 `NavFn`

`NavFn` is the classic grid-based planner familiar from earlier ROS navigation stacks.

Strengths:

- simple mental model
- established behavior
- often good enough for basic differential-drive navigation in structured maps

Limitations:

- less expressive for kinematically constrained vehicles
- may produce paths that are legal on the grid but not ideal for smooth execution

Use it when the environment is structured, the robot is simple, and you want predictable baseline behavior before moving to more advanced planners.

---

## 2.2 `Smac 2D`

`Smac 2D` is a more modern grid-based search option.

Typical advantages:

- strong performance for many structured map cases
- often better behavior and tuning flexibility than a naive legacy baseline
- good option for differential-drive or omni robots where full nonholonomic search is unnecessary

Typical consideration:

- still relies heavily on costmap quality and discretized world representation

For many warehouse AMRs, this is a strong default candidate when you want better global planning behavior without committing to a more complex kinematic model.

---

## 2.3 `Smac Hybrid-A*`

This planner incorporates vehicle orientation and motion constraints more explicitly.

Why teams choose it:

- better alignment with nonholonomic behavior
- more realistic paths for robots that cannot simply translate arbitrarily through tight geometry

Tradeoffs:

- more computational complexity
- parameter sensitivity can be higher
- you should only choose it if the vehicle and operating geometry benefit from that extra realism

This is often compelling for robots where heading and turning constraints matter materially at route-generation time.

---

## 2.4 Smac Lattice or Other Kinodynamic-Aware Options

If the platform and environment require stronger motion primitive reasoning, lattice-based or similar approaches can make sense.

This is most relevant when:

- the AMR has tighter nonholonomic limits
- the robot frequently navigates constrained approach geometries
- the global plan itself must already respect maneuver style, not just endpoint feasibility

These planners are more powerful, but the burden of good modeling and tuning is higher.

---

## PART 3 — PLANNER SELECTION FOR AMRS

---

## 3.1 Differential-Drive Warehouse AMR

Common priorities:

- narrow aisle passability
- predictable compute cost
- paths that controllers can follow smoothly
- enough conservatism near shelving and pallet corners

Reasonable choices often start with:

- `Smac 2D` for a modern grid-based default
- `NavFn` for simpler baseline validation
- `Smac Hybrid-A*` when heading and maneuver realism materially improve field behavior

The right answer depends on whether route realism or computational simplicity is more limiting in your system.

---

## 3.2 Factory Floor with More Open Space

If the environment has broader corridors and fewer tight geometry constraints, simpler planners can often perform very well.

Here the bigger value may come from:

- stable costmaps
- correct semantic zones
- controller tuning

rather than from complex planner machinery.

---

## 3.3 Planner Selection Matrix

| Environment / robot characteristic | Planner bias |
| --- | --- |
| simple differential-drive, structured map, modest compute | `NavFn` or `Smac 2D` |
| narrow aisles, strong need for robust modern grid planning | `Smac 2D` |
| heading-constrained maneuvers matter at route level | `Smac Hybrid-A*` |
| motion primitives must closely reflect vehicle behavior | lattice-style approach |

This is only a starting point. Costmap quality and controller behavior still determine whether the planner's advantages show up in production.

---

## PART 4 — WHAT THE PLANNER IS ACTUALLY OPTIMIZING

---

## 4.1 The Planner Is Cost-Map-Aware, Not Human-Common-Sense-Aware

The planner does not know what a pallet, forklift, or shipping lane means. It only knows:

- cell costs
- obstacle constraints
- search strategy
- configured penalties and heuristics

That means a path that looks strange to a human may still be the optimal solution under the current cost representation.

If the route is weird, ask whether the planner is wrong or whether the world model made that route rational.

---

## 4.2 Path Quality Metrics That Matter

When comparing planners, look beyond "did it return a path?"

Useful metrics:

- route length
- clearance from obstacles
- heading continuity
- number of sharp turns
- controller trackability
- compute latency under repeated replanning
- stability when costmap changes slightly

The best planner for an AMR is often the one that reduces downstream controller drama, not merely the one with the prettiest path in RViz.

---

## 4.3 Unknown Space and Goal Tolerance Effects

Planner behavior changes materially when:

- unknown cells are treated differently
- goal lies near inflated or unknown space
- the start pose is slightly misregistered

This is why planner failures often disappear when you fix costmap policy or localization, even though no planner parameter changed.

---

## PART 5 — WHEN A PLANNER FAILURE IS NOT REALLY A PLANNER FAILURE

---

## 5.1 "No Path Found"

Possible true causes:

- goal inside obstacle or inflated obstacle
- stale or phantom obstacles in the global costmap
- inflation and footprint closing narrow corridors
- localization placing start pose incorrectly
- unknown-space policy rejecting the route

The planner is only the component that reports the impossibility. It may not be the source of it.

---

## 5.2 Path Looks Fine, Controller Still Fails

Likely causes:

- path too close to obstacles for local execution margin
- path curvature or heading transitions are poor for the controller/vehicle pair
- local costmap conditions changed after planning
- base motion limits make the path unrealistic in practice

This is where planner/controller coupling matters. A planner is only useful if the resulting path is operationally followable.

---

## 5.3 Replanning Forever

If the tree keeps replanning but no progress is made, possibilities include:

- dynamic blockage in the environment
- costmap flicker near a constrained route
- planner returns minor route variations that do not change execution outcome
- controller cannot realize any of the available plans

This is a full-stack diagnosis problem, not just a planner-tuning problem.

---

## PART 6 — WAREHOUSE-SPECIFIC TRADEOFFS

---

## 6.1 Throughput vs Conservatism

In a dense warehouse, conservative paths with large clearance may be safer but reduce throughput by widening turns and lengthening routes.

Aggressive planners may shorten travel time but increase:

- controller difficulty
- near-obstacle sensitivity
- recovery frequency

Planner selection should reflect product goals, not just lab benchmarks.

---

## 6.2 Narrow Aisle Behavior

What matters most here:

- whether the planner believes the aisle is passable at all
- whether returned paths keep enough margin for local tracking
- whether repeated replanning stays stable when humans or forklifts temporarily intrude near aisle edges

Many narrow-aisle failures are resolved by fixing footprint/inflation policy before changing planner algorithms.

---

## 6.3 Goal Poses Near Workcells and Docks

A planner can only work with the goal it is given.

If the mission layer sends a final pose that is geometrically legal but operationally wrong, you get brittle behavior:

- awkward final orientation
- controller oscillation near the endpoint
- repeated recoveries despite a valid global route

This is why staging poses, docking entry poses, and final approach contracts matter.

---

## PART 7 — HOW TO EVALUATE A PLANNER CHANGE

---

## 7.1 Test Questions

When changing planners or planner parameters, ask:

- Did path success rate improve on representative maps?
- Did controller tracking improve or worsen?
- Did compute latency remain acceptable under replanning load?
- Did narrow-aisle behavior improve without hiding a costmap defect?
- Did recovery frequency drop in real scenarios, not just synthetic demos?

---

## 7.2 Review Checklist

- Is the planner choice matched to vehicle kinematics and aisle geometry?
- Have costmap and footprint issues been ruled out first?
- Are you optimizing for actual AMR outcomes rather than algorithm novelty?
- Can the controller reliably execute the class of paths this planner returns?
- Does the chosen planner remain stable under the environment's expected dynamic disturbance?

---

## 7.3 What You Should Be Able to Explain After This Lesson

You should now be able to explain:

1. what the global planner is and is not responsible for
2. the practical differences between common Nav2 planner choices
3. how to choose a planner for a production AMR rather than a toy demo
4. why many planner failures are rooted upstream in costmaps or localization
5. how to evaluate planner quality using operational metrics instead of just path existence

---

## 7.4 Next Step

The next planned lesson in this track is `06-nav2-local-control-and-cmdvel.md`, which will connect planner output to actual motion quality, controller tuning, and `/cmd_vel` behavior.
