---
layout: project
title: VLA Sim-to-Real Manipulation on Franka with Simulation-Only Training Data
order: 1
tech_tags: "VLA, Pi 0.5, GR00T N1.7, ACT, Isaac Sim, cuRobo, ROS 2, Franka"
gif: /assets/gifs/SIm_to_Real_Pi0p5_3Objects.gif
---

<p style="color:#555;font-size:0.95rem;margin:0 0 20px;">Apr 2026 – Present</p>

<iframe class="video"
        src="https://www.youtube.com/embed/0RFVNuVx2gY"
        title="VLA Sim-to-Real Manipulation on Franka"
        frameborder="0"
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
        allowfullscreen></iframe>

## Overview

Teaching a robot a new manipulation skill usually means collecting hundreds of human teleoperated demonstrations, one trajectory at a time. That is slow, expensive, and it does not scale. Simulation offers a way out: a data-generation script running on a GPU can produce thousands of demonstrations overnight, with perfect labels and **zero human hours**. The catch is the **sim-to-real gap**, policies trained purely in simulation tend to fall apart on real hardware, where lighting, camera placement, textures, and contact dynamics never quite match the simulator.

This project takes that bet head-on: **can a Vision-Language-Action (VLA) policy trained on simulation-only data, with no teleoperation whatsoever, transfer zero-shot to a real Franka arm?** The answer, so far, is yes. The core idea is to stop trying to make the simulator match reality exactly, and instead **widen the training distribution through domain randomization** until reality becomes just one more sample the policy has already seen. Data is generated automatically in **NVIDIA Isaac Sim**, a **Pi 0.5** VLA is fine-tuned on it, and the resulting policy runs on a real Franka through a **ROS 2** inference pipeline, achieving **zero-shot sim-to-real transfer**.

## Closing the Sim-to-Real Gap

The simulator is deliberately made *unreliable* in controlled ways, so the policy learns to be invariant to the exact conditions it cannot control at deployment. Three axes are randomized during data generation:

- **Lighting** — intensity, color temperature, and light position are randomized per episode, so the visual encoder never latches onto a single illumination pattern.
- **Camera pose** — each real camera is first calibrated to recover its true **6D pose**, which is used to place the virtual camera in the simulator. On top of that anchor, small random **nudges to position and orientation** are applied every episode, so the policy tolerates the millimeter-scale mounting errors that are unavoidable on real hardware.
- **Object texture** — object appearances are randomized so the policy grasps based on geometry and context rather than memorized surface color.

By calibrating cameras to a physically grounded starting point and then perturbing around it, the simulated observations bracket the real ones, rather than sitting next to them in a slightly different distribution.

## Pipeline

<img src="{{ '/assets/images/vla_sim2real_flowchart.png' | relative_url }}" alt="End-to-end pipeline: data generation, training, and real-robot inference" style="width:100%;height:auto;display:block;margin:16px 0;border-radius:8px;">

The system runs in four stages, from calibrating the physical cameras all the way to executing policy actions on the Franka.

### 1. Camera Calibration

Every real camera is calibrated to recover its **6D pose** relative to the robot base. These poses do double duty: they define where the virtual cameras sit in the simulator (so simulated and real viewpoints agree), and they are the anchor around which the per-episode camera randomization is applied.

### 2. Automated Data Generation in Isaac Sim

Demonstrations are generated entirely in simulation with **no teleoperation**. A rule-based data-collection setup (**MagicSim**, built on Isaac Sim) scripts the task, and **cuRobo** serves as the GPU-accelerated trajectory generator and IK solver, planning collision-free motions to the scripted grasp and place targets.

- **Physics** steps at **60 Hz**; cameras **render and record at 30 Hz** at **640 × 480**.
- Each frame logs a **9D robot state** (joint positions, joint velocities, end-effector position and quaternion, base position and quaternion, object pose) alongside an **8D action** (7 arm joints + gripper).
- Recordings are converted to the **LeRobot dataset format** (v2.1 / v3.0 depending on the target policy), keeping the observation state (joint angles in radians, gripper in meters), the RGB images (480 × 640 × 3), and the 8D action.

Because the whole loop is scripted and GPU-parallel, generating thousands of diverse demonstrations across randomized lighting, camera poses, and textures is a matter of compute, not human labor.

### 3. Training — Pi 0.5 Fine-Tuning

The LeRobot dataset is pushed to the Hugging Face Hub and used to fine-tune **Pi 0.5**, a flow-matching generalist VLA policy, using a [fork of the openpi repo](https://github.com/saifahmadgit/openpi_franka) with Franka-specific configs. The framework also supports an **ACT** policy as a lighter-weight baseline. Pi 0.5 predicts a **chunk of 50 future actions** (50 × 8) per query; ACT predicts 100 × 8, letting the robot execute a burst of actions between policy calls rather than querying every timestep.

### 4. Real-Robot Inference & Deployment

Inference is split across three machines to keep the heavy model off the robot's real-time control loop:

- **GPU server (Sheep)** — hosts the trained policy and serves action chunks on request.
- **Laptop (client node)** — subscribes to the three camera streams (`front_1`, `front_2`, `wrist`) and `/joint_states`, packages the 7D state and three RGB frames, and sends them to the policy server over a **WebSocket**. It receives a chunk of 50 actions, streams them out one block at a time (execute, wait for *done*, send the next), and re-queries once the chunk is spent.
- **Franka computer** — runs **ROS 2 control**. `JointStateBroadcaster` publishes `/joint_states` (7 arm + 2 finger joints) at **1 kHz**, and the `fer_arm_controller/follow_joint_trajectory` action executes the policy's commands on the arm. Three RealSense cameras feed the client at **30 Hz**.

The full client and execution stack lives in [openpi_franka_client](https://github.com/saifahmadgit/openpi_franka_client). Two details in that stack make the difference between a policy that runs and one that trips the robot's safety firmware:

**Observation packing must mirror training exactly.** The policy consumes a **9-dim state** (7 arm joints + 2 finger joints), and the checkpoint's normalization statistics are 9-dimensional, so truncating to 8 would normalize the gripper against joint-7's statistics and silently corrupt the input. The three camera streams are likewise remapped to the server's bimanual-Aloha key convention (`cam_high`, `cam_left_wrist`, `cam_right_wrist`) exactly as the training-time transforms did — the client reproduces that mapping by hand, since those transforms never run at inference. The returned action array is padded to the bimanual action space; only the first 8 columns (7 joints + gripper) are meaningful and the rest is discarded.

**Raw policy targets are time-parameterized before they reach the arm.** Pi 0.5 emits absolute joint targets with no notion of velocity or acceleration. Sent naively as a positions-only trajectory, the jump from the arm's *measured* position to the first target of a fresh chunk can imply velocities and accelerations well beyond the Franka's limits, tripping the firmware reflex that locks the robot into an error state. The executor instead assigns each trajectory segment a duration bounded by the joint velocity limits (with a floor at the trained step duration, so motion is never sped up), then iteratively stretches any segment that still violates the acceleration limits (since a ∝ 1/t², a segment is lengthened by √ratio), and attaches central-difference waypoint velocities so the controller interpolates smoothly. Limits are held at a conservative **10% of Franka's official per-joint maxima**. Because segment durations only ever *stretch*, the trained motion timing is preserved while every command stays inside the hardware envelope.

The client streams each chunk out at a configurable `exec_horizon` — execute the full chunk, or only the first *k* steps before re-querying — trading control latency against observation freshness. The whole loop is **asynchronous**: ROS 2 callbacks spin on a multi-threaded executor while an asyncio loop queries the policy, dispatches the trajectory, and samples the true robot state in parallel for logging. The idle window while the next chunk is being computed is measured explicitly (and shaded in the time-series debug plot), since it is the main source of stall between action bursts; to keep that window from silently killing the connection, WebSocket keepalive pings are disabled so a slow inference call is never mistaken for a dead link.

The same trained policy can also be **evaluated in simulation** (the MagicSim / Isaac Sim environment running on the server) before ever touching hardware, closing the loop between training and real-world deployment.

### Deployment Analysis

To verify that the policy's intent actually reaches the arm, the client logs the **commanded action** against the **measured robot state** at every step and plots them one-to-one. This is the single most useful debugging view during deployment: it separates *the policy is asking for the wrong thing* from *the controller is not tracking what the policy asks*.

<img src="{{ '/assets/images/vla_deployment_analysis.png' | relative_url }}" alt="Per-joint commanded policy action vs actual robot state over a 400-step rollout" style="width:100%;height:auto;display:block;margin:16px 0;border-radius:8px;">

Each panel overlays the **commanded** value from the Pi 0.5 policy (blue) and the **actual** joint state read back from the Franka (red) across a 400-step rollout, for all seven arm joints plus the gripper. The two traces sit nearly on top of each other for joints 1–6, confirming the `follow_joint_trajectory` controller tracks the chunked commands with negligible lag. The gripper panel (bottom) shows the command crossing the **close threshold (0.03 m)** around step 305, with the physical finger joint following a step later, the grasp event. The small residual offsets visible on joints 5 and 7 are exactly the kind of controller-side tracking error this plot is meant to surface, separate from any policy error.

## Current Status & Future Work

**Zero-shot sim-to-real transfer is working**: a Pi 0.5 policy trained on simulation-only, teleoperation-free data executes the task on the real Franka without any real-world fine-tuning.

The next frontier is **contact-rich manipulation** — insertions, tool use, and tasks where the interaction forces, not just the visual scene, decide success or failure. Contact dynamics are the hardest thing for a simulator to get right, so the open question is how far domain randomization and simulation-only data can be pushed before real-world contact data becomes unavoidable.

## Code

<div style="display:flex;gap:20px;flex-wrap:wrap;margin:20px 0;width:100%;">
  <a href="https://github.com/saifahmadgit/openpi_franka" target="_blank" rel="noopener"
     style="flex:1;min-width:280px;display:flex;align-items:flex-start;gap:18px;background:#eaecf4;border-radius:8px;padding:24px 28px;text-decoration:none;color:#111;">
    <svg height="36" width="36" viewBox="0 0 16 16" fill="#111" style="flex-shrink:0;margin-top:3px;"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
    <div>
      <div style="font-weight:700;font-size:1.1rem;margin-bottom:6px;">saifahmadgit / openpi_franka</div>
      <div style="font-size:0.95rem;color:#555;">Pi 0.5 fine-tuning fork with Franka configs</div>
    </div>
  </a>
  <a href="https://github.com/saifahmadgit/openpi_franka_client" target="_blank" rel="noopener"
     style="flex:1;min-width:280px;display:flex;align-items:flex-start;gap:18px;background:#eaecf4;border-radius:8px;padding:24px 28px;text-decoration:none;color:#111;">
    <svg height="36" width="36" viewBox="0 0 16 16" fill="#111" style="flex-shrink:0;margin-top:3px;"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
    <div>
      <div style="font-weight:700;font-size:1.1rem;margin-bottom:6px;">saifahmadgit / openpi_franka_client</div>
      <div style="font-size:0.95rem;color:#555;">ROS 2 inference client &amp; execution pipeline</div>
    </div>
  </a>
</div>
