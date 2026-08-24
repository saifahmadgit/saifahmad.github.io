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

Robotics foundation models now outperform task-specific policies across a wide range of manipulation problems. None of the open-weight ones, however, work zero-shot on a robot they have never seen. Each needs demonstrations from the exact setup it will be deployed on — that embodiment, those cameras, that mounting geometry — and those demonstrations are almost always collected by a human teleoperating the arm, one trajectory at a time. It is slow and expensive, and the cost is paid again from scratch for every new task, gripper, or camera placement.

This project removes the human from that loop. **Pi 0.5** is fine-tuned entirely on demonstrations generated in **NVIDIA Isaac Sim**, with **cuRobo** planning the motions, and the resulting policy is deployed **zero-shot on a real Franka** — no teleoperation, and no real-world data at any stage.

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
