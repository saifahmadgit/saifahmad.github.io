---
layout: project
title: GPT-2 from Scratch with RL Fine-Tuning
order: 6
tech_tags: "PyTorch, Transformers, PPO, NLP, Reinforcement Learning"
---

## Overview

A **decoder-only GPT-2 transformer built entirely from scratch in PyTorch** — tokenizer, causal attention, training loop, and sampling — then fine-tuned with **Proximal Policy Optimization (PPO)** to steer its outputs toward dialogue-heavy text. The project runs in two self-contained stages: base language model training on *The Adventures of Sherlock Holmes*, followed by an RL fine-tuning stage that uses a programmatic, verifiable reward instead of a learned preference model — the same "RLVR" setup now widely used in modern reasoning models.

The end result is two models you can compare side by side: a base storyteller that writes in the style of Victorian detective fiction, and an RL-tuned variant that converts the same prompt into back-and-forth dialogue.

---

## Stage 1 — Base Language Model

The architecture follows the GPT-2 spec built from first principles:

- **Pre-norm transformer**: LayerNorm is applied *before* each sub-layer (attention and MLP), which stabilises training without a warmup ramp.
- **Causal multi-head self-attention**: each head computes scaled dot-product attention masked with a lower-triangular buffer so each position can only attend to itself and earlier tokens.
- **GELU MLP**: a 4× expansion (256 → 1024 → 256) with a GELU nonlinearity, matching the original GPT-2 block.
- **Weight tying**: the token embedding matrix and the LM head share the same weights, halving the parameter count on the output projection.
- **Scaled residual init**: residual projection layers are initialised with std ∝ 1/√(2 · n_layer) so the residual stream variance stays ≈ 1 at init regardless of depth.

**Model config** (sized for the ~140k-token Sherlock Holmes corpus):

| Hyperparameter | Value |
|---|---|
| Context window | 256 tokens |
| Vocabulary | 50,257 (GPT-2 BPE) |
| Layers | 4 |
| Attention heads | 4 |
| Embedding dim | 256 |
| Dropout | 0.2 |

**Training**: 12,000 iterations with cosine LR decay (6e-4 → 6e-5), AdamW (β = 0.9 / 0.95), gradient clipping at 1.0, and Weights & Biases logging. The base model learns to generate coherent prose in the style of Conan Doyle.

---

## Stage 2 — RL Fine-Tuning with PPO

The RL stage is a minimal, from-scratch RLHF pipeline. The key design choice: replace the neural reward model with a **transparent, programmatic reward function** ("verifiable reward"). The PPO loop is otherwise identical to what InstructGPT / ChatGPT use.

### Cast of characters

| Role | What it is | Updated? |
|---|---|---|
| **Actor / policy** | The pretrained GPT-2 (LM head) | Yes — PPO updates its weights |
| **Critic / value head** | A new `Linear(256, 1)` on the shared transformer trunk | Yes — trained from scratch |
| **Reference** | A frozen copy of the base model | No — used only for the KL leash |

The actor and critic share the same transformer backbone (`trunk()`), so the critic reads the same representation of the token sequence that the policy uses to generate. Only the output heads differ.

### Reward design

The reward function scores a generated completion for **dialogue-ness** — how much of the text is characters talking inside properly closed quotation marks. It decomposes into three normalised sub-scores, each in [0, 1]:

```
reward = W_DIALOGUE × dialogue_fraction
       + W_PAIRS    × pair_score
       − W_UNBALANCED × stranded_fraction
```

| Term | What it measures | Weight |
|---|---|---|
| `dialogue_fraction` | Fraction of characters inside valid (closed, ≥ 3-char) quoted spans | 0.7 |
| `pair_score` | `min(valid_pairs / 2, 1.0)` — how many complete exchanges exist | 0.3 |
| `stranded_fraction` | Characters inside an *unclosed* quote at completion end | −1.0 (penalty) |

A perfect dialogue completion scores ≈ 1.0. Pure narration scores 0.0. The "open a quote and ramble forever" reward-hacking exploit goes **negative** — unclosed spans earn zero dialogue credit and drive the stranded penalty instead. Directional curly quotes (`"` / `"`) are parsed as strict open/close delimiters; the ambiguous straight `"` toggles, so a stray closing quote mid-stream can't invert the entire scoring state.

For GAE credit assignment, the dialogue credit is distributed **per token** (each token gets its proportional share of `dialogue_fraction`), while the pair bonus and stranded penalty attach to the last token where GAE propagates them backward.

### PPO update

Each iteration:
1. **Rollout** — sample 16 completions of 64 tokens from random 16-token prompts drawn from the training corpus.
2. **Reward** — score each completion with the dialogue reward function.
3. **KL leash** — subtract `β × KL(policy ‖ reference)` per token from the reward, where β is managed by an adaptive KL controller.
4. **GAE** — compute advantages and returns (γ = 1.0, λ = 0.95).
5. **PPO update** — 4 optimisation passes over the rollout batch (minibatch size 8), with clipped policy ratio (ε = 0.2), critic loss (weight 0.5), and entropy bonus (weight 0.01).

**Adaptive KL controller**: a proportional controller on β. Each iteration it measures `KL(policy ‖ reference)` across the batch and nudges β up if the policy is drifting too far (KL > target) or relaxes it if the policy is staying close (KL < target). This replaces a fixed penalty and prevents the slow unbounded KL drift that appeared in early fixed-β runs — the policy held close to the base model without needing manual tuning of the leash strength.

**Health diagnostics logged per iteration**: explained variance of the critic, PPO clip fraction, per-update approximate KL, gradient norm, and a distinct-token fraction to catch repetition collapse.

---

## Results

Reward rises quickly in the first ~500 PPO steps and then saturates, indicating convergence without fluency degradation:

<img src="{{ '/assets/images/gpt2_rewards.png' | relative_url }}" alt="Total reward over PPO training steps" style="width:100%;max-width:600px;height:auto;display:block;margin:20px 0;border-radius:6px;">

The clearest test: give both models the same prompt and seed and compare their completions. The base model drifts into repetitive narration; the RL fine-tuned model immediately shifts into back-and-forth dialogue:

<img src="{{ '/assets/images/gpt2_compare.png' | relative_url }}" alt="Side-by-side Gradio comparison of base and RL fine-tuned outputs" style="width:100%;height:auto;display:block;margin:20px 0;border-radius:6px;">

> Prompt: *"I was walking on the street"*, seed 1337. Base model (left): repetitive narration about doors. RL fine-tuned model (right): dialogue — *"Don't you know what?" / "Well, I am going down to take it back to the place."*

---

## Gradio Web UI

A Gradio interface lets you compare both models on one prompt without touching the command line. It loads the latest base and RL checkpoint automatically and runs both with identical sampling parameters (temperature, top-k, seed), so any behavioral difference is the fine-tuning — not random chance.

---

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
