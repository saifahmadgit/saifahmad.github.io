---
layout: project
title: Pen Catcher Robot
tags: Control, Vision, Real-Time
gif: /assets/gifs/pen_demo.gif
---

<iframe class="video" src="https://www.youtube.com/embed/lAsviFo0WxI" allowfullscreen></iframe>

---

## Overview
This project showcases a **real-time vision-based control system** designed to catch a falling pen using a robotic arm.  
The goal was to demonstrate **dynamic motion control** and **low-latency visual feedback** for intercepting a rapidly moving target — a challenging benchmark in robotic control due to the system’s tight timing constraints.

---

## System Architecture
The entire workflow integrates **high-speed computer vision** with a **predictive control loop**.  

1. **Object Detection & Tracking**
   - A high-frame-rate camera continuously captures the workspace.  
   - **OpenCV-based motion tracking** isolates the pen in flight by analyzing frame differences and centroid motion.  
   - The pen’s 2D trajectory is projected to a 3D coordinate using camera calibration data.

2. **Trajectory Prediction**
   - A **Kalman Filter** estimates the pen’s velocity and acceleration in real time.  
   - The predicted intersection point (catch location) is computed within **tens of milliseconds** of release.  
   - This prediction compensates for communication and actuation latency in the robot’s control loop.

3. **Robotic Control**
   - The robot arm (Franka Emika Panda or similar) uses **inverse kinematics (IK)** to compute the optimal joint configuration for interception.  
   - A **PD control law** refines motion during final approach to ensure smooth and stable capture.  
   - Timing synchronization between perception and control is maintained using ROS 2 publisher–subscriber nodes running at 100 Hz.

---

## Technical Highlights
- **Frameworks:** ROS 2, OpenCV, NumPy, MoveIt 2  
- **Languages:** Python, C++  
- **Average Latency:** ~80 ms perception-to-action loop  
- **Catch Success Rate:** ~70% in controlled trials  
- **Key Algorithms:** Kalman filtering, inverse kinematics, motion prediction  
- **Camera:** 60 fps RGB camera calibrated with OpenCV  

---

## Key Learnings
- Implemented a **closed-loop real-time control system** integrating computer vision and robot actuation.  
- Learned to handle **timing jitter** and **latency compensation** between asynchronous ROS 2 nodes.  
- Optimized **Kalman filter tuning** for fast-moving objects with minimal overshoot.  
- Gained insights into the trade-offs between computational speed and physical precision.

---

## Future Work
- Replace classical filters with **neural motion predictors (e.g., LSTM-based)** for better extrapolation.  
- Introduce **multi-camera triangulation** for more accurate 3D trajectory estimation.  
- Explore **model predictive control (MPC)** for optimal trajectory interception under actuator constraints.  
- Integrate with **Isaac Gym** or **Genesis** for simulated reinforcement learning training.

---

## Media
**Pen Catcher Robot Demo:**  
<iframe class="video" src="https://www.youtube.com/embed/lAsviFo0WxI" allowfullscreen></iframe>  
> Demonstrates high-speed visual tracking and predictive control enabling a robot arm to catch a free-falling pen in real time.
