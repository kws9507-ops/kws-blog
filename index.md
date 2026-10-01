--- 
layout: default 
title: My Tech Blog 
--- 
 
# My Tech Blog 
 
Oracle, SharePlex, Linux, Snowflake 
 
## Recent Posts 
 
{% for post in site.posts %} 
- [{{ post.title }}]({{ post.url }}) 
{% endfor %} 
