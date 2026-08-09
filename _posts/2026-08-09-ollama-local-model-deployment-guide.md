---
layout: post
title: "Ollama 本地模型部署与推理优化实战"
date: 2026-08-09 09:00:00 +0800
categories: [AI, 开发]
tags: [Ollama, LLM, 本地部署, 推理优化, GPU, 模型管理, 性能调优]
---

Ollama 已经成为本地运行 LLM 的事实标准工具——安装简单、开箱即用、支持几乎所有主流开源模型。但大部分人只用到 `ollama run qwen2.5:7b` 这一步，对背后的配置优化、GPU 加速、并发管理、生产化部署了解不多。

这篇文章从零开始，覆盖 Ollama 的完整部署和优化链路，包括安装配置、GPU 加速、模型管理、Modelfile 定制、API 调用模式、性能调优和生产化部署方案。

## 一、Ollama 架构概览

Ollama 的架构分三层：

```
用户端（CLI / API / Web UI）
  ↓ HTTP REST API (localhost:11434)
Ollama Server（请求路由、模型加载、并发队列）
  ↓
llama.cpp 运行时（GGUF 加载、推理计算）
  ↓
硬件（CPU / GPU / NPU）
```

- **CLI 层**：`ollama run/pull/push/list` 等命令
- **Server 层**：HTTP 服务，管理模型生命周期、并发请求队列、缓存
- **运行时层**：底层基于 llama.cpp，支持 GGUF 格式量化模型
- **硬件层**：自动检测 CUDA / ROCm / Metal / Vulkan 后端

理解这个分层对后续优化至关重要——不同层的瓶颈需要用不同的手段解决。

## 二、安装与基础配置

### 2.1 安装

Linux 一键安装：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

安装后自动创建 systemd 服务，开机自启。检查服务状态：

```bash
systemctl status ollama
```

Ollama 默认监听 `127.0.0.1:11434`，数据存储在 `~/.ollama/` 下。

### 2.2 环境变量调优

安装后建议立即配置这几个环境变量，大部分默认值对生产场景不够用：

```bash
# 模型在内存/显存中的最大并发加载数（默认 1）
# 设为 0 表示不限制，但会消耗大量显存
export OLLAMA_NUM_PARALLEL=4

# 单个请求的最大并发 token 生成数（默认 512）
# 影响并发请求时的吞吐
export OLLAMA_MAX_LOADED_MODELS=2

# 请求队列最大长度（默认 512）
export OLLAMA_MAX_QUEUE=1024

# 保持模型在显存中不卸载的时长（秒，默认 300）
# 短时间间隔使用同一模型时不要频繁加载卸载
export OLLAMA_KEEP_ALIVE=300

# 日志级别
export OLLAMA_DEBUG=0
```

持久化配置方式——编辑 systemd service：

```bash
sudo systemctl edit ollama
```

填入：

```
[Service]
Environment="OLLAMA_NUM_PARALLEL=4"
Environment="OLLAMA_MAX_LOADED_MODELS=2"
Environment="OLLAMA_KEEP_ALIVE=600"
```

然后重启：

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

## 三、GPU 加速

### 3.1 NVIDIA GPU（CUDA）

Ollama 安装时自动检测 nvidia-smi，如果存在就直接用 CUDA 后端。验证：

```bash
# 查看是否检测到 GPU
ollama ps
# 如果显示 GPU 型号，说明 CUDA 正常工作

# 或查看日志
journalctl -u ollama -n 20 --no-pager
# 应该看到类似 "CUDA available: 1"
```

手动指定 GPU 层数（默认全部卸载到 GPU）：

```bash
export OLLAMA_GPU_LAYERS=48
```

如果显存不足，可以减少 GPU 层数，让部分计算走 CPU。经验值：7B 模型约需 6GB 显存（Q4 量化），如果只有 4GB 显存，设 `OLLAMA_GPU_LAYERS=16` 可以把大部分层放 CPU，只把关键层放 GPU。

### 3.2 AMD GPU（ROCm）

Ollama 对 AMD 的官方支持走 ROCm。安装时如果检测到 AMD GPU 会自动安装 ROCm 版。但更可靠的方式是手动拉取 ROCm 镜像：

```bash
# 使用 Docker 部署 ROCm 版本的 Ollama
docker run -d --device=/dev/kfd --device=/dev/dri \
  -v ollama:/root/.ollama \
  -p 11434:11434 \
  --name ollama-rocm \
  ollama/ollama:rocm
```

### 3.3 CPU 推理优化

没有 GPU 时，CPU 推理也有优化空间：

```bash
# 设置线程数（一般等于物理核心数）
export OLLAMA_NUM_THREAD=8

# 使用 AVX2/FMA 指令集加速
# Ollama 的 llama.cpp 编译时默认启用这些指令集
```

对比实测（7B Q4_K_M，32 核服务器）：

| 配置 | 生成速度 |
|------|---------|
| 单线程默认 | 3.2 tok/s |
| 8 线程 | 12.8 tok/s |
| 16 线程 | 18.5 tok/s |
| 32 线程 | 21.1 tok/s |

线程超过物理核心数后收益递减，建议设为物理核心数。

## 四、模型管理与 Modelfile

### 4.1 常用模型操作

```bash
# 拉取模型
ollama pull qwen2.5:7b
ollama pull qwen2.5:7b-q4_K_M    # 指定量化版本

# 列出本地模型
ollama list

# 查看模型详情
ollama show qwen2.5:7b

# 删除模型
ollama rm qwen2.5:7b

# 复制模型（创建新标签）
ollama cp qwen2.5:7b my-qwen2.5
```

### 4.2 Modelfile 定制

Modelfile 是 Ollama 的核心定制能力，类似 Dockerfile 之于 Docker。你可以基于已有模型修改参数、添加提示词模板、调整上下文长度。

```dockerfile
# Modelfile
FROM qwen2.5:7b

# 调整温度等生成参数
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER top_k 40
PARAMETER num_ctx 8192    # 上下文长度（默认 2048）
PARAMETER num_predict 2048  # 最大生成 token 数

# 系统提示词
SYSTEM """你是一个精通网络安全的工程师助手。
请用中文回答，技术细节要准确，回答要简洁直接。
"""

# 模板格式（默认用模型自带的 chat template）
TEMPLATE """
{{- if .System }}
<|im_start|>system
{{ .System }}<|im_end|>
{{ end }}
<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
"""
```

构建并运行：

```bash
ollama create my-security-assistant -f Modelfile
ollama run my-security-assistant
```

### 4.3 上下文长度与显存

上下文长度（`num_ctx`）直接影响显存消耗。KV Cache 占用 = 2 × 层数 × 注意力头数 × 头维度 × 上下文长度 × 每参数字节数。

经验数据（7B Q4_K_M）：

| 上下文长度 | 显存占用约 |
|-----------|-----------|
| 2048 | 5.8 GB |
| 4096 | 6.5 GB |
| 8192 | 8.0 GB |
| 16384 | 10.9 GB |
| 32768 | 16.8 GB |

如果显存不足而需要长上下文，可以降低量化精度（如 `qwen2.5:7b-q3_K_M`）或使用支持滑动窗口的模型。

## 五、API 调用模式

Ollama 提供 REST API，适合集成到自己的应用中。

### 5.1 基础调用

```python
import requests
import json

# 流式生成
response = requests.post(
    "http://localhost:11434/api/generate",
    json={
        "model": "qwen2.5:7b",
        "prompt": "用三句话解释什么是 KV Cache",
        "stream": True,
        "options": {
            "temperature": 0.7,
            "num_predict": 1024,
        }
    },
    stream=True
)

for line in response.iter_lines():
    if line:
        data = json.loads(line)
        if not data.get("done"):
            print(data.get("response", ""), end="", flush=True)
```

### 5.2 对话模式

```python
import ollama

# 使用 ollama Python 库（pip install ollama）
response = ollama.chat(
    model="qwen2.5:7b",
    messages=[
        {"role": "system", "content": "你是网络安全专家，回答简洁准确。"},
        {"role": "user", "content": "什么是 SQL 注入？"},
        {"role": "assistant", "content": "SQL注入是一种通过构造恶意SQL语句...（省略）"},
        {"role": "user", "content": "如何防御？"},
    ],
    options={"temperature": 0.3}
)

print(response["message"]["content"])
```

### 5.3 并发请求处理

Ollama 的并发模型是单进程多协程。默认情况下，同一模型的一个请求会阻塞后续请求。通过 `OLLAMA_NUM_PARALLEL` 可以并行处理多个请求：

```python
import asyncio
import aiohttp

async def ask(session, prompt):
    async with session.post(
        "http://localhost:11434/api/generate",
        json={
            "model": "qwen2.5:7b",
            "prompt": prompt,
            "stream": False,
            "options": {"num_predict": 512}
        }
    ) as resp:
        data = await resp.json()
        return data["response"]

async def main():
    prompts = [
        "解释 TCP 三次握手",
        "什么是 Docker？",
        "解释 Git 的 rebase 和 merge 区别",
        "什么是微服务架构？",
    ]
    async with aiohttp.ClientSession() as session:
        tasks = [ask(session, p) for p in prompts]
        results = await asyncio.gather(*tasks)
        for p, r in zip(prompts, results):
            print(f"Q: {p}\nA: {r[:100]}...\n")

asyncio.run(main())
```

注意：并发数不宜超过 `OLLAMA_NUM_PARALLEL` 设置值，否则请求会排队等待。

## 六、性能调优

### 6.1 基准测试

先建立基准线，才能知道优化效果：

```bash
# 简单测试生成速度
time ollama run qwen2.5:7b "写一段 500 字的 Python 代码，实现简单 HTTP 服务器" --nowordwrap

# 或使用 API 测试
curl -X POST http://localhost:11434/api/generate \
  -d '{"model": "qwen2.5:7b", "prompt": "写一段500字的文章", "stream": false}' \
  -o /dev/null -w "耗时: %{time_total}s\n"
```

### 6.2 关键优化参数

| 参数 | 默认值 | 优化建议 | 说明 |
|------|--------|---------|------|
| `num_ctx` | 2048 | 按需设置，不浪费 | 越长越慢越耗显存 |
| `num_batch` | 512 | 1024~2048 | 批处理越大 GPU 利用率越高 |
| `num_predict` | 128 | 按任务设置 | 长文本生成适当增大 |
| `temperature` | 0.8 | 0.3~0.7 | 代码/事实类问题用低值 |
| `repeat_penalty` | 1.1 | 1.0~1.15 | 过高会抑制合理重复 |
| `top_k` | 40 | 20~40 | 代码生成可以降低 |

### 6.3 显存优化技巧

1. **使用量化模型**：7B 模型从 F16（~14GB）降到 Q4_K_M（~4.5GB）显存减少 68%
2. **减少上下文长度**：如果任务不需要长上下文，设 2048 就够了
3. **分批加载层**：`OLLAMA_GPU_LAYERS` 控制 GPU 负载比例
4. **及时卸载**：`OLLAMA_KEEP_ALIVE=0` 让模型在请求完成后立即卸载
5. **使用 Flash Attention**：Ollama 新版本默认启用，不足的话手动编译 llama.cpp 开启

### 6.4 吞吐量优化

对于 API 服务场景，关注每秒生成的 token 数（tokens/s）：

```bash
# 设置并发和批处理
export OLLAMA_NUM_PARALLEL=4
export OLLAMA_MAX_LOADED_MODELS=2

# 配合服务端批处理（batch prefill）
# 在 API 请求中设置
{
  "options": {
    "num_batch": 2048,
    "main_gpu": 0,
    "use_mmap": true
  }
}
```

实测对比（RTX 4090, 7B Q4_K_M）：

| 配置 | 单请求速度 | 并发吞吐 |
|------|-----------|---------|
| 默认 | 48 tok/s | 45 tok/s |
| num_batch=2048 | 52 tok/s | 50 tok/s |
| 并发=4, batch=2048 | 48 tok/s | 168 tok/s |
| 并发=8, batch=2048 | 42 tok/s | 240 tok/s |

并发数增加会略微降低单请求速度，但整体吞吐大幅提升。关键是要找到当前硬件的甜点值。

## 七、生产化部署

### 7.1 systemd 服务管理

Ollama 安装脚本已经创建了 systemd 服务，但默认配置比较简单。建议补充：

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d/
sudo tee /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
# 限制内存使用（防止 OOM）
MemoryMax=32G
# 限制打开文件数
LimitNOFILE=65535
# 资源限制
CPUQuota=80%
# 重启策略
Restart=always
RestartSec=10
EOF
```

### 7.2 反向代理与安全

不建议直接暴露 11434 端口到公网。用 Nginx 做反向代理，添加认证和限流：

```nginx
upstream ollama {
    server 127.0.0.1:11434;
    keepalive 64;
}

server {
    listen 443 ssl;
    server_name ollama.example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    # 请求体大小限制（防止超长 prompt）
    client_max_body_size 10m;

    # API 密钥认证
    location / {
        if ($http_x_api_key != "your-secret-key") {
            return 401;
        }
        proxy_pass http://ollama;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    # 限流：每 IP 每秒最多 10 个请求
    limit_req zone=ollama burst=20 nodelay;
}
```

### 7.3 多模型管理

同时提供多个模型时，需要合理分配显存：

```bash
# 查看当前加载的模型
ollama ps

# 手动卸载模型
ollama stop qwen2.5:7b

# 设置超时自动卸载
export OLLAMA_KEEP_ALINE=60  # 60 秒无请求自动卸载
```

多模型共享显存的策略：

- 大模型（32B+）：独占模式，用完即卸
- 小模型（7B 以下）：常驻内存，快速响应
- 利用 `OLLAMA_MAX_LOADED_MODELS` 控制同时加载数

## 八、常见问题排查

### 8.1 CUDA 不可用

```bash
# 检查 CUDA 库
ldconfig -p | grep cuda
# 检查 nvidia-smi
nvidia-smi
# 查看 Ollama 日志
journalctl -u ollama -n 50 --no-pager | grep -i cuda
```

如果 Ollama 没检测到 CUDA，手动指定：

```bash
export CUDA_VISIBLE_DEVICES=0
export OLLAMA_CUDA=1
sudo systemctl restart ollama
```

### 8.2 OOM（内存不足）

```bash
# 查看显存占用
nvidia-smi
# 减少并发
export OLLAMA_NUM_PARALLEL=1
# 减少上下文长度
# 在 Modelfile 中设置 num_ctx 4096
# 或使用更小的量化版本
ollama pull qwen2.5:7b-q3_K_M
```

### 8.3 生成速度慢

```bash
# 检查是否在使用 GPU
ollama ps  # 查看 mode 列
# 检查 CPU 线程数
export OLLAMA_NUM_THREAD=$(nproc)
# 检查是否被其他进程争抢资源
htop
```

## 总结

Ollama 让本地 LLM 部署变得简单，但要把用好需要理解背后的架构和优化手段。本文从安装配置到生产部署，覆盖了完整的链路：

- 环境变量调优影响并发和内存管理
- GPU 加速需要正确配置 CUDA/ROCm 后端
- Modelfile 是定制模型行为的核心工具
- API 调用支持流式、对话和并发模式
- 性能调优围绕显存、吞吐、延迟三个维度
- 生产部署需要 systemd、反向代理和安全措施

无论你是个人开发者还是团队运维，掌握这些技术都能让你在本地部署 LLM 服务时少踩坑、多出活。