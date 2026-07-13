---
layout: post
title: "用 AI 写完代码就上线？这 11 个检查点先过一遍"
date: 2026-07-13 14:00:00 +0800
categories: [安全, 开发]
tags: [AI编码, 安全, 合规, 上线检查]
---

用 AI 写代码的人越来越多。Cursor、Claude、GPT——写个 App 或小程序然后直接上线，流程越来越短。

但一位干了 20 年性能和安全的老工程师在 Reddit 上发了一篇帖子，开场就让人警觉：

> "I keep seeing people shipping apps built with vibe coding tools and just pushing them live. That's fine… but also slightly terrifying."

不是要设门槛，而是他见过的绝大多数翻车不是高级攻击——全是基础到不能再基础的疏忽。

下面这 11 条，按漏了哪个后果最严重排了序。

## 第一组：法律底线

### 1. 收集用户数据？先写隐私政策

只要你收集任何用户数据——哪怕只是个邮箱注册——法规就已经适用了。在中国，适用的是《个人信息保护法》（PIPL）和《数据安全法》。

很多人觉得「我就个小 App，没人管我」——等律师函到了就晚了。

### 2. 搞清楚用户数据存在哪

能说清楚吗：哪个服务器？哪个地区？谁有权限？有没有加密？

隐私政策 + 数据存储，这两项不做不是技术债，是法律风险。

## 第二组：安全基线

### 3. 检查安全 Headers

整张清单里性价比最高的就是这条。把下面这句 prompt 丢给 AI：

> "Review my app as a security specialist and make sure I have strong security headers and a solid baseline security posture"

两分钟，能挡住 CSP、X-Frame-Options、HSTS 层面一堆常见攻击。

### 4. 按 OWASP 标准扫描

Headers 只是第一道门。SQL 注入、XSS、认证绕过——OWASP Top 10 的老面孔，AI 生成的代码里一样不少。

> "Review my app against OWASP standards and highlight vulnerabilities"

### 5. 专门查 SQL 注入、XSS、认证问题

AI 生成的代码有个特点：特别喜欢字符串拼接 SQL、在前端直接 dangerouslySetInnerHTML 用户输入。

### 6. .env 里的值绝不能出现在前端

AI 写前端代码时偶尔会顺手把环境变量引用带进来。任何一个打开 F12 的人都能看到。

## 第三组：数据泄露

### 7. 检查 API 返回的数据是不是太多了

AI 写接口的经典操作：`return user`，把整张用户表全吐出去。前端明明只用了 user.name，但 Response 里什么都有。

### 8. 日志里绝对不能有密钥

调试时 `console.log(apiKey)` → 日志传到第三方监控 → 密钥泄露。这个链条发生得太频繁了。

### 9. API Key 在前端 = 已经没了

> "If your key is in the browser, assume it's already been taken."

不是可能会被偷，是默认已经被偷了。任何人打开 DevTools → Network → 复制粘贴，三秒完事。

## 第四组：修好后的刹车

### 10. 密钥移到服务端

前端 → 你的后端/代理 → 第三方 API。前端永远不直接拿 Key 调外部服务。

### 11. 加频率限制

AI 做的 App，十个有九个没有 rate limit。有人写个脚本一跑，你下个月的 API 账单就爆了。

## 让 AI 检查 AI 写的代码

流程很简单：用 AI 写完代码 → 把上面的 prompt 丢给同一个 AI → AI 检查 AI 生成的代码有没有安全漏洞。

2026 年最赛博朋克的质控流程。

## 参考

- [Reddit 原帖](https://www.reddit.com/r/vibecoding/comments/1sthzcj/if_youre_about_to_launch_a_vibe_coded_app_read/)
- 《中华人民共和国个人信息保护法》（PIPL）
- OWASP Top 10 (2021)