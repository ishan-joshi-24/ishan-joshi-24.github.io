---
layout: post
title: "Opus 4.7 Didn't Get Dumber, Your Prompts Lost Their Safety Net"
date: 2026-05-04
description: "Since 4.7 dropped, my feed has been full of complaints. The migration docs spell out exactly what changed, and it isn't the model."
---

Anyone else still on Opus 4.6?

Since 4.7 dropped, my feed has been full of complaints. *"It feels worse." "It lost its personality." "I'm switching back."*

And honestly I get it. 4.6 felt like talking to a thoughtful colleague. 4.7 feels like babysitting a very talented intern.

But here's the thing.

Anthropic's own migration docs spell it out: 4.7 *"interprets prompts more literally and explicitly. It will not silently generalize an instruction from one item to another, and it will not infer requests you didn't make."*

Translation: 4.6 was guessing what you meant. And it was usually right. 4.7 stopped guessing.

The model scored 87.6% on SWE-bench. Best Claude has ever done. It didn't get dumber. **Your prompts just lost their safety net.**

So yeah. Are you updating your prompts or just... going back?
