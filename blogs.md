---
layout: default
category: Blogs
title: Blogs
nav_order: 2
---

<!-- ## Blogs - Coming Soon -->

<h1>Latest Posts</h1>

<ul>
  {% for post in site.posts %}
    <li>
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>

[back](./)