---
layout: post
title: "Linux Namespace 隔离原理与实践：手写一个迷你容器"
date: 2026-09-13 09:00:00 +0800
categories: [开发]
tags: [Linux, namespace, 容器, 隔离, unshare, nsenter, 内核, 虚拟化, Docker, 安全, 进程隔离]
---

## 1. 两个容器在同一台机器上，为什么彼此看不见？

这大概是我被问过最多的问题。初学者会以为容器是"轻量级虚拟机"，但真相更微妙：容器和宿主机**共享同一个内核**，却通过内核提供的 Namespace 机制，让每个进程拥有**独立的全局资源视图**。

来看一个最直观的例子：

```bash
# 宿主机上查看 PID 1
$ readlink /proc/1/ns/pid
pid:[4026531836]

# 启动一个 Docker 容器，在容器内查看
$ docker run --rm alpine readlink /proc/1/ns/pid
pid:[4026532394]
```

PID namespace 不同。容器里的 PID 1 不是宿主机上的 systemd，而是容器自己的 init 进程。它们看到的进程树完全隔离。

Namespace 和 Cgroups 是一体两面的关系：
- **Cgroups** 管资源——你能用多少 CPU、多少内存
- **Namespace** 管视野——你能看到什么、访问什么

本文带你深入八个 Namespace 类型，用 `unshare` 和 `nsenter` 手搓容器隔离，最后讨论安全风险和逃逸防范。

## 二、八大 Namespace 一览

截至 Linux 6.x，内核实现了 **8 种 Namespace**（Linux 5.6 引入 Time namespace）。每种 namespace 隔离一种全局资源：

| Namespace | 克隆标识 | 隔离内容 | 引入版本 |
|:--|:--|:--|:--|
| Mount | `CLONE_NEWNS` | 文件系统挂载点 | 2.4.19 |
| PID | `CLONE_NEWPID` | 进程 ID | 2.6.24 |
| Network | `CLONE_NEWNET` | 网络设备、IP、路由表 | 2.6.24 |
| IPC | `CLONE_NEWIPC` | System V IPC、POSIX 消息队列 | 2.6.19 |
| UTS | `CLONE_NEWUTS` | 主机名、域名 | 2.6.19 |
| User | `CLONE_NEWUSER` | 用户和组 ID | 3.8 |
| Cgroup | `CLONE_NEWCGROUP` | Cgroup 根目录 | 4.6 |
| Time | `CLONE_NEWTIME` | 系统时间（单调时钟/启动时间） | 5.6 |

`lsns` 命令可以列出当前系统的所有 namespace：

```bash
$ sudo lsns
        NS TYPE   NPROCS   PID  USER     COMMAND
4026531835 cgroup    132     1  root     /sbin/init
4026531836 pid       132     1  root     /sbin/init
4026531837 user      132     1  root     /sbin/init
4026531838 uts       131     1  root     /sbin/init
4026531839 ipc       132     1  root     /sbin/init
4026531840 mnt       128     1  root     /sbin/init
4026531992 net       132     1  root     /sbin/init
```

每个 namespace 在内核中用一个 inode 编号标识。`/proc/<pid>/ns/` 目录展示了进程所属的各个 namespace：

```bash
$ ls -la /proc/$$/ns/
total 0
dr-x--x--x 2 root root 0 Sep 13 00:00 .
dr-xr-xr-x 2 root root 0 Sep 13 00:00 ..
lrwxrwxrwx 1 root root 0 Sep 13 00:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 mnt -> 'mnt:[4026531840]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 uts -> 'uts:[4026531838]'
```

注意 `pid_for_children` 和 `time_for_children`——它们表示**子进程将进入的 namespace**，这个设计是为了处理 `clone()` 创建新 namespace 时的原子性。

## 三、unshare：单命令创建隔离环境

`unshare` 是用户空间创建 namespace 的门户。原理是调用 `unshare()` 系统调用让当前进程（或子进程）进入新的 namespace。

### 3.1 UTS Namespace：改主机名不污染宿主机

```bash
$ sudo unshare --uts /bin/bash
# 现在在新的 UTS namespace 里
hostname container-host
hostname
container-host

# 另开一个终端查看宿主机主机名
$ hostname
myhost  # 不受影响
```

这比改 `/etc/hostname` 重启安全一万倍。UTS namespace 隔离了 `utsname` 结构体——`uname()` 系统调用返回的结点名和域名。

### 3.2 PID Namespace：进程树隔离

```bash
$ sudo unshare --pid --fork /bin/bash
$ echo $$
1
$ ps aux
# 你会看到只有少数几个进程——因为 /proc 还没重新挂载
```

为什么要加 `--fork`？如果不 fork，当前 shell 直接进入新 PID namespace——但 shell 自己不是新 PID namespace 的**第一个进程**（PID 1），mount 和 pid 管理会出问题。`--fork` 让 unshare 先 fork 再进入，子进程成为新 PID namespace 的 init 进程。

注意 PID namespace 隔离后**必须重新挂载 `/proc`** 才能看到正确的进程列表：

```bash
$ sudo unshare --pid --fork --mount-proc /bin/bash
$ ps aux
USER       PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   8352   876 pts/0    S    00:00   0:00 /bin/bash
root         7  0.0  0.0  11484  3388 pts/0    R+   00:00   0:00 ps aux
```

PID 1 是 bash。你杀了它，整个 namespace 就被内核回收了。

### 3.3 Network Namespace：独立网络栈

```bash
# 创建独立 net namespace
$ sudo unshare --net /bin/bash
$ ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
$ ip addr
1: lo: <LOOPBACK> mtu 65536 ...
# 只有 lo 接口，且是 DOWN 状态
$ ip link set lo up
$ ip addr add 127.0.0.1/8 dev lo
# 现在有了本地回环，但没有 eth0，没有路由表之外的默认网关
```

Network namespace 隔离了：网络设备、IP 地址、路由表、防火墙规则（iptables/nftables）、socket 缓冲区、网络栈统计信息。**所有 socket 属于创建它的 net namespace**——这就是为什么容器内的监听端口不会暴露到宿主机。

跨 namespace 通信需要 **veth pair**：

```bash
# 宿主机创建 veth pair
$ sudo ip link add veth-host type veth peer name veth-container
$ sudo ip link set veth-container netns <container-pid>
$ sudo ip netns exec <ns-name> ip addr add 10.0.0.2/24 dev veth-container
$ sudo ip netns exec <ns-name> ip link set veth-container up
$ sudo ip link set veth-host up
```

`socket` 系统调用在 `CLONE_NEWNET` 的进程里创建时，内核将 socket 绑定到当前 namespace 的协议栈。这是最底层的网络隔离保障。

### 3.4 Mount Namespace：文件系统的障眼法

Mount namespace 隔离的是**挂载点视图**——同一块磁盘，在不同 mount namespace 中看到的内容可以完全不同。

```bash
$ sudo unshare --mount /bin/bash
$ mount -t tmpfs tmpfs /tmp
# 只有当前 namespace 能看到这个 tmpfs 挂载
# 宿主机 /tmp 不受影响
```

这是容器镜像的精髓：每个容器有自己的 mount namespace，overlay2 文件系统只在该 namespace 里可见。宿主机看到的是完整的根文件系统，容器看到的是 overlay 层叠后的内容。

注意 mount namespace 有**传播类型**（propagation type）：shared、slave、private、unbindable。默认是 shared，意味着 mount namespace 里的挂载事件会传播到其他 namespace。Docker 将容器根文件系统的挂载设为 private，防止容器内的挂载泄露到宿主机：

```bash
$ mount --make-private /  # 阻止挂载传播
```

## 四、nsenter：偷渡进别人的 namespace

`nsenter` 允许你以已有进程的身份进入其 namespace——俗称"偷渡"。这在排障容器内网络问题时极其有用。

```bash
# 找到容器进程的 PID
$ docker inspect <container> --format '{{.State.Pid}}'
12345

# 进入容器的 network namespace 执行 curl
$ sudo nsenter -t 12345 -n curl http://service.internal
# 容器内能访问的网络，你现在也能访问
```

还支持组合进入多个 namespace：

```bash
# 进入容器的 mount + pid + network namespace
$ sudo nsenter -t 12345 -m -p -n /bin/bash
$ ip addr       # 容器内的 IP
$ ps aux        # 容器内的进程
$ ls /proc/1/   # 容器内的 PID 1
```

这比 `docker exec` 更底层——你不需要 Docker daemon 配合，只要有进程 PID 和 root 权限就能进去。

实际排障场景：容器内网络不通，但 `docker exec` 进去后又没有 `tcpdump`。只需要宿主机上安装好工具，用 `nsenter -n -t <PID>` 进入容器的网络栈来抓包：

```bash
$ sudo nsenter -t 12345 -n tcpdump -i eth0 -w /tmp/capture.pcap
# 在宿主机就能看到容器内网卡的流量
```

## 五、手写迷你容器：从 unshare 到真正的隔离

把上面的知识串起来，用 shell 构建一个"迷你容器"——有**独立的文件系统、进程树、网络栈、主机名**。

### 5.1 准备工作：一个最小根文件系统

```bash
$ mkdir -p /tmp/miniroot
$ cd /tmp/miniroot
# 用 busybox 构建最小 rootfs
$ wget -O busybox https://busybox.net/downloads/binaries/latest/busybox-x86_64
$ chmod +x busybox
$ for bin in sh ls cat ps mount umount ip hostname; do
    ln -s /busybox $bin
  done
$ mkdir -p proc sys dev etc
```

### 5.2 启动迷你容器

```bash
# 全部屏蔽——同时隔离所有资源
$ sudo unshare \
    --mount \
    --pid \
    --uts \
    --ipc \
    --net \
    --fork \
    --mount-proc \
    --root=/tmp/miniroot \
    /bin/sh

# 现在你已经在迷你容器里
/ # hostname my-mini-container
/ # hostname
my-mini-container
/ # ps
PID  USER     TIME  COMMAND
  1 root      0:00 /bin/sh
  8 root      0:00 ps
/ # ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
/ # mount | grep rootfs
rootfs on / type rootfs (rw)
```

我们来拆解每个参数：

| 参数 | 作用 |
|:--|:--|
| `--mount` | 新 mount namespace，挂载操作不会影响宿主机 |
| `--pid` | 新 PID namespace，进程树完全隔离 |
| `--uts` | 新 UTS namespace，可独立设置主机名 |
| `--ipc` | 新 IPC namespace，防止共享内存/信号量跨边界 |
| `--net` | 新 network namespace，只有 lo 接口 |
| `--fork` | 先 fork 再进入，让子进程成为 PID 1 |
| `--mount-proc` | 自动挂载新的 /proc（否则 ps 不准确） |
| `--root=/tmp/miniroot` | 使用 chroot 切换根目录到最小 rootfs |

### 5.3 添加网络

```bash
# 在另一个终端（宿主机）创建 veth pair
$ sudo ip link add veth0 type veth peer name veth1
# 找到容器 PID
$ PID=$(cat /tmp/miniroot/var/run/container.pid)  # 假设容器内已创建 pid 文件
# 或者用 earlier 方法定位
$ pgrep -x sh | head -1  # 需要找对进程

# 将 veth1 移入容器 netns
$ sudo ip link set veth1 netns $PID
$ sudo ip netns exec /proc/$PID/ns/net ip addr add 10.0.0.1/24 dev veth1
$ sudo ip netns exec /proc/$PID/ns/net ip link set veth1 up
$ sudo ip link set veth0 up
$ sudo ip addr add 10.0.0.2/24 dev veth0
```

现在迷你容器内：

```bash
/ # ip addr show veth1
3: veth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc ...
    inet 10.0.0.1/24 scope global veth1
/ # ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: seq=0 ttl=64 time=0.054 ms
```

容器和宿主机通了。加 NAT 就能上网，加 veth pair 就能桥接——这正是 Docker 网络模式的最底层实现。

## 六、Docker 如何使用 Namespace

Docker（准确说是 containerd + runc）在创建容器时调用 `clone()` 并传入相应的 `CLONE_NEW*` 标志。来看一个运行中的容器的 namespace 归属：

```bash
$ docker run -d --name demo nginx:alpine
$ PID=$(docker inspect demo --format '{{.State.Pid}}')
$ sudo ls -la /proc/$PID/ns/
total 0
lrwxrwxrwx 1 root root 0 Sep 13 00:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Sep 13 00:00 ipc -> 'ipc:[4026532993]'    # 新 IPC
lrwxrwxrwx 1 root root 0 Sep 13 00:00 mnt -> 'mnt:[4026532991]'    # 新 Mount
lrwxrwxrwx 1 root root 0 Sep 13 00:00 net -> 'net:[4026532994]'    # 新 Net
lrwxrwxrwx 1 root root 0 Sep 13 00:00 pid -> 'pid:[4026532992]'    # 新 PID
lrwxrwxrwx 1 root root 0 Sep 13 00:00 user -> 'user:[4026531837]'  # 共享宿主机 User
lrwxrwxrwx 1 root root 0 Sep 13 00:00 uts -> 'uts:[4026532990]'    # 新 UTS
```

**默认配置下，Docker 为容器创建独立的 mnt、pid、net、ipc、uts namespace，但保留在宿主机的 user 和 cgroup namespace 中。**

User namespace 需要显式开启：

```bash
$ dockerd --userns-remap=default
```

启用后，容器内的 root（UID 0）会被映射到宿主机上的非特权用户（通常是 `dockremap:65534`）。这意味着即使容器被攻破拿到 root，在宿主机视角也只是普通用户权限。这个映射在 `/proc/<pid>/uid_map` 中可见：

```bash
$ cat /proc/$PID/uid_map
         0     165536      65536
```

含义：容器内 UID 0~65535 映射到宿主机 UID 165536~231071。

## 七、Namespace 逃逸：隔离失效的极端情况

Namespace 提供了强大的隔离，但**它不是牢不可破的**。以下是我整理的真实逃逸场景：

### 7.1 Mount Namespace 逃逸——挂载宿主机 proc

如果容器内有 `CAP_SYS_ADMIN` 能力和宿主机 `/proc` 的访问权限：

```bash
# 在容器内，如果有 CAP_SYS_ADMIN
$ mount -t proc proc /host-proc
$ cat /host-proc/1/cmdline
/sbin/init
```

这就是为什么生产容器**必须去掉** `CAP_SYS_ADMIN`：

```bash
$ docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

### 7.2 通过宿主机空间泄露

- `--pid=host` 让容器看到宿主机所有进程——PID 1 的 ns 就是全局 ns
- `--net=host` 让容器使用宿主机网络栈——iptables 规则可以在容器内修改
- `--privileged` 直接禁用所有 namespace 隔离

这些是**有意为之**的配置，但生产中应当极度谨慎使用。

### 7.3 Kernel 漏洞引起的 namespace 逃逸

历史上 `CVE-2022-0492`（cgroup v1 release_agent）允许容器内的 `CAP_SYS_ADMIN` 进程在宿主机执行命令。`CVE-2024-21626`（runc 路径遍历）让容器进程逃逸到宿主机文件系统。

这些漏洞的共同点：**namespace 隔离的弱点不在 namespace 自身，而在内核暴露给 namespace 内进程的接口**。保持内核更新是 namespace 安全的底线。

## 八、调试 Namespace 的实用命令

```bash
# 查看所有 namespace 及其关联的进程数
lsns

# 查看特定进程的 namespace 编号
ls -la /proc/<PID>/ns/

# 比较两个进程是否在同一个 namespace（inode 相同则同 namespace）
stat -c "%i" /proc/<PID>/ns/pid
stat -c "%i" /proc/<PID2>/ns/pid

# 进入某个 namespace 调试
nsenter -t <PID> -n ip addr

# 创建持久化的 network namespace（ip netns 封装）
ip netns add test-ns
ip netns exec test-ns ip link
```

组合使用 `strace` 看 namespace 系统调用：

```bash
$ strace -e clone,unshare docker run alpine echo hi
```

你会看到 `clone(child_stack=NULL, flags=CLONE_NEWNS|CLONE_NEWPID|...)` — 这是 Docker 创建容器的底层证据。

## 九、总结

| 层面 | 知识点 |
|:--|:--|
| 隔离维度 | 8 种 namespace 覆盖文件系统、进程、网络、主机名、IPC、用户、cgroup、时间 |
| 工具 | `unshare` 创建隔离、`nsenter` 进入隔离、`lsns` 查看、`ip netns` 管理网络 |
| 容器实现 | Docker 默认启用 5 种 namespace，User namespace 需手动开启 |
| 安全基线 | 去掉 `CAP_SYS_ADMIN`、避免 `--privileged`、开启 User namespace 映射、及时更新内核 |
| 排障技巧 | `nsenter -t <PID> -n` 进入容器网络抓包、`ls -la /proc/<PID>/ns/` 确认隔离状态 |

Namespace 是 Linux 容器技术的基石。理解了它，你就不再是把容器当黑盒在用——每一层隔离的取舍、每一个 `--cap-drop` 的意义，都会变得清晰。下次遇到容器逃逸漏洞或网络不通的问题，先从 namespace 入手看隔离边界设得对不对。

当你在排障时多问一句"这个进程在哪个 namespace 里？"，你就已经超越了 90% 的运维人员。