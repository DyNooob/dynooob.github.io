---
layout: post
title: "iptables/nftables 防火墙规则实战：从入门到生产配置"
date: 2026-08-08 09:00:00 +0800
categories: [网络技术, 安全]
tags: [iptables, nftables, 防火墙, Linux防火墙, 网络安全, 规则配置]
---

Linux 防火墙是每台服务器的第一道防线。iptables 和 nftables 是 Linux 上最主流的两个防火墙框架——前者是经典方案，后者是新一代替代品。本文从实战出发，覆盖基础规则编写、常见场景配置、以及从 iptables 迁移到 nftables 的要点。

## 一、理解 Netfilter 框架

无论是 iptables 还是 nftables，底层都基于 Netfilter 钩子。网络包在内核协议栈中经过五个关键钩子点：PREROUTING、INPUT、FORWARD、OUTPUT、POSTROUTING。

```
包入站 → PREROUTING → 路由决策 → INPUT → 本地进程
                              ↘ FORWARD → POSTROUTING → 出站
本地进程 → OUTPUT → 路由决策 → POSTROUTING → 出站
```

iptables 将规则组织成三张表（filter、nat、mangle），每张表有内置链对应上述钩子。nftables 则取消了这种硬编码的链-表层次，改用更灵活的命名集合和链。

## 二、iptables 基础规则

### 2.1 查看当前规则

```bash
# 查看 filter 表规则（默认表）
iptables -L -n -v

# 查看所有表
iptables -t nat -L -n -v
iptables -t mangle -L -n -v

# 查看规则编号（便于插入/删除）
iptables -L --line-numbers -n
```

`-n` 禁用 DNS 反向解析，避免卡顿。`-v` 显示匹配计数，调试时很有用。

### 2.2 常用规则模式

**允许 SSH 入站：**

```bash
iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

先放行信任网段，再拒绝其他来源。注意规则顺序——iptables 按顺序匹配，匹配即停止。

**允许已建立连接回包：**

```bash
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
```

这一条必须放在拒绝规则之前，否则你连自己的 SSH 回包都会丢。

**Web 服务器典型配置：**

```bash
# 允许回包
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 允许本地回环
iptables -A INPUT -i lo -j ACCEPT

# 开放 HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 允许特定管理网段 SSH
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# 拒绝所有其他入站
iptables -A INPUT -j DROP
```

**错误示范：** 先加 `-j DROP` 再加 `-j ACCEPT`，后面的规则永远不会被执行。

### 2.3 高级匹配

**连接数限制（防暴力破解）：**

```bash
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --set --name SSH
iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP
```

每分钟最多允许 3 次新 SSH 连接尝试，超过则丢弃。这比单纯换端口号有效得多。

**速率限制：**

```bash
iptables -A INPUT -p tcp --dport 80 -m limit --limit 1000/s \
  --limit-burst 2000 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j DROP
```

`--limit` 设置平均速率，`--limit-burst` 设置突发峰值。适合应对 CC 攻击。

**GeoIP 匹配（需要 xt_geoip 模块）：**

```bash
iptables -A INPUT -p tcp --dport 443 -m geoip --src-cc CN -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j DROP
```

仅允许中国 IP 访问 HTTPS。生产环境建议用 nftables 的 sets 配合 ipset 做更高效的 GeoIP 过滤。

## 三、nftables 实战

nftables 是 iptables 的官方替代方案，从 Debian 11、Ubuntu 22.04、RHEL 9 开始默认安装。最大的改进是：统一的语法、原子规则替换、更快的性能、更少的代码量。

### 3.1 基础配置

```bash
# 安装（通常已内置）
apt install nftables

# 启用并启动
systemctl enable --now nftables
```

### 3.2 规则集文件

nftables 的配置文件是 `/etc/nftables.conf`，一个完整的 Web 服务器配置示例：

```bash
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;

    # 回环放行
    iif lo accept

    # 已建立连接放行
    ct state established,related accept

    # ICMP（ping 和 PMTUD）
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept

    # SSH（仅内网）
    tcp dport 22 ip saddr { 10.0.0.0/8, 192.168.0.0/16 } accept

    # HTTP/HTTPS
    tcp dport { 80, 443 } accept

    # 记录并丢弃其他
    log prefix "nftables-drop: " limit rate 10/minute
    counter drop
  }

  chain forward {
    type filter hook forward priority 0; policy drop;
  }

  chain output {
    type filter hook output priority 0; policy accept;
  }
}
```

关键点说明：

- `inet` 地址族同时覆盖 IPv4 和 IPv6，不用写两套配置。
- `policy drop` 设置默认丢弃，比 iptables 的尾部 DROP 规则更高效。
- `ct state` 替代 iptables 的 `-m state --state`。
- `log limit rate 10/minute` 防止日志淹没磁盘。
- `counter` 记录匹配包数，调试时可观察。

### 3.3 常用规则

**NAT 转发（端口映射）：**

```bash
table inet nat {
  chain prerouting {
    type nat hook prerouting priority dstnat; policy accept;

    # 将 8022 端口映射到内网主机的 22 端口
    tcp dport 8022 dnat to 192.168.1.100:22
  }

  chain postrouting {
    type nat hook postrouting priority srcnat; policy accept;

    # MASQUERADE（动态 IP 的 SNAT）
    oif eth0 masquerade
  }
}
```

**流量限制（令牌桶）：**

```bash
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;

    # 每 IP 限制 100 个并发连接
    tcp dport 80 meter web-meter { ip saddr limit rate over 100/second burst 200 packets } drop

    tcp dport 80 accept
  }
}
```

nftables 的 meter 是内核级动态哈希表，比 iptables + recent 模块更高效，且自动老化过期条目。

### 3.4 动态规则：Sets 和 Maps

Sets 是 nftables 最强大的特性之一：

```bash
table inet filter {
  set blocked_ips {
    type ipv4_addr
    flags interval
    auto-merge
    elements = {
      1.1.1.0/24,
      2.2.2.2,
      10.0.0.0/8,
    }
  }

  chain input {
    type filter hook input priority 0; policy drop;
    ip saddr @blocked_ips drop
    # ...
  }
}
```

运行时添加 IP 无需重启防火墙：

```bash
nft add element inet filter blocked_ips { 203.0.113.5 }
nft delete element inet filter blocked_ips { 1.1.1.0/24 }
```

配合 fail2ban 或 ABRT 脚本，可以实现自动封禁。

## 四、iptables 到 nftables 迁移对照

| 操作 | iptables | nftables |
|------|----------|----------|
| 允许 SSH | `-A INPUT -p tcp --dport 22 -j ACCEPT` | `tcp dport 22 accept` |
| 状态跟踪 | `-m state --state ESTABLISHED` | `ct state established` |
| 多端口 | `-m multiport --dports 80,443` | `tcp dport {80,443}` |
| 速率限制 | `-m limit --limit 10/s` | `limit rate 10/second` |
| 连接数限制 | `-m connlimit --connlimit-above 100` | `ct count 100` |
| 日志 | `-j LOG --log-prefix "DROP"` | `log prefix "DROP"` |
| 封禁网段 | `-s 10.0.0.0/8 -j DROP` | `ip saddr 10.0.0.0/8 drop` |
| NAT | `-t nat -A PREROUTING -j DNAT` | `dnat to ...` |
| 规则保存 | `iptables-save > /etc/iptables/rules.v4` | 直接编辑 `/etc/nftables.conf` |
| 规则重载 | `iptables-restore < /etc/iptables/rules.v4` | `nft -f /etc/nftables.conf` |

## 五、生产环境注意事项

### 5.1 规则持久化

**iptables：**

```bash
# Debian/Ubuntu 安装 iptables-persistent
apt install iptables-persistent
iptables-save > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6

# 手动保存
netfilter-persistent save
```

**nftables：**

```bash
# 保存当前规则集到配置文件
nft list ruleset > /etc/nftables.conf

# 或直接编辑配置文件后重载
nft -f /etc/nftables.conf
```

### 5.2 安全操作守则

```bash
# 万一锁了自己，如何重置？
# 方法 1：如果还有 SSH 会话，直接清空规则
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -F
iptables -X

# nftables 重置
nft flush ruleset

# 方法 2：使用 cron 定时重置（应急方案）
# crontab -e
# */5 * * * * /usr/sbin/iptables -F
```

生产环境做远程防火墙变更时，**永远**先开一个备用 SSH 会话，或者用 `at` 定时任务做自动回滚：

```bash
# 5 分钟后自动恢复规则
echo "nft -f /etc/nftables.conf.backup" | at now + 5 minutes
# 然后应用新规则
nft -f /etc/nftables.conf.new
```

### 5.3 调试技巧

```bash
# 查看规则匹配计数
nft list ruleset
iptables -L -n -v

# 监控实时流量
nft monitor

# 跟踪特定包
nft add rule inet filter input tcp dport 443 log prefix "HTTPS: " accept

# 计数器清零
nft reset counters
```

`nft monitor` 在调试时特别有用，它能实时显示内核处理每个包的匹配路径。

## 六、总结

iptables 和 nftables 都是强大的 Linux 防火墙工具。如果你的服务器还在用 iptables，过渡到 nftables 并不复杂——核心概念一致，只是语法更简洁。建议新部署的系统直接使用 nftables，维护旧系统时逐步迁移。

记住三条原则：

1. **默认拒绝，按需放行**——白名单模型比黑名单安全得多。
2. **先测试，后持久化**——远程配置时一定要留后路。
3. **日志要限速**——不加限制的 LOG 规则能在 5 分钟内填满你的磁盘。

最后，防火墙不是万能的。它只是纵深防御体系中的一层，配合 fail2ban、SELinux、auditd 和入侵检测系统，才能构建真正的安全防线。