---
layout: post
title: "Redis 安全加固与性能优化实战指南"
date: 2026-08-02 18:00:00 +0800
categories: [安全, 开发]
tags: [Redis, 安全加固, 性能优化, 缓存, 系统管理, 运维, ACL, 持久化]
---

Redis 是当前最流行的内存数据库，从 Session 缓存到消息队列，从排行榜到实时计数，无处不在。但很多人装完就用默认配置跑起来——没有密码、没有 ACL、持久化也没调好。这就像一个开着门的保险柜，里面的数据唾手可得。

本文从安全加固和性能优化两个维度，整理一套可以直接上手的实战配置，覆盖 Redis 6.x/7.x 版本。

## 一、安全加固

### 1.1 绑定地址与保护模式

默认配置下 Redis 监听 `127.0.0.1`，但不少人在云服务器上直接 `bind 0.0.0.0` 暴露到公网。Shodan 上搜一下 `port:6379` 能看到大量无密码实例，有些甚至直接被人种了挖矿程序。

```bash
# redis.conf
bind 127.0.0.1              # 只监听本地
# 如果必须对外暴露，指定具体内网 IP
bind 10.0.1.100
protected-mode yes          # 必须开启
port 6379                   # 建议换端口，比如 16379
```

**`protected-mode` 的工作原理**：当满足以下三个条件同时成立时，Redis 只接受来自回环地址的连接：
- 未设置 `bind`
- 未设置密码（`requirepass`）
- `protected-mode` 为 `yes`

只要设置了密码或 `bind`，保护模式就自动失效。所以**不要依赖保护模式替代密码**，它只是兜底机制。

**端口隐藏**：把默认 6379 换成高位端口（如 16379、26379），可以过滤掉绝大多数的自动化扫描：

```bash
# 同时在内核层面限制
iptables -A INPUT -p tcp --dport 16379 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 16379 -j DROP
```

### 1.2 密码认证

Redis 6.0 之前只有一个 `requirepass`，全局同一个密码，所有人都拥有全部权限。6.0 之后引入了 ACL（Access Control List），支持细粒度权限控制。

**基础密码（兼容旧版客户端）：**

```bash
# redis.conf
requirepass "your-strong-password-here"
```

客户端连接：

```bash
redis-cli -a "your-strong-password-here"
# 或连接后输入
AUTH "your-strong-password-here"
```

注意 `-a` 参数会暴露密码到进程列表，生产环境建议用 `AUTH` 命令或通过环境变量传递。

**推荐：ACL 方式（Redis 6.0+）：**

```bash
# redis.conf
aclfile /etc/redis/users.acl
```

`users.acl` 文件内容：

```
user default off                    # 禁用默认用户
user admin on >AdminPass123! ~* &* +@all
user app on >AppPass456! ~cache:* -@dangerous +@read +@write -@admin +@fast
user readonly on >ReadOnly789! ~* -@all +@read
```

ACL 规则语法说明：

| 语法 | 含义 |
|------|------|
| `on` / `off` | 启用/禁用用户 |
| `>password` | 设置密码 |
| `~pattern` | 允许访问的 key 模式（支持通配符） |
| `&pattern` | 允许访问的 Pub/Sub 频道模式 |
| `+@category` | 允许某个命令类别 |
| `-@category` | 禁止某个命令类别 |
| `+command` | 允许具体命令 |
| `-command` | 禁止具体命令 |

**命令类别速查**：

- `@read`：读取类（GET, MGET, HGET, KEYS, SCAN 等）
- `@write`：写入类（SET, SETEX, DEL, HSET 等）
- `@admin`：管理类（CONFIG, FLUSHALL, SHUTDOWN, SAVE 等）
- `@dangerous`：危险类（FLUSHALL, FLUSHDB, DEBUG, SHUTDOWN, KEYS, CONFIG 等）
- `@fast`：O(1) 或 O(log N) 的快速命令
- `@slow`：O(N) 或更慢的慢速命令

**ACL 常用操作**：

```bash
# 查看所有用户
redis-cli ACL LIST

# 查看指定用户
redis-cli ACL GETUSER app

# 在线创建用户（无需重启）
redis-cli ACL SETUSER monitor on >MonPass123! ~* +@read +@pubsub

# 删除用户
redis-cli ACL DELUSER old_user

# 查看 ACL 日志
redis-cli ACL LOG
```

### 1.3 禁用危险命令

即使有 ACL，建议在配置层面直接重命名或禁用高危命令。这是深度防御——即使 ACL 被绕过，这些命令也根本不存在：

```bash
# redis.conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
rename-command EVAL ""        # 禁用 Lua 沙箱
rename-command DEBUG ""
rename-command SHUTDOWN ""
rename-command SCRIPT ""
rename-command REPLICAOF ""
rename-command SLAVEOF ""
rename-command DEBUG ""
rename-command OBJECT ""
rename-command CLIENT ""
```

如果业务确实需要某个命令，可以重命名为一个难猜的名字：

```bash
rename-command FLUSHALL "9f8a7b3c_FLUSHALL"
```

**为什么特别禁用 `EVAL` 和 `SCRIPT`**：Redis 的 Lua 沙箱历史上出现过多次绕过漏洞（CVE-2022-0543、CVE-2023-28857 等），攻击者可以远程执行 Lua 代码甚至逃逸到系统命令。如果业务不需要 Lua 脚本，直接禁用最安全。

### 1.4 网络层加固

**使用 TLS 加密传输（Redis 6.0+）：**

```bash
# redis.conf
port 0                          # 关闭明文端口
tls-port 16379
tls-cert-file /etc/redis/redis.crt
tls-key-file /etc/redis/redis.key
tls-ca-cert-file /etc/redis/ca.crt
tls-auth-clients yes
tls-protocols "TLSv1.2 TLSv1.3"
tls-ciphers "HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4"
```

TLS 配置尤其适合跨机房或公网传输的场景。Redis 的通信协议默认是明文，任何人只要能在网络路径上抓包，就能看到所有数据。用 Wireshark 过滤 `redis` 协议就能直接看到 key 和 value。

**客户端连接 TLS 加密的 Redis：**

```bash
redis-cli --tls --cert /etc/redis/client.crt \
    --key /etc/redis/client.key \
    --cacert /etc/redis/ca.crt \
    -p 16379
```

### 1.5 Linux 内核安全

```bash
# 使用专用用户运行
useradd -r -s /sbin/nologin redis
chown -R redis:redis /var/lib/redis /etc/redis

# 配置 systemd 单元文件
cat > /etc/systemd/system/redis.service << 'EOF'
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
Restart=always
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true
PrivateTmp=true
MemoryMax=4G
MemoryHigh=3.5G

[Install]
WantedBy=multi-user.target
EOF
```

**内核参数优化：**

```bash
cat >> /etc/sysctl.conf << 'EOF'
# 禁止交换（内存数据库不应该被换出）
vm.swappiness = 1

# 允许内存过量分配（持久化 fork 时需要）
vm.overcommit_memory = 1

# 增大网络缓冲区
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# 启用 TCP 快速回收和复用
net.ipv4.tcp_tw_reuse = 1
EOF
sysctl -p

# 禁用透明大页（THP 会导致 Redis 延迟波动）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' >> /etc/rc.local
```

## 二、性能优化

### 2.1 内存管理

**最大内存限制：**

```bash
# redis.conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

`maxmemory-policy` 可选策略对比：

| 策略 | 淘汰逻辑 | 适用场景 | 注意事项 |
|------|----------|----------|----------|
| `noeviction` | 内存满时写操作返回错误 | 纯缓存，允许丢数据 | 需要业务层处理写入失败 |
| `allkeys-lru` | 淘汰最近最少使用的 key | 通用缓存 | 最常用的策略，适配大多数场景 |
| `allkeys-lfu` | 淘汰最不经常使用的 key | 热点数据明显 | 比 LRU 更精确，但内存开销稍大 |
| `volatile-lru` | 仅淘汰带 TTL 的 key | 混合使用持久化数据 | 不带 TTL 的 key 永远不会被淘汰 |
| `volatile-ttl` | 淘汰 TTL 最短的 key | 有时效性数据 | 注意 TTL 相同的 key 选择是随机的 |

**LRU vs LFU 选型**：

- **LRU（Least Recently Used）**：淘汰最近最少访问的。适合访问模式比较均匀的场景。实现简单，但偶发的大量访问会污染缓存。
- **LFU（Least Frequently Used）**：淘汰访问频率最低的。适合有明显热点的场景（比如 20% 的 key 占了 80% 的访问）。Redis 7.0+ 对 LFU 做了优化，内存开销与 LRU 相同。

**合理使用内存优化编码：**

Redis 在内部根据数据大小自动选择不同的编码方式，可以手动配置阈值：

```bash
# redis.conf —— 内存优化
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2      # -2 表示每个特殊编码的列表节点不超过 8KB
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
```

这些小对象编码优化可以减少 30%-50% 的内存占用。100 万个 10 个字段的哈希对象，用 ziplist 编码只需约 45MB，而默认的 hashtable 编码需要约 120MB。

**诊断内存使用：**

```bash
# 内存统计
redis-cli MEMORY STAT
redis-cli MEMORY DOCTOR      # 自动诊断和建议

# 找出最大的 key
redis-cli --bigkeys

# 分析内存碎片
redis-cli INFO memory | grep -E "fragmentation|rss"
```

### 2.2 持久化调优

RDB 和 AOF 各有优劣，生产环境通常两者配合，也需要根据业务场景选型。

**RDB（快照）**：

```bash
# redis.conf
save 900 1        # 900 秒内有 1 次写就快照
save 300 10       # 300 秒内有 10 次写
save 60 10000     # 60 秒内有 10000 次写
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
```

**AOF（追加日志）**：

```bash
# redis.conf
appendonly yes
appendfsync everysec    # 每秒刷盘，平衡性能和安全
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
no-appendfsync-on-rewrite yes
```

`appendfsync` 三种模式对比：

| 模式 | 安全性 | 性能 | 最多丢数据 |
|------|--------|------|------------|
| `always` | 最高 | 最慢（每次写入都 fsync） | 1 条命令 |
| `everysec` | 中 | 较好（每秒 fsync） | 1 秒数据 |
| `no` | 最低 | 最快（交给内核刷盘） | 不确定 |

**RDB + AOF 混合持久化（Redis 4.0+）：**

```bash
aof-use-rdb-preamble yes
```

混合模式下，AOF 重写时直接生成 RDB 格式的基数据，再追加增量 AOF 日志。优点是重启恢复速度快（比纯 AOF 快几十倍），同时保留 AOF 的数据安全性。

**持久化方案选型建议：**

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 纯缓存，可丢数据 | 关掉持久化 | 性能最高 |
| 缓存，重启不能清空 | 仅 RDB | 简单快速 |
| 重要数据，可容忍少量丢失 | RDB + AOF（everysec） | 兼顾性能和安全 |
| 金融/交易数据 | AOF（always）+ 主从 | 最大程度保数据 |

### 2.3 连接与 IO 优化

```bash
# redis.conf
timeout 300                     # 空闲连接 5 分钟超时
tcp-keepalive 300               # TCP keepalive 探测间隔
tcp-backlog 511                 # TCP 连接队列大小
maxclients 10000                # 最大并发连接数
```

**IO 线程（Redis 6.0+，多线程处理网络 IO）：**

```bash
io-threads 4                    # 建议 = CPU 核心数
io-threads-do-reads yes         # 启用读线程
```

**IO 线程配置要点：**

- `io-threads` 一般设为 CPU 核心数，超过 8 核心的建议设为 8
- 只有写入密集型场景收益明显，纯读或读多写少场景可能反而不如单线程——因为线程切换有开销
- Redis 仍然是单线程执行命令的——IO 线程只负责网络读写，命令执行引擎仍然是单线程
- 开启后可以通过 `redis-cli INFO IO` 查看 IO 线程的负载情况

### 2.4 慢查询监控

```bash
# redis.conf
slowlog-log-slower-than 10000    # 超过 10ms 的记录（单位：微秒）
slowlog-max-len 128
```

慢查询管理：

```bash
# 查看最近的慢查询
redis-cli SLOWLOG GET 10

# 查看慢查询数量
redis-cli SLOWLOG LEN

# 清空慢查询日志
redis-cli SLOWLOG RESET
```

结果解读：

```
1) 1) (integer) 14              # 唯一 ID
   2) (integer) 1722567890      # Unix 时间戳
   3) (integer) 45023           # 执行耗时（微秒，即 45ms）
   4) 1) "KEYS"                 # 命令
      2) "user:*"
   5) "127.0.0.1:6379"          # 客户端地址
   6) ""                        # 客户端名称
```

如果看到 `KEYS`、`SMEMBERS`、`LRANGE` 等命令出现在慢查询里，说明需要优化数据结构或查询方式。`KEYS` 尤其危险——它在 O(N) 的全量扫描时会阻塞 Redis 主线程，所有其他请求都会排队等待。

### 2.5 主从与集群

**主从复制安全：**

```bash
# 从库配置
replicaof 10.0.1.100 6379
masterauth MasterPass123!
replica-read-only yes

# 主从复制安全（要求从库使用密码认证）
masteruser repl
masterauth ReplPass456!
```

**哨兵模式（Sentinel）配置：**

```bash
# /etc/redis/sentinel.conf
port 26379
sentinel monitor mymaster 10.0.1.100 6379 2
sentinel auth-pass mymaster MasterPass123!
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

**集群模式：**

```bash
# 每个节点配置
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
# 至少需要 6 个节点（3 主 3 从）
# 每个节点至少 1 个从库才能保证高可用
```

### 2.6 生产配置清单

一个可直接用的生产级 `redis.conf` 核心配置：

```bash
# 安全
bind 127.0.0.1
protected-mode yes
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
rename-command EVAL ""

# ACL（从文件加载）
aclfile /etc/redis/users.acl

# 内存
maxmemory 80%
maxmemory-policy allkeys-lru

# 持久化
save 900 1
save 300 10
save 60 10000
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

# 连接
timeout 300
tcp-keepalive 300
maxclients 10000
io-threads 4
io-threads-do-reads yes

# 日志
loglevel notice
logfile /var/log/redis/redis.log
slowlog-log-slower-than 10000
slowlog-max-len 128

# 内核优化提示
# 别忘了设置：
#   vm.overcommit_memory = 1
#   vm.swappiness = 1
#   /sys/kernel/mm/transparent_hugepage/enabled = never
```

## 三、常见问题排查

### 3.1 连接超时或拒绝

```bash
# 检查 Redis 是否在监听
ss -tlnp | grep 6379

# 检查防火墙规则
iptables -L -n | grep 6379

# 检查网络延迟
redis-cli --latency -h 10.0.1.100 -p 6379

# 检查最大连接数是否已满
redis-cli INFO clients | grep connected_clients
```

### 3.2 内存异常增长

```bash
# 内存诊断
redis-cli MEMORY STAT
redis-cli MEMORY DOCTOR

# 查看 key 分布
redis-cli --bigkeys

# 查看 key 过期情况
redis-cli INFO keyspace

# 查看内存碎片率
redis-cli INFO memory | grep fragmentation
# 碎片率 > 1.5 说明碎片严重，考虑重启或 MEMORY PURGE
```

### 3.3 延迟波动

```bash
# 内置延迟检测
redis-cli --intrinsic-latency 100

# 查看慢查询
redis-cli SLOWLOG GET 20

# 查看延迟 event 统计
redis-cli INFO stats | grep -E "latency|total_commands"

# 检查 fork 是否频繁（bgsave 会 fork）
# 查看 /var/log/redis/redis.log 中的 bgsave 记录
```

### 3.4 数据丢失排查

```bash
# 检查持久化是否工作
redis-cli INFO persistence

# 检查 AOF 文件完整性
redis-check-aof /var/lib/redis/appendonly.aof

# 检查 RDB 文件完整性
redis-check-rdb /var/lib/redis/dump.rdb
```

## 四、压测与监控

### 4.1 基准测试

在调优前后使用 `redis-benchmark` 对比性能变化：

```bash
# 测试 QPS（每秒请求数）
redis-benchmark -h 127.0.0.1 -p 6379 -c 50 -n 100000

# 测试不同数据大小
redis-benchmark -d 1024 -t set,get,incr,lpush

# 测试 Pipeline 效果
redis-benchmark -P 16 -t get,set

# 测试特定命令
redis-benchmark -t ping,set,get -q
```

Pipeline 批量操作可以显著提升吞吐量。在 Node.js 中：

```javascript
const pipeline = redis.pipeline();
for (let i = 0; i < 1000; i++) {
  pipeline.set(`key:${i}`, `value:${i}`);
}
const results = await pipeline.exec();
```

对比单条命令循环写入，Pipeline 可以提升 5-10 倍吞吐量。

### 4.2 监控指标

生产环境至少需要监控以下指标：

```bash
# 连接数
redis-cli INFO clients | grep -E "connected_clients|blocked_clients"

# 内存
redis-cli INFO memory | grep -E "used_memory_human|used_memory_rss_human|maxmemory"

# 命中率
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"

# 命令统计
redis-cli INFO commandstats | head -20

# 持久化
redis-cli INFO persistence | grep -E "rdb_last_save_time|aof_last_rewrite_time"
```

**命中率计算**：`keyspace_hits / (keyspace_hits + keyspace_misses) * 100%`。命中率长期低于 80% 说明缓存策略有问题，需要调整 `maxmemory-policy` 或检查 key 设计。

### 4.3 Prometheus 集成

使用 `redis_exporter` 接入 Prometheus 监控体系：

```bash
# 启动 exporter
./redis_exporter \
  -redis.addr 127.0.0.1:6379 \
  -redis.password "$REDIS_PASSWORD" \
  -web.listen-address :9121
```

对应的 Prometheus 告警规则：

```yaml
groups:
  - name: redis
    rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 1m
        annotations:
          summary: "Redis instance {{ $labels.instance }} is down"

      - alert: RedisMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
        for: 5m
        annotations:
          summary: "Redis memory usage > 85% on {{ $labels.instance }}"

      - alert: RedisHitRateLow
        expr: rate(redis_keyspace_hits_total[5m]) / (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) < 0.8
        for: 10m
        annotations:
          summary: "Redis cache hit rate < 80% on {{ $labels.instance }}"

      - alert: RedisReplicationLag
        expr: redis_connected_slaves_lag_offset > 1000
        for: 1m
        annotations:
          summary: "Redis replication lag > 1000 bytes on {{ $labels.instance }}"
```

## 五、总结

Redis 安全加固和性能优化不是一次性工作，应该作为日常运维的一部分持续关注。核心要点：

1. **安全先于功能**：ACL + 绑定 IP + 禁用危险命令，三步做到基本防护。不要等到被入侵才想起改配置
2. **监控驱动优化**：通过慢查询和内存分析找到瓶颈，而不是盲目调参。每次配置变更都应有数据支撑
3. **测试环境验证**：任何配置变更先在测试环境压测，确认无副作用。`redis-benchmark` 是你的好朋友
4. **定期更新**：Redis 每个版本都有安全修复和性能改进，关注 Release Notes 及时升级

**安全检查清单**：

| 检查项 | 检查方法 | 推荐值 |
|--------|----------|--------|
| 绑定地址 | `CONFIG GET bind` | 127.0.0.1 或内网 IP |
| 密码/ACL | `INFO` 看 `requirepass` 或 `ACL LIST` | 已设置 |
| 危险命令 | `CONFIG GET rename-command` | 危险命令已禁用 |
| 运行用户 | `ps aux \| grep redis` | 专用 redis 用户 |
| 内存限制 | `CONFIG GET maxmemory` | 已设置 |
| 持久化策略 | `CONFIG GET save` | 符合业务需求 |
| 内核参数 | `sysctl vm.swappiness` | ≤ 1 |
| THP 状态 | `cat /sys/kernel/mm/transparent_hugepage/enabled` | never / always [never] |
| 慢查询 | `SLOWLOG LEN` | 没有异常慢查询 |
| 最后更新 | `INFO server` 看 `redis_version` | 最新稳定版 |

把这份清单加到你的运维巡检脚本里，每周跑一次，比出了问题再补救强得多。