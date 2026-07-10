---
layout: default
title: Archive
permalink: /archive/
---
<h1 class="page-heading">Archive</h1>

{% assign prev_year = "" %}
{% for post in site.posts %}
  {% assign year = post.date | date: "%Y" %}
  {% if year != prev_year %}
<h2 class="archive-year">{{ year }}</h2>
    {% assign prev_year = year %}
  {% endif %}
<p>
  <span class="post-meta">{{ post.date | date: "%m-%d" }}</span>
  &nbsp;<a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
</p>
{% endfor %}
