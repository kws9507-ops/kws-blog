---
title: "SharePlex"
layout: single
permalink: /categories/shareplex/
author_profile: false
---

{% assign category_posts = site.categories.SharePlex %}

{% if category_posts.size > 0 %}
  {% for post in category_posts %}
    <article class="list__item">
      <h3 class="archive__item-title" style="margin-bottom: 0.2rem;">
        <a href="{{ post.url | relative_url }}" rel="permalink">{{ post.title }}</a>
      </h3>
      <p class="page__meta" style="font-size: 0.8rem; color: #888; margin-bottom: 0.5rem;">
        <i class="far fa-calendar-alt"></i> {{ post.date | date: "%Y-%m-%d" }}
      </p>
      {% if post.excerpt %}
        <p class="archive__item-excerpt" style="font-size: 0.9rem; color: #555;">{{ post.excerpt | strip_html | truncate: 160 }}</p>
      {% endif %}
    </article>
    <hr style="margin: 1.5rem 0;">
  {% endfor %}
{% else %}
  <p style="text-align: center; color: #777; padding: 3rem 0;">📌 등록된 게시글이 없습니다.</p>
{% endif %}
