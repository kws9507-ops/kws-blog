---
layout: single
author_profile: false
---

<!-- 1. Profile & Experience 섹션 -->
<div style="background-color: #f8f9fa; border-radius: 10px; padding: 1.5rem; margin-bottom: 2rem; border: 1px solid #e9ecef;">
<h2 style="margin-top: 0; margin-bottom: 1rem; border-bottom: 2px solid #2b2b2b; padding-bottom: 0.5rem; color: #2b2b2b;">
👨‍💻 Profile & Experience
</h2>

<p style="font-size: 1.05rem; color: #212529; margin-bottom: 1.2rem;">
<strong>김우석</strong> | Data Infrastructure & Database Engineer
</p>

<div style="margin-bottom: 0;">
<h3 style="font-size: 0.95rem; margin-top: 0; margin-bottom: 0.5rem; color: #495057;">🎓 Education & Work Experience</h3>
<ul style="font-size: 0.9rem; color: #495057; padding-left: 1.2rem; margin-bottom: 0; line-height: 1.7;">
<li><strong>바이텍정보통신</strong> (2025.11 ~ 재직 중)</li>
<li><strong>TMAX Cloud</strong> (2023.06 ~ 2025.01)</li>
<li><strong>강원대학교</strong> 졸업 (2021)</li>
</ul>
</div>
</div>

<!-- 2. 최신 게시글 섹션 -->
<h2 style="margin-top: 0; margin-bottom: 1.5rem;">📑 Recent Posts</h2>

{% if site.posts.size > 0 %}
<div class="entries-list">
{% for post in site.posts limit:10 %}
<div class="list__item">
  <article class="archive__item" itemscope itemtype="https://schema.org/CreativeWork">
    <h2 class="archive__item-title no_toc" itemprop="headline" style="margin-top: 0.5rem; margin-bottom: 0.3rem;">
      <a href="{{ post.url | relative_url }}" rel="permalink">{{ post.title }}</a>
    </h2>
    <p class="page__meta" style="font-size: 0.85rem; color: #666; margin-bottom: 0.5rem;">
      <i class="far fa-calendar-alt"></i> {{ post.date | date: "%Y-%m-%d" }}
      {% if post.categories.size > 0 %}
        | <i class="far fa-folder"></i> 
        {% for cat in post.categories %}
          {% if cat == "Oracle" %}
            <span style="color: red; font-weight: bold;">{{ cat }}</span>
          {% elsif cat == "SharePlex" %}
            <span style="color: orange; font-weight: bold;">{{ cat }}</span>
          {% elsif cat == "Snowflake" %}
            <span style="color: #00a8e8; font-weight: bold;">{{ cat }}</span>
          {% else %}
            <span style="font-weight: bold;">{{ cat }}</span>
          {% endif %}
          {% unless forloop.last %}, {% endunless %}
        {% endfor %}
      {% endif %}
    </p>
  </article>
</div>
<hr style="margin: 1.2rem 0; border: 0; border-top: 1px solid #eee;">
{% endfor %}
</div>
{% else %}
<p style="text-align: center; color: #777; padding: 3rem 0;">📌 최근 작성된 게시글이 없습니다.</p>
{% endif %}
