<!-- markdownlint-disable MD033 -->

# Exercise08 — Zones, Waypoints, and Mission Layering

## Companion exercises for `../04-nav2-costmaps-and-layers.md`, `../09-nav2-waypoints-docking-zones-and-missions.md`, and `../12-nav2-amr-failure-patterns-and-capstone.md`

**Estimated time:** 90 to 115 minutes  
**Prerequisite lessons:** ../04-nav2-costmaps-and-layers.md, ../08-nav2-recoveries-progress-and-goal-checkers.md, ../09-nav2-waypoints-docking-zones-and-missions.md, ../12-nav2-amr-failure-patterns-and-capstone.md

**Mode options:**

- **Simulation:** model keepout zones, speed zones, and waypoint flows in a warehouse-style map and reason about resulting behavior.
- **Bag replay and logs:** inspect route execution, waypoint transitions, and mission outcomes from recorded runs.
- **Design review:** use the supplied map semantics and mission rules to decide what belongs in Nav2 versus the higher-level mission system.

**Validation goal:** by the end of this lab you should be able to separate map semantics, route or waypoint execution, and mission orchestration without collapsing them into one oversized navigation configuration.

---

## Overview

AMR teams often overload Nav2 with product logic that belongs elsewhere, while also pushing map semantics into the mission layer where they become inconsistent and hard to audit.

This lab is about drawing clean boundaries.

You will decide:

1. which rules belong in zones or map semantics
2. which belong in waypoint or route execution
3. which belong in the mission layer above Nav2
4. how those layers should exchange evidence and status in production

---

## Section A — Layer Ownership

Assign the primary owner for each requirement.

| Requirement | Primary owner | Why |
| --- | --- | --- |
| robot must never enter a pedestrian-only corridor | ? | ? |
| robot should slow to `0.25 m/s` while crossing a docking apron | ? | ? |
| robot must visit pick station A, then staging B, then charger C | ? | ? |
| robot should skip a station if the mission order changes mid-route | ? | ? |
| robot must abort a route if localization confidence drops below threshold | ? | ? |

Possible owners: map semantics or zones, Nav2 waypoint or route execution, mission layer, or shared contract with explicit boundary.

<details><summary>Answer guidance</summary>

Typical answers:

- never-enter rules belong in keepout or semantic map policy because they are spatial invariants
- speed reduction in a known physical area often belongs in zone semantics or route policy connected to that zone
- ordered station visiting is usually mission or waypoint-layer orchestration, not raw planner config
- order changes are mission logic
- localization-confidence abort is a cross-layer contract, but mission escalation is usually owned above Nav2

</details>

- [ ] Done

---

## Section B — Warehouse Scenario Design

You are given this simplified warehouse map description:

- two long storage aisles with bidirectional travel
- one narrow charging bay entrance
- one human-heavy packing zone near outbound staging
- one keepout area reserved for forklifts
- missions consist of pick, stage, dock, and optional charge tasks

### Task B1 — Zone Design Table

Fill in the table.

| Area | Zone type or rule | Intended behavior | Why it should not be handled as an ad hoc operator convention |
| --- | --- | --- | --- |
| packing zone | ? | ? | ? |
| charging bay entrance | ? | ? | ? |
| forklift keepout area | ? | ? | ? |
| outbound staging lane | ? | ? | ? |

<details><summary>Answer guidance</summary>

Good answers show that spatial policy should be encoded where it is reviewable and repeatable. Operator conventions are too easy to forget, apply inconsistently, or bypass during automation changes.

</details>

- [ ] Done

---

### Task B2 — Waypoint Plan Critique

A teammate proposes this route plan:

```text
Mission "pick-and-stage":
1. Navigate to aisle waypoint A
2. Navigate to packing zone midpoint
3. Navigate to staging waypoint B
4. If blocked, retry step 2 five times
5. If still blocked, clear costmaps and continue mission
```

**Questions:**

1. What is weak about using a fixed midpoint in a human-heavy zone?
2. Why is step 4 not enough as a mission policy?
3. What information should the mission layer consider before deciding to skip, wait, reroute, or abort?

<details><summary>Answer guidance</summary>

Strong answers should mention that waypoints inside dynamic human zones may be brittle, and repeated retries hide the distinction between transient congestion and a policy conflict. Good mission policy uses route state, zone semantics, obstruction duration, priority, and mission deadlines.

</details>

- [ ] Done

---

## Section C — Nav2 Versus Mission Logic Decision Work

For each feature request, decide whether to implement it primarily in Nav2 configuration, a mission service, or both.

| Feature request | Best home | Why |
| --- | --- | --- |
| dynamic speed reduction near people-heavy packing area | ? | ? |
| skip non-critical waypoint if queue time exceeds 90 seconds | ? | ? |
| docking alignment with precise final approach behavior | ? | ? |
| route-wide no-entry region for temporary floor maintenance | ? | ? |
| battery-based diversion to charger after route completion | ? | ? |

<details><summary>Answer guidance</summary>

This section tests boundary judgment:

- spatial constraints often belong in Nav2 map semantics or zone-aware routing
- mission priorities, skip logic, and battery policies usually belong above Nav2
- docking often crosses both layers: Nav2 handles motion and approach semantics, while the mission layer decides when docking is required

</details>

- [ ] Done

---

## Section D — Status and Observability Design

Your operations dashboard currently shows only `mission_running`, `mission_failed`, and `navigation_failed`.

### Task D1 — Replace the Status Model

Propose six statuses that better separate:

- zone-policy hold
- waypoint execution progress
- route obstruction
- docking-specific behavior
- mission-layer decision wait
- hard navigation failure

Then answer:

1. Why does the current model create poor incident reports?
2. Which status transitions should be recorded for later bag or log review?

<details><summary>Answer guidance</summary>

Good answers recognize that a robot waiting correctly in a speed or human zone is not the same as a route failure. Likewise, waypoint progression and docking alignment should not be flattened into a single generic navigation status.

</details>

- [ ] Done

---

## Section E — Route Review Without Hardware

Use this incident summary.

```text
Mission: pick station S4 -> packing zone -> outbound staging
Robot reached S4 successfully.
Robot slowed in the packing zone as expected.
Robot then waited 140 seconds behind intermittent human traffic.
Mission system marked navigation as failed and dispatched a charger mission.
Operators reported that the robot was behaving safely and should not have abandoned the route.
```

**Questions:**

1. Which layer most likely made the wrong decision?
2. What evidence suggests Nav2 may have behaved correctly?
3. How would you redesign the contract between Nav2 and the mission layer so this incident is classified correctly?

<details><summary>Answer guidance</summary>

The mission layer is the likely culprit here because the robot appears to have respected zone behavior and traffic conditions rather than suffering a motion failure. The redesign should expose a clear state such as `waiting_for_zone_clearance` or `transient_route_hold` so higher-level logic can distinguish safe delay from true navigation failure.

</details>

- [ ] Done

---

## Section F — Mission Policy Draft

Write a short design note covering:

1. what spatial rules are encoded in map or zone semantics
2. what waypoint sequencing logic lives above Nav2
3. how a blocked route is represented back to operations
4. when the mission system is allowed to skip, reroute, or abort

Target length: 12 to 16 lines.

<details><summary>Answer guidance</summary>

The best notes create crisp boundaries and make the status contract explicit. They avoid phrases like "Nav2 handles the mission" because that blurs ownership and makes future debugging harder.

</details>

- [ ] Done

---

## Section G — AMR Production Reflection

Answer briefly but concretely.

**G1.** Why is zone semantics a reliability feature, not just a convenience feature?

**G2.** Which kinds of waypoint logic would you refuse to bury inside a Nav2 parameter file?

**G3.** What single dashboard view would help operations understand whether a route is blocked, paused by policy, or actually broken?

<details><summary>Answer guidance</summary>

Strong answers mention auditability, repeatability, spatial safety rules, and the need to keep product policy visible at the mission layer.

</details>

- [ ] Done

---

## Deliverable Template

```text
Scenario reviewed:

Zone semantics proposed:
- keepout:
- speed:
- staging or docking:

Waypoint or mission decisions:
- sequence ownership:
- skip policy:
- reroute policy:

Status model:
1.
2.
3.
4.
5.
6.

Contract between Nav2 and mission layer:

Open risks:
```

---

## Success Criteria

You have completed this lab well if you can:

1. separate spatial semantics from mission logic without creating gaps in responsibility
2. explain what belongs in zones, waypoints, docking behaviors, and higher-level mission policy
3. redesign statuses so safe waiting and true navigation failure are not conflated
4. make the route and mission contract understandable from logs, bags, or design review alone
