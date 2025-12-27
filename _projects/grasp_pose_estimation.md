---
layout: project
title: Prompt-to-Pose Grasp Estimation
tags: Grasping, Vision, Learning
gif: /assets/gifs/grasp_demo.gif
---

<iframe class="video" src="https://www.youtube.com/embed/QoFd4tUE89s" allowfullscreen></iframe>

---

## Overview
This project presents a **prompt-to-pose grasp planning pipeline** that allows a robot to understand a **natural language instruction** (e.g., *“pick up the red mug”*) and output **6-DoF grasp poses** for that object — even in cluttered environments.  
It combines **grounded vision-language models** with **3D grasp estimation networks**, bridging perception and manipulation through learning-based inference.

---

## System Architecture
The pipeline integrates multiple perception and learning modules, each playing a distinct role:

1. **Text-Conditioned Object Localization (Grounding DINO)**
   - Accepts a user prompt describing the target object.
   - Uses **Grounding DINO** to detect and localize the object in 2D based on both image and text cues.

2. **Segmentation Refinement (SAM 2)**
   - The detected bounding box is refined into a **precise segmentation mask** using **Segment-Anything Model 2 (SAM 2)**.
   - Produces an accurate pixel-level mask that isolates the object from clutter.

3. **3D Reconstruction and Grasp Estimation (Contact-GraspNet)**
   - The segmented RGB-D data is converted into a **point cloud**.
   - **NVIDIA Contact-GraspNet** predicts a ranked set of 6-DoF grasp poses (position, orientation, and gripper width).
   - The top-scoring grasp is sent to the manipulator’s motion planner for execution.

---

## Technical Highlights
- **Frameworks:** PyTorch, Open3D, Grounding DINO, SAM 2, Contact-GraspNet  
- **Languages:** Python  
- **Input:** RGB-D image + natural-language text prompt  
- **Output:** Ranked 6-DoF grasp poses with confidence scores  
- **Average runtime:** ~1.2 s per frame on RTX 4090 GPU  
- **Applications:** Vision-language grasping, object retrieval, robotic assistants

---

## Key Learnings
- Designed a modular perception stack integrating **transformer-based grounding** and **point-cloud inference**.  
- Improved grasp reliability by filtering low-confidence poses using geometric constraints.  
- Learned to bridge **language understanding and robot action** through intermediate vision representations.  
- Validated end-to-end pipeline in both **simulation and offline datasets** before hardware testing.

---

## Future Work
- Integrate with **Franka or Unitree manipulators** for real-world validation.  
- Introduce **reinforcement learning** for adaptive grasp correction during contact.  
- Extend to **multi-object scenes** with dynamic prompt updates (“pick the smallest cup,” “move the blue block next”).  
- Explore **foundation models for robotic manipulation** (e.g., RT-X, OpenVLA).

---

## Media
Below is the demo of the prompt-to-pose grasp estimation pipeline in action:

<iframe class="video" src="https://www.youtube.com/embed/QoFd4tUE89s" allowfullscreen></iframe>
> Demonstrates end-to-end inference from natural-language prompt to 6-DoF grasp poses in a cluttered tabletop scene.
