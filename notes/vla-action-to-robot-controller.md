# From VLA Action Chunks to Robot Commands

*A vendor-neutral engineering note about robot-learning deployment. The architecture and pseudocode below are illustrative, not a claim that a particular controller or real robot was implemented or validated.*

## Why an action prediction is not a robot command

A vision-language-action (VLA) policy may predict an action chunk with shape `[horizon, action_dim]`. Before the output can be sent to a robot, an integration layer needs to answer several questions:

- Are the values **absolute joint targets, joint deltas, velocities, or Cartesian poses**?
- Are angles expressed in radians or degrees? What is the joint ordering?
- What are the gripper command semantics and valid range?
- Were the labels normalized during training? If so, what statistics reverse that transformation?
- How much time should elapse between consecutive targets?
- What should happen if a prediction is late, invalid, or incompatible with the robot's current mode?

The tensor shape alone cannot answer any of these questions.

## A vendor-neutral integration boundary

```text
timestamped images + robot observations
                  |
                  v
              VLA policy
           action chunk [H, D]
                  |
                  v
        action contract validator
   type | units | joint order | bounds
                  |
                  v
        trajectory / rate adapter
    continuity | timing | constraints
                  |
                  v
        robot SDK / command interface
                  |
                  v
       controller feedback + diagnostics
```

Keep the model-facing action schema separate from the robot SDK's wire format. This makes it easier to test conversions independently and avoids hard-coding a single robot vendor into the policy interface.

## Example action contract

An action schema for a six-axis manipulator and a gripper **might** look like this:

```yaml
action:
  representation: joint_position_absolute
  arm_dof: 6
  joint_order: [j1, j2, j3, j4, j5, j6]
  angle_unit: radians
  gripper:
    representation: discrete_increment   # illustrative, not universal
    close_step: 1
    open_step: 0
  horizon: configurable
  sample_period_s: configurable
```

The controller adapter must verify that these assumptions match the policy dataset and the target robot. Do not infer semantics merely from a seven-dimensional model output.

## Policy frequency vs. controller frequency

Suppose a source produces targets at 50 Hz and a controller expects updates at 300 Hz. Sending the same target six times does not automatically guarantee smooth, feasible motion.

A robust adapter needs to consider:

1. **Timing:** arrival time, inference delay, stale messages, and jitter.
2. **Transition state:** actual measured state and estimated velocity when a new chunk begins.
3. **Feasibility:** joint position, velocity, acceleration, and where applicable jerk limits.
4. **Execution interface:** which command modes and update semantics the SDK actually supports.
5. **Observability:** what was commanded, what was acknowledged, and what the robot did.

Trajectory-generation libraries such as Ruckig can help produce constrained trajectories for supported setups, but using such a library does **not** make a Python process or networked ROS 2 node hard-real-time.

## Handling chunk transitions

A new action chunk might arrive before the current chunk is fully executed. Blindly replacing the command queue risks a discontinuity.

A testable design should specify:

- Whether old pending targets are discarded or blended.
- How the start state for replanning is obtained.
- How position and velocity continuity are checked.
- How a missed inference deadline changes behavior.
- Who has command authority during operator intervention or fault recovery.

**Safety note:** Invalid targets should be rejected or handled by a deliberately designed feasibility/replanning policy. Silently clipping arbitrary outputs to joint limits does not establish that the resulting trajectory is safe. Application-level checks do not replace the robot manufacturer's safety mechanisms.

## Useful public tests (without real hardware)

| Test | Expected property |
| --- | --- |
| Wrong tensor shape | Reject with a useful schema error |
| NaN / infinity in action | Reject before command generation |
| Unit mismatch | Detect through explicit interface metadata and validation |
| Stale action chunk | Do not execute outdated targets |
| New chunk at a boundary | No unexplained jump in joint reference |
| Limit violation | Reject or replan according to an explicit policy |
| Lost controller feedback | Report fault and invoke the defined non-hardware recovery path |
| Simulated 50 Hz → 300 Hz adapter | Record interval distribution, jitter, and missed deadlines |

If publishing benchmarks, disclose the test machine, clock method, load, and latency distribution. A nominal 300 Hz software loop in simulation is not proof of certified real-time performance or safe physical execution.

## A reproducible experiment to build next

A small independent project could provide a fake robot SDK with joint-state feedback, an action contract validator, and a trajectory adapter. Tests would inject delayed policy results, invalid action chunks, dropped feedback, and transitions between consecutive chunks. Such a project can be open-sourced without sharing employer source code or hardware data.

---

This document is an independent, educational engineering note. It contains no proprietary SDK code, robot recordings, internal logs, or employer-specific implementation.
