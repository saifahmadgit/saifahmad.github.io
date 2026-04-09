---
layout: project
title: Sim-to-Real Reinforcement Learning Walking Including Stairs
order: 1
tech_tags: "Reinforcement Learning (PPO), Sim-to-Real, Genesis"
gif: /assets/gifs/sim_to_real.gif
---

<iframe class="video"
        style="aspect-ratio: 16 / 9.95;"
        src="https://www.youtube.com/watch?v=m1GyvmJ9dBw"
        title="Sim-to-Real Reinforcement Learning Walking Including Stairs"
        frameborder="0"
        allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
        allowfullscreen></iframe>

## Overview

**PPO locomotion policies** trained in the **Genesis** physics simulator deploy on a real **Unitree Go2**, no camera, no LiDAR, proprioceptive sensing only. Four behaviors were developed in simulation: **omnidirectional walking**, **stair climbing**, **crouching**, and **jumping**; walking and stair climbing transfer successfully to hardware.

The core problem is the **sim-to-real gap**: unmodeled actuator dynamics, sensing delays, contact uncertainty, and terrain variation cause simulation-trained policies to fail on hardware. This work closes the gap through domain randomization, sensor noise and latency injection, metric-gated curriculum learning, and per-leg adaptive stiffness.

## Workflow

<img src="{{ '/assets/images/BigPicture_RL_training.png' | relative_url }}" alt="Training and deployment workflow" style="width:100%;height:auto;display:block;margin:16px 0;border-radius:8px;">

The pipeline runs in four stages with explicit feedback loops (see diagram):

**1. RL Training**: PPO runs across **4096 parallel environments** in Genesis. The **actor** is constrained to the 49 proprioceptive signals available on hardware (IMU, joint encoders, last action); the **critic** additionally receives privileged ground-truth quantities: friction, base velocity, mass distribution, push forces, terrain heights. This asymmetric design lets the critic produce accurate value estimates without the actor depending on information unavailable at deployment. DR difficulty is not fixed — a **metric-gated curriculum** advances it only after the policy sustains high timeout rate, velocity tracking, and low fall rate for 5 consecutive checks, and retreats three times faster than it advances to prevent the policy from getting stuck.

### Asymmetric Actor-Critic Training with Metric-Gated Curriculum

<img src="{{ '/assets/images/Training.png' | relative_url }}" alt="RL training pipeline with asymmetric actor-critic and curriculum" style="width:70%;height:auto;display:block;margin:16px auto;border-radius:8px;">

**2. Convergence Check**: reward curves and policy entropy are monitored via TensorBoard. Clean convergence is not expected: as the metric-gated curriculum advances DR difficulty and layers in noise, latency, and push forces incrementally, reward oscillates rather than settling. A checkpoint is carried forward if it shows stable gait and acceptable tracking, not necessarily maximum reward.

<div style="display:flex;gap:32px;flex-wrap:wrap;margin:16px 0;justify-content:center;">
  <figure style="width:33%;margin:0;text-align:center;">
    <img src="{{ '/assets/images/walk_train.png' | relative_url }}" alt="Training reward curve" style="width:100%;border-radius:6px;">
    <figcaption style="font-size:0.85rem;color:#555;margin-top:6px;">Training reward: oscillations visible as DR difficulty increases</figcaption>
  </figure>
  <figure style="width:33%;margin:0;text-align:center;">
    <img src="{{ '/assets/images/walk_entropy.png' | relative_url }}" alt="Policy entropy curve" style="width:100%;border-radius:6px;">
    <figcaption style="font-size:0.85rem;color:#555;margin-top:6px;">Policy entropy: never settles to zero, reflecting the shifting training distribution</figcaption>
  </figure>
</div>

**3. Evaluation**: the policy is stress-tested in simulation before any hardware trial. Gait naturalness, push recovery, friction and payload extremes are evaluated under teleoperation. Policies that fail visual inspection are discarded; only checkpoints that demonstrate robust behavior across these conditions proceed to hardware.

### Stress Test in Simulation

<img src="{{ '/assets/gifs/stress_test_sim.gif' | relative_url }}" alt="Stress test in simulation" style="width:100%;height:auto;display:block;margin:16px 0;border-radius:8px;">

**4. Deployment**: the actor runs at **50 Hz** on-robot; a parallel thread streams joint position targets and per-leg Kp/Kd commands over DDS at **500 Hz**, saturating the Go2's native LowCmd protocol. The robot ramps to a default stand pose over 4 seconds before the policy activates, and a slew limiter caps per-step joint target changes at 0.1 rad to prevent jerk. Hardware trials are not just validation — they directly shape the next training cycle. Watching the robot on real terrain reveals what the simulator fails to stress: a policy that looked stable in sim but buckled under an unexpected shove motivated adding push forces to DR; joints overshooting under load motivated learning per-leg stiffness rather than fixing gains; gait instability on real floors informed which reward terms needed reweighting. Each hardware run feeds back into reward design, architecture choices, and what gets randomized.

<img src="{{ '/assets/gifs/Deployment.gif' | relative_url }}" alt="Hardware deployment on Unitree Go2" style="width:100%;height:auto;display:block;margin:16px 0;border-radius:8px;">

## Design Decisions

**Metric-gated curriculum**: adding DR, noise, latency, and pushes simultaneously causes PPO to diverge. A metric-gated curriculum increases difficulty only after the policy demonstrates sustained performance across timeout rate, velocity tracking, and fall rate. Difficulty retreats three times faster than it advances, preventing catastrophic forgetting.

**Per-leg adaptive stiffness**: beyond 12 joint position targets, the policy outputs a stiffness scalar per leg. The stance leg stiffens to support body weight; the swing leg softens to absorb contact impulses. Fixed Kp/Kd cannot express this timing-dependent compliance; with learned stiffness the policy adapts emergently, and the behavior generalizes across surfaces without manual gain tuning.

**Stair climbing (blind)**: fine-tuned from the walking checkpoint on a 13-level heightfield curriculum (step heights 2–15 cm). The actor sees the same 49 proprioceptive signals; only the training critic receives a 77-point height scan for terrain context. Training starts at level 0.65 — the walking policy already handles low steps, so the curriculum begins mid-way. Advancement thresholds are relaxed (tracking ≥ 0.45 vs ≥ 0.75 for walking) since stair climbing inherently slows the robot. DR and terrain difficulty are decoupled: below level 0.5, DR is capped so the robot focuses on the stair task; above 0.5, full DR applies. Spawn distribution favors exploration — 40% frontier, 30% near-frontier, 30% easy — to prevent forgetting easier stairs while pushing to harder ones. Reward modifications: pitch not penalized, foot clearance computed terrain-relative, foot height target raised from 7.5 cm to 17 cm.

## Conclusions and Future Work

The sim-to-real gap is inherent: no simulator fully captures actuator compliance, contact mechanics, or real-world sensor characteristics. Domain randomization addresses this by making the real robot a member of the training distribution; rather than matching sim to reality exactly, it makes reality a subset of the simulated variations.

In practice the gap persists in subtle ways. Successful policies achieve the goal (walking, climbing stairs), but the motion strategies differ between simulation and hardware. The same Kp/Kd values produce noticeably different joint behavior in the two settings, pointing to unmodeled actuator compliance and transmission elasticity.

**Future directions:**

- **System identification for actuators**: fitting simulator actuator parameters to real motor response (frequency response, current-to-torque mapping) would substantially reduce the PD gain mismatch that is currently compensated by the KP_FACTOR deployment knob
- **Exteroceptive sensing**: adding a depth camera or LiDAR to the actor observation would let the robot perceive terrain ahead of time, likely closing the remaining reliability gap in stair climbing
- **High-level policy**: a hierarchical controller that uses LiDAR or camera perception to select between low-level locomotion policies (e.g. switching from flat-ground walking to stair-climbing mode upon detecting stairs)
- **Sample efficiency**: the current pipeline trains with 4096 parallel environments for up to 10,000 iterations, which works but is not particularly efficient. Improving sample efficiency through better reward shaping, off-policy methods, or more principled curriculum design was not a focus of this work and remains an open direction

## Inspirations

- **[Extreme Parkour with Legged Robots](https://extreme-parkour.github.io/)**: overall sim-to-real reinforcement learning framework: asymmetric actor-critic, privileged critic observations, and domain randomization strategy
- **[Variable Stiffness for Robust Locomotion through Reinforcement Learning (arXiv 2502.09436)](https://arxiv.org/abs/2502.09436)**: per-leg adaptive stiffness formulation (Kd = 0.2 × √Kp), and domain randomization parameter ranges for friction, mass, motor gains, and external push forces

## Code

Both repositories are forks; this work builds on top of existing infrastructure with significant modifications for the training pipeline and deployment stack.

<div style="display:flex;gap:20px;flex-wrap:wrap;margin:20px 0;width:100%;">
  <a href="https://github.com/saifahmadgit/go2-sim2real-locomotion-rl" target="_blank" rel="noopener"
     style="flex:1;min-width:280px;display:flex;align-items:flex-start;gap:18px;background:#eaecf4;border-radius:8px;padding:24px 28px;text-decoration:none;color:#111;">
    <svg height="36" width="36" viewBox="0 0 16 16" fill="#111" style="flex-shrink:0;margin-top:3px;"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
    <div>
      <div style="font-weight:700;font-size:1.1rem;margin-bottom:6px;word-break:break-all;">saifahmadgit / go2-sim2real-locomotion-rl</div>
      <div style="font-size:0.95rem;color:#555;">Reinforcement Learning Training, Genesis + PPO</div>
    </div>
  </a>
  <a href="https://github.com/saifahmadgit/go2-sim2real-deploy" target="_blank" rel="noopener"
     style="flex:1;min-width:280px;display:flex;align-items:flex-start;gap:18px;background:#eaecf4;border-radius:8px;padding:24px 28px;text-decoration:none;color:#111;">
    <svg height="36" width="36" viewBox="0 0 16 16" fill="#111" style="flex-shrink:0;margin-top:3px;"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
    <div>
      <div style="font-weight:700;font-size:1.1rem;margin-bottom:6px;">saifahmadgit / go2-sim2real-deploy</div>
      <div style="font-size:0.95rem;color:#555;">Hardware Deployment, Unitree Python SDK</div>
    </div>
  </a>
</div>

## Slides

<a href="{{ '/assets/quaduped_locomotion_Rl.odp' | relative_url }}" download style="display:inline-flex;align-items:center;gap:10px;background:#eaecf4;border-radius:8px;padding:16px 24px;text-decoration:none;color:#111;font-weight:600;">
  <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="#111" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="12" y1="18" x2="12" y2="12"/><line x1="9" y1="15" x2="15" y2="15"/></svg>
  Download Slides (.odp)
</a>

