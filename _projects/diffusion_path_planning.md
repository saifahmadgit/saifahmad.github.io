---
layout: project
title: Diffusion Model for Path Planning from Scratch
order: 5
tech_tags: "Diffusion Model, PyTorch, Path Planning, Deep Learning"
---

<img src="{{ '/assets/images/Diffusion.png' | relative_url }}" alt="Diffusion model path planning" style="width:100%;height:auto;display:block;margin:0 0 24px 0;border-radius:8px;">

## Overview

An **image-conditioned diffusion model** built from scratch that generates collision-free paths on 2D grid maps for mobile robot navigation. Given a grid map with a marked start (green) and goal (red), the model denoises a path (blue) that navigates around obstacles, learning the distribution of valid trajectories rather than following a fixed planning algorithm.

## Architecture

The model is a **U-Net denoising network** conditioned on the occupancy grid image. Key design choice: **attention mechanisms at the bottleneck and at the encoder/decoder layers surrounding it**. Standard U-Nets struggle when the start and goal are far apart, the bottleneck compresses spatial information too aggressively, losing the long-range context needed to connect distant points. Adding attention at these layers lets the model reason over the full map extent during denoising, substantially improving path connectivity.

The forward diffusion process gradually adds noise to ground-truth paths; the reverse process learns to denoise from pure noise back to a valid path conditioned on the map image.

## Training

- **75 epochs** on a procedurally generated dataset of 2D grid maps with randomized obstacle placements, start, and goal positions
- Dataset generation is configurable: grid size and obstacle density are adjustable parameters
- Checkpoints saved every 25 epochs to track denoising quality progression
- Batch size tuned for limited GPU memory

The model learns to generate paths that avoid obstacles and connect start to goal, though the specific route may differ from ground-truth since many valid paths exist for a given map, the model samples from the distribution of plausible trajectories.

## Inference

An interactive **Gradio web GUI** allows drawing custom grid maps and querying the model in real time, making it straightforward to probe the model's behavior on novel obstacle configurations.

## Code

<div style="display:flex;gap:20px;flex-wrap:wrap;margin:20px 0;width:100%;">
  <a href="https://github.com/saifahmadgit/Diffusion_model_for_path_planning" target="_blank" rel="noopener"
     style="flex:1;min-width:280px;display:flex;align-items:flex-start;gap:18px;background:#eaecf4;border-radius:8px;padding:24px 28px;text-decoration:none;color:#111;">
    <svg height="36" width="36" viewBox="0 0 16 16" fill="#111" style="flex-shrink:0;margin-top:3px;"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
    <div>
      <div style="font-weight:700;font-size:1.1rem;margin-bottom:6px;">saifahmadgit / Diffusion_model_for_path_planning</div>
      <div style="font-size:0.95rem;color:#555;">PyTorch, U-Net with Attention, Gradio</div>
    </div>
  </a>
</div>
