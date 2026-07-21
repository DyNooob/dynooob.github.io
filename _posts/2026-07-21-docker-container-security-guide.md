---
layout: post
title: "Docker 容器安全实战：从镜像扫描到运行时防护"
date: 2026-07-21 15:00:00 +0800
categories: [网络安全, 开发]
tags: [Docker, 容器安全, 安全加固, Trivy, 容器逃逸, Seccomp, 安全基线]
---

容器安全不是"加个防火墙"那么简单。Docker 默认配置偏向易用性，很多安全选项默认关闭。一旦攻破容器，攻击者可能直接获得宿主机 root 权限。本文从镜像构建、安全扫描、运行时加固三个层面，给出可以直接落地的操作方案。

## 一、镜像安全：从源头堵漏

### 1.1 选择最小基础镜像

基础镜像越大，攻击面越大。优先选择：

```dockerfile
# 不推荐
FROM ubuntu:22.04          # ~77MB，但自带大量工具
FROM python:3.11            # ~1GB，包含编译器和文档

# 推荐
FROM alpine:3.19            # ~7MB，musl libc
FROM python:3.11-alpine     # ~50MB，够用
FROM scratch                # 0MB，静态编译专用
FROM distroless              # Google 维护，只含运行时
```

Alpine 虽然小，但 musl libc 可能和某些 Python C 扩展不兼容。生产环境更推荐 **distroless** 镜像：

```dockerfile
FROM python:3.11-slim-bookworm AS builder
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential libssl-dev \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.11-slim-bookworm
COPY --from=builder /root/.local /root/.local
COPY app.py .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```

distroless 镜像里没有 shell、没有 apt、没有包管理器，攻击者即使打进容器也无法执行反弹 shell。

### 1.2 多阶段构建消除构建产物

```dockerfile
# 阶段一：编译
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app

# 阶段二：运行
FROM scratch
COPY --from=builder /app /app
EXPOSE 8080
CMD ["/app"]
```

最终镜像只有编译好的二进制文件，没有编译器、依赖库、源码。体积从 800MB 降到 10MB，攻击面也同步缩小。

### 1.3 镜像扫描

用 Trivy 扫描镜像中的已知漏洞，这是最基础的防线：

```bash
# 安装 Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# 扫描镜像
trivy image python:3.11-slim

# 扫描 Dockerfile 中的错误配置
trivy config --severity HIGH,CRITICAL Dockerfile

# 输出 HTML 报告
trivy image --format html --output report.html myapp:latest
```

Trivy 扫描结果示例：

```
python:3.11-slim (debian 12.0)
================================
Total: 45 (UNKNOWN: 0, LOW: 20, MEDIUM: 18, HIGH: 6, CRITICAL: 1)

┌─────────────┬───────────────┬──────────┬──────────────┐
│   Library   │ Vulnerability │ Severity │   Status     │
├─────────────┼───────────────┼──────────┼──────────────┤
│ libssl3     │ CVE-2026-1234 │ CRITICAL │ fixed 3.1.5  │
│ libcrypto3  │ CVE-2026-1234 │ CRITICAL │ fixed 3.1.5  │
│ curl        │ CVE-2026-5678 │ HIGH     │ fixed 8.7.1  │
└─────────────┴───────────────┴──────────┴──────────────┘
```

建议集成到 CI/CD 中，阻止高危漏洞镜像进入生产：

```yaml
# .github/workflows/security.yml
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
          format: sarif
          output: trivy-results.sarif
          severity: HIGH,CRITICAL
          exit-code: 1  # 发现漏洞就失败
```

## 二、运行时安全：纵深防御

### 2.1 禁止特权模式

`--privileged` 是容器安全的第一大敌。开启后容器拥有宿主机全部内核能力，可以挂载设备、加载内核模块、直接访问硬件。

```bash
# 绝对不要这样
docker run --privileged myapp

# 正确的做法：按需授予能力
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

`--cap-drop=ALL` 丢弃所有内核能力，然后只添加真正需要的。Web 应用通常只需要 `NET_BIND_SERVICE`（绑定 1024 以下端口）。

### 2.2 容器逃逸防御

常见的容器逃逸路径：

| 逃逸方式 | 触发条件 | 防御措施 |
|---------|---------|---------|
| 内核漏洞 | 宿主内核有漏洞 | 保持内核更新，禁用 `SYS_ADMIN` |
| 挂载 Docker socket | 容器内挂载 `/var/run/docker.sock` | 不要挂载 socket |
| 挂载宿主机目录 | 可写挂载 + 提权 | 用只读挂载，或用 `--userns-remap` |
| `--privileged` | 粗暴开启所有权限 | 禁止使用 |
| 能力滥用 | `SYS_ADMIN`, `SYS_PTRACE` | 按需授予，不要给 `SYS_ADMIN` |

最危险的挂载是 Docker socket：

```bash
# 危险：容器内可以执行 docker 命令
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp

# 一旦容器被攻破，攻击者可以：
# docker run --privileged -v /:/host ubuntu chroot /host
# 直接拿到宿主机 root
```

### 2.3 Seccomp 安全计算模式

Seccomp 限制容器可以调用的系统调用数量。Docker 默认有一个 seccomp 白名单（约 300 个系统调用），但我们可以进一步收紧：

```bash
# 查看默认 seccomp 配置
docker run --rm alpine cat /etc/containers/seccomp.json

# 使用自定义 seccomp 配置
docker run --security-opt seccomp=./custom-seccomp.json myapp
```

自定义 seccomp 配置示例（只允许 Web 应用需要的系统调用）：

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat", "fstat",
                "mmap", "munmap", "brk", "sched_yield", "nanosleep",
                "clock_gettime", "getdents64", "exit_group", "rt_sigaction"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

这个配置只允许必要的系统调用，禁止了 `clone`（创建进程）、`execve`（执行新程序）、`mount` 等高危调用，即使攻击者拿到 RCE 也跑不了反弹 shell。

### 2.4 AppArmor / SELinux

```bash
# 加载 AppArmor 配置文件
apparmor_parser -r /etc/apparmor.d/docker-myapp

# 运行容器时指定
docker run --security-opt apparmor=docker-myapp myapp
```

AppArmor 配置文件示例：

```
#include <tunables/global>

profile docker-myapp flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # 禁止网络访问
  deny network,

  # 只读挂载
  / r,
  /proc/** r,
  /app/** r,

  # 禁止写入关键目录
  deny /etc/** w,
  deny /usr/** w,
  deny /bin/** w,
}
```

### 2.5 只读根文件系统

```bash
docker run --read-only --tmpfs /tmp --tmpfs /var/run myapp
```

`--read-only` 让容器的根文件系统变成只读。容器无法修改 `/etc`、`/bin` 等目录，即使有 RCE 漏洞也无法写文件。需要临时写入的目录（如 `/tmp`）用 `--tmpfs` 挂载到内存中。

### 2.6 用户命名空间重映射

默认情况下，容器内的 root（UID 0）在宿主机上也是 root（UID 0），只是受 Namespace 隔离。但一旦漏洞绕过隔离，就是 root 权限。

```bash
# 编辑 /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

开启后，容器内的 root 在宿主机上映射为普通用户（通常是 165536:65536）。即使发生逃逸，攻击者拿到的也是普通用户权限。

注意：开启 userns-remap 后，需要重新调整挂载卷的权限：

```bash
# 查看映射的用户
cat /etc/subuid
# docker:dynooob:65536
# 容器内 root 对应宿主机 UID 165536

# 调整卷权限
sudo chown -R 165536:165536 /data/volumes/myapp
```

### 2.7 资源限制

限制 CPU 和内存可以防止 DoS 攻击导致宿主机崩溃：

```bash
docker run \
  --memory=512m \
  --memory-swap=512m \       # 禁用 swap，防止 OOM 后变慢
  --cpus=0.5 \
  --pids-limit=100 \          # 限制进程数，防止 fork bomb
  --restart=on-failure:5 \    # 最多重启 5 次
  myapp
```

`--pids-limit` 很关键：没有限制时，一个容器可以 fork 出上万个进程，耗尽宿主机的 PID 上限。

## 三、Docker 守护进程安全

### 3.1 配置 TLS 认证

Docker API 默认监听 Unix socket（`/var/run/docker.sock`），安全可控。但如果暴露 TCP 端口，必须用 TLS：

```bash
# 生成 CA 和证书
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
openssl genrsa -out server-key.pem 4096
openssl req -subj "/CN=$(hostname)" -sha256 -new -key server-key.pem -out server.csr
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem
```

配置 daemon.json：

```json
{
  "tlsverify": true,
  "tlscacert": "/etc/docker/ca.pem",
  "tlscert": "/etc/docker/server-cert.pem",
  "tlskey": "/etc/docker/server-key.pem",
  "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"]
}
```

### 3.2 Docker Content Trust

启用内容信任，确保只拉取经过签名的镜像：

```bash
export DOCKER_CONTENT_TRUST=1
docker pull alpine:latest
# 如果镜像没有签名，会报错
```

也可以全局启用：

```json
{
  "content-trust": {
    "mode": "enabled"
  }
}
```

### 3.3 审计日志

```bash
# 监控 Docker 守护进程事件
docker events --filter 'event=create' --filter 'event=start' --filter 'event=kill' &

# 写入系统日志
docker events --format '{{json .}}' | while read line; do
  logger -t docker-audit "$line"
done &
```

## 四、综合安全启动脚本

以下是一个生产级 Docker 启动命令，集成了本文所有安全措施：

```bash
docker run -d \
  --name myapp \
  --read-only \
  --tmpfs /tmp:noexec,nosuid,size=64m \
  --tmpfs /var/run:noexec,nosuid,size=32m \
  --tmpfs /var/log:noexec,nosuid,size=128m \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=NET_RAW \
  --security-opt=no-new-privileges:true \
  --security-opt=seccomp=./custom-seccomp.json \
  --security-opt=apparmor=docker-myapp \
  --memory=512m \
  --memory-swap=512m \
  --cpus=0.5 \
  --pids-limit=100 \
  --restart=on-failure:5 \
  --user=1000:1000 \
  myapp:latest
```

## 五、日常安全巡检清单

可以用一个脚本定期检查容器安全状态：

```bash
#!/bin/bash
# check-container-security.sh

echo "=== 容器安全巡检 ==="

# 1. 检查特权容器
echo "[1] 检查特权容器..."
PRIVILEGED=$(docker ps --quiet --filter "status=running" | xargs -I{} docker inspect {} --format '{{.Name}} {{.HostConfig.Privileged}}' | grep true)
if [ -n "$PRIVILEGED" ]; then
  echo "  WARNING: 以下容器以特权模式运行:"
  echo "$PRIVILEGED"
fi

# 2. 检查挂载了 Docker socket 的容器
echo "[2] 检查 Docker socket 挂载..."
for cid in $(docker ps -q); do
  mounts=$(docker inspect $cid --format '{{range .Mounts}}{{.Source}}{{end}}')
  if echo "$mounts" | grep -q "docker.sock"; then
    echo "  WARNING: $(docker inspect $cid --format '{{.Name}}') 挂载了 Docker socket"
  fi
done

# 3. 检查容器运行用户
echo "[3] 检查容器用户..."
for cid in $(docker ps -q); do
  user=$(docker inspect $cid --format '{{.Config.User}}')
  if [ "$user" = "" ] || [ "$user" = "0" ] || [ "$user" = "root" ]; then
    echo "  WARNING: $(docker inspect $cid --format '{{.Name}}') 以 root 运行"
  fi
done

# 4. 检查是否设置了资源限制
echo "[4] 检查资源限制..."
for cid in $(docker ps -q); do
  mem=$(docker inspect $cid --format '{{.HostConfig.Memory}}')
  if [ "$mem" = "0" ]; then
    echo "  WARNING: $(docker inspect $cid --format '{{.Name}}') 没有内存限制"
  fi
done

# 5. 扫描所有运行中镜像的漏洞
echo "[5] 扫描运行中镜像漏洞..."
for cid in $(docker ps -q); do
  image=$(docker inspect $cid --format '{{.Config.Image}}')
  trivy image --severity CRITICAL --quiet "$image" 2>/dev/null
done

echo "=== 巡检完成 ==="
```

## 总结

容器安全不是一劳永逸的配置，而是持续的过程。核心原则就三条：

1. **最小化**：最小镜像、最小权限、最小攻击面
2. **纵深防御**：镜像扫描 + 运行时限制 + 内核安全机制（Seccomp/AppArmor）
3. **默认拒绝**：先丢弃所有能力，再按需添加

以上所有配置和脚本都可以直接复制使用。建议从镜像扫描和运行时权限限制开始，这两步做对了已经能挡住 90% 的常见攻击路径。