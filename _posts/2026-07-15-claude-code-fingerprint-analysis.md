---
layout: post
title: "Claude Code 的隐蔽指纹：通过 Unicode 撇号追踪代理来源"
date: 2026-07-15 12:00:00 +0800
categories: [AI, 安全]
tags: [Claude Code, 逆向, 指纹, 代理检测, 安全分析]
---

Claude Code 的二进制文件中隐藏着一套流量溯源机制。它会在系统提示词中嵌入一个不可见的 Unicode 指纹，用于识别请求是否通过第三方代理转发。

## 指纹机制

Claude Code 每日启动时生成的日期字符串看起来是普通的英文格式：

```
Today's date is 2026/07/15.
```

但其中的撇号字符不是标准的 U+0027（'），而是根据请求来源选择四种不同的 Unicode 码点之一：

| 撇号 | Unicode | 触发条件 |
|------|---------|---------|
| ' | U+0027 | 直连官方 API，不触发指纹 |
| ' | U+2019 | 域名匹配代理列表 |
| ʼ | U+02BC | 关键词匹配中国 API 服务商 |
| ʹ | U+02B9 | 域名和关键词同时命中 |

日期格式也有变化：中国时区（Asia/Shanghai、Asia/Urumqi）使用斜杠分隔（2026/07/15），其他时区使用连字符（2026-07-15）。

这样组合出 8 种不同的指纹，Anthropic 的服务端可以通过解析 prompt 中的日期字符串，判断请求来源的代理类型和用户时区。

## 触发条件

指纹只在设置了 `ANTHROPIC_BASE_URL` 环境变量且指向非官方 API 端点时触发。判断逻辑包含两个维度：

- **域名列表**：147 条预编码域名，覆盖中国科技公司（百度、阿里、字节、腾讯等）和已知 Claude 代理服务
- **关键词列表**：11 条关键词，匹配 deepseek、moonshot、minimax、zhipu、baichuan 等中国 AI 服务商

## 技术实现

二进制文件（Bun 编译的 Mach-O ARM64）中保留了完整的 JavaScript 函数体。通过 strings 扫描可以直接提取源码，无需反汇编。

编码数据使用 XOR（密钥 91）+ Base64 两层编码：

```python
import base64
data = "ODV3KDo1MC46MnU4NDZ3..."
decoded = base64.b64decode(data)
result = bytes([b ^ 91 for b in decoded])
print(result.decode())
# → cn,sankuai.com,netease.com,163.com,...
```

## 动机分析

这套机制本质上是服务端流量溯源水印，类比电影盗版溯源水印或机密文件打印追踪点阵。

Anthropic 不对中国区直接开放服务，中国用户通过代理访问时，代理使用自己的 API Key 转发请求。从 Anthropic 视角看，所有请求都来自代理服务器的 IP，看起来像一个合法的海外大客户。指纹的价值在于穿透这一层代理，识别终端用户的真实来源。

选择隐蔽而非直接封禁，可能的原因是：先收集情报了解代理生态的规模、结构和行为模式，为未来的精准策略做数据准备。直接封禁会导致代理快速更换域名，而误伤企业网关也是不希望看到的。

## 对抗与限制

这套指纹的不可绕过性在于：代理必须原样转发 prompt 文本才能正常工作。指纹嵌入在合法的 prompt 文本中，修改它会影响模型输入。

但机制本身也有局限。它不检查公网 IP、VPN 连接状态或物理位置，只依赖 `ANTHROPIC_BASE_URL` 环境变量。因此 VPN + 直连官方 API 的场景不会触发指纹。

## 参考

- 逆向目标：Claude Code v2.1.196（Bun 编译，Mach-O ARM64）