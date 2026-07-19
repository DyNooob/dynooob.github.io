---
layout: post
title: "Nginx 安全加固实战：从默认配置到生产级防护"
date: 2026-07-19 09:00:00 +0800
categories: [网络安全]
tags: [Nginx, Web安全, 安全加固, 服务器配置, 防护策略]
---

Nginx 的默认配置足以让它跑起来，但远不足以让它安全地跑在生产环境。从默认安装到生产就绪，中间隔着一层安全加固。本文不讲理论，直接从配置入手，覆盖信息隐藏、请求限制、TLS 强化、路径防护、反向代理安全、日志审计等实战环节。

## 1. 隐藏版本信息与服务指纹

Nginx 默认会在响应头中暴露版本号：

```
Server: nginx/1.26.0
```

攻击者拿到版本号，就能去 CVE 库中找对应漏洞。两步操作彻底关闭：

```nginx
# 主配置中关闭版本显示
server_tokens off;

# 如果还想进一步，可以自定义 Server 头（需要编译进 ngx_http_headers_more_module）
# 或者用 proxy_pass 时直接覆盖
```

设置后响应变为：

```
Server: nginx
```

更进一步，某些扫描工具还能通过 HTTP 响应行为（如 404 页面格式、默认错误页面）识别 Nginx。替换默认错误页面：

```nginx
error_page 400 /400.html;
error_page 403 /403.html;
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

# 或者直接外部重定向，不暴露任何页面结构
error_page 404 /index.html;
```

## 2. HTTP 安全响应头

安全响应头是浏览器端的第一道防线。以下配置直接加到 `server` 块中：

```nginx
# 强制 HTTPS 通信 (HSTS)
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# 防止点击劫持
add_header X-Frame-Options "DENY" always;

# 阻止 MIME 类型嗅探
add_header X-Content-Type-Options "nosniff" always;

# 跨域资源策略
add_header Cross-Origin-Resource-Policy "same-origin" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Embedder-Policy "require-corp" always;

# 引用来源策略
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# 权限策略，禁用不需要的浏览器 API
add_header Permissions-Policy "geolocation=(), microphone=(), camera=(), payment=()" always;

# 内容安全策略 (CSP) — 这是最复杂也最重要的一个，按需调整
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self'; frame-ancestors 'none'" always;
```

注意 `always` 参数确保即使返回 4xx/5xx 状态码，这些头也会被发送。HSTS 的 `preload` 参数需要提前在 [hstspreload.org](https://hstspreload.org) 提交域名，否则不要加。

## 3. 请求限制与防滥用

生产环境最怕的不是常规访问，而是恶意请求：暴力破解、CC 攻击、爬虫耗尽资源。

### 限制请求频率

```nginx
# 在 http 块中定义限制区域
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/s;
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
limit_req_zone $binary_remote_addr zone=static:10m rate=100r/s;

# 连接数限制
limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_conn_zone $server_name zone=server:10m;

server {
    # 登录接口：5 次/秒，超过则延迟，超过 10 次/秒直接拒绝
    location /login {
        limit_req zone=login burst=5 nodelay;
        limit_conn addr 5;
        proxy_pass http://backend;
    }

    # API 接口：30 次/秒
    location /api/ {
        limit_req zone=api burst=10 nodelay;
        limit_conn addr 10;
        proxy_pass http://backend;
    }

    # 静态资源：100 次/秒
    location /static/ {
        limit_req zone=static burst=20;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

`burst=5 nodelay` 的意思是：允许短时间内的 5 个突发请求，不做延迟处理直接放行，超出部分直接 503。不加 `nodelay` 则超出部分排队处理，这对登录接口可能不够安全。

### 限制请求体大小

```nginx
# 限制上传大小，防止大文件上传耗尽内存
client_max_body_size 10m;

# 限制请求头大小
client_header_buffer_size 1k;
large_client_header_buffers 4 8k;
```

## 4. TLS/SSL 安全配置

HTTPS 不是装上证书就完事了。TLS 协议版本和密码套件的选择直接影响安全性。

### 协议与密码套件

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    # 禁用不安全的 TLS 版本
    ssl_protocols TLSv1.2 TLSv1.3;

    # 优先使用现代密码套件
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # 优先使用服务端定义的密码顺序，而非客户端
    ssl_prefer_server_ciphers on;

    # 会话缓存，减少 TLS 握手开销
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;

    # OCSP Stapling，提升证书验证性能
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout 5s;
}
```

### 自动跳转 HTTP -> HTTPS

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

## 5. 路径遍历与文件访问控制

Nginx 默认就能提供静态文件服务，但默认配置很容易暴露敏感文件。

### 禁止访问敏感文件

```nginx
location ~* \.(env|git|svn|hg|config|conf|ini|log|sql|bak|swp|old|~)$ {
    deny all;
    return 404;
}

# 禁止访问隐藏文件（以点开头）
location ~ /\. {
    deny all;
    return 404;
}

# 禁止访问敏感目录
location ~* /(admin|backup|dump|config|logs|data)/ {
    internal;
    # 或者 deny all + return 404
}
```

### 限制目录列表

```nginx
# 全局关闭目录列表
autoindex off;

# 如果某个目录确实需要列表，显式打开
location /downloads/ {
    autoindex on;
    autoindex_exact_size off;
    autoindex_localtime on;
    # 建议加 IP 白名单或密码保护
    satisfy any;
    allow 192.168.0.0/16;
    deny all;
}
```

### 防止路径穿越攻击

反向代理时，路径穿越是常见漏洞。Nginx 默认会对 `../` 做规范化处理，但 `alias` 指令容易踩坑：

```nginx
# 危险写法
location /files/ {
    alias /data/uploads/;
}
# 访问 /files../etc/passwd 可能被解析为 /data/uploadsetc/passwd（注意拼接逻辑）

# 安全写法：确保 alias 末尾有斜杠，且 location 匹配完整
location /files/ {
    alias /data/uploads/;
}

# 更安全的做法：用 root 替代 alias
location /files/ {
    root /data;
    # 访问 /files/abc.txt 实际指向 /data/files/abc.txt
}
```

## 6. 反向代理安全

Nginx 做反向代理时，需要特别注意上游请求头。默认配置会透传客户端原始头，这可能导致攻击者伪造来源。

### 清理上游请求头

```nginx
location /api/ {
    proxy_pass http://backend;

    # 只传递必要的头
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # 清除内部请求头，防止 SSRF 等攻击
    proxy_set_header X-Internal-Auth "";

    # 限制上游连接
    proxy_connect_timeout 10s;
    proxy_read_timeout 30s;
    proxy_send_timeout 10s;

    # 禁用缓冲（某些场景下需要）
    proxy_buffering off;
}
```

### 限制上游返回头部

```nginx
# 隐藏上游返回的敏感头
proxy_hide_header X-Powered-By;
proxy_hide_header X-AspNet-Version;
proxy_hide_header X-AspNetMvc-Version;

# 覆盖上游服务器头
proxy_set_header Server "nginx";
```

## 7. 防爬虫与恶意 User-Agent

```nginx
# 在 http 块中定义
map $http_user_agent $bad_bot {
    default 0;
    ~*curl 1;
    ~*wget 1;
    ~*python-requests 1;
    ~*go-http-client 1;
    ~*scrapy 1;
    ~*ZmEu 1;
    ~*sqlmap 1;
    ~*nmap 1;
    ~*nikto 1;
    ~*masscan 1;
}

server {
    # 拒绝恶意 UA
    if ($bad_bot) {
        return 444;  # 444 是 Nginx 自定义状态码，直接断开连接不返回任何响应
    }
}
```

注意：`if` 在 location 上下文中性能较差且有坑，放在 server 块中相对安全。更好的做法是用 `map` + `return` 组合。

## 8. 日志审计与异常检测

没有日志的安全加固是盲人摸象。合理配置日志能帮你快速定位攻击。

```nginx
# 自定义日志格式，包含更多安全相关信息
log_format security '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$request_time $upstream_response_time '
                    '$http_x_forwarded_for';

# 访问日志
access_log /var/log/nginx/access.log security buffer=32k flush=5s;

# 错误日志
error_log /var/log/nginx/error.log warn;

# 单独记录 4xx/5xx 请求
map $status $loggable {
    ~^[23]  0;
    default 1;
}
access_log /var/log/nginx/error_access.log security if=$loggable;
```

配合 `fail2ban` 可以实现自动封禁。以下是一个简单的 fail2ban 规则例子（放在 `/etc/fail2ban/jail.local`）：

```ini
[nginx-req-limit]
enabled = true
filter = nginx-req-limit
action = iptables-multiport[name=ReqLimit, port="http,https", protocol=tcp]
logpath = /var/log/nginx/error.log
findtime = 600
maxretry = 10
bantime = 3600
```

## 9. 模块化最小化原则

Nginx 编译时，只加载真正需要的模块。生产环境不需要完整的默认编译：

```bash
# 查看当前编译了哪些模块
nginx -V 2>&1 | tr ' ' '\n' | grep module

# 示例：最小化编译
./configure \
    --without-http_autoindex_module \
    --without-http_ssi_module \
    --without-http_userid_module \
    --without-http_geo_module \
    --without-http_memcached_module \
    --without-http_empty_gif_module \
    --without-http_browser_module \
    --without-http_split_clients_module \
    --without-http_limit_conn_module=dynamic \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_realip_module \
    --with-http_headers_more_module
```

如果用的是发行版预编译的 Nginx，至少确保不加载多余的动态模块：

```nginx
# 只加载需要的动态模块
# load_module modules/ngx_http_headers_more_filter_module.so;
```

## 10. 一个完整的最小安全配置示例

把以上内容整合成一个可直接参考的 server 块：

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com;

    # TLS 配置
    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 8.8.8.8 valid=300s;
    resolver_timeout 5s;

    # 安全头
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; frame-ancestors 'none'" always;

    # 信息隐藏
    server_tokens off;

    # 请求限制
    limit_req zone=api burst=10 nodelay;
    limit_conn addr 10;
    client_max_body_size 10m;

    # 敏感文件保护
    location ~* \.(env|git|svn|config|log|sql|bak|swp|old)$ { deny all; return 404; }
    location ~ /\. { deny all; return 404; }

    # 反向代理
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_hide_header X-Powered-By;
        proxy_connect_timeout 10s;
        proxy_read_timeout 30s;
    }

    # 静态资源
    location /static/ {
        root /var/www/example;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 日志
    access_log /var/log/nginx/example.access.log;
    error_log /var/log/nginx/example.error.log warn;
}

# HTTP 跳转
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

## 配置验证与测试

每次修改配置后，务必验证：

```bash
# 语法检查
nginx -t

# 重载配置（不中断连接）
nginx -s reload

# 用 curl 检查响应头
curl -sI https://example.com | grep -E '^(HTTP|Server|Strict-Transport|X-Frame|X-Content|Content-Security)'

# 用 testssl.sh 检查 TLS 安全性
# https://github.com/drwetter/testssl.sh
./testssl.sh --quiet https://example.com

# 用 Mozilla Observatory 评分
# https://observatory.mozilla.org/
```

## 总结

Nginx 安全加固不是一劳永逸的事。以上 10 个环节覆盖了从信息隐藏到请求限制、从 TLS 强化到日志审计的主流加固手段，但安全配置需要随着业务变化和漏洞披露持续更新。几个关键原则：

- **最小化**：只开必要的端口和模块，只放必要的路径
- **纵深防御**：安全头、请求限制、TLS 配置各司其职，不依赖单一防线
- **可审计**：日志配齐，告警配好，才能知道谁在做什么
- **持续验证**：每次改配置后跑一遍 `nginx -t` 和安全头检查

配置完成后，建议用 Mozilla Observatory 或 Security Headers 在线工具扫一遍，看看自己拿几分。