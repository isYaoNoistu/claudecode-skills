---
name: elasticsearch
description: >
  通过 REST API（curl）操作 Elasticsearch 和 Kibana。适用于查询、索引、管理索引、
  检查集群健康状态、编写聚合、部署 Dashboard，以及排查 Elasticsearch 问题。
  支持从环境变量加载连接配置（URL、API Key 或用户名/密码）。
  涵盖：搜索（Query DSL / ES|QL）、CRUD、索引管理、Mapping、聚合、集群健康、
  ILM、Kibana API（Dashboard、数据视图、Saved Objects）、OpenTelemetry 数据模式。
---

# Elasticsearch

所有 Elasticsearch 操作均通过 REST API 使用 `curl` 完成，无需 SDK 或客户端库。

## 环境变量配置

使用前请先设置连接配置，支持两种认证方式：

```bash
# ── 必填 ──────────────────────────────────────────────
ES_URL="https://your-cluster.es.cloud.elastic.co:443"

# ── 认证方式一：API Key（推荐，适用于 Elastic Cloud）──
ES_API_KEY="your-base64-api-key"

# ── 认证方式二：用户名/密码（适用于自建集群）──────────
ES_USER="elastic"
ES_PASS="your-password"

# ── Kibana（仅使用 Kibana API 时需要）─────────────────
KIBANA_URL="https://your-deployment.kb.us-east-1.aws.elastic.cloud"
```

> 优先使用 `ES_API_KEY`；若未设置，自动回退到 `ES_USER` + `ES_PASS`。

### 认证请求模板

```bash
# API Key 认证
curl -s "${ES_URL%/}/<endpoint>" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '<json-body>'

# 用户名/密码认证
curl -s "${ES_URL%/}/<endpoint>" \
  -u "$(printenv ES_USER):$(printenv ES_PASS)" \
  -H "Content-Type: application/json" \
  -d '<json-body>'
```

**重要说明：**
- 始终用 `$(printenv ES_API_KEY)` 而非 `$ES_API_KEY`，防止变量在 curl 中展开失败导致 401 错误
- 始终用 `${ES_URL%/}` 去除末尾斜杠，防止路径出现双斜杠（如 `//_cluster/health`）

## 快速健康检查

```bash
# 集群健康状态（green/yellow/red）——Serverless 不可用
curl -s "${ES_URL%/}/_cluster/health" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# 索引概览（Serverless 和传统部署均可用）
curl -s "${ES_URL%/}/_cat/indices?v&s=store.size:desc&h=index,health,status,docs.count,store.size" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

**Serverless 注意：** 若收到 `api_not_available_exception` 错误，说明是 Serverless 模式。以下 API 在 Serverless 中**不可用**：
- `_cluster/health`、`_cluster/settings`、`_cluster/allocation/explain`
- `_cat/nodes`、`_cat/shards`
- `_nodes/hot_threads`、`_nodes/stats`
- ILM 相关 API（`_ilm/*`）

## 搜索（Query DSL）

```bash
# 简单 match 查询
curl -s "${ES_URL%/}/my-index/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "match": { "message": "error timeout" } },
    "size": 10,
    "sort": [ { "@timestamp": { "order": "desc" } } ]
  }' | jq .

# Bool 组合查询（must + filter + must_not）
curl -s "${ES_URL%/}/my-index/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must":    [ { "match": { "message": "error" } } ],
        "filter":  [ { "range": { "@timestamp": { "gte": "now-1h" } } } ],
        "must_not":[ { "term":  { "level": "debug" } } ]
      }
    },
    "size": 20
  }' | jq .
```

完整 Query DSL 参考（term、range、wildcard、nested、multi_match 等）→ [references/query-dsl.md](references/query-dsl.md)
完整 Search API 参考（ES|QL、EQL、分页、高亮等）→ [references/search-api.md](references/search-api.md)

## 文档 CRUD

```bash
# 创建/替换文档（指定 ID）
curl -s -X PUT "${ES_URL%/}/my-index/_doc/doc-123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{ "message": "hello world", "@timestamp": "2026-01-31T12:00:00Z", "level": "info" }'

# 自动生成 ID
curl -s -X POST "${ES_URL%/}/my-index/_doc" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{ "message": "auto id doc", "level": "info" }'

# 获取文档
curl -s "${ES_URL%/}/my-index/_doc/doc-123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq '._source'

# 部分更新
curl -s -X POST "${ES_URL%/}/my-index/_update/doc-123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{ "doc": { "level": "error" } }'

# 删除文档
curl -s -X DELETE "${ES_URL%/}/my-index/_doc/doc-123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 批量操作（Bulk API，NDJSON 格式）
curl -s -X POST "${ES_URL%/}/_bulk" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/x-ndjson" \
  --data-binary @- << 'EOF'
{"index":{"_index":"my-index"}}
{"message":"bulk doc 1","level":"info","@timestamp":"2026-01-31T12:00:00Z"}
{"index":{"_index":"my-index"}}
{"message":"bulk doc 2","level":"warn","@timestamp":"2026-01-31T12:01:00Z"}
EOF
```

完整文档 API 参考（upsert、按查询更新/删除、multi-get 等）→ [references/document-api.md](references/document-api.md)

## 聚合（Aggregations）

```bash
# Terms 聚合（统计各字段值的文档数）
curl -s "${ES_URL%/}/my-index/_search?size=0" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "aggs": {
      "levels": { "terms": { "field": "level", "size": 10 } }
    }
  }' | jq '.aggregations'

# 日期直方图 + 嵌套指标
curl -s "${ES_URL%/}/my-index/_search?size=0" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "range": { "@timestamp": { "gte": "now-24h" } } },
    "aggs": {
      "over_time": {
        "date_histogram": { "field": "@timestamp", "fixed_interval": "1h" },
        "aggs": {
          "avg_count": { "avg": { "field": "count" } }
        }
      }
    }
  }' | jq '.aggregations'
```

完整聚合参考（cardinality、percentiles、composite、pipeline 聚合等）→ [references/aggregations.md](references/aggregations.md)

## 索引管理

```bash
# 创建索引（含 Mapping）
curl -s -X PUT "${ES_URL%/}/my-index" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
    "mappings": {
      "properties": {
        "message":    { "type": "text" },
        "@timestamp": { "type": "date" },
        "level":      { "type": "keyword" }
      }
    }
  }'

# 获取 Mapping
curl -s "${ES_URL%/}/my-index/_mapping" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# Reindex（修改 Mapping 或重命名索引）
curl -s -X POST "${ES_URL%/}/_reindex" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "source": { "index": "old-index" },
    "dest":   { "index": "new-index" }
  }'

# 删除索引
curl -s -X DELETE "${ES_URL%/}/my-index" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

完整索引管理参考（别名、模板、ILM、Rollover、Shrink 等）→ [references/index-api.md](references/index-api.md)
Mapping 与字段类型详细参考 → [references/mapping-api.md](references/mapping-api.md)

## ES|QL（Elasticsearch 查询语言）

适用于 Elasticsearch 8.11+，管道式查询语法，更简洁易读：

```bash
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM logs-* | WHERE level == \"error\" | STATS count = COUNT(*) BY service.name | SORT count DESC | LIMIT 10"
  }' | jq .
```

## 集群与故障排查

> **注意：** 以下大多数 API 在 **Serverless** 模式下不可用，仅适用于自建或传统 Elastic Cloud 部署。

```bash
# 集群健康（为何有未分配分片？）
curl -s "${ES_URL%/}/_cluster/allocation/explain" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{ "index": "my-index", "shard": 0, "primary": true }' | jq .

# 节点概览
curl -s "${ES_URL%/}/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 长时间任务管理
curl -s "${ES_URL%/}/_tasks?actions=*search&detailed" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

完整集群 API 参考（节点统计、分片分配、挂起任务等）→ [references/cluster-api.md](references/cluster-api.md)

## Data Streams 与 ILM

> **注意：** ILM API（`_ilm/*`）在 Serverless 下不可用。列出 Data Stream 在两者都可用。

```bash
# 列出 Data Streams
curl -s "${ES_URL%/}/_data_stream" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# 创建 ILM 策略
curl -s -X PUT "${ES_URL%/}/_ilm/policy/my-policy" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "hot":    { "actions": { "rollover": { "max_age": "7d", "max_size": "50gb" } } },
        "warm":   { "min_age": "30d", "actions": { "shrink": { "number_of_shards": 1 } } },
        "delete": { "min_age": "90d", "actions": { "delete": {} } }
      }
    }
  }'
```

## Ingest Pipeline

```bash
# 创建 Pipeline（解析日志格式）
curl -s -X PUT "${ES_URL%/}/_ingest/pipeline/my-pipeline" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "processors": [
      { "grok": { "field": "message", "patterns": ["%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}"] } },
      { "date": { "field": "timestamp", "formats": ["ISO8601"] } },
      { "remove": { "field": "timestamp" } }
    ]
  }'

# 测试 Pipeline
curl -s -X POST "${ES_URL%/}/_ingest/pipeline/my-pipeline/_simulate" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "docs": [
      { "_source": { "message": "2026-01-31T12:00:00Z ERROR something broke" } }
    ]
  }' | jq .
```

## Kibana API

Kibana 有独立的 REST API，使用 `$KIBANA_URL`（不是 `$ES_URL`）。  
完整 Kibana API 参考（Dashboard、数据视图、Saved Objects、告警规则）→ [references/kibana-api.md](references/kibana-api.md)

## OpenTelemetry 数据

OTEL 数据通过 OTLP 端点写入标准索引模式：`logs-generic.otel-default`、`traces-generic.otel-default`、`metrics-generic.otel-default`。  
完整 OTEL 查询模式（ES|QL、Trace-Log 关联、KQL 示例）→ [references/otel-data.md](references/otel-data.md)

## 使用提示

- **始终使用 `jq`** 格式化 JSON 输出，ES 响应通常非常冗长
- **`?size=0`** — 仅需聚合结果时，在搜索请求中加此参数跳过文档列表
- **`_cat` API** — `_cat/indices`、`_cat/shards`、`_cat/nodes` 提供人类可读的表格输出；加 `?v` 显示列头，`?format=json` 输出 JSON
- **大量数据导出** — 超过 10,000 条时不要用 `from/size`，改用 `search_after` + PIT
- **字段类型** — `keyword` 用于精确匹配和聚合，`text` 用于全文搜索；查询前先确认 Mapping
- **日期数学** — 索引名中 `logs-{now/d}` 会解析为今日日期，便于按时间分索引
