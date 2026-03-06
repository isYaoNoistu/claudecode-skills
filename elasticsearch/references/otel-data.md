# Elasticsearch 中的 OpenTelemetry 数据

OTEL 数据通过 managed OTLP 端点写入后，会落在标准索引模式中。本参考涵盖 OTEL 日志、链路追踪和指标的查询方式。

## 索引模式

| 信号类型 | 默认索引 | 内容 |
|----------|----------|------|
| 日志 | `logs-generic.otel-default` | 应用日志、自定义事件 |
| 链路追踪 | `traces-generic.otel-default` | Span 和 Transaction |
| 指标 | `metrics-generic.otel-default` | 数值测量值 |

这些索引在数据通过 OTLP 端点到达时自动创建。

## 字段结构

OTEL 数据映射到 Elastic 字段时遵循统一结构：

```
@timestamp                          # 事件时间戳
resource.attributes.service.name    # 发出信号的服务名
resource.attributes.*               # 资源级别属性（部署环境、主机等）
attributes.*                        # 信号特定属性
```

### 日志字段

```
body                                # 日志消息 / 事件名称
severityNumber                      # 数值严重级别 (1-24)
severityText                        # INFO、WARN、ERROR 等
attributes.*                        # 日志自定义属性
resource.attributes.service.name    # 来源服务
traceId                             # 关联 Trace（在 Span 内发出时存在）
spanId                              # 关联 Span（在 Span 内发出时存在）
```

### 链路追踪字段（APM）

在 Elastic APM 中，OTEL Span 映射为：
- **Transaction** = 根 Span（无父节点）— 入口点
- **Span** = 子 Span（有父节点）— 事务内的具体操作

```
trace.id                            # 链接分布式追踪中的所有 Span
span.id                             # 当前 Span 的唯一 ID
parent.id                           # 父 Span ID（根 Span 为空）
span.name                           # 操作名称
span.duration.us                    # 耗时（微秒）
span.type                           # Span 类型（custom、db、http 等）
service.name                        # 发出 Span 的服务
```

## 查询 OTEL 日志

### ES|QL

```bash
# 查询某服务的所有日志
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM logs-generic.otel-default | WHERE resource.attributes.service.name == \"my-app\" | SORT @timestamp DESC | LIMIT 50"
  }' | jq .

# 按事件类别统计
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM logs-generic.otel-default | WHERE attributes.event.category == \"user.interaction\" | STATS count = COUNT(*) BY attributes.event.action | SORT count DESC"
  }' | jq .

# 过去一小时的错误日志
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM logs-generic.otel-default | WHERE severityText == \"ERROR\" AND @timestamp > NOW() - 1 hour | SORT @timestamp DESC | LIMIT 20"
  }' | jq .
```

### Query DSL

```bash
# 按自定义属性过滤日志
curl -s "${ES_URL%/}/logs-generic.otel-default/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "filter": [
          {"term":  {"resource.attributes.service.name": "my-app"}},
          {"range": {"@timestamp": {"gte": "now-1h"}}}
        ]
      }
    },
    "sort": [{"@timestamp": {"order": "desc"}}],
    "size": 20
  }' | jq '.hits.hits[]._source'

# 对日志消息体全文搜索
curl -s "${ES_URL%/}/logs-generic.otel-default/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must":   [{"match": {"body": "error timeout"}}],
        "filter": [{"range": {"@timestamp": {"gte": "now-24h"}}}]
      }
    },
    "size": 20
  }' | jq '.hits.hits[]._source'
```

## 查询 OTEL 链路追踪

### ES|QL

```bash
# 某服务的最慢事务
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM traces-generic.otel-default | WHERE service.name == \"my-app\" AND parent.id IS NULL | SORT span.duration.us DESC | LIMIT 10"
  }' | jq .

# 分布式追踪的所有 Span（完整调用链）
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM traces-generic.otel-default | WHERE trace.id == \"abc123def456\" | SORT @timestamp ASC"
  }' | jq .

# 按服务统计错误率
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM traces-generic.otel-default | WHERE parent.id IS NULL | STATS total = COUNT(*), errors = COUNT_IF(event.outcome == \"failure\") BY service.name | EVAL error_rate = errors / total | SORT error_rate DESC"
  }' | jq .
```

### Query DSL

```bash
# 查找某服务的根事务（root span）
curl -s "${ES_URL%/}/traces-generic.otel-default/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must":     [{"term": {"service.name": "my-app"}}],
        "must_not": [{"exists": {"field": "parent.id"}}],
        "filter":   [{"range": {"@timestamp": {"gte": "now-1h"}}}]
      }
    },
    "sort": [{"span.duration.us": {"order": "desc"}}],
    "size": 10
  }' | jq '.hits.hits[] | {name: ._source.span.name, duration_ms: (._source.span.duration.us / 1000), trace_id: ._source.trace.id}'
```

## Trace-Log 关联

在 Trace 上下文中发出的日志会自动带上 `traceId` 和 `spanId`，可用于关联追踪：

```bash
# 第一步：找到一条慢 Trace
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM traces-generic.otel-default | WHERE service.name == \"my-app\" AND span.duration.us > 5000000 | KEEP trace.id, span.name, span.duration.us | LIMIT 5"
  }' | jq .

# 第二步：获取该 Trace 的所有日志
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM logs-generic.otel-default | WHERE traceId == \"abc123def456\" | SORT @timestamp ASC"
  }' | jq .

# 第三步：获取该 Trace 的所有 Span
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM traces-generic.otel-default | WHERE trace.id == \"abc123def456\" | SORT @timestamp ASC | KEEP span.name, span.duration.us, parent.id"
  }' | jq .
```

## OTEL 数据聚合

```bash
# 按服务统计日志量（时间维度）
curl -s "${ES_URL%/}/logs-generic.otel-default/_search?size=0" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {"range": {"@timestamp": {"gte": "now-24h"}}},
    "aggs": {
      "over_time": {
        "date_histogram": {"field": "@timestamp", "fixed_interval": "1h"},
        "aggs": {
          "by_service": {
            "terms": {"field": "resource.attributes.service.name", "size": 10}
          }
        }
      }
    }
  }' | jq '.aggregations'

# 按操作统计 Span 平均耗时
curl -s "${ES_URL%/}/traces-generic.otel-default/_search?size=0" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "filter": [
          {"term":  {"service.name": "my-app"}},
          {"range": {"@timestamp": {"gte": "now-1h"}}}
        ]
      }
    },
    "aggs": {
      "by_operation": {
        "terms": {"field": "span.name", "size": 20},
        "aggs": {
          "avg_duration_ms": {"avg":         {"field": "span.duration.us"}},
          "p99_duration":    {"percentiles": {"field": "span.duration.us", "percents": [99]}}
        }
      }
    }
  }' | jq '.aggregations'
```

## KQL 模式（Kibana Discover）

在 Kibana Discover 中常用的 KQL 查询示例：

```kql
# 某服务的所有事件
resource.attributes.service.name: "my-app"

# 错误日志
severityText: "ERROR"

# 自定义属性过滤
attributes.user.id: "david"

# 字段存在性检查
attributes.frustration.type: *

# 组合过滤
resource.attributes.service.name: "my-app" AND severityText: "ERROR" AND @timestamp >= now-1h
```

## OTLP 端点配置

Elastic Cloud 提供 managed OTLP 端点（无需 APM Server）：

```bash
# 端点地址（Kibana → Add data → Applications → OpenTelemetry）
OTEL_EXPORTER_OTLP_ENDPOINT="https://your-otlp-endpoint.elastic.cloud:443"

# 认证
OTEL_EXPORTER_OTLP_HEADERS="Authorization=ApiKey YOUR_BASE64_KEY"

# 服务标识
OTEL_RESOURCE_ATTRIBUTES="service.name=my-app,deployment.environment=production"
```

OTLP 端点接受：
- `POST /v1/logs` — 日志信号
- `POST /v1/traces` — 链路追踪信号
- `POST /v1/metrics` — 指标信号

## 使用提示

- **OTEL 属性是嵌套的** — 查询 `attributes.my.field`，而非 `my.field`。不确定时先查 mapping
- **`resource.attributes` 与 `attributes`** — resource 属性描述来源（服务、主机），signal 属性描述事件本身
- **Trace 关联** — 日志上的 `traceId` 对应追踪上的 `trace.id`，注意字段名不同
- **Transaction 与 Span** — 无 `parent.id` 的根 Span 是 Elastic APM 的 Transaction，用 `parent.id IS NULL` 过滤
- **耗时单位是微秒** — `span.duration.us`，除以 1000 得毫秒
- **Data Streams** — OTEL 索引是 Data Stream，用 `_data_stream/logs-generic.otel-default` 查看健康状态
- **字段类型冲突** — 若同一属性在不同服务中类型不一致，Elasticsearch 可能拒绝写入，请保持类型一致
