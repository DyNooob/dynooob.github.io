---
layout: post
title: "Docker 镜像构建优化：从 1GB 到 100MB"
date: 2026-08-10 09:00:00 +0800
categories: [开发]
tags: [Docker, 镜像优化, 多阶段构建, Distroless, Container, DevOps, 最佳实践]
---

Docker 镜像太大，不仅浪费磁盘空间，还拖慢 CI/CD 流水线和部署速度。一个 1GB 的镜像传到生产环境，等半天；一个 100MB 的镜像，几秒搞定。

但很多人写 Dockerfile 的方式是：`FROM python:3.12`，然后 `pip install` 一堆包，最后 `COPY . .`。结果镜像里塞满了编译工具链、缓存文件、文档和用不到的依赖。

本文从基础优化到高级技巧，给出可以直接落地的方案。

## 目录

1. [为什么镜像会变大](#为什么镜像会变大)
2. [基础优化：层叠缓存与清理](#基础优化层叠缓存与清理)
3. [多阶段构建：编译与运行分离](#多阶段构建编译与运行分离)
4. [Distroless 镜像：最小化运行环境](#distroless-镜像最小化运行环境)
5. [Alpine 的陷阱与正确用法](#alpine-的陷阱与正确用法)
6. [高级技巧：DockerSlim 与 Bazel](#高级技巧dockerslim-与-bazel)
7. [实战案例：Go 应用从 1.2GB 到 12MB](#实战案例go-应用从-12gb-到-12mb)
8. [总结](#总结)

## 为什么镜像会变大

先看一个典型的"坏" Dockerfile：

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

这个镜像有多大？`python:3.12-slim` 本身约 120MB。安装 Flask、gunicorn、psycopg2、numpy 等常用包后，轻松膨胀到 500MB+。

原因有三：

| 原因 | 说明 | 典型大小 |
|------|------|----------|
| 基础镜像过大 | `python:3.12` 含完整编译工具链 | 1GB+ |
| 构建依赖残留 | `gcc`、`build-essential` 编译完就不需要了 | 200-500MB |
| 缓存和临时文件 | `pip cache`、`apt lists`、`__pycache__` | 50-200MB |
| 不必要的文件 | 测试代码、文档、.git 目录 | 10-100MB |

## 基础优化：层叠缓存与清理

### 1. 合并 RUN 指令并清理缓存

Docker 的层叠机制会保留每一层的变化。如果分多个 RUN 指令，中间产物会一直留在镜像里。

```dockerfile
# 坏：pip 缓存留在镜像里
RUN pip install -r requirements.txt

# 好：安装后立即清理
RUN pip install --no-cache-dir -r requirements.txt && \
    rm -rf /root/.cache/pip
```

### 2. apt 安装后清理

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libpq-dev \
        gcc && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

关键参数：
- `--no-install-recommends`：不安装推荐的额外包，省 30-50% apt 空间
- `apt-get clean`：清除下载的 .deb 包
- `rm -rf /var/lib/apt/lists/*`：清除包索引

### 3. 合理排序指令以利用缓存

Docker 构建时，每层缓存只在该层指令和上下文没有变化时生效。把不常变的部分提前：

```dockerfile
# 好：依赖文件单独复制，利用缓存
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    rm -rf /root/.cache/pip

# 源码最后复制（改动最频繁）
COPY . .
```

改代码时只重新构建最后几层，不用每次都重新 pip install。

## 多阶段构建：编译与运行分离

多阶段构建是 Docker 镜像优化的**核心武器**。思路很简单：用第一个阶段（builder）编译、安装依赖，用第二个阶段（runtime）只复制最终产物。

### Python 应用

```dockerfile
# === 第一阶段：构建 ===
FROM python:3.12-slim AS builder
WORKDIR /build

# 安装编译依赖
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        gcc \
        libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖到指定目录
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# 复制源码
COPY . .

# === 第二阶段：运行 ===
FROM python:3.12-slim
WORKDIR /app

# 只从 builder 复制安装好的包和源码
COPY --from=builder /root/.local /root/.local
COPY --from=builder /build/app.py .

# 设置 PATH 以找到用户安装的包
ENV PATH=/root/.local/bin:$PATH

CMD ["python", "app.py"]
```

效果：从 500MB 降到 200MB，且运行时镜像不包含 gcc 等编译工具。

### Go 应用（更极致）

Go 编译成静态二进制，是镜像优化的最佳场景：

```dockerfile
# === 第一阶段：编译 ===
FROM golang:1.22 AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o app .

# === 第二阶段：运行 ===
FROM scratch
COPY --from=builder /build/app /
EXPOSE 8080
CMD ["/app"]
```

`scratch` 是空镜像，大小 0 字节。最终镜像只包含一个可执行文件。

关键优化参数：
- `CGO_ENABLED=0`：禁用 CGO，生成静态链接二进制
- `-ldflags="-s -w"`：去掉调试信息和符号表，再省 20-30%
- `GOOS=linux`：明确目标平台

## Distroless 镜像：最小化运行环境

Google 的 [distroless](https://github.com/GoogleContainerTools/distroless) 镜像只包含运行时必要组件——glibc、SSL 证书、时区数据——没有 shell、包管理器、甚至没有 `ls`。

```dockerfile
# 多阶段构建 + distroless
FROM golang:1.22 AS builder
WORKDIR /build
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o app .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /build/app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```

优点：
- 镜像极小：Go 应用通常 10-20MB
- 攻击面最小：没有 shell、没有包管理器、没有 suid 二进制
- 安全合规：通过 CVE 扫描的问题少 90% 以上

缺点：
- 无法 exec 进入容器调试（没有 shell）
- 需要静态编译或自带动态库

对于 Python 应用，可以使用 `gcr.io/distroless/python3-debian12`：

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt
COPY . .

FROM gcr.io/distroless/python3-debian12
COPY --from=builder /root/.local /root/.local
COPY --from=builder /build /app
WORKDIR /app
ENV PATH=/root/.local/bin:$PATH
CMD ["app.py"]
```

## Alpine 的陷阱与正确用法

Alpine 基于 musl libc 而非 glibc，镜像小（~5MB），但有几个常见陷阱：

### 陷阱 1：musl 兼容性问题

某些 Python 包（如 `psycopg2`、`cryptography`）在 Alpine 上需要额外编译，且可能有运行时差异。

```dockerfile
# 在 Alpine 上安装 psycopg2 需要手动装依赖
FROM python:3.12-alpine
RUN apk add --no-cache postgresql-dev gcc musl-dev
```

### 陷阱 2：DNS 解析问题

musl 的 DNS 解析器不支持 `search` 域和 `ndots` 选项，导致 Kubernetes 中服务名解析失败。

```yaml
# 需要在 Kubernetes Pod 中手动修复
dnsConfig:
  options:
    - name: ndots
      value: "1"
```

### 陷阱 3：时区问题

Alpine 默认没有时区数据：

```dockerfile
RUN apk add --no-cache tzdata
ENV TZ=Asia/Shanghai
```

### 什么时候用 Alpine

| 场景 | 推荐 |
|------|------|
| Go 静态二进制 | `scratch` 或 `distroless` |
| Python 纯代码 | distroless 或 slim |
| 需要 shell 调试 | `slim` 或 `alpine` |
| 需要 glibc | `slim`（Debian 系） |
| 极致小体积 | `alpine`（但先测试兼容性） |

一句话：**能用 distroless 就用 distroless，用不了再用 slim，Alpine 是最后的选择**。

## 高级技巧：DockerSlim 与 Bazel

### DockerSlim：自动瘦身

[DockerSlim](https://github.com/slimtoolkit/slim) 能自动分析镜像，移除不需要的文件：

```bash
# 安装
curl -L -o ds https://github.com/slimtoolkit/slim/releases/latest/download/linux_amd64.tar.gz
tar -xzf linux_amd64.tar.gz

# 自动瘦身
./dist_linux/docker-slim build --target myapp:latest --tag myapp:slim
```

它通过静态分析 + 动态跟踪，自动检测镜像中哪些文件是实际需要的，然后构建一个最小镜像。通常能减少 50-90% 的体积。

### Bazel 构建

Google 内部使用 Bazel 构建 Docker 镜像，只打包源码中实际依赖的文件，不携带任何多余文件。

```python
# BUILD.bazel
load("@io_bazel_rules_docker//container:image.bzl", "container_image")
load("@io_bazel_rules_go//go:def.bzl", "go_image")

go_image(
    name = "app",
    srcs = ["main.go"],
    importpath = "github.com/example/app",
    goarch = "amd64",
    goos = "linux",
    static = True,
    pure = True,
)
```

Bazel 构建的镜像天然就是最小化的——只包含编译出的二进制和显式声明的依赖文件。

## 实战案例：Go 应用从 1.2GB 到 12MB

以一个真实项目为例，逐步优化一个 Go Web 应用：

### 步骤 1：原始版本

```dockerfile
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o app .
CMD ["./app"]
```

镜像大小：**1.2GB**（包含 Go 工具链、全部源码、缓存）

### 步骤 2：多阶段构建

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o app .

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/app /
CMD ["/app"]
```

镜像大小：**128MB**（-90%）

### 步骤 3：静态编译 + distroless

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o app .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/app /
CMD ["/app"]
```

镜像大小：**12MB**（-99%）

### 步骤 4：验证

```bash
# 查看镜像大小
docker images myapp
# REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
# myapp        slim       abc123def456   10 seconds ago   12MB

# 运行测试
docker run --rm myapp

# 安全扫描
trivy image myapp:slim
# 通常 0 个高危 CVE（distroless 的 CVE 极少）
```

## 总结

| 优化手段 | 最大减量 | 适用场景 | 复杂度 |
|----------|----------|----------|--------|
| 合并 RUN 指令 + 清理缓存 | 30-50% | 所有 Dockerfile | 低 |
| 多阶段构建 | 60-80% | 需要编译的语言（Go、Rust、Java、C++） | 低 |
| Distroless 基础镜像 | 50-80% | 生产环境，不调试 | 中 |
| Alpine 基础镜像 | 30-50% | 不需要 glibc 的场景 | 中 |
| 静态编译 | 90-99% | Go、Rust、纯 C 应用 | 低 |
| DockerSlim 自动瘦身 | 50-90% | 现有镜像，不想改 Dockerfile | 高 |
| Bazel 构建 | 80-99% | 大型 monorepo 项目 | 高 |

**最佳实践建议：**

1. 所有新项目默认使用多阶段构建
2. Go/Rust 应用用 scratch 或 distroless + 静态编译
3. Python/Node 应用用 distroless 或 slim + 清理缓存
4. 生产环境镜像不要保留 shell 和包管理器
5. 定期用 `trivy` 或 `grype` 扫描镜像 CVE
6. 在 CI 中自动检查镜像大小，超过阈值则告警

镜像大小不是目的，**部署速度、安全性和可维护性**才是。一个 12MB 的镜像比 1.2GB 的镜像部署快 100 倍，被攻击面小 100 倍，这才是真正价值。