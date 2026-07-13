---
layout: default
---

<div class="section-title">最新文章</div>

<ul class="post-list">
{% for post in site.posts limit:10 %}
  <li>
    <a href="{{ post.url | relative_url }}" class="post-title">{{ post.title }}</a>
    <div class="post-meta">
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
      {% if post.tags %}
      <span>
        {% for tag in post.tags limit:3 %}
        <span class="tag">{{ tag }}</span>
        {% endfor %}
      </span>
      {% endif %}
    </div>
  </li>
{% endfor %}
</ul>

{% if site.posts.size > 10 %}
<div class="pagination">
  <span style="color:#bbb;font-size:0.85rem">共 {{ site.posts.size }} 篇文章 · 显示最近 10 篇</span>
</div>
{% endif %}