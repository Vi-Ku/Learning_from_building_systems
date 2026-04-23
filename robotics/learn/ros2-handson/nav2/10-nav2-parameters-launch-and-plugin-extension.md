# 10 — Nav2 Parameters, Launch, and Plugin Extension

## How to structure configuration, launch composition, and extension decisions so an AMR navigation stack stays maintainable in production

**Prerequisite:** 09-nav2-waypoints-docking-zones-and-missions.md, 08-nav2-recoveries-progress-and-goal-checkers.md, 01-nav2-system-architecture.md
**Unlocks:** Cleaner parameter ownership, safer rollout of tuning changes, better launch decomposition, smarter plugin-extension decisions, stronger long-term maintainability for AMR navigation stacks

---

## Why Should I Care? (Context)

Teams often reach a point where Nav2 technically works but the surrounding system becomes fragile:

1. one giant YAML file contains every robot, every site, and every experiment
2. launch files hide parameter overrides in too many places
3. nobody knows whether a behavior should be changed via parameters, BT XML, or custom code
4. a small plugin need triggers a large unnecessary fork of Nav2 defaults
5. tuning changes fix one robot and quietly break another

That is not a planner or controller problem anymore. It is a configuration and architecture problem.

---

## PART 1 — PARAMETER LAYERING SHOULD MATCH OWNERSHIP

---

## 1.1 One Giant YAML Is a Trap

If one parameter file mixes:

- robot geometry
- sensor topics
- controller tuning
- site-specific map overlays
- mission mode differences
- experimental overrides

then no one can tell what changed behavior or why.

Production Nav2 needs parameter layering that mirrors real ownership boundaries.

---

## 1.2 A Practical Layering Model

Typical useful layers:

| Layer | Owns |
| --- | --- |
| platform base | shared defaults for a robot family |
| robot SKU or hardware variant | footprint, sensors, actuator-related limits |
| site or facility | maps, zones, station geometry, environment-specific constraints |
| runtime mode | sim vs hardware, debug vs production, docking mode if separated |
| last-mile override | temporary targeted experiments with explicit expiration |

This does not require many files for aesthetic reasons. It requires them for blameability and safe change control.

---

## 1.3 What Should Not Be Mixed

Do not casually mix these in one silent override layer:

- footprint changes with controller tuning
- localization frame settings with planner selection
- site-specific zones with universal motion limits

Those changes have different blast radii.

---

## PART 2 — PARAMETER DECISIONS BY CATEGORY

---

## 2.1 Good Candidates for Shared Defaults

Usually safe to share across a robot family:

- planner and controller plugin selections
- baseline progress and goal checker policy
- common BT tree choice
- common costmap structure

Only do this when the robots are truly similar enough. Shared defaults are powerful and dangerous.

---

## 2.2 Good Candidates for Robot-Specific Overrides

Usually robot-specific:

- footprint or radius
- acceleration and velocity limits
- sensor topic names and frame IDs
- minimum executable speed due to motor/base deadband
- docking approach behavior if station hardware differs

If these live in shared defaults, cross-robot regressions are inevitable.

---

## 2.3 Good Candidates for Site-Specific Overrides

Usually site-specific:

- map files
- keepout and speed zone overlays
- dock staging poses
- aisle-width-driven safety margins
- environmental sensor peculiarities

If a site forces a special rule, document it as a site rule, not as a universal Nav2 truth.

---

## PART 3 — LAUNCH STRUCTURE SHOULD EXPOSE THE SYSTEM SHAPE

---

## 3.1 Launch Files Are Architecture, Not Boilerplate

Your launch structure should make it easy to answer:

- where localization is started
- where Nav2 servers are started
- which parameter layer is active
- what namespaces apply to this robot
- whether composition is enabled
- what lifecycle manager controls which nodes

If launch files hide those answers, debugging takes longer than it should.

---

## 3.2 Useful Decomposition Patterns

Common clean split:

```text
top-level robot bringup
    -> localization launch
    -> map and semantic data launch
    -> nav2 bringup launch
    -> mission or docking services launch
```

This makes ownership boundaries visible.

---

## 3.3 Composition and Namespacing

Composition and namespaces are valuable, especially for multi-robot or performance-sensitive systems, but they increase observability demands.

Be explicit about:

- node names
- namespaces
- topic remaps
- parameter sources
- lifecycle manager scope

If namespacing is inconsistent, teams may debug the wrong robot instance or the wrong parameter set.

---

## PART 4 — WHEN TO USE PARAMETERS, BT XML, OR A PLUGIN

---

## 4.1 Use Parameters When the Mechanism Is Right but the Tuning Is Wrong

Parameter changes are appropriate when:

- the selected planner or controller family is already correct
- the BT policy is conceptually correct
- the issue is threshold, weight, frequency, tolerance, or limit tuning

Examples:

- adjust lookahead or critic weights
- change progress-checker thresholds
- tune costmap inflation or observation persistence

Do not write code for problems that are already handled by configuration.

---

## 4.2 Use BT XML When the Problem Is Policy

BT changes are appropriate when:

- retry ordering should change
- one recovery should happen before another
- a condition should gate a subtree
- goal updates should interrupt behavior differently

If the problem is "what should the robot try next," BT XML is often the right layer.

---

## 4.3 Use a Plugin When the Existing Mechanism Is Fundamentally Insufficient

Plugin work is appropriate when:

- available planner/controller/behavior logic cannot express the needed behavior
- semantic needs are not representable through existing layers
- you need custom scoring, filtering, or behavior that is stable enough to own in code

Examples:

- custom controller behavior for a platform-specific constraint
- custom costmap layer or filter tied to product semantics
- custom BT action or condition node integrating system-specific state

Code should be the answer to insufficiency, not impatience.

---

## PART 5 — PLUGIN EXTENSION DECISION CHECKLIST

---

## 5.1 Ask These Questions Before Writing a Plugin

1. can existing parameters solve it cleanly?
2. can BT policy solve it more safely?
3. is the need specific to one robot/site or a true product capability?
4. can the team test and maintain this extension for years, not days?
5. does the extension create a second architecture that future engineers must rediscover?

If the answer to the first two questions is yes, do not write code.

---

## 5.2 Good Reasons to Extend

Good reasons:

- product has stable, recurring needs not handled by current plugins
- the extension aligns with a clear ownership boundary
- the behavior can be verified with bags, simulation, and targeted field tests
- the team is willing to own documentation and rollout discipline

Bad reasons:

- tuning is hard
- defaults were not understood
- one temporary site issue created pressure for a quick hack

---

## PART 6 — COMMON CONFIGURATION FAILURE PATTERNS

---

## 6.1 Parameter Override Shadowing

Symptoms:

- a change appears to do nothing
- a fix works in one launch path but not another
- sim and hardware behave differently without obvious code changes

Typical cause:

- the real effective parameter source is not the file the team edited

This is why parameter layering must be visible and documented.

---

## 6.2 Shared Defaults Break One Robot Variant

Symptoms:

- one robot class improves
- another starts oscillating, clipping, or timing out

Typical cause:

- robot-specific constraints were stored in a supposedly shared layer

This is a configuration ownership bug, not a controller bug.

---

## 6.3 Launch Composition Hides Missing Dependencies

Symptoms:

- nodes exist but behavior is half-ready
- wrong namespace or remap causes silent miswiring
- lifecycle activation succeeds but runtime contracts fail

This happens when launch structure optimizes convenience over visibility.

---

## PART 7 — SAFE ROLLOUT OF NAV2 CHANGES

---

## 7.1 Validate in a Fixed Order

For any meaningful Nav2 change:

1. diff the effective parameters, not just the source file
2. test in simulation or bag replay when possible
3. validate open space, narrow aisle, and final approach separately
4. check logs for lifecycle, TF, checker, and recovery changes
5. roll out to one robot or site before broad adoption

This is how you keep tuning work from becoming a fleet-wide roulette wheel.

---

## 7.2 What to Instrument

Useful evidence for rollout review:

- planner and controller selected plugin IDs
- effective checker thresholds
- recovery counts and abort reasons
- command-chain topics before and after smoothing/safety
- localization and TF warnings during the run

If you cannot observe the effect, you cannot safely own the extension.

---

## 7.3 Documentation Expectations

Every non-trivial Nav2 extension should answer:

- what problem it solves
- why parameters or BT XML were insufficient
- what upstream and downstream contracts it assumes
- how to validate it
- who owns it when field failures happen

If that cannot be written cleanly, the architecture probably is not clean enough yet.

---

## PART 8 — SENIOR JUDGMENT CHECKLIST

---

## 8.1 Strong Default Behaviors

A senior-quality Nav2 configuration strategy usually has these traits:

- shared defaults are truly shared
- robot and site differences are explicit
- launch structure reveals ownership boundaries
- BT XML is used for policy, not for masking upstream defects
- plugins are added only when existing mechanisms are genuinely insufficient

---

## 8.2 Senior Interview Version

Strong answers explain when to:

- tune parameters
- modify BT XML
- split launch files
- write a plugin
- refuse a plugin and fix architecture instead

That distinction is what keeps a working Nav2 system maintainable as the AMR product grows.

---

## What You Should Be Able to Do Now

After lessons 01 through 10, you should be able to:

- trace a Nav2 goal end to end
- separate localization, costmap, planning, control, and recovery faults
- reason about waypoint flows, docking, and semantic zones
- structure parameter and launch ownership cleanly
- decide whether a new behavior belongs in tuning, BT policy, or a plugin

That is the threshold from "I know Nav2 terms" to "I can own Nav2 on a production AMR."
