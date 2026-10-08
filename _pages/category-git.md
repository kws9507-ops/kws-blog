---
title: "Git"
layout: archive
permalink: /categories/git/
author_profile: false
---

{% assign posts = site.categories.Git %}
{% if posts.size > 0 %}
<div class="entries-list">
{% for post in posts %}
<div class="list__item">
  <article class="archive__item" itemscope itemtype="https://schema.org/CreativeWork">
    <h2 class="archive__item-title no_toc" itemprop="headline" style="margin-top: 0.5rem; margin-bottom: 0.3rem;">
      <a href="{{ post.url | relative_url }}" rel="permalink">{{ post.title }}</a>
    </h2>
    <p class="page__meta" style="font-size: 0.85rem; color: #666; margin-bottom: 0.5rem;">
      <i class="far fa-calendar-alt"></i> {{ post.date | date: "%Y-%m-%d" }}
    </p>
    {% if post.excerpt %}
      <p class="archive__item-excerpt" itemprop="description" style="font-size: 0.9rem; color: #444; margin-bottom: 0;">
        {{ post.excerpt | strip_html | truncate: 160 }}
      </p>
    {% endif %}
  </article>
</div>
<hr style="margin: 1.2rem 0; border: 0; border-top: 1px solid #eee;">
{% endfor %}
</div>
{% else %}
<p style="text-align: center; color: #777; padding: 3rem 0;">📌 등록된 게시글이 없습니다.</p>
{% endif %}
