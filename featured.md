---
layout: default
title: 精选文章
---

<div class="section-title">精选文章</div>

<ul class="post-list">
{% assign featured_posts = site.posts | where_exp: "post", "post.tags contains '精选' or post.tags contains 'AI' or post.tags contains '安全' or post.categories contains '安全'" %}

{% for post in site.posts %}
  {% assign is_featured = false %}
  {% if post.tags contains "精选" %}
    {% assign is_featured = true %}
  {% endif %}
  {% if post.tags contains "实战" or post.tags contains "指南" or post.tags contains "框架" %}
    {% assign is_featured = true %}
  {% endif %}
  {% if post.categories contains "安全" or post.categories contains "AI" %}
    {% assign is_featured = true %}
  {% endif %}
  {% if post.title contains "指南" or post.title contains "实战" or post.title contains "框架" %}
    {% assign is_featured = true %}
  {% endif %}
  {% if is_featured %}
  <li>
    <a href="{{ post.url | relative_url }}" class="post-title">{{ post.title }}</a>
    <div class="post-meta">
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
      <span>{{ post.categories | join: " / " }}</span>
      {% if post.tags %}
      <span>
        {% for tag in post.tags limit:3 %}
        <span class="tag">{{ tag }}</span>
        {% endfor %}
      </span>
      {% endif %}
    </div>
  </li>
  {% endif %}
{% endfor %}
</ul>