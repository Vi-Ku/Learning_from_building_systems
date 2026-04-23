# 01 — Nav2 System Architecture

## The runtime boundaries, contracts, and data flow behind a production AMR navigation stack

**Prerequisite:** ../03-nav2-architecture.md, ../01-nodes-topics-actions.md, ../02-tf2-time-qos.md
**Unlocks:** Confident Nav2 debugging, cleaner system decomposition, faster incident triage, better launch and ownership decisions for AMR software teams

---

## Why Should I Care? (Context)

Most Nav2 failures are not caused by a bug inside Nav2 itself. They come from a bad system boundary:

1. The mission layer sends goals Nav2 cannot safely execute
2. Localization gives Nav2 a pose that is smooth but wrong
3. Perception updates the costmap too slowly or with stale data
4. Platform code clips or rewrites `/cmd_vel`
5. Operators blame the planner when the real problem is lifecycle startup, TF, or map semantics

If you do not know where Nav2 starts and stops, every navigation incident turns into random log surfing. If you do know the boundaries, you can ask the right question first:

- Did the goal contract fail?
- Did the planner fail?
- Did the controller refuse the path?
- Did the robot never receive a usable velocity command?
- Did another subsystem make Nav2 look guilty?

That is the point of this lesson: treat Nav2 as a distributed subsystem with hard contracts, not as a black box called "navigation".

---

## PART 1 — WHAT NAV2 OWNS VS WHAT IT DOES NOT

---

## 1.1 The Short Version

Nav2 owns the logic that converts a navigation goal into planned motion commands.

It does **not** own the full autonomy stack.

```text
Mission / Fleet / UI layer
    │ creates goals and policies
    ▼
Nav2
    │ computes path, follows path, runs recoveries
    ▼
Base controller / safety layer / motor drivers
    │ turn Twist into wheel motion
    ▼
Robot hardware
```

If you say "Nav2 is responsible for moving the robot," that is directionally true but operationally incomplete. In practice, Nav2 sits between systems that define intent above it and systems that enforce physical safety below it.

---

## 1.2 Ownership Table

| Area | Usually owned by Nav2 | Usually owned outside Nav2 | Why it matters |
| --- | --- | --- | --- |
| Goal execution | Yes | Mission layer decides which goal to send | Nav2 executes; it should not invent business intent |
| Global path generation | Yes | Map production and semantic zoning come from elsewhere | Planner quality depends on upstream map quality |
| Local obstacle avoidance | Partly | Sensor drivers and perception quality are upstream | Nav2 can only react to what enters the costmap |
| Robot localization consumption | Yes | Localization generation is external | Nav2 trusts TF and pose sources |
| Recovery policy | Yes | Fleet or task layer may decide escalation after repeated failure | Recovery logic belongs near navigation policy |
| Traffic coordination between robots | No | Fleet manager / traffic manager | Nav2 is usually single-robot local intelligence |
| Hard safety stop | No | Safety PLC / collision monitor / certified safety chain | Safety must override Nav2 when required |
| Docking workflow orchestration | Partly | Product-specific mission logic often wraps Nav2 | Docking needs navigation plus product rules |

**Production rule:** if a failure spans multiple boxes, the incident owner still needs to separate primary cause from visible symptom. Nav2 often shows the symptom first.

---

## 1.3 The Four Upstream Contracts Nav2 Depends On

Nav2 assumes four things are already true:

### A. The transform tree is coherent

At minimum, Nav2 expects a reliable chain like:

```text
map -> odom -> base_link -> sensor frames
```

If this chain is missing, stale, or inconsistent, every server downstream becomes noisy:

- planner cannot locate start pose correctly
- controller tracks the wrong local pose
- costmaps insert sensor data in the wrong place
- recoveries trigger for the wrong reason

### B. The robot pose is physically believable

Nav2 does not prove localization is correct. It uses the pose it receives.

That means a robot can:

- plan through a rack because AMCL drifted laterally
- refuse a goal because it thinks it starts inside an obstacle
- oscillate near a goal because heading estimate is noisy

### C. The map and sensor observations match reality closely enough

The global planner and local controller both read a filtered version of the world. If that world model is wrong, the path logic is still internally correct and operationally useless.

### D. Velocity commands can actually reach the base

Nav2 publishes motion intent, usually on `/cmd_vel` or a shaped derivative. The base stack, safety layer, and sometimes a velocity smoother still need to accept, preserve, and execute those commands.

An AMR that never moves after a valid plan often has a downstream problem, not a planning problem.

---

## PART 2 — THE CORE NAV2 SERVERS

---

## 2.1 The Canonical Runtime Topology

```text
NavigateToPose action client
            │
            ▼
      bt_navigator
       │    │    │
       │    │    └────────► behavior_server
       │    │
       │    └─────────────► controller_server
       │
       └──────────────────► planner_server

map_server ───────────────► global_costmap
sensor topics ────────────► local/global costmaps
amcl / ekf / odom ────────► planner + controller + costmaps via TF

lifecycle_manager manages startup/shutdown for the whole group
```

This is the minimum mental model you should be able to draw from memory.

---

## 2.2 `bt_navigator`: The Orchestrator

`bt_navigator` is the policy engine. It does not compute paths itself and it does not produce wheel-level control directly. It coordinates the sequence:

```text
receive goal -> compute path -> follow path -> react to failure -> recover or abort
```

Its job is to answer:

- Which action should run next?
- When should replanning happen?
- When should recovery start?
- When should a new goal preempt the current flow?

In AMR terms, `bt_navigator` decides the navigation playbook, not the underlying geometry.

---

## 2.3 `planner_server`: Global Route Computation

The planner computes a path from start pose to goal pose on the **global** costmap.

Inputs:

- current pose from TF/localization
- goal pose from the action request
- global costmap data
- planner plugin parameters

Outputs:

- a `nav_msgs/Path`
- failure if no valid path exists under the current map assumptions

The planner does **not** guarantee the robot can track the path cleanly. That is the controller's job.

---

## 2.4 `controller_server`: Path Tracking Under Local Conditions

The controller consumes:

- the current path
- current robot pose and velocity
- local costmap state

and produces motion commands.

This server is where many "the planner is bad" complaints actually land. Common reality:

- the planner produced a valid path
- the controller finds it dynamically infeasible or unsafe at runtime
- the robot slows, oscillates, or triggers recovery

That is not a planner bug unless the path quality itself is unusable.

---

## 2.5 `behavior_server`: Recovery and Utility Behaviors

`behavior_server` usually hosts behaviors such as:

- `Spin`
- `BackUp`
- `Wait`
- assisted or custom recoveries depending on plugin set

These are invoked from the BehaviorTree when the normal compute-follow flow fails.

**AMR reality:** recoveries are not just technical conveniences. They encode product policy.

Examples:

- In a human-heavy aisle, waiting may be better than backing up.
- Near a loading station, backing blindly may be unacceptable.
- In a one-way lane, repeated spinning may waste throughput and block traffic.

---

## 2.6 Costmap Servers: The Shared World Model

The costmaps are the bridge between sensors, maps, and motion logic.

```text
Global costmap: strategic route space across the map
Local costmap: tactical obstacle space around the robot
```

The planner mainly trusts the global costmap.
The controller mainly trusts the local costmap.

When these diverge from reality, Nav2 behavior diverges from operator expectations.

---

## 2.7 `lifecycle_manager`: The Control Plane

This is the most overlooked Nav2 node during debugging.

`lifecycle_manager` handles ordered state transitions for managed nodes:

```text
configure -> activate -> monitor bond -> deactivate / cleanup / shutdown
```

Without it, you can have a planner process running but inactive, a controller waiting on unavailable costmap data, or a navigator that accepts nothing because the dependency chain never reached `ACTIVE`.

If Nav2 startup is weird, lifecycle state is one of the first things to check.

---

## PART 3 — THE END-TO-END GOAL FLOW

---

## 3.1 One `NavigateToPose` Request, Step by Step

```text
1. Mission layer sends NavigateToPose(goal)
2. bt_navigator accepts goal and loads/ticks the BehaviorTree
3. Planner action node requests a path from planner_server
4. planner_server reads global costmap + pose -> returns nav_msgs/Path
5. Controller action node sends path to controller_server
6. controller_server reads local costmap + current pose -> publishes /cmd_vel
7. Robot moves; BT keeps ticking
8. Tree may replan, continue following, recover, or abort
9. Result returns to the original action client
```

Every navigation incident should be mapped onto one of these nine steps.

---

## 3.2 Data Interfaces That Matter Most

| Interface | Type | Why it matters |
| --- | --- | --- |
| `NavigateToPose` | ROS2 action | The top-level contract most apps call |
| `/plan` or internal planned path | topic/action result | Tells you whether planning succeeded at all |
| `/cmd_vel` | topic | Confirms whether Nav2 is commanding motion |
| TF transforms | transform stream | Required by almost every server |
| global/local costmap topics | topic | Shows what Nav2 thinks the world looks like |
| lifecycle services and status | lifecycle/service | Explains startup health |

**Fast triage heuristic:**

- no plan: investigate planner, costmap, goal, localization
- plan exists but no `/cmd_vel`: investigate controller, activation, path validity
- `/cmd_vel` exists but robot does not move: investigate downstream base/safety stack

---

## 3.3 Where Preemption Actually Lands

When a new goal arrives during execution, preemption does not mean "start over from scratch everywhere instantly." It means the action and BT machinery need to:

1. accept or reject the new goal
2. cancel or replace in-flight actions cleanly
3. refresh blackboard values such as `goal`
4. trigger replanning and controller update with the new target

Poor preemption handling shows up as:

- robot finishing the old path after UI changed the mission
- delayed reaction to urgent reroute requests
- stuck action handles with confusing result semantics

That is why system architecture matters: goal replacement crosses action contracts, BT policy, and server responsiveness.

---

## PART 4 — PRODUCTION BOUNDARIES FOR AMRS

---

## 4.1 The Navigation Stack Is Not the Fleet Stack

In a warehouse, a robot usually obeys more than geometry.

Examples of policies outside raw Nav2 planning:

- lane reservation
- traffic priority at intersections
- pick/drop task sequencing
- battery-aware mission planning
- docking queue management
- operator intervention workflows

Nav2 can support these policies, but should not absorb all of them.

**Bad architecture:** mission logic hidden inside a custom planner plugin.

**Better architecture:** mission system decides which goal is legal; Nav2 executes the local navigation contract for that goal.

---

## 4.2 The Safety Layer Is a Separate Authority

Many AMRs have a safety chain that can override or suppress motion:

```text
Nav2 -> velocity smoother -> collision monitor / safety PLC -> base controller
```

If this chain clips commands, Nav2 may appear sluggish, indecisive, or broken while actually behaving correctly.

Typical symptoms:

- planner succeeds, controller publishes, robot barely moves
- robot halts near pallet forks because safety field trips repeatedly
- logs show progress checker failure even though controller keeps trying

The visible Nav2 failure is secondary. The primary cause is lower in the actuation path.

---

## 4.3 Warehouse-Specific Failure Modes

### Narrow aisle optimism

The map says the aisle is clear, but pallets protrude into the lane in reality. Global planner keeps finding a route; local controller repeatedly aborts.

### Map freshness mismatch

Static map reflects last month's layout. Planner chooses a valid path on an obsolete topology.

### Goal semantics mismatch

Mission layer sends a goal at the center of a rack face instead of the legally reachable staging pose.

### Throughput-driven parameter drift

Operators increase controller aggressiveness to improve cycle time, causing overshoot near end-of-aisle turns.

Architecture helps because each of these belongs to a different owner.

---

## PART 5 — HOW TO THINK DURING INCIDENT RESPONSE

---

## 5.1 The First Five Questions

When a robot "cannot navigate," ask these in order:

1. Did Nav2 accept the goal?
2. Did a valid global path get produced?
3. Did the controller publish usable velocity commands?
4. Did the robot physically execute those commands?
5. Did recovery fail because of policy, world model, or downstream suppression?

This sequence avoids a common waste pattern: tuning planners before proving the controller or base ever had a fair chance.

---

## 5.2 Symptom-to-Layer Mapping

| Field symptom | First layer to inspect | Why |
| --- | --- | --- |
| Goal rejected immediately | action contract / lifecycle | server may be inactive or goal invalid |
| "No path found" | planner + global costmap + localization | planner failure is usually upstream-data sensitive |
| Path exists but robot never starts moving | controller + `/cmd_vel` path | proves whether local execution started |
| Robot jerks, spins, backs up, aborts | BT + controller + local costmap | recoveries are policy responses to lower-level failure |
| Robot moves incorrectly despite clean logs | TF/localization/base stack | can be a physically wrong but internally consistent system |

---

## 5.3 Architecture Review Checklist

Use this before blaming a single node:

- Can I name the owner of each interface from mission to wheel command?
- Do I know which topics prove each stage is alive?
- Do I know which failures are upstream of Nav2 versus inside Nav2?
- Can I explain why the robot is allowed to attempt this goal at all?
- Can I distinguish navigation policy from safety policy?

If not, the architecture is still fuzzy in your head.

---

## PART 6 — WHAT GOOD LOOKS LIKE

---

## 6.1 A Healthy Nav2 System

A healthy production Nav2 stack has these characteristics:

- clear ownership between mission, navigation, safety, and base control
- lifecycle-driven startup with deterministic activation order
- stable TF and believable localization
- costmaps that reflect the operating environment closely enough to be useful
- recoveries designed for the actual AMR environment, not demo defaults
- observability on actions, paths, costmaps, and velocity commands

---

## 6.2 What You Should Be Able to Explain After This Lesson

You should now be able to explain:

1. where Nav2 sits in the AMR stack
2. what each main Nav2 server is responsible for
3. how a `NavigateToPose` goal becomes motion
4. which failures belong to Nav2 and which only appear there first
5. why system boundaries matter more than memorizing node names

---

## 6.3 Next Step

Continue to `02-nav2-bringup-lifecycle-actions.md`.

That lesson turns this architecture into operational reality: how the servers are started, transitioned to `ACTIVE`, and called through their action contracts.
