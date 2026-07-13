---
layout: default
title: 搜索
---

<div class="section-title">搜索文章</div>

<input type="text" class="search-box" id="searchInput" placeholder="输入关键词搜索..." autofocus>
<div class="search-hint" id="searchHint">输入关键词开始搜索，支持标题和标签匹配</div>
<ul class="search-results" id="searchResults"></ul>

<script>
var posts = [];

fetch('{{ "/assets/search.json" | relative_url }}')
  .then(function(r) { return r.json(); })
  .then(function(data) {
    posts = data;
    document.getElementById('searchInput').disabled = false;
    document.getElementById('searchHint').textContent = '共 ' + posts.length + ' 篇文章，输入关键词搜索';
  })
  .catch(function() {
    document.getElementById('searchHint').textContent = '搜索索引加载失败';
  });

document.getElementById('searchInput').addEventListener('input', function() {
  var q = this.value.trim().toLowerCase();
  var results = document.getElementById('searchResults');
  var hint = document.getElementById('searchHint');

  if (q.length < 1) {
    results.innerHTML = '';
    hint.textContent = '共 ' + posts.length + ' 篇文章，输入关键词搜索';
    return;
  }

  var matched = posts.filter(function(p) {
    return p.title.toLowerCase().indexOf(q) !== -1 ||
           p.tags.some(function(t) { return t.toLowerCase().indexOf(q) !== -1; }) ||
           p.categories.some(function(c) { return c.toLowerCase().indexOf(q) !== -1; });
  });

  if (matched.length === 0) {
    results.innerHTML = '<li style="color:#aaa;padding:20px 0;list-style:none">未找到匹配的文章</li>';
    hint.textContent = '已搜索 ' + posts.length + ' 篇文章';
    return;
  }

  hint.textContent = '找到 ' + matched.length + ' 篇匹配文章';
  results.innerHTML = matched.map(function(p) {
    return '<li><a href="' + p.url + '">' + p.title + '</a>' +
           '<div class="meta">' + p.date + ' · ' + p.tags.slice(0,3).join(' / ') + '</div></li>';
  }).join('');
});
</script>