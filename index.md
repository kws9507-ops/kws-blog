\---

layout: default

title: "김우석의 기술 블로그"

\---



\# 김우석의 기술 블로그



Oracle, SharePlex, Linux, Snowflake 그리고 개발 공부를 기록합니다.



\## 최근 글



{% for post in site.posts %}

\- \[{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%Y-%m-%d" }}

{% endfor %}

