---
name: prometheus
description: "Prometheus 细颗粒度查询与诊断技能（原生 Prom + 适配夜莺 Categraf 指标），用「先估算、再查询、默认摘要、按需取 raw」的方式完成监控排障，避免一次性拉取高基数/大时间范围数据导致 token 爆炸。"
license: MIT
compatibility: opencode
metadata:
  audience: 运维工程师
  workflow: 监控
  side_effect: read_only
  token_policy: summary_first
  version: 1.0.0

---

# Prometheus Skill（细颗粒度 + Categraf 适配）

## 你应该如何使用本 Skill（给 AI 的路由规则）

**默认流程（强制优先级，从上到下）：**
1) **先探测指标画像**：`prom_detect_profile`（判断 node_exporter / categraf / mixed，以及优先聚合标签 ident/instance，如果用户已经明确说“这是 Categraf 指标/这是 node_exporter”，跳过 profile 探测进入下一阶段）
2) **先估算返回规模**：`prom_estimate_cardinality`（避免高基数/点位爆炸）
3) **再做查询**：优先 `prom_range_query`（`result_mode=summary`）或 `prom_instant_query`（当前值）
4) **需要解释/诊断**：把 summary 结果交给 `analyze_trend` / `promql_optimize` / `generate_promql`
5) **只有在明确需要**才用 `result_mode=raw`，且 raw 也会被截断与采样

> 重要：任何时候都不要“直接 query_range 返回全量矩阵”。先 profile → estimate → summary。

---

## 适配说明：原生 Prom + 夜莺 Categraf

你的 Prometheus 是原生 Prom，但数据里可能同时存在：
- **node_exporter 风格指标**：`node_cpu_seconds_total`、`node_memory_*`、`node_filesystem_*`、`node_network_*` 等
- **Categraf/Telegraf 风格指标**（夜莺生态常见）：`cpu_usage_idle`、`mem_used_percent`、`system_load_norm_5`、`net_drop_in` 等


本 Skill 通过 `prom_detect_profile` 自动选择 PromQL 模板：
- 若检测到 `cpu_usage_idle` / `mem_used_percent` / `system_load_norm_5` 等 ⇒ `profile=categraf_system`
- 若检测到 `node_cpu_seconds_total` / `node_memory_*` 等 ⇒ `profile=node_exporter`
- 两者都存在 ⇒ `profile=mixed`（按对象/label 决策）

**聚合标签优先级：**
- 如果 series labels 里存在 `ident` ⇒ 优先用 `ident` 聚合/筛选（更贴近夜莺对象模型）
- 否则使用 `instance`
- 若同时存在：默认 `ident`，但允许显式选择

**建议（可选但强烈推荐）：**
如果你能在 Categraf 或写入链路上增加来源标签（例如 `metrics_from="categraf"` 或 `agent="categraf"`），PromQL 会更精准、更省 token（避免扫全库）。

---

## 设计目标（减少 token 与费用）

- **Summary First**：默认只返回统计摘要（min/max/avg/p95/last/trend），不返回全量 points
- **强限制**：max_range / max_series / max_points，超过自动拒绝或降采样
- **先估算**：任何可能高基数的查询先 `prom_estimate_cardinality`
- **raw 必须“截断 + 采样”**：只允许 head/tail + 每条序列最多返回有限点位

---

## 必要配置

- `prometheus_base_url`：如 `http://localhost:9090`
- `auth`（可选）：
  - `bearer_token`
  - `basic_user` / `basic_pass`
- `timeout_seconds`：默认 10

---

## 全局默认保护参数（可覆盖）

- `max_range_seconds`: **21600**（6小时）
- `max_series`: **50**
- `max_points_per_series`: **600**
- `max_raw_points_returned_per_series`: **120**（raw 时每条最多返回点位）
- `default_result_mode`: **summary**
- `truncate_strategy`: **head_tail**
- `auto_step_policy`：
  - range ≤ 15m → step=15s
  - 15m~2h → step=30~60s
  - 2h~6h → step=120~300s
  - >6h → 拒绝（必须缩短范围或显式增大 step 并再次估算）

---

# 能力模块（Actions）

> 所有 Actions 均为 **read_only**：只调用 Prometheus HTTP API，不做任何写操作。

---

## A. 画像探测（Profile）

### 1) prom_detect_profile
**用途**：探测当前 Prom 里主机指标来自 node_exporter / categraf / mixed，并确定优先 label（ident/instance）

**输入**
- `hint_target`（可选）： `{ ident?: string, instance?: string, job?: string }`
- `time_window_seconds`（可选）：默认 3600（1h）
- `limit`（可选）：默认 50

**行为**
- 轻量探测：只做 “指标名存在性 + 少量 series labels” 的发现，不拉 points
- 输出 profile + label_strategy + 推荐 PromQL 模板集名称

**输出**
```json
{
  "profile": "node_exporter | categraf_system | mixed | unknown",
  "label_strategy": { "primary": "ident | instance", "secondary": "instance | ident" },
  "signals": {
    "found_metrics": ["cpu_usage_idle", "mem_used_percent", "node_cpu_seconds_total"],
    "found_labels": ["ident", "instance", "job"]
  },
  "next_step": "prom_estimate_cardinality"
}
````

---

## B. 规模估算（Cardinality / Risk）

### 2) prom_estimate_cardinality

**用途**：在真正查询前估算“会返回多少序列/点位”，给出风险等级与降基数建议

**输入**

* `query`（必填）：PromQL
* `start`/`end`（可选）：若提供则估算点位（否则只估算序列规模）
* `step_seconds`（可选）：`auto` 或整数
* `max_series`（可选）：默认 50
* `max_points_per_series`（可选）：默认 600

**输出**

```json
{
  "risk_level": "low | medium | high",
  "estimated_series_upper_bound": 120,
  "estimated_points_per_series": 360,
  "suggested_step_seconds": 60,
  "suggestions": [
    "为 query 增加 label 过滤（job/instance/ident/namespace/pod）",
    "用 topk() 或 sum by() 先聚合",
    "缩短时间范围或增大 step"
  ]
}
```

---

## C. 取数（Query：instant / range）

### 3) prom_instant_query

**用途**：单点查询当前值/最近值；验证 PromQL 是否可用

**输入**

* `query`（必填）
* `time`（可选，默认 now）
* `timeout_seconds`（可选，默认 10）
* `result_mode`（可选：`summary|raw`，默认 summary）
* `max_series`（可选，默认 50）

**输出（summary）**

```json
{
  "executed_at": "2026-02-26T12:00:00Z",
  "series_count": 18,
  "top_series": [
    { "labels": { "instance": "10.0.0.1:9100" }, "value": 0.92 }
  ],
  "truncated": false,
  "notes": ["如 series_count 偏大，建议加 label 过滤或 topk()"]
}
```

**输出（raw）**

* 仍会截断：最多返回 `min(max_series, 50)` 条序列；超出仅返回 head/tail + 统计

---

### 4) prom_range_query

**用途**：区间趋势查询（排障主力），默认只返回统计摘要

**输入**

* `query`（必填）
* `start`/`end`（必填）
* `step_seconds`（可选：`auto|int`，默认 auto）
* `timeout_seconds`（可选，默认 10）
* `result_mode`（可选：`summary|raw`，默认 summary）
* `max_series`（可选，默认 50）
* `max_points_per_series`（可选，默认 600）

**输出（summary）**

```json
{
  "range_seconds": 3600,
  "step_seconds": 60,
  "points_per_series": 60,
  "series_count": 12,
  "top_series": [
    {
      "labels": { "ident": "host-a" },
      "stats": { "min": 12.1, "max": 88.4, "avg": 43.2, "p50": 41.8, "p95": 81.0, "last": 55.6, "trend": "up|down|flat" }
    }
  ],
  "anomalies": [
    { "labels": { "ident": "host-a" }, "type": "spike", "at": "2026-02-26T11:33:00Z", "hint": "短时尖刺" }
  ],
  "notes": ["需要更细点位时再改为 result_mode=raw（会采样+截断）"]
}
```

**输出（raw）**

* 每条序列最多返回 `max_raw_points_returned_per_series` 点（默认 120）
* 超出范围将按 head/tail + 均匀采样裁剪


## D. 发现（Discovery：指标/标签/series）

### 5) prom_find_metrics_by_regex

**用途**：根据正则查指标名（避免全量枚举）

**输入**

* `name_regex`（必填）：如 `^node_.+_total$`
* `limit`（可选，默认 200）

**输出**

```json
{ "metric_names": ["node_cpu_seconds_total", "node_network_receive_bytes_total"], "truncated": false }
```

### 6) prom_list_label_names

**用途**：列出 label 名（限制数量）

**输入**

* `limit`（可选，默认 200）

**输出**

```json
{ "label_names": ["job","instance","ident","namespace"], "truncated": false }
```

### 7) prom_list_label_values

**用途**：列出某 label 的取值（支持 matchers 缩小范围）

**输入**

* `label_name`（必填）
* `matchers`（可选）：如 `["up{job=\"node\"}"]`
* `limit`（可选，默认 200）
* `cursor`（可选：分页游标）

**输出**

```json
{ "values": ["host-a","host-b"], "next_cursor": null, "truncated": false }
```

### 8) prom_series_lookup

**用途**：查 matcher 命中的 series（只返回 labels，不拉 points）

**输入**

* `matchers`（必填）：如 `["cpu_usage_idle{cpu=\"cpu-total\"}"]`
* `start`/`end`（可选，默认最近 1h）
* `limit`（可选，默认 200）

**输出**

```json
{ "series_labels": [ { "ident":"host-a","cpu":"cpu-total" } ], "truncated": false }
```

---

## E. 状态（Targets / Rules / Alerts / Metadata）

### 9) prom_targets_summary

**用途**：抓取目标摘要；默认只返回异常 targets

**输入**

* `state`（可选：`active|dropped|any`，默认 active）
* `only_problematic`（可选，默认 true）
* `filter_job`（可选）
* `limit`（可选，默认 50）

**输出**

```json
{
  "total_targets": 120,
  "up_targets": 118,
  "down_targets": 2,
  "down_list": [
    { "job":"node", "instance":"10.0.0.9:9100", "lastError":"context deadline exceeded", "lastScrape":"..." }
  ]
}
```

### 10) prom_alerts_summary

**用途**：当前告警；默认只返回 firing

**输入**

* `state`（可选：`firing|pending|any`，默认 firing）
* `limit`（可选，默认 50）

**输出**

```json
{ "alerts": [ { "name":"HostHighCPU", "labels": { "ident":"host-a" }, "activeAt":"..." } ] }
```

### 11) prom_rules_summary

**用途**：规则摘要；只回问题规则（firing/pending/错误）

**输入**

* `rule_type`（可选：`alerting|recording|any`，默认 any）
* `only_problematic`（可选，默认 true）
* `limit`（可选，默认 50）

**输出**

```json
{ "problem_rules": [ { "name":"HostHighCPU", "state":"firing" } ] }
```

### 12) prom_metric_metadata

**用途**：解释指标含义/type/help/unit（可用于生成面板/告警）

**输入**

* `metric`（必填）

**输出**

```json
{ "metadata": { "type":"gauge", "help":"...", "unit":"percent" } }
```

---

## F. 轻量分析（只吃 summary / 采样）

### 13) analyze_trend

**用途**：对 `prom_range_query.summary` 做趋势/异常/下一步建议（不接受全量矩阵）

**输入**

* `range_summary`（必填）：来自 `prom_range_query` 的 summary
* `baseline_summary`（可选）：对比基线

**输出**

```json
{
  "findings": ["host-a CPU p95 在过去1小时上升明显"],
  "suspected_causes": ["突发流量/线程飙升/某进程异常"],
  "next_queries": [
    "topk(5, 100 - cpu_usage_idle{cpu=\"cpu-total\", ident=\"host-a\"})",
    "rate(node_cpu_seconds_total{mode!=\"idle\", instance=\"...\"}[5m])"
  ]
}
```

### 14) generate_promql

**用途**：把“问题描述”转成 1~3 条 PromQL，并给出降基数守则

**输入**

* `question`（必填）
* `context`（可选）：`{ job, ident, instance, namespace, pod, service }`
* `profile`（可选）：来自 `prom_detect_profile`

**输出**

```json
{
  "candidates": [
    { "promql": "...", "purpose": "查看主机CPU使用率", "expected_shape": "vector|matrix" }
  ],
  "guardrails": ["必须限制对象（ident/instance）或先聚合再细化"]
}
```

### 15) promql_optimize

**用途**：优化 PromQL（降基数、降点位、提速）

**输入**

* `promql`（必填）
* `profile`（可选）

**输出**

```json
{
  "optimized_promql": ["...", "..."],
  "why": ["减少 series 数量", "减少昂贵的 label join"],
  "risk": ["聚合后会丢失细分维度"]
}
```

---

# Categraf 适配 PromQL 模板库（常用主机排障）

> 使用前先 `prom_detect_profile`，再按 profile 选模板；必要时补充 `ident/instance/job` 过滤。

## 1) CPU 使用率

* **categraf_system**

  * `100 - cpu_usage_idle{cpu="cpu-total"}`
  * （如有来源标签）`100 - cpu_usage_idle{cpu="cpu-total", metrics_from="categraf"}`
* **node_exporter**

  * `100 - avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100`

## 2) Load（归一化 5min）

* **categraf_system**

  * `system_load_norm_5`
* **node_exporter（示例）**

  * `node_load5 / count by(instance)(node_cpu_seconds_total{mode="idle"})`

## 3) 内存使用率

* **categraf_system**

  * `mem_used_percent`
* **node_exporter**

  * `(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100`

## 4) 磁盘 inode 使用率

* **categraf_system（示例写法）**

  * `disk_inodes_used / disk_inodes_total * 100`
* **node_exporter**

  * `(1 - node_filesystem_files_free / node_filesystem_files) * 100`

## 5) 网络丢包（1m 增量）

* **categraf_system**

  * `increase(net_drop_in[1m])`
  * `increase(net_drop_out[1m])`
* **node_exporter**

  * `increase(node_network_receive_drop_total[1m])`
  * `increase(node_network_transmit_drop_total[1m])`

## 6) TCP TIME_WAIT

* **categraf_system**

  * `netstat_tcp_time_wait`
* **node_exporter**

  * `node_sockstat_TCP_tw`

---

# PromQL 降基数守则（必须遵守）

1. **先限制对象**：优先加 `ident="xxx"` 或 `instance="x:port"` 或 `job="xxx"`
2. **先聚合再下钻**：`sum by(ident)` / `avg by(instance)` → 再细分 label
3. **先 topk 再展开**：`topk(10, <expr>)`
4. **对高频 counter 用 rate**：`rate(x_total[5m])` / `increase(x_total[1m])`
5. **区间查询必须控制 step**：step 不要过小；默认 auto

---

# 失败与拒绝策略（省 token + 防事故）

* 若 `end-start > max_range_seconds`：拒绝并要求缩短范围/增大 step，再 `prom_estimate_cardinality`
* 若估算 `risk_level=high`：拒绝直接查询，先返回 `suggestions`（加过滤/聚合/topk）
* 若 `series_count > max_series`：summary 仅返回 top_series，并提示如何缩小
* raw 模式：永远截断与采样，不返回全量矩阵

---

# 示例工作流（推荐）

## 示例 1：排查 host-a CPU 是否异常（混合环境）

1. `prom_detect_profile(hint_target={ident:"host-a"})`
2. `generate_promql(question="host-a CPU 使用率", context={ident:"host-a"}, profile=...)`
3. `prom_estimate_cardinality(query=候选PromQL, start=now-1h, end=now)`
4. `prom_range_query(query=..., start=..., end=..., result_mode="summary")`
5. `analyze_trend(range_summary=...)`

## 示例 2：告警 firing，先看 Prom 自身 targets 是否有问题

1. `prom_alerts_summary(state="firing")`
2. `prom_targets_summary(only_problematic=true, filter_job="node")`
3. 再回到 query/analysis 流程定位根因

---

# Prometheus API 参考（实现提示）

本 Skill 可能用到的只读端点：

* `/api/v1/query`
* `/api/v1/query_range`
* `/api/v1/series`
* `/api/v1/labels`
* `/api/v1/label/<name>/values`
* `/api/v1/targets`
* `/api/v1/rules`
* `/api/v1/alerts`
* `/api/v1/metadata`

---

## 输出约定（统一）

所有 actions 输出必须为结构化 JSON，且尽量短：

* 默认返回 `summary`
* 若返回 raw，必须包含 `truncated=true/false` 与裁剪说明
* 永远避免把全量 points 直接拼到自然语言里

```

如果你愿意，我还可以基于你们实际环境再“贴合一下”：你随便给我两条样例（复制 Prom 查询结果里的 labels 就行）：
- 一条 Categraf 指标（比如 `cpu_usage_idle` 任意一条 series labels）
- 一条 node_exporter 指标（比如 `node_cpu_seconds_total` 任意一条 series labels）

我就能把 `prom_detect_profile` 的“指标画像判定”和“聚合标签优先级”做得更准（比如你们是不是用 `ident`、还是 `host`/`hostname` 之类）。
```

