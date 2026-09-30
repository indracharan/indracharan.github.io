---
layout: page
title: Blogs
permalink: /blogs/
order: 1
---

# IT Experience & Technical Blogs

Welcome to my blog section. Here I share insights from my 16 years of experience in Software Engineering and my opinion about latest technologies.

<ul class="post-list">
  {% for post in site.posts %}
    <li>
      <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
      <h3>
        <a class="post-link" href="{{ post.url | relative_url }}">
          {{ post.title | escape }}
        </a>
      </h3>
      {% if post.excerpt %}
        {{ post.excerpt }}
      {% endif %}
    </li>
  {% endfor %}
</ul>

---
*Stay tuned for more updates!*
