---
layout: post
title: "HAProxy 负载均衡与反向代理实战指南"
date: 2026-08-05 10:00:00 +0800
categories: [网络技术, 开发]
tags: [haproxy, load-balancing, reverse-proxy, ssl, 网络, 运维, 高可用]
---

## 前言

HAProxy 是目前使用最广泛的开源负载均衡器和反向代理软件之一。它轻量、高效、稳定，单机能处理数百万并发连接，被大量生产环境用于承载七层和四层流量。相比 Nginx 的负载均衡，HAProxy 在连接管理、健康检查、ACL 控制方面更加精细，尤其在 TCP 和 HTTP 混合场景下优势明显。

本文从安装配置开始，逐步深入到生产级部署，涵盖负载均衡算法、SSL 终止、健康检查、ACL 路由、统计监控和性能调优，所有配置均可直接运行。

## 一、安装与基础概念

### 1.1 安装

```bash
# Ubuntu/Debian
apt update && apt install -y haproxy

# 验证版本
haproxy -v
# HAProxy version 2.8.x

# 查看编译选项
haproxy -vv | head -20
```

### 1.2 核心概念

HAProxy 的配置由四个核心段组成：

| 配置段 | 作用 |
|--------|------|
| `global` | 全局参数：进程数、日志、最大连接数 |
| `defaults` | 默认值，给后面的 frontend/backend 继承 |
| `frontend` | 入口：监听端口、接收请求、应用 ACL |
| `backend` | 后端服务器池：定义真实服务器和均衡策略 |

请求流：`frontend` → 匹配 ACL → 选择 `backend` → 转发到后端服务器。

### 1.3 最小配置示例

```haproxy
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    retries 3
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend web_front
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web1 192.168.1.10:80 check
    server web2 192.168.1.11:80 check
```

启动：

```bash
haproxy -f /etc/haproxy/haproxy.cfg -c  # 检查配置
systemctl restart haproxy
```

## 二、负载均衡算法详解

HAProxy 支持多种均衡算法，选择正确的算法直接影响后端资源利用率。

### 2.1 roundrobin

最基础的轮询算法，请求依次分配给每个后端。适合后端性能相近的场景。

```haproxy
backend web_servers
    balance roundrobin
    server web1 192.168.1.10:80 weight 2 check
    server web2 192.168.1.11:80 weight 1 check
    server web3 192.168.1.12:80 weight 1 check
```

`weight` 参数控制分配比例，web1 会拿到的请求是 web2 的两倍。

### 2.2 leastconn

分配给当前活跃连接数最少的服务器。适合长连接场景（WebSocket、数据库连接池）。

```haproxy
backend api_servers
    balance leastconn
    server api1 192.168.1.20:8080 check
    server api2 192.168.1.21:8080 check
```

### 2.3 source

基于客户端 IP 的哈希，保证同一 IP 始终分配到同一台后端。用于会话保持（session persistence）。

```haproxy
backend app_servers
    balance source
    hash-type consistent  # 一致性哈希，后端增减时影响最小
    server app1 192.168.1.30:3000 check
    server app2 192.168.1.31:3000 check
    server app3 192.168.1.32:3000 check
```

### 2.4 uri / url_param

基于 URI 或 URL 参数哈希，适合缓存场景——同一资源始终落入同一后端，提高缓存命中率。

```haproxy
backend cache_servers
    balance uri
    server cache1 192.168.1.40:80 check
    server cache2 192.168.1.41:80 check
```

### 2.5 算法选型建议

| 场景 | 推荐算法 |
|------|---------|
| 短连接 HTTP 请求 | roundrobin |
| WebSocket / gRPC 长连接 | leastconn |
| 需要会话保持 | source |
| 静态资源缓存 | uri / url_param |
| 后端性能差异大 | roundrobin + weight |

## 三、健康检查

健康检查是负载均衡的核心机制——HAProxy 定期探测后端，自动摘除故障节点，恢复后自动加回。

### 3.1 基本健康检查

```haproxy
backend web_servers
    server web1 192.168.1.10:80 check inter 2000 rise 3 fall 2
    # 参数说明：
    #   inter 2000   — 每 2 秒检查一次
    #   rise 3       — 连续成功 3 次标记为 UP
    #   fall 2       — 连续失败 2 次标记为 DOWN
```

### 3.2 HTTP 层健康检查

默认只检查 TCP 端口是否可达。更精确的做法是用 HTTP 请求验证应用是否正常：

```haproxy
backend api_servers
    option httpchk GET /health HTTP/1.1\r\nHost:\ api.example.com
    server api1 192.168.1.20:8080 check
    server api2 192.168.1.21:8080 check
```

后端应返回 2xx/3xx 表示健康，返回 4xx/5xx 标记为故障。

### 3.3 自定义检查间隔

对关键服务可以缩短检查间隔，对非关键服务适当延长：

```haproxy
backend db_readers
    option tcp-check
    tcp-check send PING\r\n
    tcp-check expect string +PONG
    server db1 10.0.0.1:6379 check inter 1000 rise 2 fall 3
    server db2 10.0.0.2:6379 check inter 1000 rise 2 fall 3 backup
```

`backup` 标记的服务器只在主服务器全部故障时启用。

## 四、SSL/TLS 终止

HAProxy 可以卸载 SSL 加密，将明文流量转发给后端，减轻后端服务器负担。

### 4.1 单证书配置

```haproxy
frontend https_front
    bind *:443 ssl crt /etc/ssl/haproxy/example.com.pem
    # crt 选项指定证书文件，格式为：私钥 + 证书链 合并到一个 PEM 文件
    http-request redirect scheme https unless { ssl_fc }
    default_backend web_servers
```

生成证书文件：

```bash
cat /etc/letsencrypt/live/example.com/privkey.pem \
    /etc/letsencrypt/live/example.com/fullchain.pem \
    > /etc/ssl/haproxy/example.com.pem
chmod 600 /etc/ssl/haproxy/example.com.pem
```

### 4.2 多域名多证书（SNI）

```haproxy
frontend https_front
    bind *:443 ssl crt /etc/ssl/haproxy/  # 目录模式，自动匹配
    # 每个域名对应一个 PEM 文件，命名如：example.com.pem、api.example.com.pem
    use_backend blog_servers if { ssl_fc_sni -i blog.example.com }
    use_backend api_servers  if { ssl_fc_sni -i api.example.com }
    default_backend web_servers
```

### 4.3 SSL 安全加固

```haproxy
global
    tune.ssl.default-dh-param 2048
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    ssl-default-server-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-server-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
```

## 五、ACL 与内容路由

ACL（Access Control List）是 HAProxy 最强大的功能之一，可以根据请求的任意属性做路由决策。

### 5.1 基于域名

```haproxy
frontend http_front
    bind *:80
    acl host_blog    hdr(host) -i blog.example.com
    acl host_api     hdr(host) -i api.example.com
    acl host_admin   hdr(host) -i admin.example.com

    use_backend blog_servers    if host_blog
    use_backend api_servers     if host_api
    use_backend admin_servers   if host_admin
    default_backend web_servers
```

### 5.2 基于路径

```haproxy
frontend http_front
    bind *:80
    acl path_api    path_beg -i /api/v2
    acl path_static path_beg -i /static /images /css /js
    acl path_upload path_beg -i /upload

    use_backend api_servers     if path_api
    use_backend static_servers  if path_static
    use_backend upload_servers  if path_upload
```

### 5.3 组合条件

```haproxy
frontend http_front
    bind *:80
    acl is_internal src 10.0.0.0/8 172.16.0.0/12
    acl is_admin     path_beg -i /admin
    acl is_api       path_beg -i /api

    # 内部访问 /admin 走 admin 池，外部访问 /admin 返回 403
    http-request deny if is_admin !is_internal
    use_backend admin_servers  if is_admin is_internal
    use_backend api_servers    if is_api
```

### 5.4 请求重写

```haproxy
frontend http_front
    bind *:80

    # 添加/修改请求头
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    http-request set-header X-Real-IP %[src]
    # 删除安全敏感头
    http-request del-header X-Internal-Auth

    # 重定向
    http-request redirect scheme https unless { ssl_fc }
    redirect prefix https://blog.example.com code 301 if { hdr(host) -i old-blog.example.com }
```

## 六、速率限制与 Stick Table

### 6.1 基本速率限制

使用 stick table 跟踪客户端请求频率：

```haproxy
frontend http_front
    bind *:80

    # 定义 stick table：基于客户端 IP，每 10 秒最多 100 个请求
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src

    # 超过阈值则返回 429
    acl abuse src_http_req_rate(10s) ge 100
    http-request deny deny_status 429 if abuse
```

### 6.2 并发连接限制

```haproxy
frontend http_front
    bind *:80

    stick-table type ip size 100k expire 30s store conn_cur
    http-request track-sc0 src

    acl conn_limit src_conn_cur ge 50
    http-request deny deny_status 429 if conn_limit
```

### 6.3 基于路径的细粒度限流

```haproxy
frontend http_front
    bind *:80

    # 对 API 路径做更严格的限制
    acl path_api path_beg -i /api
    http-request track-sc0 src if path_api
    stick-table type ip size 100k expire 30s store http_req_rate(10s)

    acl api_abuse src_http_req_rate(10s) ge 30
    http-request deny deny_status 429 if path_api api_abuse
```

## 七、生产部署与监控

### 7.1 统计页面

```haproxy
frontend stats
    bind *:8404
    stats enable
    stats uri /haproxy-stats
    stats refresh 5s
    stats admin if LOCALHOST
    stats auth admin:your_strong_password
```

访问 `http://your-server:8404/haproxy-stats` 查看实时监控面板，包含：
- 每个 frontend/backend 的连接数、会话速率
- 每台服务器的健康状态、流量、队列长度
- 手动启用/禁用后端服务器

### 7.2 日志配置

```haproxy
global
    log /dev/log local0 info
    log /dev/log local1 notice

defaults
    log     global
    option  httplog
    option  log-health-checks
    log-format "%ci:%cp [%t] %ft %b/%s %Tw/%Tc/%Tt %B %st %r" # 自定义格式
```

查看日志：

```bash
# rsyslog 接收 HAProxy 日志
cat > /etc/rsyslog.d/49-haproxy.conf << 'EOF'
local0.*  /var/log/haproxy.log
local1.*  /var/log/haproxy-admin.log
EOF
systemctl restart rsyslog
```

### 7.3 性能优化

```haproxy
global
    maxconn 100000
    nbproc 4          # 多进程模式（旧版本，HAProxy 2.5+ 推荐 nbthread）
    nbthread 4        # 多线程模式，利用多核 CPU
    cpu-map 1 0
    cpu-map 2 1
    cpu-map 3 2
    cpu-map 4 3

    # 缓冲区优化
    tune.bufsize 32768
    tune.maxrewrite 1024
    tune.ssl.cachesize 50000

defaults
    # 超时优化
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    timeout http-keep-alive 10s
    timeout queue   10s
    # 连接池
    option http-keep-alive
    option http-server-close   # 后端复用连接，减少三次握手
```

### 7.4 优雅重启

```bash
# 检查配置正确性
haproxy -f /etc/haproxy/haproxy.cfg -c

# 优雅重载（不中断现有连接）
haproxy -f /etc/haproxy/haproxy.cfg -p /var/run/haproxy.pid -sf $(cat /var/run/haproxy.pid)
```

生产环境建议使用 `systemctl reload haproxy` 或 `haproxy-reload` 脚本。

## 八、安全加固

### 8.1 管理接口保护

```haproxy
frontend stats
    bind *:8404 ssl crt /etc/ssl/haproxy/admin.pem
    stats enable
    stats uri /haproxy-stats
    stats refresh 5s
    stats admin if LOCALHOST
    # 暴露到外网时严格限制来源
    acl trusted_network src 10.0.0.0/8 172.16.0.0/12
    http-request deny unless trusted_network
```

### 8.2 请求头清理

```haproxy
frontend http_front
    # 删除客户端传递的敏感头
    http-request del-header X-Forwarded-For          # HAProxy 会替我们设置
    http-request del-header X-Real-IP
    http-request del-header CF-Connecting-IP
    http-request del-header True-Client-IP

    # 限制请求大小
    http-request reject if { content-length -m 10485760 }  # 拒绝 > 10MB 的请求
```

### 8.3 慢速攻击防护

```haproxy
defaults
    # 超时收包，防止慢速 HTTP 攻击
    timeout http-request 10s
    # 限制请求头大小
    option http-buffer-request
    max-http-header-size 8192
```

## 九、完整示例：生产环境配置

```haproxy
global
    log /dev/log local0 info
    maxconn 100000
    nbthread 4
    cpu-map auto:1/1-4 0-3
    user haproxy
    group haproxy
    daemon
    tune.ssl.default-dh-param 2048
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  http-server-close
    option  redispatch
    retries 3
    timeout http-request 10s
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    timeout http-keep-alive 10s
    timeout queue 10s
    max-http-header-size 8192

frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/haproxy/ alpn h2,http/1.1

    # 强制 HTTPS
    http-request redirect scheme https unless { ssl_fc }

    # 安全头
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    http-response set-header X-Content-Type-Options nosniff
    http-response set-header X-Frame-Options SAMEORIGIN

    # 域名路由
    acl host_blog  hdr(host) -i blog.example.com
    acl host_api   hdr(host) -i api.example.com

    use_backend blog_servers if host_blog
    use_backend api_servers  if host_api
    default_backend web_servers

    # 速率限制
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    acl abuse src_http_req_rate(10s) ge 200
    http-request deny deny_status 429 if abuse

    # 监控
    option forwardfor

frontend stats
    bind 127.0.0.1:8404
    stats enable
    stats uri /haproxy-stats
    stats refresh 5s
    stats auth admin:changeme

backend blog_servers
    balance leastconn
    option httpchk GET /health HTTP/1.1\r\nHost:\ blog.example.com
    server blog1 10.0.1.10:8080 check inter 2s rise 3 fall 2
    server blog2 10.0.1.11:8080 check inter 2s rise 3 fall 2

backend api_servers
    balance roundrobin
    option httpchk GET /api/health HTTP/1.1\r\nHost:\ api.example.com
    http-request set-header X-Real-IP %[src]
    server api1 10.0.1.20:8080 check inter 2s rise 3 fall 2
    server api2 10.0.1.21:8080 check inter 2s rise 3 fall 2
    server api3 10.0.1.22:8080 check inter 2s rise 3 fall 2

backend web_servers
    balance roundrobin
    server web1 10.0.1.100:80 check inter 3s rise 2 fall 3
    server web2 10.0.1.101:80 check inter 3s rise 2 fall 3
```

## 十、HAProxy 与 Nginx 选型对比

| 能力 | HAProxy | Nginx |
|------|---------|-------|
| TCP 负载均衡 | 原生支持，性能极好 | 通过 stream 模块 |
| HTTP 负载均衡 | 专业级 | 支持 |
| 健康检查 | 丰富灵活 | 基础 |
| ACL / 路由 | 强大，条件丰富 | 通过 location/config |
| 速率限制 | stick table 非常灵活 | limit_req 模块 |
| 负载均衡算法 | 十余种 | 4 种基础 |
| SSL 终止 | 支持，性能好 | 支持，性能好 |
| 静态文件服务 | 不擅长 | 擅长 |
| 配置复杂度 | 中等 | 中等 |

**选型建议**：纯负载均衡场景选 HAProxy；需要同时处理静态文件 + 反向代理选 Nginx；两者配合使用是最佳实践——HAProxy 做入口负载均衡，Nginx 做 Web 服务。

## 总结

HAProxy 是生产环境负载均衡的首选方案，胜在稳定、高效、功能全面。掌握它的核心配置并不难，关键是要理解：

1. **frontend/backend 分离**的设计思想，让路由和服务器池解耦
2. **健康检查**是自动故障转移的基石
3. **ACL** 提供了丰富的流量路由能力
4. **stick table** 是实现速率限制、会话保持等高级功能的核心机制
5. 生产部署必须配置 SSL、日志、监控和超时参数

建议本地搭建多台虚拟机或容器，用本文配置一步步上手，观察 HAProxy 在不同后端故障时的行为，这是理解其价值最直接的方式。