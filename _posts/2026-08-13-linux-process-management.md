---
layout: post
title: "Linux 进程管理：从 ps 到 systemd 的排查思路"
date: 2026-08-13 18:00:00 +0800
categories: [开发]
tags: [Linux, 进程管理, systemd, 运维, 排查]
---

进程管理是 Linux 运维的基础操作。但很多人只会 `ps aux` 和 `kill -9`，遇到进程异常、资源泄漏、僵尸进程等问题时不知道从哪入手。

## 查看进程

### ps：静态快照

```bash
ps aux                # 所有进程，BSD 格式
ps -ef                # 所有进程，标准格式
ps aux --sort=-%cpu   # 按 CPU 排序
ps aux --sort=-%mem   # 按内存排序
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head  # 自定义输出
```

### top/htop：动态监控

```bash
top                  # 实时进程列表
top -u 用户名         # 只看某个用户的进程
top -p PID1,PID2     # 只看指定 PID

htop                 # 更友好的界面（需安装）
```

top 中常用交互键：
- `P`：按 CPU 排序
- `M`：按内存排序
- `k`：杀死进程
- `u`：过滤用户
- `1`：查看每个 CPU 核心

### 进程树

```bash
pstree               # 树形结构，清晰看到父子关系
pstree -p            # 显示 PID
pstree 用户名         # 只看某个用户的进程树
```

## 排查进程异常

### CPU 高

```bash
# 找到 CPU 占用最高的进程
top -b -n 1 | head -20

# 查看进程的线程
top -H -p PID

# 查看进程的 CPU 使用详情
ps -p PID -o pid,ppid,user,%cpu,cmd,lstart
```

### 内存泄漏

```bash
# 查看进程内存使用
ps -p PID -o pid,rss,vsz,%mem,cmd

# 持续监控内存变化
watch -n 5 'ps -p PID -o pid,rss,%mem,cmd'

# 查看系统内存整体状况
free -h
cat /proc/meminfo
```

### 僵尸进程

僵尸进程是已经终止但父进程没有回收的进程。表现为状态为 Z 的进程。

```bash
# 查找僵尸进程
ps aux | grep 'Z'
ps -eo pid,stat,cmd | grep 'Z'

# 查看僵尸进程的父进程
ps -o pid,ppid,state,cmd -p ZOMBIE_PID

# 如果父进程还在，可以 kill 父进程让其回收子进程
kill -s SIGCHLD PARENT_PID
# 或直接 kill 父进程，让 init 回收
kill PARENT_PID
```

如果父进程已经死了，僵尸进程会被 init（PID 1）收养并自动回收。

### 进程卡死

```bash
# 查看进程状态（D=不可中断睡眠，R=运行，S=睡眠，T=停止，Z=僵尸）
ps -p PID -o pid,stat,wchan

# D 状态（不可中断睡眠）通常表示等待 I/O
# 如果大量 D 状态进程，检查磁盘或网络存储

# 查看进程的堆栈
cat /proc/PID/stack

# 查看进程打开的文件
lsof -p PID
```

## 进程控制

```bash
# 停止进程
kill -15 PID        # 优雅停止（SIGTERM）
kill -9 PID         # 强制杀死（SIGKILL）- 最后手段

# 暂停和恢复
kill -19 PID        # 暂停（SIGSTOP）
kill -18 PID        # 恢复（SIGCONT）

# 按名称杀死
pkill -f 进程名      # 匹配进程名
killall 进程名       # 杀死所有同名进程
```

## systemd 服务管理

现代 Linux 发行版大多使用 systemd 管理服务。

```bash
# 查看服务状态
systemctl status 服务名

# 查看所有运行中的服务
systemctl list-units --type=service --state=running

# 查看失败的服务
systemctl --failed

# 查看服务日志
journalctl -u 服务名
journalctl -u 服务名 -f    # 实时追踪
journalctl -u 服务名 --since "1 hour ago"

# 查看服务的依赖关系
systemctl list-dependencies 服务名
```

## 资源限制

当进程异常时，可能是系统资源限制导致的。

```bash
# 查看进程的资源限制
cat /proc/PID/limits

# 查看系统级的限制
ulimit -a

# 查看最大文件描述符数
cat /proc/sys/fs/file-max

# 查看当前使用的文件描述符数
lsof -p PID | wc -l
```

## 快速排查流程

```
问题现象 → 检查方向 → 命令
CPU 高     → 哪个进程    → top/ps
内存高     → 进程内存    → ps -eo pid,rss
进程卡死   → 状态/堆栈   → ps stat, /proc/pid/stack
僵尸进程   → 父进程      → ps -eo pid,stat
启动失败   → 服务状态    → systemctl status
日志错误   → 查看日志    → journalctl -u