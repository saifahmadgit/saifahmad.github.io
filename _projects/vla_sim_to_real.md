---
layout: project
title: Fine-Tuning a Robotics Foundation Model with Simulation-Only Data
order: 1
tech_tags: "VLA, Pi 0.5, GR00T N1.7, ACT, Isaac Sim, cuRobo, ROS 2, Franka"
gif: /assets/gifs/Pi_0.5_fine_tuning_with_Sim_only_Data.gif
---

<p style="color:#555;font-size:0.95rem;margin:0 0 20px;">Apr 2026 – Present</p>

<iframe class="video"
        src="https://www.youtube.com/embed/QZP_ObLCaDA"
        title="Fine-Tuning a Robotics Foundation Model with Simulation-Only Data"
        frameborder="0"
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
        allowfullscreen></iframe>

## Overview

Robotics foundation models now outperform task-specific policies across a wide range of manipulation problems. None of the open-weight ones, however, work zero-shot on a robot they have never seen. Each needs demonstrations from the exact setup it will be deployed on: that embodiment, those cameras, that mounting geometry. Those demonstrations are almost always collected by a human teleoperating the arm, one trajectory at a time. It is slow and expensive, and the cost is paid again from scratch for every new task, gripper, or camera placement.

This project removes the human from that loop. **Pi 0.5** is fine-tuned entirely on demonstrations generated in **NVIDIA Isaac Sim**, with **cuRobo** planning the motions, and the resulting policy is deployed **zero-shot on a real Franka**, with no teleoperation and no real-world data at any stage.

## Pipeline

<img src="{{ '/assets/images/blockDiagram_Pi0.5_Sim_to_Real.png' | relative_url }}" alt="End-to-end pipeline from Isaac Sim data generation to real Franka deployment" style="width:100%;height:auto;display:block;margin:16px 0;border-radius:8px;">

Demonstrations are generated entirely in **Isaac Sim** with no teleoperation. Grasp poses are precomputed for each asset from its geometry and inertial properties, and **cuRobo** plans collision-free motions to reach them, so a full demonstration is produced without a human in the loop. **Domain randomization** over lighting, textures, and backgrounds widens the training distribution until the real scene falls inside it. The resulting dataset is used to fine-tune **Pi 0.5**, and the policy runs on the real Franka through a **ROS 2** inference pipeline.

### Data Collection

Demonstrations are collected in **Isaac Sim** for **15 objects**, each episode paired with a **randomized prompt**, so the trained policy can be told which object to pick up rather than always reaching for the same one. **cuRobo** plans the collision-free motion, and the grasp pose is computed from each object's geometry and inertial properties, so no human ever drives the arm.

**PhysX** steps at **60 Hz** while rendering runs at **30 Hz**. Both streams are recorded at **30 Hz**, converted to the **LeRobot v2.1** dataset format, and pushed to the Hugging Face Hub. Generation runs on a single **NVIDIA RTX 6000 Ada Generation**, producing one 10 second episode roughly every 2 minutes.

#### Asset Preparation

Objects are taken from **GraspNet** and converted to **USD**. Each one ships as a photogrammetry scan of 0.5 M to 1.5 M triangles, far too heavy to simulate directly. The scan is decimated by quadric edge collapse to 40 k faces with UVs preserved, then again to 2 000 faces in a final optimization pass. That is roughly a 500x reduction, with bounding box drift held under 0.1 mm so the original grasp annotations remain valid in the object's local frame.

The visual mesh, the PhysX collider, and the cuRobo planning obstacle are all derived from this same decimated mesh, so what the renderer draws, what physics collides against, and what the planner avoids can never quietly disagree. Visual fidelity does take a hit at 2 000 faces, but domain randomization is already forcing the policy off surface detail and onto geometry and context, so the trade buys a large simulation speedup for little that the policy relies on.

For collision, the default convex decomposition (500 k voxel resolution, 32 hulls) is replaced with a coarsened one: **20 k voxels, 16 hulls, 32 vertices per hull**, shrink-wrapped onto the true surface. Any object whose convexity exceeds **0.90** is then swapped for a single convex hull, while concave shapes such as cups and limbed toys keep the full decomposition. Physics uses PhysX defaults, with mass computed from density times volume, since GraspNet ships no per-object masses.

### Domain Randomization

Every parameter below is resampled per episode during data generation. The frozen column is the fixed-condition baseline used to isolate how much of the transfer is actually bought by randomization.

<div class="table-wrap" markdown="1">

| Parameter | DR off (frozen) | DR on |
|---|---|---|
| HDRI backdrop | 1 texture (`kloofendal_48d_partly_cloudy`) | 8 textures, sampled per episode |
| HDRI intensity | 1000 | 600 to 2500 |
| Light position (x, y, z) m | (0, 0, 2.0) | x, y &isin; &plusmn;0.6; z &isin; 1.7 to 2.3 |
| Light orientation (r, p, y) &deg; | (0, 0, 0) | r, p &isin; &plusmn;20; yaw &isin; &plusmn;180 |
| Light intensity | 800 | 150 to 1600 |
| Light size (w &times; h) m | 1.0 &times; 1.0 | 0.15 to 2.5 each, log-uniform |
| Color temperature K | 5500 | 3000 to 8000 |
| Light color (RGB mult.) | (1.0, 1.0, 1.0) | 0.85 to 1.0 per channel |
| Table albedo | 1 grey (0.88, 0.86, 0.82) | 5 greys, 0.74 to 0.95 |
| Backdrop plates | 1 color | 1 color (not randomized) |
| Table material | `Material/Garment` | `Material/Garment` (same) |

</div>

### Training

Training uses a fork of **openpi** with a custom configuration. The model is conditioned on the robot state and three camera streams, and predicts **deltas in joint angles** rather than absolute targets. It trains on an **NVIDIA H100** in a cluster at roughly **1.3 s per step**.

### Inference and Deployment

Inference runs on a workstation with an **NVIDIA RTX 6000 Ada Generation (48 GB)**. The laptop connected to the robot streams camera frames and robot state to that server over a **WebSocket** and receives action chunks back, driving a **Franka FER** arm at **10 Hz**.

Continuous motion comes from **real-time chunking (RTC)**. The usual loop is infer, execute 50 actions, stop, infer again, which leaves the arm stationary for the length of every inference call. Instead, the client fires the next inference while the arm is still moving and sends the currently executing chunk back with the request, so the policy samples a chunk that agrees with motion already committed. No pause, and no jerk at chunk boundaries.

Doubling the control rate exposed two artifacts that had been harmless at the slower rate. Both are fixed amplitudes in radians, so the acceleration they imply scales as 1/&Delta;t&sup2;: halving the step time quadruples the disturbance.

- **Savitzky-Golay smoothing** across each chunk. 27% of consecutive policy deltas reverse sign, which is invisible at 0.15 s per step but becomes 4.7 rad/s&sup2; of buzz at 0.075 s.
- **Raised-cosine splice blending** over the roughly 0.022 rad the server leaves unpinned. The server applies soft prefix attention across the overlap rather than a hard freeze, so the boundary is worth measuring rather than assuming.

Everything here is measured rather than guessed: per-splice discontinuity, round-trip p95 latency, dropped deadlines, and joint logs sampled at the driver's native **1.4 kHz** instead of on a policy tick. A 10 Hz tick aliases away the entire frequency band that vibration lives in.

**References:** [Real-Time Execution of Action Chunking Flow Policies](https://arxiv.org/abs/2506.07339) (Black, Galliker &amp; Levine, NeurIPS 2025) &middot; [Physical-Intelligence/openpi](https://github.com/Physical-Intelligence/openpi) &middot; [scipy.signal.savgol_filter](https://docs.scipy.org/doc/scipy/reference/generated/scipy.signal.savgol_filter.html)

## Results and Analysis

The fine-tuned policy transfers **zero-shot** to the real Franka, reaching an **80% grasp success rate across 15 different objects** without a single real-world demonstration.

<img src="{{ '/assets/images/vla_deployment_analysis.png' | relative_url }}" alt="Per-joint commanded policy action versus measured robot state over a rollout" style="width:100%;height:auto;display:block;margin:16px 0;border-radius:8px;">

Each panel overlays the **commanded** action from the policy against the **measured** joint state read back from the robot, for all seven arm joints plus the gripper. Keeping these side by side separates the two failure modes that otherwise look identical on hardware: the policy asking for the wrong thing, and the controller failing to track what the policy asked for.

## Future Work

The next step is **contact-rich manipulation**, where interaction forces rather than the visual scene decide success. Contact dynamics are the hardest thing for a simulator to get right, so the open question is how far simulation-only data can be pushed before real-world contact data becomes unavoidable.

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
