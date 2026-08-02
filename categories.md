---
layout: page
title: Categories
permalink: /categories
sidebar_link: true
kicker: "INDEX · ALL POSTS BY CATEGORY"
---

<nav class="cat-index" aria-label="Filter posts by category">
  <div class="cat-index-cols">
    <button class="cat-row is-active" type="button" data-cat="all" aria-pressed="true"><span class="cat-name">all</span><span class="cat-fill"></span><span class="cat-count">{{ site.posts | size }}</span></button>
    {% for category in site.categories %}
    <button class="cat-row" type="button" data-cat="{{ category[0] | slugify }}" aria-pressed="false"><span class="cat-name">{{ category[0] }}</span><span class="cat-fill"></span><span class="cat-count">{{ category[1] | size }}</span></button>
    {% endfor %}
  </div>
</nav>

{% for category in site.categories %}
<section class="cat-section" data-cat="{{ category[0] | slugify }}">
  <h3 id="{{ category[0] | slugify }}">{{ category[0] }}</h3>
  <div class="ledger-grid">
    {% for post in category[1] %}
      {% capture ponum %}{% include post-number.html post=post %}{% endcapture %}
      <article class="ledger-card">
        <a class="lc-cover" href="{{ site.baseurl }}{{ post.url }}" aria-hidden="true" tabindex="-1">{% include cover.html post=post cols=24 rows=14 %}</a>
        <div class="lc-body">
          <p class="no-line">{{ ponum }} · {{ post.date | date: "%Y-%m-%d" }}</p>
          <h3 class="lc-title"><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h3>
          {% assign lc_dek = post.summary | default: post.description %}
          {% if lc_dek %}<p class="lc-dek">{{ lc_dek }}</p>{% endif %}
        </div>
      </article>
    {% endfor %}
  </div>
</section>
{% endfor %}

<script>
  (function () {
    var pills = document.querySelectorAll('.cat-row');
    var sections = document.querySelectorAll('.cat-section');

    function apply(cat, pushHash) {
      sections.forEach(function (s) {
        s.hidden = !(cat === 'all' || s.dataset.cat === cat);
      });
      pills.forEach(function (p) {
        var on = p.dataset.cat === cat;
        p.classList.toggle('is-active', on);
        p.setAttribute('aria-pressed', on ? 'true' : 'false');
      });
      if (pushHash) {
        history.replaceState(null, '', cat === 'all' ? location.pathname : '#' + cat);
      }
    }

    pills.forEach(function (p) {
      p.addEventListener('click', function () { apply(p.dataset.cat, true); });
    });

    // deep link: /categories#writeup opens filtered
    var initial = location.hash.replace('#', '');
    if (initial && document.querySelector('.cat-section[data-cat="' + initial + '"]')) {
      apply(initial, false);
    }
  })();
</script>
