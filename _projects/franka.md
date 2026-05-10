---
layout: project
title: Franka Vision-Guided Pick and Place
order: 2
tags: Grasping, Vision, Learning
gif: /assets/gifs/franka_demo.gif
---

**Group project with Robert, Florian, and Aravind** · Fall Quarter · ME 495 Embedded Systems in Robotics

---

<iframe class="video" src="https://www.youtube.com/embed/L8qCKQ8qogY" allowfullscreen></iframe>

## Overview

An autonomous pick-and-place system on the **Franka Emika Panda** using an **Intel RealSense D435** RGB-D camera. The robot detects objects on a table, pairs them to target locations, and executes placements using MoveIt 2 motion planning. The system handles **squares, rectangles, and cylinders**, recovers automatically from failed grasps, and adapts when objects or targets are repositioned mid-operation.

## System Architecture

**1. Calibration**: a `camera_calibration` node establishes coordinate transforms between camera, robot base, and an ArUco marker placed 0.3 m from the base. This single calibration step grounds all downstream 3D pose estimates in the robot frame.

**2. Perception**: the `grasp_node` runs YOLOv8 on the live RealSense stream, aggregating detections across multiple frames before committing to an object identity. Aggregation filters per-frame noise and handles the fact that YOLO detections carry no fixed ordering; objects and targets are paired programmatically after all detections stabilize.

**3. Planning and Execution**: a custom MoveIt 2 interface wraps OMPL trajectory planning with collision checking. The main orchestration script sequences home positioning, workspace scanning, object-target pairing, and iterative pick-and-place. Failed grasps trigger automatic re-scanning and retry rather than halting execution.

## Results

- **90% grasp-and-place success rate** across object categories in real-world trials
- **~12 s cycle time** per object (detection → grasp → placement)
- Robust to mid-operation repositioning of objects and targets

**Simulation Demo:**
<iframe class="video" src="https://www.youtube.com/embed/MUO35U0UXR4" allowfullscreen></iframe>

## Code

<div style="display:flex;gap:20px;flex-wrap:wrap;margin:20px 0;width:100%;">
  <a href="https://github.com/saifahmadgit/franka-vision-guided-manipulation" target="_blank" rel="noopener"
     style="flex:1;min-width:280px;display:flex;align-items:flex-start;gap:18px;background:#eaecf4;border-radius:8px;padding:24px 28px;text-decoration:none;color:#111;">
    <svg height="36" width="36" viewBox="0 0 16 16" fill="#111" style="flex-shrink:0;margin-top:3px;"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
    <div>
      <div style="font-weight:700;font-size:1.1rem;margin-bottom:6px;">saifahmadgit / franka-vision-guided-manipulation</div>
      <div style="font-size:0.95rem;color:#555;">ROS 2, MoveIt 2, YOLOv8, Intel RealSense</div>
    </div>
  </a>
</div>
