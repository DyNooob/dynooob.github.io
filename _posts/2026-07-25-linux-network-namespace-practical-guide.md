---
layout: post
title: "Linux 网络命名空间实战：从隔离到容器网络"
date: 2026-07-25 09:00:00 +0800
categories: [网络技术]
tags: [Linux, Network Namespace, 网络虚拟化, 容器, veth, 网桥, 网络隔离]
---

网络命名空间（Network Namespace）是 Linux 内核提供的最强大的网络虚拟化能力之一。Docker、Kubernetes 的容器网络模型，OpenStack 的虚拟网络，甚至家庭实验室的多租户隔离——底层都离不开它。

但很多人只把它当"黑盒"用。容器跑起来了，但容器里到底有个什么样的网络栈？两个容器之间怎么通信的？怎么让一个程序独享一个网络栈？

这篇文章从零开始，用命令一步步搭建网络命名空间，让你亲手摸清它的每一层。

## 一、什么是网络命名空间

Linux 命名空间（Namespace）是整个容器技术的基石。网络命名空间（`netns`）隔离的是网络栈——每个命名空间拥有独立的：

- 网络接口（lo、eth0 等）
- 路由表
- iptables / nftables 规则
- 网络栈参数（`/proc/sys/net/`）
- 套接字（socket）

默认情况下，整个系统运行在"根命名空间"（root namespace）中，也就是你平时用 `ip addr` 看到的所有网卡。创建新的命名空间后，里面除了一个 `lo` 回环接口，什么都没有——完全隔离。

### 一个简单的类比

想象一栋楼（宿主机），里面住着很多人（进程）。默认所有人共用一个大门（网络栈）。网络命名空间就是给每个房间装了一扇独立的门——房间里的人有自己的门、自己的门牌号、自己的访客名单，互不干扰。

## 二、创建和管理命名空间

### 2.1 基础操作

所有操作都通过 `ip netns` 命令完成：

```bash
# 创建两个命名空间
sudo ip netns add ns1
sudo ip netns add ns2

# 列出所有命名空间
ip netns list

# 在命名空间中执行命令
sudo ip netns exec ns1 ip addr
sudo ip netns exec ns2 hostname

# 进入命名空间的交互式 shell
sudo ip netns exec ns1 bash
# 退出用 exit

# 删除命名空间
sudo ip netns del ns1
```

刚创建出来的命名空间里只有 `lo` 接口，且默认是 down 状态：

```bash
sudo ip netns exec ns1 ip link set lo up
```

### 2.2 进程视图

一个进程一旦进入某个命名空间，它的所有网络操作都在该命名空间内进行。查看进程与命名空间的对应关系：

```bash
# 查看进程的 network namespace inode
ls -la /proc/$$/ns/net

# 查看所有命名空间的引用
sudo ls -la /run/netns/
```

`/run/netns/` 下每个文件对应一个命名空间，文件描述符就是命名空间的句柄。这意味着一件事：只要文件存在，命名空间就活着——即使里面没有进程。

## 三、veth 对：连接两个命名空间

隔离没有意义，能通信才有价值。veth（Virtual Ethernet）是一对虚拟网卡，像一根网线的两端——一端发数据，另一端收数据。

### 3.1 创建 veth 对并连接

```bash
# 创建 veth 对
sudo ip link add veth0 type veth peer name veth1

# 把两端分别塞进两个命名空间
sudo ip link set veth0 netns ns1
sudo ip link set veth1 netns ns2

# 配置 IP 并启用
sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth0
sudo ip netns exec ns1 ip link set veth0 up
sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec ns2 ip link set veth1 up

# 测试连通性
sudo ip netns exec ns1 ping -c 3 10.0.0.2
```

如果一切正常，你应该能看到 3 个成功的 ping 包。这就是最基础的容器间通信模型——Docker 的 `--net=container` 模式就是这个原理，但共用的是同一个命名空间而不是 veth 对。

### 3.2 验证隔离

```bash
# 在宿主机上看不到 veth0 和 veth1 了
ip addr show | grep veth

# 从宿主机 ping 不通
ping -c 1 10.0.0.1
```

veth 对移入命名空间后，宿主机的根命名空间就看不到这两张网卡了。以上 ping 应该返回 `Network is unreachable`。

## 四、网桥：让多个命名空间互通

veth 对只能连接两个点。要连接多个命名空间，需要网桥（Bridge）。

### 4.1 搭建网桥网络

```bash
# 创建网桥
sudo ip link add br0 type bridge
sudo ip link set br0 up

# 创建三个命名空间
for i in 1 2 3; do
    sudo ip netns add ns$i
    sudo ip link add veth-ns$i type veth peer name veth-br$i
    sudo ip link set veth-ns$i netns ns$i
    sudo ip link set veth-br$i master br0
    sudo ip link set veth-br$i up
    sudo ip netns exec ns$i ip addr add 10.0.0.10$i/24 dev veth-ns$i
    sudo ip netns exec ns$i ip link set veth-ns$i up
    sudo ip netns exec ns$i ip link set lo up
done

# 验证连通性
sudo ip netns exec ns1 ping -c 2 10.0.0.102
sudo ip netns exec ns2 ping -c 2 10.0.0.103
```

这个模式就是 Docker 默认的 `bridge` 网络模式。Docker 会在宿主机上创建一个 `docker0` 网桥，每个容器分配一个 veth 对，一端在容器里，另一端挂在 `docker0` 上。

### 4.2 查看网桥转发表

```bash
# 查看 FDB 转发表（MAC 地址学习）
bridge fdb show

# 查看网桥上的端口
bridge link show
```

网桥根据 MAC 地址学习转发，跟物理交换机一模一样。执行 `bridge fdb show` 能看到每个端口的 MAC 地址和所在接口。

## 五、让命名空间访问外网

目前命名空间只能内部通信，出不去。要让它们访问宿主机甚至外网，需要 NAT 加一条默认路由。

### 5.1 宿主机充当网关

```bash
# 在宿主机上给网桥配置 IP（充当网关）
sudo ip addr add 10.0.0.1/24 dev br0

# 给命名空间加默认路由
for i in 1 2 3; do
    sudo ip netns exec ns$i ip route add default via 10.0.0.1
done

# 测试：ping 宿主机 IP
sudo ip netns exec ns1 ping -c 2 10.0.0.1
```

### 5.2 开启 IP 转发和 NAT

```bash
# 开启 IP 转发（临时）
sudo sysctl -w net.ipv4.ip_forward=1

# 持久化
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf

# 配置 iptables NAT（MASQUERADE）
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# 测试外网联通
sudo ip netns exec ns1 ping -c 3 8.8.8.8
```

### 5.3 DNS 解析

命名空间里没有 `/etc/resolv.conf`，需要手动指定：

```bash
sudo ip netns exec ns1 bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
sudo ip netns exec ns1 ping -c 2 google.com
```

## 六、iptables 和命名空间

每个网络命名空间有自己独立的 iptables 规则链。这意味着你可以在不同命名空间应用不同的防火墙策略。

```bash
# 在 ns1 中禁止 ping 外部
sudo ip netns exec ns1 iptables -A OUTPUT -p icmp --icmp-type echo-request -j DROP

# 在 ns2 中只允许 80 端口出站
sudo ip netns exec ns2 iptables -A OUTPUT -p tcp --dport 80 -j ACCEPT
sudo ip netns exec ns2 iptables -A OUTPUT -j DROP

# 查看各自规则
sudo ip netns exec ns1 iptables -L -v
sudo ip netns exec ns2 iptables -L -v
```

宿主机根命名空间的 iptables 规则不会影响子命名空间，反之亦然。这是一个常被忽略却非常重要的隔离特性。

## 七、结合 tcpdump 抓包

在宿主机上抓不到命名空间内部的流量——因为 veth 对的内端在命名空间里。但你可以从宿主机侧抓 veth 的外端：

```bash
# 启动 tcpdump 监听 br0
sudo tcpdump -i br0 -n icmp

# 另一个终端发起 ping
sudo ip netns exec ns1 ping -c 3 10.0.0.102
```

在宿主机上能抓到经过网桥的 ICMP 包。如果要在命名空间内部抓包：

```bash
sudo ip netns exec ns1 tcpdump -i veth-ns1 -n
```

## 八、与 Docker 容器网络的对应关系

理解了命名空间，再看 Docker 网络就一目了然了：

| Docker 网络模式 | 底层实现 |
|---|---|
| `bridge`（默认） | 创建命名空间 + veth 对 + 宿主机网桥 |
| `host` | 不创建命名空间，直接使用宿主机网络栈 |
| `none` | 创建命名空间，只有 lo 接口 |
| `container:xxx` | 新容器加入已有容器的命名空间 |
| `macvlan` / `ipvlan` | 直接分配宿主机网络接口 |

```bash
# 查看 Docker 容器的命名空间
docker inspect --format '{{.NetworkSettings.SandboxKey}}' 容器名

# 进入容器命名空间（需要 root）
nsenter -t $(docker inspect -f '{{.State.Pid}}' 容器名) -n ip addr
```

`nsenter -n` 是进入已有命名空间最直接的方式，比 `docker exec` 更底层。

## 九、常见问题和排障

### 9.1 veth 对移入命名空间后找不到了

这是正常的。veth 对的一端在命名空间里，另一端在宿主机上。宿主机上的那一端如果不挂到网桥上，默认是 standalone 的，`ip addr` 不会显示 IP。用 `ip link show` 查看。

### 9.2 网桥上的命名空间互 ping 不通

检查几点：
1. 网桥是否 up：`ip link set br0 up`
2. veth 的宿主机端是否挂在网桥上：`bridge link show`
3. 命名空间内网卡是否 up：`sudo ip netns exec ns1 ip link show`
4. 命名空间内 IP 是否配置正确
5. 防火墙是否拦截：`sudo ip netns exec ns1 iptables -L`

### 9.3 ping 外网不通

最常见的三个原因：
- 宿主机 IP 转发没开：`sysctl net.ipv4.ip_forward`
- 缺少 MASQUERADE 规则：`iptables -t nat -L POSTROUTING`
- 命名空间缺少默认路由：`ip route show`

### 9.4 重启后命名空间丢失

`ip netns add` 创建的命名空间是临时的，重启后消失。持久化方案有两种：
- 使用 systemd 服务在启动时重建命名空间
- 使用 Docker 或 Podman 管理——它们会帮你打理好一切

## 十、总结

网络命名空间是 Linux 网络虚拟化的基石。理解它的核心模型，你就掌握了容器网络、虚拟化网络、甚至 SDN 的第一性原理。

本文涉及的所有命令都在一个标准的 Linux 系统上可运行（需要 `iproute2` 包，几乎所有发行版都预装）。建议你打开终端跟着敲一遍——从创建命名空间，到 veth 连接，到网桥组网，再到 NAT 出网。亲手搭建一遍比读十篇文档都管用。

下次再有人问你 Docker 的 `docker0` 网桥是怎么回事，你可以直接告诉他：不过是一个 Linux 网桥，挂了几对 veth 而已。