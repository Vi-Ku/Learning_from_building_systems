# 03 — Nav2 BT Navigator and BehaviorTree XML

## How Nav2 encodes navigation policy, recovery logic, and runtime decision flow for real AMRs

**Prerequisite:** 01-nav2-system-architecture.md, 02-nav2-bringup-lifecycle-actions.md, ../03-nav2-architecture.md
**Unlocks:** BT debugging, safe tree customization, better recovery design, clearer separation between policy bugs and algorithm bugs

---

## Why Should I Care? (Context)

Two robots can run the same planner and controller plugins yet behave very differently because their BehaviorTrees make different decisions.

That is why BT understanding matters. The tree decides:

1. when to plan
2. when to replan
3. when to keep following versus abort
4. which recovery behavior runs first
5. when a new goal interrupts current execution

If a warehouse AMR spins six times in a blocked aisle before giving up, that is not merely a controller story. It is a policy story encoded in BT XML.

---

## PART 1 — WHAT `bt_navigator` REALLY DOES

---

## 1.1 It Is the Navigation Policy Engine

`bt_navigator` receives high-level navigation actions such as `NavigateToPose` and executes a BehaviorTree that orchestrates lower-level action servers.

Think of it like this:

```text
planner_server      = route computation
controller_server   = path execution
behavior_server     = recovery behaviors
bt_navigator        = policy and sequencing across all of them
```

Without `bt_navigator`, you still have useful primitives, but you do not have a coherent navigation workflow.

---

## 1.2 Why XML Matters

The XML is not just configuration fluff. It is executable policy.

It defines:

- which nodes run in what order
- what blackboard keys they read and write
- how retries are bounded
- whether replanning is periodic or event-driven
- how goal updates short-circuit recoveries

Changing XML can alter field behavior as much as changing controller parameters.

---

## PART 2 — THE CORE BT MENTAL MODEL

---

## 2.1 Return States

Every BT node returns one of three states:

```text
SUCCESS
FAILURE
RUNNING
```

That sounds simple, but most debugging confusion comes from forgetting what `RUNNING` means.

`RUNNING` means the node is still in progress and will be ticked again on later cycles. This is normal for long-running actions like path following or spinning.

---

## 2.2 Common Control Nodes

### `Sequence`

Run children in order. Failure in one child fails the sequence.

### `Fallback` or `ReactiveFallback`

Try children in order until one succeeds.

### `RecoveryNode`

Wrap a primary behavior plus a fallback recovery subtree with a retry budget.

### `PipelineSequence`

Critical in Nav2 because it enables planning and following to coexist over repeated ticks.

### Decorators like `RateController`

Modify how often or under what condition a child is ticked.

---

## 2.3 The Blackboard

The blackboard is the shared runtime key-value space for the tree.

Typical values include:

- `goal`
- `path`
- controller or planner IDs
- flags about updated goals or recovery state

If you misunderstand blackboard ports, you can build a tree that looks structurally valid but passes stale data between nodes.

**Example:** a planner writes a new `path`, but the controller is still wired to an old key or incompatible branch path.

---

## PART 3 — READING THE DEFAULT NAVIGATION FLOW

---

## 3.1 Canonical Pattern

A common Nav2 pattern looks roughly like this:

```xml
<RecoveryNode number_of_retries="6" name="NavigateRecovery">
  <PipelineSequence name="NavigateWithReplanning">
    <RateController hz="1.0">
      <ComputePathToPose goal="{goal}" path="{path}"/>
    </RateController>
    <FollowPath path="{path}" controller_id="FollowPath"/>
  </PipelineSequence>

  <ReactiveFallback name="RecoveryFallback">
    <GoalUpdated/>
    <SequenceWithMemory name="RecoveryActions">
      <ClearEntireCostmap .../>
      <Spin .../>
      <Wait .../>
      <BackUp .../>
    </SequenceWithMemory>
  </ReactiveFallback>
</RecoveryNode>
```

This tells you several important things immediately:

1. planning is periodic, not one-shot
2. path following is the main runtime behavior
3. recoveries are bounded by retry count
4. a new goal can interrupt recovery handling

---

## 3.2 Tick-by-Tick Thinking

You should be able to narrate a few ticks by hand.

```text
Tick 1:
  plan -> SUCCESS
  follow path -> RUNNING

Tick 2:
  planner may be skipped by RateController
  follow path -> RUNNING

Tick N:
  planner replans
  follow path keeps running with updated path
```

If path following fails:

```text
main pipeline -> FAILURE
RecoveryNode enters fallback subtree
GoalUpdated? if yes, switch back toward new goal handling
otherwise run recovery actions in sequence
```

This tick-level reasoning is how you debug real BT behavior.

---

## 3.3 Why `PipelineSequence` Matters

`PipelineSequence` is one of the most operationally important Nav2 control nodes.

It supports the common requirement:

```text
keep following the current path
while occasionally computing a newer better path
```

Without this, you tend to get one of two bad extremes:

- expensive replanning too often and destabilizing execution
- no replanning while the world changes around the robot

For warehouse AMRs in dynamic aisles, replanning cadence is a real throughput and stability decision.

---

## PART 4 — CUSTOMIZATION BOUNDARIES

---

## 4.1 When BT Customization Is the Right Tool

Customize the BT when the problem is about **policy**.

Examples:

- change recovery ordering for blocked aisles
- insert a condition that reacts differently near docking zones
- gate certain recoveries when humans are nearby
- replan more or less aggressively under dynamic obstruction

Do **not** reach for BT XML first when the real issue is:

- planner algorithm mismatch
- bad controller tuning
- stale costmap observations
- localization drift

That is the line between policy and mechanism.

---

## 4.2 A Good Customization Example

Suppose your AMR operates in narrow one-way aisles where backing up is preferable to spinning because spinning increases footprint conflict with pallet edges.

A policy adjustment might be:

1. clear local costmap
2. wait briefly for human traffic to pass
3. back up a small distance
4. only then spin if necessary

That is a BT concern because you are changing *what the robot tries next*.

---

## 4.3 A Bad Customization Example

Suppose the local costmap keeps retaining stale obstacles because sensor clearing settings are wrong.

Replacing the recovery subtree to clear costmaps more often may reduce symptoms, but it does not fix the cause. That is a costmap or perception integration problem.

Overusing BT customization to mask upstream faults produces brittle navigation behavior.

---

## PART 5 — BT NODE BEHAVIOR THAT MATTERS IN PRODUCTION

---

## 5.1 `GoalUpdated`

This condition is crucial for systems with frequent rerouting.

It allows the tree to say, effectively:

```text
if a new goal arrived, stop spending time on the current recovery path
and switch focus to the latest task intent
```

Without this kind of condition, the robot can keep executing outdated recovery logic after the mission layer already changed priorities.

---

## 5.2 `ClearEntireCostmap`

Useful when the world model may contain stale or bad obstacle information.

Dangerous when abused, because repeated clearing can:

- hide real obstacles transiently
- produce unstable behavior in cluttered environments
- turn systematic observation issues into recurring temporary relief

Use it as a recovery tool, not as the main operating mode.

---

## 5.3 `Wait`, `Spin`, `BackUp`

These are not morally equal options. They embody different assumptions.

| Recovery | Good for | Risk |
| --- | --- | --- |
| `Wait` | temporary human blockage, aisle crossing traffic | throughput loss if used too often |
| `Spin` | improving sensor coverage, breaking local minima in open space | poor choice in tight racks or near protrusions |
| `BackUp` | freeing local controller in front-obstructed spaces | unsafe if rear space assumptions are wrong |

Recovery order should match operating geometry, safety constraints, and product behavior.

---

## PART 6 — WAREHOUSE AND AMR FAILURE MODES

---

## 6.1 The Infinite Courtesy Problem

Robot waits politely for transient obstruction, then replans, then waits again, then replans forever. No single step is irrational, but the total policy is operationally bad.

This is a BT design issue when the tree lacks escalation logic.

Possible fixes:

- cap retry count lower in congested zones
- escalate sooner to mission/fleet layer
- distinguish temporary aisle occupancy from hard blockage

---

## 6.2 The Recovery Pinball Problem

Robot alternates between spin and back-up without meaningfully changing conditions.

This often means:

- recovery order is not environment-appropriate
- local costmap keeps reconstructing the same obstacle field
- planner keeps returning equivalent paths into the same failure mode

The BT is where the visible loop lives, even if root cause is partly elsewhere.

---

## 6.3 The Wrong Layer Fix

Teams often respond to repeated BT failure by tuning controller parameters or swapping planners. Sometimes that helps. Often it just moves the symptom.

The discipline is to ask:

```text
Is this failure about navigation policy,
world model correctness,
or execution mechanics?
```

Only the first category belongs primarily in BT XML.

---

## PART 7 — HOW TO REVIEW A BT CHANGE

---

## 7.1 Review Checklist

- What exact failure mode is this tree change meant to address?
- Is the change policy-level rather than costmap/controller/localization-level?
- What blackboard keys are read and written?
- What happens on repeated failure? Is there a real bound?
- What happens when a new goal arrives mid-recovery?
- Does this change improve field behavior for the AMR environment, not just simulation demos?

---

## 7.2 What You Should Be Able to Explain After This Lesson

You should now be able to explain:

1. what `bt_navigator` is responsible for
2. how BT nodes, return states, and blackboard values work together
3. why `PipelineSequence` and `RateController` are central to Nav2 behavior
4. when BT XML is the right place to change behavior
5. how recovery ordering should reflect AMR operating conditions

---

## 7.3 Next Step

Continue to `04-nav2-costmaps-and-layers.md`.

That lesson covers the shared world model beneath planning, control, and many BT recovery outcomes.
