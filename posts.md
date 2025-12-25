---
layout: default
title: Posts
permalink: /posts/
---

<div class="posts-page">
  <h1>Posts</h1>

  {% if site.posts.size > 0 %}
    <div class="post-list-container">
      {% for post in site.posts %}
        <article class="post-item">
          <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
          <time class="post-date">{{ post.date | date: "%B %d, %Y" }}</time>
          {% if post.excerpt %}
            <p class="post-excerpt">{{ post.excerpt }}</p>
          {% endif %}
          <a href="{{ post.url | relative_url }}" class="read-more">Read more →</a>
        </article>
      {% endfor %}
    </div>
  {% else %}
    <p>No posts yet.</p>
  {% endif %}
</div>
