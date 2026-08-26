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

Robotics foundation models are the strongest starting point we have for manipulation. What none of the open-weight ones do is arrive ready for your robot. Unless your setup happens to match a configuration already in the pretraining mix, the model needs demonstrations from that exact embodiment: those cameras, that mounting geometry, that gripper. Those demonstrations come from a human teleoperating the arm, one trajectory at a time, and the bill is paid again for every new task, gripper, or camera placement.

This project removes the human from that loop. **Pi 0.5** is fine-tuned entirely on demonstrations generated in **NVIDIA Isaac Sim**, with **cuRobo** planning the motions, then deployed **zero-shot on a real Franka**, with no teleoperation and not one frame of data from the real setup. The task is picking: the policy is given an object's name and has to pick it up off the table. It succeeds 90% of the time on hardware it has never seen.

Picking is decided by vision. The gripper has to arrive at the right position and orientation, and the only evidence for where the object is comes from the cameras, so a miss is unambiguous and the success rate is a clean readout on whether the transfer held. The simulation does not have to look real. It has to vary enough that a real camera image looks like one more variation the policy has already learned to ignore. That is why the same recipe should carry to **visually guided manipulation** in general, to any task where success also comes down to what the cameras see and how precisely the arm can be placed from it: stacking, placing, aligning a tool to a fixture.

The second question is what this costs in data. Simulated episodes are cheap, but training has to span a wide range of lighting, textures and camera placements rather than the single real setup, and that coverage costs episodes. The measurement: 130 demonstrations to learn the task in simulation, and 650 to carry it onto the real robot. Five times as many, not fifty. The surcharge turns out to be per-dataset rather than per-task: eight objects trained together reach 80% on the real robot for roughly what the single task cost on its own.

## Pipeline

<img src="{{ '/assets/images/blockDiagram_Pi0.5_Sim_to_Real.png' | relative_url }}" alt="End-to-end pipeline from Isaac Sim data generation to real Franka deployment" class="figure-wide">

### Data Collection

The real cameras are calibrated first, recovering the 6D pose of each one in the robot base frame. Those poses are then reproduced in simulation, so the policy sees the same viewpoints in Isaac Sim that it will see on hardware.

15 objects, each episode paired with a randomized prompt so the policy can be told which object to pick. **cuRobo** plans the motion and grasps are computed from object geometry and inertia, so nothing is teleoperated.

**PhysX** runs at 60 Hz and rendering at 30 Hz, both recorded at 30 Hz, converted to **LeRobot v2.1**, and pushed to Hugging Face. One **NVIDIA RTX 6000 Ada Generation** collects a 10 second episode every 2 minutes.

Holding that rate is what drives the asset approximations. GraspNet objects are converted to **USD** and decimated from 0.5 to 1.5 M triangles down to 2 000 faces by quadric edge collapse, with bounding box drift under 0.1 mm so the grasp annotations stay valid. Render mesh, PhysX collider, and cuRobo obstacle all come from that one mesh. Collision uses a coarsened convex decomposition (20 k voxels, 16 hulls), anything above 0.90 convexity collapsed to a single hull.

Domain randomization covers light (intensity, temperature, position), object texture, background, and camera pose.

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

The question is how many demonstrations a task needs. Sim-to-sim transfer answers it first: train in simulation, test in simulation on placements the policy was never shown, and read off the episode count where it starts working. That number is the baseline, and the sim-to-real budget is measured against it.

Two objects set the scale. A cylinder has to be gripped near its centre or it slips, so picking it up measures precision in x and y alone. A cuboid adds yaw, since it has to be approached along the right axis as well. The gap between the two is the price of one extra degree of freedom, which makes it possible to estimate the budget for tasks with longer trajectories or several valid ways to succeed.

### Sim-to-Sim Baseline

The task is picking a 4 cm cylinder off a 50 by 15 cm patch of table, scored at positions the policy was never shown.

<img src="{{ '/assets/images/reach_map.png' | relative_url }}" alt="Grasp success across the table for the cylinder under three demonstration layouts" class="figure">

Five demonstration sites at 10 episodes each is 50 episodes, and the policy drops half the objects it is asked for in the spaces between those sites. Thirteen sites demonstrated once each is 13 episodes, a quarter of the data, and it picks up the object in every gap it is tested in. Returning to collect 5 episodes at each of those same 13 sites, 65 in total, changes nothing. It was already at 100% and stays there.

What separates the two is not how much data there is but how far any point on the table sits from the nearest demonstration. Five sites leave a worst spot nearly 14 cm from anything the policy was shown. Thirteen bring that under 7 cm, less than two object widths, and below that distance the policy fills in the gap on its own. More episodes at positions it already knows buy nothing.

Orientation behaves the same way, and the second figure holds the budget fixed to show it.

<img src="{{ '/assets/images/angles_map.png' | relative_url }}" alt="Grasp success across cuboid yaw angles for three ways of spending the same 130 episodes" class="figure">

All three conditions cost the same 130 episodes, 13 sites at 10 episodes each, and differ only in how those 10 are spread across the cuboid's yaw. Two angles, flat and side-on, fails 60% of the time when the object lies at anything else. Four angles handles a little over three quarters. Drawing a fresh angle every episode, so no orientation is taught twice and the largest untaught gap closes to under 9 degrees, handles all of them. Same episode count, three ways of spending it, and only the last produces a policy that treats orientation as a continuous quantity rather than a short list of memorised poses.

Coverage is the expensive thing, not volume. 13 episodes were enough for position and 130 for orientation because in each case the demonstrations were spread finer than the tolerance the task needs. Once they are, collecting more of them is money spent on nothing.

### Sim-to-Real: The Extra Data

Everything above happened in simulation. Moving the same policy onto the real Franka, with nothing changed but the robot, it picked up 3 objects in 10.

<img src="{{ '/assets/images/sim_to_real.png' | relative_url }}" alt="Real robot grasp success against episode budget, with failure mode at each budget" class="figure">

The failures say more than the number. It was not reaching for the object and missing, it was driving the gripper into the table. That is what a policy does when it has no idea where the object is, not one that is slightly off, and it is why a 30% success rate at this stage tells you nothing about how far from working you are.

Tripling the data to 390 episodes moved it to 60%, and the catastrophic failures were gone. Every failure at that budget was a reach that missed or a lift that lost the object. The failure mode changed a full step before the success rate finished moving, so a policy at 60% with no crashes is much closer to done than the same 60% would have been earlier. At 650 episodes, 5 times what the task needed in simulation, it reached 90%, essentially the simulation number.

Five times, not fifty. The original budget assumed the policy has to see the combinations of everything being randomized, and that assumption is wrong. At 650 episodes, 87% of the possible sky, table, object colour and position combinations never occurred once, and the policy scored 90% anyway. Each setting only has to be covered on its own, and since every episode redraws every setting at once, a few hundred episodes cover all twelve ranges simultaneously: 96% of each range by 130 episodes, 99% by 650.

That arithmetic explains why the surcharge is small. It does not explain why it is 5 and not 1. By 130 episodes there are no meaningful holes left in any range, and 130 is exactly where the real robot was still driving into the table. The 650 is measured, not derived. My reading is that covering a range and learning to ignore it are different purchases and the second is the expensive one, but that is an interpretation, and the ablation that would settle it, 650 episodes with the randomization ranges collapsed, has not been run.

The last result changes what any of this costs. Spread the same 650 episode budget across 8 objects instead of 1 and the real robot still runs at 80%, with no catastrophic failures anywhere. The surcharge was never per task. Every episode of every task pays into the same account because the randomization they draw from is shared, so the bill arrives once for the dataset and the ninth task does not pay it again. If you already have a large multi-task dataset collected under randomization, you have already paid for transfer, and new collection should go on new tasks.

### Conclusion

Closing the visual gap takes on the order of 600 episodes, enough to cover the randomization in light, camera angle, texture and background. Once several tasks are trained together, each needing roughly that many episodes anyway, the simulation budget is not much larger in episode count than real collection would have been.

More varied tasks and harder trajectories need more data, which is equally true of collecting on real hardware. What the analysis buys is the ability to decide how far to scale rather than guess. A mustard bottle has several valid grasp modes and awkward approaches. A facewash bottle is tapered, so a top-down grasp slips off it. cuRobo returns different trajectories for the same grasp pose depending on how close the object sits to the robot base. Each of those adds trajectory variety that has to be paid for in episodes.

Scaled on that basis, 15 objects over a wider range of trajectories at 10 000 episodes reaches nearly 80% on the real robot, with recoveries and with grasps at positions and orientations the policy was never trained on. Objects that were never in the dataset at all also work, provided their geometry is close to something that was.

## Future Work

**Contact-rich manipulation**, where interaction forces rather than the visual scene decide success. Contact dynamics are the hardest thing for a simulator to get right, so the open question is how far simulation-only data goes before real contact data becomes unavoidable.

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
