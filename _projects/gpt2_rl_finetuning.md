---
layout: project
title: GPT-2 from Scratch with RL Fine-Tuning
order: 6
tech_tags: "PyTorch, Transformers, PPO, NLP, Reinforcement Learning"
---

<img src="{{ '/assets/images/transformer.png' | relative_url }}" alt="GPT-2 transformer architecture" style="width:100%;height:auto;display:block;margin:0 0 24px 0;border-radius:8px;">

## Overview

A **decoder-only transformer built from scratch in PyTorch**, trained on *The Adventures of Sherlock Holmes*, then fine-tuned with **Proximal Policy Optimization (PPO)** to shift its output toward dialogue-heavy text — without a learned reward model. The project demonstrates two self-contained stages: base language model training followed by reinforcement learning fine-tuning using programmatic, verifiable rewards (RLVR).

## Base Model

The GPT-2 architecture is implemented from first principles: custom tokenizer, multi-head self-attention, positional embeddings, and a full training loop. The model trains for **12,000 iterations** on the Sherlock Holmes corpus and learns to generate coherent detective-novel prose.

## RL Fine-Tuning

The fine-tuning stage replaces a learned preference model with a transparent reward function — scoring text inside closed quotation marks and penalizing unclosed quotes to prevent reward hacking. Key components:

- **PPO with Generalized Advantage Estimation** for stable policy gradient updates
- **Frozen reference model** for KL regularization — keeps the policy from drifting too far from the base model
- **Adaptive KL controller** that adjusts the regularization coefficient dynamically instead of using a fixed penalty
- Reward plateaus after ~500 steps, indicating clean convergence

The fine-tuned model converts narrative passages into back-and-forth dialogue while maintaining grammatical coherence.

## Interface

A **Gradio web UI** renders base model and RL-tuned outputs side by side with matched sampling parameters (temperature, top-k, seed), making behavioral differences immediately visible.

## Code

<div style="display:flex;gap:20px;flex-wrap:wrap;margin:20px 0;width:100%;">
  <a href="https://github.com/saifahmadgit/GPT2_from_Scratch_with_RL_finetuning" target="_blank" rel="noopener"
     style="flex:1;min-width:280px;display:flex;align-items:flex-start;gap:18px;background:#eaecf4;border-radius:8px;padding:24px 28px;text-decoration:none;color:#111;">
    <svg height="36" width="36" viewBox="0 0 16 16" fill="#111" style="flex-shrink:0;margin-top:3px;"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27.68 0 1.36.09 2 .27 1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.013 8.013 0 0016 8c0-4.42-3.58-8-8-8z"/></svg>
    <div>
      <div style="font-weight:700;font-size:1.1rem;margin-bottom:6px;">saifahmadgit / GPT2_from_Scratch_with_RL_finetuning</div>
      <div style="font-size:0.95rem;color:#555;">PyTorch, PPO, Gradio</div>
    </div>
  </a>
</div>
