# 12 — Nav2 AMR Failure Patterns and Capstone

## How to recognize recurring production failure modes, reason across subsystem boundaries, and run a final capstone-style root-cause analysis like a senior engineer

**Prerequisite:** 11-nav2-debugging-observability-and-bag-analysis.md, 09-nav2-waypoints-docking-zones-and-missions.md, 08-nav2-recoveries-progress-and-goal-checkers.md
**Unlocks:** Faster incident classification, stronger cross-functional debugging judgment, better AMR reliability reviews, and a final integration exercise that tests full-stack Nav2 reasoning

---

## Why Should I Care? (Context)

By the time a team says "Nav2 is unreliable," the real problem is usually one of these:

1. repeated failure patterns were never named clearly
2. field symptoms were treated as unique incidents instead of recurring classes
3. debugging stayed trapped inside one subsystem at a time
4. temporary workarounds replaced root-cause correction
5. nobody owned the final integrated explanation

Production AMR reliability improves when teams learn to classify failures, not just react to them.

This lesson is about the failure patterns that come back again and again in warehouses, factories, and constrained indoor robot deployments.

---

## PART 1 — THE RECURRING FAILURE PATTERN MINDSET

---

## 1.1 Most Field Incidents Are Variations of Known Patterns

A mature engineer does not start from zero every time.

They ask:

- is this a localization-trust problem?
- is this a costmap world-model problem?
- is this a planner validity problem?
- is this a controller execution problem?
- is this a recovery-policy mismatch?
- is this really a mission-ownership issue disguised as navigation?

Naming the class early compresses debugging time.

---

## 1.2 Root Cause Usually Lives One Layer Upstream of the Visible Failure

Common pattern:

- visible symptom: controller aborts
- actual root cause: localization drift or stale local obstacle

Common pattern:

- visible symptom: no path found
- actual root cause: costmap semantics or TF inconsistency

Common pattern:

- visible symptom: repeated recovery exhaustion
- actual root cause: recovery policy matched the wrong failure story

This is why subsystem silo debugging performs badly.

---

## PART 2 — FAILURE PATTERN: LOCALIZATION TRUST BREAKDOWN

---

## 2.1 What It Looks Like

Symptoms:

- robot pose in RViz looks slightly believable but operationally wrong
- final approach behavior degrades near stations or shelves
- recoveries do not help for long
- planner and controller both appear inconsistent across runs

Localization trust breakdown is dangerous because everything downstream inherits it.

---

## 2.2 AMR-Specific Causes

Typical causes in real robots:

- poor wheel odometry on dusty or glossy floors
- IMU alignment or calibration drift
- scan matching degraded by repetitive aisles
- EKF configuration that looks stable in sim but not on hardware
- insufficient feature richness near docks or charging areas

These are rarely fixed by changing recovery count.

---

## 2.3 Senior Response Pattern

Good response:

1. prove localization quality relative to map and physical landmarks
2. inspect where drift starts, not just where abort happens
3. separate global navigation quality from final alignment precision
4. propose the narrowest fix that restores the localization contract

Bad response:

1. loosen goal tolerances
2. add retries
3. declare the controller more robust

---

## PART 3 — FAILURE PATTERN: COSTMAP STORY DOES NOT MATCH REALITY

---

## 3.1 What It Looks Like

Symptoms:

- no-path results in apparently open space
- robot slows or detours irrationally
- local avoidance behaves as if ghosts exist
- costmap clearing seems to help briefly, then issue returns

The world model is lying to navigation.

---

## 3.2 Common Causes

| Cause | Typical AMR effect |
| --- | --- |
| stale obstacle persistence | aisle stays blocked after pallet moved |
| over-aggressive inflation | narrow corridor becomes non-traversable |
| frame mismatch in sensor observations | obstacles appear shifted or delayed |
| keepout or speed-zone configuration error | robot behavior changes by area in confusing ways |

The planner is often behaving correctly under a bad map state.

---

## 3.3 Senior Response Pattern

Senior engineers ask:

- what exactly did the costmap believe at failure time?
- did that belief come from valid, fresh inputs?
- is the issue global, local, or both?
- is the fix perception-side, costmap-side, or semantic-overlay-side?

They do not jump straight to a planner swap.

---

## PART 4 — FAILURE PATTERN: PATH IS VALID BUT MOTION IS NOT

---

## 4.1 What It Looks Like

Symptoms:

- planner produces reasonable paths
- robot hesitates, oscillates, clips corners, or crawls
- progress checker fires even though commands are present
- `cmd_vel` exists but mission progress is poor

This is where controller logic, drivetrain reality, and odometry quality meet.

---

## 4.2 Common Causes

- controller tuning mismatched to robot geometry or drivetrain limits
- base deadband or actuator saturation not reflected in tuning
- velocity smoothing too conservative for narrow aisle transitions
- odometry under-reporting low-speed progress
- controller asked to solve a docking-grade final approach it does not own

---

## 4.3 Senior Response Pattern

Good response:

1. compare planned motion with actual motion
2. compare actual motion with odometry-reported motion
3. inspect progress-checker assumptions against real base behavior
4. decide whether this is controller tuning, drivetrain execution, or ownership mismatch

This is how you avoid endlessly retuning critics while ignoring base limitations.

---

## PART 5 — FAILURE PATTERN: RECOVERY LOOP WITHOUT LEARNING

---

## 5.1 What It Looks Like

Symptoms:

- repeated clear, spin, backup, or wait cycles
- same local geometry repeatedly triggers failure
- robot burns mission time without changing the outcome
- operators describe behavior as "stuck but busy"

This is a policy failure as much as a navigation failure.

---

## 5.2 Why It Happens

Usually because the BT and recovery policy assume the wrong failure story.

Examples:

- true blocked aisle treated as stale costmap noise
- localization confusion treated as local obstacle problem
- docking alignment issue treated as generic navigation failure

More retries on the wrong story usually produce worse operations.

---

## 5.3 Senior Response Pattern

Senior engineers ask:

- what hypothesis did each recovery encode?
- did any recovery materially change the robot's information or geometry?
- when should escalation have happened instead?

Recoveries should buy new information or a new starting state. Otherwise they are theater.

---

## PART 6 — FAILURE PATTERN: MISSION AND NAVIGATION OWNERSHIP CONFUSION

---

## 6.1 What It Looks Like

Symptoms:

- navigation goal technically succeeds but task fails operationally
- docking or pickup sequence blames Nav2 for station-specific logic gaps
- mission layer times out and the team calls it a Nav2 abort
- waypoint execution becomes a hidden state machine for business logic

Not every robot failure is owned by navigation.

---

## 6.2 High-Value Questions

Ask:

1. was the requested goal operationally correct?
2. should this have been standard navigation or a specialized docking workflow?
3. did mission policy prematurely cancel or overconstrain Nav2?
4. was downstream task logic depending on pose precision beyond the stated navigation contract?

These questions often save weeks of misdirected tuning.

---

## PART 7 — FAILURE PATTERN: CONFIGURATION DRIFT ACROSS ROBOTS OR SITES

---

## 7.1 What It Looks Like

Symptoms:

- one robot class works, another does not
- one site behaves well, another becomes fragile
- sim and hardware disagree in ways that feel mysterious
- nobody knows which parameter layer is actually active

This is not just a debugging annoyance. It is an operational risk.

---

## 7.2 Typical Causes

- hidden launch overrides
- copied YAML with partial edits
- site-specific rules embedded in shared defaults
- temporary debug parameters that never expired
- plugin or BT variants drifting without documentation

---

## 7.3 Senior Response Pattern

Good response:

1. compare effective configuration, not intended configuration
2. isolate what differs by robot, site, and runtime mode
3. restore ownership boundaries in parameter layering and launch structure

Unowned configuration entropy is a root cause category of its own.

---

## PART 8 — CAPSTONE: FINAL ROOT-CAUSE ANALYSIS EXERCISE

---

## 8.1 Scenario

You are on call for a warehouse AMR fleet.

A robot repeatedly fails when sent from a staging lane to a charging dock during the afternoon shift.
Operators report:

- robot navigates most of the route correctly
- near the dock zone it slows, oscillates, and sometimes clears costmaps
- on some runs it aborts with no progress
- on other runs it ends near the dock but not well enough for charging contact
- issue appears worse when traffic is high and pallets are temporarily staged nearby

Known context:

- map includes keepout and speed-zone semantics near the dock corridor
- localization is AMCL with wheel odom and IMU
- controller settings were recently shared across two robot variants
- mission layer still treats docking as a normal goal plus a charging command

---

## 8.2 What a Strong Capstone Answer Should Do

Your analysis should:

1. separate symptom from hypothesis
2. identify the likely failure classes involved
3. define the first evidence you would collect
4. explain which layers own which parts of the problem
5. propose a root-cause ranking, not just one guess
6. describe the safest validation path for any proposed fix

This is not a trivia exercise. It is an engineering judgment exercise.

---

## 8.3 Example of a Weak Answer

```text
Increase recovery count, lower controller aggressiveness, and try a different planner.
```

Why it is weak:

- it does not classify the failure
- it collects no evidence
- it mixes multiple layers blindly
- it could mask the problem without solving it

---

## 8.4 Example of a Stronger Answer Outline

```text
Likely interacting failure classes:
1. docking ownership mismatch
2. local costmap or semantic-zone disturbance near the dock corridor
3. robot-variant-specific low-speed execution differences

First evidence:
- bag around the final 30 to 60 seconds of approach
- TF freshness and AMCL stability near dock zone
- local costmap state and obstacle persistence
- action feedback, cmd_vel, and post-safety velocity execution
- effective parameter diff between working and failing robot variants

Preliminary judgment:
- generic navigation is being asked to finish a docking-grade alignment
- shared controller settings may be invalid for one robot variant at low speed
- temporary nearby pallets may be polluting the local costmap or creating recoverable-but-misclassified obstruction
```

That answer is stronger because it structures the investigation before prescribing a fix.

---

## 8.5 Capstone Deliverable Template

Use this template for the final exercise:

| Section | What to include |
| --- | --- |
| symptom summary | what happened, when, under what mission context |
| subsystem evidence | logs, TF, localization, costmaps, planner, controller, BT, mission |
| failure classes | recurring pattern names that apply |
| root-cause ranking | most likely to least likely with evidence |
| fix proposal | narrow, ownership-correct corrective actions |
| validation plan | replay, sim, and field confirmation steps |
| regression guard | what should be monitored so this does not silently return |

This is the kind of writeup senior engineers are expected to produce.

---

## PART 9 — WHAT SENIOR ENGINEERS DO DIFFERENTLY

---

## 9.1 They Classify Faster Without Overclaiming

Senior engineers often recognize the pattern quickly, but they still preserve rigor.

They say:

- this smells like a world-model problem
- this looks like localization trust breakdown near final approach
- this may be configuration drift across robot variants

They do not say:

- definitely the planner
- definitely a bad controller

Pattern recognition should speed investigation, not replace evidence.

---

## 9.2 They Protect Ownership Boundaries

Good seniors keep asking:

- should this be solved in mission logic?
- should this be expressed as map semantics?
- should this be a controller or progress-checker change?
- should docking remain outside generic goal navigation?

This prevents the codebase from degrading into layer confusion.

---

## 9.3 They Improve the System After the Incident

The best response to a major navigation incident is not only a patch.

It is also:

- better bag capture defaults
- better dashboards or RViz views
- better parameter ownership
- better incident templates
- better distinction between mission and navigation contracts

That is how reliability compounds.

---

## Quick Recap

- recurring failure classes speed debugging when used carefully
- visible symptoms often sit downstream of the real cause
- production AMR failures often blend localization, world-model, control, and ownership issues
- the capstone is about evidence-backed reasoning across the whole stack
- strong engineers produce ranked hypotheses, not random tuning bundles

---

## Next Lesson

Continue to 13-nav2-senior-interview-questions.md
