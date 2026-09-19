---
layout: post
title: "etcd 集群运维与故障恢复实战指南"
date: 2026-09-19 09:00:00 +0800
categories: [开发, 网络技术]
tags: [etcd, kubernetes, distributed-systems, raft, backup, disaster-recovery, database, key-value-store, ops, cluster-management]
---

## 为什么需要深入理解 etcd

etcd 是 Kubernetes 的大脑——集群状态、配置、Secret、Service 信息全都存储在 etcd 中。它基于 Raft 共识算法，提供强一致性的键值存储。K8s 集群不可用，绝大多数情况下是 etcd 出了问题。

本文从实战出发，覆盖 etcd 的安装部署、日常运维、备份恢复、性能调优、故障处理五个核心环节。所有命令均为真实可用，建议读者在测试环境跟着跑一遍。

## 一、安装与集群部署

### 1.1 单节点快速启动

```bash
# 下载 etcd
wget https://github.com/etcd-io/etcd/releases/download/v3.5.15/etcd-v3.5.15-linux-amd64.tar.gz
tar xzf etcd-v3.5.15-linux-amd64.tar.gz
sudo cp etcd-v3.5.15-linux-amd64/etcd* /usr/local/bin/

# 启动单节点
etcd --name node1 \
  --data-dir /var/lib/etcd \
  --listen-client-urls http://0.0.0.0:2379 \
  --advertise-client-urls http://192.168.1.10:2379 \
  --listen-peer-urls http://0.0.0.0:2380 \
  --initial-advertise-peer-urls http://192.168.1.10:2380 \
  --initial-cluster node1=http://192.168.1.10:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster-state new
```

### 1.2 三节点集群部署

生产环境至少三节点，奇数节点保证 Raft 能够选主容忍 (N-1)/2 个节点故障。

```bash
# 节点 1（10.0.0.1）
etcd --name infra0 \
  --data-dir /var/lib/etcd \
  --listen-client-urls https://10.0.0.1:2379 \
  --advertise-client-urls https://10.0.0.1:2379 \
  --listen-peer-urls https://10.0.0.1:2380 \
  --initial-advertise-peer-urls https://10.0.0.1:2380 \
  --initial-cluster infra0=https://10.0.0.1:2380,infra1=https://10.0.0.2:2380,infra2=https://10.0.0.3:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=/etc/etcd/ca.pem \
  --cert-file=/etc/etcd/server.pem --key-file=/etc/etcd/server-key.pem \
  --peer-client-cert-auth --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-cert-file=/etc/etcd/peer.pem --peer-key-file=/etc/etcd/peer-key.pem
```

其余两个节点只需修改 `--name`、`--*-listen-urls`、`--*-advertise-urls` 中的 IP 即可。

**关键参数说明**：

| 参数 | 作用 | 建议值 |
|------|------|--------|
| `--snapshot-count` | 触发快照的事务数 | 100000（默认） |
| `--heartbeat-interval` | Leader 心跳间隔(ms) | 100（默认） |
| `--election-timeout` | 选举超时(ms) | 1000（默认） |
| `--auto-compaction-retention` | 自动压缩保留时间 | 根据数据量设 8h/72h/revision |
| `--quota-backend-bytes` | 后端存储配额 | 8GB（默认 2GB，K8s 建议至少 8GB） |

## 二、日常运维命令

### 2.1 集群健康检查

```bash
# 检查集群成员
etcdctl member list -w table

# 检查端点健康状态
etcdctl endpoint health -w table

# 查看各端点状态详情
etcdctl endpoint status -w table

# 查看当前 Leader
etcdctl endpoint status --cluster -w table | awk 'NR>1{print $4}'
```

输出示例：

```
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|     ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEARNER | RAFT TERM  | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://10.0.0.1:2379 | 8e9e05c52164694d | 3.5.15  | 25 MB   | false     | 27         | 1023456   | 1023456     |                    |        |
+-------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

### 2.2 键值操作

```bash
# 写入键值
etcdctl put /app/config '{"port": 8080, "debug": false}'

# 读取键值
etcdctl get /app/config

# 带版本号读取
etcdctl get /app/config -w json

# 前缀查询（K8s 大量使用）
etcdctl get /registry/pods/default --prefix --limit=5

# 删除
etcdctl del /app/config

# 监听变化（Watch）
etcdctl watch /app/config
```

### 2.3 租约（Lease）管理

```bash
# 创建 60 秒租约
etcdctl lease grant 60

# 绑定键到租约
etcdctl put --lease=694d710a7a8c7f4f /tmp/session 'alive'

# 续约
etcdctl lease keep-alive 694d710a7a8c7f4f

# 查看租约
etcdctl lease list
etcdctl lease timetolive 694d710a7a8c7f4f

# 租约到期后，绑定的键自动删除
```

## 三、备份与恢复

这是运维中最关键的技能。etcd 提供了三种备份方式。

### 3.1 快照备份（Snapshot）

```bash
# 创建快照（推荐：凌晨低峰期执行）
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/healthcheck-client.pem \
  --key=/etc/etcd/healthcheck-client-key.pem \
  snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db

# 查看快照状态
etcdctl snapshot status /backup/etcd-snapshot-20260919.db -w table
```

**快照自动化的最佳实践**：

```bash
# /usr/local/bin/etcd-backup.sh
#!/bin/bash
BACKUP_DIR="/backup/etcd"
SNAPSHOT_FILE="$BACKUP_DIR/snapshot-$(date +%Y%m%d-%H%M%S).db"
RETENTION_DAYS=30

export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/etcd/ca.pem
export ETCDCTL_CERT=/etc/etcd/healthcheck-client.pem
export ETCDCTL_KEY=/etc/etcd/healthcheck-client-key.pem

etcdctl snapshot save "$SNAPSHOT_FILE"
echo "Snapshot saved: $SNAPSHOT_FILE"

# 压缩
gzip "$SNAPSHOT_FILE"
echo "Compressed: $SNAPSHOT_FILE.gz"

# 删除超过保留天数的旧快照
find "$BACKUP_DIR" -name "snapshot-*.db.gz" -mtime +$RETENTION_DAYS -delete
```

配合 crontab：

```cron
0 3 * * * /usr/local/bin/etcd-backup.sh
```

### 3.2 从快照恢复

两种情况：恢复到现有集群，或重建新集群。

**场景一：单节点数据损坏恢复**

```bash
# 1. 停止 etcd
systemctl stop etcd

# 2. 备份损坏的数据目录（保留现场）
mv /var/lib/etcd /var/lib/etcd.corrupted

# 3. 恢复数据目录
etcdctl snapshot restore /backup/etcd-snapshot-20260919.db \
  --name infra0 \
  --data-dir /var/lib/etcd \
  --initial-cluster infra0=https://10.0.0.1:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://10.0.0.1:2380

# 4. 启动 etcd
systemctl start etcd

# 5. 验证成员是否加入集群
etcdctl member list -w table
```

**场景二：集群脑裂/多数节点丢失——重建集群**

```bash
# 在幸存的节点上执行
# 导出快照
etcdctl snapshot save /tmp/etcd-snapshot-recovery.db

# 对每个新节点分别 restore
for name in node1 node2 node3; do
  etcdctl snapshot restore /tmp/etcd-snapshot-recovery.db \
    --name "$name" \
    --data-dir /var/lib/etcd \
    --initial-cluster node1=http://10.0.0.1:2380,node2=http://10.0.0.2:2380,node3=http://10.0.0.3:2380 \
    --initial-cluster-token etcd-cluster-recovery \
    --initial-advertise-peer-urls "http://10.0.0.${i}:2380"
done
```

> 注意：`--initial-cluster-token` 必须与原始集群不同，否则 Raft 日志会冲突。

### 3.3 定期备份验证

只备份不验证等于没备份。用 cron 定期检查快照完整性：

```bash
# /usr/local/bin/etcd-verify-backup.sh
#!/bin/bash
LATEST=$(ls -t /backup/etcd/snapshot-*.db.gz | head -1)

if [ -z "$LATEST" ]; then
  echo "ERROR: No backup found!"
  exit 1
fi

gunzip -c "$LATEST" > /tmp/verify-snapshot.db

# 检查快照是否可以读取
etcdctl snapshot status /tmp/verify-snapshot.db || {
  echo "ERROR: Backup verification failed: $LATEST"
  exit 2
}

# 检查损坏：临时启动一个 etcd 实例加载快照
ETCD_UNSUPPORTED_ARCH=arm64 etcd --data-dir /tmp/etcd-verify \
  --force-new-cluster --initial-cluster-token verify-token-1 \
  --initial-cluster default=http://localhost:2380 \
  --initial-advertise-peer-urls http://localhost:2380 \
  --listen-peer-urls http://localhost:2380 \
  --listen-client-urls http://localhost:2381 &
VERIFY_PID=$!
sleep 3

# 用 etcdctl 读取几个关键键验证数据完整性
ETCDCTL_ENDPOINTS=http://localhost:2381 etcdctl get / --prefix --limit=1
if [ $? -eq 0 ]; then
  echo "OK: Backup verified successfully: $LATEST"
else
  echo "ERROR: Backup data corrupted: $LATEST"
fi

kill $VERIFY_PID 2>/dev/null
rm -rf /tmp/etcd-verify /tmp/verify-snapshot.db
```

## 四、性能调优

### 4.1 磁盘是关键

etcd 对磁盘延迟极其敏感，Raft 每次写入都要 fsync。磁盘 I/O 是首要瓶颈。

```bash
# 测量磁盘 fsync 性能（应 < 10ms）
fio --name=etcd-write-test \
  --ioengine=sync --rw=randwrite \
  --bs=4k --numjobs=1 --size=256m \
  --runtime=60 --time_based --fsync=1 \
  --directory=/var/lib/etcd

# 看 p99 fsync 延迟：如超过 10ms，说明磁盘跟不上
```

**磁盘选型建议**：

| 存储类型 | 建议 | 说明 |
|---------|------|------|
| 本地 NVMe SSD | 最适合 | 延迟 < 1ms，吞吐量高 |
| 本地 SATA SSD | 可接受 | 延迟 1-3ms |
| 网络存储 (NFS/iSCSI) | 不推荐 | 网络抖动导致 Raft 超时 |
| EBS gp3 (K8s on AWS) | 需配置 IOPS | 建议 min 3000 IOPS |

**IOPS 不足的典型症状**：

```
[WARNING] low read capacity, waiting for readIndex to complete
[WARNING] apply request took too long (152ms)
```

### 4.2 etcd 配置优化

```ini
# /etc/etcd/etcd.conf.yaml
name: infra0
data-dir: /var/lib/etcd

# 监听所有接口（通过防火墙限制）
listen-client-urls: https://0.0.0.0:2379
advertise-client-urls: https://10.0.0.1:2379

# 心跳和选举（不要改动默认值除非遇到问题）
heartbeat-interval: 100
election-timeout: 1000

# 自动压缩——每 5000 个 revision 压缩一次
auto-compaction-mode: revision
auto-compaction-retention: "5000"

# 后端配额——K8s 集群建议 8GB
quota-backend-bytes: 8589934592

# 单次请求最大大小（K8s 默认 1.5MB 限制）
max-request-bytes: 1572864

# 客户端连接限制
max-concurrent-streams: 1000
```

### 4.3 数据库压缩（Defrag）

etcd 的 BoltDB 存储引擎不会自动释放已删除的空间。长时间运行的集群需要手动压缩：

```bash
# 检查当前数据库碎片
etcdctl endpoint status -w table | awk 'NR>1{print $4, $5}'

# 对不同端点执行碎片整理（各端点的 follower 优先，最后做 leader）
etcdctl --endpoints=https://10.0.0.2:2379 defrag

# 压缩后检查空间释放情况
du -sh /var/lib/etcd/member/snap/db
```

在 K8s 中运维 etcd 的场景：

```bash
# K8s 托管 etcd 示例（kubeadm）
kubectl -n kube-system exec etcd-control-plane -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    defrag
```

**Defrag 建议周期**：每月一次低峰期执行。中大型集群（>1000 个 Pod）建议每两周一次。

### 4.4 数据库大小监控

```bash
# 查看顶部消耗空间的键空间
etcdctl get / --prefix --keys-only | \
  awk -F '/' '{for(i=1;i<NF;i++) printf "%s/",$i; print ""}' | \
  sort | uniq -c | sort -rn | head -10
```

在 K8s 集群中，`/registry/events` 通常是最大数据源。可以配置事件保留策略减少 etcd 负载。

## 五、常见故障处理

### 5.1 数据库空间满（NOSPACE）

**现象**：etcd 返回 `etcdserver: mvcc: database space exceeded` 或 `NOSPACE` 错误。

**处理步骤**：

```bash
# 1. 检查空间状态
etcdctl endpoint status -w table

# 2. 手动触发压缩，释放旧版本
etcdctl compact $(etcdctl endpoint status -w json | jq -r '.[0].Status.header.revision')

# 3. 执行碎片整理释放空间
etcdctl defrag

# 4. 如果仍然 NOSPACE，需要调大配额
# 设置环境变量 ETCD_QUOTA_BACKEND_BYTES=8589934592（8GB），重启 etcd
```

**预防措施**：设置合理的 `auto-compaction-retention`，监控 db size，阈值告警（建议 75% 配额触发 warning，90% 触发 critical）。

### 5.2 Leader 频繁切换

**现象**：etcd 日志中反复出现 `leader changed`，或 `etcdctl endpoint health` 显示异常。

**原因排查**：

```bash
# 1. 检查磁盘延迟
iostat -x 1 10

# 2. 检查网络延迟（各节点之间）
mtr -r 10.0.0.2
mtr -r 10.0.0.3

# 3. 检查 etcd 日志
journalctl -u etcd --since "10 minutes ago" | grep -iE "(leader|election|timeout|slow)"

# 4. 检查心跳延迟
etcdctl --endpoints=https://127.0.0.1:2379 check perf
```

**常见原因**：

| 原因 | 解决方案 |
|------|---------|
| 磁盘 fsync 延迟过高 | 换 SSD，调整内核参数 vm.dirty_ratio |
| 网络延迟 > 50ms | 确保 etcd 节点在同一机房，万兆网络 |
| 节点时钟不同步 | 统一 NTP 服务器，配置 chrony 或 ntpd |
| CPU 资源争抢 | 给 etcd 分配专用 CPU core（cpuset） |

### 5.3 成员移除与替换

```bash
# 查看当前成员列表
etcdctl member list -w table

# 移除故障成员
etcdctl member remove 8e9e05c52164694d

# 添加新成员
etcdctl member add new-node \
  --peer-urls=https://10.0.0.4:2380

# 在新节点上用 --initial-cluster-state=existing 启动
etcd --name new-node \
  --initial-cluster-state existing \
  --initial-cluster infra0=https://10.0.0.1:2380,infra1=https://10.0.0.2:2380,new-node=https://10.0.0.4:2380 \
  --advertise-peer-urls=https://10.0.0.4:2380 \
  --listen-peer-urls=https://0.0.0.0:2380 \
  --initial-advertise-peer-urls=https://10.0.0.4:2380
```

### 5.4 读请求超时排查

```bash
# 开启详细日志定位慢请求
etcd --debug --logger=zap

# 观察慢请求日志
# [core] request ... took too long (123ms) to execute

# 检查索引滞后
etcdctl endpoint status --cluster -w table
# RAFT INDEX 和 RAFT APPLIED INDEX 相差过大 = follower 追赶中

# 如果 follower 落后严重，检查网络或磁盘
# 可传递 ---experimental-compact-hash-check-enabled=false 临时跳过一致性检查加速追赶
```

## 六、K8s 环境中的 etcd 最佳实践

### 6.1 从 K8s 控制面节点操作 etcd

```bash
# kubeadm 部署的集群
kubectl -n kube-system exec etcd-control-plane -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    endpoint health -w table

# 查看 etcd pod 日志
kubectl -n kube-system logs etcd-control-plane
```

### 6.2 etcd 数据提取

从 etcd 直接读取 K8s 资源数据（调试时极有用）：

```bash
# 查看所有集群资源
etcdctl get /registry/ --prefix --keys-only

# 提取特定 Pod
etcdctl get /registry/pods/default/my-nginx

# 提取特定 Node
etcdctl get /registry/minions/node1

# 解码 K8s 对象的 protobuf（需要先确认存储格式）
# K8s v1.26+ 默认使用 protobuf 存储
ETCDCTL_API=3 etcdctl get /registry/configmaps/kube-system/kube-proxy \
  | tail -n +2 | protoc --decode_raw
```

### 6.3 监控告警

Prometheus 查询语句推荐：

```promql
# etcd 数据库大小
etcd_mvcc_db_total_size_in_bytes

# 磁盘 fsync 延迟（p99）
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))

# Leader 变化次数
rate(etcd_server_leader_changes_seen_total[5m])

# 慢请求
rate(etcd_server_slow_read_indexes_total[5m])

# 未压缩的 revision 数
etcd_mvcc_put_total - etcd_mvcc_delete_total
推荐告警阈值：

| 指标 | 告警阈值 | 严重等级 |
|------|---------|---------|
| db size > 6.4 GB | Warning | 接近配额 8GB |
| fsync latency p99 > 10s | Critical | 磁盘问题 |
| leader changes > 0.05/s | Critical | 集群不稳定 |
| slow read index > 0 | Warning | 磁盘 I/O 瓶颈 |
| db size growth > 1GB/天 | Info | 规划扩容或调整压缩 |

## 七、总结

etcd 运维的核心要点就三条：

1. **备份验证**——定时快照 + 自动化验证，确保故障时能恢复
2. **磁盘性能**——NVMe SSD + 本地存储，定期 iostat 监控
3. **定期压缩**——auto-compaction 避免空间爆满，季度 defrag 回收碎片

掌握这些命令和排查思路，K8s 集群的 etcd 层基本不会出大问题。遇到故障时，记住最稳妥的恢复流程是：先 snapshot、再分析、后操作。没有快照之前，不要执行任何具有破坏性的命令。

如果你在维护生产 K8s 集群，建议把本文的 shell 脚本写成 cron job，并且至少做一次从裸金属到恢复全集群的故障演练——真到出事的时候，肌肉记忆比查文档快得多。