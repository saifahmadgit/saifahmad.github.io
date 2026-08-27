---
layout: project
title: Zero-Shot Sim-to-Real Fine-Tuning of a Robotics Foundation Model
order: 1
tech_tags: "VLA, Pi 0.5, GR00T N1.7, ACT, Isaac Sim, cuRobo, ROS 2, Franka"
gif: /assets/gifs/FinalProject.gif
---

<p style="color:#555;font-size:0.95rem;margin:0 0 20px;">Apr 2026 – Present</p>

<iframe class="video"
        src="https://www.youtube.com/embed/g9rbXOJX4c8"
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

The real cameras are calibrated first, recovering the 6D pose of each one in the robot base frame. Those poses are reproduced in simulation, so the policy sees the same viewpoints in Isaac Sim that it will see on hardware.

15 objects, each episode paired with a randomized prompt so the policy can be told which object to pick. **cuRobo** plans the motion and grasps are computed from object geometry and inertia, so nothing is teleoperated.

**PhysX** runs at 60 Hz and rendering at 30 Hz, both recorded at 30 Hz, converted to **LeRobot v2.1**, and pushed to Hugging Face. One **NVIDIA RTX 6000 Ada Generation** collects an episode every 2 minutes.

Holding that rate is what drives the asset approximations. GraspNet scans are converted to **USD** and decimated by quadric edge collapse, and the render mesh, PhysX collider and cuRobo obstacle all come from that one reduced mesh.

Domain randomization covers light (intensity, temperature, position), object texture, background, and camera pose.

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

Two objects set the scale. A cylinder has to be gripped near its centre or it slips, so it measures precision in x and y. A cuboid adds yaw. The difference between them is what one extra degree of freedom requires.

### Sim-to-Sim Baseline

Picking a 4 cm cylinder off a 50 by 15 cm patch of table, scored at positions the policy was never shown.

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

Five sites at 10 episodes each, 50 in total, drops half the objects in the gaps between sites. Thirteen sites at one episode each, a quarter of the data, picks up the object in every gap it is tested in. Five episodes at each of those same 13 sites, 65 in total, changes nothing. It was already at 100%.

What separates them is not the amount of data but the distance from any point on the table to the nearest demonstration. Five sites leave a worst spot nearly 14 cm away. Thirteen bring that under 7 cm, less than two object widths, and below that distance the policy fills in the gap on its own.

Orientation behaves the same way at fixed data size. All three conditions are 130 episodes, 13 sites at 10 each, differing only in how those 10 spread across the cuboid's yaw. Two angles fails 60% of the time at anything else. Four angles handles a little over three quarters. A fresh angle every episode, closing the largest untaught gap to under 9 degrees, handles all of them.

Coverage, not volume. 13 episodes for position and 130 for orientation were enough because the demonstrations were spread finer than the tolerance the task needs. Past that, more of them change nothing.

### Sim-to-Real: The Extra Data

On the real Franka, with nothing changed but the robot, the same policy picked up 3 objects in 10.

<img src="{{ '/assets/images/sim_to_real.png' | relative_url }}" alt="Real robot grasp success against episode budget, with failure mode at each budget" class="figure">

The failures say more than the number. It was not reaching and missing, it was driving the gripper into the table: a policy with no idea where the object is, not one slightly off.

At 390 episodes, 60%, and the crashes are gone, leaving missed reaches and dropped lifts. The failure mode changed before the success rate finished moving. At 650, five times the simulation requirement, 90%.

Five times, not fifty. Fifty assumed the policy has to see combinations of the randomized settings. At 650 episodes, 87% of the sky, table, colour and position combinations never occurred once, and it scored 90% anyway. Each setting only has to be covered on its own, and every episode redraws all of them at once: 96% of each range by 130 episodes, 99% by 650.

That explains why the extra data is small, not why it is 5 and not 1. By 130 episodes no meaningful holes are left in any range, and 130 is where the robot was still hitting the table. Covering a range and learning to ignore it may be different problems, but the 650 is measured, not derived, and the ablation that would settle it has not been run.

The same 650 episodes spread across 8 objects still gives 80%. Every episode of every task samples the same randomization, so the requirement is per dataset, not per task: with a multi-task dataset already collected under randomization, new collection goes on new tasks.

### Conclusion

Around 600 episodes covers the randomization in light, camera angle, texture and background. With several tasks trained together, each needing roughly that many anyway, the simulation budget is close to what real collection would have cost in episodes.

Harder trajectories need more data, in simulation as on hardware. The analysis is what makes scaling a decision rather than a guess. A mustard bottle has several valid grasp modes. A facewash bottle is tapered, so a top-down grasp slips off it. cuRobo returns different trajectories for the same grasp pose depending on how close the object sits to the robot base. Each adds trajectory variety, and that needs more episodes.

Scaled that way, 15 objects at 10 000 episodes reaches nearly 80% on the real robot, with recoveries and with grasps at positions and orientations the policy was never trained on. Objects that were never in the dataset also work when their geometry is close to something that was.

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
