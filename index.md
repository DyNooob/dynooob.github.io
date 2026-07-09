---
layout: default
---

# NOOOB Tech Blog

网络安全 · 技术开发 · 日常笔记

---

### 最新文章

{% for post in site.posts limit:10 %}
- [{{ post.title }}]({{ post.url }}) · {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

---

### 标签

{% for tag in site.tags %}
`{{ tag[0] }}`
{% endfor %}