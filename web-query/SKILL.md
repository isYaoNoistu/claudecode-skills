---
name: web-query
description: "类 Postman 的细粒度网页/API 查询与诊断技能：用「先计划、再估算、默认摘要、按需探测/按需取 raw」的方式只采集用户关心的信息，避免一次性拉全链路/拉大响应导致 token 爆炸；同时提供 DNS/TCP/TLS/HTTP 各阶段耗时与关键证据用于性能与故障定位。"
license: MIT
compatibility: opencode
metadata:
  audience: 运维工程师/接口测试工程师
  workflow: 网页/接口调试、性能分析、故障排查
  side_effect: read_only
  token_policy: summary_first
  version: 2.0.0

---

# Web-Query Skill（细颗粒度 + Summary First + 按需探测）

## 设计目标

* **少 token**：默认只返回“结论 + 关键证据摘要”，不返回整段 HTML/JSON；raw 必须显式请求且**截断+采样**。
* **更高效更细**：阶段数据采用“按需探测”，用户没要的阶段不跑；用户要证据时，返回**最短可用证据**（IP/TLS证书摘要/关键 header/JSONPath 提取字段/阶段耗时 p50/p95）。

---

## 你应该如何使用本 Skill

**强制默认流程（从上到下，优先级不可逆）：**

1. **先做意图分流**：`web_plan`（根据用户输入决定要不要跑 DNS/TCP/TLS/重试对比/内容校验/只看状态码等）
2. **先估算输出规模**：`web_estimate`（响应可能很大时：自动切换到 headers-only / body-sample / jsonpath-extract）
3. **再执行最小动作集**：只调用 plan 里必要的 actions（默认 `result_mode=summary`）
4. **需要证据再升级**：只在用户明确需要时调用 `result_mode=raw` / `web_body_sample` / `web_extract` / `web_tls_probe` 等

> 重要：任何时候不要“默认把响应体完整贴出来”。优先：摘要 → 提取 → 采样 → 最后才 raw（且 raw 也会截断）。

---

## 能力边界

* **DNS/TCP/TLS 不再默认全跑**：

  * 用户只问“接口 500/401/是否通” → 直接 HTTP 请求摘要
  * 用户问“是不是 DNS 问题/证书问题/握手慢” → 才跑对应 probe
* **阶段耗时精度**：以客户端可测时序为准（类 curl write-out）：

  * `dns`、`tcp_connect`、`tls_handshake`、`ttfb`、`download`、`total`
  * “服务器处理时间”用 `ttfb - pretransfer`近似，不强行编造“后端处理耗时”
* **连接关闭阶段**：默认不测（成本高且价值低）；用户明确要求才用 `web_conn_close_probe`

---

## 必要配置

### 基础配置（必填/可选）

* `url`（必填）
* `method`（必填）：GET/POST/PUT/DELETE/PATCH/HEAD/OPTIONS
* `timeout_ms`（可选，默认 15000）
* `connection`（可选，默认 keep-alive）：keep-alive / close
* `follow_redirects`（可选，默认 true）
* `http_version_hint`（可选）：auto / h1 / h2（仅提示，不保证）

### 参数（按需）

* `path_params` / `query_params`
* `headers`
* `body`：`type`=json|xml|form|urlencoded|raw，`content`=string/object

### 认证（按需）

* `auth.type`：none/basic/bearer/apikey/cookie
* `auth` 参数：用户名密码 / token / apikey(name,value,location) / cookies

---

## 全局默认保护参数（可覆盖）

* `default_result_mode`: **summary**
* `max_redirects`: **5**
* `max_header_kv`: **80**（响应头最多输出 80 个键值）
* `max_body_bytes_summary`: **0**（summary 默认不输出 body）
* `max_body_bytes_sample`: **4096**（sample 默认 4KB，head_tail）
* `max_body_bytes_raw`: **32768**（raw 最大 32KB，超过只给采样+统计）
* `truncate_strategy`: **head_tail**
* `redact_policy`：默认对 `Authorization`、`Cookie`、`Set-Cookie`、token/secret 字段脱敏
* `repeat_default`: **1**（多轮对比必须显式开启）

---

# Actions（细粒度模块）

> 所有 Actions 均为 **read_only**：只做探测与请求，不做写操作、不做状态修改。

---

## A. 计划与估算（先决定“跑什么”，再决定“取多少”）

### 1) web_plan

**用途**：把用户诉求转换为最小动作集（哪些阶段要测、是否要重试对比、是否要校验内容、是否要提取字段）

**输入**

```json
{
  "goal": "connectivity|status_code|auth_debug|perf_breakdown|tls_check|dns_check|content_validate|compare_runs|unknown",
  "request": { "url": "...", "method": "GET", "headers": {}, "body": null, "auth": null },
  "preferences": {
    "need_timings": true,
    "need_body": "none|sample|raw",
    "need_tls_details": false,
    "need_dns_details": false,
    "repeat": 1
  }
}
```

**输出**

```json
{
  "plan_id": "wp_20260227_xxx",
  "actions": [
    { "name": "web_http_request", "args": { "result_mode": "summary", "need_timings": true } }
  ],
  "notes": [
    "未请求证书信息，跳过 TLS 证书探测",
    "未请求 DNS 详情，跳过 DNS probe"
  ],
  "upgrade_paths": [
    { "when": "需要响应体字段", "use": ["web_extract", "web_body_sample"] },
    { "when": "怀疑证书/握手慢", "use": ["web_tls_probe"] }
  ]
}
```

---

### 2) web_estimate

**用途**：估算“这次请求可能返回多大/是否会重定向/是否应 headers-only”，避免直接拉爆

**输入**

```json
{
  "plan_id": "wp_...",
  "request": { "url": "...", "method": "GET", "headers": {}, "body": null },
  "strategy": "auto|headers_only|range_probe",
  "timeout_ms": 3000
}
```

**输出**

```json
{
  "risk_level": "low|medium|high",
  "signals": {
    "likely_large_body": true,
    "content_type_hint": "text/html",
    "redirect_possible": true
  },
  "suggestion": {
    "recommended_body_mode": "none|sample|raw",
    "recommended_actions_patch": [
      { "name": "web_http_request", "args": { "result_mode": "summary", "body_mode": "sample" } }
    ]
  }
}
```

---

## B. 基础探测（只在 plan 需要时调用）

### 3) web_dns_probe

**用途**：仅 DNS 解析（不发 HTTP），输出解析 IP 列表与耗时；可选指定 DNS 服务器

**输入**

```json
{
  "host": "api.example.com",
  "dns_server": "114.114.114.114",
  "timeout_ms": 1500
}
```

**输出**

```json
{
  "ok": true,
  "elapsed_ms": 18,
  "answers": { "A": ["93.184.216.34"], "AAAA": [] },
  "error": null
}
```

---

### 4) web_tcp_probe

**用途**：仅 TCP 连通性与 connect 耗时（不发 HTTP）

**输入**

```json
{ "host": "api.example.com", "port": 443, "timeout_ms": 2000 }
```

**输出**

```json
{
  "ok": true,
  "elapsed_ms": 32,
  "remote_ip": "93.184.216.34",
  "error": null
}
```

---

### 5) web_tls_probe（HTTPS 专属）

**用途**：仅 TLS 握手耗时 + 证书摘要（有效期/颁发者/域名匹配/SAN）+ 协商协议与加密套件

**输入**

```json
{
  "host": "api.example.com",
  "port": 443,
  "sni": "api.example.com",
  "verify_cert": true,
  "timeout_ms": 3000
}
```

**输出**

```json
{
  "ok": true,
  "elapsed_ms": 85,
  "tls": {
    "version": "TLSv1.3",
    "alpn": "h2",
    "cipher": "TLS_AES_256_GCM_SHA384"
  },
  "cert": {
    "subject_cn": "api.example.com",
    "issuer": "R3 / Let's Encrypt",
    "not_before": "2026-01-01T00:00:00Z",
    "not_after": "2026-04-01T00:00:00Z",
    "san_contains_host": true
  },
  "errors": []
}
```

---

## C. HTTP 请求（核心：默认 summary，按需 raw/采样/提取）

### 6) web_http_request

**用途**：执行一次 HTTP/HTTPS 请求并采集关键时序（dns/connect/tls/ttfb/total 等），默认只回摘要

**输入**

```json
{
  "request": {
    "url": "https://api.example.com/v1/ping?x=1",
    "method": "GET",
    "headers": { "Accept": "*/*" },
    "body": null,
    "auth": { "type": "bearer", "token": "****" }
  },
  "options": {
    "timeout_ms": 15000,
    "follow_redirects": true,
    "max_redirects": 5,
    "connection": "keep-alive",
    "need_timings": true,
    "body_mode": "none|sample|raw",
    "result_mode": "summary|raw"
  }
}
```

**输出（summary）**

```json
{
  "final_url": "https://api.example.com/v1/ping?x=1",
  "http": { "version": "h2", "status": 200, "status_text": "OK" },
  "timings_ms": {
    "dns": 12,
    "tcp_connect": 25,
    "tls_handshake": 60,
    "ttfb": 130,
    "download": 18,
    "total": 245
  },
  "network": {
    "remote_ip": "93.184.216.34",
    "remote_port": 443,
    "reused_connection": false
  },
  "sizes": { "request_bytes": 512, "response_header_bytes": 820, "response_body_bytes": 1024 },
  "headers_summary": {
    "content_type": "application/json",
    "cache_control": "no-cache",
    "server": "nginx"
  },
  "truncated": false,
  "notes": ["默认未返回响应体；如需字段可用 web_extract 或 body_mode=sample"]
}
```

**输出（raw）**

* 仍遵守截断策略：body 最高 `max_body_bytes_raw`
* headers 也限制条目数，敏感字段脱敏

```json
{
  "summary": { "...": "同上" },
  "request_raw": { "headers": { "...": "..." }, "body": null },
  "response_raw": {
    "headers": { "...": "..." },
    "body_truncated": true,
    "body_preview": "<head>....(head/tail采样)...</html>"
  }
}
```

---

### 7) web_repeat_compare

**用途**：多轮请求对比（性能波动定位），默认只回统计（p50/p95/min/max），不回每轮完整 raw

**输入**

```json
{
  "request": { "url": "...", "method": "GET", "headers": {}, "body": null, "auth": null },
  "repeat": 5,
  "options": { "timeout_ms": 15000, "need_timings": true, "body_mode": "none" }
}
```

**输出**

```json
{
  "repeat": 5,
  "http_status_distribution": { "200": 5 },
  "timings_ms_stats": {
    "dns": { "p50": 2, "p95": 6, "max": 6 },
    "tcp_connect": { "p50": 18, "p95": 40, "max": 41 },
    "tls_handshake": { "p50": 55, "p95": 120, "max": 130 },
    "ttfb": { "p50": 110, "p95": 420, "max": 460 },
    "total": { "p50": 190, "p95": 620, "max": 680 }
  },
  "suspected_bottleneck": "ttfb_spike|tls_flap|connectivity|stable",
  "notes": ["仅返回统计摘要；如需某一轮证据，可指定 run_index 再 web_http_request(raw)"]
}
```

---

## D. 内容校验与字段提取（替代“直接把大响应体贴出来”）

### 8) web_validate_content

**用途**：校验响应体格式合法性（JSON/XML/HTML）+ 可选关键字段存在性（不输出全量 body）

**输入**

```json
{
  "content_type": "application/json",
  "body_sample": "{\"code\":0,\"msg\":\"ok\"}",
  "rules": {
    "json_required_paths": ["$.code", "$.msg"],
    "expect_status": 200
  }
}
```

**输出**

```json
{
  "ok": true,
  "format": { "type": "json", "valid": true },
  "checks": [
    { "rule": "required_path", "target": "$.code", "pass": true },
    { "rule": "required_path", "target": "$.msg", "pass": true }
  ],
  "notes": []
}
```

---

### 9) web_extract

**用途**：从 JSON/XML/HTML 中按 JSONPath/XPath/CSS Selector 提取少量字段（最省 token）

**输入**

```json
{
  "format": "json|xml|html",
  "body_sample_or_raw": "...",
  "extract": {
    "jsonpath": ["$.code", "$.data.id"],
    "limit_each": 20
  }
}
```

**输出**

```json
{
  "extracted": {
    "$.code": 0,
    "$.data.id": "abc123"
  },
  "truncated": false
}
```

---

### 10) web_body_sample

**用途**：只取响应体采样（head/tail/均匀采样），用于“看大概内容但不拉全量”

**输入**

```json
{
  "request": { "url": "...", "method": "GET", "headers": {}, "body": null },
  "sample_bytes": 4096,
  "strategy": "head_tail|uniform",
  "timeout_ms": 15000
}
```

**输出**

```json
{
  "content_type": "text/html",
  "sampled_bytes": 4096,
  "body_preview": "<!doctype html>...(head)...(tail)...</html>",
  "truncated": true
}
```

---

## E. 报告生成（Postman 风格，但默认“精简版”）

### 11) web_report

**用途**：把上面 actions 的结果汇总成“Postman 风格请求详情”，但遵守 summary_first（默认不贴大段 body）

**输入**

```json
{
  "request_summary": { "...": "来自 web_http_request.summary" },
  "probes": { "dns": null, "tcp": null, "tls": null },
  "validation": null,
  "compare": null,
  "verbosity": "compact|normal|detailed"
}
```

**输出**

```json
{
  "overview": {
    "url": "...",
    "method": "GET",
    "http_version": "h2",
    "status": 200,
    "total_ms": 245,
    "response_body_bytes": 1024
  },
  "stage_timings_ms": { "dns": 12, "tcp_connect": 25, "tls_handshake": 60, "ttfb": 130, "download": 18 },
  "highlights": [
    "TTFB 占比高（130ms/245ms），更像后端处理/上游依赖慢",
    "TLS 握手正常（60ms），DNS 正常（12ms）"
  ],
  "evidence": {
    "remote_ip": "93.184.216.34",
    "content_type": "application/json",
    "server": "nginx"
  },
  "next_steps": [
    "如需定位后端：对比同机房/同域名不同路径，或开启 repeat_compare 看是否 TTFB 抖动",
    "如需业务字段：用 web_extract 提取 $.code/$.msg 而不是拉全量 body"
  ]
}
```

---

# 失败与拒绝策略（省 token + 防事故）

* **响应体过大风险**：`web_estimate` 判断 high → 自动建议 `body_mode=none/sample`，拒绝直接 raw
* **重定向过多**：超过 `max_redirects` → 停止并返回 redirect 链摘要
* **raw 必须截断**：任何 raw body 超过上限 → `body_truncated=true`，只回采样+统计
* **敏感信息必脱敏**：Authorization/Cookie/Set-Cookie/Token 字段一律掩码
* **多轮对比默认不回每轮详情**：只回统计；用户点名某一轮才取 raw 证据

---

# 推荐工作流示例（你要的“看用户要什么就给什么”）

## 示例 1：用户只想知道“接口是否通 + 状态码 + 总耗时”

1. `web_plan(goal="status_code", need_timings=true, need_body="none")`
2. `web_http_request(result_mode="summary", body_mode="none")`
3. `web_report(verbosity="compact")`

## 示例 2：用户怀疑“证书过期/TLS握手失败”

1. `web_plan(goal="tls_check", need_tls_details=true)`
2. `web_tls_probe(verify_cert=true)`
3. （必要时）`web_http_request(summary)` 佐证 HTTP 是否也失败

## 示例 3：用户说“偶发超时/波动”

1. `web_plan(goal="compare_runs", repeat=5, need_body="none")`
2. `web_repeat_compare(repeat=5)`
3. 若 p95 抖动明显：指定 `run_index` 用 `web_http_request(raw)` 拉一轮证据（仍截断）

## 示例 4：用户只关心 JSON 返回字段（不想看整段 body）

1. `web_http_request(summary, body_mode="sample")`
2. `web_validate_content(json_required_paths=["$.code","$.msg"])`
3. `web_extract(jsonpath=["$.code","$.msg","$.data.id"])`

---

# 实现提示（可选，给落地的人看）

* 阶段耗时可用 curl write-out 一类能力采集：`time_namelookup/time_connect/time_appconnect/time_starttransfer/time_total`
* HTTP 版本/ALPN/TLS/证书摘要也可在一次请求中采到；但为了“按需”，仍建议拆成 `tls_probe` 可独立跑
* body 采样建议统一走 `head_tail`，并配合 `content-type` 做 JSON/XML/HTML 轻量校验

---

## 输出约定（统一）

* 所有 actions 输出必须为**结构化 JSON**，尽量短
* 默认 `result_mode=summary`
* raw/采样必须带：`truncated`、`sample_strategy`、`bytes_returned`
* 永远避免把全量响应体直接拼进自然语言

---

如果你愿意把它再“贴合你们真实环境”（进一步减少无意义探测），你只需要给我两类真实输入样例（脱敏即可）：

* 你们最常用的 2~3 种查询目标（比如：只看状态码/看证书/排查慢）
* 你们常见接口返回的 content-type（json/html/xml/zip 等）
  我就能把 `web_plan` 的意图分流规则做得更准（比如：对 json 接口默认走 `web_extract`，对 html 默认 headers-only）。
