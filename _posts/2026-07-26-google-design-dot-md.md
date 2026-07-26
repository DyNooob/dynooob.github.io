---
layout: post
title: "Google 开源 DESIGN.md：给 AI 看的视觉规范文件"
date: 2026-07-26 18:30:00 +0800
categories: [AI, 前端]
tags: [设计系统, DESIGN.md, Google, 开源, AI编码]
---

用 AI 生成前端界面时有一个普遍问题：AI 生成的页面总是一股模板味。默认 Tailwind 颜色、Inter 字体、靛蓝色按钮——因为训练数据里这类样本最多。让它换风格，它就把紫色换成橙色，仅此而已。

Google Labs 最近开源了一个项目叫 **design.md**，试图解决这个问题。它在 GitHub 上已有近 2 万 Star。

## 设计规范的两层结构

DESIGN.md 是写给 AI Agent 看的设计系统规范文件。类比一下：CLAUDE.md 和 AGENTS.md 是教 AI 怎么写代码，DESIGN.md 是教 AI 怎么写长得对的代码。

文件分两层：

**YAML front matter**：机器可读的设计 token。颜色十六进制值、字号、间距、圆角等精确数值。

**Markdown 正文**：人可读的设计理念——为什么主色是这个、为什么按钮要做成圆角。

AI 既拿到精确数值可以执行，又理解设计意图可以判断。以下是官方 Heritage 示例：

```yaml
---
name: Heritage
colors:
  primary: "#1A1C1E"
  secondary: "#6C7278"
  tertiary: "#B8422E"
  neutral: "#F7F5F2"
typography:
  h1:
    fontFamily: Public Sans
    fontSize: 3rem
---
## 配色
- 主色(#1A1C1E): 深墨色，用于标题和正文核心文字
- 次色(#6C7278): 沉稳石板灰，用于边框、说明文字
- 强调色(#B8422E): 唯一的交互驱动色
- 中性色(#F7F5F2): 暖石灰底色，比纯白更柔和
```

AI 读了这个文件，会输出一个 Public Sans 字体、深色标题、暖灰底色、红色 CTA 按钮的页面。每个值都有出处，每段意图都有解释。

## 配套 CLI

`@google/design.md` 零配置直接运行：

```bash
npx @google/design.md lint DESIGN.md
```

四条命令：

- **lint**：校验文件，跑 8 条规则（broken 引用、缺失主色、WCAG 对比度、孤立 token 等）
- **diff**：对比两个版本，发现 token 级别的回归
- **export**：导出 Tailwind 主题配置或 W3C DTCG 标准 token
- **spec**：输出规范文档，塞进 AI 的 system prompt

## 生态

Google 定标准格式，社区填内容。VoltAgent 的 awesome-design-md 仓库把 Claude、Notion、Apple 等知名网站的视觉风格逆向翻译成 DESIGN.md 文件，已有 70+ 品牌样本。

还有 designkit.sh、getdesign.md 等第三方聚合站专门收集和浏览 DESIGN.md 文件。

## 配合 Claude Code 使用

在项目根目录创建 DESIGN.md 文件，在 CLAUDE.md 里加一行让 AI 去读它，就能让 AI 生成统一风格的 UI。搭配 Claude Code 的 frontend-design skill 效果更好。

## 参考

- [Google design.md](https://github.com/google-labs-code/design.md)
- [awesome-design-md](https://github.com/voltagent/awesome-design-md)