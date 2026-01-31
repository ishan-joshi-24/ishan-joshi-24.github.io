---
layout: post
title: "Building LinkedIn Focus: A Local AI Feed Filter in an Afternoon"
date: 2026-01-31
---

LinkedIn's algorithm doesn't know what I want. So I built my own. It took an afternoon.

## The Idea

I wanted a Chrome extension that filters my LinkedIn feed using AI. Technical content stays, engagement bait goes. Simple enough.

But I didn't want to send my feed data to external APIs. No OpenAI, no Claude API, no cloud services. Everything should run locally on my machine.

## How Easy Was It?

I asked Claude to help me build it. Within minutes, I had:
- A working Chrome extension structure
- Content script to detect LinkedIn posts
- Popup UI with toggle switches
- Integration with Ollama for local inference

A couple hours of refining—fixing selectors, tuning classification prompts, polishing the UI—and it was done.

That's it. An afternoon.

The conversation was straightforward:
- "Build me a Chrome extension that filters my LinkedIn feed locally"
- "The selectors aren't finding posts, here's the HTML"
- "Job posts are getting incorrectly filtered"
- "There's UI jitter when posts hide"

Each problem got solved in minutes. Not because I knew all the answers, but because I could describe the problem and get working solutions.

## Local Models Are Good Enough

Here's what surprised me: a 3B parameter model running on my laptop classifies posts in under a second. Accurately.

I'm using Ollama with qwen2.5:3b. It's small enough to run fast, capable enough to understand the nuance between a genuine technical post and engagement bait.

No API costs. No rate limits. No privacy concerns. The model runs entirely on my machine.

For text classification tasks like this, you don't need GPT-4. You don't need cloud inference. A local model on consumer hardware is perfectly adequate.

## What This Means

The barrier to building useful tools has collapsed.

You have a problem? You can probably solve it in an afternoon. The combination of:
- AI assistants that can write and debug code
- Local models that can run inference on a laptop
- Existing tooling (Chrome extensions, Ollama, etc.)

...means you can go from "I wish this existed" to "I built it" in hours, not weeks.

Anyone with basic programming knowledge and some LLM tooling knowledge can build something like this now. And I'm not even sure about the first—that might not be the deciding factor anymore.

The barrier is finally low enough to just solve your own problems—exactly the way you want them solved.

## Try It

The extension is open source: [github.com/ishan-joshi-24/linkedin-focus](https://github.com/ishan-joshi-24/linkedin-focus)

To run it:
1. Install Ollama and pull a model: `ollama pull qwen2.5:3b`
2. Start Ollama: `OLLAMA_ORIGINS="chrome-extension://*" ollama serve`
3. Load the extension in Chrome (developer mode)
4. Open LinkedIn

Customize what you want to see in the popup. Your feed, your rules.
