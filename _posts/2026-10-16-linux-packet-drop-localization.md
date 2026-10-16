---
layout: post
title: "Linux 网络栈丢包定位实战:从 driver 到 socket 的 6 个丢包点与精准定位"
date: 2026-10-16 18:00:00 +0800
categories: [网络技术]
tags: [Linux, 网络丢包, dropwatch, eBPF, netstat, ss, conntrack, qdisc, NIC driver, 系统排障]
---

## 一次让人抓狂的 SSH 卡顿

去年运维同事遇到一个怪事:一台用作跳板机的服务器 SSH 登录正常,但登录后敲命令随机卡顿 200-500ms,有时候完全 timeout 重连。`ping` 不丢、`tcpdump` 看 SSH 包正常到达、`netstat -s` 里 received packets 正常递增。一切看起来都对,但用户体验就是不对。

最后定位到的是 **NIC 接收环形队列(RX ring buffer)溢出**——驱动丢弃的包根本不会出现在 `tcpdump` 抓包结果里(因为已经被驱动丢在入队之前),也不会反映在 TCP 层的丢包统计里。`netstat -s` 只统计进入协议栈之后的丢包,**驱动层的丢包在传统统计里几乎透明**。

这类问题在云上尤其常见。Linux 报文从到达网卡到被应用读取,中间会经过 6 个独立的"丢包点",每个点的诊断命令和统计计数器都不同。本文按报文流转顺序梳理这 6 个点,并给出每一步的精确定位方法。

---

## 一、Linux 报文流转的 6 个丢包点

下面这张图是从 NIC 硬件到用户态 socket 的完整路径:

```
                              用户态 Socket
                                   ↑
                              内核 TCP/UDP 协议栈
                                   ↑
                              Netfilter / Conntrack / IPsec
                                   ↑
                              Qdisc 队列 (tc 流量整形)
                                   ↑
                              IP 路由 / 转发子系统
                                   ↑
                              NIC RX Ring Buffer (DMA)
                                   ↑
                          网卡硬件 PHY (光模块/网线)
```

对应 6 个独立的丢包点:

| 编号 | 丢包位置 | 触发原因 | 关键排查命令 |
|------|----------|----------|--------------|
| L1 | NIC PHY/Ring | 网卡丢包(线速、NIC ring满、FCS错) | `ethtool -S`, `ip -s link` |
| L2 | 驱动入队 | ring buffer满、alloc_skb失败 | `dropwatch`, `perf` |
| L3 | IP层/路由 | 路由 miss、netfilter drop | `iptables -L -v`, `nft` |
| L4 | Qdisc 队列 | 发送/接收队列满、限速 | `tc -s qdisc` |
| L5 | Conntrack | CT 表满、timeout过短 | `conntrack -L`, `nf_conntrack` |
| L6 | Socket 缓冲区 | 应用读不及时、SO_RCVBUF过小 | `ss -m`, `netstat -s` |

最坑的是:**前一步丢的包,后面 5 步完全感知不到**。你必须用对应阶段的工具去查那个阶段的统计,不能跨级推断。

---

## 二、第一招:从应用层倒推 —— `ss` 和 `netstat -s`

最不需要特权的起点。看协议栈累计统计,有多少包在协议栈内被丢:

```bash
# 总体统计,关注 dropped / overflows
ss -s
# TCP:   170 (estab 60, closed 90, orphaned 0, timewait 0)
# Transport totals:  in  185.2MB   out  46.8MB
# packets   in  1423901   out  367895
# dropped   in  7823       out  0        ← L6 socket buffer 丢

netstat -s | grep -E "(drop|overflow|listen|listen queue|backlog)"
# 8323 SYNs to LISTEN sockets dropped
# 7823 packets dropped in socket buffer
```

两个关键数字:

- **`packets dropped in socket buffer`**:socket 接收缓冲区满,内核从 `SKB backlog queue` 把包丢掉的累计计数。这是最常见的丢包点,通常是应用读得太慢或 `SO_RCVBUF` 设置过小。
- **`SYNs to LISTEN sockets dropped`**:accept 队列满(`tcp_max_syn_backlog` + 应用程序 backlog 总和溢出)。新连接 SYN 直接被丢,客户端 timeout。

进一步定位是哪个 socket 出问题:

```bash
# 列出所有 socket 的内存使用和丢包累计
ss -tm | head -20
# 重点看 skmem:(r<rmem_alloc>...<wmem_alloc>) 和 ts 字段
# Recv-Q 一直不空 + dropped 列非零 = 必有应用读不动
ss -tan state established | awk 'NR>1 {print $4}' \
  | sort -u | while read p; do
    echo "=== $p ==="
    ss -tan '( sport = :'$p' or dport = :'$p' )' '( state established )' -m
  done

# 看内核对每个 socket 的累计丢包
cat /proc/net/udp
# 字段: sl  local_address rem_address st tx_queue rx_queue tr tm→when retrnsmt uid  inode
# 倒数第二列 drops 是 UDP socket 队列丢包累计
```

如果 `Recv-Q` 长期 > 0,基本就是应用层消费能力不够——这条不靠内核,靠改应用代码或加 worker。

---

## 三、第二招:网卡与驱动层的丢包 —— `ethtool -S` 和 `ip -s`

驱动层丢包完全不会出现在协议栈统计里,所以你必须用 NIC 自己的计数器:

```bash
ip -s link show eth0
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     RX: bytes  packets  errors  dropped  missed  mcast
#     TX: bytes  packets  errors  dropped  carrier collsns
#       24GB   18.2M     0        7821    0       12.3K

# 这里 dropped 和 missed 才是真正驱动丢包
# errors 通常是 NIC CRC 错、frame error
```

但 `ip -s link` 显示的是 NIC 驱动的"通用 dropped",具体到 RX 路径要更细:

```bash
ethtool -S eth0 | grep -iE "(drop|miss|buf|alloc|length|error|collision)"
# rx_dropped: 7821                ← NIC 通知驱动的丢包累计
# rx_missed:  0                    ← 中断没及时处理(罕见)
# rx_no_buffer_count: 7821         ← 驱动 alloc_skb 失败(RX ring满)
# rx_alloc_fail: 0                 ← 高阶内存回收失败
# rx_csum_offload_errors: 0        ← 校验和错误
# tx_queue_dropped: 0              ← TX 满
```

**`rx_no_buffer_count` 通常就是 RSS ring buffer 满**。修复方法:

```bash
# 1. 调大 ring buffer(临时)
ethtool -G eth0 rx 4096
# 关键: 升级到 firmware 默认的 max 一般是 4096+, 老 NIC 默认 256

# 2. 启用 adaptive coalescing(中断合并)
ethtool -C eth0 adaptive-rx on

# 3. 多队列 RSS: 把不同流分到不同 CPU 中断
ethtool -L eth0 combined 8         # 8 队列
echo 0f > /proc/irq/.../smp_affinity   # 把中断绑到具体 CPU
# 推荐用 irqbalance 自动管理

# 4. 大帧 MTU
ip link set eth0 mtu 9000
```

如果 `rx_csum_offload_errors`、`rx_frame_errors`、`rx_length_errors` 这些非零,是物理层问题(网线、光模块、CRC)——换线、查交换机 FEC 配置。驱动帮不了。

---

## 四、第三招:`dropwatch` 追踪丢包发生的函数

`netstat -s` 告诉你"丢了多少",`dropwatch` 告诉你"丢在哪个函数"——这才是真正定位 L2-L4 丢包点的工具。

```bash
# CentOS/RHEL: yum install dropwatch
# Ubuntu: apt install dropwatch

dropwatch -l kas
# 加载 kallsyms,准备解析函数名

dropwatch -a
# 开始抓取丢包函数触发,Ctrl+C 后输出:
# drop at: ip_rcv+0xa3 (0x11a3)
# drop at: tcp_v4_rcv+0x42 (0x1342)        ← TCP 协议处理丢
# drop at: __udp4_lib_rcv+0x1b0 (0x121b0)
# drop at: nf_conntrack_in+0x1c0 (0x15dc0)  ← Conntrack 满丢包!
# drop at: ip_route_input_slow+0x8a ...
```

`dropwatch` 本质是 tracepoint `kfree_skb` 上的一个 BPF probe,任何地方调 `kfree_skb(skb, reason)` 都会被记录。它给了你两件事:**丢包函数名** 和 **触发次数**。

生产环境用脚本自动化抓 30 秒:

```bash
#!/bin/bash
# drop_capture.sh - 抓 30 秒内的内核丢包点
DROP_LOG=/tmp/drop-$(date +%Y%m%d-%H%M%S).log

timeout 30 dropwatch -l kas -a > "$DROP_LOG" 2>&1 &
DROP_PID=$!

sleep 30
wait $DROP_PID 2>/dev/null

echo "=== 丢包点 Top 10 ==="
grep "^drop at:" "$DROP_LOG" \
  | awk '{print $4}' \
  | sort | uniq -c | sort -rn | head -10 \
  | while read count addr; do
    fname=$(echo "$addr" | grep -oP '\+\S+' | sed 's/+//')
    echo "$count  $fname"
  done
```

一个真实排障 case:某天线上 `nginx` P99 突然涨到 2s,看 `dropwatch` 输出高频出现 `nf_conntrack_in`——果然是 conntrack 表满(`nf_conntrack_max=262144`,云上 NAT 网关流量大)。改成更大值 + 调短 timeout 之后立即恢复。

---

## 五、第四招:Conntrack 表满 ——最隐蔽的丢包元凶

Conntrack 是 NAT 网关、负载均衡器上最容易被忽视的瓶颈。它有上限,满了之后新连接直接被丢:

```bash
# 当前连接数
cat /proc/sys/net/netfilter/nf_conntrack_count
# 当前 max
cat /proc/sys/net/netfilter/nf_conntrack_max
# 如果 count/max 接近 1.0,要么改大要么排查泄漏

# 看哪些连接占了最多
conntrack -L 2>/dev/null | awk '{print $3}' | sort | uniq -c | sort -rn | head
# 1234  tcp 6 300 SYN_SENT       ← 攻击 SYN flood 的特征
# 567   udp 17 60 UNREPLIED      ← UDP DNS 反射攻击

# 看 conntrack 丢包累计(内核层面)
grep -E "(drop|insert|invalid|search)" /proc/net/netfilter/nf_conntrack
# 重点:
#   insert_failed=7891        ← 表满了
#   drop=7891                 ← 满之后插入失败的丢包累计
#   invalid=12                ← 包校验不过(CT 状态机错了)
```

调优手段:

```bash
# 1. 调大(根据内存,每条 ~300 字节)
echo 524288 > /proc/sys/net/netfilter/nf_conntrack_max
echo 524288 > /proc/sys/net/nf_conntrack_max
sysctl -w net.netfilter.nf_conntrack_max=524288

# 2. 调短 TCP established timeout(默认 432000 = 5 天太久了)
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=3600
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_close_wait=30

# 3. UDP 超时(默认 30s,可能不够)
sysctl -w net.netfilter.nf_conntrack_udp_timeout=60
sysctl -w net.netfilter.nf_conntrack_udp_timeout_stream=120

# 4. 自动回收
echo 1 > /proc/sys/net/netfilter/nf_conntrack_acct
```

更彻底:对信任的内网流量直接 bypass conntrack(性能+规避攻击):

```bash
# iptables
iptables -t raw -A PREROUTING -s 10.0.0.0/8 -d 10.0.0.0/8 -j NOTRACK
iptables -t raw -A OUTPUT -s 10.0.0.0/8 -d 10.0.0.0/8 -j NOTRACK

# nftables equivalent
nft add table inet raw
nft add chain inet raw prerouting { type filter hook prerouting priority -300 \; }
nft add rule inet raw prerouting ip saddr 10.0.0.0/8 ip daddr 10.0.0.0/8 notrack
```

---

## 六、第五招:Qdisc 队列丢包 —— 出口带宽不足

Ingress 限速在 NIC 内部不靠 Qdisc,出口(Egress)流量整形靠 `tc qdisc`。当出口速率 < 上行速率,包会堆积在 Qdisc,满了就丢:

```bash
# 看默认的 pfifo_fast / fq_codel 队列统计
tc -s qdisc show dev eth0
# qdisc fq_codel 0: dev eth0 root refcnt 2 limit 10240p flows 1024 quantum 1514
#  Sent 18376245124 bytes 18102322 pkt (dropped 78230, overlimits 0 requeues 0)
#  backlog 12488p 8MB

# 重点:
# - dropped > 0      → 队列满了,真在丢
# - overlimits > 0   → 子流规则触发,可能没真丢但要警惕
# - backlog 持续高位 → 出口饱和
```

`tc -s qdisc show` 是单条命令里最简单的出口拥塞定位工具。如果 `dropped` 大但 `backlog` 高,先怀疑:

- 应用层突发流量(没有 rate shaping)
- NIC 出口带宽低于实际流量(错误买了 1Gbps 上联却用 10Gbps)
- 流量整形规则(`tc` classful)配置不当

临时绕过方案(只读,不改系统):

```bash
# 启用 TCP 早期反馈,让发送端早知道丢了
sysctl -w net.ipv4.tcp_early_retrans=1
sysctl -w net.ipv4.tcp_abc=0
```

---

## 七、第六招:eBPF 精确定位 ——`drop_reason` tracepoint

从内核 5.15 开始,`kfree_skb` 改成了 tracepoint,**带有一个 `enum skb_drop_reason` 的枚举**,可以精确定位丢包原因(比 `dropwatch` 详细得多):

```bash
# 内核 5.17+ 推荐用 bpftrace 看丢包原因分布
# 现代发行版 6.x 默认就有了
sudo bpftrace -e '
kprobe:kfree_skb / retval == 0 / {
    @drops[arg2] = count();
}
kretprobe:kfree_skb {
    @drops[retval] = count();
}

interval:s:5 {
    print(@drops);
    clear(@drops);
    printf("---\n");
}
' 2>&1 | tee /tmp/drop-reasons.log
```

输出会长这样:

```
@drops[1]: 7821       ← SKB_NOT_DROPPED_YET(技术上不是丢包)
@drops[3]: 14         ← SKB_CONSUMED
@drops[7]: 8821       ← SKB_DROP_REASON_NO_SOCKET(6!)   ← 大量没 socket 接收
@drops[12]: 1         ← SKB_DROP_REASON_TCP_CSUM
@drops[16]: 3         ← SKB_DROP_REASON_FULL_RING
```

SKB drop reason 完整枚举在 `<linux/skbuff.h>` 里。常用对照:

| Reason Code | 含义 | 排障方向 |
|------------|------|----------|
| 5 | NOT_SPECIFIED | 通用,看 `dropwatch` 函数名 |
| 7 | NO_SOCKET | 包送到了 IP 层但没人监听,或 socket buffer 满 |
| 13 | TCP_MISS | TCP lookup 时丢了(罕见) |
| 15 | FULL_RING | NIC ring buffer 满 |
| 22 | CONNTRACK | Conntrack 表满 |
| 28 | QDISC | Qdisc 队列满 |
| 41 | IP_RT_HDR | 路由头格式错 |
| 47 | TCP_AUTH_HDR | IPsec 认证失败 |

排障脚本化:用 perfetto/aw.io 之类的工具采样一段时间,统计 reason 分布 top 3,就能直接锁定问题阶段。

---

## 八、实战排障流程图

把上面六招组合成一个判断流程:

```
应用反馈丢包/P99 高
    │
    ▼
① ss -s / netstat -s → dropped>0? 
    ├─ YES → 哪个 socket? ss -tmo, 改应用或 rmem  → done
    └─ NO  ↓
② ethtool -S → rx_dropped / rx_no_buffer_count>0?
    ├─ YES → 调 ethtool -G / 启用多队列 / 软中断隔离 → done
    └─ NO  ↓
③ grep nf_conntrack → insert_failed>0?
    ├─ YES → 调大 conntrack_max + 调短 timeout → done
    └─ NO  ↓
④ tc -s qdisc → dropped>0?
    ├─ YES → 出口拥塞,调限速或扩容 → done
    └─ NO  ↓
⑤ dropwatch / bpftrace → 高频函数在哪?
    ├─ nf_conntrack_in → 回去看 ③
    ├─ ip_rcv          → 路由 + iptables
    ├─ tcp_v4_rcv      → accept 队列满, backlog/somaxconn
    └─ net_rx_action   → 软中断饱和, 启用 NAPI 调优或 RSS 分核
    │
    ▼
⑥ iptables -L -v / nft list ruleset → 有 drop 规则?
    ├─ YES → 检查策略
    └─ NO  → 抓包 + 对比收发,vlan/MTU/物理层
```

90% 的丢包问题能在前三步就定位。剩下 10% 走第四、第五步基本也能查到。再不行的就是 MTU 黑魔法或者物理层,那是另一个故事了。

---

## 九、几个容易踩的坑

**坑 1:`tcpdump` 抓包看不到的部分丢包**。`tcpdump` 走 AF_PACKET,在 IP 层之前。如果驱动丢包根本不会进入协议栈,`tcpdump` 看到的是"看起来一切正常"。永远要结合 `ethtool -S` 的真实驱动统计。

**坑 2:`netstat -s` 的 dropped 不更新**。某些发行版把 conntrack 丢包统计放在 `/proc/net/netfilter/nf_conntrack` 而不是 `netstat`,记得两个都看。

**坑 3:`dropwatch` 输出函数地址可能找不到符号**。需要 `CONFIG_KALLSYMS=y`,一些精简内核(容器基础镜像、定制 kernel)可能关掉了。验证:

```bash
cat /proc/kallsyms | grep -c " T "
# 0 → 没开 kallsyms, dropwatch 不可用,改用 eBPF
```

**坑 4:`rx_missed` 字段含义不同**。某些 NIC(e1000、igb)有独立的 `rx_missed`,某些(mellanox、virtio)只有 `rx_dropped`。需要按驱动对字段,不要照搬。

**坑 5:MTU 黑魔法丢包**。带 vxlan/gre 隧道或 vlan tag 的场景,内层包大、外层封装后 > 1500,**默认 MTU 没改**就丢。这种丢包 `netstat -s` 看不到,但 `ifconfig eth0` 的 `errors`/`drops` 会涨。

```bash
# 排查: 抓包看 IP DF 标志位和被分片情况
tcpdump -i eth0 -nn -v 'ip[6] & 0x40 = 0' | head
# 如果有包被丢时是 DF=1 + 长度 > 实际 MTU
# 临时方案: 改 NIC MTU 到 9000 或应用层 PMTUD
```

---

## 十、收尾

Linux 丢包定位的核心,不在于记多少命令,而在于**理解报文从硬件到用户态的每一步,以及每一步在哪里被丢、谁在计数**。`netstat` 不等于万能,`tcpdump` 也不等于真相。

把今天文里这套流程放进 runbook:

1. 第一步永远看 `ip -s link` 和 `ss -s`,先排最容易查的 L1/L6
2. 启动一个轻量 `dropwatch` 后台脚本采样 30 秒做 baseline
3. 任何"net包不对"的告警先怀疑 L1/RSS/L5,再去查应用层
4. 改完任何内核参数后 `sysctl -p` 立即生效,改 conntrack 参数前先 `conntrack -L | wc -l` 看现状
5. 生产环境把监控对上:`node_exporter` 抓 `node_network_*_drop_total`,`conntrack_max`、`softnet_stat` 都纳入 Grafana

下一步如果要做得更深,可以研究一下 `perf` + 静态跟踪里的 `net:net_dev_xmit`/`net:netif_receive_skb` tracepoint,以及最近在 6.x 内核里越来越成熟的 `skb drop reason` 全枚举集成到 systemd-journal——后者基本能取代上面 80% 的工作。
