---
layout: default
title: Writing
permalink: /posts/
description: Deep-dives on backend systems, AI infrastructure, and the trade-offs behind them.
---

<section class="hero">
  <h1>Writing</h1>
  <p class="tagline">Deep-dives on the parts of a system that explain why it works the way it does, databases, distributed systems, LLM infrastructure, and the trade-offs underneath.</p>
</section>

<ul class="posts-index">
  {% for post in site.posts %}
    <li>
      <div class="post-row">
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
        <span class="when">{{ post.date | date: "%b %Y" }}</span>
      </div>
      {% if post.excerpt %}
        <p class="excerpt">{{ post.excerpt | strip_html | truncatewords: 32 }}</p>
      {% endif %}
    </li>
  {% endfor %}
</ul>
