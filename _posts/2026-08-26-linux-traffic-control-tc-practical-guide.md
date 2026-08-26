---
layout: post
title: "Linux Traffic Control (tc) 实战指南：限速、整形与网络模拟"
date: 2026-08-26
categories: 网络技术
tags: [tc, linux, networking, qos, bandwidth, netem, traffic-shaping, network]
---

## 为何需要流量控制

带宽不是无限的。无论你管理的是 10Gbps 的 IDC 出口，还是开发机上的一条 1Gbps 链路，总有某个时刻，一个进程会吃掉所有带宽，让其他服务窒息。Linux Traffic Control（tc）就是用来解决这个问题的——它能在内核层面精确控制数据包的发送速率、优先级、延迟和丢包率。

tc 不是防火墙。iptables/nftables 决定「允许还是拒绝」，tc 决定「多快、什么时候、按什么顺序」发送。它工作在 OSI 模型第二~三层，位于内核协议栈的出口（egress）和入口（ingress）路径上。

本文从零开始，覆盖 tc 的核心概念、常用 qdisc（排队规则）、实战限速配置、网络模拟（延迟/丢包/乱序），以及生产环境中的调试技巧。

## 核心概念：三要素

tc 有三个基本构建块：

| 术语 | 说明 |
|------|------|
| **qdisc** (Queueing Discipline) | 排队规则，决定数据包如何排队和发送 |
| **class** | 流量类别，用于在 qdisc 内部做层次化流量划分 |
| **filter** | 过滤器，根据包特征（IP、端口、tcindex 等）将流量分配到指定 class |

默认情况下，每个网络接口都有一个 `pfifo_fast` qdisc——一个简单的三优先级 FIFO 队列。你能看到的几乎所有 tc 配置，都是把这个默认 qdisc 替换成更复杂的方案。

### 快速验证

```bash
# 查看 eth0 当前的 qdisc
tc qdisc show dev eth0

# 查看 eth0 的 class 和 filter
tc class show dev eth0
tc filter show dev eth0
```

刚启动的服务器上，你看到的通常是 `pfifo_fast` 或 `fq_codel`（取决于发行版）。

## 常用 qdisc 详解

### 1. pfifo_fast — 默认值

三个 band（0/1/2），band 0 优先级最高。包根据 TOS 字段被分到不同 band。大多数场景下不需要动它。

### 2. fq_codel — 公平队列 + 控制延迟

`fq_codel` 是 systemd 时代很多发行版的默认 qdisc。它给每个流分配独立的队列，并用 CoDel 算法主动检测和缓解缓冲区膨胀（bufferbloat）。如果你不需要复杂的层次化限速，对延迟敏感的服务（如 SSH、DNS、Web 实时通信）直接用 `fq_codel` 就很好：

```bash
tc qdisc replace dev eth0 root fq_codel
```

### 3. tbf — Token Bucket Filter

最简单的速率限制器。它维护一个令牌桶，按固定速率产生令牌，包必须拿到令牌才能发送。适合做「单接口出口限速」：

```bash
# 限制 eth0 出口速率 100Mbps，burst 32KB
tc qdisc add dev eth0 root tbf rate 100mbit burst 32kbit latency 50ms
```

参数含义：
- `rate`：长期平均速率
- `burst`：瞬时突发的最大字节数
- `latency`：包在队列中的最大等待时间

### 4. htb — Hierarchical Token Bucket

HTB 是生产环境中最常用的 qdisc，因为它支持多层次分类限速。你可以为不同业务分配不同的带宽，并允许它们借用空闲带宽：

```bash
# 创建 HTB 根队列，总带宽 1Gbps
tc qdisc add dev eth0 root handle 1: htb default 30

# 定义根类（1:1），速率 1Gbps
tc class add dev eth0 parent 1: classid 1:1 htb rate 1gbit

# 子类：关键业务，保证 500Mbps，最大 1Gbps
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 500mbit ceil 1gbit

# 子类：普通业务，保证 200Mbps，最大 500Mbps
tc class add dev eth0 parent 1:1 classid 1:20 htb rate 200mbit ceil 500mbit

# 子类：后台任务，保证 50Mbps，最大 200Mbps
tc class add dev eth0 parent 1:1 classid 1:30 htb rate 50mbit ceil 200mbit
```

HTB 的 `rate` 是保证带宽，`ceil` 是最大带宽（允许借用父类空闲带宽）。这样设计的好处是：带宽空闲时，任何类都能用到上限；带宽紧张时，每个类至少得到保证值。

### 5. netem — Network Emulator

Netem 不是用来限速的，而是用来模拟糟糕的网络环境——延迟、抖动、丢包、乱序、重复。开发和测试网络应用时，这是必备工具：

```bash
# 模拟 100ms 延迟
tc qdisc add dev eth0 root netem delay 100ms

# 模拟 100ms 延迟 ± 20ms 抖动（正态分布）
tc qdisc add dev eth0 root netem delay 100ms 20ms distribution normal

# 模拟 5% 丢包率
tc qdisc add dev eth0 root netem loss 5%

# 模拟 0.5% 重复包 + 0.1% 损坏包
tc qdisc add dev eth0 root netem duplicate 0.5% corrupt 0.1%

# 组合：延迟 + 丢包 + 抖动
tc qdisc add dev eth0 root netem delay 80ms 10ms loss 2%
```

## 实战场景一：按 IP 限速

假设场景：服务器上有三个服务，分别监听不同端口，需要为每个服务分配独立带宽上限。

```bash
# 1. 根 qdisc
tc qdisc add dev eth0 root handle 1: htb default 20

# 2. 根类
tc class add dev eth0 parent 1: classid 1:1 htb rate 1gbit

# 3. 服务类：HTTP (80) 500Mbps
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 500mbit ceil 1gbit
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 \
    match ip dport 80 0xffff flowid 1:10

# 4. 服务类：API (443) 300Mbps
tc class add dev eth0 parent 1:1 classid 1:11 htb rate 300mbit ceil 1gbit
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 \
    match ip dport 443 0xffff flowid 1:11

# 5. 服务类：SSH (22) 50Mbps 但高优先级
tc class add dev eth0 parent 1:1 classid 1:12 htb rate 50mbit ceil 500mbit prio 0
tc filter add dev eth0 parent 1: protocol ip prio 1 u32 \
    match ip dport 22 0xffff flowid 1:12
```

每个 filter 用 `u32` 匹配器匹配目标端口，将流量导入对应的 class。`prio` 越小的 filter 优先级越高，先匹配先执行。

如果要按来源 IP 限速（比如限制某个客户端的带宽）：

```bash
tc filter add dev eth0 parent 1: protocol ip prio 2 u32 \
    match ip src 10.0.0.0/24 flowid 1:20
```

## 实战场景二：使用 cgroup 进行流量分类

现代 Linux 支持通过 cgroup v2 的 `net_cls` 或 `cgroup` 过滤器将进程的流量自动分类，无需手动写端口匹配。这在容器化环境中特别有用：

```bash
# 1. 创建 HTB 层次
tc qdisc add dev eth0 root handle 1: htb default 30
tc class add dev eth0 parent 1: classid 1:1 htb rate 1gbit
tc class add dev eth0 parent 1:1 classid 1:10 htb rate 500mbit ceil 1gbit

# 2. 添加 cgroup filter
tc filter add dev eth0 parent 1: protocol ip prio 1 cgroup

# 3. 将进程 PID 写入 cgroup 并设置 classid
mkdir -p /sys/fs/cgroup/net_cls
echo "0x0010000a" > /sys/fs/cgroup/net_cls/net_cls.classid
echo <PID> > /sys/fs/cgroup/net_cls/cgroup.procs
```

`net_cls.classid` 的格式是 `0xAAAABBBB`，其中 AAAA 是 major（handle 的 major 部分），BBBB 是 minor（classid 的 minor 部分）。`0x0010000a` 对应 `1:10`。

## 实战场景三：用 netem 模拟弱网环境测试应用

开发网络应用时，需要测试在弱网环境下的表现。用 netem 可以轻易模拟各种场景：

```bash
# 设置模拟环境（假设 eth0 是测试网卡）
tc qdisc add dev eth0 root netem \
    delay 50ms 10ms distribution normal \
    loss 1% \
    duplicate 0.1% \
    reorder 5% 50%

# 运行你的测试...
# curl, ping, 或压测工具

# 测试完成后清理
tc qdisc del dev eth0 root
```

如果只想对特定端口做延迟模拟（不影响其他流量），可以结合 `ifb`（Intermediate Functional Block）设备实现 ingress 方向的流量控制：

```bash
# 1. 加载 ifb 模块并创建虚拟接口
modprobe ifb
ip link set ifb0 up

# 2. 将 eth0 的入口流量重定向到 ifb0
tc qdisc add dev eth0 ingress
tc filter add dev eth0 parent ffff: protocol ip u32 \
    match u32 0 0 action mirred egress redirect dev ifb0

# 3. 在 ifb0 上做流量控制
tc qdisc add dev ifb0 root netem delay 200ms loss 5%
```

这样，进入 eth0 的所有流量都会经过 ifb0 上的 netem 处理，而出站流量不受影响。

## 调试与监控

### 查看实时队列状态

```bash
# 查看 qdisc 统计（包计数、丢弃、超过限制）
tc -s qdisc show dev eth0

# 查看 class 统计
tc -s class show dev eth0
```

输出中的关键字段：
- `Sent`：已发送的字节数和包数
- `dropped`：因队列满丢弃的包数
- `overlimits`：超过 `rate` 或 `ceil` 的次数
- `requeues`：需要重新入队的次数（通常表示锁竞争）

### 使用 `ss` 观察队列深度

```bash
# 查看每个 socket 的发送队列长度
ss -tni

# 重点关注 `skmem` 中的 `r`（接收缓冲）和 `w`（发送缓冲）
```

### 监控工具

```bash
# 使用 iftop 查看实时带宽
iftop -i eth0 -n

# 使用 nethogs 按进程查看带宽
nethogs eth0

# 使用 tcptop (bcc 工具) 查看每个连接的延迟
tcptop
```

## 典型故障排查

### 1. 丢包排查

```
# 发现大量 dropped
tc -s qdisc show dev eth0 | grep dropped
```

如果 `dropped` 持续增长，说明队列容量不够或限速太严。解决方案：
- 增大 `burst` 参数
- 提高 `rate` / `ceil`
- 对于 `htb`，增大 `quantum` 值（默认约 1500 字节，可适当提高到 3000-6000）

### 2. 延迟飙升

如果是 `fq_codel` 或 `cake` 之外的 qdisc，延迟飙升通常意味着 bufferbloat。检查：

```bash
# 发送一个大文件，同时在另一个终端 ping 网关
ping -c 100 <gateway>
```

如果 ping 在有负载时从 1ms 跳到 100ms+，就是 bufferbloat。换成 `fq_codel` 或 `cake` qdisc 就能缓解：

```bash
tc qdisc replace dev eth0 root cake bandwidth 1gbit
```

### 3. 规则不生效

检查 filter 是否匹配到了正确的 class：

```bash
# 用 tc filter 的 monitor 模式（需要内核支持）
# 或者手动构造测试流量并用 tcpdump 验证
tcpdump -i eth0 -n -c 100 port 80
```

常见的 filter 匹配问题：
- `u32` 匹配的 `match` 参数写成了 `ipport` 而非 `ip dport`
- filter 的 `prio` 优先级导致其他规则先匹配了

## 将配置持久化

tc 配置不会跨重启存活。有两种持久化方法：

### 方法一：systemd 服务

```bash
# /etc/systemd/system/tc-rules.service
[Unit]
Description=Traffic Control Rules
After=network.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/tc-rules.sh
ExecStop=/usr/local/bin/tc-rules-stop.sh

[Install]
WantedBy=multi-user.target
```

```bash
# /usr/local/bin/tc-rules.sh
#!/bin/bash
DEV=eth0
tc qdisc add dev $DEV root handle 1: htb default 30
tc class add dev $DEV parent 1: classid 1:1 htb rate 1gbit
tc class add dev $DEV parent 1:1 classid 1:10 htb rate 500mbit ceil 1gbit
tc filter add dev $DEV parent 1: protocol ip prio 1 u32 match ip dport 80 0xffff flowid 1:10
```

### 方法二：tc 配置文件

某些发行版（如 Alpine）支持通过 `tc` 的配置文件持久化，但更通用的做法还是用 systemd service 或 network manager 的 hook 脚本（如 `if-up.d/`）。

## 总结

| 场景 | 推荐方案 |
|------|----------|
| 单接口简单限速 | `tbf` |
| 多层次分类限速 | `htb` |
| 低延迟 + 公平分享 | `fq_codel` |
| 网络模拟（延迟/丢包） | `netem` |
| 按 IP 分流 | `u32` filter + `htb` |
| 按进程/容器分流 | `cgroup` filter + `htb` |
| 入口方向控制 | `ifb` + `ingress` qdisc |
| 现代全功能替代 | `cake` |

tc 是 Linux 网络栈中最被低估的工具之一。它的学习曲线确实陡峭——命名空间、handle、major/minor 这些概念初次接触容易困惑——但一旦掌握，你就能精确控制每一比特的流向。下次遇到「某个服务占满带宽导致其他服务超时」的问题，别急着上 iptables 限流——先去了解 tc，它才是正确的工具。

最后提醒一点：tc 配置始终在出口方向做整形（shaping）。入口方向只能做策略（policing），即直接丢弃或标记，无法缓冲后再发送。如果需要控制入站流量，需要结合 `ifb` 虚拟接口或 `ingress` qdisc 配合 `netem` 完成。