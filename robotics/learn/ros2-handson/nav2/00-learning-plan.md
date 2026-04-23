# Nav2 for AMRs — Mastery Plan

## For

Engineer who already understands basic ROS 2 nodes, TF2, and the high-level Nav2 flow, but wants deep operational ownership of navigation on a real AMR.

## Goal

Understand Nav2 end to end: architecture, lifecycle, costmaps, planners, controllers, recovery logic, localization contracts, AMR-specific constraints, debugging, tuning, and extension points.

---

## Why This Track Exists

The current ROS 2 track gives you the architectural overview, but AMR-grade Nav2 work requires a separate layer of depth:

- You need to reason about why a robot stops, oscillates, replans forever, or exhausts recoveries.
- You need to connect TF, localization, costmaps, planners, controllers, and BT execution as one system.
- You need to debug from logs, bags, topics, parameters, and field symptoms, not from theory alone.
- You need the judgment to tune and extend Nav2 for warehouse, factory, and aisle-constrained robots.

This track is designed for that jump from “I know what Nav2 is” to “I can own Nav2 in production.”

---

## Prerequisites

- `../01-nodes-topics-actions.md`
- `../02-tf2-time-qos.md`
- `../03-nav2-architecture.md`

Before starting this track, you should already be comfortable with:

- ROS 2 actions, lifecycle nodes, and parameter files
- TF tree semantics: `map -> odom -> base_link -> sensor_frame`
- Basic costmap, planner, and controller vocabulary
- Reading ROS 2 logs, bag files, and topic graphs

---

## Planned File Structure

```text
nav2/
├── 00-learning-plan.md
├── 01-nav2-system-architecture.md
├── 02-nav2-bringup-lifecycle-actions.md
├── 03-nav2-bt-navigator-and-bt-xml.md
├── 04-nav2-costmaps-and-layers.md
├── 05-nav2-global-planning.md
├── 06-nav2-local-control-and-cmdvel.md
├── 07-nav2-localization-odom-amcl-ekf.md
├── 08-nav2-recoveries-progress-and-goal-checkers.md
├── 09-nav2-waypoints-docking-zones-and-missions.md
├── 10-nav2-parameters-launch-and-plugin-extension.md
├── 11-nav2-debugging-observability-and-bag-analysis.md
├── 12-nav2-amr-failure-patterns-and-capstone.md
├── 13-nav2-senior-interview-questions.md
└── exercises/
    ├── 01-bringup-and-lifecycle.md
    ├── 02-bt-trace-and-recovery-loop.md
    ├── 03-costmap-tuning-lab.md
    ├── 04-planner-comparison-lab.md
    ├── 05-controller-tuning-lab.md
    ├── 06-localization-contract-diagnosis.md
    ├── 07-recovery-policy-design.md
    ├── 08-zones-waypoints-and-missions.md
    ├── 09-plugin-extension-lab.md
    └── 10-amr-debug-capstone.md
```

These lesson and exercise files are the planned build-out for this track.
This page is the roadmap for creating them in order and then studying them.

---

## Time Budget

- Recommended duration: 8 to 10 weeks
- Suggested load: 4 to 6 hrs/week
- Total depth target: 40 to 55 hours

This assumes you want both conceptual depth and enough repetition to diagnose Nav2 incidents under pressure.

---

## Track Outcomes

By the end of the track, you should be able to:

- Explain every major Nav2 server and how they interact during `NavigateToPose`
- Diagnose failures by separating localization, costmap, planner, controller, and BT-policy problems
- Tune a controller and costmap for an AMR in narrow aisles without relying on cargo-cult defaults
- Understand how AMCL, robot localization, wheel odometry, and IMU quality affect Nav2 behavior
- Design recovery policies that match real AMR operating conditions
- Add operational map semantics such as keepout zones, speed zones, waypoint flows, and docking behaviors
- Extend Nav2 with parameters, launch structure, or custom plugins when defaults are insufficient

---

## Weekly Schedule

### Week 1: System Architecture and Goal Flow

**Planned lesson:** `01-nav2-system-architecture.md`

**Focus:**

- End-to-end flow of a goal through `bt_navigator`, planner, controller, behavior server, and lifecycle manager
- What Nav2 owns vs what localization, perception, and the rest of the AMR stack own
- How actions, topics, transforms, and parameters meet at runtime

**Hands-on:**

- Draw the node graph for a working Nav2 bringup from memory
- Trace one `NavigateToPose` request from UI or mission layer to `/cmd_vel`

**Planned exercise:**

- `exercises/01-bringup-and-lifecycle.md`

**Senior interview focus:**

- Explain the full navigation data flow without hand-waving
- Separate navigation orchestration from localization and perception responsibilities

### Week 2: Bringup, Lifecycle, and Action Contracts

**Planned lesson:** `02-nav2-bringup-lifecycle-actions.md`

**Focus:**

- Lifecycle transitions and why partial bringup causes misleading runtime failures
- Action interfaces for `NavigateToPose`, `NavigateThroughPoses`, recovery behaviors, and waypoint flows
- Startup sequencing and server dependency ordering

**Hands-on:**

- Bring up Nav2 and intentionally break one dependency
- Observe lifecycle states and recover to a clean running system

**Planned exercises:**

- `exercises/01-bringup-and-lifecycle.md`
- `exercises/02-bt-trace-and-recovery-loop.md`

**Senior interview focus:**

- Explain lifecycle failure modes and what to inspect first in a broken bringup

### Week 3: BT Navigator and BehaviorTree XML

**Planned lesson:** `03-nav2-bt-navigator-and-bt-xml.md`

**Focus:**

- `PipelineSequence`, `RecoveryNode`, `ReactiveFallback`, decorators, blackboard values, replanning cadence
- Default tree behavior vs AMR-customized tree behavior
- When a problem belongs in BT design vs parameters vs a plugin

**Hands-on:**

- Walk a default tree tick by tick for success, planner failure, and controller failure
- Modify a recovery ordering for a blocked aisle scenario

**Planned exercises:**

- `exercises/02-bt-trace-and-recovery-loop.md`
- `exercises/07-recovery-policy-design.md`

**Senior interview focus:**

- Defend a custom BT recovery design for a warehouse robot under dynamic obstruction

### Week 4: Costmaps, Layers, and Map Semantics

**Planned lesson:** `04-nav2-costmaps-and-layers.md`

**Focus:**

- Global vs local costmaps, static layer, obstacle layer, inflation, footprint, rolling window, observation sources
- Phantom obstacles, stale obstacles, inflated corridors, unknown space policy
- Keepout zones and speed zones for AMR operations

**Hands-on:**

- Tune inflation radius and footprint on the same map and compare passability
- Inject a stale obstacle or bad observation source and inspect the resulting costmap

**Planned exercises:**

- `exercises/03-costmap-tuning-lab.md`
- `exercises/08-zones-waypoints-and-missions.md`

**Senior interview focus:**

- Explain why a planner says “no path” in an apparently open corridor

### Week 5: Global Planning

**Planned lesson:** `05-nav2-global-planning.md`

**Focus:**

- NavFn vs Smac 2D vs Smac Hybrid-A*
- Kinematic feasibility, unknown space, path quality, compute budget, aisle geometry
- Planner behavior under warehouse-style maps and blocked goals

**Hands-on:**

- Compare planner outputs on the same map under different footprint and costmap settings
- Explain when a planner failure is really a costmap failure upstream

**Planned exercise:**

- `exercises/04-planner-comparison-lab.md`

**Senior interview focus:**

- Choose a planner for a differential-drive AMR in a constrained warehouse and defend the choice

### Week 6: Local Control, Velocity Shaping, and Motion Quality

**Planned lesson:** `06-nav2-local-control-and-cmdvel.md`

**Focus:**

- DWB and Regulated Pure Pursuit
- Critics, lookahead, goal tolerances, progress checkers, goal checkers, velocity smoother
- Oscillation, corner cutting, sluggish approach, and over-conservative slowdown

**Hands-on:**

- Tune controller behavior on tight turns, aisle entry, and docking approach
- Compare controller behavior before and after parameter changes

**Planned exercise:**

- `exercises/05-controller-tuning-lab.md`

**Senior interview focus:**

- Diagnose whether a motion-quality issue is controller tuning, bad path quality, or bad localization

### Week 7: Localization Contracts and TF Correctness for Nav2

**Planned lesson:** `07-nav2-localization-odom-amcl-ekf.md`

**Focus:**

- What Nav2 expects from `map`, `odom`, and `base_link`
- AMCL vs `robot_localization`, wheel odometry, IMU fusion, transform tolerances, and timestamp drift
- Why localization defects masquerade as planning or controller problems

**Hands-on:**

- Replay a bag with localization drift and identify downstream Nav2 symptoms
- Diagnose TF lookup failures and stale transform timing

**Planned exercise:**

- `exercises/06-localization-contract-diagnosis.md`

**Senior interview focus:**

- Explain what breaks when `odom` is locally smooth but globally wrong

### Week 8: Recoveries, Goal Checking, and Mission Robustness

**Planned lesson:** `08-nav2-recoveries-progress-and-goal-checkers.md`

**Focus:**

- Clear costmaps, spin, backup, wait, retry budgets, progress checker, goal checker
- What recoveries should exist for blocked aisles, temporary humans, phantom obstacles, and stuck robots
- How AMR missions should degrade instead of failing opaquely

**Hands-on:**

- Design a recovery chain for three AMR incident patterns
- Explain when to stop retrying and escalate to the higher-level mission system

**Planned exercise:**

- `exercises/07-recovery-policy-design.md`

**Senior interview focus:**

- Design a recovery policy that is safe, observable, and operationally useful

### Week 9: Waypoints, Docking, Zones, and AMR Mission Features

**Planned lesson:** `09-nav2-waypoints-docking-zones-and-missions.md`

**Focus:**

- Waypoint follower, route execution, docking, semantic map overlays, no-go areas, speed zones
- Mission-layer interactions with Nav2 and when mission bugs look like navigation bugs

**Hands-on:**

- Model a warehouse lane policy with zones and speed constraints
- Walk a multi-stop route and reason about what belongs in Nav2 vs the task layer

**Planned exercise:**

- `exercises/08-zones-waypoints-and-missions.md`

**Senior interview focus:**

- Explain how you would represent keepout zones, preferred lanes, and docking constraints cleanly

### Week 10: Extension, Observability, and AMR Capstone

**Planned lessons:**

- `10-nav2-parameters-launch-and-plugin-extension.md`
- `11-nav2-debugging-observability-and-bag-analysis.md`
- `12-nav2-amr-failure-patterns-and-capstone.md`
- `13-nav2-senior-interview-questions.md`

**Focus:**

- Parameter layering, launch composition, namespaces, plugin choices, and when to write custom code
- Bag-based diagnosis, topic/TF inspection, action tracing, costmap introspection, and failure pattern triage
- Production scenarios: narrow-aisle aborts, repeated recovery loops, phantom obstacles, stale localization, dynamic pallet intrusion

**Hands-on:**

- Run a full capstone RCA from logs, topics, TF, and parameters
- Propose the smallest fix that addresses the real root cause

**Planned exercises:**

- `exercises/09-plugin-extension-lab.md`
- `exercises/10-amr-debug-capstone.md`

**Senior interview focus:**

- Debug a field incident with incomplete evidence and justify the first three things you would inspect

---

## Exercise Model

Every module in this track should eventually include three exercise types:

1. **Inspection drill**
Example: read a BT snippet, costmap config, or topic list and explain what it will do.

2. **Hands-on lab**
Example: run Nav2, tune a parameter, replay a bag, or inspect lifecycle state transitions.

3. **AMR diagnosis task**
Example: use logs and symptoms to determine whether the root cause lives in localization, costmaps, planning, control, or BT logic.

The capstone should combine all three.

---

## Senior-Level Interview Categories

The final interview bank should be grouped by category, not by random questions.

### 1. System Architecture and Goal Flow

- Walk through `NavigateToPose` end to end
- Explain which node owns which responsibility and what inputs each requires

### 2. TF, Time, and Localization Correctness

- Explain `map`, `odom`, and `base_link` semantics
- Diagnose transform tolerance and timestamp drift issues

### 3. Costmaps and Perception Integration

- Explain layer composition, stale obstacles, inflation side effects, and observation-source bugs

### 4. Planner Selection and Tradeoffs

- Compare NavFn, Smac 2D, and Hybrid-A* for real AMR constraints

### 5. Controller Tuning and Motion Quality

- Diagnose oscillation, slowdown, corner cutting, and poor goal convergence

### 6. BT Design and Recovery Policy

- Decide when a fix belongs in BT XML, parameters, or a plugin

### 7. Production Debugging

- Explain how to separate localization faults from planning or control faults quickly

### 8. AMR Product Constraints

- Handle speed zones, keepout zones, docking, waypoint missions, and dynamic warehouse traffic

### 9. Plugin Extension and Code Ownership

- Decide when to write a custom BT node, planner wrapper, or controller plugin

### 10. Senior Judgment

- Decide what to instrument first during a real navigation incident
- Choose defaults for a new AMR platform and defend the tradeoffs

---

## Suggested Hands-On Environment

Use whichever environment is available, but the track should be designed to work with one of these setups:

- Gazebo or Isaac Sim with a differential-drive warehouse robot
- Recorded AMR bags with TF, odom, scan, costmaps, and Nav2 action traces
- A local Nav2 bringup with maps, lifecycle nodes, and CLI tools

The exercises should not depend on owning hardware. Hardware-specific tuning can be an optional extension.

---

## Completion Standard

You are done with this track when you can do all of the following without checking references:

- Draw the Nav2 node and server architecture from memory
- Explain why a recovery loop is happening from logs alone
- Read a costmap configuration and predict likely runtime behavior
- Diagnose whether a failure is localization, costmap, planner, controller, or BT policy
- Propose a safe AMR-specific Nav2 tuning change and defend it technically
- Answer senior-level Nav2 interview questions without drifting into generic ROS 2 answers

---

## Next Step

Start here after finishing the base ROS 2 track by using this roadmap page, then writing or opening the first planned lesson: `01-nav2-system-architecture.md`

Recommended first reading order inside this subtrack:

1. `01-nav2-system-architecture.md`
2. `02-nav2-bringup-lifecycle-actions.md`
3. `03-nav2-bt-navigator-and-bt-xml.md`

Once those are comfortable, move into costmaps, planning, control, and AMR-specific debugging.
