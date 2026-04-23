<!-- markdownlint-disable MD033 -->

# Exercise07 — Nav2 Recovery Policy Design

## Companion exercises for `../03-nav2-bt-navigator-and-bt-xml.md`, `../08-nav2-recoveries-progress-and-goal-checkers.md`, and `../12-nav2-amr-failure-patterns-and-capstone.md`

**Estimated time:** 100 to 125 minutes  
**Prerequisite lessons:** ../03-nav2-bt-navigator-and-bt-xml.md, ../06-nav2-local-control-and-cmdvel.md, ../08-nav2-recoveries-progress-and-goal-checkers.md, ../12-nav2-amr-failure-patterns-and-capstone.md

**Mode options:**

- **Simulation:** trigger blocked-path and transient-obstacle scenarios in a Nav2 world and observe recovery behavior sequencing.
- **Bag replay and logs:** infer which recoveries were attempted and whether they matched the underlying failure mode.
- **Design review:** complete the exercise as a policy and BehaviorTree review for an AMR team without running hardware.

**Validation goal:** by the end of this lab you should be able to design a recovery policy that is safe, observable, and aligned with actual AMR failure modes instead of simply retrying until operations lose trust.

---

## Overview

Most bad recovery policies fail in one of two ways:

1. they retry the same losing action until the mission times out
2. they perform aggressive maneuvers that are technically valid but operationally reckless

For AMRs in warehouses or factories, recoveries are not just a Nav2 concern. They are a product policy. They decide when to retry, when to back up, when to clear costmaps, when to wait for humans, and when to escalate to the mission layer.

This lab forces you to design recoveries from symptoms, safety constraints, and observability requirements.

---

## Section A — Failure Mode Classification

Classify each incident by the first recovery category you would consider.

| Incident | First category to consider | Why |
| --- | --- | --- |
| aisle blocked by a pallet that moved in unexpectedly | ? | ? |
| local costmap appears polluted by a transient ghost obstacle | ? | ? |
| robot cannot rotate cleanly because of tight footprint near staging rack | ? | ? |
| localization confidence suddenly degrades mid-mission | ? | ? |

Choose from categories such as: wait-and-retry, clear local perception state, replan, controlled backup or reposition, localization revalidation, or mission escalation.

<details><summary>Answer guidance</summary>

The main lesson is that different root causes deserve different first moves:

- a real pallet obstruction may justify a bounded wait, replan, or mission-level reroute
- ghost obstacles may justify perception-state cleanup only if evidence supports it
- tight-footprint rotation problems may require controlled repositioning rather than repeated spin attempts
- degraded localization should usually change trust and mission state, not merely trigger motion retries

</details>

- [ ] Done

---

## Section B — Critique an Over-Retry Policy

Read the proposed policy below.

```text
On any FollowPath failure:
1. clear both costmaps
2. retry FollowPath immediately
3. if it fails again, back up 0.10 m
4. retry FollowPath immediately
5. repeat this sequence up to 8 times
6. if still failing, abort mission
```

**Questions:**

1. What is operationally weak about this policy?
2. Which failure modes might it accidentally make worse?
3. What observability is missing if this policy runs in production?

<details><summary>Answer guidance</summary>

Strong answers should mention that the policy treats all failures as equivalent, lacks branching by cause, and may erase useful evidence. It can worsen blocked-aisle congestion, localization uncertainty, and operator confusion. It also fails to expose whether the issue was perception, control, planning, or localization related.

</details>

- [ ] Done

---

## Section C — Design a Tiered Recovery Policy

You are designing for a differential-drive AMR in narrow warehouse aisles.

### Task C1 — Fill the Policy Table

| Failure signature | First recovery action | Second action if first fails | Escalation rule |
| --- | --- | --- | --- |
| temporary human obstruction with valid localization | ? | ? | ? |
| repeated no-valid-control with smooth odometry but tight local geometry | ? | ? | ? |
| planner failure with obviously blocked aisle | ? | ? | ? |
| pose jump or localization confidence drop | ? | ? | ? |

<details><summary>Answer guidance</summary>

Good designs usually keep the first action low-risk and evidence-aligned:

- temporary human obstruction: short wait then bounded replan or mission pause
- no-valid-control in tight geometry: small controlled reposition or backing maneuver after checking footprint and local costmap realism
- planner failure in a truly blocked aisle: wait, reroute, or mission-layer dispatch change rather than endless retries
- localization trust drop: pause or slow, revalidate localization, then escalate if trust does not recover

</details>

- [ ] Done

---

### Task C2 — Retry Budget Selection

For each category below, choose a retry philosophy and justify it.

| Category | Suggested retry philosophy | Why |
| --- | --- | --- |
| human crossing or forklift passing nearby | ? | ? |
| local sensor dropout for less than 2 seconds | ? | ? |
| repeated collision-risk backup in a crowded aisle | ? | ? |
| localization reset request | ? | ? |

Possible philosophies:

- many cheap retries are acceptable
- few retries with strong observability
- one retry only, then escalate
- no autonomous retry without higher-level approval

<details><summary>Answer guidance</summary>

The answer should show operational judgment rather than generic optimism. Cheap retries are acceptable when the environment is clearly transient and safety is preserved. High-motion recoveries in crowded aisles usually deserve a much stricter budget. Localization reset actions often require explicit mission or operator involvement because they change trust in the robot's state.

</details>

- [ ] Done

---

## Section D — BehaviorTree Decision Review

Assume your BT currently does this:

```text
NavigateToPose
  -> ComputePathToPose
  -> FollowPath
  -> on failure: ClearEntireCostmap
  -> Retry NavigateToPose
```

**Questions:**

1. What important diagnostic branching is missing from this tree?
2. What extra condition or subtree would you add for localization-related failures?
3. What extra condition or subtree would you add for blocked-aisle cases?

<details><summary>Answer guidance</summary>

The missing branch is cause-aware handling. A strong answer suggests separate conditions for localization trust, persistent obstacle presence, or controller-progress failure. The goal is not to write a perfect BT from memory; it is to show that clear-costmap-and-retry is not a universal policy.

</details>

- [ ] Done

---

## Section E — Recovery Observability Design

Define the minimum telemetry you would log or surface whenever a recovery is triggered.

Your answer must include:

1. one root-cause hint
2. one state snapshot from Nav2
3. one mission-layer context field
4. one operator-facing label that is not misleading

Then answer these questions:

1. Why is `recovery_executed=true` useless by itself?
2. What would you archive from bag replay or logs after a severe recovery sequence?

<details><summary>Answer guidance</summary>

Good answers include things like controller failure reason, costmap or TF snapshot, active mission step, and a label such as `blocked_aisle_waiting` or `localization_revalidation_required`. Evidence archives often include the triggering log window, costmap state, TF health, and mission context.

</details>

- [ ] Done

---

## Section F — Write a Recovery Policy Memo

Write a concise policy memo for operations and software teams that covers:

1. when the robot is allowed to recover autonomously
2. when it should pause and wait
3. when it must escalate to the mission layer or operator
4. what evidence should appear in the incident ticket after an abort

Target length: 10 to 14 lines.

<details><summary>Answer guidance</summary>

The best memos balance autonomy with trust. They explicitly limit aggressive retries, distinguish transient obstructions from state-trust failures, and require evidence capture instead of "the robot got stuck" summaries.

</details>

- [ ] Done

---

## Section G — AMR Production Reflection

Answer briefly but concretely.

**G1.** Why is a recovery policy partly a product decision and not just a BT implementation detail?

**G2.** Which two recovery actions would you treat as highest risk in a crowded warehouse, and why?

**G3.** If you could add only one dashboard view for recoveries, what would it show?

<details><summary>Answer guidance</summary>

Strong answers mention throughput, safety, operator trust, and the need to understand not just whether a recovery ran but whether it matched the actual failure mode.

</details>

- [ ] Done

---

## Deliverable Template

```text
AMR scenario:

Primary failure modes considered:

Tiered recovery policy:
1.
2.
3.

Retry budgets:
- transient obstruction:
- controller geometry issue:
- localization trust issue:

Mission escalation rules:

Observability fields:
- root cause hint:
- nav state snapshot:
- mission context:
- operator label:

Open risks:
```

---

## Success Criteria

You have completed this lab well if you can:

1. match recovery actions to failure signatures instead of applying one generic retry loop
2. justify retry budgets using safety and operational reasoning, not only technical optimism
3. show where BT branching should reflect localization, geometry, and obstacle differences
4. define telemetry that makes recoveries understandable during incident review
