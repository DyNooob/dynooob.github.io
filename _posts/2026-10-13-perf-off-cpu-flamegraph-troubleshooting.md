---
layout: post
title: "perf + off-CPU 火焰图：从'CPU 不忙但请求卡顿'里挖出真凶"
date: 2026-10-13 09:00:00 +0800
categories: [网络技术, 开发]
tags: [perf, off-cpu, flamegraph, linux, 性能分析, 排障, Nginx, 调度, 内核, 可观测性]
---

## 问题：CPU 没满，请求却卡死

监控里看到这种场景最容易让人血压高：CPU 使用率只有 30%，但是业务的 P99 延迟从 50ms 涨到 800ms。`top` 看不出来、`vmstat` 也没什么异常、`netstat` 也没堆积。

这种「CPU 不忙但事情做不完」的状况，绝大多数时候是**线程在等**——等锁、等磁盘、等网卡中断处理、等调度器重新宠幸。传统的 on-CPU 火焰图（Brendan Gregg 那张著名的）只能告诉你 CPU 正在跑什么，**看不见它在哪里停下来了**。

而 off-CPU 时间，正是这种问题的真相所在。

这篇文章从头到尾演示怎么用 Linux 内核自带的 `perf` 工具生成一张有用的 off-CPU 火焰图，定位一个故意构造的 Nginx 故障——后端 upstream 慢响应导致前端请求堆积。

## 一、为什么 on-CPU 不够用

on-CPU 火焰图的采样方式是：每隔 N 微秒中断一次，记录「当前正在运行的指令」。

听起来很完美，但实际生产里 60% 以上的性能问题不是「CPU 跑得慢」，而是「CPU 没在跑」：

- **磁盘 IO 阻塞**：`read()` 调用的线程等 page cache 或磁盘
- **网络 IO 阻塞**：`recv()` 等对面发包，或者等内核协议栈
- **同步锁竞争**：拿不到 `pthread_mutex` 的线程被 park
- **调度器不公平**：cgroup 限流、CPU share 不足，runnable 但不 running
- **页错误**：major fault 会同步等 IO

这些场景里，CPU 占用率反而低——因为线程一直在睡觉。

off-CPU 火焰图反过来：**记录线程被阻塞的时刻和原因**。把 on-CPU 和 off-CPU 两张图叠起来看，你才真正看清「CPU 跑什么」+「CPU 在等什么」。

## 二、原理：sched 事件的 tracepoint

Linux 内核的调度器（`kernel/sched/` 子系统）通过 tracepoint 把每一次调度决策都暴露给用户态：

| Tracepoint | 含义 |
|------------|------|
| `sched:sched_switch` | 发生上下文切换：哪个任务离开 CPU，下一个跑谁 |
| `sched:sched_wakeup` | 一个睡眠任务被唤醒，进入 runnable |
| `sched:sched_stat_sleep` | 任务主动进入睡眠 |
| `sched:sched_stat_blocked` | 任务因 IO 等进入 blocked |
| `sched:sched_stat_iowait` | 任务进入 iowait |

off-CPU 火焰图的核心数据就是 `sched_switch`：`prev_pid → next_pid` 配对即可算出每个任务在采样窗口内的非运行时间。

Linux 5.x+ 内核的 tracepoint 都已经编进内核，不需要额外打 patch。`perf` 自带事件源。

## 三、环境准备

```bash
# 必要工具
sudo apt install -y linux-tools-common linux-tools-$(uname -r) \
                    bpfcc-tools linux-headers-$(uname -r) \
                    git python3

# Brendan Gregg 的火焰图工具（注意是 perl 脚本）
git clone --depth 1 https://github.com/brendangregg/FlameGraph.git
export FLAMEGRAPH=$(pwd)/FlameGraph
```

权限：off-CPU 采样需要 root（要读 tracepoint buffer）。`perf` 也可以用非 root，但你会看不见其他用户的栈信息。

## 四、采样：收集 off-CPU 栈

抓 30 秒数据：

```bash
# -g  记录调用栈
# -e  指定事件
# -a  所有 CPU
# -o  输出文件
# -C 0 指定 CPU 核心（-1 表示所有）
# sleep 30 采样时长
sudo perf record \
    -e sched:sched_switch \
    -e sched:sched_stat_sleep \
    -e sched:sched_stat_blocked \
    -e sched:sched_stat_iowait \
    -ag \
    -o /tmp/perf.data \
    -- sleep 30
```

**关键点**：

- `sched_switch` 是核心，必须加，其他三个可选但显著提高精度
- `-a -g` 全 CPU、全调用栈
- 不要 `perf record -p <pid>`！只采一个进程会把别的线程上下文切丢失，火焰图变成孤儿栈
- `--` 把后面的 `sleep 30` 传给被记录进程，而不是被 `perf record` 解释为它自己的参数

采样结束后，你会看到一个 `/tmp/perf.data`，通常是几十 MB（取决于负载和进程数）。

## 五、折叠栈：perf script → flamegraph 输入

`perf script` 把 `.data` 解析为可读的事件流，再用 `stackcollapse-perf-sched.pl` 折叠成 FlameGraph 工具链需要的格式。

Brendan Gregg 的工具集原本叫 `FlameGraph`，里面有两个相关脚本：

```bash
# 1. 解析 perf.data → 人类可读文本，喂给 perf script
sudo perf script --ns -i /tmp/perf.data > /tmp/perf.unfolded.txt

# 2. 折叠成 FlameGraph 的折叠格式
$FLAMEGRAPH/stackcollapse-perf-sched.pl /tmp/perf.unfolded.txt > /tmp/perf.folded.txt
```

**注意**：`stackcollapse-perf-sched.pl` 是专门给 off-CPU 用的，它会识别 `prev_comm → next_comm` 的切换配对并计算时间差。它和 `stackcollapse-perf.pl`（用于 on-CPU）是不同脚本，这是新手的常见坑。

## 六、生成火焰图

```bash
$FLAMEGRAPH/flamegraph.pl \
    --title "Off-CPU Time (30s)" \
    --subtitle "Nginx worker waiting" \
    --countname=ms \
    --colors=io \
    /tmp/perf.folded.txt > /tmp/offcpu.svg
```

`--colors=io` 让火焰图的颜色按函数类型着色（红=lock、蓝=net、绿=disk），比默认的暖色调直观。

打开 `/tmp/offcpu.svg` 浏览器看即可。鼠标悬停能看每个栈样本数量、展开路径。

## 七、实战：定位 Nginx upstream 慢响应

我故意搭了一个有问题的场景：

```
                   ┌──────────┐
                   │ upstream │   故意 sleep 200ms 才返回
                   │ 后端 PHP │
                   └────┬─────┘
                        │
                  proxy_pass
                        │
                   ┌────┴─────┐
                   │  Nginx   │   worker_connections=2048
                   │  worker  │
                   └────┬─────┘
                        │
                   客户端 ab 压测
```

### 7.1 触发问题

```bash
# 后端 PHP 故意睡 200ms
cat > /var/www/slow.php <<'EOF'
<?php usleep(200_000); echo "ok"; ?>
EOF

# Nginx 配置
server {
    listen 8080;
    location /slow {
        proxy_pass http://127.0.0.1:9000;
        proxy_connect_timeout 1s;
        proxy_read_timeout 5s;
    }
}

# 压测启动
ab -n 5000 -c 50 http://127.0.0.1:8080/slow
```

P99 飙到 250ms，正常应该是 220ms 左右（P99 = 200ms PHP + 20ms 网络排队）。

### 7.2 在压测期间采 off-CPU

开两个终端：

- 终端 A：`ab -n 5000 -c 50 ...`
- 终端 B：`sudo perf record -e sched:sched_switch ... -ag -- sleep 30`

### 7.3 看火焰图

生成的 `/tmp/offcpu.svg` 里，你会看到栈的最顶（leaf）基本上是：

```
nginx worker:epoll_pwait
  → sys_epoll_wait
    → schedule_timeout  ← 关键！最长最宽的栈就在这里
      → schedule
```

火焰图的**宽度**代表耗时——`schedule_timeout` 这一条柱子最宽，就说明 Nginx worker 大部分时间都在 epoll_wait 里睡着（因为等 upstream 的响应）。

更深一层会看到：

```
schedule_timeout
  → inet_wait_for_connect （连 upstream 时的等待）
  → ngx_http_upstream_connect
    → ngx_event_connect_peer

schedule_timeout
  → ep_poll  （连上后等响应数据）
  → sock_read
    → pipe_read (反向：等 Nginx → client 的数据回流)
```

具体哪个栈最宽，取决于你的瓶颈究竟在「建连慢」还是「响应慢」。

### 7.4 验证假设

要确认栈解读正确，可以用 `systemtap` 或 `bpftrace` 单独做一次定量测量：

```bash
# 用 bpftrace 统计每次 epoll_wait 唤醒的平均睡眠时长
sudo bpftrace -e '
kprobe:schedule_timeout {
    @usecs = hist(arg0);  // arg0 是 jiffies 转的微秒
}
'
```

如果分布集中在 20000us (20ms)、50000us (50ms)、200000us (200ms)，就锁定了上游延迟。

## 八、读图的几个核心技巧

### 8.1 优先看最宽的栈

火焰图原则：**最宽的 peak 就是瓶颈**。如果有三个平级的宽峰，说明是多个问题叠加。

### 8.2 不要只看 leaf function

`epoll_wait`、`schedule_timeout` 这种 leaf 几乎总是会出现，但它们是症状不是病因。**顺着栈往下看两三层**才能找到真正的业务逻辑函数（如 `ngx_http_upstream_connect`、`mysql_real_query`）。

### 8.3 长方块 vs 细尖塔

- **长方块**：时间被均匀分布在这个函数——稳定瓶颈
- **细尖塔**：偶发问题——某些时刻特别慢，需要扩大采样窗口

### 8.4 异常颜色

火焰图默认配色：

- 黄色：`__lock_text`，锁竞争
- 绿色：`__lock_text`，读侧等待
- 橙色：`__dentry_*`，文件系统
- 红色：网络相关

如果火焰图大面积黄色，95% 是锁问题，可以直接看代码定位。

## 九、常见误区和坑

### 9.1 不要把 perf 跑几个小时

off-CPU 数据量随时间线性增长，30MB/min 都算正常。一小时就是 1.8GB，浏览器打开 SVG 都卡。**采样窗口控制在 30-60 秒**，必要时多次采样对比。

### 9.2 高 IO 场景的精度问题

`sched_switch` 的 timestamp 精度是 jiffies（默认 4ms 或 10ms）。短于这个的睡眠时间会被四舍五入掉——如果你的问题在毫秒级，off-CPU 采样精度不够，要考虑用 bpftrace 的 hardware event。

### 9.3 别忘了过滤无关进程

`perf` 默认采所有 CPU、所有进程。如果你只关心某个服务：

```bash
sudo perf record -e sched:sched_switch -ag -C 0 -- sleep 30
# 或者事后过滤
sudo perf script --input /tmp/perf.data | grep nginx
```

### 9.4 内核符号表

火焰图里如果出现 `[kernel.kallsyms]` 这种纯地址看不懂，需要安装对应内核的 debug symbols：

```bash
sudo apt install linux-image-$(uname -r)-dbgsym
```

否则栈里关键函数全是数字地址。

## 十、和 on-CPU 火焰图叠用

最强大的是**把两张图叠成一张对比图**——同样调用栈，宽度按 on-CPU + off-CPU 加权：

```bash
# 先采 on-CPU
sudo perf record -F 99 -ag -o /tmp/on.data -- sleep 30
sudo perf script -i /tmp/on.data | $FLAMEGRAPH/stackcollapse-perf.pl > /tmp/on.folded

# 再采 off-CPU
sudo perf record -e sched:sched_switch ... -- sleep 30
sudo perf script -i /tmp/off.data | $FLAMEGRAPH/stackcollapse-perf-sched.pl > /tmp/off.folded

# 合并
$FLAMEGRAPH/flamegraph.pl --title "All CPU Time" /tmp/on.folded /tmp/off.folded > /tmp/all.svg
```

Brendan Gregg 的 `differential flamegraph` 可以更优雅地用红色标 on-CPU 增多、绿色标 off-CPU 增多。但那是另一个专题了。

## 十一、收尾

性能排查的工具很多：

- `top` / `htop` 看现象
- `strace` / `lsof` 看系统调用和文件
- `perf` / `bpftrace` 看内核事件
- `tcpdump` 看网络包
- 火焰图呈现分布

没有银弹，但**off-CPU 火焰图是少有的能直击「线程在等什么」的硬工具**。下次看到 CPU 不忙但延迟飙升，别急着重启服务——采个 30 秒的 off-CPU，图上的最宽峰会直接告诉你答案。

工欲善其事，必先利其器。`perf` 是内核送的，火焰图是开源的，门槛不高但回报很大。
