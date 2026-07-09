---
layout: post
title: "RAG 实战：从零搭建一个本地 RAG 系统"
date: 2026-06-29 20:00:00 +0800
categories: [AI]
tags: [RAG, LLM, 实战, 教程]
---

## 从理论到代码

前面聊了 RAG 的原理和分块策略，这篇直接把代码写出来，跑一个完整的本地 RAG 系统。

## 系统架构

```
文档 → 分块 → Embedding → 向量数据库
                                    ↓
用户提问 → 检索 → 拼接 Prompt → LLM 生成 → 回答
```

## 环境准备

```bash
pip install chromadb sentence-transformers ollama
```

## 完整代码

### 1. 文档加载与分块

```python
import os
from typing import List

def load_documents(doc_dir: str) -> List[str]:
    """加载目录下的所有文本文件"""
    chunks = []
    for fname in os.listdir(doc_dir):
        path = os.path.join(doc_dir, fname)
        if fname.endswith('.txt') or fname.endswith('.md'):
            with open(path, 'r', encoding='utf-8') as f:
                content = f.read()
            # 简单按段落分块
            chunks.extend([c.strip() for c in content.split('\n\n') if c.strip()])
    return chunks
```

### 2. 向量化并存入数据库

```python
import chromadb
from sentence_transformers import SentenceTransformer

# 加载 Embedding 模型
model = SentenceTransformer('BAAI/bge-large-zh-v1.5')

# 初始化 Chroma
client = chromadb.PersistentClient(path="./chroma_db")
collection = client.get_or_create_collection("my_docs")

# 加载文档并分块
chunks = load_documents("./docs")
if chunks:
    embeddings = model.encode(chunks).tolist()
    collection.add(
        documents=chunks,
        embeddings=embeddings,
        ids=[f"chunk_{i}" for i in range(len(chunks))]
    )
    print(f"已存入 {len(chunks)} 个文本块")
```

### 3. 检索

```python
def retrieve(query: str, top_k: int = 3) -> List[str]:
    """检索最相关的文本块"""
    query_emb = model.encode([query]).tolist()
    results = collection.query(
        query_embeddings=query_emb,
        n_results=top_k
    )
    return results['documents'][0]
```

### 4. 生成回答

```python
import ollama

def ask(query: str) -> str:
    """RAG 完整流程：检索 + 生成"""
    # 检索
    chunks = retrieve(query)
    context = "\n\n".join(chunks)

    # 拼接 Prompt
    prompt = f"""基于以下资料回答问题。如果你不知道答案，就说你不清楚。

资料：
{context}

问题：{query}

回答："""

    # LLM 生成
    resp = ollama.chat(
        model="qwen2.5:7b",
        messages=[{"role": "user", "content": prompt}]
    )
    return resp['message']['content']
```

## 使用示例

```python
# 问问题
answer = ask("RAG 和微调有什么区别？")
print(answer)
```

输出示例：
```
RAG（检索增强生成）和微调的区别主要体现在：
- RAG 不需要训练模型，通过检索外部知识来补充上下文
- 微调需要训练数据，更新模型参数
- RAG 更适合知识频繁更新的场景
- 微调更适合改变模型的行为和风格
```

## 优化方向

1. **换更好的 Embedding 模型**。BGE-zh 够用，GTE-Qwen2 效果更好。
2. **优化分块策略**。根据文档结构调整块大小和重叠。
3. **增加重排序（Reranking）**。检索出 Top-N 后用更精细的模型重排。
4. **缓存热门查询**。减少重复计算。
5. **用 HyDE 增强检索**。先生成假设回答再检索。

> **完整代码不到 50 行。** RAG 的核心思想简单，但每个环节都有优化的空间。先从能跑起来开始，再逐步调优。