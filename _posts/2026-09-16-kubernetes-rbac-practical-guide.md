---
layout: post
title: "Kubernetes RBAC 实战指南：权限模型与最小权限策略"
date: 2026-09-16 09:00:00 +0800
categories: [网络安全, 容器化]
tags: [kubernetes, rbac, security, access-control, devsecops, authorization]
---

## 为什么需要 RBAC

Kubernetes 的 API 是集群的唯一操作入口——`kubectl get pods`、`kubectl delete node`、`kubectl create deployment`，所有动作最终都通过 API Server 完成。如果没有权限控制，任何能访问集群的人都能删光所有 Namespace，或者窃取 Secrets 中的数据库密码。

Role-Based Access Control（RBAC）是 Kubernetes 默认且推荐的身份验证之后的授权机制。从 v1.8 开始就是稳定的 GA 功能，v1.22 之后移除了遗留的 ABAC 支持，RBAC 成为唯一内置的授权方案。

本文不讲理论框架，直接从实战出发：**如何设计、部署、调试和审计 RBAC 策略**，确保集群安全的同时不给日常运维拖后腿。

## 四个核心对象

RBAC 只有四个资源，理解它们的关系就掌握了 90%：

| 对象 | 作用域 | 用途 |
|------|--------|------|
| **Role** | Namespace | 授予某个 Namespace 内的资源权限 |
| **ClusterRole** | 集群级 | 授予集群范围的资源权限，或作为公共模板绑定到 Namespace |
| **RoleBinding** | Namespace | 将 Role/ClusterRole 绑定到某个 Namespace 内的用户/SA |
| **ClusterRoleBinding** | 集群级 | 将 ClusterRole 绑定到集群范围 |

### 授权链路

```
Subject（User/Group/SA）
    → 被 RoleBinding / ClusterRoleBinding 引用
    → 关联一个 Role 或 ClusterRole
    → Role/ClusterRole 中包含允许的 verbs + resources
```

### verb 命名约定

所有 verb 对应 HTTP 方法：

| Verb | HTTP 方法 | 典型用途 |
|------|-----------|---------|
| `get` | GET | 读取单个资源 |
| `list` | GET (collection) | 列出资源 |
| `watch` | GET (watch) | 长连接监听变更 |
| `create` | POST | 创建资源 |
| `update` | PUT | 全量更新 |
| `patch` | PATCH | 部分更新 |
| `delete` | DELETE | 删除单个资源 |
| `deletecollection` | DELETE (collection) | 批量删除 |

## 最小权限：从零搭建一个 Namespace 隔离环境

### 场景

团队 A 负责 `team-a` Namespace，要求：
- 可以完全管理该 Namespace 下的 Pod、Deployment、Service、ConfigMap
- 可以读取 Secrets（不能创建或修改）
- 可以查看 Pod 日志
- 不能操作其他 Namespace
- 不能操作 Node、ClusterRole 等集群级资源

### 第一步：创建 ServiceAccount

```yaml
# sa-team-a.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-a-sa
  namespace: team-a
```

### 第二步：创建 Role

```yaml
# role-team-a.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: team-a
  name: team-a-role
rules:
  # 核心工作负载完全控制
  - apiGroups: ["apps"]
    resources: ["deployments", "deployments/scale", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Pod 管理（含执行和日志）
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec", "pods/portforward"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # 网络相关
  - apiGroups: [""]
    resources: ["services", "endpoints", "endpointslices"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # 配置只读 + ConfigMap 创建
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Secrets **只读**——不能读取明文密码后篡改
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
  # Ingress 管理
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # PVC 管理
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Events 只读（排查问题用）
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
```

注意 `secrets` 只有 `get/list/watch`——这是最小权限原则的关键。CI/CD 流水线经常被赋予 secrets 的 `create/update` 权限，实际上这是最大的安全漏洞之一。

### 第三步：绑定

```yaml
# rb-team-a.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-binding
  namespace: team-a
subjects:
  - kind: ServiceAccount
    name: team-a-sa
    namespace: team-a
roleRef:
  kind: Role
  name: team-a-role
  apiGroup: rbac.authorization.k8s.io
```

### 验证

```bash
# 使用 SA 的 token 进行认证
SA_TOKEN=$(kubectl -n team-a create token team-a-sa)

# 设置 kubeconfig
kubectl --token="$SA_TOKEN" -n team-a get pods
kubectl --token="$SA_TOKEN" -n team-a get secrets
kubectl --token="$SA_TOKEN" -n team-a delete pod some-pod  # 拒绝

# 跨 Namespace 应该被拒绝
kubectl --token="$SA_TOKEN" -n team-b get pods
# Error from server (Forbidden):
```

## ClusterRole 的三种使用模式

### 模式一：集群级资源授权

Node、PV、Namespace、ClusterRole 本身是集群作用域的资源，只能由 ClusterRole 授权：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/proxy", "nodes/stats"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["namespaces", "persistentvolumes"]
    verbs: ["get", "list", "watch"]
```

### 模式二：作为公共模板通过 RoleBinding 注入

这是生产中最常见的用法：定义一组通用权限的 ClusterRole，然后用 RoleBinding 绑定到不同 Namespace，无需在每个 Namespace 重复定义 Role。

Kubernetes 内置了四个这样的 ClusterRole：`view`、`edit`、`admin`、`cluster-admin`。

| 内置 ClusterRole | 权限 |
|------------------|------|
| `cluster-admin` | 完全控制所有资源 |
| `admin` | Namespace 范围完全控制（不能修改 ResourceQuota 和 Namespace 本身） |
| `edit` | 可读写大部分资源，不能读/改 Secrets 和 RBAC |
| `view` | 只读，不能读 Secrets |

```bash
# 给 team-b 的所有用户只读权限
kubectl create rolebinding team-b-view --clusterrole=view \
  --group=team-b-readers --namespace=team-b
```

### 模式三：聚合 ClusterRole

当你需要组合多个 ClusterRole 的权限（比如同时拥有 `view` + `node-reader`），可以用 `aggregationRule`：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-ops
  labels:
    rbac.aggregation/ops: "true"
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.aggregation/ops: "true"
rules: []  # 规则由匹配到的 ClusterRole 合并而来
```

任何带有 `rbac.aggregation/ops: "true"` 标签的 ClusterRole 会自动合并到 `custom-ops` 中。这在大型团队中非常有用：基础架构组加一个 Node 只读 ClusterRole 打上标签，Ops 组的成员就自动获得了权限。

## 调试 RBAC 的四个利器

### 1. `kubectl auth can-i` — 快速验证

```bash
# 当前用户能否删除 Pod
kubectl auth can-i delete pods

# 指定用户/SA 能否在指定 Namespace 操作
kubectl auth can-i get secrets --as=system:serviceaccount:team-a:team-a-sa -n team-a

# 列出所有被拒绝的操作（模拟）
kubectl auth can-i --list --as=team-a-sa -n team-a

# 查看拒绝原因（verbose 模式）
kubectl auth can-i delete pods --as=team-a-sa -n team-b -v=8
```

### 2. `kubectl describe rolebinding`

```bash
kubectl describe rolebinding team-a-binding -n team-a
```

输出清楚地显示 Subjects 和引用的 RoleRef。

### 3. `kubectl get roles --all-namespaces` 审计所有 Role

```bash
kubectl get roles --all-namespaces -o wide
```

配合 `-o yaml` 可以检查是否有过度授权的 Role。

### 4. 审计日志（API Server 审计）

这是最强大的排查手段。启用 API Server 审计日志后，每次授权决策都会被记录：

```yaml
# kube-apiserver 启动参数或配置文件
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # 记录所有 RBAC 拒绝事件
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods", "secrets", "configmaps"]
    verbs: ["create", "update", "patch", "delete"]
    # 特别关注 Forbidden
    stage: ResponseComplete
  - level: Metadata
    users: ["system:serviceaccount:kube-system:*"]
    stage: RequestReceived
```

审计日志格式（JSON 行）：

```json
{
  "kind": "Event",
  "level": "Metadata",
  "user": { "username": "system:serviceaccount:team-a:team-a-sa" },
  "verb": "delete",
  "objectRef": {
    "resource": "pods",
    "namespace": "team-a",
    "name": "web-7d8f9c"
  },
  "responseStatus": { "code": 403, "reason": "Forbidden" },
  "stage": "ResponseComplete"
}
```

## ServiceAccount Token 管理

### 自动挂载控制

从 Kubernetes v1.24 开始，ServiceAccount 不再自动创建 Secret 类型的 token。推荐做法：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-a-sa
automountServiceAccountToken: false  # 全局禁止
```

对于**不需要**访问 API 的 Pod（比如只跑业务逻辑的 Web 应用），应显式禁止 token 挂载：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  automountServiceAccountToken: false
  serviceAccountName: team-a-sa
  containers:
    - name: app
      image: nginx
```

### TokenRequest API 获取短期 Token

```bash
# 创建有效期为 1 小时的 Token
kubectl create token team-a-sa -n team-a --duration=1h

# 给 CI/CD 使用
kubectl create token ci-sa -n ci --duration=168h > /tmp/ci-kubeconfig-token
```

TokenRequest API 返回的是绑定 Pod 生命周期的投射卷挂载（Projected Volume），比静态 Secret 更安全。

## 生产环境最佳实践

### 1. 禁止通配符

```yaml
# ❌ 错误：* 太宽泛，新增资源会自动获得权限
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

# ✅ 正确：明确列出所需资源
rules:
  - apiGroups: ["apps", "extensions"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### 2. 使用 Groups 而不是单个 User

不要在 RoleBinding 中绑定个人用户名，而是绑定 LDAP/OIDC Group：

```yaml
subjects:
  - kind: Group
    name: platform-engineers     # OIDC group
    apiGroup: rbac.authorization.k8s.io
```

这样人员的入职离职只需在 IAM 系统管理，无需改 Kubernetes RBAC。

### 3. 定期审计过度授权的 ServiceAccount

```bash
# 找出所有具有 cluster-admin 权限的非系统 SA
kubectl get clusterrolebinding -o json | jq -r '
  .items[] |
  select(.roleRef.name == "cluster-admin") |
  .subjects[] |
  select(.kind == "ServiceAccount") |
  "\(.namespace)/\(.name)"'
```

### 4. 使用工具做策略即代码

**kubectl-who-can**：反向查询谁有什么权限

```bash
# 安装 kubectl 插件
kubectl krew install who-can

# 查询谁能删除 Pod
kubectl who-can delete pods -n team-a
```

**polaris**：审计集群配置安全

```bash
polaris audit --audit-path rbac --only-show-fails
```

**OPA Gatekeeper**（参考之前 DevSecOps 文章）：强制 RBAC 约束，如"禁止通配符 verb"、"禁止 cluster-admin 给非系统用户"等。

### 5. SubjectAccessReview API

程序化权限校验：

```bash
cat <<EOF | kubectl create -f -
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: system:serviceaccount:team-a:team-a-sa
  namespace: team-b
  resource: "pods"
  verb: "list"
EOF
```

返回：

```yaml
status:
  allowed: false
  reason: 'RBAC: denied by ...'
```

## 常见陷阱

### 误用 verbs 顺序

`*` 不是合法的 verb——它是 `get, list, watch, create, update, patch, delete, deletecollection` 的简写，但写法和 `["*"]` 一样。出错时检查 `kubectl describe` 会显示展开后的列表。

### 子资源未单独授权

```yaml
# ❌ 只给了 pods 权限，没给 pods/log
resources: ["pods"]
verbs: ["get"]

# ✅ 需要显式声明子资源
resources: ["pods", "pods/log"]
verbs: ["get"]
```

### 忘记 apiGroups

不同资源属于不同的 API Group。`deployments` 属于 `apps` Group，`ingresses` 属于 `networking.k8s.io`，`pods` 属于核心 Group（`""`）。

```yaml
# ❌ 这样写不会授权任何东西
resources: ["deployments"]
verbs: ["get"]

# ✅ 加上 apiGroups
apiGroups: ["apps"]
resources: ["deployments"]
verbs: ["get"]
```

### 误认为 RoleBinding 可以跨 Namespace

RoleBinding 的 subjects 和 roleRef 中的 namespace 必须一致。跨 Namespace 引用 ClusterRole 是允许的，但 binding 本身始终在一个 Namespace 内。

## 总结

Kubernetes RBAC 的设计非常简洁——四个对象、一套 verbs、组合即可。但简洁不等于容易用对，生产中最常见的安全事故都源于过度授权：给 CI/CD 用 `cluster-admin`、给开发者 Secrets 写权限、用通配符图省事。

核心原则只有一条：**给刚好够用的权限，不多给一个 verb**。结合 `kubectl auth can-i` 快速验证、审计日志持续监控、定期审计 ClusterRoleBinding，就能把 RBAC 从"能跑就行"提升到"安全可控"。

下次有人问"为什么我的 Pod 启动报 Forbidden"，不再是玄学——跑一遍 `kubectl auth can-i`，答案就在里面。