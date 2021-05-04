---
layout: post
title: Posts
collection: posts
---

These are my latest posts.
<ul>
  {% for post in site.posts %}
    <li>
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      <small>{{ post.date | date: "%Y-%m-%d" }}</small>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>