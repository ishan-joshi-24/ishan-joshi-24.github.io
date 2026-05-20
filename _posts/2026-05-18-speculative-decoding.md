---
layout: post
title: "Speculative Decoding: Make Your LLM 3x Faster by Letting It Guess"
date: 2026-05-18
description: "Your GPU is mostly idle while your LLM generates tokens. A clever protocol turns that idle time into 3x faster inference."
---

Your GPU is mostly idle while your LLM generates tokens.

A clever protocol turns that idle time into 3x faster inference.

It's called **speculative decoding**.

Here's the setup. LLMs generate one token at a time. For each token, the GPU has to load all of the model's weights from memory before it can do the math. Loading the weights is slow. The math itself is fast.

So most of the GPU's time during generation is spent loading weights, not doing math.

Here's the part that matters. The GPU can actually do math for many tokens in parallel. It just needs to know what those tokens are. Normally we don't, because each new token depends on the previous one.

The trick:

1. Use a small, fast "draft" model to guess the next 5 tokens.
2. Run the big model once, giving it those 5 candidate tokens as input. It now produces a prediction at each of the 5 positions in parallel.
3. At each position, check: would the big model have picked the same token the draft guessed? Accept the matches. Discard the rest.

Net effect: one round of loading weights, multiple tokens out instead of one.

**Theoretically:** 2 to 3x speedup is typical. Predictable text like code sees more.

**In practice:** the gain depends on how often the small model guesses right. Creative or unusual prompts see less benefit. And if the small model is completely wrong, you still get one correct token from the big model anyway. You never go backwards.

The wild part is the math. Speculative decoding is designed so the output is **mathematically identical** to running the big model alone. Same outputs. Just faster.

OpenAI, Anthropic, and Google all use it in production. It's been a real driver of falling inference costs over the last two years.

An optimization that looks like cheating but isn't. Best kind there is.
