---
layout: post
title: "Systemd 实战指南：服务管理、Unit 文件与日志"
date: 2026-08-15
categories: 开发
tags: [systemd, linux, service-management, journald, systemctl]
---

Systemd 是当今 Linux 发行版的标准初始化系统和服务管理器。无论你运行的是 Ubuntu、Debian、Fedora 还是 Arch Linux，systemd 都在幕后管理着几乎所有服务的生命周期。

本文不讲 systemd 的历史恩怨，直接从实战出发，覆盖四个核心场景：**服务管理、Unit 文件编写、Journald 日志、Timer 定时任务**。读完你能独立编写生产级 service 文件、用 journald 高效排查问题、用 timer 替代 cron。

<!-- more -->

## 1. Systemctl 日常操作

### 1.1 基础命令

```bash
# 查看服务状态
systemctl status nginx

# 启动/停止/重启
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx

# 开机自启
sudo systemctl enable nginx
sudo systemctl disable nginx

# 启用并立即启动（等效于 enable + start）
sudo systemctl enable --now nginx

# 重新加载配置（不重启进程）
sudo systemctl reload nginx

# 查看服务是否活跃
systemctl is-active nginx
systemctl is-enabled nginx
```

`reload` 和 `restart` 的区别很关键：`reload` 发送 SIGHUP 信号让进程重新读取配置，不中断连接；`restart` 完全终止进程再启动。对于 nginx、haproxy 这类支持优雅重载的服务，优先用 `reload`。

### 1.2 列出和管理 Units

```bash
# 列出所有活跃的 unit
systemctl list-units

# 列出所有已安装的服务（包括 inactive）
systemctl list-unit-files --type=service

# 按状态过滤
systemctl list-units --state=failed
systemctl list-units --state=running

# 查看服务依赖关系
systemctl list-dependencies nginx

# 查看服务引用的环境变量
systemctl show nginx | grep Environment
```

### 1.3 屏蔽与掩盖

如果你不希望某个服务被任何方式启动（包括作为依赖被拉起），可以用 `mask`：

```bash
sudo systemctl mask apt-daily.service
sudo systemctl unmask apt-daily.service
```

Mask 会在 `/etc/systemd/system/` 下创建一个指向 `/dev/null` 的符号链接，比 `disable` 更彻底。

## 2. 编写 Service Unit 文件

这是最核心的技能。Unit 文件通常放在：

- `/usr/lib/systemd/system/` — 软件包安装的默认 unit
- `/etc/systemd/system/` — 用户自定义或覆盖（优先级更高）
- `/etc/systemd/system/<name>.d/` — drop-in 配置片段

### 2.1 一个完整的 Service 示例

假设我们有一个 Python 写的 Web 服务 `/opt/myapp/app.py`，需要 systemd 管理：

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=MyApp Web Service
Documentation=https://example.com/docs
After=network.target postgresql.service
Wants=postgresql.service
Requires=network-online.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/venv/bin/python /opt/myapp/app.py
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5
TimeoutStopSec=30

# 安全加固
ProtectHome=yes
ProtectSystem=full
PrivateTmp=yes
NoNewPrivileges=yes
CapabilityBoundingSet=CAP_NET_BIND_SERVICE

# 资源限制
LimitNOFILE=65536
MemoryMax=512M
CPUQuota=80%

[Install]
WantedBy=multi-user.target
```

### 2.2 关键指令详解

**`[Unit]` 段**

| 指令 | 作用 |
|------|------|
| `After=` | 声明启动顺序，不创建依赖关系 |
| `Requires=` | 强依赖——被依赖 unit 失败时本 unit 也失败 |
| `Wants=` | 弱依赖——被依赖 unit 失败不影响本 unit |
| `BindsTo=` | 比 Requires 更强——被依赖 unit 停止时本 unit 同步停止 |
| `PartOf=` | 被依赖 unit 停止/重启时本 unit 同步停止/重启 |

**`[Service]` 段**

`Type=` 是最容易搞错的配置：

- `simple` — 默认值。ExecStart 启动的进程就是主进程，立即认为服务启动成功
- `forking` — 进程会 fork 到后台（传统 daemon 模式），需要指定 `PIDFile=`
- `oneshot` — 执行一次就结束，不保持运行。通常配合 `RemainAfterExit=yes`
- `notify` — 进程启动后通过 sd_notify() 发信号通知 systemd
- `exec` — 类似于 simple，但只在 ExecStart 进程完全 execve 后才认为启动完成

`Restart=` 策略：

| 值 | 行为 |
|----|------|
| `no` | 从不自动重启（默认） |
| `on-success` | 仅退出码为 0 时重启 |
| `on-failure` | 退出码非零、被信号杀死、操作超时时重启 |
| `on-abnormal` | 被信号杀死、超时时重启 |
| `on-watchdog` | 看门狗超时重启 |
| `always` | 任何退出原因都重启 |

**`[Install]` 段**

`WantedBy=` 和 `RequiredBy=` 决定服务在哪些 target 下被启动。`multi-user.target` 是大多数非图形服务的标准目标。

### 2.3 使用 Drop-in 覆盖配置

不改原始 unit 文件，通过 drop-in 配置文件覆盖部分参数：

```bash
sudo mkdir -p /etc/systemd/system/nginx.service.d/
sudo tee /etc/systemd/system/nginx.service.d/override.conf << 'EOF'
[Service]
LimitNOFILE=100000
Restart=always
RestartSec=3
EOF

sudo systemctl daemon-reload
sudo systemctl restart nginx
```

用 `systemctl cat nginx` 查看合并后的完整配置。

### 2.4 Oneshot 和 Timer 服务

一次性任务（比如启动时初始化目录）：

```ini
# /etc/systemd/system/init-data-dir.service
[Unit]
Description=Initialize data directory
Before=myapp.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/bin/mkdir -p /data/myapp
ExecStart=/usr/bin/chown myapp:myapp /data/myapp
ExecStart=/usr/bin/chmod 750 /data/myapp

[Install]
WantedBy=myapp.service
```

## 3. Journald 日志管理

systemd 自带 journald 日志系统，替代了传统的 syslog。日志默认存储在 `/var/log/journal/`。

### 3.1 基础查询

```bash
# 查看某个服务的日志
journalctl -u nginx

# 按时间过滤
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2026-08-14 10:00" --until "2026-08-14 12:00"

# 实时跟踪
journalctl -u nginx -f

# 只看最后 N 行
journalctl -u nginx -n 50

# 只看错误级别
journalctl -u nginx -p err

# 显示完整日志（不截断）
journalctl -u nginx --no-pager
```

### 3.2 高级用法

```bash
# 按优先级过滤（0=emerg, 1=alert, 2=crit, 3=err, 4=warning, 5=notice, 6=info, 7=debug）
journalctl -p 3 -b  # 当前启动的错误及以上

# 查看特定进程的日志
journalctl _PID=1234

# 查看特定用户的日志
journalctl _UID=1000

# 查看内核日志
journalctl -k

# JSON 格式输出
journalctl -u nginx -o json-pretty

# 导出日志用于分析
journalctl -u nginx --since "2026-08-01" --output=export > nginx-export.journal
```

### 3.3 日志持久化配置

默认 journald 日志仅保存在内存中（`/run/log/journal/`），重启后丢失。创建持久化存储：

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
```

控制日志大小和保留策略，编辑 `/etc/systemd/journald.conf`：

```ini
[Journal]
Storage=auto
Compress=yes
SystemMaxUse=500M
SystemMaxFileSize=100M
MaxRetentionSec=2week
ForwardToSyslog=no
```

修改后重启服务：

```bash
sudo systemctl restart systemd-journald
```

### 3.4 自定义日志字段

应用程序通过标准输出（stdout/stderr）打印的日志会被 journald 自动捕获。但如果你想让日志结构化，可以输出 Journald 原生格式：

```python
# Python 示例：使用 systemd.journal
import systemd.journal

journal = systemd.journal.JournalHandler()
journal.setFormatter(logging.Formatter('%(message)s'))
logger = logging.getLogger(__name__)
logger.addHandler(journal)

logger.warning("Disk space low: %s", "/data")
```

或者直接输出 JSON 到 stdout，journald 会原样保存：

```bash
echo '{"event":"request","method":"GET","status":200,"latency_ms":42}' | systemd-cat -t myapp
```

## 4. Systemd Timer 替代 Cron

Systemd timer 比 cron 更强大：支持依赖关系、日志集成、随机延迟、精细化调度。

### 4.1 基本 Timer 示例

每天凌晨 3 点执行备份脚本：

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily at 3am

[Timer]
OnCalendar=daily
OnCalendar=03:00:00
RandomizedDelaySec=300
Persistent=true

[Install]
WantedBy=timers.target
```

启用 timer 而不是 service：

```bash
sudo systemctl enable --now backup.timer
```

### 4.2 调度语法

`OnCalendar=` 使用类似 cron 的语法，但更灵活：

```
# 每天 3:00
OnCalendar=03:00:00

# 每天 3:00 和 15:00
OnCalendar=03:00:00,15:00:00

# 每周一 5:00
OnCalendar=Mon 05:00:00

# 每月第一天 2:00
OnCalendar=*-*-01 02:00:00

# 每 10 分钟
OnCalendar=*:0/10

# 工作日 9:00-18:00 每半小时
OnCalendar=Mon..Fri 09:00..18:00:0/30

# 每隔 4 小时，但偏移 15 分钟
OnCalendar=*:0/4:15
```

**`Persistent=true`** 的作用：如果服务在 Timer 触发时处于关机状态，下次开机后立即补执行。

### 4.3 Timer 的独特优势

**日志集成**：Timer 触发的任务日志自动记录在 journald 中，无需配置邮件通知。

**随机延迟**：`RandomizedDelaySec=300` 在触发时间后随机延迟 0-300 秒，避免多个任务同时启动造成负载尖峰。

**单调时间触发**：`OnUnitActiveSec=1h` 从上一次任务结束后开始计时，适合周期性维护任务：

```ini
[Timer]
OnUnitActiveSec=1h
```

**依赖控制**：Timer 任务可以依赖网络、存储等资源就绪后再执行：

```ini
[Unit]
After=network-online.target
Wants=network-online.target
```

### 4.4 Timer 管理命令

```bash
# 列出所有 timer
systemctl list-timers --all

# 查看下次执行时间
systemctl list-timers backup.timer

# 手动触发一次（不修改下次调度时间）
sudo systemctl start backup.service

# 查看 timer 状态
systemctl status backup.timer
```

## 5. 资源控制与安全沙箱

### 5.1 内存和 CPU 限制

```ini
[Service]
# 内存上限（硬限制）
MemoryMax=512M
# 内存上限（软限制，可突破）
MemoryHigh=384M
# CPU 配额（百分比）
CPUQuota=50%
# CPU 权重（1024 为基准，越高越多）
CPUWeight=200
# IO 权重
IOWeight=100
# 限制进程数
TasksMax=200
```

### 5.2 安全沙箱指令

```ini
[Service]
# 阻止访问 /home、/root、/run/user
ProtectHome=yes
# 只读保护 /usr 和 /etc，/var 可写
ProtectSystem=full
# 独立的 /tmp 和 /var/tmp
PrivateTmp=yes
# 禁止获取新权限
NoNewPrivileges=yes
# 限制系统调用白名单
SystemCallFilter=@system-service
# 只读挂载 / 目录
ProtectRoot=yes
# 隐藏其他进程
ProtectProc=invisible
# 限制网络访问
PrivateNetwork=no  # 有网络需求则设为 yes
# 限制能力集
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_NET_RAW
# 只读特定目录
BindReadOnlyPaths=/etc/ssl/certs
```

这些指令底层利用 Linux cgroups、namespaces 和 seccomp 实现，不需要额外安装工具。

## 6. 调试与排障

### 6.1 查看失败原因

```bash
# 服务启动失败
sudo systemctl start myapp
# 如果失败，查看状态
systemctl status myapp
# 查看完整日志
journalctl -u myapp -xe
```

### 6.2 分析启动耗时

```bash
# 查看系统启动总耗时
systemd-analyze

# 每个 unit 的启动耗时
systemd-analyze blame

# 关键路径
systemd-analyze critical-chain

# 图表导出（需要 systemd-analyze 支持）
systemd-analyze plot > boot.svg
```

### 6.3 检查配置

```bash
# 验证 unit 文件语法
systemd-analyze verify /etc/systemd/system/myapp.service

# 查看合并后的配置
systemctl cat myapp

# 查看依赖关系
systemctl list-dependencies myapp

# 查看环境变量
systemctl show myapp -p Environment
```

### 6.4 常见问题

**Q: 服务启动后立即退出，Restart=always 导致无限重启**

```bash
# 查看重启次数
systemctl show -p NRestarts myapp

# 临时停止重启循环
sudo systemctl stop myapp
# 或用 StartLimitInterval 限制
[Unit]
StartLimitIntervalSec=30
StartLimitBurst=3
```

**Q: 服务可以手动启动，但开机不自启**

检查 `[Install]` 段是否有 `WantedBy=multi-user.target`，并且执行了 `enable`：

```bash
sudo systemctl enable myapp
# 确认
systemctl is-enabled myapp
```

**Q: 日志不持久化**

检查 `/var/log/journal` 是否存在，以及 `journald.conf` 中 `Storage=` 不是 `volatile`。

## 总结

Systemd 是现代 Linux 管理的基础设施，值得花时间掌握。核心要点：

1. **Service 编写**：Type 和 Restart 策略选对，比任何调试技巧都重要
2. **Drop-in 覆盖**：永远不要改 `/usr/lib/systemd/system/` 下的文件，用 drop-in
3. **Journald**：替代 `grep /var/log/syslog`，用 `journalctl` 按时间、级别、unit 精确过滤
4. **Timer**：比 cron 更强，集成日志、依赖、随机延迟，生产环境优先使用
5. **安全沙箱**：`ProtectHome`、`PrivateTmp`、`NoNewPrivileges` 零成本提升安全性

最后送你一个实用别名，加到 `~/.bashrc` 里：

```bash
alias sc='systemctl'
alias scu='systemctl --user'
alias jc='journalctl'
alias jcu='journalctl -u'
alias scs='systemctl status'
alias scr='systemctl daemon-reload'
```

每次配新服务，先写 Unit 文件，再用 `systemd-analyze verify` 检查，最后 `systemctl enable --now` 一步到位。养成这个习惯，你的 Linux 服务管理会变得非常清爽。