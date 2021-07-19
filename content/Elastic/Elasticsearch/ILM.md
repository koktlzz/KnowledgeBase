---
title: "ILM"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  elastic:
    parent: "Elasticsearch"
weight: 700
---

## 概述

我们可以通过配置索引生命周期管理 ILM（Index Lifecycle Management）策略，来自动管理索引以达到某些效果：

- Rollover：当索引达到一定大小、创建超过一定时间或文档达到一定数量时，创建一个新索引；
- Shrink：减少索引中主分片的数量；
- Force merge：手动触发合并以减少索引中每个分片中的 segment 数，并释放已删除文档所使用的空间；
- Freeze：将索引设为只读，最大程度地减少其内存占用量；
- Delete：永久删除索引，包括其所有数据和元数据。

举例来说，我们要一台 ATM 的指标数据索引到 Elasticsearch 中，那么便可以配置相关的 ILM 策略：

- 当索引主分片的总大小达到 50GB 时，创建新索引；
- 将旧索引标记为只读，并将其缩小为单个分片；
- 7 天后，将索引移至较便宜的硬件上；
- 30 天后，删除索引。

## 索引的生命周期

索引的生命周期共分为 4 个阶段（Phase）：

- Hot：索引正在不断更新且可以被搜索到；
- Warm：索引不再更新但可以被搜索到；
- Cold：索引不再更新且很少被搜索。虽然依然可以被搜索，但是搜索速度可能会变慢；
- Delete：索引不再被需要，可以安全地将其删除。

当某阶段中的所有动作均完成（Phase execution）且超过了最小持续时间，ILM 便会将索引转换到下一个阶段（Phase transitions）。每个阶段默认的最小持续时间为 0，因此我们在配置 ILM 策略时，一般会为每个阶段设置一个最小持续时间。由于 Elasticsearch 只能在健康状况为绿色的集群上执行某些清理任务，所以 ILM 可能会在健康状况为黄色的集群中失效。

ILM 控制每个阶段执行动作的顺序和与每个动作相关的索引操作的步骤。

## 数据层

数据层（[data tier](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-tiers.html)）是 Elasticsearch 集群中保存相同类型索引的节点集合。此处索引的类型是根据其生命周期划分的，因此对应的数据层分别为：Hot tier，Warm tier 和 Cold tier。上述三种数据层常用来保存如日志、指标等时间序列数据，而对产品目录、用户档案等需要持久化存储的数据进行生命周期管理是没有意义的，因此 Elasticsearch 引入 Content tier 来存放这种长时间内相对恒定的数据。

- 当文档被直接写入到指定的索引时，它们会无限期地保留在 Content tier 节点上；
- 当文档被写入到数据流（[data stream](https://www.elastic.co/guide/en/elasticsearch/reference/7.11/data-streams.html)）时，它们最初位于 Hot tier 节点上。随后根据 ILM 策略移动到其他数据层节点上；
- 节点位于的数据层可以在 elasticsearch.yml 文件中的`node.roles`参数中进行配置；
- 索引的`index.routing.allocation.include._tier_preference`参数可以指定索引被分配到的数据层。

## 阶段动作（Phase action）

每个阶段中支持的动作如下：

|       | Hot     | Warm    | Cold     | Delete    |
| ------------------- | ---- | ---- | ---- | ------ |
| Set Priority        | $\checkmark$    | $\checkmark$    | $\checkmark$    |        |
| Unfollow            | $\checkmark$    | $\checkmark$    | $\checkmark$    |        |
| Rollover            | $\checkmark$    |      |      |        |
| Read-Only           | $\checkmark$    | $\checkmark$    |      |        |
| Shrink              | $\checkmark$    | $\checkmark$    |      |        |
| Force merge         | $\checkmark$    | $\checkmark$    |      |        |
| Allocate            |      | $\checkmark$    | $\checkmark$    |        |
| Migrate             |      | $\checkmark$    | $\checkmark$    |        |
| Freeze              |      |      | $\checkmark$    |        |
| Searchable Snapshot | $\checkmark$    |      | $\checkmark$    |        |
| Wait For Snapshot   |      |      |      | $\checkmark$      |
| Delete              |      |      |      | $\checkmark$      |

### Set Priority

一旦索引进入相应阶段，就为其设置优先级（`priority`）。如果节点重新启动，将先恢复优先级级别更高的索引。通常，处于 Hot 阶段的索引应具有最高值，而处于 Cold 阶段的索引应具有最低值。每个索引的`priority`默认值为 1。

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "actions": {
          "set_priority" : {
            "priority": 50
          }
        }
      }
    }
  }
}
```

### Unfollow

涉及到跨集群复制（ Cross-cluster replication），详见 [跨集群复制 (CCR) 深度解析](https://www.elastic.co/cn/webinars/cross-cluster-replication?baymax=rtp&elektra=docs&storm=sidebar3)。

### Rollover

当索引达到一定大小、创建超过一定时间或文档达到一定数量时，创建一个新索引并接收文档写入。注意，只有带有别名的索引（index alias）和数据流（data stream）支持 Rollover 动作。对于带有别名的索引，必须符合下列条件：

- 索引名称的格式符合**^.\*-\d+$**，如 my-index-000001；
- 参数`settings.index.lifecycle.rollover_alias`必须被定义。

由于创建的新索引（如 my-index-000002）的别名与旧索引相同，因此 Elasticsearch 引入参数`aliases.<alias-name>.is_write_index`来表示当前正在写入文档的索引（[write index](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-rollover-index.html#indices-rollover-is-write-index)）。例如，初始索引 my-index-000001 的该参数为`true`。当它执行 Rollover 动作后，创建了新索引 my-index-000002。那么新索引 my-index-000002`is_write_index`参数为`true`，旧索引的则自动变为`false`。

我们创建一个索引 my-index-000001，别名为 my_data。进行下列配置后便能够支持 Rollover 动作：

```json
PUT my-index-000001
{
  "settings": {
    "index.lifecycle.name": "my_policy",
    "index.lifecycle.rollover_alias": "my_data"
  },
  "aliases": {
    "my_data": {
      "is_write_index": true
    }
  }
}
```

上文提到，ILM 执行 Rollover 动作的条件为：索引达到一定大小、创建超过一定时间或文档达到一定数量，它们所对应的参数分别为：`max_size`、`max_age`、`max_docs`。常见的 Rollover 配置如下：

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover" : {
            "max_age": "7d",
            "max_size": "100GB"
            "max_docs": 100000000
          }
        }
      }
    }
  }
}
```

当指定多个参数时，只要有一个条件满足，ILM 就会执行 Rollover 动作。

### Read-Only

将索引变为只读模式。另外，若想在 Hot 阶段执行该动作，必须存在 Rollover 动作的配置，否则将被 ILM 拒绝。

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "actions": {
          "readonly" : { }
        }
      }
    }
  }
}
```

### Shrink

将索引设置为只读，并将其收缩为一个具有更少主分片数的新索引，新索引的名称为`shrink-<original-index-name>`。Shrink 动作将所有主分片全部分配到一个节点上，并在执行完毕后将与原索引关联的别名转换为新索引。与 Read-Only 动作相同，若想在 Hot 阶段执行该动作，必须存在 Rollover 动作的配置，否则将被 ILM 拒绝。

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "actions": {
          "shrink" : {
            "number_of_shards": 1
          }
        }
      }
    }
  }
}
```

### Force merge

将索引强制合并为指定的 segement 数，该操作将使索引变为只读。同样，若想在 Hot 阶段执行该动作，必须存在 Rollover 动作的配置，否则将被 ILM 拒绝。

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "actions": {
          "forcemerge" : {
            "max_num_segments": 1
          }
        }
      }
    }
  }
}
```

### Allocate

更改索引分片分配的节点和副本的个数。可选参数如下：

- `number_of_replicas`：索引副本数
- `include`：将分片分配到至少有一个参数符合配置的节点上
- `exclude`：将分片分配到参数不符合配置的节点上
- `require`：将分配分配到参数全部符合配置的节点上

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "actions": {
          "allocate" : {
            "number_of_replicas": 2,            // 分片的副本数更改为 2
            "include" : {
              "box_type": "cold"                // elasticsearch.yml 文件中的 node.attr.box_type 字段
            },
            
         
        }
      }
    }
  }
}
```

### Migrate

在生命周期的阶段转换时，通过更新索引设置中的`index.routing.allocation.include._tier_preference`参数，将索引移动到与当前阶段对应的数据层节点上。

- 如果 Warm 和 Cold 阶段中没有指定 Allocate 动作，那么 ILM 将会自动执行 Migrate 动作；
- 如果在 Allocate 动作中仅修改了分片副本数，那么 ILM 会在执行 Migrate 动作前减少副本数；
- 可以将`enabled`参数设置为`false`从而禁用 ILM 自动执行的 Migrate 动作。

### Freeze

Freeze 动作将索引的内存占用量降低到最低，该索引被搜索时的响应变慢（原理参考：[https://www.elastic.co/guide/en/elasticsearch/reference/current/frozen-indices.html](https://www.elastic.co/guide/en/elasticsearch/reference/current/frozen-indices.html))。

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "cold": {
        "actions": {
          "freeze" : { }
        }
      }
    }
  }
}
```

### Searchable Snapshot

为索引拍摄快照，并将其挂载为 [可搜索快照](/elastic/elasticsearch/snapshot/#可搜索快照)。想在 Hot 阶段执行该动作，必须存在 Rollover 动作的配置，否则将被 ILM 拒绝。若在 Hot 阶段执行 Searchable Snapshot，则在后续阶段中无法定义任何 Shrink, Force merge 和 Freeze 动作。默认情况下，当索引在 Delete 阶段被删除后，相应的快照也随之删除。

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "cold": {
        "actions": {
          "searchable_snapshot" : {
            "snapshot_repository" : "backing_repo"    // 指定存储快照的仓库为 backing_repo
          }
        }
      }
    }
  }
}
```

### Wait for snapshot

等待指定的快照生命周期管理 SLM（Snapshot Lifecycle Management）策略执行后再删除索引，这样可以确保已删除索引有快照可用。

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "delete": {
        "actions": {
          "wait_for_snapshot" : {
            "policy": "slm-policy-name"    // 名为 slm-policy-name 的 SLM 策略执行完毕后才会删除索引
          }
        }
      }
    }
  }
}
```

### Delete

永久删除索引。如果该索引执行过 Searchable Snapshot 动作且`delete_searchable_snapshot`参数为`false`，则索引被删除后快照依然保留。

```json
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "delete": {
        "actions": {
          "delete" : { }
        }
      }
    }
  }
}
```

## 为索引配置 ILM

在 Elastic Cloud 创建 ILM 策略是非常方便的：[Tutorial: Customize built-in ILM policies](https://github.com/elastic/elasticsearch/edit/7.11/docs/reference/ilm/example-index-lifecycle-policy.asciidoc)。

而将已创建的 ILM 策略关联到索引的方法有以下几种：

- 通过在索引模板中添加 ILM 策略来关联索引

```json
PUT _index_template/my_template
{
  "index_patterns": ["my-index-*"],                           
  "template": {
    "settings": {
      "index.lifecycle.name": "my_ilm_policy",    // 将名称开头为"my-index-"的索引关联到 ILM 策略 my_ilm_policy
      "index.lifecycle.rollover_alias": "test-alias" 
    }
```

- 直接修改现有索引的设置

```json
PUT my-index-*/_settings 
{
  "index": {
    "lifecycle": {
      "name": "my_ilm_policy"    // 将名称开头为"my-index-"的索引关联到 ILM 策略 my_ilm_policy
    }
  }
}
```

- 在创建新索引时直接在设置中指定 ILM 策略

```json
PUT new-index
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "index.lifecycle.name": "my_ilm_policy"    // 将 new-index 关联到 ILM 策略 my_ilm_policy
  }
}
```
