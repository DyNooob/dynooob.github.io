---
layout: post
title: "VXLAN 实战指南：Overlay 网络原理与 Linux 配置"
date: 2026-08-16
categories: 网络技术
tags: [vxlan, overlay-network, linux, networking, virtualization, docker, kubernetes]
---

## 为什么需要 VXLAN？

传统 VLAN（802.1Q）用 12 位标记 VLAN ID，上限 4096 个。这在公有云、多租户环境下远远不够——一个云平台动辄数万租户，单靠 VLAN 根本分不开。更致命的是，VLAN 依赖物理网络拓扑，跨二层域通信必须走三层路由，而很多虚拟化场景（如虚拟机迁移、容器网络）需要维持一个大二层网络。

VXLAN（Virtual Extensible LAN）就是来解决这两个问题的：

- **24 位 VNI（VXLAN Network Identifier）**，支持 1600 万个隔离网络
- **基于 IP/UDP 的 Overlay 技术**，底层可以是任何 IP 网络，不再受物理拓扑限制

简单说，VXLAN 在现有 IP 网络之上叠加了一层虚拟二层网络，让两端的主机感觉像是连在同一个交换机上。

## 核心原理

### 封装格式

VXLAN 把原始二层帧（以太网头 + IP 载荷）封装在 UDP 报文中，结构如下：

```
+-------------------------------+
|  外层 MAC 头 (Outer Eth)       |
+-------------------------------+
|  外层 IP 头 (Outer IP)         |
+-------------------------------+
|  外层 UDP 头 (目标端口 4789)    |
+-------------------------------+
|  VXLAN 头 (8 字节)             |
|  - Flags (8 bit)               |
|  - Reserved (24 bit)           |
|  - VNI (24 bit)                |
|  - Reserved (8 bit)            |
+-------------------------------+
|  原始二层帧 (Inner Eth + Payload)|
+-------------------------------+
|  FCS (4 字节)                  |
+-------------------------------+
```

关键点：

- **UDP 目标端口 4789**（IANA 分配，旧实现用 8472）
- **VNI**：24 位，标识 VXLAN 段，类似 VLAN ID
- **外层 IP**：VTEP（VXLAN Tunnel Endpoint）之间的通信
- **内层帧**：原始二层帧完整保留，包括 MAC 地址

### VTEP

VTEP 是 VXLAN 的端点，负责做封装和解封装。每个 VTEP 有一个 IP 地址（通常是主机的物理网卡 IP或一个 Loopback IP），以及一个 MAC 地址。

当 VTEP 收到内层二层帧时：

1. 查看目标 MAC 地址
2. 查 VXLAN 转发表（FDB，Forwarding Database），找到目标 MAC 对应的远端 VTEP IP
3. 用 UDP 封装原始帧，发送到远端 VTEP
4. 远端 VTEP 解封装，把原始帧投到本地 VXLAN 段

### 学习模式

VXLAN 的 MAC 学习有两种方式：

**1. 泛洪学习（Flood & Learn）**

传统方式，类似物理交换机的 MAC 学习。当 VTEP 不知道目标 MAC 在哪个远端 VTEP 时，它会通过 IP 组播将包泛洪到所有 VTEP。收到响应的 VTEP 记录 MAC↔VTEP 映射。

依赖：底层网络必须支持 IP 组播（IGMP/PIM）或配置头端复制。

**2. 分布控制平面（EVPN）**

现代方式，用 BGP EVPN 协议分发 MAC/VNI 路由，不再依赖泛洪。这是数据中心 VXLAN 的主流方案，但需要支持 EVPN 的交换机。

## Linux 原生 VXLAN 配置

Linux 内核从 3.7 开始原生支持 VXLAN。无需安装任何额外软件——`ip link` 就能创建 VXLAN 接口。

### 基础环境准备

两台 CentOS/Ubuntu 主机，假设：

```
Host A: 192.168.1.10/24
Host B: 192.168.1.20/24
```

目标是让两台主机上的 VXLAN 网络互相通信，VNI 100。

### 单播模式（手动指定远端）

**Host A：**

```bash
# 创建 VXLAN 接口
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    local 192.168.1.10 \
    remote 192.168.1.20 \
    dev eth0

# 启用接口
ip link set vxlan100 up

# 分配 IP
ip addr add 10.0.0.1/24 dev vxlan100
```

**Host B：**

```bash
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    local 192.168.1.20 \
    remote 192.168.1.10 \
    dev eth0

ip link set vxlan100 up
ip addr add 10.0.0.2/24 dev vxlan100
```

验证：

```bash
ping -c 3 10.0.0.2  # 从 Host A 执行
```

如果通了，说明 VXLAN 隧道建立成功。

**限制**：`remote` 只能指定一个对端，适合点对点场景。多节点需要组播或头端复制。

### 组播模式（多节点自动学习）

三台主机，加入组播组 239.1.1.100：

**Host A (192.168.1.10)：**

```bash
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    group 239.1.1.100 \
    dev eth0

ip link set vxlan100 up
ip addr add 10.0.0.1/24 dev vxlan100
```

**Host B (192.168.1.20)：**

```bash
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    group 239.1.1.100 \
    dev eth0

ip link set vxlan100 up
ip addr add 10.0.0.2/24 dev vxlan100
```

**Host C (192.168.1.30)：**

```bash
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    group 239.1.1.100 \
    dev eth0

ip link set vxlan100 up
ip addr add 10.0.0.3/24 dev vxlan100
```

组播模式下，MAC 地址自动学习。任意两台主机第一次通信时，会先泛洪到整个组播组，收到响应后学习到对方的 MAC↔VTEP 映射。

**前提**：底层网络必须支持 IGMP Snooping 和组播路由，否则泛洪包会打到所有节点。

### 头端复制模式（Head-End Replication）

如果底层不支持组播，可以用 `remote` 参数多次指定，Linux 内核会负责在头端复制包并发送到每个远端：

```bash
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    local 192.168.1.10 \
    remote 192.168.1.20 \
    remote 192.168.1.30 \
    dev eth0
```

注意：Linux 的 `ip link` 目前只支持一个 `remote`。多远端可以用 `bridge fdb` 手动添加：

```bash
# 创建 VXLAN 接口（不指定远端）
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    local 192.168.1.10 \
    dev eth0

ip link set vxlan100 up
ip addr add 10.0.0.1/24 dev vxlan100

# 手动添加 FDB 条目
bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 192.168.1.20
bridge fdb append 00:00:00:00:00:00 dev vxlan100 dst 192.168.1.30
```

`00:00:00:00:00:00` 是通配 MAC，表示未知单播也发往这些远端（头端复制）。

## 调试与排障

### 查看 VXLAN 接口

```bash
ip -d link show vxlan100
```

### 查看 FDB 表

```bash
bridge fdb show dev vxlan100
```

输出示例：

```
00:00:00:00:00:00 dev vxlan100 dst 192.168.1.20 self permanent
00:00:00:00:00:00 dev vxlan100 dst 192.168.1.30 self permanent
2a:3b:4c:5d:6e:7f dev vxlan100 dst 192.168.1.20 self
```

- `permanent`：静态配置的条目
- 无标记：动态学习到的条目

### 抓包验证

```bash
# 抓外层 VXLAN 包
tcpdump -i eth0 udp port 4789 -v

# 抓内层包（在 VXLAN 接口上）
tcpdump -i vxlan100 -v
```

外层抓包可以看到 VXLAN 封装：

```
IP 192.168.1.10.4789 > 192.168.1.20.4789: VXLAN, flags [I] (0x08), vni 100
IP 10.0.0.1 > 10.0.0.2: ICMP echo request
```

### MTU 问题

VXLAN 头部额外占 50 字节（外层 IP 20 + UDP 8 + VXLAN 8 + 外层以太网 14）。如果物理网络的 MTU 是 1500，VXLAN 接口的 MTU 应该设为 1450。

```bash
ip link set vxlan100 mtu 1450
```

如果 Docker 或容器网络遇到"大包不通"的状况，先查 MTU。

## 实战：VXLAN + Linux Bridge 构建虚拟网络

把 VXLAN 接口桥接到 Linux Bridge 上，就能让多个虚拟设备（veth pair、tap 设备）接入同一个 VXLAN 段。

```bash
# 创建 bridge
ip link add br100 type bridge
ip link set br100 up

# 创建 VXLAN 接口（不指定 IP）
ip link add vxlan100 type vxlan \
    id 100 \
    dstport 4789 \
    local 192.168.1.10 \
    dev eth0

ip link set vxlan100 up

# 把 VXLAN 接口加入 bridge
ip link set vxlan100 master br100

# 给 bridge 分配 IP
ip addr add 10.0.0.1/24 dev br100
```

这种模式在 OpenStack Neutron 和 Docker Overlay 网络中广泛使用。VM/容器的虚拟网卡接入 bridge，bridge 把 VXLAN 接口作为上行端口，实现跨主机的二层互通。

## 与 Docker/Kubernetes 的关系

### Docker Overlay 网络

Docker 的 Overlay 网络驱动默认使用 VXLAN（端口 4789）实现跨主机容器通信。创建方式：

```bash
docker network create -d overlay \
    --subnet 10.10.0.0/16 \
    --opt encrypted \
    my-overlay-net
```

Docker 自动管理 VTEP 和 FDB 表，用户无需关心底层细节。

### Kubernetes CNI

Kubernetes 中，Flannel 的 VXLAN 后端、Calico 的 VXLAN 模式、Weave Net 等都使用 VXLAN 作为 Overlay 方案。以 Flannel 为例：

```bash
kubectl -n kube-flannel get pods
# 查看 Flannel 配置
kubectl -n kube-flannel get cm kube-flannel-cfg -o yaml
```

Flannel 的 VXLAN 模式下，每个节点上有一个 `flannel.1` 接口，类型为 VXLAN，负责 Pod 网络的跨节点通信。

## VXLAN vs VLAN

| 特性 | VLAN (802.1Q) | VXLAN |
|------|---------------|-------|
| 网络标识 | 12 位 (4096) | 24 位 (16M) |
| 传输层 | 二层（以太网） | 三层（IP/UDP） |
| 多路径 | 不支持（STP 阻塞） | 支持（ECMP） |
| 跨数据中心 | 困难 | 原生支持 |
| 内核支持 | 全部 | Linux 3.7+ |
| 封装开销 | 4 字节 | 50 字节 |
| 性能 | 原生转发 | 需封装/解封装 |

## 性能考量

VXLAN 的封装/解封装消耗 CPU。在纯软件 VTEP（如 Linux 内核）中，每包都需要做 UDP 封装，大流量场景下 CPU 开销明显。

优化手段：

1. **GSO/GRO Offload**：Linux 内核支持 VXLAN 的 GSO（Generic Segmentation Offload）和 GRO（Generic Receive Offload），让大包在发包前才分段，收包时先合并再交给上层。确认开启：

```bash
ethtool -k eth0 | grep tx-udp_tnl
# tx-udp_tnl-segmentation: on
```

2. **硬件 Offload**：支持 VXLAN Offload 的网卡（Mellanox/CX5+、Intel XL710+）可以在硬件层面完成封装/解封装，大幅降低 CPU 负载。

```bash
ethtool -k eth0 | grep vxlan
# tx-vxlan-vni-hw-offload: on
# rx-vxlan-vni-hw-offload: on
```

3. **RSS 多队列**：确保 VXLAN 流量的 RSS 均匀分布在多个 CPU 核心上。

## 总结

VXLAN 是现代网络虚拟化的基石技术。它用 IP/UDP 封装解决了 VLAN 的 4096 限制和物理拓扑依赖问题，让"大二层网络"在任意 IP 网络上成为可能。

在 Linux 上配置 VXLAN 出奇简单——一条 `ip link add type vxlan` 就能创建隧道。真正复杂的在于 FDB 管理、组播/头端复制策略、以及 MTU 规划。生产环境中，Kubernetes 和 OpenStack 已经把这些细节封装好了，但理解底层原理对排查网络问题至关重要。

下一层值得深入的是 EVPN——用 BGP 替代组播泛洪来分发 MAC/VNI 映射，是数据中心 VXLAN 的标配方案。如果读者对 EVPN 有兴趣，可以留言告诉我。