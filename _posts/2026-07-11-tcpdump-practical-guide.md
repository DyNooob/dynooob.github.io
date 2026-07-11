---
layout: post
title: "tcpdump 实战：五个日常抓包场景"
date: 2026-07-11 15:00:00 +0800
categories: [网络技术, 安全]
tags: [tcpdump, 抓包, 网络诊断, 排查]
---

tcpdump 是最常用的命令行抓包工具，没有图形界面依赖，服务器排查问题基本靠它。

但很多人只会 `tcpdump -i eth0`，抓到一堆数据包不知道怎么过滤。这篇文章整理五个日常场景，覆盖 90% 的使用需求。

## 场景一：抓某个端口的流量

```bash
# 抓 80 端口（HTTP）
tcpdump -i eth0 port 80

# 抓 443 端口（HTTPS）
tcpdump -i eth0 port 443

# 抓多个端口
tcpdump -i eth0 port 80 or port 443
```

加上 `-n` 不解析主机名，避免 DNS 查询干扰：

```bash
tcpdump -i eth0 -n port 80
```

## 场景二：抓某个主机的流量

```bash
# 抓与特定 IP 的通信
tcpdump -i eth0 host 192.168.1.100

# 只抓从该 IP 发出的
tcpdump -i eth0 src host 192.168.1.100

# 只抓发往该 IP 的
tcpdump -i eth0 dst host 192.168.1.100
```

## 场景三：抓 TCP 三次握手

排查连接问题最常用的场景。看 SYN、SYN-ACK、ACK 有没有丢：

```bash
tcpdump -i eth0 -n "tcp[tcpflags] & (tcp-syn) != 0"
```

更直观的写法（BPF 语法）：

```bash
tcpdump -i eth0 -n 'tcp[13] & 2 != 0'
```

抓 SYN-ACK 包：

```bash
tcpdump -i eth0 -n 'tcp[13] = 18'
```

## 场景四：抓 HTTP 请求内容

```bash
# 抓 HTTP GET/POST 请求行
tcpdump -i eth0 -A -s 0 port 80 | grep -E "(GET|POST|Host:)"

# 保存到文件后用 Wireshark 分析
tcpdump -i eth0 -w capture.pcap port 80
```

`-A` 以 ASCII 格式显示包内容，`-s 0` 抓完整包（不截断）。`-w` 保存为 pcap 文件，可以用 Wireshark 打开。

## 场景五：抓 DNS 查询

```bash
# 抓 DNS 请求和响应
tcpdump -i eth0 -n port 53

# 只看 DNS 响应
tcpdump -i eth0 -n 'udp port 53 and udp[10] & 0x80 != 0'
```

## 实用参数组合

### 最简单的排查命令

```bash
tcpdump -i eth0 -n -v port 80
```

`-v` 显示更详细的信息，适合快速定位问题。

### 不看握手包，只看数据

```bash
tcpdump -i eth0 -n 'tcp port 80 and (tcp[13] & 8 != 0 or tcp[13] & 16 != 0)'
```

这个过滤掉 SYN、FIN、RST 等控制包，只看 PUSH（数据）和 ACK 包。

### 限制包数量

```bash
tcpdump -i eth0 -c 100 port 80
```

抓到 100 个包自动退出，适合测试环境。

## 理解 BPF 表达式

tcpdump 的过滤语法是 Berkeley Packet Filter（BPF）。核心规则：

- `host`：主机过滤
- `port`：端口过滤
- `net`：网段过滤，如 `net 192.168.1.0/24`
- `src/dst`：源/目标方向
- `and/or/not`：逻辑组合

组合示例：

```bash
# 抓来自 192.168.1.x 网段、发往 80 或 443 端口的包
tcpdump -i eth0 -n 'src net 192.168.1.0/24 and (dst port 80 or dst port 443)'
```

## 保存与读取

```bash
# 保存
tcpdump -i eth0 -w output.pcap

# 读取（不需要 root）
tcpdump -r output.pcap

# 读取时再加过滤
tcpdump -r output.pcap -n port 443
```

保存的 pcap 文件可以拷到本地用 Wireshark 打开，图形界面分析更方便。

## 快速参考

| 需求 | 命令 |
|------|------|
| 看某个端口 | `tcpdump -i eth0 port 80` |
| 看某个 IP | `tcpdump -i eth0 host 1.2.3.4` |
| 保存文件 | `tcpdump -i eth0 -w file.pcap` |
| 读文件 | `tcpdump -r file.pcap` |
| 不解析名字 | `tcpdump -n` |
| 只看数据 | `tcpdump -q` |
| 看内容 | `tcpdump -A`（ASCII）或 `-X`（hex+ASCII） |
| 限制数量 | `tcpdump -c 100` |

这五个场景覆盖了日常排查的大部分需求。遇到网络问题，先 `tcpdump -i eth0 -n port <怀疑的端口>` 看一眼，往往比猜配置快得多。