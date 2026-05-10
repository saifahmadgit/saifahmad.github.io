---
layout: project
title: LeHome Challenge – ICRA 2026 Garment Manipulation Competition
order: 2
tech_tags: "VLA, Diffusion Policy, Pi 0.5, Isaac Sim, PyTorch, LeRobot SO-ARM101"
gif: /assets/gifs/lehome.gif
---

<iframe class="video"
        src="https://www.youtube.com/embed/SyU1j5k8MLs"
        title="LeHome Challenge – ICRA 2026 Garment Manipulation"
        frameborder="0"
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
        allowfullscreen></iframe>

## Overview

The **LeHome Challenge** is ICRA 2026's standardized benchmarking competition for robotic garment manipulation — a domain that sits at the frontier of deformable object handling, contact-rich manipulation, and visuomotor policy generalization. The task demands that a **LeRobot SO-ARM101** reliably fold, sort, and place garments despite their infinite configuration space and unpredictable dynamics.

Our team (**Laundrynauts**) is developing a pipeline that deploys **Vision-Language-Action (VLA)** policies — specifically **Pi 0.5** fine-tuned with **Diffusion Policy** — for robust end-to-end garment manipulation. The competition has two phases: a simulation phase (Feb–Apr 2026) evaluated in Isaac Sim, followed by an on-site real-world phase at ICRA 2026 (June, Vienna). The top 8 teams from simulation advance to hardware.

## Technical Approach

**VLA Policy Deployment**

Pi 0.5 is a flow-matching generalist robot policy pre-trained on diverse manipulation data. We fine-tune it on garment-specific demonstrations using **Diffusion Policy** as the action head, which handles the multimodal action distributions that arise from deformable object manipulation — the same garment task can be completed through many physically valid trajectories.

Fine-tuning targets the SO-ARM101 action space directly, mapping visual observations (wrist + overhead RGB) to joint position commands. The challenge is distributional shift: garments crumple, fold, and occlude themselves in ways that stress visual encoders trained on rigid objects.

**Simulation Data Collection with cuRobo**

A key bottleneck in manipulation learning is demonstration data. We built an **automated data collection pipeline in Isaac Sim** using **cuRobo** for GPU-accelerated motion planning. The pipeline procedurally generates garment configurations, plans collision-free trajectories to target grasp and placement poses, and records demonstrations at scale — eliminating the need for manual teleoperation for the bulk of the training set.

cuRobo's batched IK and trajectory optimization run entirely on GPU, making it feasible to generate thousands of diverse demonstrations across garment types and initial conditions. The resulting dataset covers a much wider configuration distribution than human-collected data alone.

**Sim-to-Real Transfer**

The simulation phase uses Isaac Sim's cloth simulation for garment physics. Transferring to hardware introduces the standard gap: simulated cloth dynamics differ from real fabric in stiffness, friction, and self-contact behavior. We address this through:

- **Domain randomization** over cloth physical parameters (stiffness, friction, mass distribution)
- **Visual augmentation** to reduce texture and lighting dependency in the visual encoder
- **Wrist camera emphasis** — close-range wrist views are more robust to the global appearance shift between sim and real than overhead views

## Competition Structure

| Phase | Timeline | Platform |
|---|---|---|
| Simulation | Feb – Apr 2026 | Isaac Sim + SO-ARM101 URDF |
| Real-World (Top 8) | June 2026, ICRA Vienna | LeRobot SO-ARM101 hardware |

## Code

<div style="display:flex;gap:20px;flex-wrap:wrap;margin:20px 0;width:100%;">
  <a href="https://github.com/cwoodhayes/lehome-laundrynauts" target="_blank" rel="noopener"
     style="flex:1;min-width:280px;display:flex;align-items:flex-start;gap:18px;background:#eaecf4;border-radius:8px;padding:24px 28px;text-decoration:none;color:#111;">
    <svg height="36" width="36" viewBox="0 0 16 16" fill="#111" style="flex-shrink:0;margin-top:3px;"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
    <div>
      <div style="font-weight:700;font-size:1.1rem;margin-bottom:6px;">cwoodhayes / lehome-laundrynauts</div>
      <div style="font-size:0.95rem;color:#555;">VLA Fine-tuning, Isaac Sim Data Pipeline, cuRobo</div>
    </div>
  </a>
</div>
