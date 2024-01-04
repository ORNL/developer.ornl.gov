---
layout: page
title: Blog Posts
category: pages
navigation_weight: 2
breadcrumb: Blog
---

<ul class="blog-list">
  {% for post in site.posts %}
    <li>
      <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
      <span>{{ post.date | date_to_string }}</span>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>