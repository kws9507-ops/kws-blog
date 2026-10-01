---
layout: single
author_profile: false
---

<h2 style="margin-top: 0; margin-bottom: 1.5rem;">Recent Posts</h2>

{% if site.posts.size > 0 %}
  {% for post in site.posts limit:10 %}
    <article class="list__item">
      <h3 class="archive__item-title" style="margin-bottom: 0.2rem;">
        <a href="{{ post.url | relative_url }}" rel="permalink">{{ post.title }}</a>
      </h3>
      <p class="page__meta" style="font-size: 0.8rem; color: #888; margin-bottom: 0.5rem;">
        <i class="far fa-calendar-alt"></i> {{ post.date | date: "%Y-%m-%d" }}
        {% if post.categories.size > 0 %}
          | <i class="far fa-folder"></i> {{ post.categories | join: ", " }}
        {% endif %}
      </p>
      {% if post.excerpt %}
        <p class="archive__item-excerpt" style="font-size: 0.9rem; color: #555;">{{ post.excerpt | strip_html | truncate: 160 }}</p>
      {% endif %}
    </article>
    <hr style="margin: 1.5rem 0;">
  {% endfor %}
{% else %}
  <p style="text-align: center; color: #777; padding: 3rem 0;">📌 최근 작성된 게시글이 없습니다.</p>
{% endif %}
