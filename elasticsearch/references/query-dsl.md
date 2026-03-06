# Query DSL 参考

完整的 Elasticsearch Query DSL 查询类型参考。所有示例假设已设置 `ES_URL` 和 `ES_API_KEY`（或 `ES_USER`/`ES_PASS`）。

## 目录

- [全文查询](#全文查询)
- [精确值查询](#精确值查询)
- [复合查询](#复合查询)
- [Nested 与 Join 查询](#nested-与-join-查询)
- [地理位置查询](#地理位置查询)
- [特殊查询](#特殊查询)
- [搜索特性](#搜索特性)

## 全文查询

### match

分析查询字符串，查找包含匹配词项的文档：

```json
{ "query": { "match": { "message": { "query": "error timeout", "operator": "and" } } } }
```

- 默认操作符为 `or`（任一词项匹配即可）
- `"operator": "and"` 要求所有词项都匹配
- `"fuzziness": "AUTO"` 开启模糊匹配（容错拼写）

### multi_match

跨多个字段搜索：

```json
{
  "query": {
    "multi_match": {
      "query": "connection refused",
      "fields": ["message", "error.message", "log.original"],
      "type": "best_fields"
    }
  }
}
```

类型：`best_fields`（默认）、`most_fields`、`cross_fields`、`phrase`、`phrase_prefix`

### match_phrase

精确短语匹配（词序固定）：

```json
{ "query": { "match_phrase": { "message": "connection refused" } } }
```

### match_phrase_prefix

类似 match_phrase，但最后一个词作前缀匹配（自动补全）：

```json
{ "query": { "match_phrase_prefix": { "message": "connect" } } }
```

### query_string

Lucene 语法，强大但输入错误会抛异常：

```json
{ "query": { "query_string": { "query": "message:error AND level:critical", "default_field": "message" } } }
```

### simple_query_string

更安全的版本，忽略无效语法，不抛异常：

```json
{ "query": { "simple_query_string": { "query": "error + timeout -debug", "fields": ["message"] } } }
```

操作符：`+`（AND）、`|`（OR）、`-`（NOT）、`"..."`（短语）、`*`（前缀）、`(...)`（分组）

## 精确值查询

这些查询直接操作原始值（不分析）。适用于 `keyword`、数值、日期、布尔类型字段。

### term

精确匹配单个值：

```json
{ "query": { "term": { "level": { "value": "error" } } } }
```

⚠️ 不要对 `text` 类型字段用 `term`——text 字段经过分析，精确匹配结果不符合预期。

### terms

匹配多个值之一（类似 SQL `IN`）：

```json
{ "query": { "terms": { "level": ["error", "critical", "fatal"] } } }
```

### range

数值或日期范围查询：

```json
{ "query": { "range": { "@timestamp": { "gte": "now-1h", "lt": "now" } } } }
{ "query": { "range": { "response_time": { "gte": 500, "lte": 5000 } } } }
```

操作符：`gt`、`gte`、`lt`、`lte`。日期数学：`now`、`now-1d`、`now/d`（取整到天）。

### exists

字段存在且不为 null：

```json
{ "query": { "exists": { "field": "error.message" } } }
```

### wildcard

通配符模式匹配（`*` 匹配任意字符，`?` 匹配单个字符）：

```json
{ "query": { "wildcard": { "hostname": { "value": "web-prod-*" } } } }
```

⚠️ 以通配符开头（如 `*error`）代价极高，生产环境慎用。

### regexp

正则表达式匹配：

```json
{ "query": { "regexp": { "path": { "value": "/api/v[0-9]+/users.*" } } } }
```

### prefix

字段值以指定前缀开头：

```json
{ "query": { "prefix": { "hostname": { "value": "web-" } } } }
```

### fuzzy

容错匹配（拼写纠错）：

```json
{ "query": { "fuzzy": { "message": { "value": "eror", "fuzziness": "AUTO" } } } }
```

### ids

按文档 ID 匹配：

```json
{ "query": { "ids": { "values": ["doc-1", "doc-2", "doc-3"] } } }
```

## 复合查询

### bool

用布尔逻辑组合多个查询：

```json
{
  "query": {
    "bool": {
      "must":     [ { "match": { "message": "error" } } ],
      "filter":   [ { "range": { "@timestamp": { "gte": "now-1h" } } } ],
      "should":   [ { "term": { "level": "critical" } } ],
      "must_not": [ { "term": { "env": "dev" } } ],
      "minimum_should_match": 1
    }
  }
}
```

- `must` — 必须匹配，参与评分
- `filter` — 必须匹配，**不参与评分**（更快，可缓存）
- `should` — 选填增分（或无 must/filter 时必须满足）
- `must_not` — 排除，不参与评分

**最佳实践：** 精确过滤放 `filter`，全文搜索放 `must`。

### boosting

降权但不排除命中负向查询的结果：

```json
{
  "query": {
    "boosting": {
      "positive": { "match": { "message": "error" } },
      "negative": { "term": { "level": "debug" } },
      "negative_boost": 0.2
    }
  }
}
```

### constant_score

对过滤器结果返回固定分数：

```json
{ "query": { "constant_score": { "filter": { "term": { "level": "error" } }, "boost": 1.0 } } }
```

### function_score

自定义评分（衰减函数、随机数、字段值因子）：

```json
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "functions": [
        { "field_value_factor": { "field": "popularity", "modifier": "log1p" } }
      ]
    }
  }
}
```

## Nested 与 Join 查询

### nested

查询嵌套对象（字段类型需为 `nested`）：

```json
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            { "match": { "comments.author": "david" } },
            { "range": { "comments.date": { "gte": "now-7d" } } }
          ]
        }
      }
    }
  }
}
```

### has_child / has_parent

父子关系查询：

```json
{ "query": { "has_child": { "type": "answer", "query": { "match": { "body": "elasticsearch" } } } } }
```

## 地理位置查询

### geo_bounding_box

```json
{
  "query": {
    "geo_bounding_box": {
      "location": {
        "top_left":     { "lat": 43.1, "lon": -79.0 },
        "bottom_right": { "lat": 42.8, "lon": -78.7 }
      }
    }
  }
}
```

### geo_distance

```json
{ "query": { "geo_distance": { "distance": "10km", "location": { "lat": 42.886, "lon": -78.878 } } } }
```

## 特殊查询

### match_all / match_none

```json
{ "query": { "match_all": {} } }
{ "query": { "match_none": {} } }
```

### script

通过 Painless 脚本自定义评分或过滤：

```json
{
  "query": {
    "script": {
      "script": {
        "source": "doc['response_time'].value > params.threshold",
        "params": { "threshold": 1000 }
      }
    }
  }
}
```

## 搜索特性

### 排序

```json
{ "sort": [ { "@timestamp": "desc" }, { "_score": "desc" } ] }
```

### _source 过滤

```json
{ "_source": ["message", "level", "@timestamp"] }
{ "_source": { "includes": ["error.*"], "excludes": ["error.stack"] } }
```

### 高亮

```json
{ "highlight": { "fields": { "message": { "pre_tags": [">>"], "post_tags": ["<<"] } } } }
```

### search_after（超 10k 分页）

```json
{
  "size": 100,
  "sort": [ { "@timestamp": "desc" }, { "_id": "asc" } ],
  "search_after": ["2026-01-31T12:00:00.000Z", "abc123"]
}
```

### collapse（按字段去重）

```json
{ "collapse": { "field": "hostname" }, "sort": [ { "@timestamp": "desc" } ] }
```

### 查询时运行字段（runtime fields）

```json
{
  "runtime_mappings": {
    "duration_ms": {
      "type": "double",
      "script": "emit(doc['end_time'].value.toInstant().toEpochMilli() - doc['start_time'].value.toInstant().toEpochMilli())"
    }
  },
  "query": { "range": { "duration_ms": { "gte": 1000 } } }
}
```
