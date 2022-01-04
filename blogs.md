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
      <small>{{ post.excerpt }}</small>
     <small>{{ post.date | date: "%-d %B %Y" }}</small>
    </li>
  {% endfor %}
</ul>

[back](./)