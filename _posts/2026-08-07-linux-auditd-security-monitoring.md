---
layout: post
title: "Linux Auditd 安全监控实战指南"
date: 2026-08-07 09:00:00 +0800
categories: [安全]
tags: [auditd, Linux, 安全监控, 审计, 入侵检测, 系统安全, 合规, ausearch]
---

当服务器被入侵后，你第一个想到的问题往往是：攻击者做了什么？什么时候进来的？改了哪些文件？

如果没有审计日志，这些问题几乎无解。系统日志（`/var/log/messages`）只记录应用层事件，`history` 可以被轻易清空，`last` 记录的登录日志也能被篡改。而 `auditd`——Linux 内核级审计框架——在入侵发生时就记录了一切，而且攻击者很难清除。

本文从零开始，讲清楚 auditd 的配置、规则编写、日志分析，以及如何用它在真实场景中检测入侵行为。

## auditd 是什么

`auditd` 是 Linux 内核审计系统的用户空间守护进程。它通过内核的 `audit` 子系统捕获系统调用（syscall）、文件访问、进程执行、网络连接等事件，并写入日志。

与 `syslog` 的关键区别：auditd 在内核层面工作，**即使进程被 kill、日志服务被停掉，已经记录的事件也无法被抹去**（除非物理访问磁盘覆写）。

核心组件：

| 组件 | 作用 |
|------|------|
| `auditd` | 守护进程，接收内核事件并写日志 |
| `auditctl` | 控制规则添加/删除/查看 |
| `ausearch` | 查询审计日志 |
| `aureport` | 生成审计报告 |
| `audit.rules` | 规则配置文件（持久化） |
| `/var/log/audit/audit.log` | 默认日志文件 |

## 安装与启动

大多数 Linux 发行版内核已编译 audit 支持，只需安装用户空间工具：

```bash
# Debian/Ubuntu
apt install auditd audispd-plugins

# RHEL/CentOS/AlmaLinux
yum install audit audit-libs

# Arch Linux
pacman -S audit
```

启动并设置开机自启：

```bash
systemctl enable auditd
systemctl start auditd
systemctl status auditd
```

**注意**：`auditd` 服务**不支持**标准的 `reload` 机制。修改规则后，必须用 `auditctl -R /etc/audit/rules.d/audit.rules` 重新加载，或者重启服务（后者会中断日志记录几秒，生产环境慎用）。

## 规则语法

audit 规则分为三类：**控制规则**（配置审计系统本身）、**文件系统规则**（监控文件/目录）、**系统调用规则**（监控 syscall）。

### 控制规则

```bash
# 设置缓冲区大小（单位：条数，默认 64）
auditctl -b 8192

# 设置失败时最大速率（条/秒）
auditctl -r 0

# 锁定规则（禁止修改，直到重启）
auditctl -e 2
```

`-e 2` 是最后一道防线：一旦锁定，任何进程（包括 root）都无法再修改规则，直到重启。生产环境建议在规则部署验证无误后执行。

### 文件系统规则

```bash
# 监控文件写操作和属性变更
auditctl -w /etc/passwd -p wa -k passwd_changes

# 监控目录递归
auditctl -w /etc/nginx/ -p wa -k nginx_config

# 监控可执行文件
auditctl -w /usr/bin/ssh -p x -k ssh_exec
```

`-p` 权限掩码：
- `r` — 读
- `w` — 写
- `x` — 执行
- `a` — 属性变更（chmod、chown、acl）

`-k` 是自定义键名，用于后续过滤查询。

### 系统调用规则

文件系统规则只能覆盖有限路径，全盘监控需要 syscall 规则：

```bash
# 监控所有用户执行新程序
auditctl -a always,exit -F arch=b64 -S execve -k exec_log

# 监控时间修改
auditctl -a always,exit -F arch=b64 -S settimeofday,clock_settime -k time_change

# 监控权限提升
auditctl -a always,exit -F arch=b64 -S setuid,setgid,setreuid,setregid -k priv_esc

# 监控网络连接（需要内核支持）
auditctl -a always,exit -F arch=b64 -S connect -k net_connect
```

`-a always,exit` 表示始终记录，在 syscall 退出时记录。`-F arch=b64` 指定 64 位架构，32 位程序需要额外加 `-F arch=b32` 规则。

## 实战规则集

一个完整的安全监控规则集，按场景分类：

```bash
# ============================================
# 1. 系统关键文件监控
# ============================================

# 用户和权限
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/sudoers -p wa -k sudoers
-w /etc/sudoers.d/ -p wa -k sudoers

# SSH 配置
-w /etc/ssh/ -p wa -k ssh_config
-w /root/.ssh/ -p wa -k ssh_key
-w /home -p wa -k home_dir

# 系统配置
-w /etc/crontab -p wa -k cron
-w /etc/cron.d/ -p wa -k cron
-w /etc/cron.daily/ -p wa -k cron
-w /etc/hosts -p wa -k hosts
-w /etc/hosts.allow -p wa -k hosts
-w /etc/hosts.deny -p wa -k hosts

# 内核模块
-w /etc/modprobe.d/ -p wa -k modules
-w /etc/modules -p wa -k modules

# 日志目录（防止被清空）
-w /var/log/ -p wa -k log_files

# ============================================
# 2. 关键系统调用
# ============================================

# 执行新程序
-a always,exit -F arch=b64 -S execve -k exec

# 加载内核模块
-a always,exit -F arch=b64 -S init_module,finit_module -k kernel_module

# 修改系统时间
-a always,exit -F arch=b64 -S adjtimex,settimeofday,clock_settime -k time

# 权限提升相关
-a always,exit -F arch=b64 -S setuid,setgid,setreuid,setregid -k priv_esc

# 挂载操作
-a always,exit -F arch=b64 -S mount,umount2 -k mount

# 删除文件（危险操作）
-a always,exit -F arch=b64 -S unlink,unlinkat,rmdir -k delete

# 修改网络配置
-a always,exit -F arch=b64 -S sethostname -k network
-a always,exit -F arch=b64 -S iptables -k firewall

# 绑定端口（监听）
-a always,exit -F arch=b64 -S bind -k bind_port

# ============================================
# 3. 排除噪音（合理裁剪）
# ============================================

# 排除健康检查等高频无意义事件
-a never,exit -F arch=b64 -S execve -F uid=0 -F path=/usr/bin/systemd-run -k exclude

# 配置缓冲区
-b 8192

# 失败时操作
-f 1
```

将以上内容保存到 `/etc/audit/rules.d/custom.rules`，然后加载：

```bash
auditctl -R /etc/audit/rules.d/custom.rules
```

查看当前规则：

```bash
auditctl -l
```

## 日志分析

### audit.log 格式

原始日志格式初看很吓人，其实结构明确：

```
type=SYSCALL msg=audit(1680000000.123:456): 
  arch=c000003e syscall=59 success=yes exit=0 
  a0=7f8c4a0b2b40 a1=7f8c4a0b2b70 a2=7f8c4a0b2ba0 a3=8 
  items=2 ppid=1234 pid=5678 auid=1000 uid=0 gid=0 
  euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 
  tty=pts0 ses=3 comm="bash" exe="/usr/bin/bash" 
  key="exec"
```

关键字段解析：

| 字段 | 含义 |
|------|------|
| `type` | 事件类型（SYSCALL, PATH, CWD, PROCTITLE 等） |
| `msg` | `audit(时间戳.序列号)` |
| `syscall=59` | 系统调用号（59 = execve） |
| `success=yes` | 调用是否成功 |
| `ppid` | 父进程 PID |
| `pid` | 进程 PID |
| `auid` | 审计用户 ID（原始登录用户，su 后不变） |
| `uid` | 当前用户 ID |
| `comm` | 命令名 |
| `exe` | 可执行文件路径 |
| `key` | 自定义键名 |

**`auid` 是最重要的字段之一**。它记录的是用户的**原始登录 ID**，即使攻击者 `su` 切换到 root，`auid` 仍然保留最初登录的用户信息。这是追踪入侵源头的关键。

### ausearch 查询

`ausearch` 是 grep 的审计版，结构化查询远比 grep 精准：

```bash
# 按时间查询
ausearch -ts 09:00 -te 10:00 -k exec

# 按用户查询
ausearch -ua root -k exec

# 按文件路径查询
ausearch -f /etc/passwd

# 按 PID 查询
ausearch -p 1234

# 按事件类型
ausearch --event SYSCALL -k exec

# 按成功/失败
ausearch -sv no -k exec
```

常用时间格式：

```bash
# 最近 1 小时
ausearch -ts recent -k exec

# 今天
ausearch -ts today -k delete

# 指定时间范围
ausearch -ts 08/06/2026 -te 08/07/2026 -k priv_esc
```

### aureport 报告

`aureport` 生成汇总统计，适合快速排查：

```bash
# 查看所有事件统计
aureport

# 查看可执行文件执行统计
aureport -x

# 查看用户活动
aureport -u

# 查看登录失败
aureport -l -i

# 查看文件变更
aureport -f

# 查看时间修改
aureport -k -i | grep time_change
```

`-i` 参数将数字化的 UID、GID 等转换为可读名称。

## 四个真实场景

### 场景一：检测暴力破解 SSH

```bash
# 为 SSH 登录失败添加规则
auditctl -w /var/log/auth.log -p wa -k ssh_auth

# 查询失败的 SSH 登录尝试
ausearch -k ssh_auth -sv no --format text | grep "Failed password" | \
  awk '{print $NF}' | sort | uniq -c | sort -rn | head -10
```

更好的方式：直接监控 `sshd` 进程的 `execve`：

```bash
ausearch -k exec -c sshd --format text | grep -E "session opened|Failed password"
```

### 场景二：追踪提权行为

攻击者拿到低权限 shell 后，通常会尝试提权：

```bash
# 查询提权相关系统调用
ausearch -k priv_esc -i

# 查找提权成功的进程
ausearch -k priv_esc -sv yes -i

# 查看执行了 sudo 的进程
ausearch -k exec -c sudo -i
```

关键模式：`auid=1000` 但 `uid=0` —— 说明一个普通用户成功提权为 root。

### 场景三：查找文件被篡改

发现网站被篡改时：

```bash
# 查询 /var/www/html 最近 24 小时的文件变更
ausearch -ts today -f /var/www/html -i

# 找出谁改了文件
ausearch -f /var/www/html/index.html -i | grep SYSCALL

# 查看该进程执行的完整命令链
ausearch -p <PID> -i --format text | grep PROCTITLE
```

`PROCTITLE` 事件记录进程启动时的完整命令行，包括参数。

### 场景四：检测后门安装

攻击者常见的后门行为：

```bash
# 启动监听端口
ausearch -k bind_port -i

# 加载内核模块
ausearch -k kernel_module -i

# 修改 crontab
ausearch -k cron -i -sv yes

# 查看新创建的可执行文件（结合 PATH 事件）
ausearch -k exec -i --format text | grep -vE "(bin|usr/lib)"
```

## 实时告警

auditd 本身不推送告警，但通过 `audispd` 插件可以实现实时通知。

### 配置实时告警

```bash
vim /etc/audit/audispd-plugins/au-remote.conf
```

或者使用更轻量的方式：直接监听日志文件：

```bash
#!/bin/bash
# /usr/local/bin/audit-alert.sh

tail -Fn0 /var/log/audit/audit.log | while read line; do
  echo "$line" | grep -q "key=\"priv_esc\"" || continue
  echo "$line" | grep -q "success=yes" || continue

  # 提取关键信息
  user=$(echo "$line" | grep -oP 'auid=\K[0-9]+')
  pid=$(echo "$line" | grep -oP 'pid=\K[0-9]+')
  exe=$(echo "$line" | grep -oP 'exe="\K[^"]+')
  comm=$(echo "$line" | grep -oP 'comm="\K[^"]+')

  # 发送告警（通知管理员）
  echo "[ALERT] Privilege escalation detected: auid=$user pid=$pid exe=$exe comm=$comm" \
    | mail -s "Security Alert: Privilege Escalation" admin@example.com
done
```

### 集成到 SIEM

将 audit.log 转发到 rsyslog：

```bash
# /etc/rsyslog.d/audit.conf
module(load="imfile")

input(type="imfile"
      File="/var/log/audit/audit.log"
      Tag="auditd:"
      Severity="info"
      Facility="local6")

# 转发到远程 SIEM
*.* @@siem.internal:514
```

## 性能调优

auditd 的常见性能陷阱：

**1. 缓冲区溢出**

```bash
# 查看是否有事件丢失
ausearch --lost

# 检查日志
grep "lost" /var/log/audit/audit.log
```

如果看到 `WARNING: audit backlog limit exceeded`，说明缓冲区不够，增大 `-b` 值。

**2. 规则过多**

每条规则都增加内核开销。经验法则：

- 文件系统规则：不超过 50 条
- syscall 规则：不超过 30 条
- 用 `-F` 条件过滤，减少不必要的事件

**3. 高频路径排除**

```bash
# 排除 /tmp 下的临时文件写入（噪音多且价值低）
-a never,exit -F dir=/tmp -p wa

# 排除容器运行时噪音
-a never,exit -F dir=/var/lib/docker -p wa
```

**4. 日志轮转**

```bash
# /etc/logrotate.d/auditd
/var/log/audit/*.log {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
    postrotate
        /sbin/auditctl -R /etc/audit/rules.d/audit.rules
    endscript
}
```

## 安全注意事项

### 日志保护

audit.log 是**核心证据**，必须保护：

```bash
chmod 600 /var/log/audit/audit.log
chattr +a /var/log/audit/audit.log  # 只能追加，不能删除
```

`chattr +a` 设置 append-only 属性，即使是 root 也无法删除或覆写文件内容，只能追加。这是抵御攻击者清空日志的有效手段。

### 远程日志

单机日志不够安全。配置远程日志转发：

```bash
# 在日志服务器上
# /etc/audit/auditd.conf
tcp_listen_port = 60

# 在被监控机器上
# /etc/audisp/audisp-remote.conf
remote_server = logs.internal
port = 60
```

### 规则锁定

部署完毕后锁定规则，防止攻击者关闭审计：

```bash
auditctl -e 2
```

如果需要临时关闭审计（极度不推荐），只能是：

```bash
auditctl -e 0
```

但注意：`-e 2` 锁定后，`-e 0` 也无效，必须重启。

## 总结

auditd 是 Linux 系统安全监控的基石。它的优势在于：

1. **内核级**：不受用户态进程影响
2. **结构化**：日志格式统一，便于自动化分析
3. **灵活**：可以精确到单个文件或系统调用
4. **低成本**：合理配置下性能开销极小（< 5%）

作者在管理上百台服务器时，auditd 曾经多次在入侵事件中锁定攻击者行为——从 SSH 爆破到提权到篡改文件，每一步都有迹可循。

但 auditd 不是万能的。它不能替代 IDS（如 Snort、Suricata），不能做应用层日志分析，也不适合作为实时拦截系统。它的定位是**记录者**——记录一切，让攻击者无处遁形。

**建议**：从今天开始，在你管理的每台服务器上部署 auditd。先从小规则集开始，熟悉日志格式后再逐步扩展。当入侵真的发生时，你会感谢那个提前部署了审计的自己。

最后，预告一下：下一篇文章会讲如何用 `auditd` + `Wazuh` 搭建企业级安全监控平台，敬请期待。