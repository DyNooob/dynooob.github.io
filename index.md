---
layout: default
---

<div class="section-title">最新文章</div>

<ul class="post-list">
{% for post in paginator.posts %}
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

{% if paginator.total_pages > 1 %}
<div class="pagination">
  {% if paginator.previous_page %}
    <a href="{{ paginator.previous_page_path | relative_url }}">← 上一页</a>
  {% else %}
    <span style="visibility:hidden">← 上一页</span>
  {% endif %}

  {% for page in (1..paginator.total_pages) %}
    {% if page == paginator.page %}
      <span class="current">{{ page }}</span>
    {% else %}
      <a href="{{ site.paginate_path | relative_url | replace: ':num', page }}">{{ page }}</a>
    {% endif %}
  {% endfor %}

  {% if paginator.next_page %}
    <a href="{{ paginator.next_page_path | relative_url }}">下一页 →</a>
  {% else %}
    <span style="visibility:hidden">下一页 →</span>
  {% endif %}
</div>
{% endif %}