---
layout: post
title: "SELinux 实战指南：从强制访问控制到策略管理"
date: 2026-08-27 09:00:00 +0800
categories: [开发]
tags: [selinux, linux, security, system-administration, access-control, centos, rhel, fedora, kernel, hardening]
---

## 为什么需要 SELinux

传统的 Linux 权限模型是 **DAC（自主访问控制）**：文件所有者决定谁能读/写/执行它，root 用户拥有一切权限。这意味着一个进程一旦被攻破（比如一个存在 RCE 漏洞的 Nginx 进程），攻击者就以该进程的身份运行——如果进程是 root，就拿到了整个系统。

SELinux 实现了 **MAC（强制访问控制）**：系统管理员定义全局策略，**所有**进程和文件都有安全上下文。即使进程以 root 运行，也受策略约束。Nginx 进程只能访问标记为 `httpd_sys_content_t` 的文件，不能碰 `/etc/shadow`，不能启动 shell，不能监听非标准端口。

这不是"安全增强选项"，而是生产环境 Linux 的安全基石。RHEL/CentOS/Fedora 默认启用 SELinux，关闭它会被 Red Hat 视为"不支持的配置"。

## SELinux 三模式

SELinux 有三种运行模式：

| 模式 | 行为 | 使用场景 |
|------|------|----------|
| **Enforcing** | 拦截违规操作并记录日志 | 生产环境 |
| **Permissive** | 记录日志但不拦截 | 调试策略问题 |
| **Disabled** | 完全关闭 | 仅当确认不需要时（不推荐） |

查看当前模式：

```bash
getenforce
# Enforcing

# 或更详细的信息
sestatus
# SELinux status:                 enabled
# Current mode:                   enforcing
# Mode from config file:          enforcing
# Policy version:                 33
```

临时切换模式（重启后恢复）：

```bash
# 切换到 Permissive（排查问题）
setenforce 0

# 切回 Enforcing
setenforce 1
```

永久修改在 `/etc/selinux/config` 中：

```bash
# /etc/selinux/config
SELINUX=enforcing    # enforcing | permissive | disabled
SELINUXTYPE=targeted # targeted | minimum | mls
```

修改后需要重启生效。**不要在生产环境设为 disabled**——如果你必须关掉才能跑应用，说明配置有问题，应该修策略而不是关保护。

## 安全上下文：SELinux 的核心概念

SELinux 中每个进程和文件都有一个安全上下文（context），格式为：

```
user:role:type:level
```

查看文件上下文：

```bash
ls -Z /etc/passwd
# system_u:object_r:passwd_file_t:s0  /etc/passwd

ls -Z /var/www/html/index.html
# system_u:object_r:httpd_sys_content_t:s0  /var/www/html/index.html
```

查看进程上下文：

```bash
ps -Z $(pgrep nginx | head -1)
# system_u:system_r:httpd_t:s0  ...

ps -Z $(pgrep sshd | head -1)
# system_u:system_r:sshd_t:s0  ...
```

四字段的含义：

- **user**: SELinux 用户账户（如 `system_u`、`unconfined_u`、`user_u`），不是 Linux 用户
- **role**: 基于角色的访问控制（RBAC），如 `object_r`（文件）、`system_r`（系统进程）
- **type**: 核心字段——决定允许/禁止的操作。文件类型以 `_t` 结尾（如 `httpd_sys_content_t`），进程域也是 `_t` 结尾（如 `httpd_t`）
- **level**: MLS（多级安全）级别，默认 `s0`，高级场景用于分级保密

**关键规则**：一个域（进程）只能访问策略中明确允许的类型的文件。进程 `httpd_t` 能读 `httpd_sys_content_t`，但写 `httpd_log_t` 需要额外策略。

## 实战：配置 Nginx 在非标准端口运行

这是一个经典场景：Nginx 默认监听 80/443，但你想让它监听 8080。如果直接改配置重启，SELinux 会拦截：

```bash
# 修改 nginx.conf 监听 8080
# 启动后检查日志
journalctl -u nginx -n 20
# 可能看到：Permission denied 或无法绑定端口
```

检查 SELinux 的审计日志确认：

```bash
# 查看最近的 SELinux 拒绝记录
ausearch -m avc -ts recent

# 或查看 audit.log
grep "denied" /var/log/audit/audit.log | tail -5
```

解决方法是告诉 SELinux 允许 Nginx 使用 8080 端口：

```bash
# 查看当前允许的端口
semanage port -l | grep http

# 添加 8080 到 http_port_t
semanage port -a -t http_port_t -p tcp 8080

# 确认
semanage port -l | grep http_port_t
# http_port_t                    tcp      8080, 80, 81, 443, 488, 8008, 8009, 8443, 9000
```

现在重启 Nginx 就能正常绑定 8080 了。**不需要关闭 SELinux**。

## 实战：让 Nginx 读取自定义目录的文件

你的网站文件放在 `/data/www/` 而不是默认的 `/var/www/html/`。Nginx 配置了正确的 root 路径，但访问时返回 403。

原因：新文件的上下文是 `default_t`（未标记的通用类型），Nginx 的 `httpd_t` 域不允许读 `default_t`。

```bash
# 检查文件上下文
ls -Z /data/www/
# system_u:object_r:default_t:s0  index.html

# 修复：递归修改上下文
semanage fcontext -a -t httpd_sys_content_t "/data/www(/.*)?"
restorecon -Rv /data/www/

# 验证
ls -Z /data/www/
# system_u:object_r:httpd_sys_content_t:s0  index.html
```

**重要**：用 `semanage fcontext` + `restorecon`，而不是直接 `chcon`。`chcon` 的修改会被 `restorecon` 重置，而 `semanage fcontext` 写入策略数据库，即使执行 `restorecon` 也保持。

## 实战：允许 Nginx 连接到后端代理

当 Nginx 配置为反向代理时，需要连接到后端（如 `proxy_pass http://127.0.0.1:3000`）。SELinux 默认不允许 `httpd_t` 域发起网络连接。

```bash
# 查看 httpd 相关的布尔值
getsebool -a | grep httpd

# 允许 Nginx 发起网络连接
setsebool -P httpd_can_network_connect on

# 只允许连接到后端（不全面开放）
setsebool -P httpd_can_network_relay on
```

`-P` 参数使修改永久生效（写入策略）。不加 `-P` 只对当前会话有效。

## 实战：允许 Nginx 写日志到自定义目录

Nginx 错误日志放在 `/var/log/nginx/error.log` 没问题，但如果你改到 `/data/logs/nginx/`：

```bash
# 创建目录并设置类型
semanage fcontext -a -t httpd_log_t "/data/logs/nginx(/.*)?"
restorecon -Rv /data/logs/nginx/

# 确认
ls -Zd /data/logs/nginx/
# system_u:object_r:httpd_log_t:s0  /data/logs/nginx/
```

## 审计日志与 audit2allow

当 SELinux 阻止操作时，记录在 `/var/log/audit/audit.log`（需安装 auditd）。如果 `auditd` 未运行，某些拒绝会记录到 `dmesg` 或 `/var/log/messages`。

查看实时拒绝：

```bash
# 实时监控 SELinux 拒绝
auditctl -w /etc/shadow -p rwxa
ausearch -m avc -ts recent
```

**audit2allow** 是排查问题的利器：它读审计日志，生成允许此操作的策略模块。

```bash
# 查看最近拒绝并生成策略
grep "denied" /var/log/audit/audit.log | audit2allow -M myapp

# 这生成两个文件：
# myapp.pp  - 编译好的策略模块
# myapp.te  - 策略源码（文本）

# 安装策略
semodule -i myapp.pp

# 查看已安装的模块
semodule -l | grep myapp
```

`myapp.te` 的内容示例：

```
module myapp 1.0;

require {
    type httpd_t;
    type var_log_t;
    class file { open read write };
    class dir { search };
}

allow httpd_t var_log_t:file { open read write };
```

**不要盲目使用 audit2allow**。它生成的是"允许刚才被拒绝的操作"的策略，而不是"安全的最优策略"。先用 `audit2why` 确认是否有更合适的布尔值或类型可用：

```bash
audit2why < /var/log/audit/audit.log
```

## 布尔值管理

布尔值是 SELinux 的开关——预定义的策略选项，允许调整影响行为而不编写自定义策略。

```bash
# 列出所有布尔值
getsebool -a

# 查看特定模块的布尔值
semanage boolean -l | grep httpd

# 查看布尔值的含义
semanage boolean -l | grep -E "httpd_can_network_connect|httpd_enable_homedirs"
```

常用布尔值：

| 布尔值 | 作用 | 安全风险 |
|--------|------|----------|
| `httpd_can_network_connect` | 允许 httpd 发起网络连接 | 中 |
| `httpd_can_network_connect_db` | 允许 httpd 连接数据库 | 中 |
| `httpd_enable_homedirs` | 允许 httpd 访问用户家目录 | 高 |
| `httpd_execmem` | 允许 httpd 执行可写内存（JIT 编译需要） | 高 |
| `httpd_unified` | 统一 httpd 类型（允许读写所有 httpd 类型文件） | 中 |
| `httpd_tty_comm` | 允许 httpd 访问终端 | 低 |
| `ssh_sysadm_login` | 允许 root 通过 SSH 登录 | 高 |
| `selinuxuser_execmod` | 允许用户执行可写共享库 | 高 |
| `domain_can_mmap_files` | 允许所有域映射文件 | 低 |

```bash
# 永久启用
setsebool -P httpd_can_network_connect on

# 临时关闭
setsebool httpd_can_network_connect off
```

**原则**：启用最小的布尔值集合。例如允许 Nginx 代理后端，用 `httpd_can_network_relay` 而不是 `httpd_can_network_connect`。

## 实战：自定义应用程序的 SELinux 策略

假设你有一个自定义二进制程序 `/opt/myapp/bin/server`，需要监听 9000 端口、读 `/opt/myapp/data/` 下的配置文件、写日志到 `/opt/myapp/logs/`。

### 第一步：创建应用类型

```bash
# 为应用创建类型和域
# 最简单的做法：用 audit2allow 从实际运行中学习
```

### 第二步：运行应用并捕获拒绝

```bash
# 在 Permissive 模式下运行
setenforce 0
systemctl start myapp
# 收集日志
ausearch -m avc -ts recent > /tmp/myapp_avc.txt
```

### 第三步：生成策略

```bash
audit2allow -M myapp_local < /tmp/myapp_avc.txt
semodule -i myapp_local.pp
```

### 第四步：标记文件

```bash
semanage fcontext -a -t myapp_exec_t "/opt/myapp/bin(/.*)?"
semanage fcontext -a -t myapp_conf_t "/opt/myapp/data(/.*)?"
semanage fcontext -a -t myapp_log_t "/opt/myapp/logs(/.*)?"
restorecon -Rv /opt/myapp/

# 允许端口
semanage port -a -t myapp_port_t -p tcp 9000
```

### 第五步：切回 Enforcing 并验证

```bash
setenforce 1
systemctl restart myapp
# 检查运行情况
ausearch -m avc -ts recent
```

## 容器与 SELinux

Docker 和 Podman 在启用了 SELinux 的系统上行为不同。

**Docker**：默认关闭 SELinux 标记（需要 `--security-opt label=type:container_t` 或 daemon 配置）：

```bash
# 运行容器时启用 SELinux 约束
docker run --security-opt label=type:container_t -it nginx bash

# 或全局启用（在 daemon.json 中）
# {
#   "selinux-enabled": true
# }
```

**Podman**：默认启用 SELinux，容器进程以 `container_t` 域运行：

```bash
# 查看容器进程的上下文
podman run -d --name web nginx
ps -Z $(pgrep nginx)
# system_u:system_r:container_t:s0:c1,c2  ...
```

**常见问题**：容器挂载宿主机目录时，SELinux 阻止容器进程读/写。

```bash
# 错误方式（容器内无法写挂载目录）
podman run -v /data/uploads:/uploads:Z nginx
# :Z 参数自动重标记挂载卷的上下文为 container_file_t
# :z 参数表示多个容器共享（标记为 container_share_t）

# 或手动设置
semanage fcontext -a -t container_file_t "/data/uploads(/.*)?"
restorecon -Rv /data/uploads/
```

`:Z` 与 `:z` 的区别：
- `:Z` — 私有挂载，只给当前容器使用（`container_file_t`）
- `:z` — 共享挂载，多个容器可读（`container_share_t`）

## 常见问题排查流程

当应用出现"Permission denied"但文件权限正确时：

```
1. 确认 SELinux 是否启用
   getenforce → Enforcing

2. 检查审计日志
   ausearch -m avc -ts recent | tail -20

3. 分析拒绝原因
   audit2why < /var/log/audit/audit.log | head -20

4. 修复方案优先级：
   a) 使用正确的文件上下文 → semanage fcontext + restorecon
   b) 使用正确的端口上下文 → semanage port
   c) 启用布尔值 → setsebool -P
   d) 生成自定义策略 → audit2allow -M
   e) 永不：关闭 SELinux
```

## 推荐工具和工作流

```bash
# 安装工具包
dnf install -y policycoreutils policycoreutils-python-utils setroubleshoot

# 友好的错误解读（替代直接看 audit.log）
sealert -a /var/log/audit/audit.log

# 文件上下文参考
semanage fcontext -l | grep httpd

# 查看域间的转换规则
sesearch --allow | grep httpd_t
```

## 总结

SELinux 不是"麻烦"，而是生产环境 Linux 的安全生命线。掌握它的核心思维：

1. **类型强制**是根本——每个进程和文件都有一个类型，策略决定类型间的交互
2. **不要关 SELinux**——95% 的问题可以通过布尔值、上下文或自定义策略解决
3. **排查流程**：审计日志 → audit2why → 选择正确的修复方式
4. **容器场景**：Podman 原生支持，Docker 需额外配置，用 `:Z`/`:z` 控制挂载卷上下文

花半小时理解 SELinux 的上下文模型，能省下未来无数"为什么权限不够"的排查时间。生产环境服务器上，**SELinux 开着才叫安全**。