---
layout: post
title: "reverse-skill：11k Star 的 AI 逆向工程路由框架"
date: 2026-08-03 12:00:00 +0800
categories: [AI, 安全]
tags: [逆向, 安全工具, AI Agent, 技能编排]
---

当 AI 助手面对一个逆向任务时，它可能先跑 jadx 反编译 APK，也可能直接上 Frida Hook，或者用 IDA 打开——选哪个取决于它的训练数据，不一定对。

reverse-skill 解决了这个问题：它是一套写给 AI 看的逆向工程操作手册，告诉 AI 面对不同任务该走哪条路、用什么工具、按什么步骤来。GitHub 11,100 Star，MIT 开源。

## 五层架构

### 路由层

AI 的第一步是回答「我是谁、我要干嘛、我有什么工具」。MASTER-ROUTING.md 定义了 39 条优先级规则（R0-R38），覆盖 APK 逆向、JS 签名分析、无线电信号分析等场景。R1 匹配 APK 任务走 apk-reverse，R3 匹配 JS 签名走 js-reverse，R20 匹配报告生成走 docs-generator。没命中的走 R0 通用逆向。

### 作战契约层

动手之前必须先确认授权。scope-contract.md 是一份授权沙箱——auth 字段必须是 granted，network_profile 必须明确（离线模式/实验室模式/授权目标），才能执行操作。使用 RFC 2119 的 MUST/MUST NOT/SHOULD 语义级别，硬得像合同条款。

### 技能层

30+ 个专业子技能，每个都是独立的 SKILL.md。APK 逆向、JS 前端签名逆向、IDA 深度分析、渗透测试工具链、攻击链编排、固件渗透、LLM 安全测试……每个子技能是「如果……那么……」的决策树加工具链编排。

比如 js-reverse 走五阶段流程：Observe（观察）→ Capture（捕获）→ Rebuild（复现）→ Verify（验证）→ Document（报告）。

### 工具层

tool-index.md 是本地工具注册表，记录每个工具的实际安装路径、版本号和验证命令。bootstrap-reverse 脚本按需安装缺失工具——支持 jadx、apktool、Frida、IDA、Ghidra、BurpSuite、Nmap 等 20+ 工具链。安装后自动刷新注册表。

### 进化层

每次完成任务，AI 必须回写经验到 field-journal/。有模板、有索引、有先例。下次同类任务先查索引，复用经验。这就是自我进化。

## 借口中止表

RULES.md 里有一个中英文双语的「Excuse Rebuttal Table」，列出 AI 常见的 9 种偷懒借口和对应的驳回话术：

| AI 借口 | 系统驳回 |
|---------|---------|
| 这一步可以跳过，我先…… | 禁止跳过，输出理由等用户确认 |
| 我觉得用户不需要这个 | 永远不要替用户决定 |
| 我已经会了，没必要读 X | 先读 X，再动手 |
| 任务基本做完了，清单不用填了吧 | 清单没打勾 = 任务没完成 |

这套设计揭示了当前 AI 助手的核心弱点：它们会假装工作、会偷懒、会重复犯错。reverse-skill 不是靠更聪明的模型来解决，而是靠更严格的流程来约束。

## 跨平台部署

Windows 用 PowerShell 脚本，Kali Linux 有专属入口，Ubuntu/macOS 走 Bash 脚本。每种平台都有独立的部署文档。

**全局注入机制**：AI 第一次使用 reverse-skill 时，会自动把路由规则写入全局配置文件。以后在任何项目目录下，只要提到 APK 逆向、渗透测试等关键词，路由规则就会被自动触发，不需要每次都手动引入。

## 参考

- [reverse-skill GitHub](https://github.com/zhaoxuya520/reverse-skill)