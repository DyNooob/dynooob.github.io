---
layout: post
title: "DevSecOps 实战：用 Trivy 和 OPA 构建安全 CI/CD 流水线"
date: 2026-08-04 18:00:00 +0800
categories: [安全开发]
tags: [devsecops, trivy, opa, ci-cd, security, container]
---

## 1. 为什么需要 DevSecOps

传统开发流程中，安全往往是最后一步——上线前找安全团队做一次渗透测试，发现问题再修复。这种"安全左移"不够彻底，漏洞发现得越晚，修复成本越高。

DevSecOps 的核心思想是：**把安全嵌进流水线，让每一次构建都自动经过安全检查**。高危漏洞直接阻断构建，低危漏洞记录到工单，而不是依赖人工审核。

本文用两个工具搭建一条完整的 DevSecOps 流水线：

- **Trivy** — 开源容器镜像/文件系统/代码仓库漏洞扫描器，覆盖 OS 包漏洞和语言依赖漏洞
- **OPA (Open Policy Agent)** — 通用策略引擎，用 Rego 语言编写合规策略，做准入控制

最终效果：`git push` → CI 自动构建 → Trivy 扫描 → OPA 策略判定 → 安全则推送，不安全则阻断。

## 2. 工具链概览

### Trivy

Trivy (https://github.com/aquasecurity/trivy) 由 Aqua Security 开源，特点：

- 扫描容器镜像、文件系统、Git 仓库、Kubernetes、AWS/Azure/GCP 配置
- 数据库包含 CVE、NVD、Red Hat、Ubuntu、Debian、Alpine、Python、npm、Go、Java 等来源
- 支持 SARIF 格式输出，可集成 GitHub Code Scanning
- 误报率低，扫描速度快

### OPA

OPA 是 CNCF 毕业项目，用 Rego 声明式语言定义策略：

- 将策略从代码中解耦出来
- 支持 REST API 集成
- 适用于 Kubernetes 准入控制、CI/CD 门禁、API 授权等场景

## 3. 环境准备

安装 Trivy：

```bash
# Ubuntu/Debian
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# macOS
brew install trivy

# 验证
trivy --version
```

安装 OPA：

```bash
# Linux
curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64_static
chmod +x opa
sudo mv opa /usr/local/bin/

# 验证
opa version
```

## 4. Trivy 扫描实战

### 4.1 镜像扫描

扫描一个 Docker 镜像，输出 JSON 格式结果：

```bash
trivy image --format json --output scan.json nginx:1.25
```

只看高危漏洞：

```bash
trivy image --severity CRITICAL,HIGH nginx:1.25
```

指定漏洞数据库更新时间（首次运行会自动下载数据库）：

```bash
trivy image --cache-dir /tmp/trivy-db --download-db-only
trivy image --cache-dir /tmp/trivy-db nginx:1.25
```

### 4.2 文件系统扫描

在 CI 中，用 `trivy fs` 扫描项目目录，检查依赖漏洞：

```bash
trivy fs --severity CRITICAL,HIGH /path/to/project
```

指定扫描类型（只扫语言依赖）：

```bash
trivy fs --scanners vuln --vuln-type library /path/to/project
```

### 4.3 配置扫描 (IaC)

扫描 Kubernetes YAML、Terraform、Dockerfile 等配置问题：

```bash
trivy config --format json --output config-scan.json ./deploy/
```

### 4.4 输出格式

Trivy 支持多种输出格式，CI/CD 集成推荐 JSON 或 SARIF：

```bash
# JSON，便于程序解析
trivy image --format json nginx:1.25 | jq '.Results[] | select(.Vulnerabilities != null) | {Target, Severity: .Vulnerabilities[].Severity, PkgName: .Vulnerabilities[].PkgName, VulnerabilityID: .Vulnerabilities[].VulnerabilityID}'

# SARIF，用于 GitHub Code Scanning
trivy image --format sarif --output result.sarif nginx:1.25
```

## 5. OPA 策略编写

### 5.1 策略结构

OPA 用 Rego 语言编写策略。一个典型的策略文件结构：

```
policies/
├── policy.rego       # 主策略
├── data.json         # 外部数据（可选）
└── policy_test.rego  # 测试
```

### 5.2 编写 Trivy 扫描结果准入策略

假设 Trivy 输出 JSON 格式，我们写一条策略：**存在任何 CRITICAL 漏洞则拒绝**。

```rego
# policies/trivy-scan.rego
package trivy.policy

# 默认允许
default allow := false

# 如果存在高危漏洞则拒绝
deny[msg] {
    vuln := input.Results[_].Vulnerabilities[_]
    vuln.Severity == "CRITICAL"
    msg := sprintf("CRITICAL vulnerability found: %s in %s (%s)", [
        vuln.VulnerabilityID,
        vuln.PkgName,
        vuln.InstalledVersion
    ])
}

# 允许条件：没有拒绝项
allow {
    count(deny) == 0
}
```

### 5.3 细化策略：按严重级别和漏洞数量控制

```rego
# policies/trivy-scan-advanced.rego
package trivy.policy

default allow := false

# CRITICAL 级别的漏洞直接阻断
deny[msg] {
    vuln := input.Results[_].Vulnerabilities[_]
    vuln.Severity == "CRITICAL"
    msg := sprintf("BLOCKED: CRITICAL %s in %s (%s)", [
        vuln.VulnerabilityID,
        vuln.PkgName,
        vuln.InstalledVersion
    ])
}

# HIGH 漏洞超过 5 个则阻断
deny[msg] {
    high_vulns := {vuln.VulnerabilityID |
        vuln := input.Results[_].Vulnerabilities[_]
        vuln.Severity == "HIGH"
    }
    count(high_vulns) > 5
    msg := sprintf("BLOCKED: %d HIGH vulnerabilities found (max 5 allowed)", [count(high_vulns)])
}

# 允许条件
allow {
    count(deny) == 0
}
```

### 5.4 策略测试

```rego
# policies/trivy-scan_test.rego
package trivy.policy

test_allow_no_vulns {
    allow with input as {"Results": []}
}

test_deny_critical {
    not allow with input as {"Results": [{
        "Vulnerabilities": [{
            "VulnerabilityID": "CVE-2024-0001",
            "PkgName": "openssl",
            "InstalledVersion": "1.1.1",
            "Severity": "CRITICAL"
        }]
    }]}
}
```

运行测试：

```bash
opa test policies/
```

## 6. CI/CD 流水线集成

### 6.1 GitHub Actions 完整示例

以下是一个完整的 GitHub Actions 工作流，集成 Trivy 扫描和 OPA 策略判定：

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t app:ci .

      - name: Run Trivy vulnerability scan
        id: scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: app:ci
          format: json
          output: trivy-results.json
          severity: CRITICAL,HIGH

      - name: Download OPA
        run: |
          curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64_static
          chmod +x opa

      - name: Evaluate OPA policy
        id: opa-eval
        run: |
          # 将 Trivy 结果传入 OPA 策略评估
          result=$(./opa eval --input trivy-results.json \
            --data policies/trivy-scan.rego \
            --format json \
            "data.trivy.policy.allow")
          
          echo "result=$result" >> $GITHUB_OUTPUT
          
          # 如果不允许，输出阻断信息
          if echo "$result" | jq -e '.result == [false]' > /dev/null; then
            echo "=== Policy Violations ==="
            ./opa eval --input trivy-results.json \
              --data policies/trivy-scan.rego \
              --format pretty \
              "data.trivy.policy.deny"
            exit 1
          fi
      
      - name: Scan IaC configurations
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: ./deploy/
          format: sarif
          output: trivy-config.sarif

      - name: Upload Trivy results to GitHub Security tab
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-config.sarif
```

### 6.2 GitLab CI 集成

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - security

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

build:
  stage: build
  script:
    - docker build -t $DOCKER_IMAGE .
  only:
    - main

trivy-scan:
  stage: security
  script:
    # 安装 Trivy
    - apt-get update && apt-get install -y trivy
    # 扫描镜像
    - trivy image --format json --output trivy-results.json $DOCKER_IMAGE
    # 下载 OPA
    - curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64_static
    - chmod +x opa
    # 策略评估
    - ./opa eval --input trivy-results.json --data policies/ --format json "data.trivy.policy.allow" | jq -e '.result[0]'
  only:
    - main

push-image:
  stage: security
  script:
    - docker push $DOCKER_IMAGE
  needs: ["trivy-scan"]
  only:
    - main
```

## 7. 实战：阻断不安全镜像的部署

### 7.1 完整流程演示

```bash
# 1. 构建镜像
docker build -t myapp:latest .

# 2. 扫描镜像
trivy image --format json --output trivy-results.json myapp:latest

# 3. 查看扫描摘要
trivy image --severity CRITICAL,HIGH myapp:latest

# 4. OPA 策略判定
# 模拟一个带 CRITICAL 漏洞的输入
cat > test-input.json << 'EOF'
{
  "Results": [{
    "Vulnerabilities": [{
      "VulnerabilityID": "CVE-2024-9999",
      "PkgName": "libssl3",
      "InstalledVersion": "3.0.8",
      "FixedVersion": "3.0.9",
      "Severity": "CRITICAL",
      "Title": "Buffer overflow in libssl3"
    }]
  }]
}
EOF

opa eval --input test-input.json --data policies/trivy-scan.rego \
  --format pretty "data.trivy.policy"

# 输出：
# {
#   "allow": false,
#   "deny": ["CRITICAL vulnerability found: CVE-2024-9999 in libssl3 (3.0.8)"]
# }

# 5. 阻断构建（CI 中 exit 1）
```

### 7.2 用 Docker Compose 做本地测试

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  trivy-scanner:
    image: aquasec/trivy:latest
    volumes:
      - .:/project
      - trivy-db:/root/.cache/trivy
    command: image --severity CRITICAL,HIGH myapp:test

  opa-evaluator:
    image: openpolicyagent/opa:latest
    volumes:
      - ./policies:/policies
      - ./trivy-results.json:/input.json
    command: eval --input /input.json --data /policies --format pretty "data.trivy.policy"

volumes:
  trivy-db:
```

## 8. 让策略随时间演进

安全策略不是一成不变的。建议：

1. **渐进式收严**：初期只阻断 CRITICAL 漏洞，稳定后再纳入 HIGH 级别的量化控制
2. **例外管理**：建立漏洞豁免流程——已知风险、已计划修复的漏洞可以通过 `data.json` 加白名单
3. **策略版本化**：把策略文件放在 Git 仓库，每次修改走 PR 评审

下面是一个支持豁免的扩展策略：

```rego
# policies/trivy-scan-wl.rego
package trivy.policy

# 从外部数据加载豁免列表
exemptions := input_exemptions {
    data.exemptions
} else := []

# 不在豁免列表中的 CRITICAL 漏洞才阻断
deny[msg] {
    vuln := input.Results[_].Vulnerabilities[_]
    vuln.Severity == "CRITICAL"
    not vuln.VulnerabilityID == exemptions[_]
    msg := sprintf("BLOCKED: %s", [vuln.VulnerabilityID])
}

allow {
    count(deny) == 0
}
```

对应的 `data.json`：

```json
{
    "exemptions": ["CVE-2024-0001", "CVE-2024-0002"]
}
```

## 9. 总结

DevSecOps 不是买一个工具装上去就完事，而是**策略 + 工具 + 流程**三者结合：

| 组件 | 工具选择 | 职责 |
|------|----------|------|
| 漏洞扫描 | Trivy | 发现漏洞 |
| 策略引擎 | OPA | 判定是否放行 |
| CI 平台 | GitHub Actions / GitLab CI | 串联扫描和判定 |
| 存储 | Git + 制品仓库 | 追溯和审计 |

Trivy 负责说"有什么问题"，OPA 负责说"允不允许上线"。两者配合，让安全从"最终检查"变成"持续验证"。

下一步可以拓展的方向：

- 加入 **SBOM 生成** (Trivy 支持 `--format cyclonedx`) 做供应链安全管理
- 集成 **Kubernetes Admission Webhook**，把 OPA 策略部署到集群层面，阻止运行时不安全的镜像
- 用 **Cosign** 对镜像签名，构建从构建到部署的完整信任链

**应该把安全放在哪里？** 不是放在最后，而是放在每次代码提交的流水线里。这就是 DevSecOps 的真正含义。