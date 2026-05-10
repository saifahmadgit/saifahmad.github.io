---
layout: project
title: EKF SLAM from Scratch in C++
order: 2
tech_tags: "EKF SLAM, C++, ROS2, TurtleBot3"
gif: /assets/gifs/SLAM.gif
---

<iframe class="video"
        style="aspect-ratio: 16 / 9.95;"
        src="https://www.youtube.com/embed/qi1uTT5GMQc"
        title="EKF SLAM from Scratch in C++"
        frameborder="0"
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
        allowfullscreen></iframe>

## Overview

**Extended Kalman Filter SLAM** implemented entirely from scratch in **C++** and deployed on a real **TurtleBot3 Burger**. The system simultaneously estimates robot pose and builds a landmark map using only a 2D LiDAR, achieving up to 99.7% reduction in localization error compared to pure odometry.

Built as part of ME495 (Sensing, Navigation and Machine Learning for Robotics), every component (landmark detection, data association, and the EKF update equations) is written from first principles with no external SLAM library. Three robots are visualized simultaneously in RViz: a **red** robot tracking ground truth, a **blue** robot running pure odometry, and a **green** robot running the EKF-SLAM estimate.

## System Architecture

The project is organized into four ROS2 packages:

**nuturtle_description**: Robot URDF/xacro with configurable physical parameters.

**nusim**: A 100 Hz simulator with sensor noise modeling, used for development and validation before deploying to hardware.

**nuturtle_control**: Motor control, odometry computation, and a circle-driving node for controlled test trajectories.

**nuslam**: The complete SLAM implementation: landmark detection, data association, and the EKF estimator.

## Landmark Detection Pipeline

The robot perceives its environment through a 2D LiDAR scan. Cylindrical landmarks (physical cylinders placed in the environment) are extracted in three stages:

**1. Clustering**: LiDAR returns are grouped by Euclidean distance. Points within a threshold distance of each other form a cluster; a new cluster starts when the gap exceeds the threshold.

**2. Circle classification**: Each cluster is tested using the inscribed angle theorem. For a circular arc, the angles from endpoints to interior points are consistent; clusters that pass this geometric test are classified as cylinders and rejected otherwise.

**3. Circle fitting**: The radius and center of each detected cylinder are estimated via algebraic circle fitting (Pratt/Taubin method), giving a precise 2D position for use in the EKF update step.

## EKF SLAM

The state vector contains the full robot pose (x, y, θ) and the 2D positions of all landmarks seen so far. The filter runs a predict-update cycle at each odometry step:

**Predict**: the robot pose estimate is propagated forward using the wheel odometry motion model; landmark estimates are held fixed; the covariance is inflated by a process noise model.

**Update**: each detected landmark is matched to an existing map entry using **Mahalanobis-distance gating**: the association cost is computed in the innovation space so that both distance and uncertainty are accounted for. Landmarks that exceed the gating threshold for all existing entries are initialized as new map features. The EKF update then corrects both pose and map simultaneously, and a corrected map-to-odometry transform is broadcast so that the full ROS2 TF tree remains consistent.

## Results on Real Hardware

After driving a closed loop on the TurtleBot3 Burger, final pose error was compared between odometry and EKF-SLAM:

| Metric | Odometry | EKF-SLAM |
|--------|----------|----------|
| X error | 0.321 m | 0.010 m |
| Y error | 0.116 m | 0.002 m |
| Yaw error | 12.571° | 0.044° |

EKF-SLAM reduces translational error by **97–98%** and angular error by **99.7%**; the robot returns almost exactly to its starting pose despite significant odometry drift.

## Design Decisions

**From-scratch C++ implementation**: using an existing SLAM library would have hidden the numerical details that matter most: how the Jacobians are derived, how the state vector grows as new landmarks are added, and how data association interacts with the covariance structure. Writing every matrix operation by hand forced engagement with these details and produced a system whose failure modes are fully understood.

**Mahalanobis-distance gating for data association**: nearest-Euclidean-neighbor matching ignores uncertainty and breaks down when the robot pose estimate has drifted. Mahalanobis gating uses the full innovation covariance, so a landmark far away in Euclidean space but consistent with the current uncertainty ellipsoid is still correctly associated. This is particularly important early in a run when position uncertainty is high.

**Simultaneous pose and map correction**: updating only the robot pose given matched landmarks (as in localization against a known map) would discard the correlations built up between pose and landmark estimates. The full EKF update propagates corrections through the joint covariance, which is why revisiting a previously mapped area can sharply reduce uncertainty even after significant drift.

## Code

<div style="display:flex;gap:20px;flex-wrap:wrap;margin:20px 0;width:100%;">
  <a href="https://github.com/saifahmadgit/EKF_SLAM_from_Scratch" target="_blank" rel="noopener"
     style="flex:1;min-width:280px;display:flex;align-items:flex-start;gap:18px;background:#eaecf4;border-radius:8px;padding:24px 28px;text-decoration:none;color:#111;">
    <svg height="36" width="36" viewBox="0 0 16 16" fill="#111" style="flex-shrink:0;margin-top:3px;"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
    <div>
      <div style="font-weight:700;font-size:1.1rem;margin-bottom:6px;">saifahmadgit / EKF_SLAM_from_Scratch</div>
      <div style="font-size:0.95rem;color:#555;">Full C++ implementation: landmark detection, EKF, ROS2 packages</div>
    </div>
  </a>
</div>
