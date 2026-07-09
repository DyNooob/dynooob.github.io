---
layout: post
title: "TCP RTT 与 RTO：自适应超时重传机制详解"
date: 2026-06-29 16:00:00 +0800
categories: [网络技术]
tags: [TCP, RTT, RTO, 可靠传输]
---

## 什么是 RTT 和 RTO？

**RTT（Round Trip Time）**：数据包从发送到收到确认的时间。

**RTO（Retransmission Timeout）**：发送方等待确认的超时时间，超过这个时间就重传。

RTO 不是固定值，而是根据网络状况动态调整的——网络快就短，网络慢就长。

## 核心公式

```
EstimatedRTT = (1-α) × EstimatedRTT + α × SampleRTT
DevRTT = (1-β) × DevRTT + β × |SampleRTT - EstimatedRTT|
RTO = EstimatedRTT + 4 × DevRTT
```

- α = 0.125（推荐值）
- β = 0.25（推荐值）
- 初始 EstimatedRTT = 第一次采样值
- 初始 DevRTT = EstimatedRTT / 2

## 计算示例

**初始值：** EstimatedRTT = 30ms，DevRTT = 5ms

| 采样 | SampleRTT | EstimatedRTT | DevRTT | RTO |
|------|-----------|-------------|--------|-----|
| - | 初始 | 30.00 | 5.00 | **50.00** |
| ① | 28 | **29.75** | **4.19** | **46.50** |
| ② | 34 | **30.28** | **4.07** | **46.56** |
| ③ | 25 | **29.62** | **4.21** | **46.45** |

## 规律分析

1. **EstimatedRTT 平滑变化**。α = 0.125 让新采样只占 12.5% 权重，不会被单个异常值带偏。
2. **RTO 始终大于 EstimatedRTT**。+4×DevRTT 留出了余量，避免不必要的重传。
3. **网络波动越大，RTO 越大**。DevRTT 增加时 RTO 自动增长，适应网络变化。

## Karn 算法

如果发生重传，收到的 ACK 对应的是原始报文还是重传报文？Karn 算法规定：
- 重传时不采样 RTT
- 重传后收到 ACK 也不采样
- 直到下一次正常传输才恢复采样

> **简单说：** TCP 通过动态计算 RTT 和 RTO，自适应网络状况——网络通了就快发，网络堵了就慢发，丢包了就重传。这是 TCP 可靠传输的核心机制之一。