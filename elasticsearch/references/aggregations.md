# 聚合参考

Elasticsearch 聚合类型完整参考。所有示例均为 `POST /<index>/_search?size=0` 的请求体 JSON。

## 目录

- [指标聚合](#指标聚合)
- [桶聚合](#桶聚合)
- [管道聚合](#管道聚合)
- [常用模式](#常用模式)

## 指标聚合

### 基础指标

```json
{
  "aggs": {
    "avg_response":  { "avg":         { "field": "response_time" } },
    "max_response":  { "max":         { "field": "response_time" } },
    "min_response":  { "min":         { "field": "response_time" } },
    "sum_bytes":     { "sum":         { "field": "bytes" } },
    "total_docs":    { "value_count": { "field": "_id" } }
  }
}
```

### stats / extended_stats

一次性返回所有基础统计指标：

```json
{ "aggs": { "response_stats": { "stats":          { "field": "response_time" } } } }
{ "aggs": { "response_ext":   { "extended_stats": { "field": "response_time" } } } }
```

`extended_stats` 额外返回：标准差、方差、标准差上下界。

### cardinality（近似去重计数）

```json
{ "aggs": { "unique_users": { "cardinality": { "field": "user_id" } } } }
```

### percentiles / percentile_ranks

```json
{ "aggs": { "latency_pcts":  { "percentiles":      { "field": "response_time", "percents": [50, 90, 95, 99] } } } }
{ "aggs": { "under_200ms":   { "percentile_ranks": { "field": "response_time", "values": [200, 500, 1000] } } } }
```

### top_hits（每个桶取样本文档）

```json
{
  "aggs": {
    "by_service": {
      "terms": { "field": "service.name" },
      "aggs": {
        "latest": {
          "top_hits": { "size": 3, "sort": [ { "@timestamp": "desc" } ], "_source": ["message", "level"] }
        }
      }
    }
  }
}
```

### weighted_avg

```json
{
  "aggs": {
    "weighted_score": {
      "weighted_avg": {
        "value":  { "field": "score" },
        "weight": { "field": "importance" }
      }
    }
  }
}
```

## 桶聚合

### terms（Top N 值）

```json
{
  "aggs": {
    "top_services": {
      "terms": { "field": "service.name", "size": 20, "order": { "_count": "desc" } }
    }
  }
}
```

使用 `"missing": "unknown"` 可将缺少该字段的文档归入结果。

### date_histogram（时间直方图）

```json
{
  "aggs": {
    "over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h",
        "min_doc_count": 0,
        "extended_bounds": { "min": "now-24h", "max": "now" }
      }
    }
  }
}
```

间隔类型：`fixed_interval`（如 `30s`、`5m`、`1h`、`1d`）或 `calendar_interval`（`minute`、`hour`、`day`、`week`、`month`、`year`）。

### histogram（数值桶）

```json
{ "aggs": { "response_dist": { "histogram": { "field": "response_time", "interval": 100 } } } }
```

### range / date_range（自定义范围桶）

```json
{
  "aggs": {
    "latency_buckets": {
      "range": {
        "field": "response_time",
        "ranges": [
          { "key": "fast",   "to": 100 },
          { "key": "medium", "from": 100, "to": 500 },
          { "key": "slow",   "from": 500 }
        ]
      }
    }
  }
}
```

### filters（命名查询桶）

```json
{
  "aggs": {
    "status": {
      "filters": {
        "filters": {
          "errors":   { "term": { "level": "error" } },
          "warnings": { "term": { "level": "warn" } },
          "info":     { "term": { "level": "info" } }
        }
      }
    }
  }
}
```

### significant_terms（异常高频词）

```json
{
  "query": { "term": { "level": "error" } },
  "aggs": {
    "unusual_words": { "significant_terms": { "field": "message.keyword", "size": 10 } }
  }
}
```

### composite（分页聚合，适用于高基数场景）

```json
{
  "aggs": {
    "paginated": {
      "composite": {
        "size": 100,
        "sources": [
          { "service": { "terms": { "field": "service.name" } } },
          { "level":   { "terms": { "field": "level" } } }
        ]
      }
    }
  }
}
```

翻页时传入上次响应中的 `"after": { "service": "last-value", "level": "last-value" }`。

### multi_terms（按多字段分组）

```json
{
  "aggs": {
    "service_level": {
      "multi_terms": {
        "terms": [
          { "field": "service.name" },
          { "field": "level" }
        ],
        "size": 50
      }
    }
  }
}
```

### nested（对嵌套对象聚合）

```json
{
  "aggs": {
    "comments": {
      "nested": { "path": "comments" },
      "aggs": {
        "top_authors": { "terms": { "field": "comments.author.keyword" } }
      }
    }
  }
}
```

### sampler / diversified_sampler（限制分析文档数以提升性能）

```json
{
  "aggs": {
    "sample": {
      "sampler": { "shard_size": 200 },
      "aggs": {
        "keywords": { "significant_terms": { "field": "message.keyword" } }
      }
    }
  }
}
```

## 管道聚合

对其他聚合的输出结果进行二次计算。

### moving_fn（滑动窗口函数）

```json
{
  "aggs": {
    "over_time": {
      "date_histogram": { "field": "@timestamp", "fixed_interval": "1h" },
      "aggs": {
        "avg_response": { "avg": { "field": "response_time" } },
        "smoothed": {
          "moving_fn": { "buckets_path": "avg_response", "window": 5, "script": "MovingFunctions.unweightedAvg(values)" }
        }
      }
    }
  }
}
```

### derivative（变化率）

```json
{
  "aggs": {
    "over_time": {
      "date_histogram": { "field": "@timestamp", "fixed_interval": "1h" },
      "aggs": {
        "total":          { "sum": { "field": "count" } },
        "rate_of_change": { "derivative": { "buckets_path": "total" } }
      }
    }
  }
}
```

### cumulative_sum（累计求和）

```json
{
  "aggs": {
    "over_time": {
      "date_histogram": { "field": "@timestamp", "fixed_interval": "1d" },
      "aggs": {
        "daily_count":  { "value_count": { "field": "_id" } },
        "running_total": { "cumulative_sum": { "buckets_path": "daily_count" } }
      }
    }
  }
}
```

### bucket_script（跨兄弟指标计算）

```json
{
  "aggs": {
    "by_service": {
      "terms": { "field": "service.name" },
      "aggs": {
        "errors":  { "filter": { "term": { "level": "error" } } },
        "total":   { "value_count": { "field": "_id" } },
        "error_rate": {
          "bucket_script": {
            "buckets_path": { "err": "errors._count", "tot": "total" },
            "script": "params.err / params.tot * 100"
          }
        }
      }
    }
  }
}
```

### bucket_sort（对聚合桶排序/截取）

```json
{
  "aggs": {
    "by_service": {
      "terms": { "field": "service.name", "size": 100 },
      "aggs": {
        "avg_latency": { "avg": { "field": "response_time" } },
        "sort_by_latency": {
          "bucket_sort": { "sort": [ { "avg_latency": { "order": "desc" } } ], "size": 10 }
        }
      }
    }
  }
}
```

### bucket_selector（按条件过滤桶）

```json
{
  "aggs": {
    "by_service": {
      "terms": { "field": "service.name" },
      "aggs": {
        "avg_latency": { "avg": { "field": "response_time" } },
        "slow_only": {
          "bucket_selector": {
            "buckets_path": { "lat": "avg_latency" },
            "script": "params.lat > 500"
          }
        }
      }
    }
  }
}
```

## 常用模式

### 错误率随时间变化（SRE Dashboard）

```json
{
  "size": 0,
  "query": { "range": { "@timestamp": { "gte": "now-24h" } } },
  "aggs": {
    "over_time": {
      "date_histogram": { "field": "@timestamp", "fixed_interval": "15m" },
      "aggs": {
        "total":  { "value_count": { "field": "_id" } },
        "errors": { "filter": { "term": { "level": "error" } } },
        "error_rate": {
          "bucket_script": {
            "buckets_path": { "err": "errors._count", "tot": "total" },
            "script": "params.tot > 0 ? (params.err / params.tot * 100) : 0"
          }
        }
      }
    }
  }
}
```

### Top N 排行榜（含详情）

```json
{
  "size": 0,
  "aggs": {
    "top_endpoints": {
      "terms": { "field": "url.path.keyword", "size": 10 },
      "aggs": {
        "avg_latency": { "avg": { "field": "response_time" } },
        "p99_latency": { "percentiles": { "field": "response_time", "percents": [99] } },
        "error_count": { "filter": { "range": { "http.response.status_code": { "gte": 500 } } } }
      }
    }
  }
}
```
