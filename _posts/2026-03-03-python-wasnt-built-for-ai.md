---
layout: post
title: "Python Wasn't Built for AI"
date: 2026-03-03
description: "How a chain of accidents, starting with one developer in 2005, made the slowest language the default for the entire AI industry."
---

Python wasn't built for AI.

It's slow. Like, `mass_multiply_two_matrices_and_go_make_coffee()` slow.

Yet it powers almost every major AI system on Earth. ChatGPT, Stable Diffusion, AlphaFold, self-driving cars.

How? A chain of accidents nobody planned.

In the early 2000s, serious ML ran on MATLAB, R, and C++. Python was "that scripting language." Nobody took it seriously for computation.

Then one man changed everything.

In 2005, Travis Oliphant got tired of watching Python's scientific community tear itself apart over two competing math libraries, Numeric and numarray. So he merged them himself. Single-handedly rewrote the code. Called it NumPy (after "numerix" turned out to be trademarked).

That one act of open-source diplomacy gave Python a unified foundation for numerical computing.

Everything snowballed from there:

- **SciPy** (2001, but gained real traction after NumPy unified things)
- **Pandas** (2008, built by Wes McKinney at a hedge fund because he was frustrated with Excel)
- **scikit-learn** (2010, started as a Google Summer of Code project in 2007)

Each library built on top of the last. Not by grand design. By accident.

But here's the part most people overlook:

Python is the "glue language." You write clean, simple Python on top. Underneath, it secretly calls blazing-fast C, C++, and Fortran code.

You get the ease of Python with the speed of low-level languages. Best of both worlds. Duct-taped together beautifully.

Then the killing blow landed.

Google released TensorFlow in 2015. Python API. Facebook released PyTorch in 2016. Python API.

Two tech giants. Same choice. Debate over.

When Google and Facebook agree on something, the entire industry follows.

The irony? Julia, a language built from scratch in 2012 specifically to solve Python's speed problem, is technically better for ML in many ways. Fast like C. Easy like Python. No glue needed.

But it arrived too late. The ecosystem had already locked in.

Python didn't win because it was the best tool. It won because one developer unified a fragmented library in 2005, and the snowball never stopped.

Just like QWERTY. Not the best layout. Still on every keyboard.
