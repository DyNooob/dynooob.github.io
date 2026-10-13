---
layout: post
title: "bpftrace 单文件追踪应用延迟：从一行命令到用户态火焰图"
date: 2026-10-13 18:00:00 +0800
categories: [开发, 网络技术]
tags: [bpftrace, eBPF, 性能分析, 可观测性, ustack, 直方图, 应用性能, 内核, 排障, linux]
---

## 一个真实的线上问题

上周生产环境出了个怪事：某个 Python 服务 P50 延迟 80ms，但 P99 突然涨到 4.2 秒。Grafana 上 CPU 不高、内存不抖、网络没有丢包，传统 APM 工具只能告诉你「这个端点慢」，但慢在哪一行、卡在哪一次 syscall、哪一段逻辑——一概不知。

这次我没用 `perf`，而是用了 `bpftrace`。理由很简单：**`perf` 适合 CPU 密集型场景的剖析，但「应用突然慢下来」多数时候是 CPU 没在跑**——要么在等 IO、要么在等锁、要么在等调度。和上一篇 `perf + off-CPU` 那篇文章互补，`bpftrace` 在「追踪任意事件、生成延迟直方图、捕获用户态调用栈」三件事上比 `perf` 灵活得多。

这篇文章不讲 eBPF 内核机制（那是另一篇长文），只演示 `bpftrace` 在生产排障里最常用的几种模式：从一行直方图开始，到用户态栈火焰图，再到自定义追踪点。

## 一、bpftrace 是什么，为什么比 BCC 顺手

`bpftrace` 是一个把 eBPF 程序**用类 awk 的 DSL**写出来的单文件工具。和 BCC 的差异：

| 维度 | BCC | bpftrace |
|------|-----|----------|
| 学习成本 | 写 C + Python 胶水 | 一行 DSL，无依赖 |
| 适用场景 | 复杂生产工具 | 临时诊断、一次性脚本 |
| 内置探针 | 需手写 | 200+ 内置（tracepoint、kprobe、usdt、software） |
| 输出 | 自定义 handler | 直方图、打印、栈跟踪、计数，开箱即用 |

安装一行：

```bash
# Ubuntu 22.04+
sudo apt install bpftrace

# 验证内核支持（需要 5.x+ 且开了 BTF）
bpftrace --version
bpftrace -e 'BEGIN { printf("hello\\n"); exit(); }'
```

需要 `linux-headers-$(uname -r)` 和 `bpfcc-tools` 提供一些 libbpf 的链接支持，但写 bpftrace 脚本时不强依赖。

## 二、第一招：延迟直方图 ——5 秒定位「慢在哪一段」

假设你的服务调用了 `read()` 系统调用，你想看它的延迟分布：

```bash
sudo bpftrace -e '
kprobe:sys_read {
    $start = nsecs;
    @start[tid] = $start;
}

kretprobe:sys_read / @start[tid] / {
    $delta = nsecs - @start[tid];
    @usecs = hist($delta / 1000);
    delete(@start[tid]);
}
'
```

跑 30 秒，Ctrl+C 后打印出一张延迟分布直方图：

```
@usecs:
[1]                 3 |@@                                      |
[2, 4)              7 |@@@                                     |
[4, 8)             18 |@@@@@@@                                 |
[8, 16)            34 |@@@@@@@@@@@@                            |
[16, 32)           62 |@@@@@@@@@@@@@@@@@@@@@@@@                |
[32, 64)           41 |@@@@@@@@@@@@@@@                         |
[64, 128)          12 |@@@@@                                   |
[128, 256)          8 |@@@                                     |
[256, 512)         35 |@@@@@@@@@@@@@@                          |
[512, 1K)        1821 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[1K, 2K)          301 |@@@@@@@@@@@@@@@@@@@@                    |
[2K, 4K)           44 |@@@@@@@@@@@                             |
```

**一眼看出问题**：99% 的 `read()` 在 1ms 内返回，但有 1800+ 次落入 512µs–1ms 区间，300+ 次超过 1ms。这典型的「偶发慢 IO」——磁盘 page cache miss 或者后端存储抖动。

### 进阶：分桶到进程

只关心某个进程：

```bash
sudo bpftrace -e '
kprobe:sys_read /comm == "python3"/ {
    @start[tid] = nsecs;
}
kretprobe:sys_read / @start[tid] / {
    @usecs[comm] = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}
'
```

`@usecs[comm]` 让直方图按进程名分组，自动分开多个 Python worker 的 IO 行为差异。

## 三、第二招：用户态调用栈 ——追踪应用函数耗时

直方图告诉你「**有多慢**」，但不知道「**哪段代码在慢**」。这时候挂 USDT（User Statically-Defined Tracing）探针，或者用 uprobe 追踪用户态函数：

```bash
sudo bpftrace -e '
uprobe:/usr/lib/x86_64-linux-gnu/libsqlite3.so.0:sqlite3_step {
    @start[tid] = nsecs;
}

uretprobe:/usr/lib/x86_64-linux-gnu/libsqlite3.so.0:sqlite3_step / @start[tid] / {
    $delta = nsecs - @start[tid];
    @usecs[ustack] = hist($delta / 1000);
    delete(@start[tid]);
}
'
```

把 `ustack`（用户态栈）作为直方图的 key，**慢的调用自动展开调用链**。运行后：

```
@usecs:
[ustack]
    sqlite3_step+1
    python3`_sqlite3_step_call+127
    python3`_PyEval_EvalFrameDefault+1234
    python3`_PyEval_EvalCodeWithName+189
    python3`PyEval_EvalCode+34
    python3`PyObject_Call+96
    ...
        3 |@                                                   |
        8 |@@                                                  |
       16 |@@@                                                 |
       32 |@@@@@@@                                             |
       64 |@@@@@@@@@@@@@                                       |
      128 |@@@@@@@@@@@@@@@                                     |
      256 |@@@@@@@@@@@@@@@@@@@@@@                              |
      512 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[1K, 2K) 1821 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
```

栈底自动展开，**你不需要写一行代码就能看到「Python 解释器 → sqlite3 绑定 → sqlite3_step」这条路径上有多少慢调用**。这就是 bpftrace 的杀手锏——把 eBPF 的复杂度藏起来，给你一张直观的图。

### 找到 sqlite3.so 的实际路径

不同发行版路径不同，先确认：

```bash
sudo bpftrace -l 'uprobe:/usr/lib/*/libsqlite3.so*:*' 2>/dev/null
ldconfig -p | grep sqlite3
```

## 四、第三招：自定义 tracepoint ——「业务级」的事件追踪

直方图和栈是事后排查。生产里更需要的是**实时看到业务逻辑在做什么**。比如想知道某个 HTTP 请求从 `accept()` 到 `read()` 之间隔了多久（多半是协议解析耗时）：

```bash
sudo bpftrace -e '
kprobe:inet_csk_accept {
    @accept_ts[tid] = nsecs;
    @accept_pid[tid] = pid;
}

kprobe:sys_recvfrom / @accept_ts[tid] / {
    @gap_usecs[comm, pid] = hist((nsecs - @accept_ts[tid]) / 1000);
    delete(@accept_ts[tid]);
}

kprobe:sys_read / @accept_ts[tid] / {
    @gap_usecs[comm, pid] = hist((nsecs - @accept_ts[tid]) / 1000);
    delete(@accept_ts[tid]);
}
'
```

这个脚本里：

- `inet_csk_accept` 是 TCP accept 完成的内核函数
- 第一个真正接收数据的 `recvfrom` 或 `read` 触发「间隔」测量
- 输出按 `comm`（进程名）和 `pid` 分桶，**一个 worker 线程卡在哪里一目了然**

我自己用这个定位过一个 Redis 实例的「接受连接但迟迟不收数据」问题——根因是客户端用 `tcp_nodelay=0` 凑满了 Nagle 算法窗口，主线程被 `tcp_sendmsg` 阻塞，结果是 worker 拿到 accept 后还要等几十毫秒才真正读到请求。改客户端配置后 P99 从 38ms 降到 7ms。

## 五、第四招：内核栈 ——「谁让 CPU 闲着」

只追踪用户栈有时候看不出问题根因，因为慢是内核造成的（比如软中断、调度延迟）。结合内核栈看：

```bash
sudo bpftrace -e '
kprobe:schedule {
    @stack[kstack] = count();
}
'
interval:5 {
    print(@stack, 20);
    clear(@stack);
}
```

`interval:5` 是 bpftrace 的「每 5 秒执行一次」事件，配合 `print(@stack, 20)` 输出栈出现次数 top 20。

更聚焦的版本——只看某个进程被调度出去的内核原因：

```bash
sudo bpftrace -e '
kprobe:schedule /comm == "python3"/ {
    @ctx_switch[kstack] = count();
}
'
```

如果栈顶反复出现 `wait4` 或 `do_wait`，说明 Python 多进程模型在反复 fork；如果出现 `io_schedule`，是在等 IO；如果出现 `__mutex_lock_slowpath`，是内核锁竞争。

## 六、生产注意事项

### 1. 过滤必须严格

bpftrace 的探针是**每次事件都触发**，没过滤的 `kprobe:sys_read` 在 64 核 + 高 IO 的机器上能直接打爆 CPU。先看量级：

```bash
sudo bpftrace -e 'kprobe:sys_read { @c = count(); }' 
# Ctrl+C 后看 @c 的增长速率，乘以 4 估算每秒触发次数
```

如果每秒触发超过 10 万次，加过滤条件或换更聚焦的 tracepoint（比如 `tcp:tcp_data_queue` 而不是 `sys_read`）。

### 2. 输出缓冲要意识到

bpftrace 默认按事件打印到 stdout，但部分内核事件密度极高，肉眼根本看不过来。生产里**强烈建议落到环形缓冲**：

```bash
sudo bpftrace -e '
kprobe:inet_csk_accept {
    printf("accept pid=%d tid=%d comm=%s\\n", pid, tid, comm);
}
' | tee /var/log/bpftrace-$(date +%Y%m%d).log
```

或者用 `-o` 直接写文件：

```bash
sudo bpftrace -o /var/log/bt.log -e '...'
```

### 3. 长时间运行要保护磁盘

`printf` 在高密度事件下会很快写满磁盘。bpftrace 提供环形缓冲 + `SIGINT` 触发 flush：

```bash
sudo bpftrace --max-buffer=64M -e '...'
```

我自己在生产机器上跑追踪从来不超过 10 分钟——足够抓样本，磁盘也安全。

### 4. 容器内的限制

容器默认没有 `CAP_BPF` 和 `CAP_SYS_ADMIN`，需要：

- 宿主机直接执行（推荐）
- 或者在 Pod Security Policy / SecurityContext 里开 `privileged: true`（生产慎用）
- 或者用 [Tetragon](https://github.com/cilium/tetragon) 这类专门为容器设计的 eBPF 工具

## 七、实战模板合集

直接复制能用的几个：

**模板 1：HTTP 请求延迟（按端点直方图）**

```bash
sudo bpftrace -e '
usdt:/usr/bin/python3:cpython:function__entry /comm == "python3"/ {
    @start[tid, arg0] = nsecs;
}
usdt:/usr/bin/python3:cpython:function__return / @start[tid, arg1] / {
    @usecs[arg1] = hist((nsecs - @start[tid, arg1]) / 1000);
    delete(@start[tid, arg1]);
}
' 2>&1 | grep -v "WARNING:"
```

这需要 Python 编译时带 `--with-dtrace`（Ubuntu 的官方包默认带）。`arg0`/`arg1` 是 USDT 参数槽里的函数名和返回地址。

**模板 2：某进程所有 syscall 耗时 top 10**

```bash
sudo bpftrace -e '
kprobe:do_syscall_64 /comm == "nginx"/ {
    @start[tid] = nsecs;
    @sc[tid] = arg1;
}
kretprobe:do_syscall_64 / @start[tid] / {
    @ns_hist[@sc[tid]] = hist(nsecs - @start[tid]);
    delete(@start[tid]);
    delete(@sc[tid]);
}
'
```

**模板 3：磁盘 IO 等待任务**

```bash
sudo bpftrace -e '
kprobe:blk_mq_start_request {
    @start[tid] = nsecs;
}
kretprobe:blk_mq_start_request / @start[tid] / {
    @req_usecs[comm] = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}
'
```

`blk_mq_start_request` 是多队列块设备层提交 IO 的入口，延迟分布直接反映存储子系统压力。

## 八、和其它工具的关系

写到这里容易让人困惑：bpftrace、`perf`、`strace`、BCC 到底什么场景用什么？我自己的经验：

- **bpftrace**：临时诊断、火焰图、直方图、自定义追踪点（90% 的生产事故）
- **perf + off-CPU**：CPU 不忙但事情做不完（上周那篇文章的场景）
- **strace**：追踪单个进程的系统调用（开发调试，不适合生产）
- **BCC**：需要长期运行的复杂 eBPF 工具（网络观测器、安全检测器）
- **Tetragon / Cilium**：Kubernetes 场景下的运行时安全

工具不是越高级越好，关键是**和问题的尺度匹配**——一次性的延迟分析用 bpftrace，持续的运行时观测用 BCC/Tetragon，离线分析用 perf。

## 写在最后

回到开头的那个 Python 服务问题：用 bpftrace 挂了一个 `uprobe:libpython3.10.so.1.0:PyEval_EvalCodeWithName` 的探针，5 分钟就抓到一帧明显的栈——栈底是 `json.loads`，栈顶反复出现 `PyObject_Malloc`。根因是某次 config 改动后，业务调用把一个 80MB 的 JSON 字符串整个塞给了 `json.loads`，每次都触发大量小块内存分配。

修改业务逻辑（流式解析 + dict 直接构造）后，P99 从 4.2 秒回到 90ms。

**bpftrace 的价值就在于：你不需要修改一行业务代码、不需要重启服务，就能在 5 分钟内拿到「卡在哪一行」的真实证据**。它和 `perf` 互补——`perf` 看 CPU 在跑什么，`bpftrace` 看 CPU 没在跑的时段里发生了什么。

下次遇到「监控看着没事但用户喊慢」的场景，先来一行 `bpftrace -e '...'`。
