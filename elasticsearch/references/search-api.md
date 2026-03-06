# Search API 参考

## Query DSL

主要搜索端点，支持 GET 和 POST，附带 JSON 请求体。

```bash
# 基础搜索
curl -s "${ES_URL%/}/my-index/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must":   [{"match": {"message": "error"}}],
        "filter": [{"range": {"@timestamp": {"gte": "now-1h"}}}]
      }
    },
    "size": 20,
    "sort": [{"@timestamp": {"order": "desc"}}]
  }' | jq .
```

### 常用查询类型

**match** — 全文搜索（经过分析）：

```json
{"query": {"match": {"message": "connection timeout"}}}
```

**term** — 精确值匹配（不分析，适用于 keyword/枚举字段）：

```json
{"query": {"term": {"status.keyword": "error"}}}
```

**bool** — 组合查询：

```json
{"query": {"bool": {
  "must":     [...],    // AND — 参与评分
  "filter":   [...],    // AND — 不影响评分，更快
  "should":   [...],    // OR  — 无 must 时至少匹配一个
  "must_not": [...]     // NOT — 排除文档
}}}
```

**range** — 数值或日期范围：

```json
{"query": {"range": {"@timestamp": {"gte": "now-24h", "lte": "now"}}}}
```

**wildcard** — 通配符匹配（谨慎使用，查询较慢）：

```json
{"query": {"wildcard": {"host.keyword": "prod-web-*"}}}
```

**exists** — 字段存在：

```json
{"query": {"exists": {"field": "error.stack_trace"}}}
```

**multi_match** — 跨多字段搜索：

```json
{"query": {"multi_match": {"query": "timeout", "fields": ["message", "error.message", "log.message"]}}}
```

**query_string** — Lucene 语法（强大，适合用户输入场景）：

```json
{"query": {"query_string": {"query": "status:error AND service:api-gateway"}}}
```

### 排序

```json
{
  "sort": [
    {"@timestamp": {"order": "desc"}},
    {"_score":     {"order": "desc"}}
  ]
}
```

### 高亮

```json
{
  "query": {"match": {"message": "error"}},
  "highlight": {
    "fields": {"message": {}},
    "pre_tags": [">>"],
    "post_tags": ["<<"]
  }
}
```

### 分页

**基础分页**（最多 10,000 条）：

```json
{"from": 0, "size": 20}
```

**search_after**（深度分页）：

```json
{
  "size": 20,
  "sort": [{"@timestamp": "desc"}, {"_id": "asc"}],
  "search_after": ["2026-01-31T10:00:00.000Z", "doc-id-123"]
}
```

**Point-in-Time + search_after**（一致性分页，大规模数据导出推荐）：

```bash
# 开启 PIT
PIT=$(curl -s -X POST "${ES_URL%/}/my-index/_pit?keep_alive=5m" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq -r '.id')

# 带 PIT 搜索
curl -s "${ES_URL%/}/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d "{\"pit\":{\"id\":\"$PIT\",\"keep_alive\":\"5m\"},\"size\":100,\"sort\":[{\"@timestamp\":\"desc\"}]}" | jq .
```

### 文档计数

```bash
curl -s "${ES_URL%/}/my-index/_count" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{"query": {"range": {"@timestamp": {"gte": "now-1h"}}}}' | jq .count
```

## ES|QL（Elasticsearch 查询语言）

管道式查询语言，分析场景比 DSL 更简洁。适用于 Elasticsearch 8.11+。

```bash
curl -s -X POST "${ES_URL%/}/_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "FROM logs-* | WHERE @timestamp > NOW() - 1 hour AND level == \"error\" | STATS count = COUNT(*) BY service.name | SORT count DESC | LIMIT 10"
  }' | jq .
```

### ES|QL 常用模式

```sql
-- 过去一小时内错误数 Top10 服务
FROM logs-*
| WHERE @timestamp > NOW() - 1 hour AND level == "error"
| STATS error_count = COUNT(*) BY service.name
| SORT error_count DESC
| LIMIT 10

-- 按接口 P99 延迟排序
FROM traces-*
| WHERE @timestamp > NOW() - 24 hours
| STATS p99 = PERCENTILE(duration, 99), avg = AVG(duration) BY url.path
| WHERE p99 > 1000
| SORT p99 DESC

-- 每日唯一用户数
FROM logs-*
| WHERE @timestamp > NOW() - 7 days
| EVAL day = DATE_TRUNC(1 day, @timestamp)
| STATS unique_users = COUNT_DISTINCT(user.id) BY day
| SORT day
```

## EQL（事件查询语言）

基于序列的查询，适用于事件关联分析：

```bash
curl -s -X POST "${ES_URL%/}/logs-*/_eql/search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "sequence by host.name [process where event.type == \"start\" and process.name == \"cmd.exe\"] [file where event.type == \"creation\"]",
    "size": 10
  }' | jq .
```
