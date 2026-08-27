---
layout: project
title: Zero-Shot Sim-to-Real Fine-Tuning of a Robotics Foundation Model
order: 1
tech_tags: "VLA, Pi 0.5, GR00T N1.7, ACT, Isaac Sim, cuRobo, ROS 2, Franka"
gif: /assets/gifs/FinalProject.gif
---

<p style="color:#555;font-size:0.95rem;margin:0 0 20px;">Apr 2026 – Present</p>

<iframe class="video"
        src="https://www.youtube.com/embed/Po_mx85PwZ4"
        title="Zero-Shot Sim-to-Real Fine-Tuning of a Robotics Foundation Model"
        frameborder="0"
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
        allowfullscreen></iframe>

## Overview

Robotics foundation models are the strongest starting point we have for manipulation, but none of the open-weight ones arrive ready for your robot. Unless your setup matches something already in the pretraining mix, the model needs demonstrations from that exact embodiment: those cameras, that mounting geometry, that gripper. They come from a human teleoperating the arm one trajectory at a time, and the whole collection is repeated for every new task, gripper, or camera placement.

This project removes the human from that loop. **Pi 0.5** is fine-tuned entirely on demonstrations generated in **NVIDIA Isaac Sim**, with **cuRobo** planning the motions, then deployed **zero-shot on a real Franka**, with no teleoperation and not one frame of data from the real setup. The task is to grasp and pick up: the policy is given an object's name and has to lift it off the table. It succeeds 80% of the time on hardware it has never seen.

Grasping makes a clean test because it is decided by vision alone: the gripper has to reach the right position and orientation, and the object's location is known only from the camera images. Doing it on real hardware establishes that the visual sim-to-real gap is closed, and the same recipe extends to more complex **visually guided manipulation**.

How many simulated demonstrations this takes, and how that compares with collecting on real hardware, has also been analysed.

## Pipeline

<img src="{{ '/assets/images/blockDiagram_Pi0.5_Sim_to_Real.png' | relative_url }}" alt="End-to-end pipeline from Isaac Sim data generation to real Franka deployment" class="figure-wide">

### Data Collection

The real cameras are calibrated first, giving the 6D pose of each one in the robot base frame. Those poses are reproduced in simulation so the viewpoints match.

15 objects, each episode paired with a randomized prompt naming the object to pick. **cuRobo** plans the motion and grasps come from object geometry and inertia, so nothing is teleoperated. Domain randomization covers light (intensity, temperature, position), object texture, background, and camera pose.

**PhysX** runs at 60 Hz and rendering at 30 Hz, both recorded at 30 Hz, converted to **LeRobot v2.1** and pushed to Hugging Face. One **NVIDIA RTX 6000 Ada Generation** collects an episode every 2 minutes. Following simplifications have been done to make the simulation faster.

<img src="{{ '/assets/images/asset_approximations.png' | relative_url }}" alt="Table of what was simplified in the object scans: shape detail, mesh vertices, collision shape and texture" class="figure-wide">

### Training

**Pi 0.5** is fine-tuned with a fork of [openpi](https://github.com/Physical-Intelligence/openpi), conditioned on robot state and three cameras, predicting joint angle deltas rather than absolute targets. The **SigLIP** vision encoder trains fully unfrozen, while the VLM backbone decoder is **LoRA** fine-tuned at rank 16 and the action head at rank 32. **NVIDIA H100**, 1.3 s per step.

### Inference

Policy server on an **NVIDIA RTX 6000 Ada Generation (48 GB)**; the laptop at the robot streams cameras and state over a **WebSocket** and receives action chunks, driving a **Franka FER** arm at **10 Hz**.

**Real-time chunking** ([paper](https://arxiv.org/abs/2506.07339)) keeps the arm moving: the next inference fires while the current chunk is still executing and is sent back with the request, so the sampled chunk agrees with motion already committed. No stop at chunk boundaries.

Two client-side fixes, since these artifacts are fixed amplitudes in radians and their implied acceleration scales as 1/&Delta;t&sup2;:

- **Savitzky-Golay** ([docs](https://docs.scipy.org/doc/scipy/reference/generated/scipy.signal.savgol_filter.html)) smoothing per chunk. 27% of consecutive deltas reverse sign: invisible at 0.15 s/step, 4.7 rad/s&sup2; of buzz at 0.075 s.
- **Raised-cosine splice blend** over the ~0.022 rad the server leaves unpinned.

Joint logs are sampled at the driver's native **1.4 kHz**, not on the 10 Hz policy tick, which would alias away the band vibration lives in.

## Results and Analysis

How many demonstrations does a task need? Sim-to-sim answers that first: train in simulation, test on placements the policy was never shown, and read off the episode count where it starts working. That is the baseline the sim-to-real requirement is measured against.

### Sim-to-Sim Baseline

Two tasks set the scale. Picking up a 4 cm cylinder needs the gripper precise in x and y, since a cylinder looks the same from every direction. Picking up a cuboid adds yaw. Nothing else changes between them, so the gap between the two episode counts is what one extra degree of freedom costs.

Both are scored over a 50 by 15 cm patch of table, at placements the policy was never shown.

<div class="figure-pair">
  <figure>
    <img src="{{ '/assets/images/reach_map.png' | relative_url }}" alt="Grasp success across the table for the cylinder under three demonstration layouts">
    <figcaption>Position. Three layouts of demonstration sites across the patch.</figcaption>
  </figure>
  <figure>
    <img src="{{ '/assets/images/angles_map.png' | relative_url }}" alt="Grasp success across cuboid yaw angles for three ways of distributing the same 130 episodes">
    <figcaption>Orientation. Three ways of distributing the same 130 episodes across cuboid yaw.</figcaption>
  </figure>
</div>

Position first. Fifty episodes spread over 5 sites fails: half the objects fall in the gaps between sites. Thirteen episodes over 13 sites, a quarter of the data, picks the object up everywhere it is tested.

What counts is the spacing between demonstrations, not how many there are. Thirteen sites over the 750 cm&sup2; patch is one demonstration per 58 cm&sup2;. At that spacing the policy covers the gaps on its own; at one per 150 cm&sup2; it does not. And more data at the same spacing adds nothing: 65 episodes over those same 13 sites still scores 100%.

Yaw costs ten times more. The cuboid needs 130 episodes, the same 13 sites with 10 angles at each, and the angles have to differ. Ten episodes split over two angles fails 60% of the time on anything else, over four angles about a quarter of the time. Only a fresh angle every episode works everywhere.

So 13 episodes for x and y, 130 once yaw matters too. Every axis the gripper has to get right multiplies the data, and a task needing a full 6D pose adds roll and pitch on top of that.

### Sim-to-Real: The Extra Data

<img src="{{ '/assets/images/sim_to_real.png' | relative_url }}" alt="Real robot grasp success against episode budget, with failure mode at each budget" class="figure">

Simulation said 130 episodes. The real robot needs more, and the way it fails at 130 shows why. It does not reach for the object and miss by a little. It moves off in a completely wrong direction, as if it cannot see the object at all.

At 390 episodes that stops. The robot reaches the object, and what is left is near misses and objects dropped on the way up. The score is 60%. At 650 episodes it reaches 90%. That is five times the simulation number.

By 650 the policy has seen enough variation in light, table, object colour and camera position to stop depending on any of them. That invariance is what carries it over to the real robot.

It is a property of the dataset, not of the task. The same 650 episodes split across 8 objects still gives 80%, because every episode of every task randomizes the same way. So the cost is paid once: any task, or group of tasks, that would have needed around 650 real demonstrations anyway needs no extra episodes in simulation.

### Conclusion

Around 650 episodes of domain randomization is what it takes for the policy to stop depending on light, table, colour and camera position, and that invariance is what makes it work on the real robot. It is a one-time cost. It does not grow with the number of tasks, because every episode of every task is randomized the same way.

Everything past 650 is not a sim-to-real cost. It is what the tasks themselves need, and it would be needed on real hardware too. A mustard bottle has several valid grasp modes. A facewash bottle is tapered, so a top-down grasp slips off it. cuRobo returns different trajectories for the same grasp pose depending on how close the object sits to the robot base. Each of these adds trajectories the policy has to see, in simulation or on hardware.

So once a dataset is around 650 episodes or larger, simulation costs no extra episodes. Here that dataset is 15 objects and 10 000 episodes, reaching nearly 80% on the real robot, with recoveries and with grasps at positions and orientations that were never trained on. Objects that were not in the dataset work too when their geometry is close to one that was. None of it is teleoperated.

## Next Steps

The same recipe applies to longer trajectories and multi-step tasks, as long as the gap is only visual. What changes is how much data is needed, not the method.

Contact-rich tasks need more. The visual part carries over, but forces decide success, so randomization has to extend to friction, mass and inertia, restitution and contact stiffness. The collision approximations also have to be revisited: a coarse convex decomposition is enough when contact only has to be plausible for a grasp, and not when contact is the task.

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
