---
layout: post
title: "AI Agent 全链路可观测性实战：从 prompt 到 tool call 的审计、追踪与异常检测"
date: 2026-10-17 17:30:00 +0800
categories: [AI, 安全开发]
tags: [AI Agent, 可观测性, OpenTelemetry, Langfuse, 工具调用审计, trace, prompt 注入检测, LLM 安全, 异常检测, DevSecOps]
---

## 为什么 Agent 系统比传统服务更难观测

传统微服务的可观测性三大支柱（log、metric、trace）已经成熟。但 LLM Agent 的执行链路天然有几个特征，让现有方案直接套上去会失灵：

- **决策不可重放**：同样的 prompt、不同的温度采样，输出不同，传统 trace 假设同一 span 的重放结果一致。
- **跨进程边界跨越模型**：prompt → LLM 推理（远端 GPU）→ 结构化输出 → 工具调用（本地）→ 工具输出回到 prompt，trace 上下文会断裂。
- **失败模式语义化**：HTTP 500 是显式错误，Agent 失败可能是"调通了但拿错了数据"、"调通了但解读错了"、"调通了但没调用任何工具"——这些都需要专属信号。
- **上下文污染**：工具返回内容里可能含 prompt injection，传统 span 标签无法标记"这段返回已被污染"。

今天落地一套 Agent 专属的可观测性栈：每个 prompt/工具调用/工具返回都有 trace id，结构化字段覆盖到 token 成本、模型版本、工具选择原因、注入风险评分，metric 维度支持"成本/成功率/异常模式"三维，告警直接对接 SOC。

## 一、追踪模型：四段式 trace 设计

把一次 Agent 调用拆成 4 段固定 span：

```
[user_request]            ← 用户原始输入
  └─ [llm_decision]       ← LLM 推理（决策这一步要不要调工具、调哪个）
       └─ [tool_call]      ← 工具实际执行
            └─ [tool_post] ← 返回值后处理（截断、注入扫描、注入 prompt）
  └─ [llm_answer]        ← 最终回答生成（如果没调工具）
       └─ [answer_post]
```

每段都有固定 attribute 集合，跨厂商一致：

| Attribute | 类型 | 说明 |
|-----------|------|------|
| `agent.span_type` | enum | `user_request / llm_decision / tool_call / llm_answer` |
| `agent.session_id` | string | 会话维度 |
| `agent.user_id` | string | 用户维度 |
| `agent.model` | string | `claude-sonnet-4.5` 等 |
| `agent.token_in` | int | 输入 token |
| `agent.token_out` | int | 输出 token |
| `agent.cost_usd` | float | 该 span 累计成本（cents 精度） |
| `agent.tool.name` | string | 工具名（tool_call 段专用） |
| `agent.tool.latency_ms` | int | 工具延迟 |
| `agent.tool.risk_level` | enum | `read / write / destructive / external`（基于工具分级，见上篇） |
| `agent.injection.score` | float | 工具返回内容注入风险评分 0-1 |
| `agent.injection.flagged` | bool | 是否被标记为可疑 |
| `agent.error.type` | enum | `model_error / tool_error / rate_limit / timeout / injection_blocked` |

这套 attribute 名字一旦定下来，不要轻易改——下游的 trace 检索、metric 聚合、SOC 告警规则全靠它。

## 二、用 OpenTelemetry SDK 落地四段 span

下面以 Python 为例，封装一个独立的 tracer，复用在所有 Agent 调用入口：

```python
# agent_trace.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

provider = TracerProvider(
    resource=Resource.create({"service.name": "agent-runtime", "service.version": "1.4.2"})
)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint="otel-collector:4317")))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("agent.runtime")

def span_user_request(session_id: str, user_id: str, prompt: str):
    span = tracer.start_span("user_request", attributes={
        "agent.span_type": "user_request",
        "agent.session_id": session_id,
        "agent.user_id": user_id,
        "agent.prompt.len": len(prompt),
    })
    return span

def span_llm_decision(parent, model: str, prompt: str, response: str, token_in: int, token_out: int):
    span = tracer.start_span("llm_decision", attributes={
        "agent.span_type": "llm_decision",
        "agent.model": model,
        "agent.token_in": token_in,
        "agent.token_out": token_out,
        "agent.cost_usd": cost_calc(model, token_in, token_out),
        "agent.raw_response.len": len(response),
    }, context=trace.set_span_in_context(parent))
    return span
```

关键点是 **context 传递**——`llm_decision` 必须是 `user_request` 的子 span，`tool_call` 必须是 `llm_decision` 的子 span。OTel 的 `set_span_in_context` 保证 trace id 一致，Jaeger/Tempo 端才能把整条链串起来。

## 三、工具调用的注入风险评分

工具返回内容是 Agent 系统最大的注入入口。必须对每个 `tool_call` span 的返回做实时风险评分。这里用一个轻量规则引擎做第一道闸门，深度模型做第二道：

```python
# injection_scan.py
import re
HIGH_RISK_PATTERNS = [
    (re.compile(r"ignore\s+(all\s+)?(previous|prior|above)\s+instructions?", re.I), 0.4),
    (re.compile(r"system\s*:\s*you\s+are", re.I), 0.3),
    (re.compile(r"<\s*\|.*\|\s*>", re.S), 0.2),     # ChatML/特殊 token 残留
    (re.compile(r"(curl|wget)\s+.*\|\s*(ba)?sh", re.I), 0.5),
    (re.compile(r"(/etc/passwd|/root/\.ssh|id_rsa)", re.I), 0.5),
    (re.compile(r"(exec|eval|system|popen)\s*\(", re.I), 0.3),
]

def quick_score(text: str) -> tuple[float, list[str]]:
    hits = []
    score = 0.0
    for pat, weight in HIGH_RISK_PATTERNS:
        m = pat.search(text)
        if m:
            score += weight
            hits.append(m.group(0)[:50])
    return min(score, 1.0), hits
```

把 `quick_score` 结果写进 `agent.injection.score` 和 `agent.injection.flagged`。**评分阈值 0.6 直接阻断后续 LLM 调用**，返回给用户一个固定话术："工具返回包含可能的注入指令，本次操作已中止"。0.3-0.6 区间放行但在 span 上标记 `flagged=true`，触发 SOC 告警让人工复查。

```python
def span_tool_call(parent, tool_name: str, args: dict, raw_return: str):
    risk_score, hits = quick_score(raw_return)
    blocked = risk_score >= 0.6
    span = tracer.start_span("tool_call", attributes={
        "agent.span_type": "tool_call",
        "agent.tool.name": tool_name,
        "agent.tool.risk_level": TOOL_REGISTRY[tool_name]["risk"],
        "agent.tool.latency_ms": int((time.monotonic() - t0) * 1000),
        "agent.injection.score": risk_score,
        "agent.injection.flagged": risk_score >= 0.3,
        "agent.injection.blocked": blocked,
        "agent.injection.hits": ";".join(hits)[:200],
    }, context=trace.set_span_in_context(parent))
    if blocked:
        span.set_attribute("agent.error.type", "injection_blocked")
    return span, blocked
```

## 四、Metric 维度：成本/成功率/异常模式

trace 解决"这一次发生了什么"，metric 解决"整体在出什么趋势"。Agent 系统三个核心 metric：

```yaml
# prometheus rules
- name: agent_cost_anomaly
  expr: |
    sum by (model) (rate(agent_cost_usd_total[5m]))
    > 3 * avg_over_time(sum by (model) (rate(agent_cost_usd_total[5m]))[1h] offset 1h)
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Agent 成本异常飙升"

- name: agent_injection_flagged
  expr: |
    sum by (tool_name) (rate(agent_injection_flagged_total[5m])) > 0.1
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "检测到工具返回注入尝试"

- name: agent_tool_error_rate
  expr: |
    sum by (tool_name) (rate(agent_tool_errors_total[5m]))
    /
    sum by (tool_name) (rate(agent_tool_calls_total[5m])) > 0.2
  for: 15m
```

第一条防御"成本失控"——某用户陷入死循环、某 prompt 被注入后反复调高成本工具、模型切换后单价上涨 5 倍。第二条对接 SOC——注入被检测到但未阻断，意味着规则阈值偏低或攻击者绕过。第三条传统 SRE 维度，但 Agent 上下文里要看 tool_name 而不是 endpoint，因为同一工具不同 prompt 失败原因完全不同。

## 五、会话级审计：跨 span 重建

单次 trace 不够用，**审计必须能跨 trace 重建一个 session 完整行为**。这是 OTel 自身不直接提供的，需要在存储层做：

```promql
# Tempo / Jaeger 检索：按 session_id 拉所有 span
{tempodata.span.attrs.agent.session_id = "sess-abc123"}
```

落到日志里用 Loki：

```logql
{app="agent-runtime"} | json | agent_session_id="sess-abc123"
```

在审计后台点击 session id，直接看到完整对话、工具调用顺序、每步成本、是否触发注入告警、最终是否被阻断。这是**合规审计和安全复盘的核心入口**。

## 六、几个实际踩过的坑

**坑 1：OTel SDK 默认不传 `agent.*` 自定义 attribute 给 LLM 推理厂商**。Anthropic / OpenAI 的 SDK 各自有自己的 instrumentation，但 prompt 内容、token 成本、响应延迟这些字段不会自动透传。要么手工调 `span.set_attribute()`，要么给每家厂商写自己的 instrumentation 包（社区已有 `opentelemetry-instrumentation-openai` 等，但版本落后于 SDK）。

**坑 2：trace 上下文在异步 tool 并行调用时会乱**。Agent 有时会一次返回多个 tool_use，宿主并行执行。如果用 `asyncio.gather` 同时启多个 `tool_call` span，它们都拿 `llm_decision` 作为 parent，trace 树形是对的；但日志和 metric 的"并行工具数"维度丢失。补救：手工在每次并行启动时加一个 `agent.batch_size` attribute，并在 metric 端按 batch 聚合。

**坑 3：注入评分模型的 false positive**。规则引擎容易把正常的英文文档里出现"ignore previous errors"这类技术文档误判。第一版上线两周后 false positive 高达 18%，所有命中都被 SOC 人工过滤。最终改成"0.3-0.6 静默打标不告警，只在 weekly report 里汇总"，0.6 以上才实时告警。false positive 降到 2% 以下。

**坑 4：成本 metric 没法拆解到 prompt 级别**。`agent_cost_usd_total` 是所有 span 累加，但想看"哪些 prompt 类型最贵"必须再拆一层。建议每个 `user_request` span 也带 `agent.prompt.category`（代码生成/翻译/数据分析），下游用 `sum by (prompt_category) (rate(agent_cost_usd_total[5m]))` 看分项。

**坑 5：trace 采样在 Agent 场景要慎重**。Agent 单次会话跨几十个 span，全采样存储成本爆炸。但**所有触发注入告警的会话必须全量保留**——这部分不能采样。落地做法：默认 5% 采样，但 `agent.injection.flagged=true` 的 span 强制 100% 采样（OTel tail sampling 配置）。

## 七、一套最小可落地的部署清单

如果只能今天晚上就上线，给运维同事这套最小可行版本：

1. **OTel Collector** 部署到现有 k8s 集群，导出到 Tempo + Loki + Prometheus（一栈三后端）
2. **Python tracer 库**封装一次，所有 Agent 入口强制使用（用 lint 卡死直接调 SDK 的代码）
3. **注入评分**先用上面这套 6 条正则快速跑通，深度模型第二周接
4. **三条告警**：成本飙升、注入告警、工具错误率，PrometheusRule 直接部署
5. **审计后台**：session_id 全量检索页，对接 SOC
6. **采样策略**：默认 5%，告警强制 100%，保留 30 天

第二周开始补：
- 模型版本维度（同一 prompt 不同模型行为对比）
- A/B 实验维度（灰度发布时 trace 必带 `agent.experiment_bucket`）
- 用户维度 cost cap（单个用户单日成本超阈值自动降级到小模型）

## 八、收尾

Agent 可观测性的核心不是"接 OTel"，而是**把模型决策的语义和工具执行的审计结合成一套可检索、可告警、可复盘的数据**。trace id 是骨架，attribute 是肌肉，metric 是神经，SOC 是大脑。四者齐全，Agent 系统才真正"可运维"。

下一篇会写 LLM 推理侧的 observability：vLLM/TGI 的内置 metrics + 自定义 hook，把 token 生成延迟、prefill/decode 拆分、KV cache 命中率这些 GPU 侧信号也并进同一套 trace。这块和今天内容互补——前者是 Agent 决策层观测，后者是模型推理层观测，合起来才是完整的 Agent SRE 体系。