---
layout: page
title: Categories
permalink: /categories
sidebar_link: true
---

{% for category in site.categories %}
  <h3 id="{{ category[0] | slugify }}">{{ category[0] }}</h3>
  <div class="ledger-grid">
    {% for post in category[1] %}
      {% capture ponum %}{% include post-number.html post=post %}{% endcapture %}
      <article class="ledger-card">
        <a class="lc-cover" href="{{ site.baseurl }}{{ post.url }}" aria-hidden="true" tabindex="-1">{% include cover.html post=post cols=24 rows=14 %}</a>
        <div class="lc-body">
          <p class="no-line">{{ ponum }} · {{ post.date | date: "%Y-%m-%d" }}</p>
          <h3 class="lc-title"><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h3>
        </div>
      </article>
    {% endfor %}
  </div>
{% endfor %}
