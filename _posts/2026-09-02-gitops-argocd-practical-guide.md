---
layout: post
title: "GitOps 与 ArgoCD 实战指南"
date: 2026-09-02 09:00:00 +0800
categories: [开发]
tags: [GitOps, ArgoCD, Kubernetes, DevOps, CI/CD, 自动化]
---

## 引言

传统的 CI/CD 推送模型（Push-based）将构建产物直接推送到目标环境，部署状态与 Git 仓库之间缺乏强关联。随着集群规模增长，环境漂移（Configuration Drift）、回滚困难、权限分散等问题日益突出。

GitOps 提供了一种替代范式：**以 Git 仓库作为集群状态的唯一事实来源（Single Source of Truth）**，所有变更通过 Pull Request 审批后合并，再由 Operator 自动将声明状态同步到集群中。ArgoCD 是目前 Kubernetes 生态中最成熟的 GitOps 实现，本文将带你从零搭建一套完整的 GitOps 工作流。

## 1. GitOps 核心原则

GitOps 的核心理念可以概括为三条：

1. **声明式配置**：整个集群的期望状态用 YAML 或 Helm Chart 描述，存储在 Git 仓库中
2. **不可变基础设施**：不做原地修改，所有变更通过更新 Git 仓库中的声明文件完成
3. **自动同步**：Operator 持续对比 Git 仓库与集群实际状态，自动修复漂移

与传统的 `kubectl apply` 或 `helm upgrade --install` 不同，GitOps 模式下**没有人直接操作集群**——所有操作都通过 Git 工作流完成，天然具备审计、回滚、权限控制能力。

## 2. 环境准备

### 2.1 安装 ArgoCD

在 Kubernetes 集群中安装 ArgoCD 非常简单：

```bash
# 创建命名空间
kubectl create namespace argocd

# 安装 ArgoCD（生产环境）
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 等待所有 Pod 就绪
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

安装完成后，暴露 API Server 以便访问 UI：

```bash
# 方式一：修改 Service 为 LoadBalancer
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# 方式二：使用 Port Forward（本地开发）
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 2.2 获取初始密码

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

默认用户名是 `admin`，首次登录后建议立即修改密码：

```bash
argocd account update-password
```

### 2.3 安装 argocd CLI

```bash
# Linux
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd

# 登录
argocd login <ARGOCD_SERVER> --username admin
```

## 3. 定义第一个 Application

### 3.1 仓库结构设计

一个典型的 GitOps 仓库结构如下：

```
infra-repo/
├── apps/                    # 应用定义
│   ├── nginx/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── redis/
│       └── ...
├── envs/                    # 环境配置
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
└── clusters/                # 集群配置
    └── production/
        └── argocd-apps.yaml
```

### 3.2 创建示例应用

先创建一个简单的 Nginx 部署：

```yaml
# apps/nginx/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  labels:
    app: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

```yaml
# apps/nginx/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo
spec:
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```yaml
# apps/nginx/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

### 3.3 通过 CLI 创建 Application

```bash
argocd app create nginx-demo \
  --repo https://github.com/your-org/infra-repo.git \
  --path apps/nginx \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

参数说明：
- `--sync-policy automated`：启用自动同步，仓库变更后自动部署
- `--auto-prune`：Git 中删除的资源自动从集群中删除
- `--self-heal`：集群中手动修改的资源会被 Git 状态覆盖

### 3.4 声明式 Application 定义

更推荐的方式是通过 YAML 定义 Application，并将其本身也纳入 Git 管理（App of Apps 模式）：

```yaml
# clusters/production/nginx-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-demo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/infra-repo.git
    targetRevision: main
    path: apps/nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
```

提交后，ArgoCD 会自动感知并同步：

```bash
git add clusters/production/nginx-app.yaml
git commit -m "add nginx-demo application"
git push
```

## 4. 高级同步策略

### 4.1 手动同步与波次同步

对于需要按顺序部署的应用（如先部署数据库迁移，再部署应用），可以使用 Sync Waves：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/sync-wave: "-5"  # 负值越小的越先执行
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp/migration:latest
      restartPolicy: Never
```

Sync Waves 的排序规则：
- Wave 值越小，执行越早
- 同一 Wave 内的资源并行部署
- 前一 Wave 的所有资源健康后，才进入下一 Wave

### 4.2 渐进式交付（Progressive Delivery）

结合 Argo Rollouts 实现蓝绿部署或金丝雀发布：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-rollout
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {duration: 30s}
        - setWeight: 50
        - pause: {duration: 30s}
        - setWeight: 100
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:v2.0.0
```

使用 Argo Rollouts 替代 Deployment 后，ArgoCD 会自动管理 Rollout 资源，每次更新都会经过灰度放大流程。

## 5. 多环境与多集群管理

### 5.1 通过 Kustomize 管理环境差异

```yaml
# envs/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../apps/nginx
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
    target:
      kind: Deployment
```

```yaml
# envs/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../apps/nginx
patches:
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
    target:
      kind: Deployment
```

### 5.2 多集群注册

ArgoCD 可以管理多个 Kubernetes 集群：

```bash
# 添加远程集群
argocd cluster add context-name \
  --name production-cluster \
  --kubeconfig /path/to/kubeconfig
```

然后在 Application 的 `destination` 中指定集群：

```yaml
spec:
  destination:
    server: https://production-cluster.example.com:6443
    namespace: production
```

## 6. 安全最佳实践

### 6.1 仓库访问控制

使用 Deploy Key 代替 Personal Access Token：

```bash
# 生成专用 SSH 密钥
ssh-keygen -t ed25519 -C "argocd@your-org" -f argocd-deploy-key

# 将公钥添加到仓库的 Deploy Keys 中（只读权限）
# 在 ArgoCD 中创建 Repository Secret
argocd repo add git@github.com:your-org/infra-repo.git \
  --ssh-private-key-path argocd-deploy-key
```

### 6.2 敏感信息管理

**永远不要**在 Git 仓库中明文存储密码或密钥。使用 Sealed Secrets 或 External Secrets Operator：

```yaml
# 使用 Sealed Secrets
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
spec:
  encryptedData:
    password: AgBy3i4...  # 加密后的密文
```

Sealed Secrets 的工作原理：
1. 在本地用 `kubeseal` 工具加密 Secret
2. 加密后的 SealedSecret 可以安全地提交到 Git
3. ArgoCD 同步后，集群内的 Sealed Secrets Controller 自动解密为普通 Secret

### 6.3 RBAC 与审计

ArgoCD 支持细粒度的 RBAC 配置：

```yaml
# argocd-rbac-cm ConfigMap
data:
  policy.csv: |
    p, role:dev, applications, get, */*, allow
    p, role:dev, applications, sync, */*, deny
    p, role:admin, applications, *, */*, allow
    p, role:admin, clusters, *, *, allow

    g, alice, role:dev
    g, bob, role:admin
```

所有通过 ArgoCD 执行的同步操作都会记录在审计日志中：

```bash
# 查看同步历史
argocd app get nginx-demo
argocd app history nginx-demo
```

## 7. 实际工作流

一个完整的 GitOps 工作流示例如下：

```
开发者 → 修改 YAML → 提交 PR → CI 检查（kubeval + conftest）
  → 审批合并 → Git 仓库更新 → Webhook 通知 ArgoCD
  → ArgoCD 拉取最新配置 → 对比集群状态 → 自动同步
  → 同步结果通知（Slack/邮件）
```

配置 CI 中的策略检查（使用 Conftest）：

```yaml
# .github/workflows/gitops-check.yaml
name: GitOps Validation
on: [pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate Kubernetes manifests
        uses: instrumenta/kubeval-action@master
        with:
          files: ./apps
      - name: Policy check with Conftest
        uses: open-policy-agent/conftest-action@v2
        with:
          files: ./apps
          policy: ./policy
```

## 8. 常见问题与排障

### 8.1 同步状态 OutOfSync

```bash
# 查看差异
argocd app diff nginx-demo

# 强制同步
argocd app sync nginx-demo --prune

# 查看同步结果
argocd app get nginx-demo -o wide
```

### 8.2 同步超时

调整 Application 的同步超时时间：

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
    syncOptions:
      - Validate=false
      - Timeout=300  # 秒
```

### 8.3 资源卡在 Progressing 状态

检查资源健康检查配置：

```yaml
# 自定义健康检查
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
```

## 总结

GitOps + ArgoCD 将 Git 工作流的成熟理念引入 Kubernetes 部署管理，带来了几个关键收益：

- **可审计**：每一次部署变更都有 Git 记录，谁、什么时间、改了什么都清晰可查
- **可回滚**：`git revert` 即可恢复到任意历史版本，无需连接集群
- **可复现**：从零到集群部署，只需克隆仓库并运行 ArgoCD
- **自愈**：集群状态自动与 Git 对齐，手动修复的「救火」操作消失

如果你的团队还在用 `kubectl apply` 管理生产环境，或者每次发版都提心吊胆，GitOps 是值得认真考虑的方向。从一个小应用开始，逐步扩展到全集群，这套方法论会让你的部署流程变得可预测、可重复、可依赖。