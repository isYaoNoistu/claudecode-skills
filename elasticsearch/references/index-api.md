# 索引管理 API 参考

## 创建索引

```bash
curl -s -X PUT "${ES_URL%/}/my-new-index" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "timestamp":   {"type": "date"},
        "message":     {"type": "text"},
        "level":       {"type": "keyword"},
        "duration_ms": {"type": "float"}
      }
    }
  }'
```

## 删除索引

```bash
curl -s -X DELETE "${ES_URL%/}/my-old-index" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

**使用通配符**（危险——需先确认 `action.destructive_requires_name: false`）：

```bash
curl -s -X DELETE "${ES_URL%/}/temp-*" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

## 索引设置

**查看设置：**

```bash
curl -s "${ES_URL%/}/my-index/_settings" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq '.[]|.settings.index'
```

**更新设置**（仅限动态配置项）：

```bash
curl -s -X PUT "${ES_URL%/}/my-index/_settings" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "index": {
      "number_of_replicas": 2,
      "refresh_interval": "30s"
    }
  }'
```

常用动态设置：`number_of_replicas`、`refresh_interval`、`max_result_window`、`routing.allocation.require.*`

## 列出索引

```bash
# 人类可读，按大小降序
curl -s "${ES_URL%/}/_cat/indices?v&s=store.size:desc&h=index,health,status,docs.count,store.size" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# JSON 格式
curl -s "${ES_URL%/}/_cat/indices?format=json&s=store.size:desc" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '.[] | {index: .index, docs: .["docs.count"], size: .["store.size"]}'

# 按模式过滤
curl -s "${ES_URL%/}/_cat/indices/logs-*?v&s=store.size:desc" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

## 开启 / 关闭索引

关闭的索引不消耗资源，但无法搜索。

```bash
curl -s -X POST "${ES_URL%/}/my-index/_close" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"

curl -s -X POST "${ES_URL%/}/my-index/_open" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

## Reindex（重新索引）

在索引间复制数据（适用于 Mapping 变更、索引重命名等）：

```bash
curl -s -X POST "${ES_URL%/}/_reindex" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {"index": "old-index"},
    "dest":   {"index": "new-index"}
  }'
```

**带查询过滤：**

```bash
curl -s -X POST "${ES_URL%/}/_reindex" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "index": "logs-*",
      "query": {"range": {"@timestamp": {"gte": "2026-01-01"}}}
    },
    "dest": {"index": "logs-2026"}
  }'
```

**跨集群 Reindex：**

```bash
curl -s -X POST "${ES_URL%/}/_reindex" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "remote": {"host": "https://other-cluster:9200"},
      "index": "source-index"
    },
    "dest": {"index": "dest-index"}
  }'
```

## 别名（Aliases）

```bash
# 列出所有别名
curl -s "${ES_URL%/}/_cat/aliases?v" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 创建别名
curl -s -X POST "${ES_URL%/}/_aliases" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "actions": [
      {"add": {"index": "logs-2026.02", "alias": "logs-current"}}
    ]
  }'

# 原子切换别名（零停机蓝绿切换）
curl -s -X POST "${ES_URL%/}/_aliases" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "actions": [
      {"remove": {"index": "logs-2026.01", "alias": "logs-current"}},
      {"add":    {"index": "logs-2026.02", "alias": "logs-current"}}
    ]
  }'
```

## 索引生命周期管理（ILM）

```bash
# 获取 ILM 策略
curl -s "${ES_URL%/}/_ilm/policy/my-policy" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# 查看索引的 ILM 状态
curl -s "${ES_URL%/}/my-index/_ilm/explain" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# 重试失败的 ILM 步骤
curl -s -X POST "${ES_URL%/}/my-index/_ilm/retry" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

## Shrink / Split

```bash
# Shrink（减少分片数——索引需先设为只读）
curl -s -X PUT "${ES_URL%/}/my-index/_settings" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{"index.blocks.write": true}'

curl -s -X POST "${ES_URL%/}/my-index/_shrink/my-index-shrunk" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{"settings": {"index.number_of_shards": 1}}'

# Split（增加分片数）
curl -s -X POST "${ES_URL%/}/my-index/_split/my-index-split" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{"settings": {"index.number_of_shards": 4}}'
```

## Rollover

```bash
curl -s -X POST "${ES_URL%/}/logs-current/_rollover" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "conditions": {
      "max_age":  "7d",
      "max_size": "50gb",
      "max_docs": 10000000
    }
  }'
```

## 索引统计

```bash
# 全量统计
curl -s "${ES_URL%/}/my-index/_stats" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq '.indices["my-index"].total'

# 指定维度
curl -s "${ES_URL%/}/my-index/_stats/docs,store,indexing,search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```
