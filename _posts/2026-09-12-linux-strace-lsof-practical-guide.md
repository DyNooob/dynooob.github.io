---
layout: post
title: "Linux strace 与 lsof 实战排障指南：从入门到生产级应用"
date: 2026-09-12 09:00:00 +0800
categories: [开发]
tags: [Linux, strace, lsof, 系统管理, 排障, 性能分析, 调试, 系统调用, 文件描述符, 进程分析]
---

## 问题：Linux 排障的两个最强工具你吃灰了吗？

接手一台出问题的服务器，进程跑着但没响应、端口打不开、磁盘 unmount 不掉、Nginx 返回 502 但日志没报错——这些场景每天都在发生。

多数人会先 `ping`、`curl`、`telnet` 一通，不行就重启。但真正想找到根因，你需要两把手术刀：**strace** 和 **lsof**。

这两个命令分别是系统调用追踪器和文件描述符探针。组合使用，你能看到进程的每一根毛细血管——它打开了哪些文件、连接了谁、卡在哪个系统调用上、在读写什么数据。

这篇文章不讲 man page，用真实故障场景带你上手。读完你能在生产环境中直接用。

## 一、lsof：谁动了我的文件？

### 1.1 基本认知

`lsof` = List Open Files。在 Linux 里"一切皆文件"——普通文件、目录、socket、pipe、设备、共享内存，全部被当作文件描述符管理。lsof 就是枚举这些信息的万能工具。

**安全提示**：普通用户只能看自己的进程，`sudo lsof` 才能看全局。排障时永远用 root。

### 1.2 实战场景一：端口被谁占了？

```
$ sudo lsof -i :8080
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
java     12345 ubuntu  234u  IPv4 54321      0t0  TCP 0.0.0.0:8080 (LISTEN)
```

`-i :端口号` 直接告诉你哪个进程在监听。参数组合：

| 命令 | 作用 |
|------|------|
| `lsof -i :80` | 查 80 端口占用 |
| `lsof -i TCP` | 只看 TCP 连接 |
| `lsof -i @192.168.1.1` | 只显示与某 IP 相关的连接 |
| `lsof -iTCP:1-1024` | 端口范围过滤 |

**生产排障案例**：Jenkins 构建节点突然连不上。`lsof -i :50000` 发现多个旧进程残留，`kill -9` 后正常。

### 1.3 实战场景二：设备或文件系统被谁占用？

```
$ sudo umount /mnt/data
umount: /mnt/data: target is busy.

$ sudo lsof /mnt/data
COMMAND   PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
bash     25678 root  cwd    DIR  8,3     4096   2   /mnt/data
```

一个 bash 进程的 cwd（当前工作目录）在 `/mnt/data` 里。`kill -9 25678` 或者 `cd /` 切出去就行。

同理删除文件后磁盘空间没释放：

```
$ sudo lsof | grep deleted
tail      31234 root  4w   REG  8,3 1073741824 12345 /var/log/app.log (deleted)
```

tail 进程仍然持有已删除日志文件的句柄，空间不会真正释放。重启进程或关掉句柄解决。

### 1.4 实战场景三：进程到底在操作哪些文件？

```
$ sudo lsof -p 12345
COMMAND  PID  USER   FD   TYPE     DEVICE  SIZE/OFF     NODE NAME
nginx   12345 root  cwd    DIR      8,3       4096        2 /
nginx   12345 root  rtd    DIR      8,3       4096        2 /
nginx   12345 root  txt    REG      8,3    1420800  1234567 /usr/sbin/nginx
nginx   12345 root  mem    REG      8,3    2029592  2345678 /usr/lib/x86_64-linux-gnu/libc.so.6
nginx   12345 root    0u   CHR     1,3        0t0        4 /dev/null
nginx   12345 root    1u   CHR     1,3        0t0        4 /dev/null
nginx   12345 root    2u   CHR     1,3        0t0        4 /dev/null
nginx   12345 root    3u  IPv4    54321        0t0        TCP *:80 (LISTEN)
nginx   12345 root    4u  IPv4    54322        0t0        TCP *:443 (LISTEN)
nginx   12345 root    5u  unix    54323        0t0        /var/run/nginx.sock
```

每一行类型含义：

| 字段 | 含义 |
|------|------|
| FD | 文件描述符编号 + 模式（u=读写, r=只读, w=只写） |
| TYPE | 文件类型（REG=普通文件, DIR=目录, IPv4=网络, unix=Unix socket） |
| NAME | 实际路径或连接详情 |

FD 列的常见值：

- `cwd` — 当前工作目录
- `rtd` — root 目录
- `txt` — 程序代码段（可执行文件）
- `mem` — 内存映射文件（动态库）
- `0u/1u/2u` — stdin/stdout/stderr
- `3u` 及以上 — 打开的文件或 socket

### 1.5 lsof 实用组合拳

```bash
# 查看某用户的全部打开文件
lsof -u www-data

# 排除某个用户（看系统进程）
lsof -u ^root

# 查看进程的网络连接详情
lsof -i -a -p 12345

# 只显示 LISTEN 状态的连接
lsof -iTCP -sTCP:LISTEN

# 显示所有 Unix socket 连接
lsof -U

# 看哪些进程打开了某个 SUID 文件
lsof /usr/bin/sudo
```

## 二、strace：让进程开口说话

strace 拦截并记录进程的系统调用（system calls）和信号。任何程序与内核交互——读写文件、收发网络包、创建进程——都通过系统调用。strace 就是这些交互的录音带。

### 2.1 基本用法

```bash
# 跟踪已运行的进程
strace -p 12345

# 从头启动并跟踪
strace -o /tmp/trace.log ls /tmp

# 限制跟踪的子调用（推荐）
strace -e trace=open,read,write -p 12345
```

**重要警告**：strace 会让目标进程慢 10-50 倍。生产环境慎重，先用 -p 挂上去观察几秒钟就撤。

### 2.2 实战场景四：进程卡住不动

一个 Python HTTP 服务突然不响应了：

```bash
$ sudo strace -p 54321
epoll_wait(5, [], 128, 30000) = 0
epoll_wait(5, [], 128, 30000) = 0
epoll_wait(5, [], 128, 30000) = 0
...
```

进程在 epoll_wait 里正常等待事件，说明它空闲——问题不在进程本身，可能是上游没发请求。

换个场景：

```bash
$ sudo strace -p 54321
connect(6, {sa_family=AF_INET, sin_port=htons(5432), sin_addr=inet_addr("10.0.0.10")}, 16) = -1 EINPROGRESS (Operation now in progress)
connect(6, {sa_family=AF_INET, sin_port=htons(5432), sin_addr=inet_addr("10.0.0.10")}, 16) = -1 EINPROGRESS (Operation now in progress)
connect(6, {sa_family=AF_INET, sin_port=htons(5432), sin_addr=inet_addr("10.0.0.10")}, 16) = -1 EINPROGRESS (Operation now in progress)
```

进程卡在连接 PostgreSQL 5432 端口上，一直 EINPROGRESS，说明数据库不可达。`telnet` 或 `nc -zv 10.0.0.10 5432` 确认后，发现是安全组规则变更了。

### 2.3 实战场景五：配置文件路径错误

Nginx 启动失败，报 "unknown directive"——但你检查了一遍配置文件语法没错。用 strace 看看 Nginx 到底读了哪些文件：

```bash
$ sudo strace -e openat -f nginx -t 2>&1 | grep conf
[pid 123] openat(AT_FDCWD, "/etc/nginx/nginx.conf", O_RDONLY) = 7
[pid 123] openat(AT_FDCWD, "/etc/nginx/conf.d/default.conf", O_RDONLY) = 8
[pid 123] openat(AT_FDCWD, "/etc/nginx/conf.d/extra.conf", O_RDONLY) = 9
[pid 123] openat(AT_FDCWD, "/etc/nginx/conf.d/typo.conf~", O_RDONLY) = -1 ENOENT (No such file or directory)
```

等等——`typo.conf~` 是 Vim 残留的备份文件。`include conf.d/*.conf` 并没有排除 `~` 后缀文件，导致 Nginx 尝试加载一个残缺的备份文件。删掉备份文件，问题解决。

### 2.4 实战场景六：诊断性能瓶颈——慢在哪？

```bash
$ sudo strace -c -p 12345
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.12    0.321432        3214      1003       998 read
 30.23    0.215432        2154      1002         0 write
 12.11    0.086325         863      1001         0 epoll_wait
  5.67    0.040432          40      1001         0 openat
  4.21    0.030000          30      1000         0 close
  2.66    0.019000          19      1000         0 fstat
------ ----------- ----------- --------- --------- ----------------
100.00    0.712621       0.710      7007       998 total
```

`-c` 模式汇总系统调用时间和次数。注意 `read` 调用的错误率（998/1003 ≈ 99.5%），大量 read 返回错误（通常是 EAGAIN 或 EWOULDBLOCK）。这通常意味着：

1. 非阻塞 socket 在读空缓冲区时重复轮询
2. 没有用 epoll 的事件驱动，在 busy-loop

**修复方向**：检查代码是否用 `select/poll` 代替 `epoll` 做事件循环，或在非阻塞模式下缺少适当的休眠。

### 2.5 实战场景七：读文件内容（不修改进程）

```bash
# 读数据库进程正在写入的日志，只读不动
$ sudo strace -e read -p $(pidof mysqld) 2>&1 | head -20
read(38, "2026-09-12 01:23:45,678 [INFO] ", 4096) = 32
read(38, "slow query: SELECT * FROM user", 4096) = 32
```

想看进程访问了哪个 URL？对网络 fd 做 read/write 截获：

```bash
$ sudo strace -e read,write -s 256 -p 12345
write(6, "GET /api/v1/users HTTP/1.1\r\nHost", 28) = 28
read(6, "HTTP/1.1 200 OK\r\nContent-Type:", 4096) = 256
```

`-s 256` 让 strace 显示更多字节内容（默认 32 字节）。

## 三、lsof + strace 联合排障实战

### 3.1 案例：API 服务间歇性 502

**现象**：Nginx 反代的 Java 应用每隔几分钟返回 502。

**排查过程**：

第一步，lsof 确认连接状态：

```bash
$ sudo lsof -i -a -p $(pidof java)
java 23456 app    56u  IPv4 123456      0t0  TCP 10.0.0.5:37284->10.0.0.10:3306 (ESTABLISHED)
java 23456 app    57u  IPv4 123457      0t0  TCP 10.0.0.5:37286->10.0.0.10:3306 (ESTABLISHED)
java 23456 app    58u  IPv4 123458      0t0  TCP 10.0.0.5:37288->10.0.0.10:3306 (CLOSE_WAIT)
java 23456 app    59u  IPv4 123459      0t0  TCP 10.0.0.5:37290->10.0.0.10:3306 (ESTABLISHED)
```

发现 fd 58 处于 CLOSE_WAIT 状态——对端（MySQL）关闭了连接，但 Java 应用没有调用 close()。

第二步，strace 确认 fd 58 的行为：

```bash
$ sudo strace -e trace=read,write,close -p 23456 2>&1 | grep "fd 58\|58 "
write(58, "\3\0\0\1\3SELECT ...", ...) = -1 EPIPE (Broken pipe)
```

Java 试图向已关闭的 MySQL 连接写数据，收到 EPIPE 错误，抛出异常导致请求失败。

**根因**：MySQL 的 `wait_timeout` 默认 28800 秒（8 小时），但 HAProxy 在中间层设置了 `timeout server 300s`，如果连接池里的连接空闲超过 5 分钟，HAProxy 会断开。Java 应用的连接池没有做有效性检测，直接使用已断开的连接。

**修复**：在 JDBC 连接池配置中启用 `validationQuery` 和 `testOnBorrow`。

### 3.2 案例：磁盘空间报警但 du 对不上

**现象**：`df -h` 显示 `/` 使用率 95%，但 `du -sh /*` 加起来不到 50%。

```bash
$ sudo lsof | grep deleted
rsyslogd  1234  root  6w   REG  8,3 524288000 123456 /var/log/syslog (deleted)
```

rsyslogd 持有已删除的 syslog 文件句柄，5GB 空间没释放。重启 rsyslog 服务即可。

## 四、高级技巧与注意事项

### 4.1 strace 效率优化

生产环境 strace 的正确打开方式：

```bash
# 只跟踪耗时超过 0.1ms 的系统调用
strace -w -T -p 12345 2>&1 | awk -F'[<>]' '{if($2+0 > 0.1) print}'

# 跟踪新创建的线程（-f 跟随 fork）
strace -ff -e trace=network -o /tmp/net_trace -p 12345

# 只跟踪特定的系统调用（大幅减少开销）
strace -e trace=%network -p 12345
# %network = socket, connect, accept, sendto, recvfrom 等
# %file    = open, stat, read, write 等
# %desc    = close, ioctl, fcntl 等
# %process = fork, exec, exit 等
```

### 4.2 lsof 性能注意事项

- `lsof` 默认遍历 `/proc` 下所有进程，大系统数百个进程时第一次运行可能耗时 1-3 秒
- 用 `lsof -p PID` 指定进程最快
- `lsof +D /path` 递归目录查找，比 grep 输出快

### 4.3 替代工具对照

| 场景 | lsof/strace | 替代方案 | 优势 |
|------|------------|----------|------|
| 全局限文件句柄 | `lsof` | `/proc/sys/fs/file-nr` | 全局统计更快 |
| 系统调用耗时 | `strace -c` | `perf stat` | 开销更低 |
| 连续文件监控 | `strace -e trace=file` | `inotifywait` | 事件驱动更准 |
| 网络连接快照 | `lsof -i` | `ss -tulpn` | socket 统计更快 |
| 进程栈跟踪 | `strace`（慢） | `/proc/PID/stack` | 零开销看内核栈 |

### 4.4 安全提醒

- strace 会降低进程性能，不要在压测环境或高并发生产环境长时间运行
- lsof 输出可能包含敏感数据（文件路径、连接 IP），分享排查日志前脱敏
- strace 需要 `CAP_SYS_PTRACE` 或 root 权限，容器环境下默认不可用。需要 `--cap-add=SYS_PTRACE` 或 `securityContext.privileged=true`

## 五、速查表：排障时直接复制

```bash
# 端口谁在监听？
sudo lsof -i :<端口号>

# 文件被谁占用？
sudo lsof <文件路径>

# 进程所有打开文件
sudo lsof -p <PID>

# 进程卡在哪？
sudo strace -p <PID>

# 进程系统调用统计
sudo strace -c -p <PID>

# 只看网络相关调用
sudo strace -e trace=%network -p <PID>

# 已删除但未释放空间的文件
sudo lsof | grep deleted

# CLOSE_WAIT 连接
sudo lsof -iTCP -sTCP:CLOSE_WAIT

# 追踪子进程（多线程应用必加 -f）
sudo strace -ff -p <PID>

# 看进程读/写内容（最多 256 字节）
sudo strace -e read,write -s 256 -p <PID>
```

## 总结

strace 和 lsof 是 Linux 排障工具箱里最靠近内核的两把工具。不需要记住所有参数，核心就三句话：

1. **lsof** 回答"谁开了什么"——端口、文件、socket，一眼看穿
2. **strace** 回答"在做什么"——系统调用级别查看进程行为
3. **组合使用**——lsof 定位异常句柄 → strace 抓拍现场 → 确认根因

下次服务器出问题，别急着重启。先 `lsof` 问一句谁在搞鬼，再 `strace` 看一眼它在干什么。