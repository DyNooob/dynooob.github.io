---
layout: post
title: "Kubernetes Pod Security 实战：从 PSP 到 Pod Security Admission"
date: 2026-07-23 09:00:00 +0800
categories: [网络安全]
tags: [Kubernetes, PodSecurity, PSA, PSS, 容器安全, 安全基线, 安全开发]
---

Kubernetes 的安全模型里，最容易被忽视的一环是 **Pod 安全策略**。默认情况下，集群中的任何 Pod 都能以 root 运行、挂载主机路径、使用 hostNetwork、提权——这些能力一旦被恶意容器或受损镜像利用，整个集群的隔离边界就形同虚设。

Kubernetes v1.21 弃用了旧版的 PodSecurityPolicy（PSP），v1.23 引入了 Pod Security Admission（PSA）作为替代方案。v1.25 起 PSP 彻底移除。如果你还在用旧版集群，或者刚刚迁移到新版本，这篇指南会帮你一次性搞清楚 PSA 的用法、迁移路径和常见坑。

## 为什么 PSP 被放弃了

PodSecurityPolicy（PSP）的问题在于它的设计太「重」了：

1. **RBAC 耦合混乱**——你需要为每个 PSP 创建对应的 ClusterRole、RoleBinding，绑定到 ServiceAccount。一个策略需要 3-4 个 YAML 文件才能生效。
2. **没有默认策略**——新建集群没有任何 PSP 生效，安全团队以为「有了 PSP 就安全了」，实际上开发者根本没被限制到。
3. **审计困难**——PSP 不提供清晰的违规报告，遇到 Pod 创建失败，你得反复翻 API Server 日志才能定位到是哪个策略拦住了。
4. **Admission 顺序问题**——PSP 作为 mutating + validating webhook 同时存在，策略评估顺序不透明，经常出现预期外的 allow/deny 结果。

社区花了两年多才承认 PSP 的设计有问题，最终在 v1.25 彻底移除，用 PSA 替代。

## Pod Security Admission 的核心概念

PSA 的精髓就三个元素：**标准（Standards）**、**模式（Modes）**、**层级（Levels）**。

### 三个安全层级

| 层级 | 名称 | 说明 | 典型限制 |
|------|------|------|----------|
| `privileged` | 特权 | 无限制，信任的 workloads 使用 | 无限制 |
| `baseline` | 基线 | 最小限制，能防止已知权限提升 | 禁止 privileged 容器、hostNetwork、hostPID、hostIPC、hostPath 卷等 |
| `restricted` | 严格 | 强化限制，遵循 Pod 安全最佳实践 | 禁止所有 baseline 限制项 + 限制 seccomp、AppArmor、SELinux、capabilities、seccompProfile |

### 三种执行模式

| 模式 | 行为 |
|------|------|
| `enforce` | 违反策略 → Pod 被拒绝创建 |
| `audit` | 违反策略 → 记录审计事件，允许创建 |
| `warn` | 违反策略 → 返回警告给用户，允许创建 |

这三个模式可以独立设置，互不干扰。这意味着你可以同时做：

- 对 `default` 命名空间 enforce baseline，拒绝违规 Pod
- 对 `production` 命名空间 enforce restricted，严格限制
- 对 `legacy` 命名空间 audit restricted + warn baseline，观察但不阻断

### 两种配置方式

**方式一：命名空间标签（推荐）**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

**方式二：AdmissionConfiguration 资源（全局配置）**

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: [kube-system, kube-public]
```

生产环境建议用命名空间标签的方式，这样每个团队可以独立控制自己的命名空间策略，不需要改 API Server 启动参数。

## 实战：从零配置 PSA

### 第一步：检查集群版本

```bash
kubectl version --short
# 确保 Server Version >= v1.23
```

PSA 在 v1.23 进入 Beta，v1.25 正式 GA。如果你的集群版本低于 v1.23，先升级集群。

### 第二步：验证 PSA 是否启用

```bash
kubectl -n kube-system get pods -l component=kube-apiserver -o yaml | grep -A5 PodSecurity
```

如果输出中包含 `PodSecurity` 插件，说明 PSA 已经启用。大多数集群管理工具（kubeadm、RKE、EKS、AKS、GKE）在 v1.23+ 默认启用。

### 第三步：为现有命名空间添加标签

不要一上来就 enforce，不然现有 Pod 可能全部被驱逐。正确的做法是先用 audit + warn 模式观察。

```bash
# 先审计 + 警告，观察现有 workload
kubectl label ns default \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/audit-version=latest \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/warn-version=latest
```

### 第四步：查看审计日志

```bash
# 查看 audit 模式记录的事件
kubectl get events -n default | grep -i PodSecurity
```

或者直接查 API Server 日志：

```bash
kubectl -n kube-system logs -l component=kube-apiserver --tail=1000 | grep "PodSecurity"
```

### 第五步：确认没有违规后，启用 enforce

```bash
kubectl label ns default \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  --overwrite
```

### 第六步：验证

创建一个违反 baseline 策略的 Pod 试试：

```bash
kubectl run test-privileged --image=nginx --privileged
# 应该被拒绝：Error: container has securityContext.privileged=true
```

创建一个合规的 Pod：

```bash
kubectl run test-ok --image=nginx --restart=Never
kubectl delete pod test-ok
```

## 命名空间级别的策略矩阵

规模型集群通常有多个命名空间，每个命名空间的安全需求不同。下面是一个推荐的分级配置：

| 命名空间 | 用途 | Enforce 层级 | 说明 |
|----------|------|-------------|------|
| `kube-system` | 系统组件 | 豁免 | 部分组件需要 hostNetwork / privileged |
| `gatekeeper-system` | 策略引擎 | 豁免 | 需要特权运行 OPA/Gatekeeper |
| `monitoring` | 监控系统 | baseline | 大部分采集器不需要特权 |
| `production` | 生产业务 | restricted | 严格限制，最小权限 |
| `staging` | 预发环境 | restricted | 和生产一致，提前发现问题 |
| `development` | 开发环境 | baseline | 灵活一点，但也要有底线 |
| `ci-cd` | CI/CD Runner | baseline | Runner 需要 docker-in-docker 或特权时单独处理 |

```yaml
# 生产命名空间
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

## 绕过受限策略的合法方式

有些场景确实需要提权——比如使用 Istio sidecar、Cilium CNI 的 DaemonSet、或者监控系统需要访问宿主机网络。这时候不能用特权命名空间一刀切，而是用 **exemptions** 或 **AdmissionConfiguration** 精确控制。

### 方法一：命名空间豁免

在 AdmissionConfiguration 中配置豁免的命名空间：

```yaml
exemptions:
  namespaces: [kube-system, istio-system, cert-manager]
```

### 方法二：RuntimeClass 豁免

如果你的集群使用 gVisor、Kata Containers 等安全沙箱 runtime，可以豁免这些 RuntimeClass：

```yaml
exemptions:
  runtimeClasses: [gvisor, kata]
```

### 方法三：用户豁免

特定用户（比如 CI/CD ServiceAccount）可以豁免：

```yaml
exemptions:
  usernames: [system:serviceaccount:ci-cd:runner]
```

## 迁移：从 PSP 到 PSA

如果你还在用 PSP，迁移到 PSA 的步骤：

### 1. 评估现有 PSP 策略

```bash
# 列出所有 PSP
kubectl get psp
# 检查每个 PSP 的规则
kubectl get psp <name> -o yaml
```

### 2. 映射到对应的 PSA 层级

PSP 的每个规则都能映射到 PSA 的三个层级之一：

| PSP 规则 | PSA 层级 |
|----------|----------|
| `privileged: true` | privileged |
| `hostNetwork: true` | baseline 及以上 |
| `hostPID: true` | baseline 及以上 |
| `hostIPC: true` | baseline 及以上 |
| `allowedHostPaths` | baseline 及以上 |
| `seLinux` | baseline 及以上限制 SELinux type |
| `runAsUser: MustRunAsNonRoot` | restricted |
| `seccomp: RuntimeDefault` | restricted |
| `capabilities: drop: ALL` | restricted |
| `allowPrivilegeEscalation: false` | restricted |

### 3. 分阶段迁移

**阶段一：PSP + PSA 共存（v1.21-v1.24）**

```bash
# 添加 audit + warn 标签，观察违规
kubectl label ns production \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted
```

**阶段二：逐个命名空间切换**

修复所有 audit 日志中的违规后，启用 enforce。

**阶段三：删除 PSP 资源**

```bash
kubectl delete psp --all
```

### 4. 常见违规场景与修复

**场景 1：privileged: true**

```yaml
# 违规
securityContext:
  privileged: true

# 修正：去掉 privileged，改用 capabilities
securityContext:
  capabilities:
    add: ["NET_ADMIN", "SYS_ADMIN"]
```

**场景 2：runAsUser: 0（root 运行）**

```yaml
# 违规
securityContext:
  runAsUser: 0

# 修正：使用非 root 用户
securityContext:
  runAsUser: 1000
  runAsNonRoot: true
```

**场景 3：没有设置 seccompProfile**

```yaml
# 违规（restricted 层级要求）
# 没有 seccompProfile 字段

# 修正
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

**场景 4：先添加 capabilities 但没 drop all**

```yaml
# 违规（restricted 层级要求 drop ALL）
securityContext:
  capabilities:
    add: ["NET_ADMIN"]

# 修正
securityContext:
  capabilities:
    drop: ["ALL"]
    add: ["NET_ADMIN"]
```

## 用工具辅助迁移

社区有几个工具可以帮你加速 PSP 到 PSA 的迁移：

### kubectl-who-can

```bash
# 查看哪些 Pod 实际使用了某个 PSP
kubectl who-can use psp/restricted
```

### psp-psa-migrator

```bash
# 分析现有 PSP 并输出对应的 PSA 标签建议
git clone https://github.com/kubernetes-sigs/psp-psa-migrator
cd psp-psa-migrator
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python3 main.py --psp-list <(kubectl get psp -o name)
```

### kyverno 的 PSA 分析能力

如果你用 Kyverno 做策略引擎，它的 `psa-pss` 策略可以帮你验证 Pod 是否合规：

```bash
# 安装 Kyverno 并启用 PSA 策略
kubectl create -f https://raw.githubusercontent.com/kyverno/policies/main/pod-security/baseline/disallow-privileged-containers/disallow-privileged-containers.yaml
```

## 严格模式下的常见陷阱

### 陷阱 1：Init Container 也被检查

PSA 不仅检查主容器，也检查 Init Container 和 Ephemeral Container。一个常见的错误是只在主容器设了 `securityContext`，Init Container 没设：

```yaml
# 会失败——init container 没有 securityContext
initContainers:
- name: init
  image: busybox
  command: ["sh", "-c", "echo init"]
containers:
- name: app
  image: myapp:latest
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
```

### 陷阱 2：Pod SecurityContext vs Container SecurityContext

Pod 级别的 securityContext 会被所有容器继承，但如果容器自己定义了 securityContext，则容器的定义覆盖 Pod 的定义。restricted 层级要求每个容器都满足条件，不是只检查 Pod 级别的。

### 陷阱 3：Ephemeral Containers 也需要合规

`kubectl debug` 创建的临时容器也在 PSA 的检查范围内。如果你在 restricted 命名空间里 debug，需要 `--image=busybox:1.36` 这样非 root 镜像，或者临时改命名空间标签。

### 陷阱 4：namespace 标签的继承问题

命名空间标签只影响该命名空间内的 Pod 创建，不影响子资源如 Deployment、StatefulSet 的创建。但 Deployment 创建的 Pod 会被检查，所以如果你在 Deployment 里写了违规的 `template.spec`，Deployment 创建成功，但 Pod 创建失败，Deployment 的 ReplicaSet 会一直卡在 `CreateContainerConfigError`。

## 生产环境建议

1. **先 audit + warn，再 enforce**——不要在生产集群上一上来就 enforce restricted，先跑一周观察。
2. **使用 `latest` 版本标签**——`enforce-version: latest` 会自动跟随集群版本升级。如果不想自动升级，可以固定到 `v1.28` 这样的具体版本。
3. **结合 OPA/Gatekeeper 或 Kyverno**——PSA 只覆盖 Pod 安全，不覆盖其他资源（如 NetworkPolicy、Ingress、RBAC）。对于更复杂的策略，用 OPA/Gatekeeper 或 Kyverno 做补充。
4. **监控 PSA 违反事件**——配置 Prometheus 告警规则，当 PSA 拒绝事件超过阈值时告警：

```yaml
# PrometheusRule
groups:
- name: psa-violations
  rules:
  - alert: PSAViolation
    expr: increase(apiserver_admission_webhook_rejection_count{name="podsecurity"}[5m]) > 0
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "PSA 拒绝了一个 Pod 创建请求"
```

5. **CI/CD 中提前验证**——在 CI 流水线中用 `kubectl dry-run` 验证 Pod 规格是否合规：

```bash
kubectl apply -f deployment.yaml --dry-run=server --validate=false 2>&1 \
  | grep -i "PodSecurity" && echo "PSA 违规，请检查" || echo "合规"
```

## 总结

Pod Security Admission 是 Kubernetes 内置的 Pod 安全机制，相比 PSP 有三大优势：配置简单（一个标签就够）、审计友好（Events + 日志清晰）、迁移平滑（audit 模式无痛观察）。但它不是银弹——它只解决 Pod 规格层面的安全问题，不覆盖网络策略、RBAC、镜像扫描等维度。安全是分层防御，PSA 是你的第一道防线，但不是最后一道。

如果你正在从 PSP 迁移到 PSA，记住三个原则：先观察再执行、从 baseline 开始逐步收紧、用工具辅助分析。迁移完成后，restricted 层级应该成为生产环境的默认配置，privileged 层级只应该出现在明确豁免的命名空间中。