---
layout: post
title: "I Dropped a Neural Net"
date: 2026-02-23 13:49:00 -0500
description: Recovering the exact layer ordering of a shuffled 96-layer ResNet.
tags: [machine-learning, neural-networks]
categories: [research]
related_posts: false
thumbnail: assets/img/publication_preview/dropped-neural-net.png
---

A recent [Dwarkesh Patel podcast](https://www.dwarkeshpatel.com) with John Collison and Elon Musk featured an interesting puzzle from Jane Street: they trained a neural net, shuffled all 96 layers, and asked to put them back in order.

I thought about it and wrote a paper.

## The puzzle

Given the unlabelled layers of a Residual Network and its training dataset, recover the exact ordering. The problem decomposes into two sub-problems:

1. **Pairing.** Each residual block has an input projection and an output projection. Match them: $$48!$$ possibilities.
2. **Ordering.** Arrange the reassembled blocks in sequence: another $$48!$$ possibilities.

The combined search space is $$(48!)^2 \approx 10^{122}$$---more configurations than atoms in the observable universe.

## Key insight: dynamic isometry

Stability conditions during training (like dynamic isometry) leave a signature in the weights. Specifically, the product $$W_{\text{out}} W_{\text{in}}$$ for correctly paired layers exhibits a negative diagonal structure. This gives us a clean signal: the **diagonal dominance ratio** reliably identifies correct pairings.

## Ordering via hill-climbing

For ordering, we seed-initialize with a rough proxy (delta-norm or $$\|W_{\text{out}}\|_F$$) and then hill-climb to zero mean squared error on the training set. The approach converges to the exact original ordering.

## Links

- Paper: [arXiv:2602.19845](https://arxiv.org/abs/2602.19845)
- Code: [github.com/hynwprk/droppedaneuralnet](https://github.com/hynwprk/droppedaneuralnet)
