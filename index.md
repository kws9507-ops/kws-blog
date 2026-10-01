---
layout: single
title: "김우석의 기술 블로그"
---

# 김우석의 기술 블로그

Git · Linux · Oracle · SharePlex · Snowflake

> Database Engineer를 목표로 기술을 공부하고,
> 직접 실습한 내용을 기록하는 기술 블로그입니다.

---

## 📚 기술 분야

### Git

Git과 GitHub를 이용한 버전 관리와 협업

### Linux

Linux 명령어와 서버 관리

### Oracle

Oracle Database, SQL, DBA 및 장애 대응

### SharePlex

Oracle 데이터베이스 복제, CDC 및 운영

### Snowflake

Data Warehouse, RBAC, Iceberg 및 데이터 플랫폼

---

## 📝 최근 글

{% for post in site.posts limit:5 %}

### [{{ post.title }}]({{ site.baseurl }}{{ post.url }})

`{{ post.date | date: "%Y-%m-%d" }}`

{% endfor %}
