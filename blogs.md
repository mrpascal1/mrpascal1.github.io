---
layout: default
category: Blogs
---

# Blogs

<ul>
  {% for post in site.posts %}
    <li class="blog">
      <h3><u><a href="{{ post.url }}">{{ post.title }}</a></u></h3>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>

[back](./)