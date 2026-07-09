---
layout: home
title: ""
---

# 🚀 NOOOB Tech Blog

> 网络安全 · 技术开发 · 警校生的日常

---

### 📝 最新文章

{% for post in site.posts limit:5 %}
- [{{ post.title }}]({{ post.url }}) · {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

---

### 📂 分类

{% for category in site.categories %}
- **{{ category[0] }}** ({{ category[1].size }} 篇)
{% endfor %}

---

### 🏷️ 标签

{% for tag in site.tags %}
`{{ tag[0] }}` ({{ tag[1].size }})
{% endfor %}