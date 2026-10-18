---
layout: post
title: "HTTP 请求走私实战：从 CL.TE 边界差到反代加固"
date: 2026-10-18 18:00:00 +0800
categories: [网络安全]
tags: [HTTP 请求走私, request smuggling, CL.TE, TE.CL, 反向代理, nginx, h2c, 协议攻击, web security, burp, httpx]
---

## 为什么请求走私至今还能在生产环境挖到洞

HTTP Request Smuggling（RFC 7230 早就警告过）从 2005 年公开至今已经二十年了，但真实场景里它仍然属于高危漏洞，因为它依赖的不是单点漏洞，而是**前端代理与后端服务器对同一份 HTTP 报文边界的理解不一致**。这种不一致只要存在一个反代（CDN、WAF、API Gateway）就存在可能，加上 H2/H1 边界转换、gRPC 复用、waf 漏检，整个攻击面反而在变大。

今天把走私的几种边界差、复现 POC、用 httpx + Burp 验证、以及生产侧的硬性修复（不只是"用 HTTP/2"）一起讲清楚。

## 一、四种边界差速查

走私的本质是 **Content-Length 与 Transfer-Encoding 谁说了算** 的歧义。下面四种是实战最常见的组合：

| 类型 | 前端 | 后端 | 关键差异 |
|------|------|------|----------|
| CL.TE | 用 `Content-Length` | 用 `Transfer-Encoding: chunked` | 后端按 chunk 切，前端按 CL 切 |
| TE.CL | 用 `Transfer-Encoding` | 用 `Content-Length` | 反过来，前端先切完 |
| TE.TE | 都看 TE | 都看 TE | 双方都支持 obfuscation，看谁先放弃 |
| H2 → H1 | HTTP/2 客户端 | HTTP/1.1 后端 | h2c 升级、:method 头、伪头解析差 |

外加一个**前端解析 Bug 类**：Nginx 历史上 `Content-Length: 0X` (十六进制)、`Content-Length: 0 ` (末尾空格) 的兼容性问题，导致部分请求被吞进下一个请求的 body。

## 二、用 httpx 复现 CL.TE

工具准备：

```bash
pip install httpx[http2]
```

靶场我用 PortSwigger 的官方 lab 也可以自己起一个最简版本：

```python
# smuggler_cl_te.py
import httpx

front = "http://127.0.0.1:8888"   # 反代（默认用 CL）
back  = "http://127.0.0.1:8080"   # 真实后端（默认用 TE）

# 前端看到 1 个请求，body 长度是 CL=6
# 后端按 chunked 切，第一段是 0\r\n\r\n（结束），第二段被当作下一个请求的 prefix
payload = (
    "POST / HTTP/1.1\r\n"
    "Host: 127.0.0.1\r\n"
    "Content-Length: 6\r\n"
    "Transfer-Encoding: chunked\r\n"
    "\r\n"
    "0\r\n"
    "\r\n"
    "G"                       # ← 这一个字节会被走私到下一个请求的 body 头部
)

with httpx.Client(http2=False) as c:
    r = c.post(front + "/", content=payload.encode(),
               headers={"Host": "127.0.0.1"})
    print(r.status_code, r.text[:200])
```

判定标准是**第二次普通请求的响应被污染**——比如发两个普通 GET，第二个返回里能看到走私进去的那段字符，或者 status code 是走私请求里指定的。

更稳的判定方式：走私一段完整的 prefix，让下一个受害者请求落到一个**只有认证后才能访问的路径**：

```
走私 prefix：
GET /admin HTTP/1.1
Host: 127.0.0.1
Foo: bar

走私后端将把这段当成下一个请求的头部，与后面真正的请求 body 拼接形成完整请求。
```

## 三、用 Burp 插件验证差异

手动跑脚本容易误判（前端可能直接 400 拒掉走私），所以更可靠的是用 `HTTP Request Smuggler` Burp 插件做黑盒探测：

1. 抓两个连续请求的最后一个字节和下一个请求的第一个字节；
2. 插件自动用 `Transfer-Encoding: chunked`、`Content-Length: 0` 等组合试差；
3. 如果第二个请求的响应被污染，会显示 `Possible CL.TE vulnerability` 或 `TE.CL vulnerability`。

注意：插件默认会发探测请求，**对生产系统跑前请先在测试环境**，否则可能造成真实用户请求被劫持。

## 四、H2 → H1 走私（最容易在 2024+ 的环境里挖到）

现代架构基本都会在边界做 H2→H1 降级（CDN → 反代 → 业务）。H2 的请求头在边界转换时被序列化到 H1 表单，差异点集中在：

- **`transfer-encoding` 头不被 RFC 7540 允许**，但部分网关会原样写入 H1；
- **`:method` 伪头**：H2 里 `PRI * HTTP/2.0` 这种 connect 升级请求，转 H1 时变成 `OPTIONS * HTTP/1.1`，部分老后端会接收；
- **请求走私用的 `h2c upgrade` 通道**：客户端发 `HTTP/1.1 101 Switching Protocols` 升级到 H2 后，第二个 H2 请求的 body 会被走私到 H1 后端的下一个连接。

检测方法：

```bash
# 用 nghttp2 直连反代试 h2c 升级
nghttp -nv --upgrade http://target:8080/
```

如果后端是 Node、Jetty、Cloudflare 的某些老版本，会接受 `:path` 重复、`:method` 与 header Method 不一致等畸形请求，这里面就有走私窗口。

## 五、生产侧加固（不只是升级 HTTP/2）

最常被引用却**不充分**的修复是"统一启用 HTTP/2 端到端"，但这只解决了 H2 边界的部分场景。实战里真正能堵死的硬性配置：

### 1. 反代层禁用 TE/CL 冲突请求

**Nginx**（`nginx.conf`）：

```nginx
server {
    # 拒绝带 Transfer-Encoding 的请求
    if ($http_transfer_encoding ~* "chunked") {
        return 400;
    }
    # 或更彻底：rewrite 后丢弃 TE
    proxy_set_header Transfer-Encoding "";
    proxy_pass http://backend;
}
```

**Caddy**（`Caddyfile`）：

```
reverse_proxy backend:8080 {
    header_up -Transfer-Encoding
}
```

### 2. 用 HTTP/1.1 + 强制连接复用策略

如果业务必须在反代池内做 H1 连接复用，确保 `keepalive_requests` 一致，避免某个 worker 用 CL 切完、另一个 worker 接力时解析错位：

```nginx
upstream backend {
    server 10.0.0.1:8080;
    keepalive 64;
    keepalive_requests 1000;
    keepalive_timeout 60s;
}
```

### 3. 后端 Web 容器层

- **Tomcat**：`maxSwallowSize` 不要设过大；
- **Node http**：避免使用同时支持 CL 与 TE 的解析器分支；
- **Envoy / Istio**：用 `HttpConnectionManager` 的 `normalize_headers: true`，会自动合并重复头。

### 4. WAF / CDN 侧

Cloudflare、Akamai 的"请求走私缓解"开关默认是开的，但部分自定义 WAF 规则会**复写 TE 头却不复写 CL**，等于重新引入歧义。审计时重点看：是否有规则 `request.headers.get('Transfer-Encoding')` 被读但 `Content-Length` 没被同步处理。

## 六、检测与告警（蓝队视角）

如果生产环境已经部署了反代，想知道是不是历史上被走私过：

1. **NetFlow / eBPF 抓连接**：走私成功后，受害者的请求会落到攻击者控制的路径上，业务侧会观察到 `User-Agent`、`Cookie` 异常的请求来自同一个源 IP；
2. **业务日志指纹**：走私请求往往带特定前缀（攻击者构造的 GET /admin），可以基于 NetFlow 元数据里 `http.request.uri` 异常路径做告警；
3. **连接级时序**：走私成功后，正常用户请求的处理时间会出现抖动，因为攻击者走私的请求挤占了 worker。

## 七、小结

请求走私二十年没死的根本原因是**反代、后端、浏览器、CDN 任何一环换了实现，都可能重新引入边界差**。挖洞侧记住三件事：

1. CL.TE / TE.CL 是基础，TE.TE 用 `Transfer-Encoding: \tchunked` 这种 obfuscation 探测；
2. H2 → H1 边界是 2024+ 之后挖到最多洞的地方，关注 `:path` 重复、`PRI` 请求、h2c upgrade；
3. 修复侧永远不要只信任"用了 HTTP/2 就行"——反代层显式丢弃 `Transfer-Encoding`、强制后端协议一致才是硬性手段。

下次再遇到反代架构，先用 `httpx` + `HTTP Request Smuggler` 跑一遍，10 分钟内大概率能知道有没有边界差。