---
layout: post
title: "Linux 网络性能排查：从 ping 到 tcpdump 的排查路径"
date: 2026-07-17 11:00:00 +0800
categories: [网络技术, 开发]
tags: [Linux, 网络排查, 性能, 排障]
---

网络出问题的时候，很多人第一反应是 ping。ping 不通就说网络断了。但实际排查中，问题往往藏在更细的层面——延迟高、丢包、连接重置、DNS 解析慢。这篇文章整理一条从简单到深入的排查路径。

## 第一层：连通性检查

```bash
ping -c 10 目标IP
```

ping 能告诉你三件事：通不通、延迟多少、有没有丢包。

如果 ping 不通，先确认目标机器是否禁 ping（很多服务器会禁 ICMP）。换 tcping 或 curl 试：

```bash
# 测试 TCP 连通性（需要安装 tcping）
tcping 目标IP 80

# 或者用 curl
curl -v --connect-timeout 5 http://目标IP:端口
```

## 第二层：路由追踪

ping 通但延迟高，用 traceroute 看路径：

```bash
traceroute -n 目标IP
mtr -n 目标IP   # MTR 结合了 traceroute 和 ping，持续检测
```

MTR 的每一行代表一个跳点，Loss% 列能看出在哪一跳开始丢包。如果最后一跳之前都正常，只有目标丢包，通常是目标机器自身的防火墙限制。如果中间某跳开始丢包，那一段链路有问题。

## 第三层：DNS 排查

很多"网络问题"其实是 DNS 问题。先确认解析是否正常：

```bash
# 查询解析结果
dig 域名 +short

# 查询具体 DNS 服务器
dig @8.8.8.8 域名

# 查看解析耗时
dig 域名 | grep "Query time"

# 检查系统 DNS 配置
cat /etc/resolv.conf
```

如果解析慢，可以换 DNS 服务器或启用本地 DNS 缓存（如 systemd-resolved、dnsmasq）。

## 第四层：端口与服务

能 ping 通但服务连不上，检查端口：

```bash
# 查看本地监听端口
ss -tlnp

# 检查远程端口是否开放
nc -zv 目标IP 端口

# 持续检测端口连通性
while true; do nc -zv 目标IP 端口 2>&1; sleep 1; done
```

`ss` 比 `netstat` 更快更轻量，推荐优先使用。

## 第五层：连接状态

查看系统当前连接的状态分布：

```bash
ss -tan | awk '{print $1}' | sort | uniq -c
```

正常系统应该有大量 ESTABLISHED 和 TIME-WAIT。如果出现大量 SYN-SENT 或 CLOSE-WAIT，说明连接异常：

- SYN-SENT 过多：目标机器不响应，或防火墙拦截
- CLOSE-WAIT 过多：应用层没正确关闭连接，代码 bug
- TIME-WAIT 过多：短连接频繁建立，正常现象，但太多可能耗尽端口

## 第六层：带宽与吞吐

```bash
# iperf3 测试带宽（服务端先启动）
# 服务端：iperf3 -s
# 客户端：iperf3 -c 服务器IP

# 检查网卡流量
ip -s link show eth0

# 实时查看带宽
nload
iftop
```

## 第七层：抓包分析

前面都查不出问题时，直接看数据包：

```bash
# 抓目标端口的包
tcpdump -i eth0 -n port 80 -w capture.pcap

# 看 TCP 握手是否正常
tcpdump -i eth0 -n 'tcp port 80 and tcp[13] & 2 != 0'

# 看重传
tcpdump -i eth0 -n 'tcp port 80 and (tcp[13] & 8 != 0)'
```

抓包能发现一些上层工具看不出来的问题：TCP 窗口太小、零窗口、重传率过高、乱序到达等。

## 排查路径总结

```
问题现象 → 检查方向 → 工具
接通但慢 → 延迟/丢包 → ping -c, mtr
完全不通 → 连通性 → ping, tcping, curl
能通但连不上 → 端口 → nc, ss
时好时坏 → DNS → dig
非常慢 → 带宽 → iperf3, nload
查不出 → 抓包 → tcpdump
```

从最外层开始，逐层深入，不要一上来就抓包。大部分问题在前三层就能定位。