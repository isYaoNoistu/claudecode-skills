# 聚合 API 快速参考

聚合在 `_search` 请求的 `aggs` 键下运行。结合 `"size": 0` 使用可跳过文档命中，仅返回聚合结果。

## 指标聚合

```bash
curl -s "${ES_URL%/}/my-index/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "avg_duration":   {"avg":        {"field": "duration_ms"}},
      "max_duration":   {"max":        {"field": "duration_ms"}},
      "p95_duration":   {"percentiles":{"field": "duration_ms", "percents": [50, 95, 99]}},
      "unique_users":   {"cardinality":{"field": "user.id"}},
      "total_bytes":    {"sum":        {"field": "response.bytes"}},
      "duration_stats": {"stats":      {"field": "duration_ms"}}
    }
  }' | jq '.aggregations'
```

## 桶聚合

**Terms**（按字段值分组）：

```bash
curl -s "${ES_URL%/}/my-index/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "by_status": {
        "terms": {"field": "status.keyword", "size": 20},
        "aggs": {
          "avg_duration": {"avg": {"field": "duration_ms"}}
        }
      }
    }
  }' | jq '.aggregations'
```

**Date histogram**（时间桶）：

```json
{
  "size": 0,
  "aggs": {
    "over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "1h"
      },
      "aggs": {
        "error_count": {
          "filter": {"term": {"level": "error"}},
          "aggs": {"count": {"value_count": {"field": "_id"}}}
        }
      }
    }
  }
}
```

**calendar_interval** 可选值：`minute`、`hour`、`day`、`week`、`month`、`quarter`、`year`  
**fixed_interval** 可选值：`30s`、`1m`、`5m`、`1h`、`1d` 等

**Range**（自定义范围桶）：

```json
{
  "size": 0,
  "aggs": {
    "latency_ranges": {
      "range": {
        "field": "duration_ms",
        "ranges": [
          {"to": 100,          "key": "fast"},
          {"from": 100, "to": 500, "key": "normal"},
          {"from": 500,        "key": "slow"}
        ]
      }
    }
  }
}
```

**Histogram**（固定宽度数值桶）：

```json
{
  "size": 0,
  "aggs": {
    "response_sizes": {
      "histogram": {"field": "response.bytes", "interval": 1000}
    }
  }
}
```

**Filter / Filters**（命名查询桶）：

```json
{
  "size": 0,
  "aggs": {
    "levels": {
      "filters": {
        "filters": {
          "errors":   {"term": {"level": "error"}},
          "warnings": {"term": {"level": "warn"}},
          "info":     {"term": {"level": "info"}}
        }
      }
    }
  }
}
```

## 嵌套聚合

聚合可以嵌套——桶内可含子聚合：

```json
{
  "size": 0,
  "aggs": {
    "by_service": {
      "terms": {"field": "service.name", "size": 10},
      "aggs": {
        "over_time": {
          "date_histogram": {"field": "@timestamp", "fixed_interval": "1h"},
          "aggs": {
            "p99": {"percentiles": {"field": "duration_ms", "percents": [99]}}
          }
        }
      }
    }
  }
}
```

## 管道聚合

对其他聚合的结果进行二次计算。

**Derivative**（变化率）：

```json
{
  "aggs": {
    "per_hour": {
      "date_histogram": {"field": "@timestamp", "fixed_interval": "1h"},
      "aggs": {
        "total":      {"sum": {"field": "bytes"}},
        "bytes_rate": {"derivative": {"buckets_path": "total"}}
      }
    }
  }
}
```

**Moving average**（滑动平均）：

```json
{
  "aggs": {
    "per_day": {
      "date_histogram": {"field": "@timestamp", "calendar_interval": "day"},
      "aggs": {
        "daily_errors": {"value_count": {"field": "_id"}},
        "smoothed": {
          "moving_avg": {"buckets_path": "daily_errors", "window": 7}
        }
      }
    }
  }
}
```

**Bucket sort**（Top-N 桶）：

```json
{
  "aggs": {
    "by_service": {
      "terms": {"field": "service.name", "size": 100},
      "aggs": {
        "error_rate": {
          "avg": {"script": {"source": "doc['level'].value == 'error' ? 1 : 0"}}
        },
        "worst_first": {
          "bucket_sort": {"sort": [{"error_rate": {"order": "desc"}}], "size": 5}
        }
      }
    }
  }
}
```

## Composite 聚合

高效翻页遍历所有桶（适用于大规模数据导出）：

```bash
curl -s "${ES_URL%/}/my-index/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 0,
    "aggs": {
      "my_composite": {
        "composite": {
          "size": 1000,
          "sources": [
            {"service": {"terms": {"field": "service.name"}}},
            {"hour":    {"date_histogram": {"field": "@timestamp", "fixed_interval": "1h"}}}
          ]
        }
      }
    }
  }' | jq .
# 使用响应中的 "after_key" 翻下一页：
# "after": {"service": "api", "hour": 1706745600000}
```
