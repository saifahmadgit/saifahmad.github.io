---
layout: project
title: Franka Vision-Guided Pick and Place
tags: Grasping, Vision, Learning
gif: /assets/gifs/grasp_demo.gif
---


## Overview
This project demonstrates a **vision-guided robotic pick-and-place system** built on the **Franka Emika Panda manipulator**, integrating perception, planning, and control modules in **ROS 2** and **MoveIt 2**. The goal was to enable the robot to autonomously detect, identify, and manipulate objects in its workspace with minimal human supervision.

---

## System Architecture
The complete system consists of three major subsystems:

1. **Perception Layer (Intel RealSense + YOLO)**
   - The Intel RealSense D435 RGB-D camera captures color and depth data in real time.
   - A YOLOv8 model performs **object detection**, outputting 2D bounding boxes.
   - Depth information is fused with camera intrinsics to compute **3D object poses**, which are then passed to the planning module.

2. **Planning and Control Layer (MoveIt 2 + ROS 2 Actions)**
   - The 3D poses are translated into target frames for the Franka end-effector.
   - MoveIt 2 computes collision-free trajectories using the **OMPL** planner.
   - The trajectories are executed via **ROS 2 action servers**, with real-time feedback for execution monitoring.

3. **Grasp Execution**
   - The Franka gripper closes with adaptive force control once contact is detected.
   - The robot lifts the object and places it accurately at the designated goal location.

---

## Technical Highlights
- **Frameworks:** ROS 2 Humble, MoveIt 2, OpenCV, YOLOv8, Intel RealSense SDK  
- **Languages:** Python, C++  
- **Hardware:** Franka Emika Panda, Intel RealSense D435  
- **Average success rate:** 90% real-world grasp-and-place accuracy  
- **Cycle time:** ~12 s per object (detection → grasp → placement)  
- **Simulation:** Full pipeline replicated in **Gazebo** for testing perception and planning modules before real deployment.

---

## Key Learnings
- Integrated asynchronous ROS 2 nodes for perception and planning without blocking the control loop.
- Tuned MoveIt 2 planning parameters to handle tight workspaces with better trajectory smoothness.
- Calibrated camera and robot frames precisely using hand-eye calibration for sub-centimeter accuracy.
- Learned to bridge high-level perception (YOLO) with low-level control (Franka state feedback).

---

## Future Work
- Incorporate **semantic segmentation** (via **SAM 2**) for improved object localization in cluttered scenes.
- Implement **reinforcement learning-based grasp refinement** using Contact-GraspNet or Diffusion Policies.
- Add **human–robot collaboration features** for shared workspace safety.

---

## Media
Below are recorded demos showcasing the system in action:

**Real Robot Demo:**  
<iframe class="video" src="https://www.youtube.com/embed/L8qCKQ8qogY" allowfullscreen></iframe>

**Simulation Demo:**  
<iframe class="video" src="https://www.youtube.com/embed/MUO35U0UXR4" allowfullscreen></iframe>
