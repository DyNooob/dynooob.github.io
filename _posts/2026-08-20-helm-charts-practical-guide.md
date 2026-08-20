---
layout: post
title: "Helm Charts 实战：从零到生产级部署"
date: 2026-08-20
categories: [开发]
tags: [Helm, Kubernetes, DevOps, 容器化, 部署, 云原生, Charts, K8s]
---

## 为什么需要 Helm

Kubernetes 原生资源（Pod、Service、Deployment）用 YAML 定义，但一个微服务动辄需要 5-8 个 YAML 文件。手动管理这些文件会面临三个问题：

- **重复**：开发、预发、生产三个环境，每个环境都要复制粘贴，改几个参数
- **版本管理**：没有统一的版本化部署机制，回滚靠手动 apply 旧文件
- **模板缺失**：YAML 不支持变量、循环、条件判断，配置只能靠 sed 或 envsubst

Helm 解决了这些问题：它是 Kubernetes 的包管理器，把一组 K8s 资源打包成 **Chart**，通过模板引擎实现参数化，通过 Chart 仓库实现分发，通过 Release 管理实现版本化部署。

一个 Helm Chart 的典型部署流程只需要一条命令：

```bash
helm install my-app ./my-chart --values prod-values.yaml
```

本文从零开始，带你写一个生产级的 Helm Chart。

## Chart 结构解剖

创建一个 Helm Chart 最快的方式是用 `helm create`：

```bash
helm create demo-app
```

生成的结构如下：

```
demo-app/
├── Chart.yaml          # 元数据：名称、版本、依赖
├── values.yaml         # 默认配置值
├── charts/             # 依赖的子 Chart（helm dependency 下载到这里）
├── templates/          # Go 模板文件，渲染为 K8s YAML
│   ├── NOTES.txt       # helm install 后的提示信息
│   ├── _helpers.tpl    # 可复用的模板函数
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   └── tests/          # Chart 测试
│       └── test-connection.yaml
└── .helmignore         # 打包时忽略的文件
```

### Chart.yaml 关键字段

```yaml
apiVersion: v2
name: demo-app
description: 一个示例 Web 应用
type: application
version: 0.1.0
appVersion: "1.16.0"
kubeVersion: ">=1.21.0"
keywords:
  - web
  - demo
maintainers:
  - name: your-name
    email: your@email.com
dependencies:
  - name: redis
    version: "~17.0.0"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

- `version`：Chart 自身的版本，每次升级 Chart 时递增
- `appVersion`：应用版本，与应用镜像 tag 对应
- `kubeVersion`：限制兼容的 K8s 版本
- `dependencies`：声明依赖的外部 Chart，`helm dependency update` 自动下载

## 编写第一个模板

模板文件放在 `templates/` 目录下，用 Go 模板语法结合 Sprig 函数库实现动态渲染。

### 基础 Deployment

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "demo-app.fullname" . }}
  labels:
    {{- include "demo-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "demo-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "demo-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          env:
            {{- toYaml .Values.env | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

### 对应的 values.yaml

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: ""
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

env:
  - name: APP_ENV
    value: "production"
  - name: LOG_LEVEL
    value: "info"

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

### _helpers.tpl 中的复用函数

`_helpers.tpl` 中的模板函数不会被直接渲染为 K8s 资源，而是被其他模板引用。`helm create` 自动生成了一套标准 helper：

```yaml
{{- define "demo-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "demo-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{- define "demo-app.labels" -}}
helm.sh/chart: {{ include "demo-app.name" . }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "demo-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

这些 helper 确保所有资源使用一致的标签命名规范，符合 K8s 推荐标签。

## 模板进阶技法

### 条件判断与默认值

```yaml
# 只有在 ingress.enabled 时才创建 Ingress 资源
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "demo-app.fullname" . }}
  annotations:
    {{- toYaml .Values.ingress.annotations | nindent 4 }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: Prefix
            backend:
              service:
                name: {{ include "demo-app.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

### 循环遍历列表

```yaml
# values.yaml 中定义
extraVolumes:
  - name: config
    mountPath: /etc/config
    configMap: app-config
  - name: secrets
    mountPath: /etc/secrets
    secret: app-secrets

# 模板中渲染
volumes:
{{- range .Values.extraVolumes }}
  - name: {{ .name }}
    {{- if .configMap }}
    configMap:
      name: {{ .configMap }}
    {{- else if .secret }}
    secret:
      secretName: {{ .secret }}
    {{- end }}
{{- end }}
```

### 命名模板传参

Helm 的 `include` 默认只能传入顶层的 `.`。要让模板函数接受额外参数，用字典传参：

```yaml
{{- define "demo-app.probe" -}}
{{- $probe := .probe }}
httpGet:
  path: {{ $probe.path }}
  port: {{ $probe.port }}
initialDelaySeconds: {{ $probe.initialDelaySeconds | default 5 }}
periodSeconds: {{ $probe.periodSeconds | default 10 }}
{{- end }}

# 调用时传入字典
livenessProbe:
  {{- include "demo-app.probe" (dict "probe" .Values.livenessProbe) | nindent 2 }}
readinessProbe:
  {{- include "demo-app.probe" (dict "probe" .Values.readinessProbe) | nindent 2 }}
```

## 依赖管理与子 Chart

一个 Web 应用通常依赖 Redis 或 PostgreSQL。用 `dependencies` 声明：

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "~12.0.0"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
    alias: db
```

然后 `helm dependency update` 下载子 Chart 到 `charts/` 目录。子 Chart 的 values 可以通过 `values.yaml` 的同名 key 覆盖：

```yaml
# values.yaml
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    password: changeme!
  primary:
    persistence:
      size: 8Gi
```

子 Chart 的模板变量通过 `.Values.postgresql` 访问。注意子 Chart 内部的 `values.yaml` 有默认值，父 Chart 的 values 会合并覆盖。

## Hooks：在部署的特定时刻执行任务

Helm Hooks 让你在 Release 生命周期的特定点执行 Pod Job：

| Hook | 触发时机 |
|------|---------|
| pre-install | 安装前 |
| post-install | 安装后 |
| pre-upgrade | 升级前 |
| post-upgrade | 升级后 |
| pre-delete | 删除前 |
| post-delete | 删除后 |
| test | `helm test` 时 |

数据库迁移是典型场景：

```yaml
# templates/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "demo-app.fullname" . }}-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          command: ["npm", "run", "migrate"]
          env:
            {{- toYaml .Values.env | nindent 12 }}
```

- `helm.sh/hook-weight`：控制多个 hook 的执行顺序（数值越小越先执行）
- `helm.sh/hook-delete-policy`：控制 hook 资源是否保留

## 多环境配置管理

### 按环境拆分 values 文件

```
config/
├── base.yaml          # 通用配置
├── development.yaml
├── staging.yaml
└── production.yaml
```

部署时指定多个 values 文件，后面覆盖前面：

```bash
helm install my-app ./demo-app \
  -f config/base.yaml \
  -f config/production.yaml \
  --set image.tag=v1.2.3
```

### 全局值与局部值

```yaml
# values.yaml
global:
  environment: production
  imageRegistry: my-registry.cn

# 在模板中通过 .Values.global.environment 访问
# 子 Chart 也能访问 .Values.global（全局值穿透到所有子 Chart）
```

## Chart 测试

`helm test` 运行 `templates/tests/` 目录下的 Pod，验证部署是否正常：

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "demo-app.fullname" . }}-test-connection"
  labels:
    {{- include "demo-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "demo-app.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

运行测试：

```bash
helm test my-app
```

## 生产级实践清单

### 1. 版本管理

- Chart 版本采用 SemVer：`0.1.0` → `0.2.0` → `1.0.0`
- `appVersion` 与应用镜像 tag 保持一致
- 每次修改模板逻辑都递增 Chart 版本

### 2. 安全加固

```yaml
# 安全上下文
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
  seccompProfile:
    type: RuntimeDefault

# Pod 安全上下文
podSecurityContext:
  runAsNonRoot: true
  fsGroup: 2000
```

### 3. 资源限制

Always set requests and limits. 未设置资源限制的 Pod 可能被 OOMKill 或影响集群稳定性。

```yaml
resources:
  limits:
    cpu: 1
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi
```

### 4. CI/CD 集成

以下是一个 GitHub Actions 的 Helm 发布流水线：

```yaml
# .github/workflows/release-chart.yaml
name: Release Helm Chart

on:
  push:
    branches: [main]
    paths: ['charts/**', 'Chart.yaml']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.16.0

      - name: Lint Chart
        run: helm lint ./demo-app

      - name: Package Chart
        run: helm package ./demo-app -d ./release

      - name: Push to OCI Registry
        run: |
          helm push ./release/demo-app-*.tgz oci://ghcr.io/${{ github.repository }}/charts
```

### 5. 使用 OCI 仓库分发

Helm v3.8+ 原生支持 OCI 仓库，把 Chart 像容器镜像一样 push 到 Harbor、GitHub Container Registry 或 AWS ECR：

```bash
# 登录
helm registry login ghcr.io -u $USER

# 推送
helm package ./demo-app
helm push demo-app-0.1.0.tgz oci://ghcr.io/my-org/charts

# 安装
helm install my-app oci://ghcr.io/my-org/charts/demo-app --version 0.1.0
```

OCI 仓库比传统的 ChartMuseum 更方便，因为复用已有的镜像仓库权限和扫描能力。

## 常见坑

### 模板缩进错误

`nindent` 和 `indent` 是 Helm 模板中最容易出错的地方。记住：

- `nindent N`：先换行，再缩进 N 个空格
- `indent N`：在当前行缩进 N 个空格

```yaml
# 正确
env:
  {{- toYaml .Values.env | nindent 2 }}

# 错误（会破坏 YAML 缩进）
env:
  {{ toYaml .Values.env }}
```

### 值覆盖顺序

Helm 的值合并顺序（后面的覆盖前面的）：

1. `values.yaml`（Chart 内置）
2. `-f` 指定的 values 文件（按顺序）
3. `--set` 参数
4. `--set-json` 参数

### 模板名称冲突

如果你的 Chart 引用了子 Chart 的同名模板，子 Chart 的模板会覆盖父 Chart 的。解决方案：所有模板用 Chart 名称前缀，如 `{{ define "myapp.fullname" }}`。

## 总结

Helm 将 Kubernetes 部署从分散的 YAML 文件升级为可参数化、可版本化、可复用的 Chart。本文涉及的核心实践：

- 用 `helm create` 生成标准 Chart 结构，在此基础上修改
- 通过 `_helpers.tpl` 集中管理命名逻辑，避免模板重复
- 用 `values.yaml` 分层管理多环境配置
- 用 Hooks 管理数据库迁移等生命周期任务
- 用 OCI 仓库分发 Chart，与镜像仓库共用权限

下一步可以深入 Helmfile 或 ArgoCD 实现 GitOps 工作流，或者用 Helm Plugin 扩展 Chart 功能。

如果你正在把现有的 K8s YAML 迁移到 Helm，建议从最常用的服务开始，先写一个简单的 Deployment + Service 模板，逐步迭代完善。