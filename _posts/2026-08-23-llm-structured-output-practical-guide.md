---
layout: post
title: "本地 LLM 结构化输出实战：JSON模式、约束解码与Grammar引导"
date: 2026-08-23 09:00:00 +0800
categories: [AI]
tags: [llm, structured-output, json-mode, grammar, constrained-decoding, llama-cpp, vllm, ollama, local-ai, prompt-engineering]
---

## 为什么需要结构化输出

大语言模型输出的是自然语言文本，但生产环境往往需要**机器可解析的结构化数据**。无论是提取发票信息、生成 API 响应、处理日志分析结果，还是构建 AI Agent 的 tool call，最终都需要一个可靠的 JSON、YAML 或自定义格式。

直接靠 Prompt 让 LLM 输出 JSON 存在三个问题：

1. **格式不稳定** —— 偶尔会输出多余的前导/后置文本（"Here is the result:"），或者键名拼写错误
2. **解析失败** —— 嵌套结构容易漏掉闭合括号，JSON 解析器直接报错
3. **安全风险** —— 输出中可能包含意外内容，导致下游系统处理异常

解决方案不是靠"更好的 Prompt"，而是用**约束解码（Constrained Decoding）**从模型生成阶段就限制输出格式。本文覆盖四种主流方案，从最简单到最灵活，全部可用本地模型实操。

## 方案一：Prompt 引导 + 后处理（零依赖）

虽然不完美，但这是最轻量的方案——不需要任何框架支持，只需要一个清晰的提示模板和一层校验兜底。

```python
import json
import re
from typing import Optional

def extract_json_from_llm_output(text: str) -> Optional[dict]:
    """从 LLM 输出中提取第一个合法的 JSON 对象"""
    # 尝试直接解析
    text = text.strip()
    if text.startswith("```"):
        # 去掉代码块标记
        text = re.sub(r'^```(?:json)?\s*\n?', '', text)
        text = re.sub(r'\n?```\s*$', '', text)
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass
    # 用正则找第一个 { ... } 结构
    brace_pattern = r'\{[^{}]*\}'
    # 对嵌套结构需要更复杂的匹配
    depth = 0
    start = -1
    for i, ch in enumerate(text):
        if ch == '{':
            if depth == 0:
                start = i
            depth += 1
        elif ch == '}':
            depth -= 1
            if depth == 0 and start >= 0:
                try:
                    return json.loads(text[start:i+1])
                except json.JSONDecodeError:
                    start = -1
    return None

# 提示模板
SYSTEM_PROMPT = """你是一个只输出JSON的助手。
请严格按照以下JSON Schema输出，不要包含任何其他文字、代码块标记或解释。

Schema:
{
  "name": "string",
  "severity": "low|medium|high|critical",
  "issues": ["string"],
  "recommendations": ["string"]
}
"""

result = extract_json_from_llm_output(raw_output)
if result is None:
    # 兜底：返回默认结构
    result = {"error": "parse_failed", "raw": raw_output}
```

**适用场景**：快速原型、非关键任务、模型能力足够强（GPT-4 级别）。
**缺点**：仍有一定概率失败，且需要写正则做后处理。

## 方案二：Ollama JSON Mode

Ollama 从 0.3.0 版本开始内置了 JSON mode，通过 `format: json` 参数开启。它的实现原理是在解码阶段强制模型只生成合法的 JSON token 序列。

```bash
curl -X POST http://localhost:11434/api/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.1:8b",
    "messages": [
      {"role": "system", "content": "你是一个漏洞分析工具。分析给出的日志并返回JSON格式结果。"},
      {"role": "user", "content": "2026-08-23 03:14:22 ERROR [sshd] Failed password for root from 203.0.113.42 port 54321 ssh2"}
    ],
    "format": "json",
    "stream": false,
    "options": {
      "temperature": 0
    }
  }'
```

返回示例：

```json
{
  "timestamp": "2026-08-23 03:14:22",
  "source": "sshd",
  "event": "failed_login",
  "username": "root",
  "source_ip": "203.0.113.42",
  "port": 54321,
  "severity": "high"
}
```

### Ollama JSON Mode 的局限

- 它只保证输出是**合法 JSON**，不保证符合你的 Schema
- 键名和结构完全由模型决定，你无法精确控制字段
- 如果要严格 Schema，需要结合下游校验

用 Python 调用并校验：

```python
import requests
import json
from jsonschema import validate, ValidationError

SCHEMA = {
    "type": "object",
    "required": ["event", "source_ip", "severity"],
    "properties": {
        "event": {"type": "string"},
        "source_ip": {"type": "string", "format": "ipv4"},
        "severity": {"type": "string", "enum": ["low", "medium", "high", "critical"]}
    }
}

def query_ollama_structured(prompt: str, schema: dict) -> dict:
    resp = requests.post("http://localhost:11434/api/chat", json={
        "model": "llama3.1:8b",
        "messages": [
            {"role": "system", "content": f"输出JSON。必须包含这些字段: {', '.join(schema['required'])}"},
            {"role": "user", "content": prompt}
        ],
        "format": "json",
        "stream": False,
        "options": {"temperature": 0}
    })
    data = resp.json()
    try:
        result = json.loads(data["message"]["content"])
        validate(instance=result, schema=schema)
        return result
    except (json.JSONDecodeError, ValidationError) as e:
        return {"error": str(e), "raw": data["message"]["content"]}
```

## 方案三：vLLM 的 Guided Decoding

vLLM 提供了更强大的约束解码能力，支持三种引导方式：

- **JSON Schema** —— 用 Pydantic 模型或 JSON Schema 约束输出
- **Regex** —— 正则表达式约束
- **Grammar** —— 自定义 CFG 语法

### 3.1 基于 Pydantic 的 JSON 约束

```python
from pydantic import BaseModel, Field
from vllm import LLM, SamplingParams
from vllm.sampling_params import GuidedDecodingParams

class SecurityAlert(BaseModel):
    alert_id: str = Field(description="告警ID")
    title: str = Field(description="告警标题")
    severity: str = Field(description="严重程度", pattern="^(low|medium|high|critical)$")
    source_ip: str = Field(description="源IP地址")
    dest_port: int = Field(description="目标端口", ge=1, le=65535)
    description: str = Field(description="告警描述")
    recommendations: list[str] = Field(description="处置建议")

llm = LLM(model="Qwen/Qwen2.5-7B-Instruct")

prompt = """分析以下网络流量日志，生成结构化的安全告警：

时间: 2026-08-23 02:15:33
协议: TCP
源IP: 198.51.100.88:49321 -> 目标: 10.0.1.5:3389
负载: RDP 连接尝试，来自外网 IP
上下文: 该 IP 过去 24 小时已尝试连接 15 次"""

sampling_params = SamplingParams(
    temperature=0,
    max_tokens=512,
    guided_decoding=GuidedDecodingParams(
        json=SecurityAlert.model_json_schema(),
        backend="outlines"  # 或 "lm-format-enforcer"
    )
)

outputs = llm.generate([prompt], sampling_params)
result = outputs[0].outputs[0].text
print(result)
```

vLLM 的 guided decoding 在生成过程中，每一步只允许采样符合约束的 token，因此**输出 100% 符合 Schema**，不会出现格式错误。

### 3.2 正则约束

当你只需要输出一个特定格式的字符串（如 IP 地址、端口组合）时，用正则：

```python
sampling_params = SamplingParams(
    temperature=0,
    max_tokens=50,
    guided_decoding=GuidedDecodingParams(
        regex=r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}:\d{1,5}",
        backend="outlines"
    )
)
```

### 3.3 Response Grammar（CFG）

对于需要结构化嵌套但 Schema 不固定的场景，可以用 CFG 语法定义输出格式。vLLM 支持 GBNF（GGML BNF）格式和 EBNF 格式：

```python
# 定义输出语法：一个或多个命令，每个命令有操作和参数
grammar = """
root ::= command+
command ::= "{" action "," params "}"
action ::= "\"action\":" " "? "\"" ("allow" | "block" | "log" | "alert") "\""
params ::= "\"target\":" " "? stringval "," "\"reason\":" " "? stringval
stringval ::= "\"" [a-zA-Z0-9_ ]+ "\""
"""

sampling_params = SamplingParams(
    temperature=0,
    max_tokens=256,
    guided_decoding=GuidedDecodingParams(
        grammar=grammar,
        backend="outlines"
    )
)
```

## 方案四：llama.cpp 的 GBNF Grammar

如果你使用 llama.cpp（或基于它的工具如 Ollama 底层），GBNF（GGML BNF）是最强大的约束方式。它允许你定义完整的语法规则，精确控制输出格式。

### 安装与基础用法

```bash
# 使用 llama.cpp server 模式
./llama-server \
  -m qwen2.5-7b-instruct-q4_k_m.gguf \
  --grammar-file security_alert.gbnf \
  --temp 0 \
  -c 4096
```

GBNF 语法文件示例 `security_alert.gbnf`：

```gbnf
root ::= "{" ws alert_fields ws "}"
alert_fields ::= field_severity "," field_source "," field_target "," field_action
field_severity ::= "\"severity\":" ws ("\"low\"" | "\"medium\"" | "\"high\"" | "\"critical\"")
field_source ::= "\"source_ip\":" ws string
field_target ::= "\"target_ip\":" ws string
field_action ::= "\"action\":" ws ("\"allow\"" | "\"block\"" | "\"log\"")
string ::= "\"" [^"]* "\""
ws ::= [ \t\n]*
```

### 通过 Python 绑定使用 GBNF

```python
import llama_cpp
from llama_cpp.llama_grammar import LlamaGrammar

llm = llama_cpp.Llama(
    model_path="qwen2.5-7b-instruct-q4_k_m.gguf",
    n_ctx=4096,
    verbose=False
)

grammar = LlamaGrammar.from_string(r"""
root ::= "[" ws items ws "]"
items ::= item ("," ws item)*
item ::= "{" ws field_id "," ws field_name "," ws field_status ws "}"
field_id ::= "\"id\":" ws number
field_name ::= "\"name\":" ws string
field_status ::= "\"status\":" ws ("\"healthy\"" | "\"degraded\"" | "\"down\"")
number ::= [0-9]+
string ::= "\"" [^"]* "\""
ws ::= [ \t\n]*
""")

output = llm.create_completion(
    "List all services and their health status:\n- nginx: running\n- postgres: running\n- redis: degraded",
    grammar=grammar,
    max_tokens=256,
    temperature=0
)
print(output["choices"][0]["text"])
```

在 GBNF 中，你可以精确控制：
- 枚举值（`"low" | "medium" | "high"`）
- 数值范围（`[0-9]+` 或 `[1-9][0-9]?`）
- 可选字段（`field?`）
- 重复模式（`field*` 或 `field+`）
- 空白符处理（显式定义 `ws` 规则）

## 方案对比与选型

| 方案 | 约束强度 | 部署复杂度 | 灵活性 | 适用场景 |
|------|----------|------------|--------|----------|
| Prompt + 后处理 | 低 | 零部署 | 高 | 快速原型、非关键任务 |
| Ollama JSON Mode | 中 | 低 | 低 | 简单 JSON 输出 |
| vLLM Guided Decoding | 高 | 中 | 高 | 生产级 JSON/Regex 约束 |
| GBNF Grammar | 最高 | 中 | 最高 | 复杂格式、自定义 DSL |

### 选型建议

**如果你在快速验证**：Ollama JSON Mode + 后校验就够了，5 分钟上手。

**如果你在生产环境用 vLLM**：直接用 Guided Decoding 的 Pydantic 绑定，零额外运维成本。

**如果你需要生成非 JSON 格式**（如 SQL 查询、配置文件、协议数据）：GBNF 是唯一选择，它的表达能力远超 JSON Schema。

**如果你需要高吞吐 + 严格约束**：vLLM 的 outlines 后端在 batch 场景下做了优化，性能优于逐条调用 GBNF。

## 实战：构建一个日志分析 Pipeline

用 vLLM + Pydantic 构建一个完整的日志结构化提取 Pipeline：

```python
from typing import Literal
from pydantic import BaseModel, Field
from vllm import LLM, SamplingParams
from vllm.sampling_params import GuidedDecodingParams
import json

# 定义输出模型
class LogEntry(BaseModel):
    timestamp: str = Field(pattern=r"\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}")
    source: str
    event_type: Literal["auth_failure", "connection_attempt", "process_start",
                        "file_change", "network_scan", "unknown"]
    username: str | None = None
    source_ip: str | None = Field(default=None, pattern=r"\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}")
    dest_port: int | None = Field(default=None, ge=1, le=65535)
    severity: Literal["info", "low", "medium", "high", "critical"]
    raw_message: str

class LogBatch(BaseModel):
    entries: list[LogEntry]
    total_count: int
    summary: str

# 初始化模型
llm = LLM(model="Qwen/Qwen2.5-7B-Instruct")

# 批量日志
logs = [
    "Aug 23 03:14:22 server sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2",
    "Aug 23 03:15:01 server CRON[12346]: pam_unix(cron:session): session opened for user www-data",
    "Aug 23 03:16:33 server kernel: [12347.123456] nf_conntrack: table full, dropping packet",
]

for log in logs:
    prompt = f"""Extract structured information from this log entry.
Return ONLY valid JSON matching the schema.

Log: {log}"""

    sp = SamplingParams(
        temperature=0,
        max_tokens=256,
        guided_decoding=GuidedDecodingParams(
            json=LogEntry.model_json_schema(),
            backend="outlines"
        )
    )
    outputs = llm.generate([prompt], sp)
    entry = json.loads(outputs[0].outputs[0].text)
    print(f"[{entry['severity']}] {entry['event_type']}: {entry['source_ip'] or 'N/A'}")
```

运行结果：
```
[high] auth_failure: 10.0.0.99
[info] process_start: N/A
[critical] network_scan: N/A
```

## 常见陷阱

### 1. Schema 过于复杂导致生成变慢

约束解码每一步只允许合法 token，Schema 越复杂，搜索空间越小，但每步的判断开销也越大。对 1000+ token 的 Schema，生成速度可能下降 30-50%。**建议分步生成**，不要一次性要求模型输出巨大的嵌套结构。

### 2. 温度设置

结构化输出场景下，**temperature 必须设为 0 或接近 0**。高温会引入随机性，即使约束解码保证格式正确，内容质量也会下降。

### 3. 字段名冲突

如果你的 Schema 包含模型词汇表中罕见的字段名（如 `x_forwarded_for`），模型可能难以生成。建议用自然语言命名字段，在后处理中映射。

### 4. 流式输出与约束解码

Ollama 的 JSON Mode 和 vLLM 的 guided decoding 都支持流式输出，但流式场景下客户端无法逐 token 校验格式——需要等完整输出后再解析。GBNF 在流式场景下效果最好，因为语法保证每一步都是合法的部分前缀。

## 总结

结构化输出是从"能用"到"好用"的关键一步。四种方案各有适用场景：

- **Prompt + 后处理**解决不了的根本问题，靠约束解码才能根治
- **Ollama JSON Mode** 是最简单的入门方案，适合快速验证
- **vLLM Guided Decoding** 是生产环境的首选，尤其适合 API 服务场景
- **GBNF Grammar** 拥有最强表达能力，适合非 JSON 格式和复杂约束

无论选哪种方案，记住一个原则：**从模型生成阶段就限制输出格式，不要依赖下游解析来修复格式问题**。这不仅能消除解析失败，还能让模型把注意力集中在内容质量上，而非格式合规。