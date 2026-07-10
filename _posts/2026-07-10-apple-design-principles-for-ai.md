---
layout: post
title: "17 条 Apple 设计原则：如何让 AI 写出流畅的交互界面"
date: 2026-07-10 10:00:00 +0800
categories: [前端, AI]
tags: [设计, 动效, Apple, 前端, AI编码, 交互设计]
---

用 AI 写前端的人都有一个共同的痛点：功能能做出来，但动画总差那么一口气。

弹窗出现了，但缓动曲线不对；边框用了纯色，而不是半透明阴影；按钮按下去没有反馈，拖拽松手后直接跳到终点，没有惯性。这些问题每一个单独看都不大，但叠加在一起——界面就从"精致"变成了"能用就行"。

Sonner 作者 Emil Kowalski（前 Vercel 设计工程师，现 Linear 设计团队成员）给了一个解法：**把 Apple 的 17 条设计原则写成规则文件，装进 AI 编码助手。**

AI 编码助手没有品味，但你可以把品味写成规则交给它。

## 谁在做这件事

Emil Kowalski 是 Sonner 的作者——这个 React toast 组件每周 npm 下载量超过 1300 万次。他还做了 Vaul，一个在移动端替代 Dialog 的抽屉组件。曾在 Vercel 设计团队参与 Next.js 文档、Geist 字体和 Vercel Dashboard 的设计，现在在 Linear 的 Web 团队做设计工程师。

2026 年 7 月 9 日，他在 X 上宣布了一个新 skill：`/apple-design`。

> "Apple's WWDC videos are a goldmine of knowledge. I've combed through my favorite ones and came up with 17 design and motion principles."

不到一天，这条推文获得近 12 万次观看，近 5000 人收藏——说明大量开发者有同样的感受：AI 写代码够用了，但 AI 做界面，确实需要一份"品味说明书"。

## 这不是一个 UI 库，而是一个"AI 品味文件"

所谓的 skill，是给 AI 编码助手（如 Claude Code、Cursor）用的规则文件。纯 Markdown，没有任何代码依赖，可以直接打开 `SKILL.md` 阅读。

Emil 的仓库 [emilkowalski/skills](https://github.com/emilkowalski/skills)（GitHub 5,900+ Star）目前包含四个 skill：

- **emil-design-eng**：面向设计工程师的综合 skill，以动画为主，兼顾设计建议
- **review-animations**：按一套严格规则审查交互动效
- **animation-vocabulary**：用准确的词汇告诉 AI 你想要什么效果
- **apple-design**：17 条 Apple 级设计原则，适配 Web 平台

安装只需一行：

```bash
npx skills@latest add emilkowalski/skills
```

之后 AI 编码助手在生成界面时会自动参考这些规则。

## 17 条原则的核心逻辑

这些原则的核心来源是 Apple WWDC 2018 的经典演讲 **"Designing Fluid Interfaces"**，辅以 WWDC 2020 的 "The Details of UI Typography" 和 2026 年的 "Principles of Great Design"。Emil 的评价是："Designing Fluid Interfaces 是 Apple 最好的演讲之一，那里的建议是永恒的。"

这套原则不是在教你"怎么做动画"，而是在教你"怎么让界面不再像一台电脑"。

### 1-3：响应 · 操控 · 可中断

按下按钮的瞬间就要有反馈——不是等手指抬起来才高亮。拖拽时元素必须 1:1 跟随手指，而且要尊重抓取的位置偏移。

最关键的是：**任何一个动画，用户在中间任何时候都可以抓住它、反向操作**——不用等动画播完。CSS transition 做不到这个，需要 spring（弹簧动画）。

### 4-6：弹簧 · 速度 · 动量

Apple 把物理课本里的 mass/stiffness/damping 三元组转换成了设计师友好的两个参数：

- **Damping ratio（阻尼比）**：控制回弹
- **Response（响应时间）**：控制速度

Emil 在 skill 里直接给出了 Apple 的实际数值：

| 交互类型 | 阻尼比 | 响应时间 |
|---------|--------|---------|
| 移动/重定位（如 PiP） | 1.0 | 0.4s |
| 旋转 | 0.8 | 0.4s |
| 抽屉/底部弹出层 | 0.8 | 0.3s |

默认值：阻尼比 1.0（临界阻尼，无回弹），优雅且不分散注意力。只在用户做了抛掷动作（flick/throw）时才加一点弹性（阻尼比 ~0.8）。

**重点**：松手瞬间，动画必须以手指离开时的精确速度继续——这是"流畅"和"还行"之间最关键的差距。

### 7-10：路径对称 · 来源锚定 · 橡皮筋 · 方向提示

- 一个面板从右边滑进来，就必须从右边滑出去——进右出下会让人困惑
- 弹窗、菜单、浮层要从触发它的按钮"长"出来（设置 `transform-origin` 到触发位置）
- 拖到边界时用渐进阻力，就像橡皮筋——拉得越远阻力越大。硬停读起来像界面冻住了

### 11-15：帧率 · 材质 · 反馈 · 无障碍 · 排版

- 每一帧的位移量不能超过人眼的感知阈值
- 用 `requestAnimationFrame`，只动 `transform` 和 `opacity`
- 模糊材质（`backdrop-filter`）传达层级关系
- 声音、触觉、视觉反馈必须在同一帧触发
- `prefers-reduced-motion` 不是"去掉动画"，而是"换成更温和的等价体验"
- 大标题用负 `letter-spacing`，正文接近 0

### 16-17：哲学与流程

Apple 的八大设计原则（Purpose / Agency / Responsibility / Familiarity / Flexibility / Simplicity / Craft / Delight）是这 17 条的底层。

> "Delight 不是撒在上面的彩色纸屑，而是你把前七个原则做对之后自然产生的结果。"

原型交互比一百张静态设计稿都有用。

## 品味可以编码

Emil 在博客 [Agents with Taste](https://emilkowal.ski/ui/agents-with-taste) 里把这件事讲得最透：

> "There's no magic involved. Almost every 'taste' decision has a logical reason if you look close enough."

比如——为什么一个元素出现时应该从 `scale(0.95)` 开始，而不是 `scale(0)`？因为现实世界里一个气球放气之后仍然有形状，它从不会完全消失。`scale(0)` 让元素看起来像凭空冒出来的。

这就是"品味"背后的"为什么"。

他在博客里展示了一些打包成 skill 的规则，比如缓动曲线决策流程：

```
元素正在进入/离开视口？ → ease-out
元素在屏幕上移动/变形？ → ease-in-out  
hover 状态变化？         → ease
恒定运动？               → linear
```

时长指南：
- 微交互：100-150ms
- 标准 UI：150-250ms
- 弹窗/抽屉：200-300ms
- UI 动画不超过 300ms
- 退出动画比进入动画快约 20%

这些规则单独看都不复杂。但当 AI 遵守了所有规则和什么都不遵守——出来的界面就是两个世界。

## 这件事不限于动画

排版、颜色、间距——任何你能说清楚"为什么"的设计决策，都可以编码成 skill。

当然，skill 不能替代你自己的品味。它能把你的品味放大、让 AI 执行得更一致——但规则本身来自 Emil 的经验和审美。你完全可以在它的基础上加自己的规则，定制成自己团队的设计语言。

说到底，Emil 在 README 里的那句话才是这件事的本质：

> "AI doesn't replace such expertise — it amplifies what you can get out of it and makes you way better relative to others. So learn to code, design, or develop expertise in any other field. It's extremely valuable."

## 参考链接

- [emilkowalski/skills](https://github.com/emilkowalski/skills) — GitHub 仓库
- [Designing Fluid Interfaces (WWDC 2018)](https://developer.apple.com/videos/play/wwdc2018/803/) — Apple 原始演讲
- [Agents with Taste](https://emilkowal.ski/ui/agents-with-taste) — Emil 的博客
- [animations.dev](https://animations.dev) — Emil 的动效课程