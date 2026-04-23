<!-- markdownlint-disable MD033 -->

# Exercise09 — Nav2 Plugin Extension Lab

## Companion exercises for `../10-nav2-parameters-launch-and-plugin-extension.md`, `../11-nav2-debugging-observability-and-bag-analysis.md`, and `../12-nav2-amr-failure-patterns-and-capstone.md`

**Estimated time:** 95 to 120 minutes  
**Prerequisite lessons:** ../08-nav2-recoveries-progress-and-goal-checkers.md, ../09-nav2-waypoints-docking-zones-and-missions.md, ../10-nav2-parameters-launch-and-plugin-extension.md, ../11-nav2-debugging-observability-and-bag-analysis.md, ../12-nav2-amr-failure-patterns-and-capstone.md

**Mode options:**

- **Simulation:** compare existing Nav2 behaviors and decide whether parameters or mission logic already solve the problem.
- **Log and bag review:** analyze repeated incidents and decide whether a plugin extension is justified.
- **Architecture review:** complete the exercise as a design decision memo without writing C++ plugin code.

**Validation goal:** by the end of this lab you should be able to decide when Nav2 needs extension, when it only needs configuration, and when the real solution belongs outside Nav2 entirely.

---

## Overview

Many teams reach for custom plugins too early.

That usually creates one of three problems:

1. custom code is written to hide a misunderstood configuration issue
2. mission or product logic is improperly forced into a Nav2 plugin
3. extension points are chosen without a clear validation plan or rollback path

This lab is a decision exercise. The goal is not to implement a plugin but to justify whether one should exist at all.

---

## Section A — Configuration, Mission Layer, or Plugin?

For each requirement, choose the most likely solution home.

| Requirement | Best first solution home | Why |
| --- | --- | --- |
| custom slowdown behavior only inside a narrow map-defined zone | ? | ? |
| business rule that skips low-priority waypoints during a rush order | ? | ? |
| controller needs a new scoring term tied to trailer articulation limits | ? | ? |
| docking approach needs special final-pose acceptance logic not covered by existing checkers | ? | ? |
| operator wants a custom dashboard label when localization trust drops | ? | ? |

<details><summary>Answer guidance</summary>

High-quality answers resist unnecessary extension:

- zone-specific slowdown may already be solvable through zone semantics or route policy
- business-priority logic belongs in the mission layer
- a new scoring term tightly tied to motion evaluation may justify a controller extension
- special goal acceptance or docking-specific behavior may justify a custom checker or dedicated behavior if existing options are insufficient
- dashboard labels are not a plugin problem

</details>

- [ ] Done

---

## Section B — Plugin Proposal Critique

Read the proposal below.

```text
Proposal: create a custom planner plugin because robots sometimes wait too long near the packing zone.

Observed symptoms:
- robots slow correctly in the packing zone
- routes remain valid
- humans often block the lane for 20 to 60 seconds
- mission deadlines are sometimes missed

Rationale:
- a custom planner can make smarter business decisions about urgency
```

**Questions:**

1. Why is a custom planner plugin probably the wrong first move?
2. Which layer actually appears underpowered from this evidence?
3. What evidence would you require before considering any Nav2 extension here?

<details><summary>Answer guidance</summary>

The proposal confuses route validity with business urgency. The planner can already produce a path; the harder problem is policy around waiting, rerouting, or reprioritizing missions. That belongs in mission logic unless there is concrete evidence that route generation itself is deficient.

</details>

- [ ] Done

---

## Section C — Extension Decision Framework

Define a five-question checklist that must be answered before approving a Nav2 plugin extension.

Your checklist must include:

1. proof that configuration was exhausted responsibly
2. proof that the problem really lives inside Nav2
3. measurable success criteria
4. rollout and rollback considerations
5. observability requirements

<details><summary>Answer guidance</summary>

Good answers usually force the team to prove the current behavior is insufficient, prove where the insufficiency lives, define a measurable win, and protect operations with rollback and telemetry.

</details>

- [ ] Done

---

## Section D — Pick the Correct Extension Point

Choose the most plausible extension point for each scenario.

| Scenario | Most plausible extension point | Why |
| --- | --- | --- |
| need a new local trajectory scoring concept | ? | ? |
| need a different criterion for declaring goal reached near a docking target | ? | ? |
| need a new route-level recovery after repeated planner failure in a special area | ? | ? |
| need custom obstacle semantics imported from warehouse operations system | ? | ? |

Possible extension points include controller critics or plugin internals, goal checker, behavior or recovery plugin, costmap layer or semantic input pipeline, or non-Nav2 mission service.

<details><summary>Answer guidance</summary>

The lesson here is precise ownership. A controller critic changes local motion scoring. A goal checker changes completion semantics. Semantic obstacle rules may belong in a costmap layer or upstream semantic map ingestion. Route-level business policy may belong above Nav2 unless the behavior is genuinely navigation-generic.

</details>

- [ ] Done

---

## Section E — Validation Plan Before Code

You are tentatively approving one custom extension for a docking-related goal checker.

### Task E1 — Write the Pre-Code Validation Plan

Your plan must include:

1. a replay or simulation scenario set
2. a baseline using current built-in behavior
3. metrics for success and failure
4. a rollout safety gate

Then answer:

1. What would convince you that the extension should be rejected even after it is implemented?
2. What telemetry would you require in production for the first rollout?

<details><summary>Answer guidance</summary>

Strong answers use before-and-after comparison, not intuition. The safety gate should include measurable improvement and no regression in normal navigation. Production telemetry should include checker outcomes, near-goal behavior, abort reasons, and one mission context field.

</details>

- [ ] Done

---

## Section F — Architecture Memo

Write a concise architecture memo covering:

1. the problem statement
2. why configuration or mission logic is insufficient
3. the chosen extension point
4. how it will be validated and rolled back
5. what evidence must appear in review before merge

Target length: 12 to 16 lines.

<details><summary>Answer guidance</summary>

The best memos read like engineering approval documents, not feature wishlists. They are explicit about boundaries, metrics, and rollback.

</details>

- [ ] Done

---

## Section G — AMR Production Reflection

Answer briefly but concretely.

**G1.** Why is "we can just write a plugin" a dangerous instinct on a navigation team?

**G2.** Which extension points would you treat as highest operational risk, and why?

**G3.** If you could add one mandatory review question before any Nav2 extension lands, what would it be?

<details><summary>Answer guidance</summary>

Strong answers emphasize maintainability, hidden ownership drift, and the tendency for custom plugins to become product-policy dumping grounds unless the review bar is disciplined.

</details>

- [ ] Done

---

## Deliverable Template

```text
Requirement under review:

Current evidence:

Why configuration is or is not enough:

Why mission logic is or is not enough:

Chosen solution home:

If plugin extension:
- extension point:
- success metrics:
- rollout gate:
- rollback plan:
- required telemetry:

If not plugin:
- correct owning layer:

Open risks:
```

---

## Success Criteria

You have completed this lab well if you can:

1. distinguish real Nav2 extension needs from configuration or product-policy problems
2. pick a plausible extension point only after proving the problem belongs there
3. define measurable validation and rollback criteria before code exists
4. frame plugin work as an operationally accountable design choice instead of a convenience shortcut
