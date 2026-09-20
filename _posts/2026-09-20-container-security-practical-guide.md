---
layout: post
title: "容器安全实战指南：从镜像扫描到运行时防护"
date: 2026-09-20 09:00:00 +0800
categories: [安全开发, 容器化]
tags: [docker, container-security, trivy, seccomp, apparmor, runtime-security, devsecops, pod-security, falco, least-privilege]
---

## 容器安全不只是"别跑 root"

容器不是轻量级虚拟机。它们共享宿主机内核，这意味着一个容器逃逸就能威胁整个节点。很多团队关注编排、监控、镜像大小，却把安全留到出事才补救。

这篇文章从**构建时、部署时、运行时**三个维度，给出可以直接落地的容器安全方案。每条建议都配具体命令或配置，不是空谈原则。

---

## 一、构建时安全：从源头控制风险

### 1.1 镜像扫描：别把漏洞装进镜像

镜像扫描是容器安全的第一道防线。推荐用 **Trivy**，开源、快、覆盖全。

安装和扫描一行搞定：

```bash
# 安装 Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# 扫描本地镜像
trivy image nginx:1.25

# 扫描 Dockerfile (IaC 扫描)
trivy fs --severity HIGH,CRITICAL .
```

输出示例（精简）：

```
nginx:1.25 (debian 12.6)
========================
Total: 23 (UNKNOWN: 0, LOW: 12, MEDIUM: 8, HIGH: 2, CRITICAL: 1)

┌──────────────┬────────────────┬──────────┬───────────────────┐
│   Library    │ Vulnerability  │ Severity │     Status        │
├──────────────┼────────────────┼──────────┼───────────────────┤
│ libssl3      │ CVE-2026-1234  │ CRITICAL │ fixed in 3.0.15   │
│ zlib         │ CVE-2026-2345  │ HIGH     │ fixed in 1.3.1    │
└──────────────┴────────────────┴──────────┴───────────────────┘
```

**在 CI/CD 中阻断策略**：不要只扫描不行动。设置严重性门槛，高危及以上直接阻断构建。

```yaml
# .github/workflows/docker-scan.yml
name: Container Security Scan
on: [push]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t app:${{ github.sha }} .
      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: app:${{ github.sha }}
          format: table
          exit-code: 1
          severity: HIGH,CRITICAL
```

`exit-code: 1` 是关键——扫描到高危漏洞就让 pipeline 失败，阻止漏洞镜像进入生产。

### 1.2 Dockerfile 安全最佳实践

**原则一：绝不跑 root。**

```dockerfile
# ❌ 危险
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
# 默认以 root 运行，容器被攻破 = 宿主机 root 等价

# ✅ 安全
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-slim
RUN groupadd -r appuser && useradd -r -g appuser -d /app appuser
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY . .
USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```

**原则二：使用 Distroless 或 Slim 基础镜像。**

```dockerfile
# 从 node:20 (约 1GB) 降到 gcr.io/distroless/nodejs20 (约 120MB)
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["server.js"]
```

Distroless 镜像没有 shell、没有包管理器、没有多余工具——攻击者拿到 shell 也没地方可去。

**原则三：不要用 ADD，用 COPY。**

```dockerfile
# ❌ ADD 会自动解压 tar 文件、解析 URL，行为不可预测
ADD app.tar.gz /app/

# ✅ COPY 只做文件复制，行为明确
COPY app.tar.gz /app/
```

**原则四：固定基础镜像版本。**

```dockerfile
# ❌ 每次构建可能拉到不同版本
FROM node:20

# ✅ 固定 sha256，确保构建可复现且无意外更新
FROM node:20@sha256:a1b2c3d4e5f6...
```

---

## 二、部署时安全：配置即控制

### 2.1 使用只读根文件系统

容器默认可写。但如果应用只在特定目录写数据，把根文件系统设为只读是极大的安全收益——攻击者无法写二进制或脚本到容器里。

Kubernetes 配置：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop: ["ALL"]
        # 只添加真正需要的 capability
        add: ["NET_BIND_SERVICE"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

关键配置解释：

| 配置 | 作用 |
|------|------|
| `readOnlyRootFilesystem: true` | 禁止写入容器文件系统 |
| `runAsNonRoot: true` | 拒绝以 uid 0 运行 |
| `runAsUser: 1000` | 显式指定普通用户 |
| `capabilities.drop: ["ALL"]` | 扔掉所有 Linux Capability，只加必要的一个 |

### 2.2 Seccomp 配置文件

Seccomp 限制容器能调用的系统调用。一个典型的 Web 应用只需要 50-80 个 syscall，Linux 有超过 400 个。

Docker 默认有 seccomp 配置，但宽松。建议用更严格的 profile：

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "accept4", "bind", "brk", "clock_gettime",
        "close", "connect", "dup", "epoll_create1",
        "epoll_ctl", "epoll_wait", "exit_group",
        "fchmod", "fchown", "fcntl", "fdatasync",
        "fstat", "fsync", "ftruncate", "futex",
        "getdents64", "getpeername", "getpid",
        "getsockname", "getsockopt", "getrandom",
        "listen", "lseek", "mkdir", "mmap",
        "mprotect", "munmap", "nanosleep", "newfstatat",
        "openat", "pipe2", "poll", "pread64",
        "pwrite64", "read", "readlinkat", "recvfrom",
        "recvmsg", "rename", "rt_sigaction",
        "rt_sigprocmask", "sendmmsg", "sendmsg",
        "sendto", "set_robust_list", "set_tid_address",
        "setsockopt", "shutdown", "sigaltstack",
        "socket", "statx", "symlinkat", "sync_file_range",
        "unlinkat", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

应用 seccomp 配置：

```yaml
# Kubernetes
securityContext:
  seccompProfile:
    type: Localhost
    localhostProfile: "profiles/strict-webapp.json"
```

```bash
# Docker
docker run --security-opt seccomp=strict-webapp.json myapp:1.0
```

### 2.3 AppArmor 配置文件

AppArmor 进一步限制文件访问、网络访问和进程能力。

一个 Web 应用的 AppArmor profile 示例：

```
#include <tunables/global>

profile docker-webapp flags=(attach_disconnected) {
  #include <abstractions/base>

  # 只读访问应用文件
  /app/** r,

  # 日志和临时文件
  /var/log/app/** w,
  /tmp/** rw,

  # 禁止访问的系统文件
  /etc/shadow r,
  /etc/gshadow r,
  /root/** r,

  # 网络
  network tcp,
  network inet stream,

  # 拒绝执行其他程序
  /bin/** ix,
  /usr/bin/** ix,
}
```

加载 profile：

```bash
# 加载到内核
apparmor_parser -r -W /etc/apparmor.d/docker-webapp

# 运行容器时附加
docker run --security-opt apparmor=docker-webapp myapp:1.0
```

---

## 三、运行时安全：持续监控与响应

### 3.1 Pod Security Standards (PSS)

Kubernetes v1.23+ 内置了 Pod Security Admission，分三个级别：

| 级别 | 说明 | 适用场景 |
|------|------|----------|
| **Privileged** | 无限制 | 基础设施组件（ingress controller、CNI 插件） |
| **Baseline** | 最低限度的限制 | 标准应用，防止已知提权 |
| **Restricted** | 严格遵循 Pod 安全最佳实践 | 高安全要求的工作负载 |

在命名空间级别应用：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

`enforce` 会直接拒绝违规 Pod，`audit` 记录日志，`warn` 给用户警告但不拒绝。上线新策略时推荐先 audit/warn 再 enforce。

### 3.2 Falco：运行时异常检测

**Falco** 是 CNCF 孵化的运行时安全工具，监控系统调用并基于规则告警。

安装：

```bash
# 在 Linux 节点上
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
  gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] \
  https://download.falco.org/packages/deb stable main" | \
  tee -a /etc/apt/sources.list.d/falcosecurity.list

apt-get update && apt-get install -y falco
```

核心规则示例（Falco 默认规则已很全面，但建议自定义）：

```yaml
# /etc/falco/rules.d/custom.yaml
- rule: Container Drift Detected (New Binary Written)
  desc: 检测容器运行时是否有新可执行文件写入
  condition: >
    evt_type=open or evt_type=openat
    and evt.dir=<
    and container.id != host
    and fd.typechar='f'
    and fd.filename endswith '.elf'
    and evt.arg.flags contains O_CREAT
  output: >
    New ELF binary written in container
    (user=%user.name container=%container.info file=%fd.name)
  priority: CRITICAL
  tags: [container, drift, malware]

- rule: Reverse Shell in Container
  desc: 检测容器内反向 shell 连接
  condition: >
    evt_type=connect
    and container.id != host
    and proc.name in (bash, sh, python, python3, nc, ncat, perl, ruby, php)
    and fd.sip not in (trusted_ips)
  output: >
    Possible reverse shell from container
    (proc=%proc.name cmdline=%proc.cmdline container=%container.info fd=%fd.name)
  priority: CRITICAL
  tags: [container, shell, network]

- rule: Unexpected Outbound Connection
  desc: 容器发起非预期出站连接
  condition: >
    evt_type=connect
    and container.id != host
    and fd.sport > 1024
    and not proc.name in (allowed_egress_procs)
    and not fd.sip in (allowed_egress_ips)
  output: >
    Unexpected outbound connection
    (proc=%proc.name container=%container.info dest=%fd.sip:%fd.sport)
  priority: WARNING
  tags: [container, network, exfiltration]
```

Falco 可以配置多种输出：

```yaml
# /etc/falco/falco.yaml
json_output: true
json_include_output_property: true

program_output:
  enabled: true
  keep_alive: true
  program: "curl -X POST -H 'Content-Type: application/json' \
    -d @- http://alertmanager:9093/api/v1/alerts"

http_output:
  enabled: true
  url: http://webhook:8080/falco
```

### 3.3 拒绝特权容器

特权容器几乎等于宿主机 root。坚决不允许：

```yaml
# OPA/Gatekeeper 策略或 Kyverno ClusterPolicy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-privileged-containers
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-privileged
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Privileged containers are not allowed."
      pattern:
        spec:
          containers:
          - securityContext:
              privileged: false
```

---

## 四、安全清单：从零加固一个容器

把这个清单贴到你的 README 里：

### 构建阶段
- [ ] 基础镜像版本固定到 sha256
- [ ] 使用 Distroless 或 Slim 镜像
- [ ] 多阶段构建，最终镜像不包含构建工具
- [ ] 以非 root 用户运行
- [ ] CI/CD 中集成镜像扫描（Trivy），阻断高危漏洞
- [ ] 定期扫描基础镜像（设置 cron 任务）

### 部署阶段
- [ ] `readOnlyRootFilesystem: true`
- [ ] `runAsNonRoot: true` + `runAsUser: 1000+`
- [ ] `capabilities.drop: ["ALL"]`，只添加必要项
- [ ] 应用自定义 seccomp profile（禁止非必要 syscall）
- [ ] 命名空间级别 PSS: Restricted
- [ ] 使用 `.dockerignore` 排除敏感文件

### 运行时
- [ ] 部署 Falco agent，开启运行时监控
- [ ] 配置自定义规则（文件漂移、反弹 shell、异常出站）
- [ ] 拒绝特权容器策略（Kyverno/OPA）
- [ ] 启用审计日志
- [ ] 限制出站网络（NetworkPolicy）

---

## 五、常见误区

**误区一："容器内部有 root 也没事，反正容器隔离了。"**

事实：容器和宿主机共享内核。如果容器以 root 运行，且存在内核漏洞（如 CVE-2022-0492 — cgroup 提权），容器 root 可以逃逸到宿主机 root。永远用非 root 用户。

**误区二："我用了精简镜像，所以安全了。"**

事实：精简镜像减少了攻击面，但不等于没有漏洞。仍需要定期扫描和更新。

**误区三："Dockerfile 加了 USER 就安全了。"**

事实：`USER` 只是进程 UID，但如果镜像里没有去除 setuid 二进制（如 `sudo`、`passwd`），普通用户仍可能提权。建议：

```dockerfile
# 去掉 setuid/setgid 位
RUN find / -perm /6000 -type f -exec chmod a-s {} \; 2>/dev/null || true
```

**误区四："Falco 装了就安全了。"**

事实：Falco 是检测工具不是防御工具。规则需要根据业务不断调优，告警需要与响应流程联动。装了不配规则等于白装。

---

## 总结

容器安全不是单一环节能解决的。从**构建时扫描镜像、写安全的 Dockerfile**，到**部署时加固配置、限制能力**，再到**运行时持续监控**，三层防御缺一不可。

真正落地的安全策略，不是纸上谈兵的原则，而是 CI 脚本里的 `exit-code: 1`、Dockerfile 里的 `USER 1000`、K8s 清单里的 `readOnlyRootFilesystem: true` 和 Falco 规则里的一条条告警。

从今天做起：选镜像 → 写 Dockerfile → 加 seccomp → 开 Falco。四步走完，你的容器已经比 90% 的线上项目安全。