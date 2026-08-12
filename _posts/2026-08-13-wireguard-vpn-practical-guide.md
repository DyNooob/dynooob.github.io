---
layout: post
title: "WireGuard VPN 实战指南：从部署到调优"
date: 2026-08-13 08:00:00 +0800
categories: 网络技术
tags: WireGuard VPN Linux 网络隧道 加密通信
---

## 为什么选择 WireGuard

WireGuard 是近年来最受关注的 VPN 方案之一，2016 年由 Jason A. Donenfeld 发起，2020 年正式合入 Linux 5.6 内核。它的设计哲学是"少即是多"——代码量只有 OpenVPN 的百分之一左右，却提供了更高的安全性、更低的延迟和更简洁的配置。

### 与 OpenVPN / IPsec 的对比

| 特性 | WireGuard | OpenVPN | IPsec (strongSwan) |
|------|-----------|---------|---------------------|
| 内核集成 | Linux 5.6+ 内置 | 需 tun/tap + 用户态 | 需内核模块或用户态 |
| 代码量 | ~4,000 行 | ~600,000 行 | ~400,000 行 |
| 握手延迟 | ~1 RTT | ~3-5 RTT | ~3-6 RTT |
| 配置复杂度 | 一对密钥对 | CA + 证书 + 参数 | 多阶段 + 证书 |
| 加密套件 | 固定：ChaCha20+Poly1305+Curve25519+HKDF+BLAKE2 | 可配置，易选错 | 可配置，易选错 |
| 连接保持 | 静默长连接，无心跳 | 需 keepalive 配置 | 需 DPD 配置 |
| 漫游支持 | 原生支持 | 需额外配置 | 有限支持 |

WireGuard 核心优势在于：**加密套件固定**（不存在配置错误导致降级攻击的风险）、**代码量极小**（审计难度低）、**连接无状态**（服务端不保存连接状态，天生抗 DDoS）。

## 安装 WireGuard

### Linux

主流发行版均已内置 WireGuard 内核模块（Linux 5.6+），只需安装用户态工具：

```bash
# Debian / Ubuntu
apt update && apt install wireguard-tools

# RHEL / CentOS 8+ / Rocky Linux
dnf install wireguard-tools

# Arch Linux
pacman -S wireguard-tools

# Alpine
apk add wireguard-tools
```

确认安装成功：

```bash
wg --version
# 输出: wireguard-tools v1.0.20210914 - go1.22.0 等
```

### macOS

```bash
brew install wireguard-tools
```

### Windows

从 [wireguard.com/install](https://www.wireguard.com/install/) 下载安装包，自带 GUI 和命令行工具。

## 基础配置：点对点隧道

我们先搭建一个最简单的点对点场景：两台 Linux 服务器之间建立 VPN 隧道。

### 生成密钥对

每台机器都需要自己的密钥对：

```bash
# 生成私钥
wg genkey | tee private.key
# 输出: uD9n4V5k3x...（私钥，不可泄露）

# 从私钥派生公钥
cat private.key | wg pubkey > public.key
# 输出: 7T8pF2qR...（公钥，分发给对端）
```

### 服务端配置（Server A）

创建 `/etc/wireguard/wg0.conf`：

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <Server A 的私钥>

[Peer]
PublicKey = <Client B 的公钥>
AllowedIPs = 10.0.0.2/32
```

### 客户端配置（Client B）

```ini
[Interface]
Address = 10.0.0.2/24
PrivateKey = <Client B 的私钥>

[Peer]
PublicKey = <Server A 的公钥>
Endpoint = <Server A 的公网IP>:51820
AllowedIPs = 10.0.0.0/24
PersistentKeepalive = 25
```

### 启动隧道

```bash
# 启动
wg-quick up wg0

# 查看状态
wg show

# 测试连通性
ping 10.0.0.1
```

`wg show` 输出示例：

```
interface: wg0
  public key: 7T8pF2qR...
  private key: (hidden)
  listening port: 51820

peer: uD9n4V5k...
  endpoint: 203.0.113.5:51820
  allowed ips: 10.0.0.2/32
  latest handshake: 1 minute, 30 seconds ago
  transfer: 1.42 KiB received, 2.35 KiB sent
```

## 全流量路由（VPN 网关模式）

点对点只能访问对方内部网络。要让客户端**所有流量都走 VPN**（隐私保护、绕过限制、远程办公），需要配置 NAT 转发。

### 服务端开启 IP 转发

```bash
# 临时生效
sysctl -w net.ipv4.ip_forward=1

# 永久生效
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

### 配置 iptables NAT

```bash
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
iptables -A FORWARD -s 10.0.0.0/24 -j ACCEPT
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
```

持久化规则（Debian/Ubuntu 使用 `iptables-persistent`）：

```bash
apt install iptables-persistent
netfilter-persistent save
```

### 客户端配置文件修改

```ini
[Interface]
Address = 10.0.0.2/24
PrivateKey = <Client B 的私钥>
DNS = 1.1.1.1  # 可选：使用 VPN 的 DNS

[Peer]
PublicKey = <Server A 的公钥>
Endpoint = <Server A 的公网IP>:51820
AllowedIPs = 0.0.0.0/0, ::/0    # 所有流量走 VPN
PersistentKeepalive = 25
```

关键区别：`AllowedIPs = 0.0.0.0/0, ::/0` 告诉 WireGuard 将所有流量路由到对端。

## 多客户端接入

WireGuard 没有"用户管理"的概念——每个对等端只是一个 `[Peer]` 块。添加多个客户端只需在服务端追加：

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <Server A 的私钥>
PostUp = iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

[Peer]
# 客户端 B
PublicKey = <Client B 的公钥>
AllowedIPs = 10.0.0.2/32

[Peer]
# 客户端 C
PublicKey = <Client C 的公钥>
AllowedIPs = 10.0.0.3/32

[Peer]
# 客户端 D（移动设备，不固定 IP）
PublicKey = <Client D 的公钥>
AllowedIPs = 10.0.0.4/32
```

注意 `PostUp`/`PostDown` 指令——它们在隧道启动/停止时自动执行 iptables 规则，比手动管理更方便。

## 性能调优

### MTU 优化

WireGuard 默认 MTU 为 1420 字节（以太网 1500 - 20 IP头部 - 8 UDP头部 - 32 WireGuard 封装开销 - 20 备用）。在某些网络环境下（尤其是 PPPoE、GRE 隧道、移动网络），需要调小：

```ini
[Interface]
Address = 10.0.0.1/24
MTU = 1280
PrivateKey = ...
```

1280 是 IPv6 要求的最小 MTU，兼容性最好。如果环境允许，可以尝试 1420 → 1440 逐步调大，用 `ping -M do -s <size>` 测试：

```bash
# 测试 MTU（size = 1472 对应 1500 以太网 MTU）
ping -M do -s 1472 -c 3 10.0.0.1
# 如果 Frag needed 报错，减小 size
```

### 多队列优化

多核系统上启用多队列可以提升吞吐：

```bash
ethtool -L eth0 combined 4  # 先确保物理网卡开启多队列
```

WireGuard 内核模块从 5.6 起支持多队列，无需额外配置。如果使用 `wg-quick`，它会在 `$WG_QUICK_USERSPACE_IMPLEMENTATION` 环境变量指定时使用用户态实现（如 boringtun），此时多队列需要手动配置 ring buffer。

### 内核参数调优

```bash
# 增大 UDP 缓冲区
sysctl -w net.core.rmem_max=26214400
sysctl -w net.core.wmem_max=26214400

# BBR 拥塞控制（TCP 性能优化）
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr

# 加快连接复用
sysctl -w net.ipv4.tcp_tw_reuse=1
```

## 日志与排障

### 开启 WireGuard 日志

```bash
# 查看内核模块日志
dmesg | grep wireguard

# 实时跟踪
dmesg -w | grep wireguard
```

### 常见问题

**1. 握手不成功**

```
Handshake for peer did not complete after 5 seconds
```

检查项：
- 服务端防火墙是否放行 UDP 51820 端口
- 客户端 `Endpoint` 地址是否正确（公网 IP + 端口）
- 密钥对是否匹配（公钥分发给对端，私钥自己保留）
- 客户端是否在 NAT 后未设置 `PersistentKeepalive`

**2. 能 ping 通但无法访问网络**

- 检查服务端 `net.ipv4.ip_forward` 是否开启
- 检查 iptables MASQUERADE 规则是否生效
- 检查客户端 `AllowedIPs` 是否包含 `0.0.0.0/0`
- 检查 DNS 是否配置（`/etc/resolv.conf` 或 `[Interface] DNS =`）

**3. 连接不稳定，频繁断开**

- 移动网络下设置 `MTU = 1280` 避免分片
- 增大 `PersistentKeepalive` 间隔（默认 25 秒，NAT 超时短的可设 15 秒）
- 检查服务端是否达到了 `net.core.rmem_max` 上限

## 用 systemd 管理 WireGuard

`wg-quick` 生成的 systemd 服务让管理更规范：

```bash
# 开机自启
systemctl enable wg-quick@wg0

# 启动
systemctl start wg-quick@wg0

# 查看状态
systemctl status wg-quick@wg0

# 重启
systemctl restart wg-quick@wg0
```

## 安全注意事项

1. **私钥保护好**：`/etc/wireguard/` 目录默认只有 root 可读。如果误泄露，立即换密钥。
2. **监听端口隐藏**：WireGuard 不响应未认证的数据包，端口扫描只能看到 `closed` 或 `open|filtered`，不会泄露服务信息。
3. **禁用 PreSharedKey 除非必要**：WireGuard 支持额外的对称密钥层（`PresharedKey`），但在大多数场景下 Curve25519 已经足够。多一层密钥意味着多一层管理风险。
4. **定期轮换密钥**：自动化脚本每月生成新密钥对，更新配置后重启服务。
5. **防火墙白名单**：即使 WireGuard 本身安全，也建议在 `iptables` 层面限制 `INPUT` 链，只允许已知 IP 段访问 51820 端口（如果客户端 IP 固定）。

## 实战场景：远程办公网络

一个典型的小团队远程办公网络架构：

```
[员工 A 笔记本] ───┐
[员工 B 手机]    ───┤─── [云端 WireGuard 服务器] ─── [内网服务]
[员工 C 台式机]  ───┘      10.0.0.1/24             192.168.1.0/24
```

服务端配置示例：

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <server-private>
# 自动 NAT 转发到内网
PostUp = iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -d 192.168.1.0/24 -j MASQUERADE
PostUp = iptables -A FORWARD -s 10.0.0.0/24 -d 192.168.1.0/24 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -d 192.168.1.0/24 -j MASQUERADE
PostDown = iptables -A FORWARD -s 10.0.0.0/24 -d 192.168.1.0/24 -j ACCEPT

[Peer]
# 员工 A
PublicKey = <A-public>
AllowedIPs = 10.0.0.2/32

[Peer]
# 员工 B
PublicKey = <B-public>
AllowedIPs = 10.0.0.3/32

[Peer]
# 员工 C
PublicKey = <C-public>
AllowedIPs = 10.0.0.4/32
```

客户端只需配置 `Endpoint` 指向服务器公网 IP，`AllowedIPs` 设置为 `192.168.1.0/24`（仅内网流量走 VPN）或 `0.0.0.0/0`（全流量）。

## 总结

WireGuard 用极简的设计解决了 VPN 的核心问题：加密传输。它的优势不仅在于性能（实测在同等硬件条件下比 OpenVPN 快 2-4 倍），更在于配置的确定性——几乎不可能因为配置错误而降低安全性。

从单机隧道到多客户端远程办公网络，WireGuard 都能胜任。配合 `wg-quick` 和 systemd，管理开销几乎为零。对于有网络隔离需求的团队，它是最值得尝试的现代 VPN 方案。

下一步可以探索 WireGuard 与 Docker 结合、Kubernetes 跨集群互连（`submariner` 或 `kilo`），以及 `wg-dynamic` 自动发现对等端等进阶场景。