---
name: jenkins
description: "Jenkins 细颗粒度流水线查询与执行技能：默认 summary_first + 字段白名单 + 分页/截断，避免一次性拉全量 job/build/log 导致 token 爆炸。支持查询 Job/Build/Queue/Agent 状态、常规巡检与基础执行（触发构建/带参数构建/终止构建）。Jenkins 开启 CSRF（useCrumbs=true）时，所有 POST 必须先获取 crumb（带 cookies），再携带 crumb+cookies 触发，触发成功返回 HTTP 201。"
license: MIT
compatibility: opencode
metadata:
  audience: 运维工程师/CI工程师
  workflow: 持续集成/发布排障/流水线巡检
  side_effect: read_only_by_default
  token_policy: summary_first
  version: 1.0.1
---

# Jenkins Skill（细颗粒度 + Summary First + CSRF Crumb）

## 1) 设计目标（省 token / 省调用 / 可执行但可控）
- **Summary First**：默认只返回摘要（状态/耗时/最近构建/失败原因片段），不回大 JSON、不贴长日志。
- **最少工具调用**：按用户意图选最小动作集（通常 1~3 次 API）。
- **日志强限制**：控制台日志只允许 tail/head、关键字提取；禁止拉全量 consoleText。
- **写操作门禁**：触发构建/终止构建属于有副作用操作，只有用户明确“执行/触发/停止”才允许调用。
- **CSRF Crumb 流程固化**：useCrumbs=true 时，POST/stop 等写操作必须：先取 crumb（带 cookies）→ 再带 crumb+cookies 请求。

---

## 2) 必要配置（连接 Jenkins）
- `jenkins_base_url`: 例如 `http://192.168.1.1:8080`（建议不以 `/` 结尾）
- `auth`（推荐 API Token）：
  - `user`: 用户名
  - `api_token`: API Token
- `timeout_seconds`: 默认 10
- `verify_tls`: 默认 true（自签名可关）
- `use_crumbs`: 默认 true（你环境为 true）

---

## 3) 全局默认保护参数（可覆盖）
- `max_jobs_returned`: 50
- `max_builds_returned`: 20
- `max_queue_items`: 50
- `max_agents_returned`: 50
- `default_log_lines`: 120
- `max_log_lines`: 300
- `max_log_bytes`: 32768
- `truncate_strategy`: `head_tail`
- `default_result_mode`: `summary`（summary | raw）
- `cookie_jar_strategy`: `per_action`（默认每次写操作单独获取 crumb；如果要更省调用，可改为 per_session 缓存）
- `crumb_cache_ttl_seconds`: 120（仅当启用 per_session 缓存时使用）

---

## 4) 路由规则（AI 如何使用本 Skill）

### 4.1 默认查询流程（强制优先级）
1) **先定位 job**：`jenkins_find_job`
2) **先看摘要**：`jenkins_job_summary`
3) **排障下钻**：
   - `jenkins_build_summary`（默认 lastBuild/lastFailedBuild）
   - `jenkins_console_tail`（只 tail / 关键字过滤）
4) **队列/节点问题**：
   - `jenkins_queue_summary`
   - `jenkins_agents_summary`

### 4.2 写操作门禁（必须满足）
只有当用户明确包含：
- 动作词：`触发/执行/运行/重跑/开始构建/停止/终止/取消`
- 且对象明确：job 名（或 build number）
才允许调用写 actions；否则仅返回“如何执行”的 dry plan。

### 4.3 CSRF 触发构建固定流程
当 useCrumbs=true 时，任何 POST 类动作必须按顺序执行：

1) `jenkins_get_crumb`（携带 cookies）
2) `jenkins_trigger_build` / `jenkins_trigger_build_with_params` / `jenkins_abort_build`（携带 cookies + crumbHeader）
3) 成功判定：**HTTP 201**（build / buildWithParameters 常见成功码）

> 注意：crumb 请求必须写入 cookie jar；触发构建时必须带回 cookie jar。

---

## 5) Actions（能力模块）

> 所有 actions 输出结构化 JSON，默认 summary；日志类严格截断。

---

# A. 发现与定位（Discovery）

## A1) jenkins_find_job
**用途**：按关键词/正则定位 job（含 folder/multibranch）
**输入**
- `query`（必填）
- `scope`（可选）：`all|folder:<path>`
- `limit`（默认 50）

**输出（summary）**
```json
{
  "matched": 3,
  "items": [
    { "full_name": "team-a/service-x/build", "url": "http://.../job/team-a/job/service-x/job/build/" }
  ],
  "truncated": false
}


---

# B. Job / Build 摘要

## B1) jenkins_job_summary

**用途**：获取 job 健康与最近构建摘要（默认入口）
**输入**

* `job_full_name`（必填）
* `builds_limit`（默认 10）

**输出（summary）**

```json
{
  "job": { "full_name": "team-a/service-x/build", "disabled": false, "color": "blue|red|disabled|..." },
  "last_build": { "number": 123, "result": "SUCCESS|FAILURE|ABORTED|null", "duration_ms": 63123, "timestamp": "..." },
  "health": [ { "score": 80, "description": "..." } ],
  "recent_builds": [
    { "number": 123, "result": "SUCCESS", "duration_ms": 63123, "timestamp": "..." }
  ],
  "notes": ["如需失败原因：用 jenkins_build_summary + jenkins_console_tail"]
}
```

## B2) jenkins_build_summary

**用途**：获取某次 build 的结果、触发原因、变更摘要、（可选）阶段摘要
**输入**

* `job_full_name`（必填）
* `build_number`（可选，默认 lastBuild）

**输出（summary）**

```json
{
  "build": { "number": 123, "result": "FAILURE", "building": false, "duration_ms": 42000, "timestamp": "..." },
  "causes": ["Started by user xxx", "Git push"],
  "scm": { "branch": "main", "commit": "abcd..." },
  "stages": [
    { "name": "Build", "status": "FAILED", "duration_ms": 12000 }
  ],
  "next_step": "jenkins_console_tail"
}
```

---

# C. 控制台日志（强限制）

## C1) jenkins_console_tail

**用途**：获取控制台日志尾部（禁止全量）
**输入**

* `job_full_name`（必填）
* `build_number`（可选，默认 lastBuild）
* `lines`（默认 120，最大 300）
* `keyword`（可选）：只返回包含关键字的行（更省）
* `max_bytes`（默认 32768）

**输出（summary）**

```json
{
  "build_number": 123,
  "mode": "tail",
  "lines_returned": 120,
  "truncated": true,
  "highlights": [
    "ERROR: ...",
    "java.lang.OutOfMemoryError ..."
  ]
}
```

---

# D. Queue / Agent / 巡检

## D1) jenkins_queue_summary

**用途**：查看队列拥塞与排队原因
**输入**

* `limit`（默认 50）

**输出（summary）**

```json
{
  "queue_size": 12,
  "top_items": [
    { "id": 999, "task": "team-a/service-x/build", "why": "Waiting for next available executor" }
  ],
  "notes": ["如 why 指向节点不足，继续查 jenkins_agents_summary"]
}
```

## D2) jenkins_agents_summary

**用途**：查看 agent 在线/离线与 executor 使用情况
**输入**

* `only_problematic`（默认 true）
* `limit`（默认 50）

**输出（summary）**

```json
{
  "total": 20,
  "online": 18,
  "offline": 2,
  "problem_agents": [
    { "name": "agent-01", "offline": true, "offline_cause": "..." }
  ]
}
```

## D3) jenkins_health_check

**用途**：对某 job 做常规巡检摘要
**输入**

* `job_full_name`（必填）
* `window_builds`（默认 10）

**输出（summary）**

```json
{
  "job_full_name": "...",
  "signals": {
    "disabled": false,
    "recent_failure_rate": 0.3,
    "last_success_age_minutes": 180,
    "avg_duration_ms": 65000,
    "queue_pressure": "low|medium|high",
    "agent_risk": "low|medium|high"
  },
  "findings": [
    "最近10次失败率偏高（3/10）",
    "上次成功距今 180 分钟"
  ],
  "recommended_next": [
    "jenkins_build_summary(build=lastFailedBuild)",
    "jenkins_console_tail(keyword=\"ERROR\")"
  ]
}
```

---

# E. CSRF Crumb（写操作前置）

## E1) jenkins_get_crumb

**用途**：当 Jenkins 开启 CSRF（useCrumbs=true）时，获取 crumb 并保存 cookies
**输入**

* `auth`（可选，默认用全局 auth）
* `cookie_jar_id`（可选）：用于在同一轮操作中复用 cookies（建议由调用方生成一个 UUID）
* `timeout_seconds`（可选）

**行为**

* GET `${base}/crumbIssuer/api/json`
* 必须携带 BasicAuth
* 必须保存 Set-Cookie（cookie jar）
* 返回 crumb 与 crumbRequestField（通常是 Jenkins-Crumb）

**输出**

```json
{
  "cookie_jar_id": "jar_abc123",
  "crumb": "13433c05eeb5...",
  "crumbRequestField": "Jenkins-Crumb"
}
```

---

# F. 执行操作（有副作用：严格门禁 + CSRF 必须）

## F1) jenkins_trigger_build

**用途**：触发构建（无参数）
**前置**：必须先 `jenkins_get_crumb` 得到 `cookie_jar_id + crumbRequestField + crumb`
**输入**

* `job_full_name`（必填）
* `crumb_context`（必填）：来自 `jenkins_get_crumb` 输出
* `expect_status`（默认 201）

**行为**

* POST `${job_url}/build`
* Header：`${crumbRequestField}: ${crumb}`
* 携带 cookie jar
* 成功判定：HTTP **201**

**输出**

```json
{
  "ok": true,
  "http_status": 201,
  "queued": true,
  "notes": ["如需查看队列：jenkins_queue_summary；如需追踪构建号：可从 queue item 继续查"]
}
```

## F2) jenkins_trigger_build_with_params

**用途**：触发参数化构建
**前置**：必须先 `jenkins_get_crumb`
**输入**

* `job_full_name`（必填）
* `params`（必填）：键值对
* `crumb_context`（必填）
* `content_type`（默认 `application/x-www-form-urlencoded`）
* `expect_status`（默认 201）
* `validate_only`（可选，默认 false）：true 时只检查参数是否存在（通过 `jenkins_job_parameters`），不触发

**行为**

* POST `${job_url}/buildWithParameters`
* Header：crumb + content-type
* Body：urlencoded params
* 携带 cookie jar
* 成功判定：HTTP **201**

**输出**

```json
{
  "ok": true,
  "http_status": 201,
  "validate_only": false,
  "queued": true,
  "params_used": { "BRANCH": "main", "ENV": "uat" }
}
```

## F3) jenkins_abort_build

**用途**：终止正在运行的 build
**前置**：必须先 `jenkins_get_crumb`
**输入**

* `job_full_name`（必填）
* `build_number`（必填）
* `crumb_context`（必填）
* `expect_status`（默认 200|201|302 之一；不同 Jenkins 可能返回重定向）

**行为**

* POST `${job_url}/${build_number}/stop`
* Header：crumb
* 携带 cookie jar

**输出**

```json
{
  "ok": true,
  "build_number": 123,
  "aborted": true
}
```

---

## 6) CSRF 示例（对照用）

> 注意：示例用占位符，推荐使用 API Token，不要用明文密码。

```bash
# 1) 获取 crumb（写入 cookies）
curl -s -c /tmp/cookies.txt -u "admin:<api_token>" \
  "http://192.168.1.1:9090/crumbIssuer/api/json"

# 2) 触发构建（带 cookies + crumb header），成功应返回 HTTP 201
curl -s -b /tmp/cookies.txt -u "admin:<api_token>" \
  -X POST \
  -H "Jenkins-Crumb: <crumb_value>" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  "http://192.168.1.1/job/Jenkins-test/build"
```

---

## 7) 输出约定（统一）

* 所有 actions 返回结构化 JSON，尽量短
* 默认 summary；raw 仅用于“必须展示细节”的场景
* 控制台日志必须截断/采样/关键字过滤，不允许全量输出
* 写操作必须：**用户明确意图 + crumb+cookies 前置**

---
