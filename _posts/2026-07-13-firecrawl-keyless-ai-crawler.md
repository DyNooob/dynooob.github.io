---
layout: post
title: "Firecrawl 去掉了 API Key：AI 爬虫进入无钥匙时代"
date: 2026-07-13 23:22:00 +0800
categories: [AI, 开发]
tags: [Firecrawl, 爬虫, MCP, AI Agent, 开源]
---

Firecrawl 最近做了一个大胆的改动：去掉了 API Key。

现在你调用它的接口，不需要注册、不需要申请密钥、不需要配置环境变量。直接调就行，每月还白送 1000 次免费额度。

## Firecrawl 是什么

一个专门给 AI 用的网页数据接口。你给它一个网址，它返回：

- 干净的 Markdown 正文（去掉了导航栏、广告、页脚）
- 结构化的 JSON（你定义 schema，它按结构提取）
- 截图、HTML 元数据

此外还支持网页爬取、本地文件解析、arXiv 语义搜索、异步浏览器研究 Agent、GitHub 仓库信息搜索。

这个项目在 GitHub 上有 130K+ Star，属于社区 Top 100 仓库。有 15 万+ 家公司在用，包括 Apple、Canva、Stanford、Zapier、Replit 等。

它的 MCP 服务已经被安装过 40 多万次，可能是世界上安装量最多的 MCP 之一。

## 三个核心能力

- **Search**：搜索整个互联网，每个结果直接带完整网页内容
- **Scrape**：抓取单个页面，JS 渲染、动态加载都能处理
- **Interact**：让 AI 在网页上点击、填表、翻页、走登录流程

简单说，它是 AI Agent 的眼睛和手。

## 无 Key 模式怎么用

三个入口同时支持无 Key 调用：

**MCP（推荐）**
```bash
claude mcp add --transport http firecrawl https://mcp.firecrawl.dev/v2/mcp
```
Agent 自己就能完成接入，不需要你在中间手动传 Key。

**CLI**
```bash
npx firecrawl-cli@latest
```

**REST API**
以前调 API 要写 Authorization header，现在直接调：
```bash
curl https://api.firecrawl.dev/v2/scrape
```

每月 1000 次免费额度自动给，用超了再去注册账号升级付费。

## 这背后的逻辑

表面上看只是去掉了一个步骤。但仔细想想，Firecrawl 想得很清楚：

API Key 是给人的——开发者注册、付费、管理 Key。但 Agent 不会注册账号，也不会自己绑邮箱。它只会调用接口。

当 AI Agent 越来越多地成为 API 的主要消费者时，无 Key 调用就会从特权变成默认。Firecrawl 这一步，等于提前押注了这个趋势。

这跟它一直以来开源、免费送额度的策略是一脉相承的：先把开发者心智占住，规模化阶段再变现。典型的基础设施卡位战打法。

## 参考

- [Firecrawl GitHub](https://github.com/nicegui/firecrawl)
- [Firecrawl 官网](https://firecrawl.dev)