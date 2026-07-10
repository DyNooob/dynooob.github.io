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
      <span class="tags">
        {% for tag in post.tags limit:5 %}
        <span class="tag">{{ tag }}</span>
        {% endfor %}
      </span>
      {% endif %}
    </div>
  </li>
{% endfor %}
</ul>

<div class="section-title">标签</div>
<div class="tag-list">
{% for tag in site.tags %}
  <a href="/?tag={{ tag[0] | url_encode }}">{{ tag[0] }} ({{ tag[1] | size }})</a>
{% endfor %}
</div>