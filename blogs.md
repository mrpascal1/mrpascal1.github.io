---
layout: default
category: Blogs
title: Blogs
nav_order: 2
---

# Latest

<ul>
  {% for post in site.posts %}
    <li>
      <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
      {{ post.excerpt }}
    <p>{{ post.date | date: "%-d %B %Y" }}</p>
    </li>
  {% endfor %}
</ul>

[back](./)