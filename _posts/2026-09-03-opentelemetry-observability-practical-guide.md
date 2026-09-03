---
layout: post
title: "OpenTelemetry 可观测性实战指南"
date: 2026-09-03
categories: [开发]
tags: [OpenTelemetry, 可观测性, 分布式追踪, Tracing, Metrics, 监控, SRE, O11y, 微服务, Observability]
---

## 为什么需要 OpenTelemetry

可观测性（Observability）在过去几年从一个时髦词变成了落地实践，但工具碎片化的问题始终没有解决。你想做分布式追踪，选 Jaeger 还是 Zipkin？想采集指标，用 Prometheus 还是 StatsD？想收集日志，EFK 还是 Loki？更麻烦的是，每种工具都有自己的 SDK、API 和数据格式，换一个后端就得改一遍代码。

OpenTelemetry（简称 OTel）就是为了解决这个问题而生的。它不是另一个监控后端，而是一个**统一的 instrumentation 框架**——你只用一套 API 打点，数据可以发给任何兼容的后端。CNCF 的第二个毕业级可观测性项目（仅次于 Prometheus），社区支持覆盖了几乎所有主流语言和框架。

这篇文章不聊概念，直接从安装开始，一步步搭出一套完整的 OTel 可观测性管道。

## 架构概览

OpenTelemetry 的架构分为三层：

```
+------------------+       OTLP        +------------------+       Export        +------------------+
|   Instrumented   |  ==============>  |  OTel Collector  |  ===============>  |    Backend(s)     |
|   Application    |  (gRPC/HTTP)      |  (Agent/Gateway) |  (Prometheus/       |  Jaeger/Prom/     |
|   (SDK + API)    |                   |                  |   Jaeger/OTLP)     |  Loki/...         |
+------------------+                   +------------------+                    +------------------+
```

- **仪表化层**：你的应用程序通过 OTel SDK 产生 traces、metrics、logs
- **Collector 层**：接收、处理、转发遥测数据，支持过滤、采样、批处理
- **后端层**：存储和展示数据，Prometheus（指标）、Jaeger（追踪）、Loki（日志）

## 第一步：部署 OTel Collector

Collector 是整个管道的核心。你可以把它部署成 sidecar（Agent 模式）或独立集群（Gateway 模式）。

### 二进制安装

```bash
# 下载最新版
curl -LO https://github.com/open-telemetry/opentelemetry-collector-releases/releases/latest/download/otelcol-contrib_linux_amd64.tar.gz
tar -xzf otelcol-contrib_linux_amd64.tar.gz
sudo mv otelcol-contrib /usr/local/bin/otelcol

# 验证
otelcol --version
```

### 基础配置

创建一个 `otel-collector-config.yaml`：

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
  # 采样：生产环境建议只保留 10% 的追踪
  probabilistic_sampler:
    sampling_percentage: 10

exporters:
  # 控制台输出（调试用）
  debug:
    verbosity: detailed
  # Prometheus 指标暴露
  prometheus:
    endpoint: 0.0.0.0:8889
  # Jaeger 追踪
  otlp:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug, otlp]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug, prometheus]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug]
```

### 启动 Collector

```bash
otelcol --config otel-collector-config.yaml
```

启动后，Collector 会在 `4317`（gRPC）和 `4318`（HTTP）端口监听 OTLP 数据。

## 第二步：用 Python 自动仪表化

OTel 最实用的功能之一是**自动仪表化**（auto-instrumentation）——不需要改一行业务代码，就能给 Flask/Django/FastAPI 应用加上分布式追踪。

### 安装

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

`opentelemetry-bootstrap` 会自动检测你已安装的框架并安装对应的 instrumentation 包。

### 启动时注入

```bash
OTEL_SERVICE_NAME=my-python-app \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
OTEL_TRACES_SAMPLER=parentbased_traceidratio \
OTEL_TRACES_SAMPLER_ARG=0.1 \
opentelemetry-instrument \
  python app.py
```

就这么简单。现在每个 HTTP 请求都会生成一个 trace，包含从请求进入、数据库查询、外部 API 调用到返回响应的完整链路。

### 验证数据

启动 Collector 后，发一个请求到你的应用，看 Collector 控制台输出：

```
Span #0
    Trace ID       : a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6
    Parent ID      : 
    ID             : 1a2b3c4d5e6f7a8b
    Name           : GET /
    Kind           : Server
    Start time     : 2026-09-03 10:00:00.123456 +0000 UTC
    End time       : 2026-09-03 10:00:00.234567 +0000 UTC
    Status         : Unset
    Attributes:
      -> http.method: GET
      -> http.target: /
      -> http.status_code: 200
```

## 第三步：手动打点——精确控制

自动仪表化覆盖了 80% 的场景，但有些关键路径需要手动控制。比如你想追踪一个核心业务函数的执行时间和参数。

### 安装 SDK

```bash
pip install opentelemetry-sdk opentelemetry-api
```

### 代码示例

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource

# 配置 TracerProvider
resource = Resource.create({"service.name": "payment-service"})
provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(
    OTLPSpanExporter(endpoint="http://localhost:4317")
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

# 手动创建追踪
def process_payment(order_id: str, amount: float) -> bool:
    with tracer.start_as_current_span("process_payment") as span:
        span.set_attribute("order_id", order_id)
        span.set_attribute("amount", amount)
        
        # 添加事件
        span.add_event("payment.started", {"timestamp": "..."})
        
        try:
            # 调用外部支付网关
            result = call_payment_gateway(order_id, amount)
            span.set_attribute("payment.success", result)
            span.add_event("payment.completed", {"result": str(result)})
            return result
        except Exception as e:
            # 记录错误
            span.record_exception(e)
            span.set_status(trace.Status(trace.StatusCode.ERROR, str(e)))
            raise
```

### 关键点

- **`start_as_current_span`**：上下文管理器，自动处理 span 的嵌套和父子关系
- **`set_attribute`**：给 span 加标签，用于后续过滤和聚合
- **`add_event`**：在 span 时间线上标记关键事件
- **`record_exception`**：记录异常信息，包括堆栈

## 第四步：Metrics——指标采集

除了追踪，OTel 也支持指标采集。你可以用同一套 API 打点，发给 Prometheus 或任何兼容后端。

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter

# 配置 MeterProvider
reader = PeriodicExportingMetricReader(
    OTLPMetricExporter(endpoint="http://localhost:4317"),
    export_interval_millis=10000
)
provider = MeterProvider(metric_readers=[reader])
metrics.set_meter_provider(provider)

meter = metrics.get_meter(__name__)

# 计数器：记录请求总数
request_counter = meter.create_counter(
    "http.requests.total",
    description="Total number of HTTP requests",
    unit="1"
)

# 直方图：记录请求延迟
latency_histogram = meter.create_histogram(
    "http.request.duration_ms",
    description="HTTP request duration in milliseconds",
    unit="ms"
)

# 使用
def handle_request():
    import time
    start = time.time()
    
    # 处理请求...
    process()
    
    duration = (time.time() - start) * 1000
    request_counter.add(1, {"method": "GET", "path": "/api/users"})
    latency_histogram.record(duration, {"method": "GET", "status": "200"})
```

### 指标类型速查

| 类型 | 用途 | 适用场景 |
|------|------|----------|
| Counter | 只增不减 | 请求数、错误数、订单数 |
| UpDownCounter | 可增可减 | 队列长度、活跃连接数 |
| Histogram | 分桶统计 | 延迟、请求体大小 |
| Gauge | 瞬时值 | CPU 使用率、内存占用 |

## 第五步：关联日志与追踪

可观测性不止是追踪和指标，日志的关联同样重要。OTel 通过 Trace Context 把日志和 trace 关联起来，让你能从一个错误日志直接跳到对应的 trace。

### Python logging 集成

```python
import logging
from opentelemetry.instrumentation.logging import LoggingInstrumentor

# 自动注入 trace context 到日志
LoggingInstrumentor().instrument(set_logging_format=True)

logger = logging.getLogger(__name__)

# 现在所有日志都会带上 trace_id 和 span_id
logger.error("Payment failed for order %s", order_id)
# 输出: 2026-09-03 10:00:00 ERROR [trace_id=a1b2c3d4...] Payment failed for order 12345
```

### 在 Collector 中关联

Collector 可以自动从日志中提取 trace context 并关联到对应 trace，不需要额外配置。

## 第六步：生产环境部署要点

### 1. 采样策略

全量采集所有 trace 既昂贵又没必要。生产环境建议用**头采样**（head sampling）：

```yaml
processors:
  probabilistic_sampler:
    sampling_percentage: 5  # 只保留 5%
```

对于高价值请求（如支付、错误），可以用**尾采样**（tail sampling）做智能保留：

```yaml
processors:
  tail_sampling:
    policies:
      - name: errors-policy
        type: status_code
        config:
          status_codes: [ERROR, UNSET]
      - name: latency-policy
        type: latency
        config:
          threshold_ms: 1000
      - name: randomized-policy
        type: probabilistic
        config:
          sampling_percentage: 10
```

### 2. 背压保护

用 `memory_limiter` 处理器防止 Collector OOM：

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128
```

当内存超过限制时，Collector 会主动丢弃数据，而不是崩溃。

### 3. 批处理优化

```yaml
processors:
  batch:
    timeout: 5s           # 最多等 5 秒
    send_batch_size: 8192  # 攒够 8192 条再发
    send_batch_max_size: 0 # 不限制单批最大数量
```

### 4. Docker Compose 完整部署

```yaml
version: "3.8"
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus metrics

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686" # UI
      - "4317:4317"   # OTLP gRPC
    environment:
      - COLLECTOR_OTLP_ENABLED=true

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  app:
    build: .
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_SERVICE_NAME=my-app
```

## 常见问题排查

### 1. 看不到 trace 数据

```bash
# 检查 Collector 是否在监听
netstat -tlnp | grep 4317

# 测试 OTLP 端点
grpcurl -plaintext localhost:4317 list

# 检查应用日志
opentelemetry-instrument python app.py --log-level debug
```

### 2. 内存持续增长

确保 `memory_limiter` 处理器配置正确，并且 exporter 能及时消费数据。如果后端挂了，队列会积压，导致内存上涨。

### 3. 采样率不生效

检查 sampler 配置的顺序——`OTEL_TRACES_SAMPLER` 环境变量会覆盖代码中的设置。如果你在代码里配了 sampler，别忘记取消环境变量。

### 4. Span 没有正确关联

确认请求头中传递了 `traceparent`（W3C Trace Context 标准）。大多数 OTel 自动仪表化会自动处理，但如果你手动发 HTTP 请求，需要传播 context：

```python
from opentelemetry import propagate
from opentelemetry.propagate import inject

headers = {}
inject(headers)  # 自动注入 traceparent 头
requests.get("http://downstream-service/api", headers=headers)
```

## 总结

OpenTelemetry 的价值不在于它比某个工具更好，而在于它**统一了标准**。学一套 API，就能对接任何可观测性后端。对于团队来说，这意味着：

- **减少锁定**：今天用 Jaeger，明天想换 Grafana Tempo，改一行 exporter 配置就行
- **降低上手成本**：新服务只需要装好 SDK，自动仪表化就能出数据
- **统一数据模型**：追踪、指标、日志共享同一个 context，排查问题时能串起来看

如果你的项目还没有可观测性，从 OTel 开始是最有远见的选择。如果已经有 Prometheus 和 Jaeger，也可以逐步迁移到 OTel 作为统一的 instrumentation 层，后端保持不变。

下一步可以试试给生产环境的一两个服务加上自动仪表化，跑一天看看效果——你会惊讶于发现多少之前看不到的问题。