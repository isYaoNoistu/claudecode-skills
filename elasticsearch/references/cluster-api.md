# 集群与节点 API 参考

> **注意：** 以下大多数 API 在 **Serverless** Elasticsearch 中不可用，仅适用于自建集群或传统 Elastic Cloud 部署。

## 集群健康

```bash
# 快速健康检查
curl -s "${ES_URL%/}/_cluster/health" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '{status, number_of_nodes, active_shards, unassigned_shards}'

# 按索引查看健康状态（找出非 green 的索引）
curl -s "${ES_URL%/}/_cluster/health?level=indices" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '.indices | to_entries[] | select(.value.status != "green") | {index: .key, status: .value.status}'
```

状态含义：
- **green**：所有主分片和副本分片均已分配
- **yellow**：所有主分片已分配，部分副本缺失（单节点集群常见）
- **red**：部分主分片未分配，数据可能不可访问

## 集群统计

```bash
curl -s "${ES_URL%/}/_cluster/stats" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '{
    nodes:      .nodes.count.total,
    indices:    .indices.count,
    docs:       .indices.docs.count,
    store_size: .indices.store.size_in_bytes,
    memory:     .nodes.jvm.mem.heap_used_in_bytes
  }'
```

## 集群设置

```bash
# 获取所有设置（persistent + transient）
curl -s "${ES_URL%/}/_cluster/settings?include_defaults=true&flat_settings=true" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# 更新 persistent 设置
curl -s -X PUT "${ES_URL%/}/_cluster/settings" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "persistent": {
      "cluster.routing.allocation.enable": "all"
    }
  }'
```

## 节点统计

```bash
# 所有节点统计
curl -s "${ES_URL%/}/_nodes/stats" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '.nodes | to_entries[] | {name: .value.name, heap_pct: .value.jvm.mem.heap_used_percent, cpu: .value.os.cpu.percent}'

# 指定统计维度
curl -s "${ES_URL%/}/_nodes/stats/jvm,os,fs" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# 热线程（排查节点性能问题）
curl -s "${ES_URL%/}/_nodes/hot_threads" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

## 节点信息

```bash
curl -s "${ES_URL%/}/_nodes" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  | jq '.nodes | to_entries[] | {name: .value.name, roles: .value.roles, version: .value.version}'
```

## 分片分配

```bash
# 列出分片及分配情况（按大小降序）
curl -s "${ES_URL%/}/_cat/shards?v&s=store:desc&h=index,shard,prirep,state,store,node" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 为什么某个分片未分配？
curl -s -X POST "${ES_URL%/}/_cluster/allocation/explain" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "index": "my-index",
    "shard": 0,
    "primary": true
  }' | jq .

# 手动重新路由分片
curl -s -X POST "${ES_URL%/}/_cluster/reroute" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "commands": [
      {"allocate_replica": {"index": "my-index", "shard": 0, "node": "node-2"}}
    ]
  }'
```

## 挂起任务

```bash
curl -s "${ES_URL%/}/_cluster/pending_tasks" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

curl -s "${ES_URL%/}/_cat/pending_tasks?v" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

## 任务管理

```bash
# 列出运行中的任务
curl -s "${ES_URL%/}/_tasks?detailed=true&actions=*reindex*" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# 取消任务
curl -s -X POST "${ES_URL%/}/_tasks/node_id:task_id/_cancel" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```

## Cat API 快捷命令

集群概览命令（加 `?v` 显示列头，`?format=json` 输出 JSON）：

```bash
# 集群健康一行概览
curl -s "${ES_URL%/}/_cat/health?v"         -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 节点资源概览
curl -s "${ES_URL%/}/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent,node.role" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 索引按大小排序
curl -s "${ES_URL%/}/_cat/indices?v&s=store.size:desc" -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 分片按大小排序
curl -s "${ES_URL%/}/_cat/shards?v&s=store:desc"       -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 段信息
curl -s "${ES_URL%/}/_cat/segments?v"                  -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 正在进行的恢复
curl -s "${ES_URL%/}/_cat/recovery?v&active_only"      -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# 线程池（排查拒绝问题）
curl -s "${ES_URL%/}/_cat/thread_pool?v&h=node_name,name,active,queue,rejected" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)"

# Fielddata 内存占用
curl -s "${ES_URL%/}/_cat/fielddata?v"                 -H "Authorization: ApiKey $(printenv ES_API_KEY)"
```
