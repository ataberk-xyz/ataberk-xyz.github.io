---
layout: page
title: Categories
permalink: /categories
sidebar_link: true
---

{% for category in site.categories %}
  <h3>{{ category[0] }}</h3>
  <ul>
    {% for post in category[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a> | {{ post.date | date: "%B %d, %Y"}}</li>
    {% endfor %}
  </ul>
{% endfor %}
