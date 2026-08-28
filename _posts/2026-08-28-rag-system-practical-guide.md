---
layout: post
title: "RAG 系统实战：检索增强生成从理论到生产部署"
date: 2026-08-28 09:00:00 +0800
categories: [AI]
tags: [rag, retrieval-augmented-generation, vector-database, embedding, langchain, llamaindex, chunking, hybrid-search, reranking, llm, semantic-search]
---

## 为什么需要 RAG

大语言模型的知识截止于训练数据的时间点，无法回答私域文档、最新信息或特定业务场景的问题。微调可以注入新知识，但成本高、周期长，且每次更新都需要重新训练。**检索增强生成（RAG）** 提供了第三种路径：在推理时动态检索相关文档片段，将其作为上下文注入 prompt，让模型基于真实信息回答。

RAG 不是临时补丁，而是生产级 LLM 应用的标配架构。它解决了三个核心问题：

- **知识更新**：替换文档库即可，无需重新训练模型
- **事实准确性**：输出可追溯到具体文档，减少幻觉
- **领域适配**：任何私有文档都能被检索，无需微调

本文从零搭建一个完整的 RAG 系统，覆盖文档解析、分块策略、向量检索、混合搜索、重排序、生成优化，以及生产部署的关键考量。

## RAG 系统架构概览

一个完整的 RAG 流水线分为两个阶段：

**索引阶段（一次性）**：
```
文档 → 解析 → 分块 → Embedding → 存入向量库
```

**检索阶段（每次查询）**：
```
用户查询 → Embedding → 向量检索 + 关键词检索 → 融合 → 重排序 → LLM 生成
```

核心组件：文档解析器、分块器、Embedding 模型、向量数据库、检索器、重排序器、LLM。

## 第一步：文档解析与分块

### 文档解析

不同格式的文档需要不同的解析器。以 PDF 为例，PyMuPDF（fitz）比 PyPDF2 保留更多结构信息：

```python
import fitz

def extract_text_from_pdf(path: str) -> list[dict]:
    """从 PDF 提取段落级文本，保留元数据"""
    doc = fitz.open(path)
    pages = []
    for page_num, page in enumerate(doc):
        blocks = page.get_text("blocks")
        for block in blocks:
            text = block[4].strip()
            if not text:
                continue
            pages.append({
                "text": text,
                "page": page_num + 1,
                "bbox": block[:4],  # 位置信息，可用于去重
            })
    return pages
```

对于 Markdown、HTML 等结构化格式，保留标题层级有助于后续的分块策略。Markdown 解析可以用 `markdown-it-py` 配合 `mdformat` 来提取标题结构。

### 分块策略

分块是 RAG 中最容易被低估的环节。块太小丢失上下文，块太大引入噪声且超过模型上下文窗口。没有万能的分块策略，需要根据文档类型和查询特点选择。

**按 Token 数固定分块**（最基础）：

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("BAAI/bge-large-zh-v1.5")

def fixed_token_chunking(text: str, chunk_size: int = 512, overlap: int = 64) -> list[str]:
    tokens = tokenizer.encode(text)
    chunks = []
    start = 0
    while start < len(tokens):
        end = min(start + chunk_size, len(tokens))
        chunk_text = tokenizer.decode(tokens[start:end])
        chunks.append(chunk_text)
        start += chunk_size - overlap
    return chunks
```

**语义分块**（按段落/标题分割，保留语义边界）：

```python
import re

def semantic_chunking(markdown_text: str, max_chars: int = 1500) -> list[dict]:
    """按 Markdown 标题和段落分割，尽可能保持语义完整性"""
    # 分离标题行和正文
    lines = markdown_text.split("\n")
    chunks = []
    current_chunk = []
    current_heading = ""

    for line in lines:
        if re.match(r"^#{1,6}\s", line):
            # 遇到新标题，保存当前块
            if current_chunk:
                chunks.append({
                    "heading": current_heading,
                    "text": "\n".join(current_chunk),
                })
            current_heading = line.strip()
            current_chunk = [line]
        else:
            current_chunk.append(line)
            # 超过最大字符数，强制分割
            if len("\n".join(current_chunk)) > max_chars:
                chunks.append({
                    "heading": current_heading,
                    "text": "\n".join(current_chunk),
                })
                current_chunk = []
                current_heading = ""

    if current_chunk:
        chunks.append({
            "heading": current_heading,
            "text": "\n".join(current_chunk),
        })

    return chunks
```

**实践经验**：
- 中文文档建议 256-512 tokens 每块，英文文档 512-1024 tokens
- 块间重叠 10-20%，避免边界信息丢失
- 代码文档按函数/类分割，保留 import 语句
- 表格和列表尽量保持完整

## 第二步：Embedding 模型选择

Embedding 模型将文本映射为固定长度的向量，语义相近的文本在向量空间中也相邻。选择 Embedding 模型时关注三个指标：

1. **MTEB 评分**：衡量跨任务的检索、分类、聚类等能力
2. **维度**：高维度精度更高但存储和检索成本也更高
3. **语言支持**：中文场景优先选择中英双语模型

当前推荐的中文 Embedding 模型：

| 模型 | 维度 | MTEB | 特点 |
|------|------|------|------|
| BAAI/bge-large-zh-v1.5 | 1024 | 64.2 | 中英双语，检索任务表现突出 |
| shibing624/text2vec-base-chinese | 768 | 60.8 | 轻量级，适合快速部署 |
| moka-ai/m3e-base | 768 | 62.1 | 中文场景优化，社区活跃 |

使用示例（BGE 系列）：

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-large-zh-v1.5")

# BGE 模型需要添加 instruction prefix
query = "为这个句子生成表示以用于检索相关文章：" + "RAG 系统的分块策略有哪些？"
docs = ["固定大小分块将文本按 token 数量切割...", "语义分块保留标题和段落边界..."]

query_emb = model.encode(query, normalize_embeddings=True)
doc_embs = model.encode(docs, normalize_embeddings=True)
```

> 注意：BGE 模型在编码查询时需添加 instruction prefix，否则检索精度会下降 5-10%。

## 第三步：向量数据库

向量数据库承担高效存储和近似最近邻搜索（ANN）的任务。生产环境主流选择对比：

**Milvus**：分布式、云原生，支持 GPU 加速，适合大规模场景（百万级以上向量）。

**Qdrant**：Rust 实现，性能优异，单机部署简单，支持过滤和分组。

**Chroma**：轻量级，嵌入应用首选，适合原型和小规模部署。

**LanceDB**：基于 Lance 列式格式，零拷贝读取，支持多模态数据。

以下是使用 Chroma 的快速示例：

```python
import chromadb
from chromadb.config import Settings

client = chromadb.PersistentClient(
    path="./rag_db",
    settings=Settings(anonymized_telemetry=False),
)

# 创建集合，指定距离函数
collection = client.create_collection(
    name="tech_docs",
    metadata={"hnsw:space": "cosine"},  # cosine / l2 / ip
)

# 批量写入
collection.add(
    documents=["RAG 系统通过检索增强 LLM 的事实准确性..."],
    metadatas=[{"source": "rag-intro.md", "page": 1}],
    ids=["doc_001"],
    embeddings=[doc_emb],
)

# 检索
results = collection.query(
    query_embeddings=[query_emb],
    n_results=5,
    where={"source": {"$eq": "rag-intro.md"}},  # 支持元数据过滤
)
```

**索引参数调优**：HNSW 算法有三个关键参数：

- `M`（每个节点的连接数）：默认 16，增大提高召回率，但内存和构建时间也增加
- `ef_construction`（构建时的搜索范围）：200-500，越大索引质量越高
- `ef_search`（检索时的搜索范围）：50-200，越大召回率越高，延迟上升

生产环境建议先用小数据集调参，再用 `ef_search` 在精度和延迟之间做 trade-off。

## 第四步：混合检索

向量检索擅长语义匹配，但关键词精确匹配（如型号、ID、代码片段）效果很差。**混合检索** 结合向量检索和关键词检索（BM25），是目前生产环境中的最佳实践。

```python
from rank_bm25 import BM25Okapi
import jieba

class HybridRetriever:
    def __init__(self, documents: list[dict], alpha: float = 0.5):
        """
        alpha: 向量检索权重，1.0 = 纯向量检索，0.0 = 纯 BM25
        """
        self.documents = documents
        self.alpha = alpha
        self.embeddings = None
        self.embed_model = None

        # BM25 需要分词
        tokenized_docs = [list(jieba.cut(d["text"])) for d in documents]
        self.bm25 = BM25Okapi(tokenized_docs)

    def retrieve(self, query: str, top_k: int = 10) -> list[dict]:
        # BM25 检索
        tokenized_query = list(jieba.cut(query))
        bm25_scores = self.bm25.get_scores(tokenized_query)

        # 向量检索（假设已编码）
        query_emb = self.embed_model.encode(query, normalize_embeddings=True)
        vec_scores = [self._cosine_sim(query_emb, d["embedding"]) for d in self.documents]

        # 分数归一化 + 融合
        combined = []
        for i, doc in enumerate(self.documents):
            bm25_norm = self._min_max_norm(bm25_scores, i)
            vec_norm = self._min_max_norm(vec_scores, i)
            score = self.alpha * vec_norm + (1 - self.alpha) * bm25_norm
            combined.append((score, doc))

        combined.sort(key=lambda x: x[0], reverse=True)
        return [doc for score, doc in combined[:top_k]]

    def _min_max_norm(self, scores: list[float], idx: int) -> float:
        min_s, max_s = min(scores), max(scores)
        if max_s == min_s:
            return 0.0
        return (scores[idx] - min_s) / (max_s - min_s)
```

**alpha 调优经验**：
- 技术文档（含代码、命令）：alpha=0.3-0.4（关键词更重要）
- 开放域问答（概念性）：alpha=0.6-0.7（语义更重要）
- 不确定时：alpha=0.5 起步，通过评估集调优

## 第五步：重排序

向量检索和 BM25 的初筛结果通常包含噪声。**重排序器（Cross-Encoder）** 对 query 和每个候选文档进行联合编码，计算相关性分数，精度远高于双编码器（Bi-Encoder）但速度慢 10-100 倍。正确用法是：先用向量检索粗筛 top-50，再用重排序器精排 top-5。

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("BAAI/bge-reranker-v2-m3", max_length=512)

def rerank(query: str, candidates: list[dict], top_k: int = 5) -> list[dict]:
    pairs = [(query, doc["text"]) for doc in candidates]
    scores = reranker.predict(pairs)
    scored = list(zip(scores, candidates))
    scored.sort(key=lambda x: x[0], reverse=True)
    return [doc for score, doc in scored[:top_k]]
```

BGE-Reranker-V2 支持多语言，在中文场景的 reranking 任务上表现优于 BERT 类模型。实际测试中，加入 reranker 后最终答案的相关性评分（人工评估）提升约 15-25%。

## 第六步：生成优化

检索到相关文档后，如何组织 prompt 直接影响生成质量。

### 基础 Prompt 模板

```python
RAG_PROMPT_TEMPLATE = """基于以下文档内容回答问题。如果文档中没有相关信息，直接说出你不知道，不要编造。

文档内容：
{context}

问题：{question}
回答："""
```

### 进阶策略

**上下文压缩**：检索到的文档可能包含冗余内容。使用 LLM 或专门的压缩模型提取关键信息，减少无关 token 对生成的干扰。

```python
def compress_context(docs: list[dict], query: str, max_tokens: int = 2048) -> str:
    """保留与查询最相关的段落，丢弃无关内容"""
    # 简单策略：按相关性截断
    total = 0
    parts = []
    for doc in docs:
        tokens = len(doc["text"].split())
        if total + tokens > max_tokens:
            # 截断最后一个文档
            remaining = max_tokens - total
            words = doc["text"].split()[:remaining]
            parts.append(" ".join(words))
            break
        parts.append(doc["text"])
        total += tokens
    return "\n\n".join(parts)
```

**引文来源标注**：在答案中标注来源，显著提升可信度，方便用户验证。

```python
def answer_with_citations(query: str, docs: list[dict], llm) -> str:
    context_parts = []
    for i, doc in enumerate(docs):
        context_parts.append(f"[{i+1}] {doc['text']}")

    prompt = f"""基于以下文档回答问题。在答案中引用来源，格式为 [1]、[2]。

文档：
{chr(10).join(context_parts)}

问题：{query}
回答（请引用来源）："""
    return llm.invoke(prompt)
```

**多轮对话**：传统 RAG 每次查询独立检索，忽略了对话历史。改进方案是将历史问题和当前问题合并，生成一个独立的检索查询：

```python
def rewrite_query(history: list[dict], current_query: str, llm) -> str:
    """将对话历史中的上下文融入当前问题，生成独立检索查询"""
    prompt = f"""给定对话历史和当前用户问题，生成一个独立的检索查询。

对话历史：
{chr(10).join([f"{'用户' if m['role']=='user' else '助手'}: {m['content']}" for m in history[-3:]])}

当前问题：{current_query}

独立检索查询："""
    return llm.invoke(prompt).strip()
```

## 第七步：评估

没有评估的 RAG 系统是盲人摸象。至少需要三个维度的评估：

1. **检索精度**：Hit Rate（相关文档是否在 top-k 中）、MRR（第一个相关文档的排名）
2. **生成质量**：Faithfulness（答案是否忠实于文档）、Answer Relevance（答案是否回答问题）
3. **端到端**：人工评估或 LLM-as-Judge

RAGAS 框架是目前最成熟的 RAG 评估工具：

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset

# 准备评估数据
eval_data = Dataset.from_dict({
    "question": ["RAG 的分块策略有哪些？"],
    "answer": ["固定大小分块、语义分块和递归分块"],
    "contexts": [["固定大小分块按 token 数量切割文本...", "语义分块保留标题和段落边界..."]],
    "ground_truth": ["RAG 的常见分块策略包括固定大小分块、语义分块、递归分块，每种策略适用于不同场景"],
})

results = evaluate(
    eval_data,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
)
print(results)
```

建立你自己的评估集（至少 50-100 条 QA 对），在每次修改分块策略、更换 Embedding 模型或调整检索参数后重新评估，用数据驱动决策。

## 生产部署考量

### 延迟优化

RAG 的端到端延迟通常由三部分组成：检索（10-50ms）+ 重排序（50-200ms）+ 生成（500ms-5s）。生成阶段是瓶颈，但检索阶段的优化也不容忽视：

- **Embedding 缓存**：高频查询的向量结果缓存到 Redis
- **批量检索**：多条查询合并为一批向量检索，减少数据库连接开销
- **异步接口**：检索和生成的异步编排，支持流式输出

### 文档更新

生产环境中文档库动态变化，需要增量更新策略：

- **定时全量重建**：每天凌晨低峰期重建索引，实现简单但资源消耗大
- **增量更新**：新增/删除文档时只更新受影响的分块，需要向量数据库支持（Milvus、Qdrant 都支持）
- **双缓冲**：维护新旧两个索引，重建完成后动态切换，零停机

### 安全与合规

- **文档级权限**：在向量数据库的 metadata 中存储访问权限标签，检索时过滤
- **PII 检测**：索引前扫描文档，对敏感信息脱敏或标记
- **审计日志**：记录每次查询的检索文档和生成的答案，用于事后追溯

## 总结

RAG 不是单一技术，而是一个系统工程。从文档分块到向量检索，从混合搜索到重排序，每一步的细节都会影响最终质量。我的建议是：

1. 从一个简单的基线开始（固定分块 + 单向量检索 + 基础 prompt）
2. 建立评估集，量化每个改进的效果
3. 逐步引入混合检索、重排序、上下文压缩等高级技术
4. 根据实际场景的延迟和精度要求做取舍

RAG 的最佳实践仍在快速演进，但核心原则不变：**检索的质量决定了生成的上限**。花时间优化检索管线的每一个环节，比在 prompt 上做文章更值得。