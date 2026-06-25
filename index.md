---
layout: home
title: Portfolio
---

<!-- 1) Quadruped -->
<div class="project-row">
  <div class="project-media">
    <img src="{{ '/assets/gifs/RL_overview.gif' | relative_url }}" alt="Unitree Go2 RL Locomotion" loading="lazy">
  </div>
  <div class="project-info">
    <p><a class="project-title-link" href="{{ site.baseurl }}/projects/quadruped-locomotion-rl/"><strong>Sim-to-Real Reinforcement Learning Walking Including Stairs</strong></a></p>
    <p class="project-tags"><span class="tag">Reinforcement Learning</span><span class="tag">Genesis Simulation</span><span class="tag">Sim-to-Real Transfer</span></p>
    <p>Closing the sim-to-real gap for quadruped locomotion. PPO policies trained in Genesis with domain randomization, sensor noise injection, and latency modeling deploy directly on a real Unitree Go2. The robot achieves omnidirectional walking and blind stair climbing, using only proprioceptive sensing and a learned per-leg stiffness that adapts compliance in real time.</p>
    <p><a href="https://github.com/saifahmadgit/go2-sim2real-locomotion-rl" target="_blank" rel="noopener">GitHub ↗</a></p>
  </div>
</div>

<!-- 2) EKF SLAM -->
<div class="project-row">
  <div class="project-media">
    <img src="{{ '/assets/gifs/SLAM.gif' | relative_url }}" alt="EKF SLAM from Scratch" loading="lazy">
  </div>
  <div class="project-info">
    <p><a class="project-title-link" href="{{ site.baseurl }}/projects/ekf-slam/"><strong>EKF SLAM from Scratch in C++</strong></a></p>
    <p class="project-tags"><span class="tag">EKF SLAM</span><span class="tag">C++</span><span class="tag">ROS 2</span><span class="tag">TurtleBot3</span></p>
    <p>Extended Kalman Filter SLAM implemented entirely from scratch in C++ and deployed on a real TurtleBot3 Burger. Landmark detection via LiDAR clustering and circle fitting, Mahalanobis-distance data association, and simultaneous pose and map correction reduce localization error by up to 99.7% compared to pure odometry.</p>
    <p><a href="https://github.com/saifahmadgit/EKF_SLAM_from_Scratch" target="_blank" rel="noopener">GitHub ↗</a></p>
  </div>
</div>

<!-- 3) LeHome Challenge -->
<div class="project-row">
  <div class="project-media">
    <img src="{{ '/assets/gifs/lehome.gif' | relative_url }}" alt="LeHome Challenge Garment Manipulation" loading="lazy">
  </div>
  <div class="project-info">
    <p><a class="project-title-link" href="{{ site.baseurl }}/projects/lehome-challenge/"><strong>LeHome Challenge – ICRA 2026 Garment Manipulation Competition</strong></a></p>
    <p class="project-tags"><span class="tag">VLA</span><span class="tag">Diffusion Policy</span><span class="tag">Pi 0.5</span><span class="tag">Isaac Sim</span><span class="tag">LeRobot SO-ARM101</span></p>
    <p>Deploying fine-tuned Pi 0.5 VLA policies for deformable garment manipulation on the LeRobot SO-ARM101. Built an automated demonstration collection pipeline in Isaac Sim using cuRobo for GPU-accelerated motion planning, enabling large-scale sim data generation without manual teleoperation.</p>
    <p><a href="https://github.com/cwoodhayes/lehome-laundrynauts" target="_blank" rel="noopener">GitHub ↗</a></p>
  </div>
</div>

<!-- 4) Franka -->
<div class="project-row">
  <div class="project-media">
    <img src="{{ '/assets/gifs/franka_demo.gif' | relative_url }}" alt="Franka Vision-Guided Pick & Place" loading="lazy">
  </div>
  <div class="project-info">
    <p><a class="project-title-link" href="{{ site.baseurl }}/projects/franka/"><strong>Franka Vision-Guided Pick & Place</strong></a></p>
    <p class="project-tags"><span class="tag">ROS 2</span><span class="tag">MoveIt 2</span><span class="tag">YOLOv8</span><span class="tag">Intel RealSense</span></p>
    <p>A complete vision-guided pick-and-place system integrating YOLOv8 object detection, 3D pose estimation, and MoveIt 2 trajectory planning to autonomously grasp and place objects on a Franka Panda robot.</p>
    <p style="font-size:0.9rem;color:#555;">Group project with Robert, Florian, and Aravind</p>
    <p><a href="https://github.com/saifahmadgit/franka-vision-guided-manipulation" target="_blank" rel="noopener">GitHub ↗</a></p>
  </div>
</div>

<!-- 5) Diffusion Path Planning -->
<div class="project-row">
  <div class="project-media">
    <img src="{{ '/assets/images/Diffusion.png' | relative_url }}" alt="Diffusion Model for Path Planning" loading="lazy">
  </div>
  <div class="project-info">
    <p><a class="project-title-link" href="{{ site.baseurl }}/projects/diffusion-path-planning/"><strong>Diffusion Model for Path Planning from Scratch</strong></a></p>
    <p class="project-tags"><span class="tag">Diffusion Model</span><span class="tag">PyTorch</span><span class="tag">Path Planning</span><span class="tag">Deep Learning</span></p>
    <p>An image-conditioned diffusion model built from scratch that generates collision-free robot paths on 2D grid maps. A U-Net with attention at the bottleneck learns to denoise valid trajectories around obstacles, with a Gradio GUI for interactive inference.</p>
    <p><a href="https://github.com/saifahmadgit/Diffusion_model_for_path_planning" target="_blank" rel="noopener">GitHub ↗</a></p>
  </div>
</div>

<!-- 6) GPT-2 RL -->
<div class="project-row">
  <div class="project-media">
    <img src="{{ '/assets/images/transformer.png' | relative_url }}" alt="GPT-2 from Scratch with RL Fine-Tuning" loading="lazy">
  </div>
  <div class="project-info">
    <p><a class="project-title-link" href="{{ site.baseurl }}/projects/gpt2-rl-finetuning/"><strong>GPT-2 from Scratch with RL Fine-Tuning</strong></a></p>
    <p class="project-tags"><span class="tag">PyTorch</span><span class="tag">Transformers</span><span class="tag">PPO</span><span class="tag">NLP</span></p>
    <p>A decoder-only GPT-2 transformer built from scratch and trained on Sherlock Holmes, then fine-tuned with PPO to steer outputs toward dialogue using programmatic verifiable rewards — no learned reward model required.</p>
    <p><a href="https://github.com/saifahmadgit/GPT2_from_Scratch_with_RL_finetuning" target="_blank" rel="noopener">GitHub ↗</a></p>
  </div>
</div>

<!-- 7) Pen -->
<div class="project-row">
  <div class="project-media">
    <img src="{{ '/assets/gifs/pen_demo.gif' | relative_url }}" alt="Pen Catcher Robot" loading="lazy">
  </div>
  <div class="project-info">
    <p><a class="project-title-link" href="{{ site.baseurl }}/projects/pen-catcher/"><strong>Pen Catcher Robot</strong></a></p>
    <p class="project-tags"><span class="tag">ROS 2</span><span class="tag">OpenCV</span><span class="tag">Kalman Filter</span></p>
    <p>A real-time vision system that uses OpenCV tracking and Kalman filter prediction to intercept a falling pen with a robotic arm under tight latency constraints.</p>
    <p><a href="https://github.com/saifahmadgit/MS_Robotics_PenChallenge" target="_blank" rel="noopener">GitHub ↗</a></p>
  </div>
</div>

<!-- 8) Grasp -->
<div class="project-row">
  <div class="project-media">
    <img src="{{ '/assets/gifs/grasp_demo.gif' | relative_url }}" alt="Prompt-to-Pose Grasp Estimation" loading="lazy">
  </div>
  <div class="project-info">
    <p><a class="project-title-link" href="{{ site.baseurl }}/projects/grasp-pose-estimation/"><strong>Prompt-to-Pose Grasp Estimation</strong></a></p>
    <p class="project-tags"><span class="tag">Grounding DINO</span><span class="tag">SAM 2</span><span class="tag">Contact-GraspNet</span><span class="tag">6-DoF Grasping</span></p>
    <p>A natural language-guided grasp planning pipeline using Grounding DINO for object localization, SAM 2 for segmentation, and Contact-GraspNet to generate 6-DoF grasp poses from text descriptions.</p>
    <p><a href="https://github.com/saifahmadgit/prompt-guided-robotic-grasping" target="_blank" rel="noopener">GitHub ↗</a></p>
  </div>
</div>

<!-- 9) Jack -->
<div class="project-row">
  <div class="project-media">
    <img src="{{ '/assets/gifs/jack_demo.gif' | relative_url }}" alt="Jack-in-the-Box Dynamics" loading="lazy">
  </div>
  <div class="project-info">
    <p><a class="project-title-link" href="{{ site.baseurl }}/projects/jack-in-the-box/"><strong>Jack-in-the-Box Dynamics</strong></a></p>
    <p class="project-tags"><span class="tag">Hybrid Dynamical Systems</span><span class="tag">SymPy</span><span class="tag">SciPy</span></p>
    <p>Physics-based simulation and analysis of a Jack-in-the-Box mechanism, modeling the hybrid dynamics of spring oscillation combined with discrete impact events.</p>
  </div>
</div>
