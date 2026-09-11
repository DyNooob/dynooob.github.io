---
layout: post
title: "Kubernetes Gateway API 实战指南：从 Ingress 到新一代入口网关"
date: 2026-09-11 09:00:00 +0800
categories: [网络技术]
tags: [Kubernetes, Gateway API, Ingress, 入口网关, 云原生, 网络技术, k8s, 流量管理, 服务网格]
---

## 为什么需要 Gateway API？

Kubernetes Ingress 从 1.1 版本就存在了，十几年下来它的局限性越来越明显：

- **只能处理七层流量**（HTTP/HTTPS），四层 TCP/UDP 路由完全依赖厂商注解
- **缺乏租户隔离**——所有路由规则堆在一个资源里，团队间协作困难
- **厂商锁定**——每个 Ingress Controller 用不同的 annotation 实现差异化功能，迁移成本高
- **表达力不足**——无法描述 header-based routing、流量拆分、金丝雀发布等高级场景

Gateway API 是 Kubernetes 网络 SIG 在 2020 年启动的项目，目标就是解决这些问题。它不是 Ingress 的升级版，而是一个**全新的、表达能力更强的、面向角色的 API 框架**。2026 年 7 月，Gateway API v1.2 正式发布，GA 状态，生产可用。

这篇文章从零开始，带你理解 Gateway API 的核心概念，然后在真实集群上用 Contour 部署一套功能完整的入口网关。

## 一、核心概念：三层模型

Gateway API 最大的设计亮点是**角色分离**。传统 Ingress 把所有决策权交给集群管理员一个人，Gateway API 把责任拆成三层：

```
+-----------------------+
|   GatewayClass        |  ← 基础设施提供商定义（Ingress 厂商）
+-----------------------+
          |
+-----------------------+
|   Gateway             |  ← 集群管理员定义（平台团队）
+-----------------------+
          |
+-----------------------+     +-----------------------+
|   HTTPRoute           |     |   TCPRoute / TLSRoute |  ← 应用开发者定义
|   (七层 HTTP 路由)     |     |   (四层 / TLS 路由)     |
+-----------------------+     +-----------------------+
```

### 1.1 GatewayClass

类似 StorageClass 之于 PV——GatewayClass 定义了底层网关实现（用什么 Controller、有什么能力）。通常由基础设施团队或云厂商预置。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

### 1.2 Gateway

平台团队实例化一个 GatewayClass，绑定到具体的网络入口。你可以把它理解为"把某个负载均衡器/代理进程挂到集群上"。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "*.example.com"
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.example.com"
    tls:
      mode: Terminate
      certificateRefs:
      - name: example-tls
```

注意这里的 hostname 字段——Gateway 层面就可以做域名级别的准入控制，比 Ingress 的全局路由灵活得多。

### 1.3 HTTPRoute / TCPRoute / TLSRoute

应用开发者创建路由规则，挂接到某个 Gateway 上。一个 Gateway 可以挂接多个 Route，一个 Route 也可以被多个 Gateway 引用——这是 Ingress 做不到的。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-route
spec:
  parentRefs:
  - name: prod-gateway
    namespace: infra
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: api-v1
      port: 8080
    - matches:
    - path:
        type: PathPrefix
        value: /v2
    backendRefs:
    - name: api-v2
      port: 8080
```

这里 parentRefs 可以跨 namespace 引用——应用 A 的 Route 可以挂在平台团队管理的 Gateway 上，而不需要集群管理员权限。

## 二、实战：用 Contour 部署 Gateway API

理论说够了，上实战。我选择 Contour 作为 Gateway 实现，原因很简单：它原生支持 Gateway API，部署简单，性能可靠。Envoy Gateway 和 Istio 也是好选择，但 Contour 的 CRD 更简洁，适合入门。

### 2.1 环境准备

需要一个 Kubernetes 集群（v1.26+ 支持 Gateway API CRD）。Minikube 或 kind 都可以：

```bash
kind create cluster --name gateway-demo
```

### 2.2 安装 Gateway API CRD

```bash
# 安装标准 CRD
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

验证安装：

```bash
kubectl get crd | grep gateway
# 应该看到: gatewayclasses.gateway.networking.k8s.io
#           gateways.gateway.networking.k8s.io
#           httproutes.gateway.networking.k8s.io
#           (还有参考授权和 GRPCRoute 等)
```

### 2.3 部署 Contour

```bash
# Contour 附带了 GatewayClass 和 Gateway Controller
kubectl apply -f https://projectcontour.io/quickstart/contour-gateway.yaml
```

等 Pod 启动：

```bash
kubectl -n projectcontour get pods
# NAME                       READY   STATUS
# contour-gateway-proxy-xxx   1/1     Running
# contour-gateway-xxx         1/1     Running
```

查看 Contour 自动创建的 GatewayClass：

```bash
kubectl get gatewayclass
# NAME          CONTROLLER                          ACCEPTED   AGE
# contour       io.projectcontour.gateway/v1beta1   True       30s
```

### 2.4 创建示例应用

部署两个版本的 API 作为演示后端：

```yaml
# demo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
      version: v1
  template:
    metadata:
      labels:
        app: api
        version: v1
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:latest
        args: ["-text=Hello from API v1"]
        ports:
        - containerPort: 5678
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
      version: v2
  template:
    metadata:
      labels:
        app: api
        version: v2
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo:latest
        args: ["-text=Hello from API v2"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-v1
spec:
  selector:
    app: api
    version: v1
  ports:
  - port: 8080
    targetPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: api-v2
spec:
  selector:
    app: api
    version: v2
  ports:
  - port: 8080
    targetPort: 5678
```

应用并验证：

```bash
kubectl apply -f demo-app.yaml
kubectl get svc
# NAME     TYPE        CLUSTER-IP     PORT(S)
# api-v1   ClusterIP   10.96.100.1    8080/TCP
# api-v2   ClusterIP   10.96.100.2    8080/TCP
```

### 2.5 创建 Gateway 和 HTTPRoute

这是核心配置。注意 platform 团队和应用团队的不同视角：

```yaml
# gateway.yaml — 平台团队创建
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: demo-gateway
spec:
  gatewayClassName: contour
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All    # 允许所有 namespace 的路由挂接
---
# httproute.yaml — 应用团队创建
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: demo-route
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "demo.local"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: api-v1
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /v2
    backendRefs:
    - name: api-v2
      port: 8080
```

```bash
kubectl apply -f gateway.yaml
kubectl apply -f httproute.yaml
```

### 2.6 验证流量路由

获取 Gateway 的监听地址：

```bash
kubectl get gateway demo-gateway -o jsonpath='{.status.addresses[0].value}'
# localhost 或 LoadBalancer IP
```

访问测试：

```bash
# 测试 /v1 路由
curl -H "Host: demo.local" http://localhost/v1
# Hello from API v1

# 测试 /v2 路由
curl -H "Host: demo.local" http://localhost/v2
# Hello from API v2

# 无匹配路径 -> 404
curl -H "Host: demo.local" http://localhost/admin
# 404 Not Found
```

至此，一个基础的 Gateway API 入口网关已经跑通了。

## 三、高级场景：流量拆分与灰度发布

Gateway API 原生支持流量权重拆分（traffic splitting），不需要额外注解。这是 Ingress 无论如何都做不到的能力。

### 3.1 权重路由

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "shop.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: shop-stable
      port: 8080
      weight: 90
    - name: shop-canary
      port: 8080
      weight: 10
```

90% 流量去 stable 版本，10% 去 canary 版本。不需要 sidecar、不需要 Service Mesh，Gateway 原生支持。

### 3.2 Header-Based 路由

按请求特征分流到不同后端：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ab-testing
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - headers:
      - name: X-User-Group
        value: beta
    backendRefs:
    - name: app-next
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: app-stable
      port: 8080
```

这在 A/B 测试场景下非常实用——beta 用户看到新版，其余用户不受影响。

### 3.3 请求镜像

把流量同时复制一份到测试后端，不影响线上响应：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mirror-route
spec:
  parentRefs:
  - name: demo-gateway
  hostnames:
  - "api.example.com"
  rules:
  - backendRefs:
    - name: api-prod
      port: 8080
    filters:
    - type: RequestMirror
      requestMirror:
        backendRef:
          name: api-test
          port: 8080
```

## 四、多团队协作模式

Gateway API 最被低估的能力是多租户（multi-tenancy）支持。在一个 Gateway 上，多个团队可以安全地共享入口：

```
Namespaces:
  infra/                   平台团队
    Gateway: prod-gateway
      listeners:
        - "*.team-a.example.com"
        - "*.team-b.example.com"

  team-a/                  应用团队 A
    HTTPRoute: team-a-route
      parentRefs: infra/prod-gateway
      hostnames: ["svc.team-a.example.com"]

  team-b/                  应用团队 B
    HTTPRoute: team-b-route
      parentRefs: infra/prod-gateway
      hostnames: ["svc.team-b.example.com"]
```

每个团队在自己的 namespace 内管理路由规则，不需要接触集群级别的 Gateway 资源。平台团队通过 `allowedRoutes` 控制谁能挂接，通过 `hostname` 限制域名范围。

```yaml
# 平台团队限制：只允许 team-a 挂接 team-a.example.com 域名
listeners:
- name: team-a
  protocol: HTTP
  port: 80
  hostname: "*.team-a.example.com"
  allowedRoutes:
    namespaces:
      from: Selector
      selector:
        matchLabels:
          team: a
```

## 五、Gateway API vs Ingress vs Service Mesh

| 能力 | Ingress | Gateway API | Service Mesh |
|------|---------|-------------|--------------|
| 七层路由 | 基础 | 丰富（header/weight/mirror） | 丰富 |
| 四层路由 | 不支持 | 原生支持（TCPRoute） | 支持 |
| 多团队隔离 | 不支持 | 原生支持 | 需额外配置 |
| 流量拆分 | 注解实现 | 原生字段 | 原生支持 |
| 金丝雀发布 | 注解实现 | 原生字段 | 原生支持 |
| 东西向流量 | 不支持（只在入口） | 仅入口 | 网格内全流量 |
| 学习曲线 | 低 | 中 | 高 |
| 生产成熟度 | GA（十几年） | GA（v1.2，2026） | 视具体实现 |

Gateway API 填补了 Ingress 和 Service Mesh 之间的巨大空白——它比 Ingress 强大得多，又比 Service Mesh 轻量得多。

## 六、生产部署注意事项

### 6.1 TLS 证书管理

Gateway 的证书引用支持 Secret 和 Provider-agnostic 的 ReferenceGrant：

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-cert-ref
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: Gateway
  to:
  - group: ""
    kind: Secret
```

跨 namespace 引用证书时必须配合 ReferenceGrant，这是安全设计——防止任意 Gateway 读取其他 namespace 的证书。

### 6.2 监控与可观测性

Contour（基于 Envoy）原生暴露 Prometheus 指标：

```yaml
# Contour 已经默认开启 metrics
# 查看指标端点
kubectl -n projectcontour port-forward svc/contour-envoy 9002:9002
curl localhost:9002/metrics | grep envoy_http_downstream_rq
```

你应该关注的指标：
- `envoy_cluster_upstream_rq_xx` — 按响应码分类的请求数
- `envoy_cluster_upstream_rq_active` — 当前活跃请求数
- `envoy_listener_downstream_cx_active` — 活跃连接数
- `envoy_http_downstream_rq_time` — 请求延迟分布

### 6.3 性能调优

Envoy（Contour 底层代理）的几个关键参数：

```yaml
# Contour 配置：调整连接池和超时
apiVersion: projectcontour.io/v1alpha1
kind: ContourConfiguration
metadata:
  name: contour-config
spec:
  envoy:
    listener:
      connectionBalancer: precise    # 精确连接均衡
    timeouts:
      requestTimeout: 30s            # 请求超时
      idleTimeout: 120s              # 空闲连接超时
    cluster:
      maxRequestsPerConnection: 100  # 每个连接最大请求数
```

## 七、小结

Gateway API 是 Kubernetes 网络生态中最重要的演进之一。它不是 Ingress v2，而是一个全新的框架——面向角色、原生支持高级路由、天然多团队隔离、无厂商锁定。

如果你还在用 Ingress + 一堆 annotation 实现金丝雀发布和流量拆分，是时候看看 Gateway API 了。2026 年的今天，它的核心特性已经 GA，主流实现（Contour、Envoy Gateway、Istio、NGINX Gateway Fabric）全部可用。

下一步要学的：
1. **GRPCRoute** — 原生 gRPC 路由，比 HTTPRoute 更精准
2. **BackendTLSPolicy** — 后端 TLS 直连，端到端加密
3. **Service Mesh 集成** — Gateway API 已被 Istio 和 Linkerd 接受为入口 API

### 参考资源

- [Gateway API 官方文档](https://gateway-api.sigs.k8s.io/)
- [Contour Gateway API 指南](https://projectcontour.io/guides/gateway-api/)
- [Envoy Gateway](https://gateway.envoyproxy.io/)
- [Gateway API v1.2 Release Notes](https://github.com/kubernetes-sigs/gateway-api/releases/tag/v1.2.0)