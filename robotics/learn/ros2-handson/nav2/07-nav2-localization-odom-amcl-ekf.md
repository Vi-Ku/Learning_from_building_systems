# 07 — Nav2 Localization, `odom`, AMCL, and EKF

## The frame contracts, estimator boundaries, and timing rules Nav2 depends on in production AMRs

**Prerequisite:** 06-nav2-local-control-and-cmdvel.md, 02-nav2-bringup-lifecycle-actions.md, 01-nav2-system-architecture.md
**Unlocks:** Faster TF diagnosis, cleaner localization architecture choices, fewer fake planner/controller investigations, better AMR incident triage under drift and jump conditions

---

## Why Should I Care? (Context)

Many Nav2 incidents are blamed on the wrong layer because localization defects surface as navigation symptoms:

1. planner says no path because the robot pose is slightly inside an obstacle
2. controller oscillates because heading estimate jitters
3. costmaps smear observations because transform timing is stale
4. the robot looks fine in `odom` but is globally wrong in `map`
5. AMCL jumps and the team calls it a controller failure

Nav2 does not create localization truth. It consumes the transform and pose contracts you provide.

If those contracts are vague, duplicated, or time-incoherent, the whole navigation stack becomes noisy in ways that look unrelated.

---

## PART 1 — THE FRAME CONTRACTS NAV2 EXPECTS

---

## 1.1 The Core Chain

The usual contract is:

```text
map -> odom -> base_link -> sensor frames
```

Nav2 depends on each link meaning something specific.

| Frame | Meaning | What should be true |
| --- | --- | --- |
| `map` | globally anchored world frame | stable against the map or global reference |
| `odom` | locally smooth motion frame | continuous, drift allowed, no sudden jumps |
| `base_link` | robot body frame | immediate pose of the robot body |

If your system uses those names but not those meanings, debugging becomes much harder than it should be.

---

## 1.2 Why `map` and `odom` Must Stay Distinct

`map` and `odom` solve different problems.

`odom` should be smooth enough for control.
`map` should be globally corrected enough for navigation over the environment.

That means:

- `odom` may drift gradually without breaking local control immediately
- `map` may jump slightly when localization corrects global error
- controllers care most about smooth local consistency
- planners and global costmaps care about global correctness

When teams collapse these concepts into one frame, they usually trade away either control stability or global correctness.

---

## 1.3 The Usual Publisher Responsibilities

Common AMR arrangement:

```text
wheel odom + IMU + maybe other local sensors
    -> robot_localization EKF
    -> publishes odom -> base_link

AMCL on static map
    -> publishes map -> odom
```

This is not the only legal architecture, but it is common because it preserves the separation between local smoothness and global correction.

---

## PART 2 — ODOMETRY, IMU, EKF, AND AMCL ROLES

---

## 2.1 Wheel Odometry

Wheel odometry provides short-horizon motion continuity.

Strengths:

- high-rate local motion estimate
- good short-term smoothness
- essential for local control continuity

Weaknesses:

- drift accumulates
- slip and floor conditions corrupt it badly
- global truth is absent

Nav2 can operate with wheel odometry for local behavior, but not reliably for global map navigation on a production AMR without some form of global correction.

---

## 2.2 IMU

IMU mainly helps stabilize rotational estimation and short-term motion dynamics.

Useful contributions:

- better heading continuity
- less dependence on wheel-only yaw inference
- improved local estimator robustness during acceleration or mild slip

Common mistakes:

- fusing low-quality IMU orientation without understanding bias and mounting
- assuming IMU makes global localization accurate
- ignoring timestamp quality and then blaming Nav2 for TF lag

IMU improves local estimation. It does not replace a global localization source.

---

## 2.3 `robot_localization` EKF or UKF

`robot_localization` usually owns the local fused estimate.

Typical responsibilities:

- combine wheel odometry and IMU
- provide a smooth `odom -> base_link` transform
- publish consistent velocity and pose estimates for downstream consumption

What it should not do casually:

- compete with another node for the same TF edge
- publish a globally corrected frame without the team understanding the consequences

Duplicate TF publishers are one of the fastest ways to create non-repeatable Nav2 behavior.

---

## 2.4 AMCL

AMCL usually owns the global correction against a known 2D map.

It answers:

```text
where is the robot in the map, given sensor observations and a prior pose estimate?
```

Operationally, AMCL should correct global drift while preserving the local smoothness contract through the `map -> odom` edge.

If AMCL is unstable, Nav2 may:

- replan from implausible starts
- see goals or obstacles in the wrong place
- trigger recoveries that look irrational

---

## PART 3 — WHAT NAV2 NEEDS FROM LOCALIZATION

---

## 3.1 Correctness First, Smoothness Second, but Both Matter

Nav2 needs both:

- global enough correctness to make map-level planning meaningful
- local enough smoothness to make controller behavior stable

If global correctness is missing, the planner lies.
If smoothness is missing, the controller panics.

That is why `map` and `odom` must cooperate instead of competing.

---

## 3.2 Transform Availability and Timing

A transform that exists eventually is not good enough.

Nav2 needs transforms that are:

- available when requested
- stamped consistently with sensor and costmap data
- recent enough for the configured tolerances

If transform timing is poor, you will see symptoms such as:

- costmap update warnings
- controller lookup failures
- delayed obstacle insertion
- intermittent planner or behavior failures

This often looks random because the data is present, just not aligned in time.

---

## 3.3 Covariance Still Matters Indirectly

Nav2 does not reason about covariance in the same deep way a localization stack does, but covariance quality still matters because it signals whether pose confidence is believable.

When localization becomes noisy or unstable, downstream effects often include:

- twitchy goal convergence
- inconsistent path tracking
- recovery loops after apparent near-success

A pose that is smooth but wrong is often more dangerous operationally than one that is obviously broken.

---

## PART 4 — FAILURE PATTERNS THAT MASQUERADE AS NAV2 PROBLEMS

---

## 4.1 `odom` Is Smooth but Globally Wrong

This is a classic warehouse failure mode.

Symptoms:

- local controller looks stable
- global planner routes around obstacles that are not really where the robot is
- robot appears shifted relative to racks or lanes
- docking approaches miss consistently by a fixed bias

Typical causes:

- AMCL not correcting enough
- poor initial pose or relocalization handling
- map mismatch versus environment reality

This is not a controller-tuning issue.

---

## 4.2 `map -> odom` Jumps

Sudden AMCL correction or unstable global localization can create jumps.

Downstream symptoms:

- path appears to move under the robot
- controller issues abrupt corrections
- costmap alignment looks momentarily wrong
- operators say the robot "twitched" or "teleported" in RViz

If jumps are frequent, first diagnose localization stability and observation quality before touching controller settings.

---

## 4.3 Wrong Frame IDs or Duplicate Publishers

Two common integration bugs:

1. the right data is published under the wrong frame name
2. two different nodes publish the same TF edge

Consequences include:

- intermittent transform tree inconsistency
- behavior that changes between launches
- costmap insertion in the wrong place
- impossible-to-reproduce planner or controller failures

If the same run behaves differently after restart, inspect TF ownership immediately.

---

## 4.4 Timestamp Drift and Stale TF

Transforms can be logically correct and still operationally unusable.

Watch for:

- sensor timestamps from unsynchronized clocks
- delayed odometry publication
- transform tolerances too strict for actual latency
- simulated time misconfiguration

The controller and costmaps are especially sensitive to this because they run continuously and expect recent transforms.

---

## PART 5 — AMCL VS EKF: WHAT PROBLEM EACH ONE SOLVES

---

## 5.1 AMCL Does Not Replace the Local Estimator

AMCL is not your main short-horizon motion estimator.

It is a global localization component against a map.

If you try to treat AMCL as the whole localization stack for a production AMR without understanding the smoothness implications, local control quality usually suffers.

---

## 5.2 EKF Does Not Replace Global Localization

A local EKF can be beautifully smooth and still wrong over time.

That means:

- aisle tracking can look fine for a while
- docking offset can grow slowly
- planner start pose can drift relative to map obstacles

This is why AMCL plus local filtering is such a common pairing.

---

## 5.3 Selection Checklist

Use a local fusion source such as `robot_localization` when:

- you need smooth `odom -> base_link`
- wheel odometry alone is not stable enough
- IMU helps rotational continuity materially

Use AMCL when:

- you have a trustworthy static map
- lidar or matching observations support 2D global localization
- global drift matters for missions, docking, and route reliability

If you do not have a good map-matching story, do not pretend Nav2 will make one up.

---

## PART 6 — A FIELD DIAGNOSTIC FLOW FOR LOCALIZATION CONTRACTS

---

## 6.1 Start With TF Semantics, Not With Parameters

When Nav2 looks confused, verify:

1. who publishes `odom -> base_link`
2. who publishes `map -> odom`
3. whether those publishers are unique and intentional
4. whether timestamps are recent and coherent
5. whether the pose makes physical sense in RViz on the map

This is faster than reading parameter files blindly.

---

## 6.2 Then Separate Global and Local Symptoms

Ask two questions:

```text
Is local motion tracking smooth?
Is global map alignment correct?
```

That gives you a quick classification:

| Local smoothness | Global correctness | Likely issue |
| --- | --- | --- |
| good | bad | AMCL or map alignment problem |
| bad | good | local estimator, IMU, or timing problem |
| bad | bad | broader localization or TF integration failure |

This table is far more useful in incidents than memorizing package names.

---

## 6.3 Only Then Revisit Planner or Controller Complaints

If localization contracts are bad, planner and controller complaints are downstream artifacts.

Do not tune around them.

Examples of bad compensations:

- inflating goal tolerances to hide pose jitter
- weakening obstacle logic to hide map misalignment
- changing controller aggressiveness to mask stale transforms

Those changes usually create new failures elsewhere.

---

## PART 7 — AMR-SPECIFIC DECISIONS AND FAILURE CHECKLISTS

---

## 7.1 Practical Rules for Warehouse AMRs

1. keep `odom -> base_link` smooth enough for control even when global localization is imperfect
2. keep `map -> odom` globally meaningful even if it corrects drift slowly
3. never allow ambiguous TF ownership
4. validate time synchronization before blaming Nav2
5. test docking and narrow-aisle behavior after any localization change, not just open-space navigation

---

## 7.2 Fast Failure Checklist

If the robot replans forever:

- check `map` alignment first

If the robot oscillates near the goal:

- check heading stability and transform freshness first

If the robot cannot pass a corridor that looks open:

- check whether the start pose and obstacle positions are globally aligned first

If RViz looks different every launch:

- check TF publishers first

---

## 7.3 Senior Interview Version

A strong answer explains why:

- `odom` must be smooth
- `map` must be globally meaningful
- AMCL and EKF solve different problems
- duplicate TF publishers are catastrophic
- stale transform timing can look like planner or controller instability

If you can explain those cleanly, you understand the real Nav2 localization contract.

---

## Next Lesson

Continue to 08-nav2-recoveries-progress-and-goal-checkers.md.
That lesson turns controller and localization failure symptoms into explicit retry, recovery, and escalation policy for production AMRs.
