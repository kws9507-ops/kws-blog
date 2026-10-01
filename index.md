---
layout: home
author_profile: true
---

# 김우석의 기술 블로그

Git · Linux · Oracle · SharePlex · Snowflake

## 📝 최근 글

{% for post in site.posts limit:5 %}

### [{{ post.title }}]({{ site.baseurl }}{{ post.url }})

`{{ post.date | date: "%Y-%m-%d" }}`

{% endfor %}
