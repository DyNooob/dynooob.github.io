---
layout: post
title: "Prometheus 监控与告警实战指南"
date: 2026-08-18
categories: [开发]
tags: [Prometheus, 监控, Alertmanager, Grafana, PromQL, 告警, 运维, SRE, 可观测性]
---

## 为什么需要 Prometheus

在微服务架构和容器化部署大行其道的今天，监控系统已经从"锦上添花"变成了"基础设施"。没有监控，你就是在盲人摸象：服务挂了不知道，磁盘满了不知道，API 延迟飙升了也不知道。

Prometheus 之所以成为云原生监控的事实标准，靠的不是花哨的功能，而是几个扎实的设计选择：

- **拉模式（Pull）**：Prometheus 主动去目标抓取数据，而不是等目标推送。这让你能精确控制采集频率，也避免了被攻击者用推送数据灌爆监控系统。
- **多维数据模型**：每个指标带一组标签（label），可以灵活地按任意维度聚合、切片。
- **PromQL**：一套功能强大的查询语言，能做比率、聚合、预测、异常检测。
- **内置告警**：通过 Alertmanager 实现告警的分组、抑制、静默和路由。

这篇文章不讲空洞的概念，直接从安装开始，一步步搭出一套可用的监控+告警系统。

## 架构概览

Prometheus 生态的四个核心组件：

```
+----------------+     scrape     +------------------+
|  Node Exporter |<---------------+                  |
|  (系统指标)     |                |                  |
+----------------+                |                  |
                                  |   Prometheus     |
+----------------+     scrape     |   Server         |
|  Custom Exporter|<--------------+  (存储+查询)      |
|  (业务指标)     |                |                  |
+----------------+                |                  |
                                  +--------+---------+
                                           |
                                    alert  |  rules
                                           v
+----------------+     routes     +------------------+
|  Alertmanager  |<--------------+                  |
|  (告警管理)     |                |   Grafana        |
+----------------+                |  (可视化)         |
        |                         +------------------+
        v
  通知渠道 (Email/Slack/Webhook)
```

## 一、Docker Compose 快速部署

用 Docker Compose 是最快的上手方式，生产环境也完全可以在此基础上扩展。

```yaml
# docker-compose.yml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:v2.54.0
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=30d"
      - "--web.console.libraries=/etc/prometheus/console_libraries"
      - "--web.console.templates=/etc/prometheus/consoles"
      - "--web.enable-lifecycle"  # 允许热加载配置
    ports:
      - "9090:9090"
    restart: unless-stopped
    networks:
      - monitor

  node_exporter:
    image: prom/node-exporter:v1.8.0
    container_name: node_exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    ports:
      - "9100:9100"
    restart: unless-stopped
    networks:
      - monitor

  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"
    restart: unless-stopped
    networks:
      - monitor

  grafana:
    image: grafana/grafana:11.0.0
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    restart: unless-stopped
    networks:
      - monitor

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitor:
    driver: bridge
```

## 二、配置 Prometheus 抓取目标

核心配置文件 `prometheus.yml`：

```yaml
# prometheus.yml
global:
  scrape_interval: 15s      # 全局抓取间隔
  evaluation_interval: 15s   # 规则评估间隔
  scrape_timeout: 10s        # 抓取超时

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  # 监控 Prometheus 自身
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # 监控宿主机
  - job_name: "node"
    scrape_interval: 30s
    static_configs:
      - targets: ["node_exporter:9100"]

  # 监控容器（cAdvisor）
  - job_name: "cadvisor"
    scrape_interval: 30s
    static_configs:
      - targets: ["cadvisor:8080"]

  # 监控 API 服务（示例）
  - job_name: "api-server"
    scrape_interval: 10s
    metrics_path: "/metrics"
    scheme: http
    static_configs:
      - targets: ["api-server:8080"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: "(.*):.*"
        replacement: "${1}"
```

### 关键配置项说明

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `scrape_interval` | 抓取间隔 | 15s（默认），高负载服务可设为 10s |
| `evaluation_interval` | 规则评估间隔 | 必须与告警规则匹配，通常 15s |
| `scrape_timeout` | 抓取超时 | 小于 scrape_interval，通常设为 10s |
| `storage.tsdb.retention.time` | 数据保留时间 | 根据磁盘容量，通常 15-30d |

**关于抓取间隔的一个经验**：不要盲目追求短间隔。每 5s 抓一次和每 15s 抓一次，数据量相差 3 倍，但诊断问题时的信息量相差不大。默认 15s 对大多数场景足够。

## 三、PromQL 查询语言实战

PromQL 是 Prometheus 的查询语言，也是用好 Prometheus 的核心能力。这里不讲所有语法，只说最常用的几个模式。

### 3.1 基础查询

```promql
# 直接查询指标
node_cpu_seconds_total

# 带标签过滤
node_cpu_seconds_total{mode="idle"}

# 正则匹配标签
node_cpu_seconds_total{mode=~"user|system|iowait"}

# 取最新值
node_load1
```

### 3.2 速率与比率

```promql
# CPU 使用率（最常用的查询）
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 磁盘使用率
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)

# 网络吞吐量
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])
```

### 3.3 聚合操作

```promql
# 按实例平均 CPU 使用率
avg by(instance) (rate(node_cpu_seconds_total{mode!="idle"}[5m])) * 100

# 所有实例的 CPU 使用率总和
sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) * 100

# 百分比分布（分位数）
histogram_quantile(0.95, sum by(le) (rate(http_request_duration_seconds_bucket[5m])))
```

### 3.4 预测与趋势

PromQL 的 `predict_linear` 函数非常实用，可以用线性回归预测未来趋势：

```promql
# 预测磁盘 6 小时后是否满
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 6*3600) < 0

# 预测内存 4 小时后是否耗尽
predict_linear(node_memory_MemAvailable_bytes[4h], 4*3600) < 0
```

## 四、Recording Rules 性能优化

Recording rules 是 Prometheus 里一个被很多人忽略但极其有用的功能。它的核心思想是：**把复杂的查询提前算好存起来，查询时直接读结果**。

### 为什么需要 Recording Rules

假设你有一个 Dashboard，每次刷新都要执行下面这个复杂的查询：

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

如果 Dashboard 上有 10 个面板，每个面板刷新一次就要执行一遍。而在大范围时间窗口（如 7d）上执行这个查询，数据量很大，响应会很慢。

Recording rules 把这个问题解决了：**Prometheus 每隔 evaluation_interval 自动计算一次，把结果存储为新的时间序列**。

### 配置示例

```yaml
# rules/recording_rules.yml
groups:
  - name: node_recording_rules
    interval: 30s
    rules:
      - record: node:cpu_usage_avg:rate5m
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

      - record: node:memory_usage_ratio
        expr: 1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

      - record: node:disk_usage_ratio:mountpoint
        expr: 1 - node_filesystem_avail_bytes / node_filesystem_size_bytes
        labels:
          metric_type: storage

      - record: node:network_receive_rate:5m
        expr: sum by(instance) (rate(node_network_receive_bytes_total[5m]))

      - record: node:network_transmit_rate:5m
        expr: sum by(instance) (rate(node_network_transmit_bytes_total[5m]))
```

### Recording Rule 命名规范

Prometheus 社区推荐了一套命名模式：

```
层级:类别:单位:聚合方式
```

- `node:cpu_usage_avg:rate5m` — 节点级别，CPU 使用率，平均，5m 速率
- `node:memory_usage_ratio` — 节点级别，内存使用率，比率

这样团队里的人一看就知道这个指标是什么、怎么算的。

### 使用 Recording Rules 后的查询对比

```promql
# 之前
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 之后
node:cpu_usage_avg:rate5m
```

简单直接，Dashboard 的加载速度从秒级降到毫秒级。对于有大量 Dashboard 和 Grafana 用户的团队，Recording rules 能显著降低 Prometheus 的查询负载。

## 五、Alerting Rules 告警规则

告警规则是 Prometheus 监控的灵魂。没有告警的监控只是"事后诸葛亮"——你只是在看历史数据。

### 5.1 节点告警规则

```yaml
# rules/alerting_rules.yml
groups:
  - name: node_alerts
    interval: 30s
    rules:
      - alert: NodeDown
        expr: up{job="node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "节点 {{ $labels.instance }} 已离线"
          description: "{{ $labels.instance }} 已经离线超过 1 分钟"

      - alert: HighCpuUsage
        expr: node:cpu_usage_avg:rate5m > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "节点 {{ $labels.instance }} CPU 使用率过高"
          description: "CPU 使用率当前为 {{ $value | humanize }}%，持续超过 5 分钟"

      - alert: HighMemoryUsage
        expr: node:memory_usage_ratio > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "节点 {{ $labels.instance }} 内存使用率过高"
          description: "内存使用率当前为 {{ $value | humanizePercentage }}，持续超过 5 分钟"

      - alert: DiskSpaceLow
        expr: node:disk_usage_ratio:mountpoint{mountpoint="/"} > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "节点 {{ $labels.instance }} 磁盘空间不足"
          description: "磁盘使用率 {{ $value | humanizePercentage }}，超过 85% 阈值"

      - alert: DiskWillFillIn6Hours
        expr: predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[6h], 6*3600) < 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "节点 {{ $labels.instance }} 磁盘将在 6 小时内写满"
          description: "根据当前增长趋势预测，磁盘将在 6 小时内写满"
```

### 5.2 服务告警规则

```yaml
  - name: service_alerts
    interval: 15s
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
          * 100 > 5
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "服务 {{ $labels.job }} 错误率过高"
          description: "5xx 错误率 {{ $value | humanize }}%，超过 5% 阈值"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, sum by(le) (rate(http_request_duration_seconds_bucket{job="api-server"}[5m])))
          > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "API 响应延迟过高"
          description: "P95 延迟 {{ $value }}s，超过 1s 阈值"

      - alert: TargetDown
        expr: up{job!~"prometheus|node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "抓取目标 {{ $labels.instance }} 不可达"
          description: "目标 {{ $labels.job }}/{{ $labels.instance }} 已离线超过 1 分钟"
```

### 5.3 for 参数的理解

很多人不理解 `for` 的作用。看这个例子：

```yaml
- alert: HighCpuUsage
  expr: node:cpu_usage_avg:rate5m > 90
  for: 5m        # <-- 持续 5 分钟才触发
```

`for: 5m` 的意思是：**CPU 使用率超过 90% 并持续达到 5 分钟，才触发告警**。如果 CPU 只是短暂飙到 95% 然后回落，不会告警。

这个参数用来过滤"毛刺"——短暂的异常波动通常不需要处理，持续的问题才需要关注。对于不同的指标，合适的 `for` 值不同：

| 指标类型 | 建议 for 值 | 原因 |
|---------|------------|------|
| 服务离线 | 1m | 尽快响应 |
| CPU/内存 | 5m | 容忍短时波动 |
| 磁盘空间 | 5m+ | 变化缓慢，不需要秒级响应 |
| 错误率 | 3m | 需要平衡及时性和误报 |

## 六、Alertmanager 告警管理

Prometheus 负责"判断"是否要告警，Alertmanager 负责"发送"告警并把告警管好。

### 6.1 配置 Alertmanager

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: "smtp.example.com:587"
  smtp_from: "alert@example.com"
  smtp_auth_username: "alert@example.com"
  smtp_auth_password: "your-password"

route:
  group_by: ["alertname", "severity"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "default"

  routes:
    - match:
        severity: critical
      receiver: "pagerduty"
      repeat_interval: 30m

    - match:
        severity: warning
      receiver: "email-notify"
      repeat_interval: 4h

receivers:
  - name: "default"
    email_configs:
      - to: "team@example.com"

  - name: "email-notify"
    email_configs:
      - to: "team@example.com"
        headers:
          subject: "[WARNING] {{ .GroupLabels.alertname }}"

  - name: "pagerduty"
    pagerduty_configs:
      - routing_key: "your-pagerduty-key"
        severity: critical
```

### 6.2 告警分组与抑制

Alertmanager 的三个核心概念：

**分组（Grouping）**：把类似的告警合并成一条通知。比如 10 台机器同时 CPU 高，你不会想收 10 封邮件，而是一封邮件列出 10 台机器。

```yaml
route:
  group_by: ["alertname", "severity"]
  group_wait: 30s     # 等 30s，收集同一组内的告警
  group_interval: 5m  # 同一组告警，5 分钟内不再重复发送
```

**抑制（Inhibition）**：如果某个节点宕机了，那这个节点上的所有服务不可达告警都是"噪音"。抑制功能可以自动忽略这些衍生告警：

```yaml
inhibit_rules:
  - source_match:
      alertname: "NodeDown"
      severity: critical
    target_match:
      severity: ">=warning"
    equal: ["instance"]
```

**静默（Silencing）**：在维护窗口期内，手动屏蔽某些告警。可以通过 Alertmanager 的 Web UI 或 `amtool` 命令行工具设置。

## 七、Grafana 可视化集成

装好 Prometheus 后，Grafana 是标准的数据可视化方案。

### 7.1 添加数据源

1. 访问 `http://localhost:3000`（默认账号密码 admin/admin）
2. 进入 Configuration → Data Sources → Add data source
3. 选择 Prometheus，URL 填 `http://prometheus:9090`
4. 点击 Save & Test

### 7.2 导入现成 Dashboard

Grafana 社区有大量现成的 Dashboard，可以直接导入使用：

- **Node Exporter Full** (ID: 1860) — 系统监控面板
- **Docker Monitoring** (ID: 893) — 容器监控面板
- **Prometheus 2.0 Stats** (ID: 3662) — Prometheus 自身状态

```bash
# 通过 API 导入 Dashboard
curl -X POST "http://admin:admin@localhost:3000/api/dashboards/import" \
  -H "Content-Type: application/json" \
  -d '{
    "dashboard": {
      "id": null,
      "title": "Import Example",
      "panels": [],
      "timezone": "browser"
    },
    "overwrite": true
  }'
```

## 八、生产环境最佳实践

### 8.1 配置热加载

修改 Prometheus 配置后不需要重启服务：

```bash
# 方式一：API 热加载（需要 --web.enable-lifecycle）
curl -X POST http://localhost:9090/-/reload

# 方式二：发送 SIGHUP 信号
kill -HUP $(pgrep prometheus)
```

### 8.2 数据持久化与备份

```bash
# 使用 promtool 检查 TSDB 健康状态
promtool tsdb analyze /prometheus

# 备份 TSDB 数据
tar -czf prometheus-backup-$(date +%Y%m%d).tar.gz /prometheus/

# 限制数据保留时间
--storage.tsdb.retention.time=30d
--storage.tsdb.retention.size=50GB
```

### 8.3 联邦集群

当监控规模超过单台 Prometheus 的处理能力时，可以使用联邦集群（Federation）分层采集：

```yaml
# 上层 Prometheus 从下层 Prometheus 抓取聚合数据
scrape_configs:
  - job_name: "federate"
    scrape_interval: 60s
    honor_labels: true
    metrics_path: "/federate"
    params:
      match[]:
        - '{__name__=~"node:.*"}'
        - '{__name__=~"instance:.*"}'
    static_configs:
      - targets:
          - "prometheus-dc1:9090"
          - "prometheus-dc2:9090"
```

### 8.4 告警接收者选择

| 接收方式 | 适用场景 | 配置复杂度 |
|---------|---------|-----------|
| Email | 非紧急通知、日报 | 低 |
| Slack/钉钉/飞书 | 团队即时通知 | 中 |
| PagerDuty | 值班轮换、升级策略 | 高 |
| Webhook | 自定义集成 | 中 |
| Telegram | 个人通知 | 低 |

### 8.5 常见问题排查

**Q: 数据不显示怎么办？**
```bash
# 检查 Prometheus 自身状态
curl http://localhost:9090/api/v1/status/runtimeinfo
curl http://localhost:9090/api/v1/status/buildinfo

# 检查目标状态
curl http://localhost:9090/api/v1/targets

# 检查规则
curl http://localhost:9090/api/v1/rules
```

**Q: Prometheus 内存占用过高？**
- 检查 `--storage.tsdb.retention.time` 和 `--storage.tsdb.retention.size`
- 减少 scrape_interval（从 15s 改为 30s）
- 减少不必要的 label 数量
- 使用 Recording rules 减少查询时的计算量

**Q: 告警没收到？**
- 检查 Alertmanager 的 Web UI `http://localhost:9093/#/alerts`
- 用 `amtool` 测试告警路由：`amtool config-check alertmanager.yml`
- 检查 Prometheus 的 `Alertmanager` 目标是否 UP

## 总结

本文从零搭建了一套完整的 Prometheus 监控体系，包括：

1. **数据采集**：通过 Node Exporter 和自定义 Exporter 收集指标
2. **存储与查询**：Prometheus 时序数据库 + PromQL 灵活查询
3. **性能优化**：Recording rules 预计算结果，降低查询负载
4. **告警管理**：Alerting rules 定义告警条件，Alertmanager 负责路由和通知
5. **可视化**：Grafana 提供 Dashboard 展示

监控系统建设的核心思路是：**先有数据，再告警，再优化**。不要一开始就追求"完美"——先搭起来，让数据流进来，然后根据实际需求逐步完善告警规则和 Dashboard。

下一步可以探索的方向：Thanos（长期存储）、Prometheus Operator（Kubernetes 运维）、OpenTelemetry（统一遥测数据采集）。