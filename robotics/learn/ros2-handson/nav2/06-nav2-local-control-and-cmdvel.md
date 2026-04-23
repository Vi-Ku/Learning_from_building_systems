# 06 — Nav2 Local Control and `/cmd_vel`

## How controller plugins, velocity shaping, and downstream command contracts determine real AMR motion quality

**Prerequisite:** 05-nav2-global-planning.md, 04-nav2-costmaps-and-layers.md, 01-nav2-system-architecture.md
**Unlocks:** Cleaner controller selection, faster motion-quality diagnosis, better docking approaches, fewer oscillation and slowdown incidents in narrow aisles

---

## Why Should I Care? (Context)

Most field complaints about Nav2 are not really about planning. They sound like this instead:

1. the robot corners too wide and clips aisle margins
2. the robot slows too early and kills throughput
3. the robot oscillates near the goal and then times out
4. the robot follows the path in RViz but the real base looks jerky
5. docking approach looks hesitant, unstable, or over-conservative

Those are local-control and command-chain problems.

The global planner answers "where should I go?"
The controller answers "what velocity command is safe and useful right now?"

If you cannot separate controller behavior from path quality, localization quality, and downstream velocity handling, every motion problem becomes random parameter fiddling.

---

## PART 1 — WHAT LOCAL CONTROL ACTUALLY OWNS

---

## 1.1 The Controller's Job

The controller consumes:

- the current path
- current robot pose and velocity
- local costmap state
- controller parameters

and produces a motion command, typically a `geometry_msgs/Twist`.

Conceptually:

```text
global path + current pose + local obstacles + velocity limits
    -> controller_server
    -> /cmd_vel or intermediate velocity topic
    -> smoother / safety / base controller
    -> wheel motion
```

The controller does not own mission policy, localization generation, or wheel-level control. It sits in the middle and is judged by the quality of the whole chain.

---

## 1.2 Motion Quality Is a Stack, Not a Single Knob

Poor motion quality can come from any of these layers:

| Layer | Typical symptom | Common false accusation |
| --- | --- | --- |
| Global path quality | path enters corners badly or hugs shelves | "DWB is bad" |
| Localization | path tracking looks noisy or unstable near goal | "RPP lookahead is wrong" |
| Local controller | oscillation, corner cutting, sluggish approach | "planner is wrong" |
| Velocity smoother | stop-start motion, delayed acceleration | "controller is indecisive" |
| Safety or base control | command clipping or zeroing | "Nav2 stopped publishing" |

Production rule: before tuning controller parameters, prove the command path from controller output to wheel motion is intact and unsurprising.

---

## 1.3 The `/cmd_vel` Contract

In many AMRs, Nav2 does not talk to the motors directly.

Typical chain:

```text
controller_server -> /cmd_vel_nav
velocity_smoother -> /cmd_vel
collision monitor / safety layer -> filtered command
base controller -> wheel commands
```

If one of those stages rewrites, delays, or clips commands, the robot can look like it has a controller problem when it actually has a downstream policy problem.

Check all of these before blaming the controller:

- is the published command rate stable?
- does another node overwrite the same topic?
- are linear and angular limits clipped downstream?
- does a safety layer hold zero velocity because of stale observations?
- does the base reject tiny angular commands because of deadband?

---

## PART 2 — DWB: SAMPLE, SCORE, PICK

---

## 2.1 The DWB Mental Model

DWB samples candidate velocity commands, simulates short trajectory rollouts, scores them with critics, and picks the best legal option.

Think of it as:

```text
generate candidate twists
    -> simulate short forward trajectories
    -> score each trajectory with critics
    -> reject unsafe ones
    -> publish best remaining command
```

This makes DWB flexible and debuggable, but also sensitive to critic balance.

---

## 2.2 What the Critics Are Really Doing

Different critic sets vary by configuration, but the common ideas are stable:

- obstacle critics punish trajectories that get too close to obstacles
- path alignment critics reward staying near the planned path
- goal alignment critics reward pointing toward the goal or goal heading
- forward-motion preferences reward decisive progress instead of dithering
- rotate-to-goal logic handles final heading alignment differently from path pursuit

This is why DWB can look brilliant in one corridor and terrible in another. The chosen command is a weighted compromise between several partial truths.

---

## 2.3 Classic DWB Failure Patterns

| Field symptom | Likely driver |
| --- | --- |
| robot oscillates left-right in open space | competing critics, noisy pose, weak forward preference |
| robot hugs one shelf edge | path or obstacle critic balance too weak or footprint/inflation mismatch |
| robot stalls before entering a narrow aisle | obstacle scoring too conservative or footprint too large upstream |
| robot rotates too much near goal | goal heading logic and rotate-to-goal behavior dominate too early |
| robot makes tiny indecisive commands | sample resolution, critic balance, or downstream deadband |

Do not read those as one-to-one parameter names. They are diagnosis categories first.

---

## 2.4 When DWB Is a Strong Fit

DWB is often a good fit when:

- the platform is differential drive or omni and the environment is structured
- you want interpretable local behavior with critic-based tradeoffs
- you need direct control over obstacle clearance versus path adherence
- you want a controller that can be tuned without changing path semantics globally

It is less pleasant when the team wants cargo-cult defaults with no appetite for understanding critic interactions.

---

## PART 3 — REGULATED PURE PURSUIT

---

## 3.1 The RPP Mental Model

Regulated Pure Pursuit follows the path by choosing a lookahead point and generating curvature toward it, then regulating speed according to path geometry, obstacle proximity, and approach state.

Conceptually:

```text
pick lookahead point on path
    -> compute curvature toward it
    -> scale linear speed based on path shape and constraints
    -> slow down near goal, sharp turns, or nearby obstacles
```

RPP often feels smoother and easier to reason about for aisle following and longer continuous motion.

---

## 3.2 Why Teams Like RPP

Common reasons:

- path following can feel more natural and less twitchy
- lookahead gives a direct handle on smoothness versus aggressiveness
- approach slowdown and curvature regulation can produce good warehouse behavior quickly
- tuning often maps better to operator intuition than a large critic set

This does not mean it is universally better. It means its mental model is closer to "follow the path cleanly" than "score many tiny trajectory samples."

---

## 3.3 Common RPP Failure Patterns

| Field symptom | Likely driver |
| --- | --- |
| corner cutting near shelves | lookahead too large, path too coarse, localization offset |
| jerky low-speed docking approach | lookahead too short, poor downstream velocity smoothing, base deadband |
| over-conservative slowdown everywhere | regulation too strong or obstacle/cost scaling too defensive |
| overshoot on final heading | approach logic weak, goal checker mismatch, poor velocity floor handling |
| good aisle following but weak tight-turn entry | lookahead and curvature settings not matched to footprint and path shape |

RPP frequently exposes path-quality problems more clearly because it tries to follow path geometry consistently.

---

## 3.4 When RPP Is a Strong Fit

RPP is often attractive when:

- the path geometry is reasonably clean
- the robot mainly needs smooth, continuous forward motion
- you care about aisle flow and predictable approach behavior
- the team wants fewer interacting scoring terms than DWB

It is not a magic answer for bad localization, bad paths, or downstream command clipping.

---

## PART 4 — MOTION QUALITY IS MORE THAN THE CONTROLLER

---

## 4.1 Velocity Smoothers Matter

A velocity smoother can improve jerk and protect the base, but it can also hide the controller's true intent.

Possible outcomes:

- controller publishes crisp commands, smoother makes them physically reasonable
- controller publishes decisive turns, smoother attenuates them into sluggish motion
- controller publishes tiny corrections, smoother plus motor deadband erases them entirely

If motion is poor, compare the controller output topic and the final base command topic before tuning the controller itself.

---

## 4.2 The Base Controller and Deadbands

Many bases ignore very small linear or angular commands.

That creates a nasty pattern:

1. Nav2 publishes small corrective twists
2. the base does not move enough to satisfy progress
3. progress checker thinks the robot is stuck
4. recoveries trigger even though the controller was trying to work

This is not a progress-checker bug. It is a control-chain contract bug.

---

## 4.3 Path Quality Still Matters

No local controller rescues a globally bad path for free.

Examples:

- if the path enters a rack corner too tightly, DWB and RPP may both behave badly for different reasons
- if the planner produces stair-step heading changes, the local controller may look unstable even when tuned reasonably
- if inflation collapses corridor width, the controller may slow or abort for valid safety reasons

Always ask whether the controller is failing on a bad request.

---

## PART 5 — DIAGNOSING MOTION QUALITY IN THE FIELD

---

## 5.1 Symptom-to-Cause Separation

| Symptom | First thing to verify | If verified, next suspect |
| --- | --- | --- |
| oscillation in open aisle | localization smoothness and heading noise | controller critic/lookahead tuning |
| robot tracks path in RViz but not on floor | downstream `/cmd_vel` chain | base deadband or safety layer |
| too slow near obstacles | local costmap/inflation realism | controller regulation or critics |
| overshoot near goal | goal checker tolerances and approach mode | final heading control parameters |
| repeated stuck detection | real motion at wheel/base layer | progress checker thresholds |

The order matters. Do not start from parameters if the robot's actual motion disagrees with commanded motion.

---

## 5.2 A Minimal Diagnostic Flow

When a robot "moves badly," do this in order:

1. inspect the global path for obvious geometric nonsense
2. inspect localization stability on the same run
3. compare controller output with final base command
4. inspect local costmap around the moment of hesitation
5. only then tune controller parameters

This is the shortest path to root cause on most AMR incidents.

---

## 5.3 Tight Turns, Aisle Entry, and Docking Are Different Problems

Do not tune for them as if they are identical.

| Scenario | Dominant concern |
| --- | --- |
| tight turn in aisle network | curvature, footprint clearance, path shape |
| aisle entry from open space | heading capture without over-slowing |
| final docking approach | slow-speed stability, goal checking, downstream deadband |

One controller can handle all three, but the validation cases must stay separate.

---

## PART 6 — SELECTION AND TUNING CHECKLISTS

---

## 6.1 DWB vs RPP Decision Checklist

Choose DWB when:

- you want explicit trajectory scoring with interpretable tradeoffs
- obstacle clearance versus path adherence is the main local challenge
- the team can actually reason about critic balance

Choose RPP when:

- smooth path following is the primary need
- the environment is aisle-heavy and path geometry is reliable
- the team wants tuning centered around lookahead, curvature, and regulation behavior

If neither controller looks good quickly, do not assume both are bad. Re-check path and localization contracts.

---

## 6.2 Safe Tuning Order

Use this order instead of changing everything at once:

1. prove localization is stable enough for local control
2. prove the path is geometrically sane for the footprint
3. prove the final command path to the base is honest
4. tune one controller family at a time
5. validate separately on open space, tight aisles, and goal approach

This avoids cargo-cult parameter piles.

---

## 6.3 What Good Looks Like

For a production AMR, good local control usually means:

- stable aisle tracking without shelf-hugging
- predictable speed reduction near obstacles, not random hesitation
- smooth final approach without false stuck detection
- recoveries triggered rarely and for understandable reasons
- similar behavior across repeated runs of the same route

If the same route looks different every run, suspect localization or world-model instability before tuning more.

---

## PART 7 — AMR FAILURE PATTERNS TO REMEMBER

---

## 7.1 The Five Fastest Misdiagnoses

1. blaming DWB when the path is already too close to obstacles
2. blaming RPP when `map -> odom -> base_link` timing is unstable
3. blaming the planner when the velocity smoother is the real bottleneck
4. blaming progress checking when the base ignores tiny commands
5. blaming Nav2 when a safety layer zeroes velocity downstream

If you remember those five, you will avoid a large fraction of useless controller retuning.

---

## 7.2 Senior Interview Version

If asked how to debug poor Nav2 motion quality on an AMR, a strong answer separates:

- path quality
- localization quality
- controller algorithm choice
- command-chain integrity
- base and safety behavior

If you jump straight to "I would tune DWB critics," that answer is incomplete.

---

## Next Lesson

Continue to 07-nav2-localization-odom-amcl-ekf.md.
That lesson explains why many controller and planner complaints are actually localization and TF contract failures upstream.
