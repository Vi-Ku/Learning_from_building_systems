# 13 — Nav2 Senior Interview Questions

## A production-oriented interview bank for AMR navigation engineers, with answer guidance that tests systems judgment instead of memorized ROS 2 vocabulary

**Prerequisite:** 11-nav2-debugging-observability-and-bag-analysis.md, 12-nav2-amr-failure-patterns-and-capstone.md, 01-nav2-system-architecture.md
**Unlocks:** Better self-assessment, stronger interview preparation, sharper explanations of Nav2 tradeoffs, and a clearer picture of what senior-level ownership actually sounds like

---

## Why This Lesson Exists

Many Nav2 interviews stay too shallow.

They ask:

- what is a costmap?
- what is TF?
- what is AMCL?

Those questions matter, but they do not reliably identify someone who can own an AMR navigation stack in production.

Senior-level interviews should test:

- system decomposition
- debugging discipline
- tradeoff reasoning
- operational judgment
- ability to separate navigation from adjacent layers
- ability to explain failures precisely

This bank is organized around those goals.

---

## How to Use This Interview Bank

For each question:

- focus on structure before detail
- define ownership boundaries clearly
- use AMR-specific examples when helpful
- explain tradeoffs, not just the happy path
- mention failure modes and observability where relevant

Strong answers do not need to be long, but they should be ordered, concrete, and defensible.

---

## CATEGORY 1 — SYSTEM ARCHITECTURE AND OWNERSHIP

---

## 1. Walk me through a `NavigateToPose` request end to end in Nav2

**What the interviewer is testing:** whether you can explain the runtime flow without vague hand-waving.

**Strong answer guidance:**

- start from the action client or mission layer
- explain `bt_navigator` as the orchestration layer
- describe planner, controller, behavior server, costmaps, and lifecycle manager roles
- explain how TF, localization, and sensor data are upstream dependencies rather than Nav2-owned internals
- end at `cmd_vel` generation and result reporting

**What a weak answer sounds like:**

- listing node names without explaining interactions
- pretending Nav2 owns localization directly
- skipping BT policy and recoveries entirely

---

## 2. What does Nav2 own, and what should remain outside Nav2 in a production AMR stack?

**What the interviewer is testing:** whether you can maintain clean boundaries under real product pressure.

**Strong answer guidance:**

- Nav2 owns navigation orchestration, planning, local control, and bounded recovery behavior
- localization, perception, mission logic, fleet coordination, and docking-specific business workflows often sit adjacent to Nav2
- explain that some responsibilities are shared through contracts rather than hard boundaries
- give examples of mission-layer concerns that should not be pushed into BT XML or planner tuning

**Strong signal:** explicitly mention that docking, task execution, and fleet dispatch are often where ownership confusion starts.

---

## 3. When would you solve a problem with parameters, BehaviorTree XML, or a plugin?

**What the interviewer is testing:** whether you know how to choose the right layer.

**Strong answer guidance:**

- parameters for tuning an already-correct mechanism
- BT XML for policy and sequencing changes
- plugin code when the mechanism itself is insufficient
- mention maintainability and blast radius, not just technical possibility
- mention testing expectations before introducing a custom plugin

**Weak answer pattern:** saying "I would write a plugin" whenever defaults are inconvenient.

---

## CATEGORY 2 — LOCALIZATION, TF, AND WORLD MODEL CONTRACTS

---

## 4. A robot keeps reaching near the goal but behaves inconsistently in the final meter. How do you reason about whether this is localization, controller tuning, or goal-checker configuration?

**What the interviewer is testing:** whether you can separate interacting layers instead of guessing.

**Strong answer guidance:**

- begin with pose trust near the goal and compare map estimate to physical reality
- then inspect controller behavior and actual base response
- finally inspect whether goal success criteria match the operational task
- explain that final-meter issues are often docking-grade or staging-grade problems, not generic navigation alone

**Good bonus point:** mention that loosening tolerance can shift failure downstream instead of solving it.

---

## 5. What TF problems commonly masquerade as planner or controller problems?

**What the interviewer is testing:** whether you debug upstream contracts first.

**Strong answer guidance:**

- stale transforms
- wrong frame ownership or naming
- inconsistent timestamps
- static transform mistakes that shift perception or footprint alignment
- explain how these surface as bad costmaps, oscillation, or false no-path outcomes

**Weak answer pattern:** treating TF as setup boilerplate instead of a live runtime dependency.

---

## 6. Why can a planner be correct when operators insist the aisle looks open?

**What the interviewer is testing:** whether you understand world-model truth vs human visual intuition.

**Strong answer guidance:**

- explain that planners operate on the current costmap and footprint constraints
- mention stale obstacles, inflation effects, unknown space, semantic overlays, and sensor frame issues
- emphasize that the right question is what the planner believed, not what a human remembers seeing

---

## CATEGORY 3 — CONTROLLERS, MOTION QUALITY, AND EXECUTION REALITY

---

## 7. A planner produces good paths, but the robot oscillates or crawls. What do you inspect first?

**What the interviewer is testing:** whether you can bridge software output and physical execution.

**Strong answer guidance:**

- compare path geometry, controller commands, actual motion, and odometry feedback
- inspect progress-checker assumptions and base deadband
- distinguish between controller tuning issues and drivetrain execution limits
- mention narrow-aisle or final-approach context if relevant

**Strong answer tone:** methodical and comparative, not "I would change critic weights first."

---

## 8. How would you choose between DWB and Regulated Pure Pursuit for an indoor AMR?

**What the interviewer is testing:** whether you can reason in tradeoffs rather than brand loyalty.

**Strong answer guidance:**

- discuss path tracking style, smoothness, compute needs, geometry, and operational constraints
- mention how aisle width, desired motion behavior, and robot kinematics influence the choice
- state that controller choice should be validated against real site geometry and task patterns

**Weak answer pattern:** presenting one controller as universally best.

---

## 9. Why can `cmd_vel` presence still be consistent with a navigation failure?

**What the interviewer is testing:** whether you understand downstream execution and observability.

**Strong answer guidance:**

- commands may be too small to overcome deadband
- downstream safety layers may gate them
- odometry may not reflect actual motion correctly
- the robot may move, but not enough to satisfy progress logic

This answer gets stronger if you explicitly separate command intent from executed motion.

---

## CATEGORY 4 — RECOVERIES, BEHAVIOR TREES, AND FAILURE POLICY

---

## 10. How do you decide whether a recovery policy is well-designed?

**What the interviewer is testing:** whether you see recoveries as operational policy, not random retries.

**Strong answer guidance:**

- each recovery should reflect a plausible failure story
- each retry should buy either new information or a meaningfully changed state
- retry budget should match the environment and operational cost of delay
- escalation should happen when repeated actions are not changing the situation

**Great answer detail:** mention that endless busy behavior destroys operator trust faster than fast escalation.

---

## 11. When would you modify the default Nav2 BT tree for a warehouse robot?

**What the interviewer is testing:** policy reasoning and BT literacy.

**Strong answer guidance:**

- when retry ordering or gating must match operational conditions better
- when blocked-aisle handling differs from default assumptions
- when mission-specific conditions need safe interaction with navigation
- explain why BT changes are policy changes, not tuning changes

**Weak answer pattern:** editing the tree without a clear failure story or test plan.

---

## 12. A robot repeatedly clears costmaps and spins with no success. What does that tell you?

**What the interviewer is testing:** whether you can interpret the behavior policy outcome.

**Strong answer guidance:**

- recoveries are likely assuming the wrong failure story
- possible deeper issues include localization breakdown, persistent true blockage, or ownership mismatch near docking/task logic
- explain that the incident should be reclassified rather than given more retries

---

## CATEGORY 5 — DEBUGGING, OBSERVABILITY, AND INCIDENT RESPONSE

---

## 13. What is your first-response workflow when an AMR navigation incident happens in production?

**What the interviewer is testing:** whether you can operate under pressure without random tuning.

**Strong answer guidance:**

- record mission context and exact timestamps
- collect logs and bag data
- verify lifecycle and action state
- inspect TF, localization, costmaps, planner, controller, and recovery story in a fixed order
- produce a ranked hypothesis list backed by evidence

**Strong signal:** the answer has a repeatable sequence.

---

## 14. What data would you want in a rosbag for a Nav2 incident, and why?

**What the interviewer is testing:** whether you understand offline debugging requirements.

**Strong answer guidance:**

- TF and TF static
- odometry and localization outputs
- costmap-relevant sensor topics
- action goal, feedback, and result
- `cmd_vel` and any downstream gated command topic
- contextual mission or diagnostic topics

Then explain what questions those data streams answer.

---

## 15. How do you distinguish root cause from downstream fallout in logs?

**What the interviewer is testing:** disciplined reasoning from timelines.

**Strong answer guidance:**

- read logs chronologically
- find the last normal state and first meaningful anomaly
- rank messages by subsystem upstreamness
- do not let the final abort message dominate the diagnosis

This is a strong senior signal because many engineers never learn to read logs as causality.

---

## CATEGORY 6 — PRODUCTION TUNING, ROLLOUTS, AND CONFIGURATION CONTROL

---

## 16. How do you structure Nav2 parameters so changes stay safe across robot variants and sites?

**What the interviewer is testing:** whether you can run a maintainable navigation program, not just a demo stack.

**Strong answer guidance:**

- layer by ownership: platform, robot variant, site, runtime mode, temporary override
- avoid mixing geometry, localization, and site semantics casually
- explain how to compare effective configuration, not just source files
- mention rollback discipline and targeted rollout

---

## 17. A tuning change improves one robot but breaks another. How do you respond?

**What the interviewer is testing:** configuration ownership discipline.

**Strong answer guidance:**

- assume the shared-vs-specific boundary is wrong until proven otherwise
- compare effective config and physical robot differences
- restore proper layering before inventing more exceptions
- explain how you would validate the corrected layering

---

## 18. How would you safely roll out a major navigation change?

**What the interviewer is testing:** operational maturity.

**Strong answer guidance:**

- verify in bag replay or simulation where possible
- test against representative scenarios: open aisle, tight aisle, final approach, dynamic obstruction
- limit rollout to one site or robot subset first
- monitor logs, recoveries, success rate, and operator feedback
- keep rollback simple and fast

Senior answers make rollout sound controlled, not heroic.

---

## CATEGORY 7 — AMR-SPECIFIC DESIGN JUDGMENT

---

## 19. Why is docking often a bad fit for plain `NavigateToPose` all the way to contact?

**What the interviewer is testing:** whether you understand navigation vs specialized final behavior.

**Strong answer guidance:**

- docking often needs tighter final tolerances, alignment behavior, and station-specific confirmation
- generic navigation may be appropriate to a staging pose, but not necessarily for the final contact routine
- explain the benefit of clean ownership between approach navigation and docking-specific logic

---

## 20. When should keepout zones, speed zones, or route semantics live in map-linked navigation configuration instead of mission logic?

**What the interviewer is testing:** whether you can place geographic policy in the right layer.

**Strong answer guidance:**

- when the rule is spatial, repeatable, and shared across many missions
- when operators and tooling need one visible truth for the environment
- when the behavior should be consistent regardless of which workflow invoked navigation

This gets stronger if you explicitly warn against duplicating geographic rules in business logic.

---

## 21. A team says, "Nav2 is unreliable in our warehouse." What questions do you ask first?

**What the interviewer is testing:** executive-level diagnostic framing.

**Strong answer guidance:**

- what failure classes dominate: no path, oscillation, final approach aborts, recovery exhaustion, localization drift?
- where and when do incidents cluster?
- what changed recently in maps, parameters, robot hardware, or mission flows?
- what evidence exists: bags, logs, metrics, operator reports?

The best answers show you know how to turn a vague complaint into an engineering program.

---

## CATEGORY 8 — CAPSTONE QUESTIONS

---

## 22. Give me a root-cause ranking for this scenario: the robot navigates most routes well, but afternoon dock approaches near staged pallets frequently fail with oscillation and occasional recovery exhaustion

**What the interviewer is testing:** integrated reasoning across the whole stack.

**Strong answer guidance:**

- offer a ranked list, not a single guess
- likely include local costmap pollution near the dock, docking ownership mismatch, localization quality near final approach, or robot-specific low-speed execution issues
- explain what evidence would reorder the ranking
- describe the safest validation approach for each candidate

---

## 23. Tell me about a time you would reject a proposed fix even if it appears to improve success rate

**What the interviewer is testing:** engineering integrity.

**Strong answer guidance:**

- examples: loosening goal tolerances, hiding localization problems with retries, widening keepout exceptions, masking drift with forceful recoveries
- explain why local success-rate gain may damage operational correctness or debuggability
- show willingness to defend system quality over easy optics

---

## 24. What does senior ownership of Nav2 look like beyond writing code?

**What the interviewer is testing:** leadership and systems maturity.

**Strong answer guidance:**

- defining ownership boundaries
- setting observability expectations
- standardizing incident debugging workflows
- maintaining safe parameter and rollout discipline
- mentoring others away from cargo-cult tuning
- improving documentation and failure taxonomy after incidents

This answer often separates senior from merely experienced.

---

## How Interviewers Can Use This Bank

If you are interviewing someone:

- start with system flow and ownership
- move into debugging and failure classification
- use one or two capstone scenarios to test synthesis
- push for tradeoffs and evidence, not slogans
- reward clarity, structure, and operational realism over trivia density

The strongest Nav2 engineers usually sound calm, specific, and hard to confuse.

---

## Final Self-Assessment Prompt

After working through this track, you should be able to answer:

```text
If a warehouse AMR fails a goal, can I explain the likely failure class,
the evidence I need, the ownership boundary involved, and the safest fix path
without defaulting to random parameter changes?
```

If the answer is yes, you are thinking much closer to senior level.

---

## Track Wrap-Up

This completes the planned Nav2 track from architecture through production ownership.

The real test after this point is not whether you can define Nav2 terms.
It is whether you can debug, tune, extend, and defend navigation decisions in a real AMR system with evidence and good boundaries.
