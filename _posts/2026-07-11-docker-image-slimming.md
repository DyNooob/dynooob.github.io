---
layout: post
title: "Docker 镜像瘦身：从 2GB 到 300MB"
date: 2026-07-11 18:00:00 +0800
categories: [开发]
tags: [Docker, 容器, 镜像优化, 部署]
---

跑 AI 项目时经常遇到一个问题：一个 Docker 镜像动不动就 2GB 以上，传输慢、占用磁盘、拉取也慢。尤其是用 CUDA 基础镜像的，更是重量级选手。

其实大多数场景下，镜像可以瘦到原来的十分之一。

## 最常见的体积杀手

```dockerfile
FROM python:3.11
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

这个 Dockerfile 出来的镜像至少 1GB+。原因：

1. `python:3.11` 完整镜像自带大量系统工具和文档
2. pip 安装时下载的 wheel 缓存不清理
3. 构建中间层全部保留

## 方案一：用 slim 镜像

```dockerfile
FROM python:3.11-slim
```

`python:3.11-slim` 约 120MB，而完整版约 900MB。少了 gcc、make、系统工具等编译依赖，但 Python 运行环境完整。

如果你的包需要编译（如 numpy、pandas），需要额外装 build-essential：

```dockerfile
FROM python:3.11-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

## 方案二：多阶段构建

最实用的瘦身手段。把构建环境和运行环境分开：

```dockerfile
# 第一阶段：构建
FROM python:3.11-slim AS builder

RUN apt-get update && apt-get install -y --no-install-recommends build-essential
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 第二阶段：运行
FROM python:3.11-slim

COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . /app
WORKDIR /app

CMD ["python", "app.py"]
```

最终镜像只包含运行需要的文件，build-essential、pip 缓存、编译中间产物全都不在最终层里。

## 方案三：CUDA 镜像的瘦身

AI 项目的痛点。`nvidia/cuda:12.1-runtime` 约 1.5GB，`nvidia/cuda:12.1-devel` 约 3.5GB。

```dockerfile
# 只装 runtime，不装 devel
FROM nvidia/cuda:12.1-runtime

# 或者用更轻量的 nvidia/cuda-base
FROM nvidia/cuda:12.1-base
```

如果只是跑推理（不需要编译 CUDA 代码），用 `runtime` 就够了。`devel` 镜像多了编译器、头文件、静态库，体积翻倍。

## 方案四：清理 pip 和 apt 缓存

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt

RUN apt-get update && apt-get install -y \
    libgl1 \
    && rm -rf /var/lib/apt/lists/*
```

`--no-cache-dir` 不让 pip 保存 wheel 缓存。`rm -rf /var/lib/apt/lists/*` 删除 apt 下载的包索引。

## 查看镜像体积

```bash
# 查看所有镜像
docker images

# 查看具体镜像的层
docker history 镜像名:tag

# 分析体积
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    ghcr.io/wagoodman/dive:latest 镜像名:tag
```

`dive` 是最好用的镜像分析工具，可以逐层查看每个文件，找到哪些文件占用了空间。

## 实战对比

以一个 FastAPI + 机器学习推理服务为例：

| 优化步骤 | 镜像体积 | 节省 |
|---------|---------|------|
| 原始（python:3.11） | 1.8 GB | - |
| 改用 slim | 950 MB | 47% |
| 多阶段构建 | 520 MB | 71% |
| 清理 pip 缓存 | 480 MB | 73% |
| 精简依赖 | 320 MB | 82% |

从 1.8GB 到 320MB，省了 80%。

## 几个原则

- 尽量用 `-slim` 或 `-alpine` 基础镜像
- 构建和运行环境分离（多阶段构建）
- pip 和 apt 的缓存要在同一层清理
- `.dockerignore` 排除不需要的文件

```dockerignore
__pycache__/
*.pyc
.git/
.vscode/
venv/
*.md
tests/
```

这样每次构建时，不会被本地开发文件污染镜像上下文。

## 什么时候不用太在意体积

- 只在本地开发调试的镜像
- 内网传输带宽充足的场景
- 需要完整调试工具链的开发环境

瘦身是为了解决传输慢、部署慢的问题。如果没这些痛点，保持可读性比节省几百 MB 更重要。