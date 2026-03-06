# 文档 API 参考

## 获取文档

```bash
# 按 ID 获取
curl -s "${ES_URL%/}/my-index/_doc/abc123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq '._source'

# 只返回指定字段
curl -s "${ES_URL%/}/my-index/_doc/abc123?_source_includes=timestamp,message" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq '._source'
```

## 创建/替换文档

```bash
# 指定 ID（PUT = 创建或完整替换）
curl -s -X PUT "${ES_URL%/}/my-index/_doc/abc123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "timestamp": "2026-02-01T12:00:00Z",
    "message": "hello world",
    "level": "info"
  }'

# 自动生成 ID（POST）
curl -s -X POST "${ES_URL%/}/my-index/_doc" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "timestamp": "2026-02-01T12:00:00Z",
    "message": "auto-id document"
  }'
```

## 更新文档

**部分更新**（合并字段）：

```bash
curl -s -X POST "${ES_URL%/}/my-index/_update/abc123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "doc": {
      "level": "warn",
      "updated_at": "2026-02-01T13:00:00Z"
    }
  }'
```

**脚本更新**（计算值）：

```bash
curl -s -X POST "${ES_URL%/}/my-index/_update/abc123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "script": {
      "source": "ctx._source.retry_count += 1",
      "lang": "painless"
    }
  }'
```

**Upsert**（存在则更新，不存在则创建）：

```bash
curl -s -X POST "${ES_URL%/}/my-index/_update/abc123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "doc":    {"counter": 1},
    "upsert": {"counter": 1, "created": "2026-02-01T12:00:00Z"}
  }'
```

## 删除文档

```bash
curl -s -X DELETE "${ES_URL%/}/my-index/_doc/abc123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

## 按查询删除

```bash
curl -s -X POST "${ES_URL%/}/my-index/_delete_by_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "range": {"@timestamp": {"lt": "now-90d"}}
    }
  }'
```

## 按查询更新

```bash
curl -s -X POST "${ES_URL%/}/my-index/_update_by_query" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {"term": {"status": "pending"}},
    "script": {
      "source": "ctx._source.status = \"expired\"",
      "lang": "painless"
    }
  }'
```

## Multi-Get（批量获取）

```bash
curl -s -X POST "${ES_URL%/}/_mget" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "docs": [
      {"_index": "my-index", "_id": "abc123"},
      {"_index": "my-index", "_id": "def456"}
    ]
  }' | jq .
```

## Bulk API（批量操作）

批量写入大量文档时使用，比逐条写入高效得多。使用 NDJSON 格式（换行符分隔 JSON）。

```bash
curl -s -X POST "${ES_URL%/}/_bulk" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/x-ndjson" \
  -d '
{"index": {"_index": "my-index", "_id": "1"}}
{"timestamp": "2026-02-01T12:00:00Z", "message": "first doc"}
{"index": {"_index": "my-index", "_id": "2"}}
{"timestamp": "2026-02-01T12:01:00Z", "message": "second doc"}
{"delete": {"_index": "my-index", "_id": "old-doc"}}
'
```

**从文件批量写入：**

```bash
curl -s -X POST "${ES_URL%/}/_bulk" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/x-ndjson" \
  --data-binary @bulk-data.ndjson | jq '.errors'
```

**性能建议：**
- 每次请求最优大小：5~15 MB
- 超过 10 条文档时一律使用 `_bulk` 而非逐条请求
- 检查响应中的 `errors: true` 字段，逐条排查失败项
