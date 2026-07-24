---
layout: post
title: "LLM 推理中的 KV Cache 原理与优化实战"
date: 2026-07-24 09:00:00 +0800
categories: [AI/ML]
tags: [LLM, KV Cache, 推理优化, vLLM, PagedAttention, 内存管理, 模型部署]
---

大语言模型推理慢、显存吃紧，这是部署时最头疼的两个问题。很多人第一反应是"换小模型"或"上量化"，但忽略了推理引擎内部一个关键机制——**KV Cache**。理解它，你才能真正掌握推理优化的命脉。

这篇文章从原理讲到实战，涵盖 KV Cache 的内存模型、vLLM 的 PagedAttention 优化、以及如何在你自己的推理服务中榨干每一 MB 显存。

## 一、KV Cache 是什么

### 自回归解码的痛点

LLM 生成文本是逐个 token 进行的。以 "The cat sat on the" 接 "mat" 为例：

生成第 1 个 token "The" 时，模型计算所有注意力层。
生成第 2 个 token "cat" 时，需要重新计算 "The" 的注意力。
生成第 3 个 token "sat" 时，需要重新计算 "The" 和 "cat" 的注意力。

每生成一个新 token，前面所有 token 的 Key 和 Value 矩阵都要重新算一遍。序列越长，重复计算越严重——复杂度是 O(n²) 的二次增长。

### KV Cache 的核心思想

**缓存，不重算。**

在第一个 token 的计算过程中，把每一层 Attention 的 Key 矩阵和 Value 矩阵保存下来。后续生成新 token 时，只需要：

1. 计算新 token 的 Query、Key、Value
2. 从缓存读取之前所有 token 的 Key 和 Value
3. 计算当前 token 的 Attention 输出

公式表达更清晰。标准的 Attention 计算：

```
Attention(Q, K, V) = softmax(Q × K^T / √d) × V
```

有了 KV Cache 后，第 t 步的计算变为：

```
K_cache = concat(K_cache, K_t)      # 追加新 Key
V_cache = concat(V_cache, V_t)      # 追加新 Value
Attention(Q_t, K_cache, V_cache)    # 仅计算当前 Query
```

这样每步计算量从 O(n²) 降为 O(n)，推理速度提升数倍到数十倍。

## 二、KV Cache 的显存到底吃了多少

大多数人知道 KV Cache 省算力，但没意识到它吃显存有多凶。

### 计算公式

单次请求的 KV Cache 显存占用：

```
KV Cache 大小 = 2 × num_layers × num_heads × head_dim × seq_len × dtype_bytes
```

- `2`：Key 和 Value 各一份
- `num_layers`：Transformer 层数
- `num_heads`：每层注意力头数
- `head_dim`：每个头的维度
- `seq_len`：序列长度（输入 + 输出）
- `dtype_bytes`：数据类型字节数（FP16=2, FP32=4, INT8=1）

### 实际算一笔账

以 LLaMA-3-70B 为例：

| 参数 | 值 |
|------|-----|
| num_layers | 80 |
| num_heads | 64 |
| head_dim | 128 |
| dtype | FP16 (2 bytes) |

输入 1024 tokens，输出 1024 tokens（共 2048 seq_len）：

```
KV Cache = 2 × 80 × 64 × 128 × 2048 × 2
         = 2 × 80 × 64 × 128 × 4096
         = 5,368,709,120 bytes
         ≈ 5.0 GB
```

**单请求 5GB。** 如果并发 8 个请求，光 KV Cache 就吃掉 40GB 显存。

换 7B 模型看看：

| 参数 | 值 |
|------|-----|
| num_layers | 32 |
| num_heads | 32 |
| head_dim | 128 |
| dtype | FP16 |

同样 2048 seq_len：

```
KV Cache = 2 × 32 × 32 × 128 × 2048 × 2
         = 1,073,741,824 bytes
         ≈ 1.0 GB
```

7B 模型单请求也要 1GB。所以部署 LLM 时，**KV Cache 的显存占用往往比模型权重本身更难对付**——权重是固定的，KV Cache 随并发和序列长度动态增长。

### 经验公式

```
每请求 KV Cache (GB) ≈ 2 × num_layers × num_heads × head_dim × (input_len + output_len) × 2 × 1e-9
```

简化版（FP16 通用模型）：

```
~0.5 MB × num_layers × num_heads × head_dim × (seq_len / 1024)
```

## 三、传统实现的三大浪费

理解了 KV Cache 的显存占用，再看传统实现（HuggingFace Transformers、DeepSpeed）的问题：

### 1. 预分配浪费

大多数框架在请求开始时预分配最大长度的缓存。假设设置 `max_seq_len=4096`，但实际只用了 512 个 token——87.5% 的显存被浪费了。

### 2. 内存碎片

不同请求的序列长度不同，用完释放后会产生大量碎片。频繁申请释放大块连续显存，CUDA 的 `cudaMalloc` 开销不可忽视。

### 3. 无法共享

即使两个请求共用同一个前缀（如系统提示词），传统实现也不共享 KV Cache，每个请求独立缓存。

## 四、vLLM 的 PagedAttention 革命

2023 年 vLLM 团队提出的 PagedAttention 是 KV Cache 管理的里程碑。它的核心思想借鉴了操作系统虚拟内存的分页机制。

### 原理

把连续的 KV Cache 逻辑空间映射到不连续的物理块：

```
逻辑视角（连续）：
[Token 0] [Token 1] [Token 2] [Token 3] ... [Token N]

物理视角（分页）：
Block 0: [Token 0][Token 1][Token 2][Token 3]
Block 1: [Token 4][Token 5][Token 6][Token 7]
...
Block M: [Token N-3][Token N-2][Token N-1][Pad]
```

每个 Block 固定大小（通常 16 或 32 token），物理块不要求连续。这样：

- **按需分配**：序列增长时才分配新块，不预分配
- **零碎片**：固定大小块，释放后可直接复用
- **块共享**：相同前缀（如系统提示词）的请求共享同一组物理块，Copy-on-Write 机制保证正确性

### 性能对比

| 框架 | 内存利用率 | 吞吐量 | 支持前缀共享 |
|------|-----------|-------|------------|
| HF Transformers | ~20-40% | 1x | 否 |
| DeepSpeed | ~30-50% | 1.5-2x | 否 |
| vLLM (PagedAttention) | ~90-95% | 8-12x | 是 |

vLLM 在 ShareGPT 和 LLaMA 上的测试显示，PagedAttention 能将内存利用率从 20-40% 提升到 90% 以上，同等 GPU 显存下吞吐量提升 8-12 倍。

## 五、量化 KV Cache：INT8 和 FP8

除了架构优化，还有一条路：**降低 KV Cache 的数据精度**。

### INT8 KV Cache

将 FP16 的 KV Cache 量化到 INT8，显存直接减半，同时保持大部分精度。

```python
# 伪代码示意
def quantize_kv_cache(kv_cache_fp16):
    scale = kv_cache_fp16.abs().max(dim=-1, keepdim=True)
    kv_cache_int8 = (kv_cache_fp16 / scale * 127).round().to(torch.int8)
    return kv_cache_int8, scale

def dequantize_kv_cache(kv_cache_int8, scale):
    return kv_cache_int8.float() * scale / 127
```

### FP8 KV Cache

NVIDIA H100/H200 支持原生 FP8 计算。在 FP8 精度下，KV Cache 的读写带宽翻倍，且无需手动量化——硬件直接支持。

### KVCache 量化策略对比

| 策略 | 显存节省 | 精度损失 | 硬件要求 |
|------|---------|---------|---------|
| FP16（基线） | 0% | 无 | 通用 |
| INT8 对称量化 | 50% | 极小 | 通用 |
| FP8 | 50% | 极小 | H100/H200 |
| INT4 | 75% | 中等 | 实验性 |
| 稀疏化（2:4） | 50% | 小 | A100/H100 |

实践建议：线上服务先用 INT8 KV Cache，几乎无精度损失，显存直接减半。如果使用 H100，用 FP8 更省心。

## 六、实战：配置 vLLM 的 KV Cache 参数

vLLM 是目前最主流的推理引擎，我们用它演示如何调优 KV Cache。

### 基础配置

```bash
# 启动 vLLM 推理服务
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Meta-Llama-3-8B \
    --gpu-memory-utilization 0.95 \
    --max-model-len 8192 \
    --kv-cache-dtype auto \
    --swap-space 4
```

关键参数：

| 参数 | 作用 | 建议值 |
|------|------|-------|
| `--gpu-memory-utilization` | 分配给模型+KV Cache的显存比例 | 0.85-0.95 |
| `--max-model-len` | 最大序列长度 | 根据业务需求 |
| `--kv-cache-dtype` | KV Cache 精度：auto/fp8/fp8_e4m3 | auto 或 fp8 |
| `--swap-space` | CPU 交换空间（GB） | 4-16 |
| `--block-size` | PagedAttention 块大小 | 16 或 32 |
| `--max-num-batched-tokens` | 每次推理最大 token 数 | 4096-8192 |

### 监控 KV Cache 使用

```bash
# 查看 vLLM API 返回的统计信息
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Meta-Llama-3-8B",
    "messages": [{"role": "user", "content": "Hello"}],
    "max_tokens": 100
  }' | python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d.get('usage',{}), indent=2))"
```

vLLM 会在返回的 usage 中附带 KV Cache 统计信息（vLLM >= 0.4.0）。

### 内核参数调优

```bash
# 设置更大的块大小以减少内存管理开销（适合长序列）
--block-size 32

# 减小块大小以更精细地控制内存（适合短序列高并发）
--block-size 16

# 启用前缀缓存（共享系统提示词时非常有用）
--enable-prefix-caching
```

### 实际测试脚本

```python
# kv_cache_bench.py
import time
import torch
from vllm import LLM, SamplingParams

def benchmark_kv_cache(model_name, max_num_seqs, max_model_len):
    llm = LLM(
        model=model_name,
        gpu_memory_utilization=0.90,
        max_model_len=max_model_len,
        max_num_seqs=max_num_seqs,
        enable_prefix_caching=True,
    )
    
    prompts = [
        "Explain quantum computing in simple terms." * 5,
        "Write a Python function to merge two sorted lists." * 5,
        "What is the capital of France? Describe its history." * 5,
    ] * (max_num_seqs // 3)
    
    sampling_params = SamplingParams(
        temperature=0.7,
        max_tokens=512,
    )
    
    start = time.time()
    outputs = llm.generate(prompts, sampling_params)
    elapsed = time.time() - start
    
    total_tokens = sum(len(o.outputs[0].token_ids) for o in outputs)
    total_prompt = sum(len(o.prompt_token_ids) for o in outputs)
    
    print(f"Model: {model_name}")
    print(f"Concurrent: {max_num_seqs}")
    print(f"Max seq len: {max_model_len}")
    print(f"Total prompt tokens: {total_prompt}")
    print(f"Total generated tokens: {total_tokens}")
    print(f"Elapsed: {elapsed:.2f}s")
    print(f"Throughput: {total_tokens/elapsed:.1f} tok/s")
    print(f"KV Cache overhead: ~{total_tokens * 2 * 32 * 32 * 128 * 2 / 1e9:.2f} GB (estimated)")

if __name__ == "__main__":
    benchmark_kv_cache("meta-llama/Meta-Llama-3-8B", 8, 4096)
```

## 七、超越 KV Cache 的进阶优化

### 1. Multi-Query Attention (MQA) 和 Grouped-Query Attention (GQA)

标准 Multi-Head Attention (MHA) 为每个头维护独立的 Key 和 Value。MQA 让所有头共享同一组 K/V，GQA 则分组共享。

```
MHA:  num_kv_heads = num_query_heads      # 每个头独立 K/V
GQA:  num_kv_heads = num_query_heads / g   # 每 g 个头共享一组 K/V
MQA:  num_kv_heads = 1                     # 所有头共享一组 K/V
```

KV Cache 与 `num_kv_heads` 成正比。MQA 将 KV Cache 缩小到 1/8~1/32，GQA 则更灵活。

LLaMA-2 使用 MHA，LLaMA-3 使用 GQA（8 个 KV head），Llama-3.1-405B 也用 GQA——这直接关系到 KV Cache 的显存占用。

### 2. Prefix Caching / RadixAttention

当多个请求共享相同前缀（如系统提示词、few-shot 示例），前缀缓存可以避免重复计算。SGLang 的 RadixAttention 将前缀缓存组织成 Radix Tree，匹配效率更高。

```python
# SGLang 的前缀缓存示例
import sglang as sgl

@sgl.function
def multi_turn(system_prompt, user_input):
    sgl.system(system_prompt)  # 这个前缀会被缓存
    sgl.user(user_input)
    sgl.assistant(sgl.gen("answer", max_tokens=256))

# 多次调用，system_prompt 的 KV Cache 只计算一次
for question in questions:
    multi_turn.run(system_prompt=prompt, user_input=question)
```

### 3. 推测解码 (Speculative Decoding)

用一个小的 drafter 模型先生成多个候选 token，再用大模型验证。这样虽然 KV Cache 计算量不变，但每次验证可以生成多个 token，相当于摊薄了 KV Cache 的读写开销。

## 八、总结与最佳实践

| 场景 | 推荐策略 |
|------|---------|
| 单请求长序列（>4096 tokens） | 使用 PagedAttention + 前缀缓存，启用 INT8 KV Cache |
| 高并发短序列（<1024 tokens） | 减小 block-size 到 16，提高 GPU 显存利用率 |
| 共享系统提示词 | 启用 prefix caching 或 RadixAttention |
| 硬件 H100/H200 | 使用 FP8 KV Cache，原生支持 |
| 极致吞吐 | 考虑 MQA/GQA 架构模型 + 推测解码 |
| 显存不足 | 启用 swap-space 到 CPU 内存，但会牺牲延迟 |

**一句话总结：KV Cache 是 LLM 推理优化的核心战场。** 不理解它，你调参就像蒙眼开车。从 PagedAttention 解决内存碎片，到 INT8/FP8 量化降低显存占用，再到 MQA/GQA 架构层面的根本优化——每一步都能带来 2-10 倍的吞吐提升。

下次部署 LLM 时，别只盯着模型大小和量化位数。检查你的推理引擎对 KV Cache 的处理方式，往往能发现更大的优化空间。