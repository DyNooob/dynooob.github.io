---
layout: post
title: "Kubernetes 网络策略 (Network Policies) 实战指南"
date: 2026-08-22
categories: 网络技术
tags: [kubernetes, network-policy, networking, security, container, cni, k8s, 云原生, 网络隔离]
---

## 为什么需要 Network Policies

Kubernetes 默认的"扁平网络"模型意味着：**集群内的任何 Pod 都能直接访问任何其他 Pod**。不管它们在哪台节点、哪个命名空间，只要知道对方的 IP 或 Service 名称，就能通信。

这在开发环境里很方便，但在生产环境就是灾难——一个被攻陷的 Pod 可以自由扫描整个集群、访问数据库、窃取敏感数据。没有网络隔离，你等于把整栋楼的门都敞开着，只靠门锁（应用层鉴权）防贼。

**Network Policies 就是这扇门的门禁系统**。它工作在 L3/L4 层，通过 CNI 插件（Calico、Cilium、Weave 等）实现，控制 Pod 级别的入站（Ingress）和出站（Egress）流量。

说人话：你定义一套规则，说"谁可以访问谁"，然后 CNI 插件把这些规则翻译成 iptables / eBPF / ipset 规则，下发到每台节点。

## 前提条件

不是所有集群都支持 Network Policies。你的 CNI 插件必须实现了 Kubernetes 的 `NetworkPolicy` API。常见的选择：

- **Calico**：最成熟，功能最全，支持全局网络策略
- **Cilium**：基于 eBPF，性能好，支持 L7 策略
- **Weave**：简单，但功能有限
- **Antrea**：VMware 出品，Open vSwitch 底层

如果你用的 CNI 是 Flannel（默认不启用网络策略），或者某种云厂商的托管 CNI 且没开启策略引擎，Network Policies 不会生效。

验证方法：

```bash
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
# 然后检查是否有 calico/cilium 相关 Pod 在运行
kubectl get pods -n kube-system | grep -E 'calico|cilium|antrea'
```

## 核心概念

### 选择器机制

Network Policy 的核心是选择器——它决定"这条规则对谁生效"。

```yaml
spec:
  podSelector:
    matchLabels:
      app: backend
```

这条规则只作用于带有 `app: backend` 标签的 Pod。不写 podSelector（留空）则匹配命名空间内所有 Pod。

### 策略类型

每条 Network Policy 可以定义三种规则：

- **podSelector**：目标 Pod（策略作用对象）
- **policyTypes**：声明这条策略包含 Ingress、Egress 还是两者都包含
- **ingress**：入站规则（谁可以访问目标 Pod）
- **egress**：出站规则（目标 Pod 可以访问谁）

### 匹配逻辑

**白名单模型**：一旦有一条 Network Policy 选中了某个 Pod，该 Pod 的流量就变成"默认拒绝，只允许显式白名单"。没有被任何策略选中的 Pod 则保持默认的"允许所有"。

**Ingress 规则**：多个 `from` 条目是 OR 关系——只要匹配任意一条就允许。

**Egress 规则**：多个 `to` 条目也是 OR 关系。

**规则内部的字段**：`from` 和 `to` 可以包含 `podSelector`、`namespaceSelector`、`ipBlock`，多个字段之间是 AND 关系。

## 场景一：默认拒绝所有流量

这是最基础也最重要的策略——先锁死，再开门。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

这个策略说：`production` 命名空间里的所有 Pod，禁止任何入站和出站流量。

应用之后，这个命名空间里的 Pod 将完全无法通信——包括访问 DNS、连接数据库、响应 HTTP 请求。除非你额外添加白名单规则。

**生产环境的推荐做法**：每个命名空间至少挂一条默认拒绝策略，然后根据业务需求添加白名单。这比"先开放再收紧"安全得多。

## 场景二：微服务三层隔离

假设典型的微服务架构：

```
前端 (app: frontend)  →  后端 (app: backend)  →  数据库 (app: db)
```

目标：
- 前端可以访问后端
- 后端可以访问数据库
- 数据库不接受来自前端的直接访问
- 外部流量只能到达前端

### 后端策略：只允许前端访问

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
          protocol: TCP
```

这条策略说：只有 `app: frontend` 标签的 Pod 可以访问后端 Pod 的 8080 端口。其他任何来源（包括其他命名空间的 Pod）都不行。

### 数据库策略：只允许后端访问

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - port: 5432
          protocol: TCP
```

数据库只对后端开放 5432 端口，前端永远无法直接连接数据库。

### 前端策略：允许外部流量

前端通常需要通过 Ingress 暴露，所以需要允许来自集群内任意来源的访问（Ingress Controller 的 Pod 可能在任何命名空间）：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
  ingress:
    - ports:
        - port: 80
          protocol: TCP
```

`from` 为空表示"允许所有来源"。这通常用于前端入口，因为 Ingress Controller 的 Pod 不在同一个命名空间，你不能用 `podSelector` 匹配它。

## 场景三：命名空间级别的隔离

如果你有多个租户或团队共享一个集群，可以用命名空间选择器来隔离。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-team-a
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: a
```

这个策略允许 `team: a` 标签的命名空间里的所有 Pod 访问 `team-a` 命名空间中的所有 Pod。其他命名空间被拒绝。

结合 `namespaceSelector` 和 `podSelector` 可以实现更细粒度的控制：

```yaml
    - from:
        - namespaceSelector:
            matchLabels:
              tier: monitoring
          podSelector:
            matchLabels:
              app: prometheus
```

这个规则说：只有 `tier: monitoring` 命名空间中 `app: prometheus` 标签的 Pod 可以访问。注意，这里的 `namespaceSelector` 和 `podSelector` 是 AND 关系——必须同时满足。

## 场景四：允许监控和日志采集

监控系统（Prometheus、Grafana、Loki）通常需要访问集群中的许多 Pod 来采集指标和日志。你可以在每个命名空间里添加一条策略，允许监控系统访问。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              app: monitoring
      ports:
        - port: 8080
          protocol: TCP
```

但这样每个命名空间都要加一条规则，很麻烦。更好的做法是使用 **全局网络策略**——如果你用的是 Calico，可以用 `GlobalNetworkPolicy`：

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-prometheus-scrape
spec:
  selector: "app == 'prometheus'"
  ingress:
    - action: Allow
      protocol: TCP
      destination:
        ports:
          - 8080
          - 9090
```

但这是 Calico 的扩展 API，不是标准 Kubernetes 的 Network Policy。如果只依赖标准 API，你需要在每个命名空间放一条策略。

## 场景五：限制出站流量

入站流量限制只是安全的一半。出站流量同样重要——如果攻陷了一个 Pod，攻击者首先要做的事就是外连（C2 回连、下载 payload、数据外泄）。

**只允许访问 DNS 和特定服务**：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    # 允许访问 DNS（CoreDNS 通常在 kube-system 命名空间）
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    # 允许访问数据库
    - to:
        - podSelector:
            matchLabels:
              app: db
      ports:
        - port: 5432
          protocol: TCP
```

注意，Kubernetes 1.22+ 的命名空间自动带有 `kubernetes.io/metadata.name` 标签，值就是命名空间名称。所以你可以用 `namespaceSelector: matchLabels: { kubernetes.io/metadata.name: kube-system }` 来精确匹配命名空间。

## 场景六：基于 IP 段的规则

当你需要访问集群外部的服务（比如托管数据库、外部 API），可以用 `ipBlock`：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
            except:
              - 10.0.0.5/32
      ports:
        - port: 3306
          protocol: TCP
```

这条规则允许后端 Pod 访问 `10.0.0.0/24` 网段（除了 `10.0.0.5`）的 3306 端口。

`ipBlock` 也支持 `from` 规则，用于限制外部来源的访问。

## 测试与验证

策略写完后，一定要验证。最常用的方法是用 `netshoot` 或 `busybox` 镜像启动调试 Pod，然后测试连通性。

### 1. 部署调试 Pod

```bash
kubectl run test-pod --image=nicolaka/netshoot -n production -- sleep 3600
kubectl exec -it test-pod -n production -- /bin/bash
```

### 2. 测试连通性

```bash
# 测试能否访问后端
curl -v http://backend-svc.production:8080

# 测试能否访问外网
curl -v https://www.google.com

# 测试 DNS 解析
nslookup kubernetes.default.svc.cluster.local
```

### 3. 查看策略生效状态

```bash
# 查看集群中的 Network Policies
kubectl get networkpolicies -n production

# 查看策略的详细 YAML
kubectl describe networkpolicy backend-policy -n production
```

### 4. 从被拒绝的 Pod 测试

最真实的测试是用一个"不应该被允许"的 Pod 来访问：

```bash
# 在另一个命名空间起一个 Pod
kubectl run attacker --image=nicolaka/netshoot -n default -- sleep 3600
kubectl exec -it attacker -n default -- curl -v http://backend-svc.production:8080
# 应该被拒绝（超时或连接被拒绝）
```

## 常见陷阱

### 陷阱 1：忘记添加 DNS 出站规则

这是最常见的错误。当你给 Pod 加了 Egress 限制后，Pod 无法解析 DNS——因为 CoreDNS 的请求被策略挡住了。结果是：Pod 能通过 IP 访问服务，但无法通过 Service 名称访问。

**解决方案**：在 Egress 策略中显式放行 DNS。

### 陷阱 2：Ingress Controller 跨命名空间问题

你的前端 Pod 只允许来自 `app: frontend` 的 Pod 访问？那 Ingress Controller 的流量会被拒绝——因为 Ingress Controller 运行在 `ingress-nginx` 命名空间，标签不匹配。

**解决方案**：前端策略的 `from` 留空（允许所有），或者用 `namespaceSelector` 匹配 Ingress Controller 所在的命名空间。

### 陷阱 3：策略顺序无关但规则内字段是 AND

新手经常搞混：`from` 下面的多个条目是 OR（匹配任一即允许），但一个条目里多个字段是 AND（必须同时匹配）。

```yaml
# OR：匹配 namespaceSelector 或 podSelector 任一
    - from:
        - namespaceSelector: ...
        - podSelector: ...

# 但这样是 AND：必须同时匹配 namespace 和 pod
    - from:
        - namespaceSelector: ...
          podSelector: ...
```

### 陷阱 4：默认拒绝策略影响 kube-proxy 健康检查

NodePort 类型的 Service 依赖 kube-proxy 的健康检查。如果你给节点上的 Pod 加了严格的入站规则，kube-proxy 的健康检查可能被拒绝，导致 Service 被认为不可用。

**解决方案**：确保 `--healthz-port` 对应的端口在策略中放行，或者使用 ClusterIP 替代 NodePort。

### 陷阱 5：策略数过多影响性能

每个 Network Policy 都会在 CNI 层面转换成一组 iptables 规则。集群中如果有上千条策略，iptables 规则的数量可能达到数万条，影响网络延迟和节点 CPU。

**解决方案**：
- 使用 Cilium（eBPF 替代 iptables）
- 合并多条策略，减少总数
- 使用命名空间级别的策略，而不是 Pod 级别的细粒度策略

## 调试技巧

当策略不生效时，按以下步骤排查：

### 1. 确认 CNI 支持

```bash
kubectl get pods -n kube-system | grep -E 'calico|cili|antrea'
```

如果没有输出，你的 CNI 可能不支持 Network Policies。

### 2. 查看 CNI 的日志

```bash
# Calico
kubectl logs -n kube-system -l k8s-app=calico-node --tail=100

# Cilium
kubectl exec -n kube-system -l app=cilium -- cilium endpoint list
```

### 3. 抓包确认

在节点上用 tcpdump 确认流量是否被丢弃：

```bash
# 在节点上抓包，查看是否有 SYN 包被 Drop
tcpdump -i any port 8080 -nn
```

### 4. 检查 iptables 规则

```bash
# 查看 Calico 生成的 iptables 链
iptables -L -n -t filter | grep -E 'cali|KUBE'
```

## 总结

Network Policies 是 Kubernetes 生产环境中不可或缺的安全组件。它不复杂——核心就三个概念（选择器、入站、出站），但能为你挡住大部分的内部威胁。

**核心要点**：

1. **先拒绝，再放行**——每个命名空间挂一条默认拒绝策略
2. **出站规则同样重要**——防止数据外泄和 C2 回连
3. **用完测试 Pod 验证**——不要相信理论上的连通性
4. **注意 DNS**——出站策略别忘了放行 CoreDNS
5. **选择支持策略的 CNI**——Calico 和 Cilium 是最推荐的选择

如果你的集群还没有任何 Network Policy，今天就从默认拒绝策略开始吧。五分钟后就能搞定，收益却远超你的投入。