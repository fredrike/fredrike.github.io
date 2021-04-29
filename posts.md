---
layout: post
title: Posts
collection: posts
---

This is my latest posts.
<ul>
  {% for post in site.posts %}
    <li>
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      <small>{{ post.date | date: "%Y-%m-%d" }}</small>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>