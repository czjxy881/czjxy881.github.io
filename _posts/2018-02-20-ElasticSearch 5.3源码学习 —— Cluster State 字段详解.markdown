---
layout: single
title: "ElasticSearch 5.3源码学习 —— Cluster State 字段详解"
date: 2018-02-20 23:59:08 +0800
comments: true
categories: ElasticSearch
---
### 背景
+ ES采用事件模型进行状态变更，其ClusterState为其唯一持久化的集群状态信息。

### 字段详解
+ ES 5.3的ClusterState如下所示，主要分为
    + routingTable - 路由表
    + nodes - 节点信息
    + metaData - 元数据，也可以有custom扩展
    + blocks - 系统限制
    + customs - 自定义项目，如snapshots，可用插件扩展
+ 下面就具体以实例说明下每个值的具体作用，在集群中`GET _cluster/state`即可看见

```json
{
    "cluster_name": "elasticsearch",
    "version": 158326,
    "state_uuid": "iYcpseRmQYmacnkkfZHsjg",
    "master_node": "WychrvtFSFiZpChp0ElnVw",
    "blocks": {
        "global": { // 全局限制
            "1": {  // 限制id,所有block类型见附录
                "description": "state not recovered / initialized",
                "retryable": true, //是否重试，如果为true则会监听state变化，在发生变化时自动重试
                "disableStatePersistence": true, //为true将会清空之前持久化数据
                "levels": [ // 限制的项目，一共有读，写，元数据读，元数据写四种
                    "read",
                    "write",
                    "metadata_read",
                    "metadata_write"
                ]
            }
        },
        "indices": { //索引级别的限制
            "xxx": {
                "4": {
                    "description": "index closed",
                    "retryable": false,
                    "levels": [
                        "read",
                        "write"
                    ]
                }
            }
        }
    },
    "nodes": { //节点详细信息
        "9WPdZv8ESIS9_jWv26Dogw": {
            "name": "f1dba2f9-48c8-5896-97e2-983a520f1757",
            "ephemeral_id": "jbV855g3QQKj8_wpeWr3Jg",
            "transport_address": "1.2.3.4:9300",
            "attributes": {
                "box_type": "hot"
            }
        }
    },
    "metadata": { //元数据
        "cluster_uuid": "jsWIOmkxQgCkmxN5_baWmg",
        "templates": {}, //模板
        "indices": {
            "test": {
                "state": "open",
                "settings": {},
                "mappings": {},
                "aliases": [],
                "primary_terms": { 
                    "0": 13 //代表这个shard的primary切换的次数，用于区分新旧primary
                },
                "in_sync_allocations": {
                    "0": [ // 拥有最新数据的allocation,如果primary丢失，就从这个列表里选出个新的主.如果节点恢复，并且其id仍在这个列表中，则认为数据没有缺少，不做恢复。当副本没有返回ack时，primary会通知master将其从这个列表中移除，等其再返回全部ack后再加上。因此网络抖动可能造成master大量压力
                        "yxSi6kjZQC2I3LhwNbOdIQ", 
                        "xNMX9KECSm6Au3Q4R-NoQA"
                    ]
                }
            }
        },
        "ingest": {},//ingest的pipeline数据
        "index-graveyard": {
            "tombstones": [ //删掉的索引的墓碑,默认500个，防止node回来时不知道索引已经删掉了
                {
                    "index": {
                        "index_name": "append_only_sc_template_record.2017-11-25",
                        "index_uuid": "woWP1c_iQs-M3HHUR1XOJQ"
                    },
                    "delete_date_in_millis": 1512921691703
                }
            ]
        }
    },
    "routing_table": { //每个shard的具体路由信息
        "indices": {
            "test": {
                "shards": {
                    "0": [
                        {
                            "state": "STARTED", //共有四种状态，UNASSIGNED，INITIALIZING，STARTED,RELOCATING
                            "primary": true,
                            "node": "9WPdZv8ESIS9_jWv26Dogw",
                            "relocating_node": null, //如果为relocating状态，这个值表示relocating的目标机器
                            "shard": 0,
                            "index": "test",
                            "allocation_id": {
                                "id": "yxSi6kjZQC2I3LhwNbOdIQ" //当前allocation的uid，和in_sync_allocations对应
                            }
                        },
                        {
                            "state": "UNASSIGNED",
                            "primary": false,
                            "node": null,
                            "relocating_node": null,
                            "shard": 0,
                            "index": "test",
                            "recovery_source": { //恢复源,EMPTY_STORE,EXISTING_STORE,PEER,SNAPSHOT,LOCAL_SHARDS五种
                                "type": "PEER"
                            },
                            "unassigned_info": {
                                "reason": "CLUSTER_RECOVERED",
                                "at": "2017-09-27T03:26:36.488Z",
                                "delayed": false,
                                "allocation_status": "no_attempt"
                            }
                        }
                    ]
                }
            }
        }
    },
    "routing_nodes": { //这个就是把routing_table按照node重新整理了一下
        "unassigned": [
            {
                "state": "UNASSIGNED",
                "primary": false,
                "node": null,
                "relocating_node": null,
                "shard": 0,
                "index": "test",
                "recovery_source": {
                    "type": "PEER"
                },
                "unassigned_info": {
                    "reason": "CLUSTER_RECOVERED",
                    "at": "2017-09-27T03:26:36.488Z",
                    "delayed": false,
                    "allocation_status": "no_attempt"
                }
            }
        ],
        "nodes": {
            "9WPdZv8ESIS9_jWv26Dogw": [
                {
                    "state": "STARTED",
                    "primary": true,
                    "node": "9WPdZv8ESIS9_jWv26Dogw",
                    "relocating_node": null,
                    "shard": 0,
                    "index": "test",
                    "allocation_id": {
                        "id": "yxSi6kjZQC2I3LhwNbOdIQ"
                    }
                },
                {
                    "state": "STARTED",
                    "primary": false,
                    "node": "9WPdZv8ESIS9_jWv26Dogw",
                    "relocating_node": null,
                    "shard": 0,
                    "index": "test",
                    "allocation_id": {
                        "id": "xNMX9KECSm6Au3Q4R-NoQA"
                    }
                }
            ]
        }
    }
}
```

### 附录
+ ClusterBlock 列表

|Block Name|id|
|:--|:--|
|STATE_NOT_RECOVERED_BLOCK|1|
|NO_MASTER_BLOCK_ALL|2|
|NO_MASTER_BLOCK_WRITES|2|
|INDEX_CLOSED_BLOCK|4|
|INDEX_READ_ONLY_BLOCK|5|
|CLUSTER_READ_ONLY_BLOCK|6|
|INDEX_READ_BLOCK|7|
|INDEX_WRITE_BLOCK|8|
|INDEX_METADATA_BLOCK|9|
|TRIBE_METADATA_BLOCK|10|
|TRIBE_WRITE_BLOCK|11|

### 参考资料
+ [Tombstones](https://www.elastic.co/guide/en/elasticsearch/reference/current/misc-cluster.html)
+ [primary_term](https://www.elastic.co/blog/elasticsearch-sequence-ids-6-0)
+ [in_sync_allocation](https://www.elastic.co/blog/tracking-in-sync-shard-copies)

### 版权声明
+ 自由转载-非商用-非衍生-保持署名（创意共享3.0许可证）
+ 本文首发于: http://czjxy881.coding.me/

{% include toc %}