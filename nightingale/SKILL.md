---
name: nightingale
description: "Nightingale（夜莺）MCP 细颗粒度平台技能：基于官方 toolsets（alerts/targets/datasource/mutes/notify_rules/alert_subscribes/event_pipelines/users/busi_groups）。默认 summary_first + 字段白名单 + 分页，尽量少调用、少返回，避免 token 爆炸；当需要验证告警表达式/趋势分析时按需联动 Prometheus skills（estimate→summary→raw按需）。"
license: MIT
compatibility: opencode
metadata:
  audience: 运维工程师
  workflow: 告警管理/监控目标排障/事件响应/团队协作
  side_effect: read_only_by_default
  token_policy: summary_first
  version: 1.0.0
---

# Nightingale Skill（MCP 细颗粒度 + 联动 Prometheus）

## 1. 总体设计目标（省 token / 省调用 / 可联动）
- **Summary First**：默认只输出“摘要 + 关键证据”，不输出大段 JSON、不贴长文本字段。
- **最少工具调用**：先根据用户意图选择最小动作集（通常 1~2 个 MCP 调用）。
- **字段白名单**：所有 list/get 的输出做字段裁剪，避免 MCP 返回体过大。
- **分页与 topN**：list 默认只返回前 N 条，并提供 next/继续方式（不自动翻到底）。
- **必要时联动 Prometheus**：当用户提出“验证表达式/看趋势/为什么触发/是否命中”等，才把 PromQL 丢给 Prometheus skills（强制 estimate→summary→raw按需）。

---

## 2. 路由规则（AI 如何决定调用哪些 MCP 工具）

### 2.1 强制优先级（从上到下）
1) **用户明确问告警规则**（“有哪些规则/规则详情/某条规则怎么写的/阈值是多少/规则里通知到哪”）
   - 优先：`alerts.list_alert_rules` 或 `alerts.get_alert_rule`
2) **用户明确问活跃告警/历史告警**（“当前哪些在 firing/过去24h有哪些/某事件id详情”）
   - 活跃：`alerts.list_active_alerts` / `alerts.get_active_alert`
   - 历史：`alerts.list_history_alerts` / `alerts.get_history_alert`
3) **用户明确问监控目标/主机**（“业务组有哪些主机/离线目标/某实例状态”）
   - `targets.list_targets`
4) **用户明确问数据源**（“有哪些 datasource/某数据源 id 是啥”）
   - `datasource.list_datasources`
5) **用户明确要做事件响应**（“创建/更新屏蔽/查看屏蔽/看通知规则/看流水线执行记录”）
   - 屏蔽查询：`mutes.list_mutes` / `mutes.get_mute`
   - 屏蔽写入（有副作用，默认禁用，见 2.3 门禁）：`mutes.create_mute` / `mutes.update_mute`
   - 通知：`notify_rules.list_notify_rules` / `notify_rules.get_notify_rule`
   - 流水线：`event_pipelines.*`
6) **用户明确问订阅/团队协作**
   - 订阅：`alert_subscribes.*`
   - 用户/团队：`users.*`
7) **用户只说“夜莺里看看/帮我排查”但没有明确对象**
   - 默认只做：`alerts.list_alert_rules`（或先 `busi_groups.list_busi_groups` 确定 gid 后 list）
   - 不自动拉 targets/notify/mutes/pipelines（除非用户明确要）

### 2.2 业务组 gid 决策
- 用户给了 gid：直接用
- 用户给业务组名但没 gid：`busi_groups.list_busi_groups` 找匹配
- 用户没给任何业务组信息：
  - 先 `busi_groups.list_busi_groups`（只返回 topN id/name）
  - 再提示用户选择（或默认选 “Default Busi Group/第一个可访问业务组”）

> **缓存策略**：同一轮会话里，业务组列表/数据源列表获取一次后应复用，不要重复 list。

### 2.3 写操作门禁（默认只读）
本技能默认 **read_only**。只有满足以下条件才允许触发写操作：
- 用户明确包含：**动作词 + 对象词**
  - 动作词：创建/新增/更新/修改/变更/调整
  - 对象词：屏蔽规则（mute）
- 且用户提供必要参数（起止时间/匹配条件/业务组 gid 等）

允许写入的仅限：
- `mutes.create_mute`
- `mutes.update_mute`

> 注意：**告警规则的创建/修改/删除**不在当前 MCP 工具范围（无对应 tool），不得“脑补”写操作。

---

## 3. 全局保护参数（默认值，可按需覆盖）
- `default_limit`: 50（任何 list 默认最多 50）
- `hard_max_limit`: 200（用户明确要求更多时上限 200）
- `max_text_len`: 256（名称/描述/备注等长字段截断）
- `truncate_strategy`: topN + 截断 + 字段白名单
- `default_time_range`：
  - 活跃告警：默认最近 1h（或不指定让后端默认）
  - 历史告警：默认最近 24h（用户没给时间范围时）
- `output_mode`：
  - 默认只输出摘要表（id/name/state/time/关键标签/简短原因）
  - 仅当用户点名某 id 才 get 详情

---

## 4. 官方工具集（Actions）与“省 token 使用法”

> 所有 Actions 均映射你贴的官方工具，下面是“何时调用 + 最短输出形态”。

### A) busi_groups
#### A1. `busi_groups.list_busi_groups`
**用途**：列出当前用户可访问业务组（只输出 id + name）
**调用时机**：用户没提供 gid 或业务组名需要匹配时
**输出（summary）**：
- `[{id,name}]` + `truncated`

---

### B) alerts（告警/规则）
#### B1. `alerts.list_active_alerts`
**用途**：列出当前活跃告警（支持过滤）
**输出（summary）字段白名单建议**：
- `event_id, rule_id, rule_name, severity, state, first_trigger_time, last_eval_time, tags(top5), brief_text`
**策略**：只返回 topN；如需某条详情再 `get_active_alert`

#### B2. `alerts.get_active_alert`
**用途**：按事件 ID 获取活跃告警详情
**策略**：只在用户点名 event_id 或需要证据时调用；输出裁剪（不贴长原文）

#### B3. `alerts.list_history_alerts`
**用途**：列出历史告警（默认最近 24h）
**输出字段**：`event_id, rule_id, rule_name, severity, fired_at, recovered_at, duration, tags(top5)`

#### B4. `alerts.get_history_alert`
**用途**：按事件 ID 获取历史告警详情（同 get_active_alert 策略）

#### B5. `alerts.list_alert_rules`
**用途**：列出业务组告警规则
**输出（summary）字段白名单建议**：
- `rule_id, name, enabled, severity(if any), datasource_hint(if any), update_at, brief_expr(截断)`
**策略**：默认只列 topN；用户要“某条规则详情/表达式/通知关联”才 `get_alert_rule`

#### B6. `alerts.get_alert_rule`
**用途**：获取告警规则详情（用于：查看表达式/评估周期/for/标签/通知关联等）
**输出（details）字段白名单建议**：
- `id, name, enabled, expr/promql, for, every/eval_interval, datasource_hint, notify_rule_ids(if any), tags/labels(top10), update_at`

---

### C) targets（监控目标）
#### C1. `targets.list_targets`
**用途**：列出被监控主机/目标（支持过滤，如离线超过 5 分钟）
**输出字段建议**：
- `ident/instance, job(if any), state(up/down), last_scrape, last_error(截断), labels(top8)`
**策略**：默认只回 “down/问题目标”；用户明确要全量再扩大过滤。

---

### D) datasource
#### D1. `datasource.list_datasources`
**用途**：列出所有可用数据源（用于把规则 expr 交给 Prometheus skills 时选择 base_url）
**输出字段**：`id,name,type,url(可选/脱敏), is_default`
**策略**：只有当用户明确要“数据源列表/规则验证需要确定数据源”时才调用。

---

### E) mutes（屏蔽规则）
#### E1. `mutes.list_mutes`
**用途**：列出业务组的告警屏蔽规则
**输出字段**：`id,name,enabled,start,end,matchers(top5),creator,update_at`

#### E2. `mutes.get_mute`
**用途**：屏蔽规则详情（只在点名 id 时）

#### E3. `mutes.create_mute`（写）
**门禁**：仅当用户明确“创建屏蔽规则”且提供关键参数时才调用
**输出**：`created_id + 摘要(时间窗/匹配条件/业务组)`（不贴完整 payload）

#### E4. `mutes.update_mute`（写）
**门禁**：仅当用户明确“更新屏蔽规则”且提供 id 与变更内容
**输出**：`id + diff_summary`

---

### F) notify_rules（通知规则）
#### F1. `notify_rules.list_notify_rules`
**用途**：列出所有通知规则（用于“这条规则通知到哪/有哪些通知规则”）
**输出字段**：`id,name,channels/receivers(摘要),enabled,update_at`

#### F2. `notify_rules.get_notify_rule`
**用途**：通知规则详情（仅点名 id 时）

---

### G) alert_subscribes（订阅）
#### G1. `alert_subscribes.list_alert_subscribes`
#### G2. `alert_subscribes.list_alert_subscribes_by_gids`
#### G3. `alert_subscribes.get_alert_subscribe`
**策略**：同上：list 先摘要，点名再 get。

---

### H) event_pipelines（事件流水线）
#### H1. `event_pipelines.list_event_pipelines`
#### H2. `event_pipelines.get_event_pipeline`
#### H3. `event_pipelines.list_event_pipeline_executions`
#### H4. `event_pipelines.list_all_event_pipeline_executions`
#### H5. `event_pipelines.get_event_pipeline_execution`
**策略**：默认只输出 “pipeline id/name/最近执行状态/失败摘要”；排查某次执行再 get_execution。

---

### I) users（用户/团队）
#### I1. `users.list_users`
#### I2. `users.get_user`
#### I3. `users.list_user_groups`
#### I4. `users.get_user_group`
**策略**：只返回必要字段（id/name/email(可选脱敏)/members_count）；点名再取成员详情。

---

## 5. 联动 Prometheus skills（何时联动 + 怎么联动）

### 5.1 触发联动的典型语句
- “这条规则为什么触发？”
- “这个告警表达式命中了哪些机器？”
- “帮我验证一下表达式在过去 1 小时趋势”
- “阈值是不是太低/是否波动？”

### 5.2 联动流程（强制）
1) `alerts.get_alert_rule` 获取 `expr/promql` + （如果有）`datasource_hint`
2) （可选）`datasource.list_datasources` 仅在需要把 hint 映射到 Prom base_url 时调用
3) 调 Prometheus skills（必须遵守它的 guardrails）：
   - `prom_estimate_cardinality(query=expr, start/end, step=auto)`
   - `prom_range_query(result_mode=summary)`
   - `analyze_trend(range_summary=...)`
4) 返回最终结论：只用 summary 统计（min/max/p95/last/trend/top_series），不贴全量点位

> 约束：Prometheus raw 模式必须显式请求，并且截断+采样。

---

## 6. 输出模板（固定短格式）

### 6.1 规则列表（默认）
- `gid/name`
- `top_rules`: `id | name | enabled | datasource | brief_expr | updated_at`
- `truncated/next_hint`

### 6.2 活跃告警（默认）
- `count`
- `top_alerts`: `event_id | rule_name | severity | since | tags(top5) | brief`
- `next_hint`（过滤条件建议：severity/gid/关键标签）

### 6.3 目标状态（默认）
- `down_count/up_count`
- `down_list`: `ident/instance | last_scrape | last_error(截断)`

---

## 7. 失败与降级策略（防 token 爆炸 + 防误操作）
- list 返回过多：只给 topN + 提示过滤（关键词/时间范围/gid/状态）
- get 返回字段过大：字段白名单 + 截断长字段
- toolsets 未启用（例如你只启用了 alerts,busi_groups，却调用 targets）：提示用户在 MCP 启动参数里启用对应 toolsets（例如追加 `--toolsets targets`）
- 写操作：若 read_only=true 或用户意图不明确 → 拒绝并给“如何触发”的一句示例

---

## 8. 推荐工作流示例（最少调用）

### 示例 1：Default Busi Group 有哪些告警规则？
1) `busi_groups.list_busi_groups`（仅首次）
2) `alerts.list_alert_rules(gid=...)`

### 示例 2：某条告警为什么触发？验证表达式命中情况（联动 Prom）
1) `alerts.get_alert_rule(id=...)`
2) Prometheus：`prom_estimate_cardinality` → `prom_range_query(summary)` → `analyze_trend`

### 示例 3：创建 2 小时屏蔽（写操作门禁）
1) 判断用户明确“创建屏蔽规则”且给出：gid + matchers + start/end
2) `mutes.create_mute`
3) 返回 created_id + 摘要

---