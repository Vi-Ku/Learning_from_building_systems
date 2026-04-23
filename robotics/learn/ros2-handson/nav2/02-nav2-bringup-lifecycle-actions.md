# 02 — Nav2 Bringup, Lifecycle, and Action Contracts

## Why partial startup, inactive servers, and misunderstood action semantics cause most "Nav2 is broken" incidents

**Prerequisite:** 01-nav2-system-architecture.md, ../01-nodes-topics-actions.md
**Unlocks:** Deterministic bringup, faster startup debugging, correct action usage from mission code, cleaner preemption and cancellation behavior

---

## Why Should I Care? (Context)

Many teams lose days debugging Nav2 behavior that is really just incomplete bringup.

Typical examples:

1. The planner process exists, but the node never reached `ACTIVE`
2. The BT navigator is alive, but its dependencies are not activated in the right order
3. The mission layer assumes `NavigateToPose` is a fire-and-forget request instead of a long-running contract
4. A cancelled goal keeps affecting the robot because preemption and cleanup are misunderstood

Nav2 is built on lifecycle nodes and ROS2 actions. If you do not understand both, every startup or execution problem looks nondeterministic.

---

## PART 1 — BRINGUP IS A STATE MACHINE, NOT JUST A LAUNCH FILE

---

## 1.1 What "Bringup" Really Means

Bringup is not merely starting processes. It means getting all required nodes to the correct lifecycle state with working dependencies.

```text
process started != node configured != node active != system healthy
```

For Nav2, a healthy runtime usually means:

- required nodes exist
- parameters loaded successfully
- costmaps initialized
- TF dependencies available
- lifecycle transitions completed
- bonds established with the lifecycle manager

Until that chain finishes, startup is incomplete.

---

## 1.2 The Important Lifecycle States

Nav2 servers are typically `LifecycleNode`s.

```text
UNCONFIGURED -> INACTIVE -> ACTIVE
      ▲            │         │
      └──── cleanup┘         └── deactivate / shutdown on error or operator request
```

### `UNCONFIGURED`

The node exists but has not allocated or initialized the runtime resources needed for work.

### `INACTIVE`

The node has configured its resources but does not yet process the full runtime workload.

### `ACTIVE`

The node is ready for normal operation.

**Practical rule:** a node listed in `ros2 node list` is not evidence that Nav2 is ready.

---

## 1.3 Why Ordered Activation Matters

The startup chain usually has dependencies like:

```text
map_server / localization
    -> costmaps
    -> planner + controller + behavior server
    -> bt_navigator
```

If a downstream server activates before a required upstream dependency is usable, startup can fail hard or succeed in a misleading half-ready state.

Examples:

- controller activates before TF is stable and immediately reports transform problems
- planner activates before map data is ready and cannot build a usable planning space
- navigator starts while controller or planner is inactive and later rejects goals

This is why the lifecycle manager exists.

---

## PART 2 — THE LIFECYCLE MANAGER IS THE STARTUP CONDUCTOR

---

## 2.1 What `lifecycle_manager` Does

`lifecycle_manager` owns ordered transitions for a configured list of managed nodes.

Its responsibilities include:

- sending `configure` requests in order
- sending `activate` requests in order
- monitoring bond connections to managed nodes
- initiating shutdown or recovery on failure depending on configuration

Conceptually:

```text
for node in managed_nodes:
    configure(node)

for node in managed_nodes:
    activate(node)

monitor bonds while running
```

---

## 2.2 The Two Most Common Startup Failures

### Failure A: Node fails during `configure`

Typical causes:

- missing parameter file keys
- invalid plugin name
- map file unavailable
- transform dependencies missing at startup
- invalid footprint or costmap configuration

### Failure B: Node configures but never becomes meaningfully usable

Typical causes:

- inputs are technically present but stale
- localization is live but wrong frame IDs are used
- planner is active but global costmap stays empty or unknown
- controller is active but local costmap never receives observations

This second case is more dangerous because the state machine can look green while runtime behavior is red.

---

## 2.3 What to Inspect First During Bringup Debugging

1. Lifecycle state of each managed node
2. Parameter load errors during configuration
3. TF availability and frame naming
4. Map and sensor topics actually arriving with expected QoS
5. Bond disconnect or timeout messages from the lifecycle manager

The wrong debugging habit is jumping directly into BT XML or planner tuning before proving startup health.

---

## PART 3 — THE ACTION CONTRACTS THAT MATTER

---

## 3.1 Why Actions Matter in Nav2

Navigation is a long-running operation with feedback, cancellation, and final result semantics. That is exactly why Nav2 uses ROS2 actions rather than plain services.

An action gives you:

- goal acceptance or rejection
- periodic feedback
- cancellation
- final result

In production AMRs, this matters because navigation requests are frequently preempted by:

- updated tasks
- traffic management decisions
- operator intervention
- safety events
- docking or charging workflows

---

## 3.2 `NavigateToPose`

This is the most common top-level contract.

Conceptually:

```text
Goal: single target pose
Feedback: progress toward completion, remaining distance/time depending on setup
Result: succeeded, canceled, or failed/aborted
```

What the caller must get right:

- target frame must be correct, usually `map`
- goal pose must be physically reachable, not just visually near the destination
- caller must handle failure as a real operational event, not a rare exception

**AMR mistake:** sending the geometric center of a workcell instead of the legal staging pose for the robot footprint and approach direction.

---

## 3.3 `NavigateThroughPoses`

This contract extends single-goal navigation into a sequence of poses.

Use it when the path semantics matter:

- checkpoint traversal
- lane-constrained movement through specific corridor anchors
- staged navigation near docking or handoff zones

This is not a generic fleet workflow system. It is still a navigation contract. The caller must decide whether the sequence belongs in navigation or in a higher mission controller.

---

## 3.4 Other Important Action Surfaces

Depending on configuration and integration, you may also see or care about:

- `ComputePathToPose`
- `FollowPath`
- waypoint following actions
- recovery-related actions such as `Spin` or `BackUp`

These are useful both for runtime behavior and for diagnosis:

- if `ComputePathToPose` fails in isolation, the problem is upstream of path following
- if `FollowPath` fails on a known-good path, the problem is in local execution, local costmap, or controller policy

That isolation is valuable during incident triage.

---

## PART 4 — GOAL ACCEPTANCE, FEEDBACK, PREEMPTION, CANCELLATION

---

## 4.1 Goal Acceptance Is a Gate, Not a Guarantee

When an action server accepts a goal, it means the request passed initial validity checks. It does **not** mean navigation will succeed.

The actual outcome still depends on:

- planning feasibility
- controller behavior
- local environment dynamics
- lifecycle health during execution
- recovery policy

Mission software should never interpret "goal accepted" as "job complete soon."

---

## 4.2 Feedback Is Operationally Useful

Good action clients use feedback for more than UI niceness.

Examples:

- detect that progress has stalled before a hard timeout
- inform fleet logic that the robot is in a retry loop
- update operator consoles with remaining distance or recovery state
- correlate navigation feedback with safety or traffic events

If feedback is ignored, you lose one of the best real-time observability surfaces in Nav2.

---

## 4.3 Preemption and Cancellation

These are different ideas.

### Cancellation

The caller wants the current goal terminated.

Expected system behavior:

- cancel in-flight actions
- stop further tree progress for the old goal
- unwind or clean up without leaving stale intent behind

### Preemption

A new goal replaces the old one.

Expected system behavior:

- old goal no longer drives motion intent
- blackboard goal state updates cleanly
- replanning occurs for the new target
- controller tracks the new path rather than drifting through old commands

**Common bug pattern:** the mission layer sends a new goal but the robot visibly keeps executing the old one for too long. That is usually action/preemption handling, not planner quality.

---

## PART 5 — WHAT A GOOD BRINGUP SEQUENCE LOOKS LIKE

---

## 5.1 Reference Bringup Timeline

```text
1. Start map server and localization support
2. Confirm TF tree is coherent enough for navigation
3. Start Nav2 managed servers
4. lifecycle_manager configures each node
5. lifecycle_manager activates each node
6. Verify costmaps are receiving expected data
7. Send a known-good navigation goal
8. Confirm plan, feedback, and /cmd_vel appear as expected
```

This is a healthier mental model than "run launch file and hope."

---

## 5.2 Production Bringup Advice for AMRs

### Separate infrastructure readiness from task readiness

A robot may be booted, networked, and localized enough to report alive, but not ready to accept mission goals.

Expose a readiness state that depends on Nav2 lifecycle health.

### Fail fast on missing critical dependencies

Do not allow a mission layer to flood goals into a system that is not `ACTIVE`.

### Make startup observable

Operators should be able to answer:

- which managed node failed?
- in which lifecycle transition?
- from which parameter or dependency error?

If startup debugging requires SSH plus guesswork, your bringup is under-instrumented.

---

## PART 6 — FAILURE MODES THAT LOOK RANDOM BUT ARE NOT

---

## 6.1 "The Robot Worked Yesterday"

Often means one of these changed:

- map content
- parameter file
- frame naming
- sensor topic availability
- plugin configuration
- startup order after a launch change

Lifecycle and action semantics help because they turn these from vague anecdotes into concrete checks.

---

## 6.2 "Goal Accepted, Then Immediate Abort"

Likely areas:

- planner cannot find route
- goal pose invalid or inside obstacle
- dependency became unavailable mid-execution
- BT recovery exhausted very quickly due to local conditions

Action acceptance alone did not prove the system was ready to complete the task.

---

## 6.3 "Server Is Running But Nothing Happens"

Likely areas:

- node inactive despite process presence
- action client waiting on a server name mismatch
- costmap or TF starvation leaving execution stalled
- mission layer never actually sending the goal it claims to send

This is why lifecycle state and action introspection are first-class operational tools.

---

## PART 7 — OPERATOR AND ENGINEER CHECKLISTS

---

## 7.1 Bringup Checklist

- Are all required Nav2 nodes present?
- Are all managed nodes configured and active?
- Are costmaps receiving map/sensor inputs?
- Is TF coherent for `map -> odom -> base_link`?
- Can a known safe test goal be accepted and planned?

---

## 7.2 Action-Client Checklist

- Are you sending the goal in the correct frame?
- Do you handle rejection, feedback, cancel, and abort distinctly?
- Can your client preempt safely without leaving stale UI or mission state?
- Do you treat repeated aborts as a workflow event instead of endlessly retrying the same invalid request?

---

## 7.3 What You Should Be Able to Explain After This Lesson

You should now be able to explain:

1. why Nav2 bringup is a lifecycle problem, not just a launch problem
2. what `lifecycle_manager` actually controls
3. why an active process is not proof of a healthy node
4. how `NavigateToPose` and related actions behave operationally
5. why cancellation and preemption bugs often masquerade as navigation bugs

---

## 7.4 Next Step

Continue to `03-nav2-bt-navigator-and-bt-xml.md`.

That lesson explains how `bt_navigator` turns these action contracts into real navigation policy through BehaviorTree XML.
