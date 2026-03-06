# Kibana API 参考

Kibana 拥有独立的 REST API，使用 `$KIBANA_URL`（不是 `$ES_URL`）。

## 认证

```bash
# Kibana URL（通常是 .kb. 域名，对应 ES 的 .es. 域名）
KIBANA_URL="https://your-deployment.kb.us-east-1.aws.elastic.cloud"

# Kibana 所有写操作必须带 kbn-xsrf 头
curl -s "${KIBANA_URL}/<endpoint>" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '<json-body>'
```

**与 ES API 的关键区别：**
- 所有 POST/PUT/DELETE 请求必须带 `kbn-xsrf: true` 头
- ES API Key 同样适用于 Kibana
- URL 规律：将 ES URL 中的 `.es.` 替换为 `.kb.`

## 健康检查

```bash
curl -s "${KIBANA_URL}/api/status" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '{name: .name, status: .status.overall.level, version: .version.number}'
```

## Spaces（工作空间）

Kibana Space 用于隔离 Dashboard、Saved Object 和数据视图。非默认 Space 需要在 API 路径前加 `/s/{spaceId}`：

```bash
# 默认 Space — 无前缀
curl -s "${KIBANA_URL}/api/data_views" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# 自定义 Space
curl -s "${KIBANA_URL}/s/my-space/api/data_views" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

## 数据视图（Data Views）

数据视图（原"索引模式"）告知 Kibana 要查询哪些索引。

```bash
# 列出数据视图
curl -s "${KIBANA_URL}/api/data_views" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '.data_view[] | {id, title}'

# 创建数据视图
curl -s -X POST "${KIBANA_URL}/api/data_views/data_view" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "data_view": {
      "id": "my-logs-view",
      "title": "logs-*",
      "timeFieldName": "@timestamp"
    },
    "override": true
  }'

# 删除数据视图
curl -s -X DELETE "${KIBANA_URL}/api/data_views/data_view/my-logs-view" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true"
```

**`override: true`** — 若已存在同 ID 的数据视图则覆盖，适合幂等脚本。

## Saved Objects（保存对象）

Saved Object 是 Kibana 的存储模型，包括 Dashboard、可视化、数据视图、搜索等。

```bash
# 查找 saved objects（Serverless 不可用）
curl -s "${KIBANA_URL}/api/saved_objects/_find?type=dashboard&per_page=100" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '.saved_objects[] | {id, title: .attributes.title}'

# 获取单个 saved object（Serverless 不可用）
curl -s "${KIBANA_URL}/api/saved_objects/dashboard/my-dashboard-id" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# 批量创建（Serverless 不可用，使用 _import 替代）
curl -s -X POST "${KIBANA_URL}/api/saved_objects/_bulk_create?overwrite=true" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '[{"type": "dashboard", "id": "my-dashboard", "attributes": {"title": "My Dashboard"}}]'
```

类型：`dashboard`、`visualization`、`lens`、`search`、`index-pattern`、`map`、`tag`

**Serverless Kibana 说明：** 在 Elastic Cloud Serverless 上，仅 `_import` 和 `_export` 可用于 Saved Object 管理。`_find`、`_bulk_create`、单个 GET 和 DELETE 接口均返回 `400 Bad Request`，请使用 `_import?overwrite=true` 进行创建和更新。

## Dashboard 导入 / 导出

```bash
# 导出 Dashboard（含依赖项）
curl -s -X POST "${KIBANA_URL}/api/saved_objects/_export" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "objects": [{"type": "dashboard", "id": "my-dashboard-id"}],
    "includeReferencesDeep": true
  }' > dashboard-export.ndjson

# 导入 Dashboard（NDJSON 格式，覆盖已有）
curl -s -X POST "${KIBANA_URL}/api/saved_objects/_import?overwrite=true" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  --form file=@dashboard-export.ndjson | jq .
```

导出格式为 NDJSON（换行符分隔 JSON），每行一个 Saved Object。

## 编程式创建 Dashboard

在 Elastic Cloud Serverless 和可移植部署场景下，使用 **by-value** Dashboard（可视化内嵌在 Dashboard 内，无需独立 Saved Object）。

### Dashboard 结构

```json
{
  "type": "dashboard",
  "id": "my-dashboard",
  "coreMigrationVersion": "8.8.0",
  "typeMigrationVersion": "10.3.0",
  "attributes": {
    "title": "我的 Dashboard",
    "description": "Dashboard 描述",
    "timeRestore": true,
    "timeTo": "now",
    "timeFrom": "now-24h",
    "refreshInterval": {"pause": false, "value": 30000},
    "panelsJSON": "[...面板数组的 JSON 字符串...]",
    "optionsJSON": "{\"useMargins\":true,\"syncColors\":false}"
  },
  "references": []
}
```

**关键：** `coreMigrationVersion` 和 `typeMigrationVersion` 是 Serverless Kibana 导入的**必填字段**。缺少时会返回 500 错误。

### 面板格式（by-value）

`panelsJSON` 中每个面板包含网格位置和内嵌的可视化配置：

```json
{
  "type": "lens",
  "gridData": {"x": 0, "y": 0, "w": 12, "h": 8, "i": "panel-1"},
  "panelIndex": "panel-1",
  "embeddableConfig": {
    "attributes": {
      "visualizationType": "lnsMetric",
      "title": "总事件数",
      "state": {
        "filters": [],
        "query": {"query": "", "language": "kuery"},
        "datasourceStates": {
          "formBased": {
            "layers": {
              "layer1": {
                "columns": {
                  "col1": {
                    "operationType": "count",
                    "label": "Count",
                    "dataType": "number",
                    "isBucketed": false,
                    "sourceField": "___records___"
                  }
                },
                "columnOrder": ["col1"]
              }
            }
          }
        },
        "visualization": {
          "layerId": "layer1",
          "layerType": "data",
          "metricAccessor": "col1"
        }
      },
      "references": [
        {"type": "index-pattern", "id": "my-data-view", "name": "indexpattern-datasource-layer-layer1"}
      ]
    }
  }
}
```

网格使用 48 列布局。常用面板尺寸：指标 `w:12 h:7`，图表 `w:24 h:14`，表格 `w:48 h:12`。

### Lens 可视化类型

| 类型 | `visualizationType` | 用途 |
|------|---------------------|------|
| 指标卡 | `lnsMetric` | 单个 KPI 数字 |
| XY 图 | `lnsXY` | 折线、柱状、面积图 |
| 饼图/环形图 | `lnsPie` | 占比分布 |
| 数据表格 | `lnsDatatable` | 表格钻取 |

### 列操作类型

| 操作 | 描述 | `isBucketed` | `sourceField` | `scale` |
|------|------|--------------|---------------|---------|
| `count` | 文档计数 | false | `"___records___"`（必填！） | `"ratio"` |
| `unique_count` | 基数 | false | 字段名 | `"ratio"` |
| `sum`/`avg`/`min`/`max` | 指标聚合 | false | 字段名 | `"ratio"` |
| `terms` | Top Values 分组 | true | 字段名 | `"ordinal"` |
| `date_histogram` | 时间桶 | true | 字段名 | `"interval"` |
| `filters` | 自定义过滤器分组 | true | — | `"ordinal"` |

**关键注意：**
- `count` 列**必须**设置 `"sourceField": "___records___"`，否则 Kibana 报 `aggValueCount requires the "field" argument`
- XY 图中所有列**应**包含 `"scale"` 字段，缺少时图表可能静默为空
- **TSDB gauge 指标**（如 `metrics.system.memory.utilization`）无法通过 Lens 面板使用 `avg`/`sum`，图表会静默为空。改用 `count` 或通过 Kibana 基础设施 UI 查看

### 部署 Dashboard 完整示例

```bash
# 生成 NDJSON（含数据视图 + Dashboard）
cat > /tmp/dashboard.ndjson << 'EOF'
{"type":"index-pattern","id":"my-logs","coreMigrationVersion":"8.8.0","attributes":{"title":"logs-*","timeFieldName":"@timestamp"}}
{"type":"dashboard","id":"my-dashboard","coreMigrationVersion":"8.8.0","typeMigrationVersion":"10.3.0","attributes":{"title":"My Dashboard","timeRestore":true,"timeFrom":"now-24h","timeTo":"now","panelsJSON":"[{\"type\":\"lens\",\"gridData\":{\"x\":0,\"y\":0,\"w\":48,\"h\":8,\"i\":\"p1\"},\"panelIndex\":\"p1\",\"embeddableConfig\":{\"attributes\":{\"visualizationType\":\"lnsMetric\",\"title\":\"Total Docs\",\"state\":{\"filters\":[],\"query\":{\"query\":\"\",\"language\":\"kuery\"},\"datasourceStates\":{\"formBased\":{\"layers\":{\"l1\":{\"columns\":{\"c1\":{\"operationType\":\"count\",\"label\":\"Count\",\"dataType\":\"number\",\"isBucketed\":false,\"sourceField\":\"___records___\"}},\"columnOrder\":[\"c1\"]}}}},\"visualization\":{\"layerId\":\"l1\",\"layerType\":\"data\",\"metricAccessor\":\"c1\"}},\"references\":[{\"type\":\"index-pattern\",\"id\":\"my-logs\",\"name\":\"indexpattern-datasource-layer-l1\"}]}}}]"},"references":[]}
EOF

# 导入至 Kibana
curl -s -X POST "${KIBANA_URL}/api/saved_objects/_import?overwrite=true" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  --form file=@/tmp/dashboard.ndjson | jq .
```

## 告警规则

```bash
# 列出规则
curl -s "${KIBANA_URL}/api/alerting/rules/_find?per_page=100" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '.data[] | {id, name: .name, enabled: .enabled}'

# 禁用规则
curl -s -X POST "${KIBANA_URL}/api/alerting/rule/{rule_id}/_disable" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true"

# 启用规则
curl -s -X POST "${KIBANA_URL}/api/alerting/rule/{rule_id}/_enable" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true"
```

## 使用提示

- **`kbn-xsrf: true`** 是每个写操作的必填头，缺少时 Kibana 返回 400
- **By-value Dashboard** 将可视化内嵌于 Dashboard，无需管理独立 Saved Object ID，Elastic Cloud Serverless 必须使用此方式
- **NDJSON 导入** 是编程部署 Dashboard 最可靠的方式
- **数据视图 ID** 必须与可视化引用中的 ID 一致，不匹配会导致"字段未找到"错误
- **Space 前缀** `/s/{spaceId}/api`，默认 Space 不需要前缀
- **Kibana URL 推导**：将 ES URL 的 `.es.` 替换为 `.kb.`
