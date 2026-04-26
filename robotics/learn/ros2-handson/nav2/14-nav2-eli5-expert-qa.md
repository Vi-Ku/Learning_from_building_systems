# 14 — Nav2: Explain Like I'm 5 + Expert Questions & Exercises

## Every core Nav2 concept explained simply, then challenged at senior level

**Prerequisite:** 01-nav2-system-architecture.md through 13-nav2-senior-interview-questions.md
**Unlocks:** Dual-fluency — explain Nav2 to anyone, defend every design decision under expert interrogation, and derive exercises from real AMR failure scenarios

---

## How to Use This Page

Each section has four layers:

- 🧒 **ELI5** — single analogy you could explain at a dinner table
- ⚙️ **Reality** — the 3-sentence technical truth behind the analogy
- 🎯 **Senior Q&A** — interview-grade questions with strong and weak answer patterns
- 🏋️ **Exercise** — a concrete task you can do in a terminal, a bag file, or on paper

Work all four layers per concept. The analogy keeps the mental model honest. The questions expose whether you can defend it.

---

# CONCEPT 1 — What Is Nav2?

---

## 🧒 ELI5

Imagine you live in a big house and need to walk from your bedroom to the kitchen without bumping into furniture. You would:

1. Look at a map of the house to plan a route
2. Walk along that route while watching for chairs or toys that weren't there when you drew the map
3. If you hit a dead end, back up and try a different way

Nav2 does exactly this — for a robot. The "map" is already drawn. Nav2 picks the route, watches for new obstacles while moving, and retries when it gets stuck.

## ⚙️ Reality

Nav2 is a ROS 2 action-based navigation framework. It converts a goal pose (target position + orientation) into a stream of velocity commands by orchestrating a planner (global path), a controller (local trajectory), a pair of costmaps (obstacles), and a BehaviorTree (retry policy). Every Nav2 component is a lifecycle node, started in a defined order via lifecycle manager, so partial startup never produces silently broken behavior.

---

## 🎯 Senior Q&A — Nav2 Big Picture

### Q1. What problem does Nav2 actually solve, and what does it deliberately leave to other systems?

**Strong answer:** Nav2 solves the navigation problem — given a pose goal, produce collision-free velocity commands, retry sensibly when blocked. It explicitly does not solve: map building (SLAM), ego-motion estimation (localization owns that), fleet coordination, docking logic, or obstacle classification. The division is intentional: anything that must persist across navigation goals lives outside Nav2.

**Weak answer:** Nav2 does autonomous navigation. It handles everything from SLAM to path following.

**Why this matters:** Interviewers use this question to filter for engineers who will push business logic into BT XML versus engineers who know when to build an adjacent service.

---

### Q2. If a robot with Nav2 arrives correctly 95 % of the time and fails 5 %, where do you start debugging?

**Strong answer:** Segment the 5 % by failure mode category first: did the goal contract fail (bad frame, invalid pose), did the planner fail (no path found), did the controller fail (oscillation, velocity clipping), did localization drift (costmap ghost, TF gap), or did recovery exhaust its budget? Each category has a distinct diagnostic entry point. Never start by tuning parameters before identifying the layer.

**Weak answer:** I would increase the inflation radius and retry count.

---

## 🏋️ Exercise 1 — System Decomposition on Paper

**Task:** Draw a box diagram with five rows: Mission Layer, Nav2, Localization, Perception, and Platform. For each pair of adjacent rows, write down: (a) what flows from the top layer to the bottom layer, and (b) what flows back up.

**Expected deliverable:** A diagram in which Nav2's inputs are `goal_pose`, `map -> odom -> base_link` TF, costmap sensor data, and a lifecycle trigger. Nav2's output is `/cmd_vel`. Localization is upstream, not inside Nav2.

**Self-check:** Can you find at least one real failure scenario where each interface can break independently of the others?

---

# CONCEPT 2 — BehaviorTree Navigator

---

## 🧒 ELI5

Think of a robot trying to cross a street. Its "brain" follows a simple rule list:

1. Look both ways — is it clear?
2. If yes → walk across
3. If a car appears → stop and wait
4. If waiting too long → go back to the curb and find a different crossing
5. If you tried three crossings and all failed → give up and tell the person holding your hand

Nav2's BehaviorTree is that rule list. It runs continuously, checks conditions, triggers actions, and decides when to give up and report failure.

## ⚙️ Reality

The `bt_navigator` node runs a BehaviorTree.CPP tree loaded from an XML file. The tree ticks on every iteration, evaluating condition nodes (is the robot close enough? did the planner succeed?) and triggering action nodes (ComputePathToPose, FollowPath, Spin, BackUp, ClearCostmap). The XML is the policy; the plugins are the mechanisms. Changing the XML changes retry policy without touching any C++ code.

---

## 🎯 Senior Q&A — BehaviorTree

### Q1. A robot keeps spinning in place after failing to reach a goal. Which BT node is most likely responsible, and how do you confirm it?

**Strong answer:** A Spin recovery node running in a recovery subtree is the candidate. Confirm by reading the active BT XML: look for a `Spin` action node wired into a recovery sequence, then check `bt_navigator` logs for which subtree is ticking. If Spin completes and the retry condition is never satisfied (path still fails), the tree may be looping through Spin → ComputePath → Spin indefinitely without a retry limit. Fix: add a `RetryUntilSuccessful` counter or a `RateController` guard before the recovery subtree.

**Weak answer:** I would increase the spin duration.

---

### Q2. When would you modify the BT XML versus writing a new BT plugin in C++?

**Strong answer:** Modify XML when the mechanism (planner, controller, recovery server) already exists but the policy (sequence, retry count, which recovery to try first) is wrong. Write a C++ plugin when no existing action node can express the required behavior — for example, a custom dock-approach action that Nav2's stock servers cannot support. The threshold is: if you are encoding a new mechanism, write code; if you are encoding a new policy, write XML.

**Weak answer:** I always write a custom plugin for flexibility.

---

### Q3. How does the BT interact with Nav2 lifecycle at runtime?

**Strong answer:** The BT itself runs inside `bt_navigator`, which is a lifecycle node. The BT loads after `bt_navigator` transitions to Active. During execution, the BT action nodes call action servers (planner, controller, recovery), which are also lifecycle nodes. If any of those servers is not in Active state, the action call will fail immediately, causing the BT node to return FAILURE — which the tree handles via its configured recovery or retry logic. A lifecycle crash mid-execution propagates as action server unavailability, not as a crash of bt_navigator itself.

---

## 🏋️ Exercise 2 — BT XML Trace

**Setup:** Use your Nav2 simulation (e.g., `nav2_bringup` with Turtlebot3) or the default BT XML file found at `nav2_bt_navigator/behavior_trees/navigate_to_pose_w_replanning_and_recovery.xml`.

**Task:**
1. Open the BT XML file and draw the full tick sequence as a flowchart.
2. Identify every recovery action node and the condition that triggers it.
3. Mark the maximum number of retries before the tree returns FAILURE to the action client.
4. Modify the XML to add a `Wait` node (5 seconds) before the first Spin recovery. Test in simulation — does behavior change?

**Self-check:** Can you trace a specific log line from `bt_navigator` back to the XML node that produced it?

---

# CONCEPT 3 — Costmaps

---

## 🧒 ELI5

Imagine your house floor map covered in colorful markers:

- **Red** = a wall or table (never walk here)
- **Orange** = close to a wall (be careful)
- **Green / empty** = safe to walk

The robot has **two** copies of this map:
- A **big copy** of the whole house for planning the trip
- A **small copy** of just what's nearby, updating in real time as new furniture appears

The big copy only updates when you scan the whole house. The small copy updates every second from the camera.

## ⚙️ Reality

Nav2 runs two `costmap_2d` node instances: the global costmap (full map extent, used by planners to find a path) and the local costmap (rolling window around the robot, used by the controller for obstacle avoidance during execution). Each costmap is built from pluggable layers: `StaticLayer` (initial occupancy grid map), `ObstacleLayer` (sensor observations marking obstacles), `InflationLayer` (cost gradient around obstacles). The two costmaps have independent update rates, sensor subscriptions, and inflation parameters. Misconfiguring them is the single most common source of Nav2 failures.

---

## 🎯 Senior Q&A — Costmaps

### Q1. The robot plans a path through a gap that does not physically exist. What layers and parameters would you investigate first?

**Strong answer:** Three candidates in order of likelihood: (1) The global costmap `StaticLayer` is using a stale map (check map server and map age). (2) The global costmap `ObstacleLayer` is raycasting too aggressively, clearing obstacles that sensors cannot see because of occlusion. (3) The inflation radius is too small, so the costmap shows a navigable gap that the physical robot cannot fit through. Diagnosis: publish static visualization of the global costmap (`nav2_costmap_2d/costmap` topic), compare to ground truth, find the discrepancy layer by layer.

**Weak answer:** Increase the inflation radius.

---

### Q2. What is the correct inflation radius formula and what happens if it is set too high?

**Strong answer:** Inflation radius should be at least the robot's inscribed circle radius (worst-case physical clearance). A conservative rule: `inflation_radius >= robot_radius + desired_clearance`. If set too high, the inflated cost region extends far from obstacles, narrowing all corridors in the costmap. In a warehouse aisle, this causes the global planner to find no path through a navigable corridor because every cell near the wall exceeds `lethal_cost_threshold`. Symptoms: `ComputePathToPose` returns FAILURE in known-navigable environments.

---

### Q3. A new obstacle appears between two scans. The robot collides with it. What is the minimum architectural change to prevent this?

**Strong answer:** This is a sensor update rate / costmap update rate / controller reaction time chain. Investigate: (1) How fast does the obstacle layer mark the new obstacle? Governed by `observation_sources` update frequency and `obstacle_max_range`. (2) How fast does the local costmap update? Governed by `update_frequency`. (3) How fast does the controller replan? Governed by its `controller_frequency`. If all three are fast enough but the robot still collides, the issue is the obstacle being outside the local costmap window. Fix: increase local costmap `width` and `height`, or add a dedicated short-range sensor with high update rate. A secondary fix is to lower `footprint` padding to give the controller more emergency reaction margin.

---

## 🏋️ Exercise 3 — Costmap Layer Isolation

**Task:**
1. In a running Nav2 simulation, open `rviz2` and visualize both the global and local costmap topics.
2. Drive a simulated obstacle (or use a bag with a moving obstacle) in front of the robot.
3. Observe the latency between the obstacle appearing in the laser scan and appearing in the local costmap.
4. Change `update_frequency` from the default to 2 Hz and document the change in latency.
5. Disable the `InflationLayer` in the global costmap parameters and send a goal through a narrow corridor. Document what happens.

**Self-check:** Can you explain every colored cell in the costmap visualization, including the gradient from lethal (254) to free (0)?

---

# CONCEPT 4 — Global Planner

---

## 🧒 ELI5

Before you drive somewhere new, you look at Google Maps, click your destination, and get a route. You do this once before you start driving. That's the global planner.

It does not care about cars that pull out as you drive. It just figures out the best roads to take given the map it has right now.

## ⚙️ Reality

The global planner (`nav2_planner`) receives a start pose and a goal pose, queries the global costmap, and returns a `nav_msgs/Path`. NavFn (Dijkstra/A*) and Smac (hybrid A*, state lattice) are the stock plugins. The planner runs once per NavigateToPose request or on replanning triggers defined in the BT. It does not track the robot's real-time position — the controller owns real-time tracking. Planner failure means no path was found given the current static obstacle state; it does not mean the path is safe to execute.

---

## 🎯 Senior Q&A — Global Planner

### Q1. NavFn and Smac are both available. How do you decide which to use for a warehouse AMR?

**Strong answer:** NavFn (A* or Dijkstra) produces kinematically-unconstrained paths — the path curves freely and does not respect the robot's turning radius or differential-drive constraints. Smac Hybrid A* produces kinematically-feasible paths by planning in SE(2) or SE(3) space, respecting minimum turning radius. For a warehouse AMR with tight aisles and a significant minimum turning radius, NavFn produces paths the controller cannot execute without deviating far from the path or getting stuck at turns. Use Smac with `minimum_turning_radius` set from the actual robot spec, and verify in simulation that the planned path matches what the robot can physically drive.

**Weak answer:** NavFn is faster so use NavFn.

---

### Q2. The planner returns a valid path in simulation but the robot often fails to complete it. Where is the mismatch?

**Strong answer:** Three main sources: (1) The global costmap is not receiving the same sensor data as the local costmap during execution, so the path passes through obstacles that only appear at runtime. (2) The robot's `footprint` in the planner differs from the physical footprint (check both the `robot_base_frame` footprint and TF). (3) The path accuracy is lower than the controller's tracking tolerance — the controller gives up following a path that diverges faster than it can correct. Diagnostic: compare the planned path to the costmap during execution, not just at plan time.

---

## 🏋️ Exercise 4 — Planner Comparison Lab

**Task:**
1. Set up two Nav2 parameter files: one with NavFn, one with SmacHybrid.
2. Place a goal 20 meters away with a 90-degree turn in the middle.
3. Record the planned path cost and path length for each planner.
4. Observe whether the controller completes the path without oscillation or path deviation.
5. Intentionally set `minimum_turning_radius` in Smac to 10× larger than the robot's spec. Document what happens to the plan quality.

**Self-check:** Can you describe the exact difference between the two paths in terms of curvature continuity and total cost?

---

# CONCEPT 5 — Local Controller

---

## 🧒 ELI5

Back to driving: Google Maps gave you the route. Now as you drive, you are constantly adjusting the steering wheel — going a little left to avoid a pothole, slowing down for a pedestrian who stepped out. That constant adjustment is the local controller.

It takes the global path as a guide and generates the actual steering commands moment by moment.

## ⚙️ Reality

The local controller (`nav2_controller`) receives the global path, the local costmap, and the robot's current pose, and outputs `geometry_msgs/Twist` on `/cmd_vel`. DWB (Dynamic Window Approach successor) and MPPI (Model Predictive Path Integral) are the stock plugins. The controller runs at high frequency (typically 20 Hz) and must balance: tracking the reference path closely, avoiding local obstacles not in the global plan, respecting velocity and acceleration limits, and reaching the goal pose with sufficient precision for the progress checker to accept it.

---

## 🎯 Senior Q&A — Local Controller

### Q1. The robot oscillates side to side while following a straight path. What controller parameters do you investigate?

**Strong answer for DWB:** Oscillation is typically caused by cost weights competing: the path-tracking cost and the obstacle avoidance cost create a saddle point the controller cannot escape. Investigate: `PathAlign` critic weight versus `ObstacleFootprint` critic weight. If inflation pushes the robot off-path and the path cost pulls it back, the result is oscillation. Fix: reduce inflation radius in the local costmap, increase `scale` of `PathAlign`, or add `RotateToGoal` as a terminal behavior to align before the goal precision check. **For MPPI:** investigate `trajectory_critic_weights`, particularly obstacle and goal cost weights.

---

### Q2. What is the difference between goal tolerance in the controller versus goal tolerance in a progress checker?

**Strong answer:** Controller goal tolerance (`xy_goal_tolerance`, `yaw_goal_tolerance`) is the threshold at which the controller considers the goal "reached" and stops generating velocity commands. Progress checker (`required_movement_distance`, `required_movement_angle`, `movement_time_allowance`) monitors whether the robot is making *progress toward* the goal during execution — it does not check proximity to the goal. A robot that stops moving due to a stuck recovery cycle can pass proximity checks but fail progress checks. They are independent failure modes and are not redundant.

---

## 🏋️ Exercise 5 — Controller Tuning Lab

**Task:**
1. Start a Nav2 simulation with DWB controller.
2. Send a goal 10 meters straight ahead.
3. Open a second terminal: `ros2 topic echo /cmd_vel` — record linear.x and angular.z at 5-second intervals.
4. Gradually reduce the `PathAlign` critic weight in the DWB parameters (not the yaml; use `ros2 param set` to test live) until oscillation appears.
5. Document the minimum weight that keeps the path stable, and explain *why* that value stabilizes behavior.

**Self-check:** Can you sketch the critic cost landscape that leads to oscillation, showing the saddle point?

---

# CONCEPT 6 — Localization (AMCL + EKF)

---

## 🧒 ELI5

You wake up in a hotel room in the dark. You don't know exactly where you are in the room. But you have:
- A map of the room you memorized before sleeping
- A flashlight (laser scanner) to check what's around you

You shine the flashlight, see a wall and a desk, and match those to your map. Now you have a good guess of where you are. Each time you move and scan again, your guess gets better.

That is AMCL. The robot takes its laser readings, matches them to the map, and figures out "I must be *here*."

## ⚙️ Reality

AMCL (Adaptive Monte Carlo Localization) is a particle filter. It maintains a set of hypothetical robot poses (particles) sampled from the probability distribution of "where the robot might be," and weights and resamples them against incoming laser scan data. `robot_localization` EKF runs in parallel, fusing IMU and odometry into a smooth continuous `/odom` estimate in the `odom` frame. AMCL corrects the cumulative drift of the EKF by updating the `map → odom` transform. Nav2 consumes the composed `map → odom → base_link` TF tree, not the raw localization output directly. If AMCL diverges, the entire TF tree is wrong, and all of Nav2's reasoning about obstacles is wrong.

---

## 🎯 Senior Q&A — Localization

### Q1. The robot navigated correctly for 10 minutes, then started driving into walls. AMCL logs show normal operation. What do you investigate?

**Strong answer:** Three scenarios with similar symptoms: (1) AMCL particle filter converged to a wrong hypothesis that happens to be consistent with recent laser scans — check the particle cloud `(ros2 topic echo /particlecloud)` spread, and check the map-odom correction magnitude over time. If the correction is drifting, AMCL lost the map match. (2) The physical map was updated but the map server was not restarted with the new map — check map timestamp versus robot deployment. (3) A dynamic obstacle (a parked pallet, a new shelf) is consistently masking the features AMCL relies on — inject a high-density scan from a different sensor to re-anchor localization.

**Weak answer:** Restart AMCL with `ros2 service call /reinitialize_global_localization`.

---

### Q2. What is the responsibility contract between AMCL and robot_localization EKF? What breaks if only one of them is wrong?

**Strong answer:** The EKF maintains frame continuity — it integrates odometry and IMU at high rate to ensure the `odom → base_link` transform is never stale. AMCL corrects the map-relative drift — it updates the `map → odom` transform at laser-scan rate. If EKF is wrong (e.g., IMU noise is unfiltered, encoder slip is not modeled), the `odom` frame drifts faster than AMCL can correct; the costmap marks obstacles in wrong positions before AMCL updates. If AMCL is wrong, the `map → odom` correction is wrong; the global costmap and planner operate on a misaligned coordinate frame. The two failures have different latency signatures — EKF errors appear in `/odom` first; AMCL errors appear in `/amcl_pose` covariance first.

---

## 🏋️ Exercise 6 — Localization Contract Diagnosis

**Task:**
1. Run `robot_localization` EKF only (no AMCL) and drive the robot 5 meters forward and back to its start.
2. Record `/tf` at start and end: does `map → odom → base_link` return to the origin?
3. Now add AMCL. Repeat the same path. Compare the end pose error.
4. Deliberately publish a bad initial pose (`ros2 topic pub /initialpose ...`) 2 meters off. Send a Nav2 goal and observe whether the robot recovers localization via AMCL particle resampling or drives into a wall.
5. Document the minimum laser-scan quality (percentage of valid beams) at which AMCL particle filter maintains convergence in your test environment.

---

# CONCEPT 7 — Recovery Behaviors

---

## 🧒 ELI5

You are pushing a shopping cart and it gets stuck on a rough patch of floor. What do you do?

1. Push harder (try again)
2. Back up and try a different angle
3. Spin the wheels to shake it loose
4. Give up and call for help

Nav2 recovery behaviors are steps 1–4 for a stuck robot. They run in order when normal navigation fails, before the whole navigation action is cancelled.

## ⚙️ Reality

Recovery behaviors in Nav2 are action servers (Spin, BackUp, Wait, ClearEntireCostmap, ClearLocalCostmap) that the BT calls when the primary planning-and-control loop returns FAILURE. Each is a pluggable server in `nav2_behaviors`. The BT XML defines which recoveries run, in which order, and how many times. A common pattern: ClearLocalCostmap → ClearGlobalCostmap → BackUp → Spin → retry goal. The recovery budget (total retries) is controlled by the `RecoveryNode` or `RetryUntilSuccessful` BT decorators around the recovery subtree.

---

## 🎯 Senior Q&A — Recovery Behaviors

### Q1. ClearEntireCostmap keeps running but the robot remains stuck. What has this told you?

**Strong answer:** If ClearEntireCostmap is clearing and the planner still fails, the obstacle is being re-marked immediately by the sensor pipeline — meaning there is a real obstacle (or a spurious sensor reading) that persists between clears. If clearing succeeds and the planner still fails, the obstacle is in the static map layer (not cleared by ClearEntireCostmap, which only clears the obstacle layer). In either case, the BT retry loop around ClearEntireCostmap is burning retries on a recovery that cannot succeed — the right diagnostic is to pause execution and inspect the costmap state manually before assuming more retries will help.

---

### Q2. Design a recovery policy for a warehouse AMR that must not spin in place (product liability, aisle width). What do you change?

**Strong answer:** Remove `Spin` from the recovery subtree in the BT XML. Replace with: (1) `BackUp` (configurable distance) to break the stuck state, (2) `ClearLocalCostmap` to remove phantom obstacles, (3) Replanning with a `wait_at_goal` timeout. Add a `GoalUpdater` node in the BT if the fleet layer can send an intermediate waypoint to route around the blocked area. Add a hard timeout using `RateController` + `ForceFailure` to escalate to the fleet layer when no recovery succeeds within N seconds. Document the maximum backup distance as a safety parameter reviewed by the mechanical/safety team.

---

## 🏋️ Exercise 7 — Recovery Policy Design

**Task:**
1. Edit the BT XML to add a maximum retry count of 2 on recovery behaviors (using `RetryUntilSuccessful`).
2. Place a permanent obstacle in the path in simulation.
3. Observe: after 2 recovery cycles, does the BT return FAILURE to the action client?
4. Add a `Wait` node (10 seconds) before `ClearEntireCostmap` in one recovery step. Does this ever help? Under what scenario?
5. Write a one-page recovery policy specification for a fictional warehouse AMR: list which recovery behaviors are allowed, the maximum retry count per behavior, the escalation trigger, and the fleet notification contract.

---

# CONCEPT 8 — Lifecycle Nodes

---

## 🧒 ELI5

Before a space rocket launches, the team runs through a checklist: power on, fuel check, communications check, systems ready. They don't fire the engines before the checklist is done.

Nav2 nodes work the same way. Every node starts in a "just configured, not running" state. The lifecycle manager runs its own checklist — "is the map loaded? is localization ready? is the costmap initialized?" — and only then transitions all nodes to the running state.

If any node fails its transition, the whole stack stops cleanly instead of silently producing bad behavior.

## ⚙️ Reality

ROS 2 lifecycle nodes have four main states: Unconfigured, Inactive, Active, Finalized, plus transitions between them (configure, activate, deactivate, cleanup). The `nav2_lifecycle_manager` automates these transitions in a hardcoded dependency order: map server → AMCL → costmaps → planner → controller → behavior server → bt_navigator. If any node fails its `on_configure` or `on_activate` callback, the lifecycle manager aborts and logs the failure rather than leaving half the stack in Active state. This prevents subtle bugs where, say, bt_navigator starts sending goals before the planner is ready.

---

## 🎯 Senior Q&A — Lifecycle Nodes

### Q1. Nav2 fails to start and the lifecycle manager logs say "timeout waiting for service". What is the typical root cause and how do you reproduce it reliably?

**Strong answer:** The lifecycle manager transitions nodes in sequence and waits for each `change_state` service to respond. A timeout means the target node did not respond within the configured timeout period. Root causes in order of frequency: (1) The node started but its `on_configure` is blocking on a slow operation (map loading, parameter parsing, plugin initialization for large costmaps). Fix: increase `bond_timeout` and `service_call_timeout` in lifecycle manager params. (2) The node crashed silently before the transition; check `/rosout` for the node's own error. (3) A dependency is not running yet (e.g., the map server is not up because a SLAM node that it depends on hasn't finished). Fix: add an explicit map-ready check to the launch file.

---

### Q2. You need Nav2 to support hot-swap of the global planner at runtime. How would you approach this with lifecycle nodes?

**Strong answer:** A lifecycle node can be deactivated, reconfigured, and reactivated without crashing the stack. The approach: (1) Deactivate `bt_navigator` and `nav2_controller` first (they depend on the planner). (2) Deactivate `nav2_planner`. (3) Restart `nav2_planner` with new plugin configuration. (4) Reactivate in reverse order. This requires a lifecycle orchestrator beyond the default lifecycle_manager, which does not support partial reactivation. Alternative: design the planner as a plugin interface where the active plugin can be changed via a ROS 2 service at runtime without restarting the node — this is how `nav2_planner` plugin swapping is actually intended to work, via `LoadBehaviors` / `UnloadBehaviors` style service calls.

---

## 🏋️ Exercise 8 — Lifecycle Observation

**Task:**
1. Start Nav2 with full bringup.
2. Run `ros2 lifecycle list` to enumerate all managed nodes and their current states.
3. Manually deactivate `nav2_controller` with `ros2 lifecycle set /controller_server deactivate`.
4. Send a navigation goal — observe what happens to the BT (it should return FAILURE when it tries to call FollowPath).
5. Reactivate the controller: `ros2 lifecycle set /controller_server activate`. Send the same goal. Document whether recovery succeeds and what the BT XML does to handle the temporary server absence.

---

# CONCEPT 9 — TF2 Contracts in Navigation

---

## 🧒 ELI5

Imagine three nested "where am I?" reference points:

1. **The house address** (`map` frame) — fixed to the world; never changes while the robot runs
2. **Where you started walking today** (`odom` frame) — drifts slowly over time as wheels slip
3. **Where your feet are right now** (`base_link`) — always the robot's exact current center

To answer "where are my feet on the map?", you chain them: feet → walking-start → house-address. The laser scanner is a child of `base_link` (it moves with the robot), so you also know where each laser beam hits on the map.

If any link in that chain is stale, wrong, or missing, every sensor reading gets projected onto the wrong part of the map — and Nav2 avoids walls that don't exist and drives into ones that do.

TF2 maintains this chain. Nav2 depends entirely on it being accurate and up to date.

## ⚙️ Reality

Nav2 requires a full TF tree: `map → odom → base_link → (sensor_frames)`. The `map → odom` transform is published by AMCL (or a SLAM node). The `odom → base_link` transform is published by the platform's odometry or `robot_localization` EKF. The `base_link → sensor_frame` transforms are static and defined in the robot's URDF. Nav2 reads this tree for goal validation, costmap coordinate transforms, controller tracking, and collision geometry. A stale transform (extrapolation beyond the TF buffer) produces a `LookupException` and causes planning or control steps to fail silently or with cryptic log warnings.

---

## 🎯 Senior Q&A — TF2

### Q1. Nav2 logs show "Could not transform from map to base_link: Lookup would require extrapolation into the past." What does this mean and how do you fix it?

**Strong answer:** TF2 has a time buffer of published transforms. The log means Nav2 tried to look up the transform at a timestamp that is older than the oldest entry in the TF buffer for that transform chain. Causes: (1) Clock synchronization is off between nodes — check `use_sim_time` parameter consistency if in simulation, or NTP drift if on hardware. (2) The odometry publisher is running at low frequency, causing large gaps in the `odom → base_link` chain; the costmap or controller tried to look up a time between two published transforms. Fix: increase odometry publication rate, verify `use_sim_time` is consistent, or increase the TF buffer duration in the problematic node.

---

### Q2. Someone reports that the robot's costmap obstacles are offset by 0.3 meters from their actual position. What TF error would produce this symptom?

**Strong answer:** A static transform error in `base_link → laser_frame` (the URDF or static TF publisher). If the laser is mounted at a different physical position than declared in the TF, all laser scan points are projected into the costmap at the wrong coordinates. Reproduce by running `ros2 run tf2_tools view_frames` and overlaying the laser frame onto a visualization of the robot model. Alternatively: place a known obstacle at a known distance from the robot, check the costmap cell, and compute the offset back to the frame error. This is a calibration issue, not a Nav2 issue — the fix is to update the URDF to match physical reality.

---

## 🏋️ Exercise 9 — TF2 Contract Audit

**Task:**
1. Run `ros2 run tf2_tools view_frames` and open `frames.pdf`.
2. Verify that `map → odom → base_link → laser_frame` are all present with no gaps.
3. Run `ros2 topic hz /tf` and `/tf_static` for 30 seconds. Record the update rates for each transform.
4. Deliberately reduce odometry publish rate to 2 Hz. Send a navigation goal and observe the latency increase in controller reaction time.
5. Introduce a 10 cm deliberate offset in the `base_link → laser_frame` static transform. Navigate to a goal 5 meters away. Document the costmap error in cm and match it to the transform offset.

---

# EXPERT QUESTIONS BANK — 10 Hard System-Level Questions

---

These questions do not map to a single concept. They test the ability to reason across multiple Nav2 layers simultaneously. There are no "correct" answers — strong answers are structured, trace system boundaries, and reason about failure modes.

---

## E1. A robot running in production passes all simulation tests but consistently replans every 3 seconds in the field. What is the most likely cause, and what three measurements would you collect to confirm?

**Answer guidance:** Replanning is triggered either by the BT replanning policy or by controller failure. In production environments, the most common source is the global costmap being updated by field laser data that invalidates the planned path (dynamic obstacles or people invalidating path cells). Measurements: (1) Record `/plan` topic — does the path change significantly between replans or only slightly? Slight changes suggest sensor noise; large changes suggest real obstacles. (2) Record costmap diff between two consecutive plans. (3) Record `bt_navigator` active tick node to confirm which BT node is triggering the replan request.

---

## E2. You need Nav2 and a legacy 2D laser-based obstacle detection system to cooperate without rewriting either. How do you architect this?

**Answer guidance:** Map the legacy system's obstacle output to `sensor_msgs/LaserScan` or `sensor_msgs/PointCloud2` and register it as an `observation_source` in the Nav2 costmap ObstacleLayer. No Nav2 code changes required. The key contract is: the legacy system must publish in a frame that is in the TF tree, at a rate sufficient for the costmap update cycle, with a `frame_id` and `stamp` that TF2 can resolve at costmap update time. The failure scenario to prevent: the legacy system publishes with a wall-clock timestamp when Nav2 is running with `use_sim_time=true` — this causes TF lookup failures and the obstacle layer silently ignores the data.

---

## E3. Navigation works in an empty warehouse but fails in a populated one with pallet jacks moving. What is the minimum architectural change to make it acceptable?

**Answer guidance:** Dynamic obstacle tracking is not built into the default Nav2 costmap pipeline. The ObstacleLayer marks and clears, but does not predict. Options: (1) Reduce decay time on the ObstacleLayer so dynamic obstacles clear quickly after the robot moves past them. (2) Add a velocity obstacle layer plugin (e.g., `social_costmap_layer`) that inflates moving obstacles in the direction of their velocity. (3) Add a separate keepout zone around warehouse lanes where pallet jacks operate, using a `KeepoutFilter` costmap filter. The minimum useful change is reducing `scan_time` / `observation_persistence` to let the costmap clear faster. The right architectural change for a production floor is (2) + (3).

---

## E4. Two engineers disagree: one says the MPPI controller is better than DWB for their AMR; the other says DWB is more predictable. Who is right and what experiment would settle it?

**Answer guidance:** Neither is universally right — both are correct in different operational regimes. MPPI samples many trajectories forward in time and picks the cost-optimal one; it is more capable in highly dynamic or nonlinear environments but has higher compute cost and is harder to tune than DWB. DWB is more predictable because its cost critics are individually interpretable and tunable. The experiment: define a fixed obstacle configuration and goal, run 50 navigations with each controller, record: (1) success rate, (2) path deviation from planned path, (3) time to goal, (4) CPU time per control cycle, (5) collision count. The result tells you which is better *for your environment and robot* — not in general.

---

## E5. How would you implement a Nav2-based system that navigates in a dynamic map (shelving units that move between shifts)?

**Answer guidance:** This requires map management beyond the static map server. Options: (1) Run a SLAM node that updates the map in real time — but this is expensive and introduces localization uncertainty. (2) Maintain a map database keyed by shift/time and load the correct map at shift start. (3) Use `nav2_map_server`'s `LoadMap` service to swap maps without restarting the stack. (4) Use a dedicated dynamic zone layer in the costmap (KeepoutFilter with temporal triggers) to mark/clear known shelf positions without changing the base map. The production architecture is usually (2) + (4): a static base map with a dynamic overlay, because SLAM at shift start introduces too much localization variance for warehouse precision requirements.

---

## E6. You are asked to instrument Nav2 to answer: "How much time does the robot spend planning vs. controlling vs. recovering in a typical shift?" Design the telemetry pipeline.

**Answer guidance:** The BT navigator emits action feedback. The approach: (1) Subscribe to `nav2_msgs/action/NavigateToPose` feedback topic — it contains `current_pose` and can be timestamped against state transitions by tracing which BT node is active (exposed via `bt_navigator` feedback). (2) Add a custom BT decorator that writes state entry/exit events to a ROS 2 topic. (3) Collect into a bag, parse offline, or publish to a metrics aggregator (e.g., InfluxDB via a ROS 2 bridge). The raw data: planning_duration (time from ComputePathToPose request to result), control_duration (time FollowPath was active), recovery_duration (time any recovery node was active), failure_events (count per type). Aggregate per goal to get per-route statistics.

---

## E7. A safety certification reviewer asks: "What guarantees does Nav2 provide against collisions?" How do you answer accurately?

**Answer guidance:** Nav2 does not provide hard real-time safety guarantees against collisions. It is a software navigation stack, not a safety-rated system. The accurate answer: Nav2 provides best-effort obstacle avoidance through its costmap-controller pipeline, with configurable safety margins (inflation radius, footprint padding). Guarantees it does NOT make: latency bounds (costmap update, controller cycle), sensor reliability, physical footprint accuracy, or network availability for action clients. Real AMR safety architecture separates Nav2's best-effort navigation from a hardware-level safety system (scan-based safety field, hardware e-stop) that operates independently of ROS 2 and cuts power if the robot enters a defined unsafe zone. Nav2 must not be presented as a safety system to a certification reviewer.

---

## E8. After a ROS 2 humble → iron upgrade, the robot navigates to within 0.5 m of the goal, then rotates randomly instead of aligning with goal yaw. What changed?

**Answer guidance:** The default BT behavior for goal alignment was modified between Humble and Iron. In Humble, the default BT uses `RotateToGoal` as a terminal behavior; in Iron, the `GoalChecker` has stricter `yaw_goal_tolerance` by default and triggers earlier. Investigate: (1) Compare the active BT XML between the two versions — `RotateToGoal` may have been removed or its trigger condition changed. (2) Check the `goal_checker` plugin's `yaw_goal_tolerance` parameter — a tighter tolerance may cause the checker to reject the pose and trigger rotation before the robot is close enough to rotate stably. (3) Check whether the `controller_frequency` was changed between versions; a lower frequency increases the angular error per cycle.

---

## E9. You have two identical robots running the same Nav2 config in the same warehouse. One completes 95 % of navigations successfully; the other completes 80 %. Where do you look first?

**Answer guidance:** Identical software + hardware + environment but different success rates = hardware or calibration difference. Systematic investigation: (1) Compare costmaps live on both robots navigating the same path — do their costmaps look different? Difference points to sensor variation (lidar calibration, laser intensity, mounting angle). (2) Compare localization particle cloud spread — is the worse robot's AMCL less converged? Could indicate a lidar that outputs noisier scans. (3) Compare odometry drift over 10-meter laps — encoder gear wear, wheel circumference mismatch, or IMU drift can all degrade EKF quality and impair localization, which feeds into costmap quality. (4) Check `/cmd_vel` to motor driver latency — a hardware variance in the drive train (e.g., motor controller firmware version) can change control response even if the Nav2 output is identical.

---

## E10. Describe a Nav2 system that can navigates autonomously for 8 hours, recovers from localization failure, handles dynamic obstacles, and produces an audit log of all navigation decisions. What components would you add beyond a default Nav2 install?

**Answer guidance:** Beyond default Nav2: (1) Localization health monitor — subscribes to `/amcl_pose` covariance, triggers re-localization or alert when covariance exceeds threshold. (2) Map update mechanism — shift-aware map loader using `LoadMap` service. (3) Dynamic obstacle layer — social costmap or velocity-aware obstacle costmap plugin. (4) Mission watchdog — fleet-layer service that retries failed Nav2 goals with alternative targets, prevents infinite recovery loops. (5) Audit logger — custom BT decorator or BT observer that writes every action node state transition to a structured log (timestamp, node, result, robot pose, goal). (6) Health dashboard — a `/nav2_health` topic aggregated from lifecycle states, localization covariance, costmap update latency, and goal success rate published at 1 Hz. None of these are in the default install; all are necessary for production operation.

---

## Summary Table

| Concept | ELI5 In One Line | Most Common Senior Trap |
|---|---|---|
| Nav2 System | A goal-to-velocity pipeline, not a full autonomy stack | Claiming Nav2 owns localization or SLAM |
| BehaviorTree | Policy in XML, mechanisms in plugins | Modifying plugins when XML change is sufficient |
| Costmaps | Two obstacle grids with different scopes and update rates | Setting inflation_radius without knowing robot radius |
| Global Planner | One-shot path on the static map | Using NavFn for a differential-drive robot with aisle constraints |
| Local Controller | High-frequency real-time path tracking | Tuning critics before isolating the oscillation source |
| AMCL / EKF | Particle filter corrects EKF drift via map matching | Treating AMCL error and EKF error as the same symptom |
| Recovery Behaviors | Escalating unstuck policy; last resort before failure | Running recoveries that cannot succeed (obstacle is in static layer) |
| Lifecycle Nodes | Start-order enforcement that prevents silent half-init | Not adjusting `bond_timeout` when costmaps are large |
| TF2 Contracts | A chain of coordinate rulers — any broken link breaks all of Nav2 | Using wall-clock stamps when use_sim_time is active |
