---
layout: page
title: Categories
permalink: /categories
sidebar_link: true
---

{% for category in site.categories %}
  <h3 id="{{ category[0] | slugify }}">{{ category[0] }}</h3>
  <ul class="posts-list">
    {% for post in category[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a> <small>&middot; {{ post.date | date: "%B %-d, %Y"}}</small></li>
    {% endfor %}
  </ul>
{% endfor %}
