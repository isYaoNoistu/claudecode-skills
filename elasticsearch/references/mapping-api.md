# Mapping 与模板 API 参考

## 获取 Mapping

```bash
# 完整 mapping
curl -s "${ES_URL%/}/my-index/_mapping" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq '.[]|.mappings.properties'

# 指定字段的 mapping
curl -s "${ES_URL%/}/my-index/_mapping/field/message" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

## 添加字段（Put Mapping）

Mapping 只能扩展，**无法修改已有字段类型**。修改字段类型必须 Reindex。

```bash
curl -s -X PUT "${ES_URL%/}/my-index/_mapping" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "new_field": {"type": "keyword"},
      "location":  {"type": "geo_point"}
    }
  }'
```

## 常用字段类型

| 类型 | 适用场景 | 备注 |
|------|----------|------|
| `text` | 全文搜索 | 经过分析，不可聚合 |
| `keyword` | 精确匹配、过滤、聚合 | 不分析，默认最长 256 字符 |
| `long`/`integer`/`short`/`byte` | 整数 | 选择能容纳数据的最小类型 |
| `float`/`double`/`half_float`/`scaled_float` | 小数 | `scaled_float` 适合金额 |
| `date` | 时间戳 | 支持多种格式 |
| `boolean` | 布尔值 | |
| `ip` | IPv4/IPv6 地址 | 支持 CIDR 查询 |
| `geo_point` | 经纬度坐标 | 适用于地理查询 |
| `nested` | 对象数组 | 保留对象边界，避免扁平化问题 |
| `object` | JSON 对象 | 默认扁平化存储 |
| `flattened` | 任意 JSON | 单字段，类 keyword 查询 |

## 多字段（Multi-fields）

同一字段映射为多种类型：

```json
{
  "properties": {
    "title": {
      "type": "text",
      "fields": {
        "keyword": {"type": "keyword"},
        "autocomplete": {
          "type": "text",
          "analyzer": "autocomplete_analyzer"
        }
      }
    }
  }
}
```

用 `title` 全文搜索，用 `title.keyword` 聚合，用 `title.autocomplete` 自动补全。

## 动态 Mapping

控制新字段的处理方式：

```json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "known_field": {"type": "text"}
    }
  }
}
```

取值：`true`（自动映射，默认）、`runtime`（映射为运行时字段）、`false`（忽略 mapping，仍存储）、`strict`（拒绝未知字段）。

## 动态模板（Dynamic Templates）

按模式自动映射字段：

```json
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keywords": {
          "match_mapping_type": "string",
          "mapping": {"type": "keyword"}
        }
      },
      {
        "labels_as_keywords": {
          "path_match": "labels.*",
          "mapping": {"type": "keyword"}
        }
      }
    ]
  }
}
```

## 索引模板（Index Templates）

对匹配特定模式的新索引自动应用设置和 Mapping：

```bash
# 创建索引模板
curl -s -X PUT "${ES_URL%/}/_index_template/logs-template" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["logs-*"],
    "priority": 100,
    "template": {
      "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 1
      },
      "mappings": {
        "properties": {
          "@timestamp": {"type": "date"},
          "message":    {"type": "text"},
          "level":      {"type": "keyword"}
        }
      }
    }
  }'

# 列出所有模板
curl -s "${ES_URL%/}/_index_template" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq '.index_templates[].name'

# 删除模板
curl -s -X DELETE "${ES_URL%/}/_index_template/logs-template" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

## 组件模板（Component Templates）

可复用的模板构建块：

```bash
curl -s -X PUT "${ES_URL%/}/_component_template/base-mappings" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "template": {
      "mappings": {
        "properties": {
          "@timestamp":  {"type": "date"},
          "host.name":   {"type": "keyword"}
        }
      }
    }
  }'

# 在索引模板中引用组件模板
curl -s -X PUT "${ES_URL%/}/_index_template/my-template" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["my-*"],
    "composed_of": ["base-mappings"],
    "priority": 200
  }'
```

## 运行时字段（Runtime Fields）

查询时动态计算字段，无需 Reindex：

```bash
curl -s "${ES_URL%/}/my-index/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "runtime_mappings": {
      "day_of_week": {
        "type": "keyword",
        "script": "emit(doc[\"@timestamp\"].value.dayOfWeekEnum.getDisplayName(TextStyle.FULL, Locale.ROOT))"
      }
    },
    "query": {"term": {"day_of_week": "Monday"}}
  }' | jq .
```
