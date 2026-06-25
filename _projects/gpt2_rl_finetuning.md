---
layout: project
title: GPT-2 from Scratch with RL Fine-Tuning
order: 6
tech_tags: "PyTorch, Transformers, PPO, NLP, Reinforcement Learning"
---

## Overview

A **decoder-only GPT-2 transformer built entirely from scratch in PyTorch** — custom tokenizer pipeline, causal multi-head self-attention, training loop, and sampling — then fine-tuned with **Proximal Policy Optimization (PPO)** to shift its writing style toward dialogue. The project runs in two self-contained stages: a base language model trained on *The Adventures of Sherlock Holmes*, and an RL fine-tuning stage that uses a programmatic, verifiable reward function instead of a learned preference model.

This "RLVR" (verifiable reward) setup is the same approach behind modern reasoning models: rather than training a neural reward model on human preferences, you define a reward you can compute and measure directly. The PPO machinery is otherwise identical to what InstructGPT and ChatGPT use in their final alignment stage.

The result is two models you can compare side by side: a base storyteller that generates Victorian detective prose, and an RL-tuned variant that shifts the same prompt into back-and-forth dialogue — with the behavioral difference entirely attributable to the fine-tuning, not to any difference in sampling.

---

## Stage 1 — Data Preparation and Tokenization

The corpus is *The Adventures of Sherlock Holmes* downloaded from Project Gutenberg. A data preparation script strips the boilerplate header and footer, splits into train/val, and tokenizes both splits with **tiktoken** using the GPT-2 BPE vocabulary (50,257 tokens). The tokenized data is cached to binary files (`train.bin` / `val.bin`) so subsequent training runs load directly from disk without re-tokenizing.

The corpus is deliberately small — approximately 130,000 BPE tokens — which is far smaller than the 40 GB GPT-2 was trained on. The model config is sized down to match: a 4-layer, 4-head, 256-embedding-dim architecture that avoids the extreme overfitting a full GPT-2 small (~117M params) would exhibit on this dataset.

---

## Stage 1 — Model Architecture

The architecture implements GPT-2 from first principles with no use of `transformers` or any high-level library:

**Pre-norm transformer blocks**: LayerNorm is applied *before* each sub-layer (attention and MLP) rather than after. This stabilises training without requiring an extended warmup ramp, and matches the pre-norm convention used in modern LLMs.

**Causal multi-head self-attention**: the sequence is split into `n_head` independent heads, each computing scaled dot-product attention with a lower-triangular causal mask registered as a buffer (not a learnable parameter). Each head operates on `n_embd // n_head = 64` dimensions. The outputs are concatenated and projected back to `n_embd` with a dropout-regularised linear layer.

**GELU MLP**: each block contains a 4× expansion feedforward network (`256 → 1024 → 256`) with a GELU nonlinearity, matching the original GPT-2 block design.

**Weight tying**: the token embedding matrix and the LM head projection share the same weight matrix. This halves the parameter count on the largest matrix in the network and acts as an implicit regulariser — the model cannot learn contradictory representations in the embedding and the unembedding layers.

**Scaled residual initialisation**: residual projection layers (`proj` inside both attention and MLP) are initialised with standard deviation ∝ 1/√(2 · n\_layer). This keeps the variance of the residual stream close to 1 at initialisation regardless of depth, preventing gradient explosion or vanishing before the first update.

**Shared trunk for actor and critic**: the transformer body exposes a `trunk()` method that returns the final hidden states without applying the LM head. In the RL stage, both the policy (LM head) and the value head (a separate `Linear(n_embd, 1)`) read from the same trunk, so the critic and actor share all representations up to their respective output projections.

**Model config**:

| Hyperparameter | Value |
|---|---|
| Context window | 256 tokens |
| Vocabulary | 50,257 (GPT-2 BPE) |
| Transformer layers | 4 |
| Attention heads per layer | 4 |
| Embedding dimension | 256 |
| Head dimension | 64 |
| MLP hidden dimension | 1,024 |
| Dropout | 0.2 |
| Total parameters | ~3M |

---

## Stage 1 — Training

**Optimiser**: AdamW with β = (0.9, 0.95) and weight decay 0.1. The higher β₂ and weight decay follow GPT-2's original recipe and stabilise training on the small corpus.

**Learning rate schedule**: linear warmup over 100 steps from 0 to 6e-4, then cosine decay to a floor of 6e-5 (one-tenth of peak LR). The cosine schedule prevents the abrupt LR drop of step decay and is standard in LLM pretraining.

**Gradient clipping**: global norm clipped at 1.0 every step to prevent occasional large gradient spikes from destabilising the weights.

**Evaluation**: every 500 steps, the training loop pauses, switches the model to eval mode, and estimates train and val loss over 100 mini-batches each. Checkpoints are saved at every evaluation point so any intermediate model can be loaded for generation or RL fine-tuning.

**Training run**: 12,000 iterations with batch size 8 and context window 256. Each step the model sees 2,048 tokens. Training and val losses are logged to **Weights & Biases** for monitoring convergence and overfitting. The base model learns to generate coherent detective-novel prose conditioned on a prompt.

---

## Stage 2 — RL Fine-Tuning with PPO

The RL stage is a minimal, from-scratch RLHF-style pipeline. The key design choice: replace the neural reward model (which requires a separate training phase on human preference data) with a **transparent, programmatic reward function** that directly measures the target behavior. The PPO loop is otherwise identical to what InstructGPT uses.

### Roles in the pipeline

| Role | What it is | Weights updated? |
|---|---|---|
| **Actor / policy** | The pretrained GPT-2 (LM head). Generates completions; its weights are what PPO optimises. | Yes |
| **Critic / value head** | A new `Linear(256, 1)` layer on top of the shared transformer trunk. Estimates expected return from each state. Trained from random init. | Yes |
| **Reference model** | A frozen copy of the base model loaded at the start of RL training. Used only to compute the KL divergence penalty. Its weights never change. | No |

The actor and critic share the same transformer backbone. This is efficient (one forward pass computes both the action distribution and the value estimate) and means the critic's advantage estimates are grounded in the same token representations the policy uses to make decisions.

### What the RL "environment" looks like

Unlike robotics RL where the environment is a simulator, here the environment is simply **text generation**:

- **State**: the current token sequence (prompt + tokens generated so far)
- **Action**: sampling the next token from the policy's distribution
- **Episode**: generating 64 tokens from a 16-token prompt sampled from the training corpus
- **Terminal reward**: the dialogue score of the completed 64-token continuation

This is an episodic, text-generation RL setup: each rollout is a complete episode of fixed length, and the reward is computed once the episode is done.

---

## Reward Design

The reward function scores a generated completion for **dialogue-ness** — how much of the generated text consists of characters talking inside properly closed quotation marks. It decomposes into three normalised sub-scores, each bounded to [0, 1]:

```
reward = 0.7 × dialogue_fraction
       + 0.3 × pair_score
       − 1.0 × stranded_fraction
```

**`dialogue_fraction`** (weight 0.7): the fraction of total generated characters that fall inside *valid* quoted spans. A quoted span is only valid if it is properly closed and contains at least 3 characters — this prevents the model from earning credit by inserting bare quote characters.

**`pair_score`** (weight 0.3): `min(valid_pairs / 2, 1.0)` — rewards the model for producing multiple complete back-and-forth exchanges, not just one quoted phrase.

**`stranded_fraction`** (penalty, weight 1.0): the fraction of characters inside an *unclosed* quote at the end of the completion. This is the anti-hacking term.

A perfect dialogue completion scores ≈ 1.0. Pure narration scores exactly 0.0. The construction ensures the degenerate optimum — open a quotation mark and ramble forever to maximise `dialogue_fraction` — goes **negative**: unclosed spans earn zero dialogue credit and simultaneously drive the stranded penalty.

**Curly vs. straight quotes**: the corpus uses directional curly quotes (`"` / `"`). These are parsed as strict open/close delimiters (not toggles), because a stray closing `"` mid-stream (common when the model emits `,"`) would otherwise flip the parity state and invert the reward for the entire rest of the completion. The ambiguous straight `"` toggles state as usual.

**Per-token rewards for GAE**: PPO's Generalized Advantage Estimation requires a per-token reward signal for credit assignment. The dialogue fraction credit is distributed proportionally across tokens (each token earns its share of the characters it covers). The pair bonus and stranded penalty, which are completion-level signals, attach to the final token, and GAE propagates them backward through the episode.

---

## PPO Update Loop

Each iteration of the RL loop:

**1. Rollout** — sample 16 completions of 64 tokens from random 16-token prompts drawn from the training corpus. No gradient is tracked here.

**2. Score** — run each completion through the reward function to get per-token dialogue rewards.

**3. KL penalty** — compute `KL(policy ‖ reference)` per token from the log-probabilities of the current policy and the frozen reference model. Subtract `β × KL` from the reward signal. This "KL leash" prevents the policy from drifting so far from the base model that it loses fluency.

**4. GAE** — compute advantages and bootstrapped returns (γ = 1.0, λ = 0.95). Normalise advantages across the batch.

**5. PPO update** — run 4 optimisation passes over the rollout batch in minibatches of 8:
- **Policy loss**: clipped surrogate objective with ε = 0.2
- **Value loss**: MSE between predicted values and bootstrapped returns, weight 0.5
- **Entropy bonus**: encourages exploration by penalising overconfident distributions, weight 0.01
- Gradient norm clipped at 1.0 before each step

### Adaptive KL controller

Early runs used a fixed β for the KL penalty. The problem: the policy's KL relative to the reference slowly drifted upward for the entire 2,000-step run without ever stabilising, degrading text fluency as the policy moved further from the base model.

The fix is an **adaptive KL controller** (the same design as InstructGPT's): a proportional controller on β that nudges it up when the measured KL exceeds the target and relaxes it when the policy stays close. The error is clipped to ±20% per step so β never jumps discontinuously. With a target of 4.0 nats, the controller holds the policy at a productive distance from the base model — close enough to maintain fluency, far enough to actually learn dialogue.

### Health diagnostics logged per iteration

To catch training instabilities early, each iteration logs a full diagnostic breakdown to Weights & Biases:

| Metric | What it catches |
|---|---|
| Explained variance of value head | Critic quality — approaches 1.0 when critic predictions match actual returns |
| PPO clip fraction | Whether the clipping is binding; high → policy updates are too large |
| Approximate KL per update (Schulman k3) | How much each gradient step moves the policy |
| Gradient norm | Pre-clip magnitude; spikes indicate unstable gradients |
| Distinct-token fraction | Repetition / collapse detector — drops when the model starts looping |
| Per-reward-term decomposition | Which of the three reward terms actually dominates |

---

## Difficulties and Solutions

**Reward hacking via open quotes**: an early reward version scored any character inside quotation marks. The policy discovered it could open a single quote and generate arbitrary tokens forever, accumulating dialogue credit without ever writing real dialogue. The fix was to only commit a quoted span as valid once it is *closed* and contains at least 3 characters. Unclosed spans now accumulate the stranded penalty instead, turning the cheat strategy into a net loss.

**KL drift with fixed penalty**: with a fixed β = 0.02, the policy's KL relative to the reference crept from ~3.0 nats to ~4.4 nats (peak 6.3) over the course of 2,000 iterations without ever stabilising. The fluency of the generated text degraded visibly in later samples. Switching to the adaptive KL controller with a target of 4.0 nats and horizon 10,000 completions held the KL in the 3.5–4.5 nat band for the entire run and kept the generated text coherent.

---

## Results

Reward rises quickly in the first ~500 PPO steps as the policy learns the basic "write dialogue" behavior, then saturates and stays roughly flat for the remaining 1,500 steps. The adaptive KL controller prevents any visible fluency degradation — later samples are still coherent Sherlock Holmes-style text, just dialogue-heavy.

<img src="{{ '/assets/images/gpt2_rewards.png' | relative_url }}" alt="Total reward over 2000 PPO training steps" style="width:100%;max-width:600px;height:auto;display:block;margin:20px 0;border-radius:6px;">

The clearest demonstration: give both models the same prompt and the same random seed, so any difference is the fine-tuning, not random variation.

<img src="{{ '/assets/images/gpt2_compare.png' | relative_url }}" alt="Side-by-side Gradio comparison: base model (left) vs RL fine-tuned model (right)" style="width:100%;height:auto;display:block;margin:20px 0;border-radius:6px;">

> Prompt: *"I was walking on the street"*, temperature 0.55, seed 1337.
> **Base model (left)**: drifts into repetitive narration — *"and I had always seen the door and the door of the bed..."*
> **RL fine-tuned model (right)**: immediately shifts into dialogue — *"Don't you know what?" / "Well, I am going down to take it back to the place."*

---

## Gradio Web UI

A Gradio interface loads the latest base and RL checkpoint automatically and runs both with identical sampling parameters (temperature, top-k, seed), so any behavioral difference is the fine-tuning, not random chance. The UI exposes sliders for max tokens, temperature, and seed, and renders both completions side by side in real time. On a remote or headless server, Gradio generates a temporary public `gradio.live` URL so the interface can be reached from any browser.

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
