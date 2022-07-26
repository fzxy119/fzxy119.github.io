---
title: Es RESTful Api
date: 2022-07-26 13:54:57
tags:
- ES
categories:
- DB
---

### 查询所有索引信息

* http://{{host}}/_cat/indices

### 索引映射查看

* get http://{{host}}/agent-info

### 索引设置查看

* get http://{{host}}/agent-info/_settings

### 索引删除

* delete  http://{{host}}/agent-info

### 精确匹配

* get  http://{{host}}/agent-info/_search

```
{
    "query":{
        "match":{
            "ag_name": "大石桥市旺达水果蔬菜便利店"

        }
    }
}
```

### 索引全文检索

* get  http://{{host}}/agent-info/_search

### 关键词分析

* get http://{{host}}/_analyze
```
{
  "analyzer" : "ik_max_word",
  "text" : "739183800@qq.com"
}
```

### _cat操作类表

* get http://{{host}}/_cat

```
=^.^=
/_cat/allocation
/_cat/shards
/_cat/shards/{index}
/_cat/master
/_cat/nodes
/_cat/tasks
/_cat/indices
/_cat/indices/{index}
/_cat/segments
/_cat/segments/{index}
/_cat/count
/_cat/count/{index}
/_cat/recovery
/_cat/recovery/{index}
/_cat/health
/_cat/pending_tasks
/_cat/aliases
/_cat/aliases/{alias}
/_cat/thread_pool
/_cat/thread_pool/{thread_pools}
/_cat/plugins
/_cat/fielddata
/_cat/fielddata/{fields}
/_cat/nodeattrs
/_cat/repositories
/_cat/snapshots/{repository}
/_cat/templates
/_cat/ml/anomaly_detectors
/_cat/ml/anomaly_detectors/{job_id}
/_cat/ml/trained_models
/_cat/ml/trained_models/{model_id}
/_cat/ml/datafeeds
/_cat/ml/datafeeds/{datafeed_id}
/_cat/ml/data_frame/analytics
/_cat/ml/data_frame/analytics/{id}
/_cat/transforms
/_cat/transforms/{transform_id}

```

### 健康检查

* get http://{{host}}/_cat/health
> 1658815275 06:01:15 ryx green 3 3 10 8 0 0 0 0 - 100.0%
>
### 查看集群信息

* get http://{{host}}/_cluster/health
```
{
    "cluster_name": "ryx",
    "status": "green",
    "timed_out": false,
    "number_of_nodes": 3,
    "number_of_data_nodes": 3,
    "active_primary_shards": 8,
    "active_shards": 10,
    "relocating_shards": 0,
    "initializing_shards": 0,
    "unassigned_shards": 0,
    "delayed_unassigned_shards": 0,
    "number_of_pending_tasks": 0,
    "number_of_in_flight_fetch": 0,
    "task_max_waiting_in_queue_millis": 0,
    "active_shards_percent_as_number": 100.0
}
```
### 查看集群master
* get http://{{host}}/_cat/master
```
TzHcEWPTRW-D8vFqj0lTUQ 12.3.10.105 12.3.10.105 node-105
```
### 查看集群节点信息
* get http://{{host}}/_cat/nodes
```
12.3.10.205 65 97 3 0.02 0.11 0.17 cdfhilmrstw - node-205
12.3.10.222 15 94 3 0.03 0.07 0.06 cdfhilmrstw - node-222
12.3.10.105 51 98 2 0.06 0.08 0.02 cdfhilmrstw * node-105
```
### 查看集群分片信息
* get http://{{host}}/_cat/shards
```
agent-info       2 p STARTED 13569   1.8mb 12.3.10.205 node-205
agent-info       1 p STARTED 13104   1.8mb 12.3.10.205 node-205
agent-info       0 p STARTED 13476   1.8mb 12.3.10.222 node-222
.security-7      0 r STARTED     7  25.7kb 12.3.10.205 node-205
.security-7      0 p STARTED     7  25.7kb 12.3.10.222 node-222
.geoip_databases 0 r STARTED     4     4mb 12.3.10.205 node-205
.geoip_databases 0 p STARTED     4     4mb 12.3.10.222 node-222
order-info       2 p STARTED   570 266.7kb 12.3.10.222 node-222
order-info       1 p STARTED   508 249.2kb 12.3.10.205 node-205
order-info       0 p STARTED   521   259kb 12.3.10.222 node-222
```