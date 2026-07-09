---
layout: post
title: "RAG 进阶：Embedding 模型怎么选？"
date: 2026-06-29 19:00:00 +0800
categories: [AI]
tags: [RAG, Embedding, LLM, 向量]
---

## Embedding 是 RAG 的核心

Embedding 模型把文本转成向量，向量之间的相似度决定了检索质量。选对 Embedding 模型，RAG 就成功了一半。

## 主流 Embedding 模型对比

### 开源模型

| 模型 | 维度 | 最大输入 | 语言 | 特点 |
|------|------|---------|------|------|
| BGE-zh-v1.5 | 768 | 512 | 中文优先 | 中文检索标杆 |
| BGE-en-v1.5 | 768 | 512 | 英文优先 | 英文检索标杆 |
| E5-mistral-7b | 4096 | 4096 | 多语言 | 大模型，效果好 |
| GTE-Qwen2 | 2048 | 8192 | 中英 | 阿里出品，中文强 |
| Jina-embeddings-v3 | 1024 | 8192 | 多语言 | 支持长文本 |
| Nomic-Embed-Text-v1.5 | 768 | 8192 | 英文 | 可调维度 |

### 闭源 API

| 服务 | 模型 | 维度 | 特点 |
|------|------|------|------|
| OpenAI | text-embedding-3-small | 1536 | 性价比高 |
| OpenAI | text-embedding-3-large | 3072 | 效果最好 |
| Cohere | embed-multilingual-v3.0 | 1024 | 多语言 |
| 阿里通义 | text-embedding-v3 | 2048 | 中文最强 |

## 选型的关键指标

### 1. 检索质量（MTEB 评分）

MTEB 是业界标准的 Embedding 评测基准，包含 8 个任务：
- 分类、聚类、配对分类、重排序
- 检索、STS（语义相似度）、摘要、Reranking

查 MTEB Leaderboard，选你任务类型得分高的模型。

### 2. 语言支持

| 场景 | 推荐 |
|------|------|
| 纯中文 | BGE-zh、GTE-Qwen2、通义 |
| 纯英文 | OpenAI、Nomic、BGE-en |
| 中英混合 | BGE-zh（还行）、GTE-Qwen2（最佳） |

### 3. 输入长度

- 短文本（300字以内）：大多数模型都行
- 长文本（1000字以上）：选支持 8192 的模型，或自己分块

### 4. 嵌入维度

- 高维度（1536+）：信息更丰富，但存储和检索成本更高
- 低维度（384-768）：速度更快，存储更省，效果也能接受

## 实际操作建议

### 如果你用本地部署

```python
from sentence_transformers import SentenceTransformer

# BGE 中文（推荐）
model = SentenceTransformer('BAAI/bge-large-zh-v1.5')
embeddings = model.encode("你的文本")

# GTE-Qwen2（阿里出品，中文效果好）
model = SentenceTransformer('Alibaba-NLP/gte-Qwen2-1.5B-instruct')
```

### 如果你用 API

```python
from openai import OpenAI

client = OpenAI()
resp = client.embeddings.create(
    model="text-embedding-3-small",  # 性价比之选
    input="你的文本"
)
embedding = resp.data[0].embedding
```

## 常见误区

1. **维度越高越好？** 不一定。有些 384 维的模型在小数据集上比 1536 维的效果还好。
2. **用同一个 LLM 的 Embedding 最好？** 不一定。Embedding 和生成是两回事，分开选效果更优。
3. **中文语料用英文模型也够用？** 不够。中文 Embedding 模型在中文任务上明显优于英文模型。

> **一句话：** 中文场景首选 BGE-zh 或 GTE-Qwen2，英文场景用 OpenAI text-embedding-3-small 性价比最高。先用起来再优化，不要为了选模型陷入无限比较。