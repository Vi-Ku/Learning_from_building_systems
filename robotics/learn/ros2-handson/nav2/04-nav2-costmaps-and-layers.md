# 04 — Nav2 Costmaps and Layers

## How Nav2 turns maps and sensor observations into a navigable world model, and why AMRs fail when that model drifts from reality

**Prerequisite:** 01-nav2-system-architecture.md, 03-nav2-bt-navigator-and-bt-xml.md, ../02-tf2-time-qos.md
**Unlocks:** Costmap debugging, inflation tuning, obstacle-layer reasoning, safer aisle behavior, clearer root-cause analysis for "no path" and oscillation failures

---

## Why Should I Care? (Context)

When engineers say "the planner is broken," they are often really saying:

1. the costmap contains an obstacle that is not physically there
2. the costmap is missing an obstacle that is physically there
3. the inflation settings make a passable aisle look blocked
4. the robot footprint makes a legal route impossible in the current configuration
5. unknown space policy does not match the operating environment

Costmaps are not just an implementation detail. They are Nav2's working picture of the world. If that picture is wrong, every server above it becomes predictably wrong too.

---

## PART 1 — WHAT A COSTMAP IS

---

## 1.1 A Costmap Is a Grid of Navigation Risk

At a high level, a costmap is a 2D grid where each cell describes how safe or costly it is for the robot to occupy that space.

```text
free space        -> easy to traverse
inflated space    -> near obstacle, allowed but penalized
lethal obstacle   -> not traversable
unknown           -> depends on policy
```

This grid is the world model used by planning and control.

---

## 1.2 Global vs Local Costmaps

### Global costmap

Used mainly by the global planner.

Characteristics:

- map-wide or large-area representation
- stable enough for route computation
- often includes static map plus selected obstacle information

### Local costmap

Used mainly by the controller.

Characteristics:

- rolling window around the robot
- higher tactical sensitivity to recent sensor data
- used for immediate collision avoidance and short-horizon feasibility

**Operational insight:**

- If global costmap is wrong, you get bad routes or no routes.
- If local costmap is wrong, you get hesitation, oscillation, near-collision behavior, or excessive recoveries.

---

## PART 2 — THE LAYER MODEL

---

## 2.1 Layers Build the Final Master Costmap

The final costmap is usually composed from multiple layers.

```text
master costmap
    = static layer
    + obstacle/voxel layer
    + inflation layer
    + optional semantic filters or custom layers
```

Each layer contributes information differently.

---

## 2.2 Static Layer

The static layer comes from the occupancy grid map.

It provides long-lived environmental structure such as:

- walls
- racks
- permanent machinery footprints
- mapped corridor geometry

This is the backbone of the global world model.

**Failure mode:** if the facility layout changed but the map did not, the static layer can make the planner confidently wrong.

---

## 2.3 Obstacle Layer or Voxel Layer

This layer ingests sensor observations such as lidar or depth data.

Its job is to:

- mark obstacle cells when hits are observed
- clear obstacle cells when free space is observed
- fuse dynamic environment evidence into the costmap

Key configuration ideas:

- marking vs clearing
- observation source names and topics
- obstacle height handling for 3D sensors
- persistence and buffering behavior

**AMR reality:** many phantom-obstacle incidents are obstacle-layer configuration problems, not planner problems.

---

## 2.4 Inflation Layer

Inflation does not create obstacles. It expands their influence.

Why it exists:

- keep path planning away from walls and pallet corners
- account for localization uncertainty and control error
- encourage safer path margins

Why it causes trouble:

- too much inflation makes narrow but legal aisles look impossible
- too little inflation leads to aggressive paths that are hard to track safely

Inflation is one of the most operationally sensitive Nav2 settings for warehouse robots.

---

## 2.5 Costmap Filters and Semantic Layers

Modern AMR deployments often need semantics beyond raw occupancy.

Examples:

- keepout zones around forbidden areas
- speed zones near humans or workcells
- restricted approach zones near docks

These are not classical obstacle layers, but they shape motion policy through the costmap representation.

This matters because product requirements often enter navigation through map semantics, not through planner code.

---

## PART 3 — FOOTPRINT, CLEARANCE, AND PASSABILITY

---

## 3.1 The Footprint Is a First-Class Contract

The robot footprint tells Nav2 how much space the robot occupies.

If it is wrong, the rest of the stack becomes misleading:

- too small -> planner/controller accept unsafe routes
- too large -> planner says no path in routes the physical robot could take

In warehouses, footprint accuracy matters because tolerances are often tight relative to aisle width.

---

## 3.2 Aisle Passability Is Not Just Geometry

Whether a robot can pass an aisle depends on the combination of:

- map resolution
- footprint dimensions
- inflation radius
- localization uncertainty
- controller tracking quality

That means two robots using the same map can have different passability outcomes under different tuning.

---

## 3.3 Why an Apparently Open Corridor Can Still Produce "No Path"

Typical reasons:

1. inflated obstacles overlap after accounting for footprint
2. unknown space blocks planning under current policy
3. stale obstacle marks remain from sensor history
4. localization places the robot or goal slightly inside inflated/lethal space
5. map resolution plus footprint discretization closes a narrow gap

This is one of the most common senior-level Nav2 debugging questions because the answer requires costmap reasoning, not intuition from looking at the physical aisle.

---

## PART 4 — OBSERVATION SOURCES AND STALE WORLD MODELS

---

## 4.1 Marking and Clearing Must Both Work

A sensor layer is only useful if it can both detect new obstacles and remove old ones when space becomes free.

If marking works but clearing fails:

- pallets linger in the costmap after removal
- robot sees ghost walls
- planner repeatedly fails or routes around empty space

If clearing is too aggressive:

- real obstacles disappear too early
- controller behaves overconfidently

This is a balancing problem, not a binary one.

---

## 4.2 Observation Timing Matters

Even with correct geometry, bad timing can poison the world model.

Examples:

- lidar data arrives late relative to TF
- sensor topic stalls under network load
- transform tolerance is too small for actual latency
- bag replay uses time settings that make costmap updates appear flaky

When timing goes wrong, the costmap can be syntactically valid and operationally misleading.

---

## 4.3 Rolling Window Behavior in the Local Costmap

The local costmap usually moves with the robot.

That makes it excellent for:

- nearby obstacle reaction
- tight maneuvering
- controller safety margin decisions

But it also means:

- stale local obstacles can follow the robot's working space for a while
- short observation horizons may miss upcoming geometry if sensor placement is poor
- tuning decisions affect how early the controller reacts near corners and aisle entries

---

## PART 5 — UNKNOWN SPACE POLICY

---

## 5.1 Unknown Does Not Mean the Same Thing Everywhere

Unknown cells can be treated differently depending on planner and configuration.

In some deployments, unknown space is effectively forbidden.
In others, it is traversable but risky.

This policy depends on environment type.

### Structured warehouse with fixed navigation lanes

Conservative unknown policy is common because the robot should operate mostly in mapped space.

### Semi-structured industrial floor with evolving layouts

Some tolerance for unknown exploration may be necessary, though often outside standard AMR production behavior.

---

## 5.2 Why Unknown-Space Decisions Affect Throughput

Overly conservative unknown handling can:

- reject valid temporary routes
- increase deadlock frequency near partially observed areas
- cause unnecessary human intervention

Overly permissive handling can:

- send robots through poorly understood space
- create unacceptable safety or predictability problems

This is a product and operations decision, not just a technical parameter choice.

---

## PART 6 — AMR FAILURE MODES ROOTED IN COSTMAPS

---

## 6.1 Phantom Pallet Problem

The costmap still shows a pallet after the pallet is gone.

Likely causes:

- clearing rays not configured correctly
- observation persistence too long
- sensor blind spot preventing clearing evidence
- TF/timestamp mismatch making clearing invalid

Visible symptom:

- planner says no path or takes weird detours through obviously free space

---

## 6.2 Overinflated Corridor Problem

Two rack edges plus inflation plus footprint leave no legal free channel in the grid.

Visible symptom:

- corridor looks passable to humans
- planner refuses route
- operators blame map or planner without checking inflation math

---

## 6.3 Dynamic Obstacle Chatter

Humans or forklifts create rapid obstacle changes near the robot.

Visible symptoms:

- controller repeatedly slows and resumes
- robot appears indecisive at intersections
- BT recovery may trigger because local execution cannot make stable progress

This is often a local costmap behavior problem combined with controller policy.

---

## 6.4 Wrong Footprint, Wrong Conclusions

If the footprint is copied from a CAD bounding box without considering safety margins, sensor offsets, or control accuracy, the costmap may consistently misclassify viability.

Result:

- some aisles become impossible in software
- other near-collision routes look legal

This is why footprint tuning deserves real validation, not guesswork.

---

## PART 7 — HOW TO TUNE AND REVIEW COSTMAPS

---

## 7.1 Practical Tuning Order

Start with this order:

1. verify map correctness
2. verify footprint accuracy
3. verify obstacle source marking and clearing
4. tune inflation for safe but passable margins
5. review unknown-space policy
6. add semantic zones only after the base world model is trustworthy

This order prevents a common anti-pattern: compensating for bad obstacle data with semantic or planner changes.

---

## 7.2 Review Checklist

- Does the costmap match the operating environment closely enough?
- Are obstacle sources timely and correctly framed?
- Is the footprint realistic for the actual AMR body and control behavior?
- Does inflation reflect safety needs without collapsing aisle passability?
- Are keepout and speed zones used for product policy rather than to hide map defects?

---

## 7.3 What You Should Be Able to Explain After This Lesson

You should now be able to explain:

1. why global and local costmaps serve different roles
2. how layers combine into the final world model
3. why footprint and inflation settings strongly affect route feasibility
4. how stale or mistimed observations create phantom navigation failures
5. why many planner complaints are really costmap complaints upstream

---

## 7.4 Next Step

Continue to `05-nav2-global-planning.md`.

That lesson builds on the costmap model and explains planner selection, tradeoffs, and failure patterns for production AMRs.
