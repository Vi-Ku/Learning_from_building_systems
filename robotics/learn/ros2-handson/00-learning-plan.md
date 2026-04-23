# ROS2 Hands-On Exercises — Learning Plan

## For

Engineer who reads ROS2 logs and bags daily but has not yet built the muscle memory.

## Goal

Write, debug, and tune any ROS 2 node from scratch, and understand TF2 plus the high-level Nav2 architecture.

---

## Why This Track Exists

You have 1524 lines of ROS 2 theory in `zephyr/study-notes/05-jetson-ros2.md`.
You debug Nav2, TF lookups, and QoS issues in production every week.
But theory without hands-on exercises does not build the "just know it" reflex.

**Goal:** After 3 weeks, you can write a working ROS 2 node, diagnose TF failures, and read a Nav2 behavior tree without looking anything up.

---

## Weekly Schedule

### Week 1: Core ROS2 Patterns (3 hrs)

- Nodes, publishers, subscribers, timers
- Services (sync) vs Actions (async with feedback)
- Lifecycle nodes and why an AMR uses them
- QoS profiles: reliability, durability, history, and when `BEST_EFFORT` drops frames

**Exercises:** `exercises/01-pub-sub-timer.md`, `exercises/03-action-server.md`

### Week 2: TF2 + Time + QoS (3 hrs)

- Transform tree: `map → odom → base_link → sensor_frame`
- `lookupTransform()` with timeout and why it throws
- Latency and time sync: `use_sim_time`, stamp tolerance
- QoS mismatch and the silent subscription failure

**Exercises:** `exercises/02-tf2-broadcaster.md`

### Week 3: Nav2 Architecture (4 hrs)

- BehaviorTree XML: Navigate, ComputePath, FollowPath, Recovery
- Costmaps: global vs local, inflation radius, lethal obstacles
- Controller plugins: DWB and RPP, and what parameters map to robot tuning
- How `robot_localization` feeds `odom` to Nav2

**Exercises:** `exercises/04-nav2-diagnostics.md`

**Deep-track next:** `nav2/index.html` — dedicated Nav2 mastery plan for AMR production depth.

---

## Status

🟢 Content complete — 3 lessons + 4 exercises.
