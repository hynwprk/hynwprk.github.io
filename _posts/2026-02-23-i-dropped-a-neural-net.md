---
layout: post
title: "I Dropped a Neural Net"
date: 2026-02-23 13:49:00 -0500
description: I was the first to solve Jane Street's puzzle, recovering the exact layer ordering of a shuffled 96-layer ResNet from a search space of 10^122.
tags: [machine-learning, neural-networks, combinatorics]
categories: [research]
related_posts: false
thumbnail: assets/img/publication_preview/dropped-neural-net.png
---

A recent [Dwarkesh Patel podcast](https://www.dwarkeshpatel.com) with John Collison and Elon Musk featured an interesting puzzle from Jane Street: they trained a neural net, shuffled all 96 layers, and asked to put them back in order.

I was the first to solve it!

---

## The puzzle

> *Oh no! I dropped an extremely valuable trading model and it fell apart into linear layers! I need to rebuild it before anyone notices, but I can't remember how these pieces go together, or how it was trained. All I have left are the pieces of the model and some historical data. Can you help me figure out how to put it back together?*

We're given 97 weight matrices (the "pieces") and 10,000 data points. The architecture is a 48-block ResNet where each block computes:

$$\text{Block}_k(x) = x + W_{\text{out}}^{(k)} \text{ReLU}\!\big(W_{\text{in}}^{(k)} x + b_{\text{in}}^{(k)}\big) + b_{\text{out}}^{(k)}$$

with $$W_{\text{in}}^{(k)} \in \mathbb{R}^{96 \times 48}$$ and $$W_{\text{out}}^{(k)} \in \mathbb{R}^{48 \times 96}$$.

{% include figure.liquid loading="eager" path="assets/img/dropped-neural-net/Generated_image.png" class="img-fluid rounded z-depth-1" caption="The 48-block ResNet. Each block applies input projection, ReLU, output projection, then a residual connection." %}

The problem decomposes into two sub-problems:

1. **Pairing.** Each residual block has an input projection and an output projection. They've been separated. Match them back up: $$48!$$ possibilities.
2. **Ordering.** Arrange the 48 reassembled blocks in the correct sequence: another $$48!$$ possibilities.

The combined search space is $$(48!)^2 \approx 10^{122}$$, more configurations than atoms in the observable universe.

{% include figure.liquid loading="eager" path="assets/img/dropped-neural-net/problem_diagram.png" class="img-fluid rounded z-depth-1" caption="The two-stage decomposition: first pair input/output projections, then order the reassembled blocks." %}

---

## Pairing via diagonal dominance

I trained a small model with the same architecture on the same data and noticed something: the product $$W_{\text{out}}^{(k)} W_{\text{in}}^{(k)}$$ for correctly paired layers has a **negative diagonal structure**. This isn't a coincidence. It follows from dynamic isometry.

**Why?** If a residual block satisfies dynamic isometry ($$\mathbb{E}[J_f^\top J_f] = I$$), then expanding the Jacobian $$J_f = I + J_r$$ gives:

$$\mathbb{E}[J_r + J_r^\top + J_r^\top J_r] = 0$$

Taking the trace and using the half-activation assumption ($$\mathbb{E}[D(x)] = \tfrac{1}{2}I$$):

$$\text{tr}(W_{\text{out}} W_{\text{in}}) = -\mathbb{E}[\|J_r(x)\|_F^2] < 0$$

This gives us a metric. For each candidate pair $$(i, j)$$, compute the **diagonal dominance ratio**:

$$d(i, j) = \frac{|\text{tr}(W_{\text{out}}^{(j)} W_{\text{in}}^{(i)})|}{\|W_{\text{out}}^{(j)} W_{\text{in}}^{(i)}\|_F}$$

Correct pairs: $$d \in [1.76, 3.28]$$. Incorrect pairs: $$d \in [0.00, 0.58]$$. Complete separation with a gap of 1.18. The Hungarian algorithm recovers all 48 pairs perfectly.

{% include figure.liquid loading="eager" path="assets/img/dropped-neural-net/pairing_histogram.png" class="img-fluid rounded z-depth-1" caption="Complete separation: 48 correct pairs (red) vs. 2,256 incorrect pairs (gray)." %}

{% include figure.liquid loading="eager" path="assets/img/dropped-neural-net/matrices_correct_vs_wrong.png" class="img-fluid rounded z-depth-1" caption="Top: correctly paired matrices show negative diagonal structure. Bottom: incorrect pairings show no structure." %}

{% include figure.liquid loading="eager" path="assets/img/dropped-neural-net/all48_diag_pairing.png" class="img-fluid rounded z-depth-1" caption="All 48 product matrices from the original model. Every block exhibits a negative diagonal. Traces range from -13.5 to -7.4." %}

---

## Ordering

With all 48 blocks correctly paired, we need to put them in order.

### Seed initialization

Sort blocks by their **delta-norm**, the mean magnitude of the residual contribution:

$$\delta_k := \mathbb{E}_{x \sim \mathcal{D}}\!\left[\|r_k(x)\|_2\right]$$

Residual perturbation magnitude tends to increase with depth in trained networks. Sorting by ascending $$\delta_k$$ gives MSE = 0.0368.

Surprisingly, a completely **data-free** proxy, sorting by $$\|W_{\text{out}}^{(k)}\|_F$$, also works (MSE = 0.0759).

{% include figure.liquid loading="eager" path="assets/img/dropped-neural-net/sort_delta_norm.png" class="img-fluid rounded z-depth-1" caption="Delta-norm seed ordering. Bar height = ground-truth position." %}

### Bradley-Terry ranking

For all $$\binom{48}{2} = 1{,}128$$ pairs, swap their positions in the seed ordering and measure the MSE change $$g_{AB}$$. Fit a margin-weighted Bradley-Terry model:

$$p_{A \prec B} = \frac{1}{1 + \exp(-g_{AB}/T)}$$

Sorting by BT strength yields MSE = 0.00299, a 12x improvement over the seed.

Out of $$\binom{48}{3} = 17{,}296$$ directed triples, only **66** (0.38%) exhibit a transitivity violation. These are spatially local (spanning < 15 ground-truth positions) and weak (median margin $$1.8 \times 10^{-4}$$ vs. overall median 0.041).

{% include figure.liquid loading="eager" path="assets/img/dropped-neural-net/sort_bt_complete.png" class="img-fluid rounded z-depth-1" caption="After BT reranking, much closer to the correct ordering." %}

### Bubble repair

Finally, greedy adjacent-swap hill-climbing on MSE: sweep through all 47 adjacent pairs, swap if it reduces MSE, repeat until a full sweep produces zero swaps.

| Round | Swaps | MSE | Cumulative swaps |
|-------|-------|-----|------------------|
| 0 (BT init) | - | 0.002989 | 0 |
| 1 | 21 | 0.001025 | 21 |
| 2 | 7 | 0.000553 | 28 |
| 3 | 5 | 0.000200 | 33 |
| 4 | 3 | 0.000052 | 36 |
| 5 | 1 | **0.000000** | 37 |

Five rounds, 37 swaps, exact solution.

---

## You don't even need Bradley-Terry

In practice, direct hill-climbing from the delta-norm seed converges to MSE = 0 on its own, no pairwise comparisons needed. It takes 13 rounds and 122 swaps instead of 5 rounds and 37, but the BT step requires 1,128 forward passes for the pairwise comparisons, so the end-to-end cost is comparable.

Even more surprising: the $$\|W_{\text{out}}\|_F$$ seed (which is *data-free* and starts at a *higher* MSE) converges faster: 6 rounds, 72 swaps. I don't fully understand why.

---

## Open questions

- **What is this data?** I spent time trying to figure out what the features represent and how they were processed. Very difficult, possibly synthetic. I'm curious to find out.
- **Why is delta-norm such a good proxy for depth?** Are there theoretical guarantees?
- **Why does the worse initialization converge faster?** Is the loss landscape more favorable from the $$\|W_{\text{out}}\|_F$$ starting point?

---

## Links

- Paper: [arXiv:2602.19845](https://arxiv.org/abs/2602.19845)
- Code: [github.com/hynwprk/droppedaneuralnet](https://github.com/hynwprk/droppedaneuralnet)
