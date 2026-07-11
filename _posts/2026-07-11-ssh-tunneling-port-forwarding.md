---
layout: post
title: "SSH 隧道与端口转发：三种模式的使用场景"
date: 2026-07-11 15:00:00 +0800
categories: [安全, 网络技术]
tags: [SSH, 隧道, 端口转发, 内网穿透, 代理]
---

做网络安全或者后端开发，SSH 隧道是绕不开的基础技能。无论是调试内网服务、绕过防火墙限制，还是搭建代理通道，SSH 的端口转发功能都能派上用场。

SSH 端口转发本质上是在一条 SSH 连接上复用多个数据流。有三种模式，各自对应不同的场景。

## 本地转发（-L）

最常见的用法。把本地某个端口收到的流量，通过 SSH 服务器转发到目标地址。

```
ssh -L 本地端口:目标地址:目标端口 用户@跳板机
```

**典型场景**：你本地跑了一个服务，想访问公司内网的数据库。

```bash
# 把本地的 3307 端口映射到内网数据库的 3306
ssh -L 3307:db-internal.company.com:3306 user@bastion.company.com
```

连接后，`mysql -h 127.0.0.1 -P 3307` 就是在操作内网数据库。

## 远程转发（-R）

反过来。把远程服务器收到的流量，转发到本地。

```
ssh -R 远程端口:本地地址:本地端口 用户@公网服务器
```

**典型场景**：你在一个没有公网 IP 的开发机上跑了一个 Web 服务，想让别人访问。

```bash
# 在开发机上执行：把公网服务器的 8080 映射到本地的 3000
ssh -R 8080:localhost:3000 user@public-server.com
```

别人访问 `public-server.com:8080`，流量会经过 SSH 隧道到达你本地的 3000 端口。

这个模式的关键是公网服务器需要开启 `GatewayPorts yes`，否则默认只绑定到 127.0.0.1。

## 动态转发（-D）

最灵活的模式。在本地启动一个 SOCKS5 代理，所有通过代理的流量都由远程服务器转发。

```
ssh -D 本地端口 用户@跳板机
```

**典型场景**：你需要一个加密的代理通道，或者绕过网络限制。

```bash
# 在本地 1080 端口启动 SOCKS5 代理
ssh -D 1080 user@vps.example.com
```

配置浏览器或系统代理为 `SOCKS5 127.0.0.1:1080`，所有流量都走远程服务器出口。

## 实用技巧

### 保持连接存活

SSH 默认长时间没数据就会断开。加上这些参数保持连接：

```bash
ssh -o ServerAliveInterval=60 -o ServerAliveCountMax=3 -L ...
```

### 后台运行

```bash
ssh -f -N -L 3307:localhost:3306 user@bastion
```

- `-f`：后台运行
- `-N`：不执行远程命令，只做端口转发

### 配置文件管理复杂隧道

在 `~/.ssh/config` 里定义：

```
Host bastion
    HostName bastion.company.com
    User jumper
    ServerAliveInterval 60
    LocalForward 3307 db-internal:3306
    LocalForward 8080 internal-web:80
```

然后直接 `ssh bastion` 就能建立所有隧道。

### 多级跳转（-J）

通过多台机器串联：

```bash
ssh -J user@bastion1,user@bastion2 -L 3307:db:3306 user@target
```

## 注意事项

- SSH 隧道是加密的，但速度受限于两端带宽和 SSH 的加密开销
- 长时间大量数据传输，考虑用更轻量的方案（如 WireGuard）
- 远程转发（-R）有安全风险——不要随意开放端口到公网
- 生产环境建议用 autossh 自动重连

这些操作在 CTF 比赛、渗透测试、日常开发调试中都很常用。理解了三种模式的区别，遇到具体场景就知道该用哪个了。