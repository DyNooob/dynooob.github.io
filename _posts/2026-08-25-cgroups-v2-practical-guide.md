---
layout: post
title: "Cgroups v2 资源管理实战：从 systemd 到容器"
date: 2026-08-25 09:00:00 +0800
categories: [开发]
tags: [cgroups, linux, systemd, containers, resource-management, memory, cpu, io, kernel, docker, kubernetes, performance]
---

## 1. 为什么 Cgroups v2 值得关注

Cgroups（Control Groups）是 Linux 内核的资源隔离机制，几乎所有容器化、资源限制场景都依赖它。Cgroups v2 在 Linux 4.5 引入并逐步完善，到 5.2 之后成为主流内核的默认配置，但很多开发者仍然在用 v1 的思维操作 v2，踩坑不断。

Cgroups v2 和 v1 最大的区别在于：**统一层级树**。v1 为每个资源控制器（CPU、内存、IO 等）挂载独立的树，导致复杂的依赖管理和不一致的行为。v2 将所有控制器放在同一个层级树中，一个进程只属于一个 cgroup，不再有"一个进程横跨多个不同树"的混乱局面。

这里有一个你在生产环境一定会遇到的实际差异：当 `/sys/fs/cgroup` 下只有 `cgroup.controllers` 一个文件而没有独立子目录时，说明你的系统正在运行 cgroups v2。确认方法很简单：

```bash
$ stat -fc %T /sys/fs/cgroup/
cgroup2fs
```

如果输出是 `cgroup2fs`，就是 v2；如果是 `tmpfs`，则是 v1。

## 2. 核心概念与架构

### 2.1 统一层级树

在 v2 中，所有控制器都挂载在 `/sys/fs/cgroup/` 下。每个子目录就是一个 cgroup 实例，控制器通过 `cgroup.controllers` 和 `cgroup.subtree_control` 两个文件逐级激活。

```
/sys/fs/cgroup/
├── cgroup.controllers      # 当前可用的控制器列表
├── cgroup.subtree_control  # 子 cgroup 可继承的控制器
├── system.slice/           # systemd 管理的系统服务
├── user.slice/             # 用户会话
├── docker/                 # Docker 容器（如果使用 cgroupfs 驱动）
└── ...
```

### 2.2 控制器激活机制

v2 的控制器是**按需激活**的。根节点默认不启用任何控制器，需要逐层向下传递：

```bash
# 查看当前根节点可用的控制器
$ cat /sys/fs/cgroup/cgroup.controllers
cpuset cpu io memory hugetlb pids rdma misc

# 开启 memory 和 cpu 控制器
$ echo "+memory +cpu" > /sys/fs/cgroup/cgroup.subtree_control
```

这个设计的核心思想是：**父节点决定子节点能使用哪些控制器**，子节点不能超越父节点授予的范围。

### 2.3 进程管理

v2 中，每个进程只能属于一个 cgroup，通过 `cgroup.procs` 和 `cgroup.threads` 管理：

```bash
# 将当前 shell 进程加入一个 cgroup
$ mkdir -p /sys/fs/cgroup/my-app
$ echo $$ > /sys/fs/cgroup/my-app/cgroup.procs
```

## 3. 内存控制器实战

### 3.1 限制内存使用

这是最常用的场景——防止某个进程耗尽系统内存：

```bash
$ mkdir -p /sys/fs/cgroup/limit-mem
$ echo "+memory" > /sys/fs/cgroup/cgroup.subtree_control

# 设置 256MB 硬限制
$ echo 268435456 > /sys/fs/cgroup/limit-mem/memory.max
# 设置 128MB 软限制（超过后可能被回收但不会被 OOM kill）
$ echo 134217728 > /sys/fs/cgroup/limit-mem/memory.high

# 关联进程
$ echo $PID > /sys/fs/cgroup/limit-mem/cgroup.procs
```

关键文件说明：

| 文件 | 含义 |
|------|------|
| `memory.current` | 当前内存用量（字节） |
| `memory.min` | 最低保障内存（硬性预留） |
| `memory.low` | 低优先级内存（软性预留） |
| `memory.high` | 内存软限制（超过后触发回收） |
| `memory.max` | 内存硬限制（超过触发 OOM） |
| `memory.swap.max` | swap 上限（设为 0 禁用 swap） |

### 3.2 内存压力通知（PSI）

v2 引入了 Pressure Stall Information（PSI），可以精确监控资源压力。这是 v1 完全没有的能力：

```bash
$ cat /sys/fs/cgroup/limit-mem/memory.pressure
some avg10=0.00 avg60=0.00 avg300=0.00 total=0
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

`some` 表示至少有一个任务在等待内存，`full` 表示所有任务都在等待。配合 `systemd.resource-control` 可以实现主动内存压力响应：

```ini
# /etc/systemd/system/my-service.service.d/override.conf
[Service]
MemoryHigh=512M
MemoryMax=1G
```

### 3.3 OOM 控制

v2 的 OOM 行为更精细。`memory.oom.group` 可以控制是否杀死整个 cgroup 的所有进程：

```bash
# 默认 0：只杀死触发 OOM 的单个进程
# 设为 1：杀死整个 cgroup 的所有进程
$ echo 1 > /sys/fs/cgroup/limit-mem/memory.oom.group
```

这在处理容器场景时特别有用——一个容器内的进程 OOM 时，你可能希望整个 Pod 重启而不是留下一个半死不活的状态。

## 4. CPU 控制器实战

### 4.1 CPU 权重分配

v2 的 CPU 控制器使用**权重**（weight）而非 v1 的 `cpu.shares`。权重范围 1-10000，默认 100：

```bash
$ mkdir -p /sys/fs/cgroup/cpu-high
$ echo "+cpu" > /sys/fs/cgroup/cgroup.subtree_control
$ echo 1000 > /sys/fs/cgroup/cpu-high/cpu.weight
$ echo $PID > /sys/fs/cgroup/cpu-high/cgroup.procs
```

权重 1000 的进程比权重 100 的进程获得大约 10 倍的 CPU 时间（在竞争时）。注意：**权重只在 CPU 竞争时生效**，如果有空闲核心，权重没有意义。

### 4.2 CPU 硬限制

使用 `cpu.max` 设置固定 CPU 上限：

```bash
# 格式：quota period
# 限制为 2 个核心（每个 period 100ms 内最多 200ms）
$ echo "200000 100000" > /sys/fs/cgroup/cpu-high/cpu.max

# 解除限制
$ echo "max 100000" > /sys/fs/cgroup/cpu-high/cpu.max
```

### 4.3 CPU 亲和性（cpuset）

v2 将 cpuset 集成到统一层级中，不再需要单独挂载：

```bash
# 限制使用 CPU 0-1
$ echo "0-1" > /sys/fs/cgroup/cpu-high/cpuset.cpus
# 限制使用 NUMA 节点 0
$ echo "0" > /sys/fs/cgroup/cpu-high/cpuset.mems
```

这在 NUMA 架构的服务器上效果显著——避免跨 NUMA 节点访问内存，可以降低 10-30% 的内存延迟。

## 5. IO 控制器实战

### 5.1 IO 权重限制

和 CPU 类似，IO 控制器使用权重模型：

```bash
$ mkdir -p /sys/fs/cgroup/io-low
$ echo "+io" > /sys/fs/cgroup/cgroup.subtree_control

# 设置 IO 权重（范围 1-10000，默认 100）
$ echo "253:0 100" > /sys/fs/cgroup/io-low/io.weight
# 253:0 是设备号（major:minor），可以用 lsblk 查看
```

### 5.2 IO 带宽和 IOPS 硬限制

```bash
# 限制读带宽为 10MB/s
$ echo "253:0 rbps=10485760" > /sys/fs/cgroup/io-low/io.max
# 限制写带宽为 5MB/s
$ echo "253:0 wbps=5242880" > /sys/fs/cgroup/io-low/io.max
# 限制读 IOPS 为 1000
$ echo "253:0 riops=1000" > /sys/fs/cgroup/io-low/io.max
# 限制写 IOPS 为 500
$ echo "253:0 wiops=500" > /sys/fs/cgroup/io-low/io.max
```

注意：IO 限制只在 cgroups v2 内核写回（writeback）机制下工作。如果容器直接使用 `O_DIRECT` 绕过页缓存，写限制可能不生效。

## 6. 与 systemd 集成

systemd 是 cgroups v2 最常用的操作系统级管理工具。从 v232 开始支持 cgroups v2，v247 之后全面切换到 v2。

### 6.1 通过 service 文件配置资源限制

```ini
# /etc/systemd/system/my-app.service
[Service]
ExecStart=/usr/local/bin/my-app
CPUWeight=500
MemoryMax=1G
MemoryHigh=768M
IOWeight=200
TasksMax=500
```

### 6.2 运行时修改

```bash
# 查看当前资源限制
$ systemctl show my-app --property=MemoryMax,CPUWeight

# 运行时修改
$ systemctl set-property my-app MemoryMax=2G CPUWeight=750
```

这些修改会立即生效并持久化到 `/etc/systemd/system.control/my-app.d/`。

### 6.3 使用 systemd-run 快速启动限制资源

```bash
# 启动一个限制 512MB 内存的临时 shell
$ systemd-run --user -t --property=MemoryMax=512M /bin/bash

# 启动一个限制 1 核心的临时命令
$ systemd-run --property=CPUQuota=100% --property=MemoryMax=256M \
    /usr/bin/python3 /tmp/heavy-script.py
```

## 7. 容器场景中的 cgroups v2

### 7.1 Docker 配置

Docker 从 20.10 开始默认支持 cgroups v2。确认配置：

```bash
$ docker info | grep cgroup
 Cgroup Driver: systemd
 Cgroup Version: 2
```

推荐使用 `systemd` cgroup 驱动而非 `cgroupfs`，因为 systemd 是 cgroups v2 的权威管理者，`cgroupfs` 可能导致资源状态不一致。

### 7.2 Kubernetes 配置

K8s 从 1.24 开始 GA 支持 cgroups v2。kubelet 的配置要点：

```yaml
# kubelet 配置
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

节点级别确认：

```bash
$ kubectl get nodes -o json | jq '.items[0].status.nodeInfo.containerRuntimeVersion'
"containerd://1.7.x"
```

### 7.3 容器资源限制的实际效果

通过 cgroups v2，容器的资源限制可以被内核直接执行，而不是通过容器运行时代理。下面的例子展示了一个 Pod 的 cgroup 树：

```bash
# 在宿主机上查看某个容器的 cgroup
$ find /sys/fs/cgroup/kubepods/ -name "cgroup.procs" -exec grep -l "12345" {} \;
/sys/fs/cgroup/kubepods/besteffort/pod123456/abc123/cgroup.procs

# 查看该容器的内存限制
$ cat /sys/fs/cgroup/kubepods/besteffort/pod123456/abc123/memory.max
536870912
```

## 8. 监控与调试

### 8.1 实时监控

```bash
# 查看某个 cgroup 的 CPU 使用率
$ cat /sys/fs/cgroup/my-app/cpu.stat
usage_usec 123456789
user_usec 100000000
system_usec 23456789
nr_periods 0
nr_throttled 0
throttled_usec 0

# 查看内存使用统计
$ cat /sys/fs/cgroup/my-app/memory.stat
anon 52428800
file 104857600
kernel_stack 1048576
...
```

### 8.2 使用工具监控

`systemd-cgtop` 是查看 cgroup 资源使用最方便的工具，类似 top 但按 cgroup 分组：

```bash
$ systemd-cgtop
Control Group                            Tasks   %CPU   Memory  Input/s Output/s
/kubepods.slice/                         -       0.2    1.2G     0B      0B
/system.slice/docker.service             2       0.0    28.0M    0B      0B
/user.slice/user-1000.slice              45      0.5    456.7M   0B      0B
```

`iostat -c` 配合 PSI 文件可以快速定位 IO 压力：

```bash
$ grep -r "" /sys/fs/cgroup/system.slice/memory.pressure /sys/fs/cgroup/system.slice/io.pressure
```

### 8.3 常见问题排查

**问题 1：`echo: write error: Device or resource busy`**

原因：试图将进程加入一个已经配置了控制器的 cgroup，但该进程已经属于另一个 cgroup 的子进程。解决方案：将进程先移出原 cgroup，或者使用 `cgroup.procs` 写入父进程而非子进程。

**问题 2：memory.high 设了但不生效**

原因：`memory.high` 是软限制，内核只在内存压力下回收。要强制回收，可以降低 `memory.high` 值，或者配合 `memory.max` 使用硬限制。

**问题 3：容器内 `free -m` 显示宿主机内存而非限制值**

原因：容器和宿主机共享内核，`free` 读取的是 `/proc/meminfo`（宿主机全局信息）。正确用法是 `cat /sys/fs/cgroup/memory.current` 查看容器实际内存。K8s 建议使用 `kubectl top pod`。

## 9. 从 v1 迁移到 v2 的注意事项

如果你的项目还在用 cgroups v1，迁移时需要注意：

1. **不再有独立的控制器挂载点**：`/sys/fs/cgroup/memory` 这样的路径不在了，统一到 `/sys/fs/cgroup/` 下。
2. **接口文件重命名**：`memory.limit_in_bytes` → `memory.max`，`cpu.shares` → `cpu.weight`，`blkio.weight` → `io.weight`。
3. **内核线程也受控**：v2 中 `cgroup.threads` 可以控制内核线程的资源，之前无法限制。
4. **控制器激活规则**：一个 cgroup 不能同时激活某个控制器，而父节点没有激活它。
5. **不再支持 `cgroup.clone_children`**：v2 使用 `cgroup.subtree_control` 统一控制。

## 10. 总结

Cgroups v2 是 Linux 资源管理的未来，所有主流发行版（Ubuntu 22.04+、Debian 12+、Fedora 36+、RHEL 9+）都已经默认使用 v2。掌握 v2 的核心概念和实战操作，对于容器化、系统调优和资源隔离至关重要。

几个关键点回顾：
- 统一层级树，所有控制器在一个挂载点下
- 控制器按需激活，父节点决定子节点权能
- 权重模型替代 shares 模型，更直观
- PSI 提供精确的资源压力监控
- systemd 是管理 cgroups v2 的首选工具
- 容器场景推荐使用 systemd cgroup 驱动

理解 cgroups v2 不只是为了配参数，更是为了理解你的程序在操作系统层面如何被调度和约束。当生产环境出现内存泄漏、CPU 节流、IO 抖动时，cgroups v2 的监控和调试能力是你第一道防线。