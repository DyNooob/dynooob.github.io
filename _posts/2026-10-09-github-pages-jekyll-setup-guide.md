---
layout: post
title: "从零搭建一个能持续更新的技术博客：GitHub Pages + Jekyll 全流程"
date: 2026-10-09 13:00:00 +0800
categories: [博客搭建, 效率工具]
tags: [github-pages, jekyll, cron, blog, automation, static-site, markdown, obsidian, knowledge-base]
---

## 写博客不是写一篇，而是写一百篇

很多人搭好博客后写了 3 篇就停了。**真正决定博客生死的，不是模板好不好看，而是更新能不能持续**。

这篇文章讲我从零搭起一个技术博客的全流程，重点不是 Jekyll 怎么配置（网上教程一大堆），而是**怎么让博客持续有内容更新、怎么让工具自动化、怎么处理国内网络环境**。每一步都给出具体的方案和踩过的坑。

---

## 一、先选技术栈：GitHub Pages + Jekyll

### 1.1 为什么不用 Hugo / Hexo / Next.js？

我尝试过 Hexo、Vue 自建、Gatsby，最后回到 Jekyll。理由：

- **零本地依赖**：GitHub Pages 原生支持 Jekyll，**本地不需要 Node.js、Ruby、bundle install**。直接 `git push` 就部署。
- **远程主题 + 自定义布局**：用 `remote_theme: pages-themes/hacker@v0.2.0` + 自己的 `_layouts/` 覆盖默认，**既省事又有控制权**。
- **Markdown 原生**：`_posts/` 下放 `.md` 文件，自动渲染为 HTML。

### 1.2 Jekyll 的两个核心概念

```
tech-blog/
├── _config.yml        # 站点配置（标题、主题、插件）
├── index.md           # 首页（layout: default）
├── about.md           # 关于页
├── _posts/            # 文章目录
│   └── YYYY-MM-DD-slug.md
├── _layouts/          # 自定义布局（覆盖远程主题）
├── _data/             # 数据文件（用于自动更新标记）
├── assets/css/style.css
└── search.md          # 搜索页（客户端实时搜索）
```

**`_config.yml` 决定全局**，**`_layouts/` 决定 HTML 结构**，**`_posts/` 决定内容**。三者分离，互不干扰。

---

## 二、博客设计的「极简主义」

### 2.1 我最终的设计方向

经历了 4 个迭代：太复杂 → 极致简洁 → 简约但不简单 → **就黑白**。

```
✅ 纯白底 #fff + 纯黑文字 #1a1a1a
✅ 系统字体（不加 Google Fonts）
✅ 窄容器 max-width: 680px
✅ 分割线统一灰色 #e0e0e0 / #eee / #f0f0f0
✅ 无卡片圆角（border-radius 不超过 4px）
✅ 无阴影、无渐变、无暖色调
❌ 不加 Google Fonts
❌ 不加任何 emoji 装饰
❌ 不做 tag 云
```

**为什么是黑白**：黑白是**最安全、最耐看、最少争议**的选择。彩色主题一开始觉得好看，三个月后觉得刺眼。黑白永远不会过时。

### 2.2 不要被「漂亮」绑架

很多博客主题有：
- 左侧固定导航
- 右侧 TOC
- 底部社交按钮
- 代码块彩虹配色
- 标签云

这些**看起来专业**，但**每次写文章时你都得分心考虑排版**。极简主义的好处是：**写完文字，剩下的 CSS 自动搞定**。

### 2.3 一个完整的 post 示例

```markdown
---
layout: post
title: "文章标题"
date: 2026-10-09 13:00:00 +0800
categories: [分类名]
tags: [标签1, 标签2, 标签3]
---

## 正文标题

正文内容……
```

frontmatter 必填：`layout`、`title`、`date`。**`date` 必须在过去**（Jekyll 默认 `future: false`，未来日期的文章不会渲染）。

---

## 三、自动化更新：让博客自己长出来

### 3.1 核心问题：博客不会自己写文章

博客死掉的最大原因不是没技术，是**没有持续的内容来源**。**3 个内容来源**让博客不断更：

1. **每日自动化 commit**（保活绿墙）
2. **RSS 抓取 + 自己改写**（生成文章素材）
3. **日常工作的副产品**（整理 KB 时顺便发博客）

### 3.2 自动 commit 脚本

```bash
#!/usr/bin/env bash
# ~/.hermes/scripts/blog-auto-update.sh
BLOG_DIR="/path/to/tech-blog"
cd "$BLOG_DIR" || exit 1

mkdir -p _data
DATE=$(date '+%Y-%m-%d %H:%M')
echo "🔄 最后更新: $DATE" > _data/uptime.yml

git add -A
if git diff --cached --quiet; then exit 0; fi

git commit -m "📝 update: $DATE"

# 国内网络：HTTP/1.1 + 长超时 + 重试3次
for i in 1 2 3; do
    if git -c http.version=HTTP/1.1 -c http.lowSpeedLimit=100 -c http.lowSpeedTime=90 push origin main 2>&1; then
        exit 0
    fi
    sleep 5
done
exit 1
```

注册成 cron（每天 9:00 和 21:00）：

```bash
# cron 创建
job_name="博客自动更新"
schedule="0 9,21 * * *"
```

### 3.3 GitHub 绿墙不计数的问题

**坑**：自动 commit 用的是 `git config user.email` 的邮箱，但**如果这个邮箱没关联到 GitHub 账号**，commit 不会计入贡献图（绿墙），也不会显示头像。

**诊断**：
```bash
curl -s "https://api.github.com/repos/USER/REPO/commits?per_page=3" \
  | python3 -c "import sys,json; [print(c.get('author')) for c in json.load(sys.stdin)]"
```

如果 `author: N/A` → 邮箱未关联。

**修复**：在 GitHub Settings → Emails 中添加并验证这个邮箱。

---

## 四、国内网络：让 git push 不超时

### 4.1 三个稳定性方案

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **HTTP/1.1 + 长超时** | 普通 commit | 简单 | 大文件可能失败 |
| **.git/config 永久配置** | 长期稳定 | 一次配置 | 只能针对该仓库 |
| **SSH over 443** | 企业网络限制 | 绕过防火墙 | 配置复杂 |

### 4.2 推荐：HTTP/1.1 + 重试 3 次

```bash
git -c http.version=HTTP/1.1 \
    -c http.lowSpeedLimit=100 \
    -c http.lowSpeedTime=90 \
    push origin main
```

实测：普通 commit 失败概率从 30% 降到 5%，加 3 次重试后基本 100% 成功。

### 4.3 .git/config 永久配置

```ini
[http]
    version = HTTP/1.1
    lowSpeedLimit = 100
    lowSpeedTime = 90
    postBuffer = 524288000
```

写进博客目录的 `.git/config`，所有 push 自动应用。

### 4.4 Token 过期问题

**症状**：`fatal: Authentication failed for 'https://github.com/...'`

**解决**：GitHub PAT token 过期了。重新生成 fine-grained token，更新 remote URL：

```bash
# Fine-grained token 必须用 x-access-token
git remote set-url origin "https://x-access-token:NEW_TOKEN@github.com/USER/REPO.git"
```

---

## 五、内容流水线：从日常到博客

### 5.1 三层 KB 架构

```
KB/                         # 知识库
├── 50-公安联考/            # 备考类
├── 学习资料/               # 日常抓取
│   ├── AI学习/
│   │   ├── 文章/           # 深度文章
│   │   └── 快讯/           # 短消息（早报、简讯）
│   ├── 技术文章/
│   ├── 网络安全/
│   └── 公安联考/
├── 10-Projects/            # 项目笔记
└── 20-Academic/            # 学术笔记
```

**「文章/」存深度内容，「快讯/」存短消息**。每攒 30-50 篇整理一次。

### 5.2 RSS 自动抓取

```python
# ~/.hermes/scripts/rss_monitor.py
# 每天 08:00 抓取 RSS，按主题分类归档
import feedparser
from datetime import date

SOURCES = [
    ('AI学习', 'https://rsshub.app/...'),
    ('技术文章', 'https://sspai.com/feed'),
    ('网络安全', 'https://github.blog/security/feed/'),
]

for category, url in SOURCES:
    feed = feedparser.parse(url)
    for entry in feed.entries[:5]:
        save_to_kb(category, entry)
```

cron 每天 08:00 触发。

### 5.3 KB → 博客的改写流程

不是直接复制 RSS 内容到博客（版权 + 缺乏深度），而是：

```
RSS 文章
  ↓ 阅读理解（KB 已有完整版）
  ↓ 提取核心观点
  ↓ 加入自己的分析（结合实际项目经验）
  ↓ 用自己的语言重写
  ↓ 无图片（避免版权）
  ↓ 字数 2000-4000
  ↓ 提交博客
```

**关键**：**博客上读不出原文的影子**，只看到你对该主题的独立思考。

---

## 六、Jekyll 构建的 3 个致命坑

### 6.1 `where_exp` 漏引号

```liquid
{% assign x = collection | where_exp:"item", item.id == 1 %}
```

**错**：第二个参数不带引号 → `Expected end_of_string but found id`。

**对**：
```liquid
{% assign x = collection | where_exp:"item", "item.id == 1" %}
```

**症状**：GitHub Pages 整个构建失败，首页看不到新文章。**Warning 不会终止，Exception 才会**。

### 6.2 文章日期在未来

Jekyll 默认 `future: false`，未来日期的文章不会渲染。**症状**：本地能看见，GitHub 上没有。

**预防**：写文章前先 `date '+%Y-%m-%d %H:%M:%S %z'` 确认当前时间，按当前时间设置 frontmatter。

### 6.3 `paginate:` 配置但没装插件

GitHub Pages 不支持自定义 jekyll 插件（除了 `whitelist` 几个），但很多人装 `jekyll-paginate`。**症状**：本地 OK，GitHub 构建失败。

**解决**：用客户端 JS 分页，不依赖插件：

```html
<!-- index.html -->
<script>
fetch('/assets/search.json').then(r => r.json()).then(posts => {
    const perPage = 10;
    let page = 1;
    function show() {
        const start = (page-1) * perPage;
        const slice = posts.slice(start, start + perPage);
        // 渲染 + 翻页按钮
    }
});
</script>
```

---

## 七、持续运营：让博客不死掉

### 7.1 三个核心指标

| 指标 | 含义 | 监控 |
|------|------|------|
| **commit 频率** | 博客活跃度 | GitHub 贡献图 |
| **文章数** | 内容深度 | search.json 长度 |
| **页面构建** | 站点可用性 | Actions 状态 |

### 7.2 写博客的最佳节奏

不要追求每天一篇（容易疲劳），目标是**每周 1-2 篇 + 每天 commit**：

| 时段 | 操作 |
|------|------|
| 9:00 | 自动化 commit（保活）|
| 13:00-14:00 | RSS 阅读 + KB 整理 |
| 21:00 | 自动化 commit + 当天总结 |
| 周末 | 整理一周 KB → 发 1 篇博客 |

### 7.3 KB 和博客的双向链接

**Obsidian 的 `[[双链]]` 让 KB 和博客形成网状**：

```markdown
# KB/学习资料/AI学习/文章/Agent框架.md
这篇文章讨论了 [[某开源 Agent 框架]] 的设计

# 博客文章
参考了我之前的 KB 笔记 [[Agent框架]]
```

**双向链接的好处**：你在 KB 写笔记，几个月后写博客时可以快速找到相关素材；反之博客读者看到引用可以跳回 KB。

---

## 八、完整的「博客搭建清单」

如果你现在想从零搭一个博客，按这个清单走：

1. ☐ 创建 GitHub 仓库 `<username>.github.io`
2. ☐ 写 `_config.yml`（远程主题 + 插件）
3. ☐ 写 `_layouts/default.html` + `post.html`
4. ☐ 写 `assets/css/style.css`（极简黑白）
5. ☐ 创建 `index.md`（layout: default）+ `about.md`
6. ☐ 创建第一篇 `_posts/YYYY-MM-DD-first-post.md`
7. ☐ `git add && git commit && git push`
8. ☐ 等 1-2 分钟，访问 `https://<username>.github.io`
9. ☐ 设置自动 commit 脚本（保活绿墙）
10. ☐ 配置 RSS 抓取 → KB → 博客流水线

---

## 总结

博客不是技术活，是**长期主义**。

- 搭博客 1 天
- 写第一篇 1 周
- 持续写 3 年
- 形成知识体系 5 年

**前 3 个月最痛苦**，没什么流量、没什么反馈。坚持下来 1 年后，回头看会觉得所有文章都是自己的「思维资产」——它们**只属于你**，永远在网络上，不会丢失。

---

## 参考

- [Jekyll 官方文档](https://jekyllrb.com/docs/)
- [GitHub Pages 文档](https://docs.github.com/en/pages)
- [pages-themes/hacker](https://github.com/pages-themes/hacker)
- [Liquid 模板语法](https://shopify.github.io/liquid/)
