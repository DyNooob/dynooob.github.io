---
layout: post
title: "JWT 安全攻防实战指南"
date: 2026-07-30 14:00:00 +0800
categories: 安全开发
tags: [jwt, security, authentication, python, devsecops]
---

## 为什么 JWT 会成为攻击面

JSON Web Token（JWT）已经成为现代 Web 服务中最主流的认证方案。它的无状态特性让后端不必维护 session 存储，配合微服务和分布式架构简直是天作之合。但正是这种"无需查库"的信任模型，让 JWT 成为攻击者眼中的高价值目标。

一个 JWT 的安全问题，轻则导致越权访问，重则直接账户接管。本文从攻击者视角出发，梳理了 JWT 最常见的 7 类漏洞，并给出 Python 环境下的实战修复方案。读完你应该能审出 90% 的 JWT 安全隐患。

## 第一攻：alg:none 签名绕过

JWT 的 header 由三部分组成：`alg`（算法）、`typ`（类型）、`kid`（可选）。有些库在实现时默认信任 token 中声明的算法类型。攻击者把 `alg` 改成 `none`，JWT 库就会跳过签名验证。

**攻击示例：**

```python
import jwt

# 有漏洞的验证方式
def verify_token_vulnerable(token):
    # 默认使用 token 的 header 中的算法
    payload = jwt.decode(token, options={"verify_signature": False})
    return payload

# 攻击者构造的 token
evil_token = "eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJhZG1pbiIsInJvbGUiOiJhZG1pbiJ9."
# header: {"alg":"none","typ":"JWT"}
# payload: {"sub":"admin","role":"admin"}
# 没有签名部分
```

**防御方案：**

```python
def verify_token_secure(token, secret):
    # 强制指定算法，不信任 token 声明
    try:
        payload = jwt.decode(token, secret, algorithms=["HS256"])
        return payload
    except jwt.InvalidAlgorithmError:
        raise
```

关键点：永远不要使用 `options={"verify_signature": False}` 或允许用户通过 header 指定算法。在 `jwt.decode()` 中明确传 `algorithms` 参数，覆盖掉 token 自带的 `alg` 字段。

## 第二攻：算法混淆 (Algorithm Confusion)

RSA 是非对称加密，HMAC 是对称加密。攻击者拿到一个 RSA 签名的公钥后，用这个公钥作为 HMAC 的密钥重新签名——因为 HMAC 和 RSA 的签名验证接口长得一样，有些库用同一个方法处理。

**攻击流程：**

```python
# 服务端暴露了公钥
public_key = """-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
-----END PUBLIC KEY-----"""

# 攻击者用公钥作为 HMAC 密钥签名
evil_token = jwt.encode(
    {"sub": "admin", "role": "admin"},
    public_key,
    algorithm="HS256"
)
```

**防御方案：**

```python
def verify_token_anti_confusion(token, public_key):
    # 方案1：显式禁止 HMAC 算法
    try:
        payload = jwt.decode(token, public_key, algorithms=["RS256"])
        return payload
    except jwt.InvalidAlgorithmError:
        raise

# 方案2：使用单独的库或方法区分对称和非对称
# PyJWT 从 2.0+ 开始，当传入 RSA 公钥时自动拒绝 HMAC
# 但仍需在代码中明确指定 algorithms
```

最佳实践：如果使用非对称算法，永远只允许 `RS256`、`ES256` 等，拒绝 `HS256`。同时确保传入的 `key` 类型与算法匹配——公钥给 RSA，私钥给 HMAC。

## 第三攻：弱密钥爆破

HMAC 签名依赖一个共享密钥。如果密钥不够强，攻击者可以离线爆破。

```python
import base64
import hmac
import hashlib
from itertools import product
import string

def brute_force_jwt(token, wordlist):
    """演示 HMAC 弱密钥爆破"""
    # 从 token 中提取 header 和 payload
    parts = token.split(".")
    header_payload = f"{parts[0]}.{parts[1]}"
    target_sig = parts[2]

    for secret in wordlist:
        sig = base64.urlsafe_b64encode(
            hmac.new(
                secret.encode(), header_payload.encode(), hashlib.sha256
            ).digest()
        ).rstrip(b"=").decode()

        # 注意：PyJWT 的 base64 编码不带 padding
        if sig == target_sig:
            return secret
    return None
```

**防御方案：**

```python
import secrets

# 生成足够强的密钥（至少 256 bits = 32 bytes）
def generate_secure_secret():
    return secrets.token_hex(32)  # 64 hex chars, 256 bits

# 正确使用
SECRET_KEY = generate_secure_secret()
# 示例输出：a7f8d91e4b2c3f5a6e7d8c9b0a1f2e3d4c5b6a7f8e9d0c1b2a3f4e5d6c7b8a9
```

不要用 `"secret"`、`"jwt_secret"`、`"password123"` 或任何短于 32 字节的密钥。推荐用 `secrets.token_hex(32)` 或 `openssl rand -hex 32` 生成。生产环境把密钥放到环境变量或密钥管理服务中，不要硬编码。

## 第四攻：KID 注入与路径遍历

`kid`（Key ID）是 JWT header 中的一个可选字段，用于告诉服务端用哪把密钥验证。有些实现用 `kid` 去文件系统或数据库查找密钥文件，这给路径遍历和 SQL 注入留了后门。

**有漏洞的实现：**

```python
import os

def get_public_key_by_kid(kid):
    # 漏洞：直接拼接路径
    key_path = f"/etc/jwt_keys/{kid}"
    if os.path.exists(key_path):
        with open(key_path) as f:
            return f.read()
    return None

# 攻击者构造的 token header
# {"kid": "../../../etc/passwd", "alg": "HS256"}
# 如果服务端用这个 kid 读文件，攻击者可以读取任意文件
```

**更隐蔽的注入：**

```python
# 攻击者构造的 token header
# {"kid": "/dev/null", "alg": "HS256"}
# 如果服务端用 /dev/null 的内容作为密钥（空字符串），
# 攻击者也能直接用空字符串签名
```

**防御方案：**

```python
import re

def get_public_key_by_kid_safe(kid):
    # 方案1：白名单校验
    ALLOWED_KEYS = {"key1", "key2", "key3"}
    if kid not in ALLOWED_KEYS:
        raise ValueError(f"Unknown kid: {kid}")

    # 方案2：正则限制只有字母数字
    if not re.match(r"^[a-zA-Z0-9_-]+$", kid):
        raise ValueError(f"Invalid kid format: {kid}")

    key_path = f"/etc/jwt_keys/{kid}.pem"
    if os.path.exists(key_path):
        with open(key_path) as f:
            return f.read()
    return None
```

如果必须使用 `kid`，做白名单验证或严格的正则校验。绝对不要用用户输入的字符串直接拼接路径或 SQL 查询。

## 第五攻：令牌泄露与重放

JWT 一旦签发，在过期前一直有效。如果 token 被中间人截获（比如明文 HTTP、日志记录、URL 参数泄漏），攻击者可以直接使用。

**Token 泄露的常见场景：**

```
1. URL 参数传递：https://example.com/api?token=xxx
   → 浏览器历史记录、referer header 都会泄漏

2. 日志记录：access_log 中打印 Authorization header
   → 运维人员或日志聚合服务能看到

3. 前端存储：localStorage 存储 JWT + XSS 漏洞
   → 攻击者通过 XSS 窃取 localStorage

4. 第三方脚本：CDN 引入的 JS 脚本读取页面内容
   → 恶意脚本扫描 DOM 中的 token
```

**防御方案：**

```python
from datetime import datetime, timedelta
import hashlib
import hmac

def create_secure_token(user_id, secret, ttl_minutes=15):
    """签发短生命周期 + 绑定客户端特征"""
    now = datetime.utcnow()
    payload = {
        "sub": user_id,
        "iat": now,
        "exp": now + timedelta(minutes=ttl_minutes),
        # 绑定客户端指纹（可选，增加安全性）
        "jti": secrets.token_hex(16),  # 唯一 token ID
    }
    return jwt.encode(payload, secret, algorithm="HS256")


# 服务端维护一个 token 黑名单（用于登出场景）
revoked_tokens = set()

def verify_with_blacklist(token, secret):
    try:
        payload = jwt.decode(token, secret, algorithms=["HS256"])
        jti = payload.get("jti")
        if jti and jti in revoked_tokens:
            raise jwt.InvalidTokenError("Token has been revoked")
        return payload
    except jwt.ExpiredSignatureError:
        # 过期 token 自动失效
        raise
```

关键实践：
- 使用 `localStorage` 还是 `httpOnly Cookie`？后者更安全，因为 JS 无法读取。
- 设置合理的 `exp`（过期时间），普通 API 15-30 分钟，refresh token 7 天。
- 敏感操作（转账、修改密码）要求二次验证或短时效 token。
- 使用 `jti`（JWT ID）实现服务端级别撤销。

## 第六攻：Claims 过度信任

JWT 的 payload 是 base64 编码，不是加密。任何人都能解码看到内容。有些开发者把敏感信息放在 payload 中，或者信任 payload 中的角色/权限字段而不做服务端校验。

```python
# 有问题的实现：信任 token 中的角色
def check_admin_vulnerable(token, secret):
    payload = jwt.decode(token, secret, algorithms=["HS256"])
    # 漏洞：直接信任 payload 中的 role
    if payload.get("role") == "admin":
        return True
    return False

# 攻击者解码后看到 payload 结构
# 然后尝试修改 role 字段，用弱密钥签名
```

**防御方案：**

```python
def check_admin_secure(token, secret):
    payload = jwt.decode(token, secret, algorithms=["HS256"])
    user_id = payload.get("sub")

    # 核心原则：从数据库查询权限，不信任 token 中的 claims
    user_role = query_user_role_from_db(user_id)
    return user_role == "admin"
```

不要在 payload 中放密码、身份证号、信用卡号等敏感信息。权限校验永远查数据库，不信任 token 中的声明。JWT 只做身份标识（我是谁），不做权限声明（我能做什么）。

## 第七攻：Refresh Token 绕过后端验证

很多架构用 access token + refresh token 双令牌方案。如果实现不当，refresh token 反而成为新的攻击面。

**常见问题：**

```python
# 问题1：refresh token 永不过期
def refresh_access_token_vulnerable(refresh_token):
    # 只验证签名，不检查过期时间
    payload = jwt.decode(refresh_token, REFRESH_SECRET, algorithms=["HS256"])
    return create_access_token(payload["sub"])

# 问题2：refresh token 不绑定设备
# 攻击者窃取 refresh token 后可以在任意设备上生成 access token

# 问题3：旧 refresh token 失效后仍可重复使用
# 没有 rotation 机制
```

**防御方案：**

```python
def create_token_pair(user_id, device_id, secret):
    access_token = jwt.encode({
        "sub": user_id,
        "type": "access",
        "exp": datetime.utcnow() + timedelta(minutes=15),
        "device": device_id,
    }, secret, algorithm="HS256")

    refresh_token = jwt.encode({
        "sub": user_id,
        "type": "refresh",
        "exp": datetime.utcnow() + timedelta(days=7),
        "device": device_id,
        "jti": secrets.token_hex(16),  # 用于 rotation
    }, secret, algorithm="HS256")

    return access_token, refresh_token


def rotate_refresh_token(old_refresh_token, old_jti, user_id, secret):
    """Refresh Token Rotation：每次使用后签发新令牌，旧令牌立即失效"""
    # 验证旧 token
    payload = jwt.decode(old_refresh_token, secret, algorithms=["HS256"])

    # 检查是否已被撤销（在 redis 中）
    if redis.sismember(f"revoked_refresh:{user_id}", old_jti):
        # 检测到令牌重用，可能被窃取，撤销该用户所有 refresh token
        redis.del(f"refresh_token:{user_id}:{payload['device']}")
        raise SecurityException("Refresh token reuse detected")

    # 撤销旧 token
    redis.sadd(f"revoked_refresh:{user_id}", old_jti)

    # 签发新令牌对
    return create_token_pair(user_id, payload["device"], secret)
```

Refresh token 必须有有效期，绑定设备/客户端，并且实现 rotation（每次使用刷新后，旧 token 立即失效）。检测到 token 重用视为攻击，撤销该用户所有 session。

## 综合防御清单

把以下检查点做成你的 JWT 安全 checklist：

```
[ ] 强制指定算法，不信任 header 中的 alg
[ ] 非对称算法和对称算法使用不同的验证路径
[ ] 密钥 >= 256 位，存储在环境变量或密钥管理服务
[ ] kid 做白名单或严格正则校验
[ ] 不使用 URL 参数传递 token
[ ] 使用 httpOnly Cookie 而非 localStorage 存储 token
[ ] 设置合理的过期时间（15-30 分钟）
[ ] 权限校验基于数据库，不信任 token claims
[ ] 敏感信息不放在 payload 中
[ ] Refresh token 实现 rotation 机制
[ ] 记录并监控 token 验证失败日志
[ ] 提供登出接口，将 token 加入黑名单
```

## 工具推荐

快速验证 JWT 配置的工具：

```bash
# 1. jwt_tool - 全功能 JWT 安全测试
pip install jwt_tool
jwt_tool eyJhbGciOiJIUzI1NiIs... -t http://target.com/api -X GET

# 2. 本地解码检查（不联网）
python3 -c "
import jwt
token = 'eyJhbGciOi...'
# 不解码 algs，只查看 payload
header = jwt.get_unverified_header(token)
payload = jwt.decode(token, options={'verify_signature': False})
print('Header:', header)
print('Payload:', payload)
"

# 3. 密钥强度测试（hashcat + jwt 模式）
# hashcat -m 16500 jwt.txt wordlist.txt
```

## 总结

JWT 不是天生不安全的，问题出在实现上。七个攻击面归根结底是同一个问题：**信任了不该信任的东西**——信任用户提供的算法、信任 header 中的 kid、信任 payload 中的权限、信任过期的 token 不会被重用。

防御思路也很简单：最小化信任边界。服务端必须掌握所有关键决策：用什么算法、用什么密钥、用户有什么权限。JWT 只是一个身份凭证，不是授权声明。

安全开发的核心原则始终如一：不要信任用户输入，包括他们发来的 token。