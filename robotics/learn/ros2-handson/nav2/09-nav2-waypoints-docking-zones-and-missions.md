# 09 — Nav2 Waypoints, Docking, Zones, and Missions

## How to layer multi-stop routes, docking behavior, semantic map zones, and AMR task logic without confusing navigation ownership

**Prerequisite:** 08-nav2-recoveries-progress-and-goal-checkers.md, 04-nav2-costmaps-and-layers.md, 02-nav2-bringup-lifecycle-actions.md
**Unlocks:** Cleaner mission architecture, better use of waypoint flows, safer docking integration, stronger keepout and speed-zone design, fewer ownership bugs between Nav2 and task orchestration

---

## Why Should I Care? (Context)

Real AMRs do not just go to one pose and stop.
They run missions such as:

1. visit several stations in sequence
2. pass through lane checkpoints
3. slow down in shared human zones
4. avoid keepout areas and one-way semantics
5. stage, align, and dock cleanly at charging or workstations

If you treat all of that as one giant "navigation" blob, the system becomes hard to debug and harder to extend.

This lesson is about ownership boundaries:

- what Nav2 should own directly
- what the mission layer should orchestrate
- what semantic map features should shape the navigation stack

---

## PART 1 — SINGLE GOAL NAVIGATION VS MISSION EXECUTION

---

## 1.1 `NavigateToPose` Is a Primitive, Not a Full Mission System

`NavigateToPose` answers one question:

```text
can the robot get to this pose under the current navigation policy?
```

It does not answer:

- what sequence of stops the business workflow requires
- whether a station is currently available
- how long to wait before retrying the next task
- whether a failed docking attempt should trigger charging fallback or operator escalation

That is mission-layer logic.

---

## 1.2 Multi-Stop Work Needs More Than One Goal

Production AMRs often need:

- ordered checkpoint traversal
- branch decisions between stops
- per-stop task execution
- partial completion and retry semantics

That is where waypoint or route execution becomes useful, but it still does not replace a task orchestrator.

---

## PART 2 — WAYPOINT FLOWS AND ROUTE EXECUTION

---

## 2.1 Why Waypoints Exist

Waypoints are valuable when the route itself carries meaning.

Examples:

- travel through specific lane anchors
- stop at inspection points
- approach a station through a preferred corridor
- break long movement into operationally meaningful segments

This is different from simply asking the planner for the shortest path to a single destination.

---

## 2.2 When Waypoints Belong in Nav2

Waypoint handling belongs close to Nav2 when the need is mostly geometric or route-execution related.

Good examples:

- follow an ordered path through checkpoints
- pause at known intermediate poses
- attach simple per-waypoint tasks such as wait or snapshot

It belongs higher up when:

- the mission decides dynamically which stop to skip
- business logic or fleet policy changes the route mid-run
- downstream task results determine the next navigation branch

---

## 2.3 Common Waypoint Failure Patterns

| Symptom | Likely cause |
| --- | --- |
| robot succeeds each leg but overall mission logic is wrong | waypoint layer is being used like a full task engine |
| robot revisits a checkpoint unexpectedly | mission or preemption state machine bug |
| path through checkpoints looks unnatural | waypoint set is over-constraining geometry |
| stop-and-go motion at every point kills throughput | waypoint granularity is too fine |

More waypoints do not automatically mean better control.

---

## PART 3 — DOCKING IS NOT JUST ANOTHER GOAL

---

## 3.1 Why Docking Needs Special Treatment

Docking usually has tighter requirements than ordinary navigation:

- tighter final pose tolerance
- better heading alignment
- slower and more deliberate final approach
- station-specific sensing or contact logic
- recovery policy that differs from generic aisle navigation

That makes docking a layered workflow, not just a `NavigateToPose` with a stricter goal.

---

## 3.2 A Good Docking Architecture

A typical production pattern is:

```text
mission layer chooses target station
    -> navigate to staging pose near dock
    -> enter docking-specific alignment / final approach routine
    -> confirm docked state via station-specific feedback
```

This preserves a clean split:

- Nav2 handles approach navigation
- docking routine handles precision final behavior and station semantics

---

## 3.3 Bad Docking Patterns

Common bad patterns:

1. sending the exact dock contact pose as a normal navigation goal from far away
2. hiding docking precision problems by loosening goal tolerance
3. reusing generic recoveries for failed dock alignment without dock-specific logic

These usually work in demos and fail under real repeatability requirements.

---

## PART 4 — ZONES AND SEMANTIC MAP OVERLAYS

---

## 4.1 Keepout Zones

Keepout zones express "never plan or drive here" semantics.

Use them for:

- forbidden aisles
- machine envelopes
- unsafe floor sections
- maintenance areas

If a keepout zone is treated as a mission-layer hint instead of a navigation-layer constraint, planners may still route through it and force higher layers to clean up the damage.

---

## 4.2 Speed Zones

Speed zones express "you may pass here, but with restricted behavior."

Examples:

- human crossover areas
- dock approach corridors
- high-slip surfaces
- sensor-poor corners where slower motion is safer

These are often better expressed as map semantics than as ad hoc mission rules because they affect local motion quality everywhere in that region.

---

## 4.3 Preferred Lanes and Routing Semantics

Some environments want more than obstacles and speed limits.

Examples:

- one-way aisles
- preferred traffic lanes
- forbidden turn pockets
- merge zones near conveyor intersections

Not all of these belong inside default costmaps. Some belong in higher-level route planning or fleet traffic management. The key is to avoid duplicating contradictory policy in two layers.

---

## 4.4 Costmap Filters and Semantic Overlays

Costmap filters are often the clean way to express keepout and speed semantics because they modify navigation behavior from map-linked data instead of from scattered if-statements.

Use semantic overlays when:

- the rule is spatial and repeatable
- many missions need the same rule
- you want RViz and operator tooling to reason about the same policy surface

Do not hardcode zone semantics in task logic if the real truth is geographic.

---

## PART 5 — WHAT BELONGS IN NAV2 VS THE MISSION LAYER

---

## 5.1 Ownership Matrix

| Concern | Best owner |
| --- | --- |
| single-goal path planning and following | Nav2 |
| local obstacle avoidance and recoveries | Nav2 |
| keepout and speed semantics tied to map areas | Nav2 plus semantic map/filter setup |
| business decision about which station to visit next | mission layer |
| retrying a pickup task after manipulator failure | mission layer |
| deciding between multiple robots for the same job | fleet / mission layer |
| final station-specific docking confirmation | docking routine / mission layer |

Good architecture reduces ambiguity here.

---

## 5.2 The Most Common Layering Mistake

The most common mistake is trying to encode business workflow inside navigation or encode navigation policy inside business workflow.

Examples:

- task code manually slows the robot instead of using speed zones
- Nav2 waypoint flow is stretched into a full mission state machine
- mission layer tries to micromanage every recovery step

Those choices create duplicate logic and harder incident ownership.

---

## PART 6 — FAILURE PATTERNS IN PRODUCTION AMR MISSIONS

---

## 6.1 Zone Ignored

If the robot ignores a keepout or speed zone, inspect:

1. whether the semantic data is loaded at all
2. whether the correct costmap is using the filter
3. whether the zone affects planning, control, or both as intended
4. whether the mission route bypassed the intended geographic assumption

This is often a wiring or scope mistake, not a planner algorithm issue.

---

## 6.2 Dock Approach Looks Fine Until Final Meter

Likely causes:

- staging pose is wrong
- final alignment belongs to a dock-specific routine, not generic navigation
- localization precision is insufficient near the station
- goal checker or progress checker semantics do not match docking mode

Do not keep retuning global navigation for the last half meter of a station-specific task.

---

## 6.3 Multi-Stop Route Keeps Failing Mid-Mission

Possible causes:

- recovery budget should reset or escalate differently per leg
- mission layer does not understand partial completion
- waypoints are too dense and make local control brittle
- preemption logic is corrupting the active route state

This is where navigation and orchestration need a clear contract.

---

## PART 7 — DESIGN CHECKLISTS FOR AMR TEAMS

---

## 7.1 When to Use Waypoints

Use waypoints when:

- route geometry matters
- the intermediate poses are operationally meaningful
- you want ordered movement through known anchors

Do not use waypoints just because the robot behaved oddly once on a single goal. That is often a planner, costmap, or controller problem.

---

## 7.2 When to Use Zones

Use zones when the rule is:

- spatially stable
- repeated across missions
- understandable as a map property

Examples include keepout, speed reduction, and restricted approach spaces.

---

## 7.3 When to Split Off a Docking Routine

Split docking out of generic navigation when:

- final pose tolerance is materially tighter than normal goals
- station semantics differ between sites or hardware versions
- you need station-specific sensing or contact confirmation
- recovery behavior near the dock is not the same as aisle recovery behavior

That is almost always true in real AMRs.

---

## 7.4 Senior Interview Version

Strong answers explain:

- why waypoint flows do not replace a mission layer
- why docking is a staged workflow, not a normal goal from far away
- why keepout and speed rules should often be expressed as map semantics
- where Nav2 ownership ends and task/fleet logic begins

That is the difference between a demo integration and a production AMR architecture.

---

## Next Lesson

Continue to 10-nav2-parameters-launch-and-plugin-extension.md.
That lesson shows how to organize the configuration, launch, and plugin structure that supports all of these AMR behaviors without creating an unmaintainable parameter pile.
