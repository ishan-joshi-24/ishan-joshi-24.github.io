---
layout: page
title: About
permalink: /about/
description: Applied AI Engineer. LLM and retrieval systems in production.
---

I'm an **Applied AI Engineer**. I ship LLM and retrieval systems that hold up in production. Semantic search at hundreds-of-thousands scale, agent backends that act on real platforms, document parsers that know when to say "I don't know."

I own these features end-to-end. Schema design, embedding pipelines, prompt structure, the audit prompts that gate model output, the caches and crons underneath, and the production observability that tells me when something is drifting. The model call is one line of code. Building around it so it survives users is the actual job.

---

<div class="about-row">
  <span class="label">Title</span>
  <span class="value">Applied AI Engineer</span>
</div>
<div class="about-row">
  <span class="label">Open to</span>
  <span class="value">Senior AI roles · Remote or hybrid</span>
</div>
<div class="about-row">
  <span class="label">AI / LLM</span>
  <span class="value">OpenAI · LangChain · pgvector · HNSW · embeddings · prompt design</span>
</div>
<div class="about-row">
  <span class="label">Backend</span>
  <span class="value">TypeScript · Node.js · Python · Java · TypeORM · BullMQ · Kafka</span>
</div>
<div class="about-row">
  <span class="label">Data</span>
  <span class="value">PostgreSQL · MariaDB / MySQL · Redis · BigQuery</span>
</div>
<div class="about-row">
  <span class="label">Infra</span>
  <span class="value">AWS · GCP · Docker · Kubernetes · Helm · GitLab CI</span>
</div>
<div class="about-row">
  <span class="label">Observability</span>
  <span class="value">Sentry · Datadog · Winston</span>
</div>

---

## How I work

I like the parts of an AI feature that aren't the model call. The eligibility filter that has two years of edge cases baked in. The audit log that turns "we trust the agent" into "we trust the agent because here's exactly what it did last Tuesday." The cache TTL that's the difference between a cron poisoning Redis and a cron being free.

The most underrated skill in shipping AI is **knowing what to not let the LLM do.** The model is good at composition. It is bad at confidence calibration, at staying inside its lane, at refusing to invent. Designing around that (schemas, audit prompts, propose-don't-commit, return-null-when-unsure) is most of the job.

## What I'm into right now

- Embedding pipelines that cost cents instead of dollars
- The generator + judge pattern. Two LLM calls beat one, almost always
- Agent backends with deliberately tiny vocabularies
- pgvector at the point where HNSW vs IVFFlat actually matters
- The boundary between deterministic backend code and best-effort LLM calls

## Get in touch

- **Email** &middot; [joshishan21@gmail.com](mailto:joshishan21@gmail.com)
- **GitHub** &middot; [ishan-joshi-24](https://github.com/ishan-joshi-24)
- **LinkedIn** &middot; [ishan-joshi180](https://www.linkedin.com/in/ishan-joshi180)
- **CV** &middot; [download](https://drive.google.com/file/d/1ipVHNXCfd8J5EC3rWVcqMt68juANxKbY/view?usp=drive_link)
