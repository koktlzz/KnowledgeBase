---
title: "Infrastructure"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  elastic:
    parent: "Elasticsearch"
weight: 300
---

## 基本概念与 MySQL 的对比

|ElasticSearch|MySQL|
|----|----|
|Index|Database|
|Type （未来将删除）|Table|
|Document|Row|
|Field|Column|
|Mapping|Schema|
|Everything is indexed|Index|
|Query DSL|SQL|
|GET HTTP|SELECT|
|PUT HTTP|UPDATE|

## 倒排索引

为了实现倒排索引，Elasticsearch 中的每个文档（document）中不仅保存了其自身数据，还包括了以下内容：

- _id 字段：每个文档在索引中的唯一标识；
- 词频 TF：每个 token 在文档中出现的次数，用于相关性评分；
- 位置 Position：token 在文档中分词的位置，用于 match_phrase 类型的查询；
- 偏移量 Offset：记录 token 开始和结束的位置，实现高亮显示。
  
### 创建倒排索引

对文档内容进行分词，形成一个个 token（可以理解为单词），保存 token 和文档_id 之间的映射关系。详见 [Text Analysis](/elastic/elasticsearch/textanalysis)。

### 检索倒排索引

先对检索内容进行分词（适用于 match 查询方法，term/terms 查询不分词），然后在倒排索引中寻找匹配的 token，最后返回 token 对应的文档_id 以及根据文档中 token 匹配情况给出的评分 score。

## 建立索引流程

- 在 Elasticsearch 集群中，节点是对等的。节点间会通过自己的一些规则选取集群的 Master，Master 节点会负责集群状态信息的改变，并同步给其他节点。
- 建立索引的请求先发送到 Master 节点，Master 建立完索引后，将集群状态同步至 Slave 节点。
- 只有建立索引需要经过 Master 节点，而文档可以根据路由规则写入到集群中的任意节点，因此数据的写入压力是分散到整个集群的。

## 分片内文档写入流程

分片（shard）是 Elasticsearch 的最小工作单元，向单一分片内写入文档的流程如下图所示：

![demo.001](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/demo.001.png)

### In-memory Buffer

在主分片节点上，文档会先被写入到内存（In-memory Buffer）中，此时数据还不能被搜索到。

### Refresh

经过一段时间（由`refresh_interval`参数控制）或者内存缓冲满了，Elasticsearch 会将内存中的文档 Refresh 到文件系统缓存（Filesystem Cache）中，并清除内存中的对应文档。

### Filesystem Cache

Lucene 把每次生成的倒排索引，叫做一个段 (Segment)，它无法被修改，只能被合并和删除。另外使用一个 Commit point 文件，记录索引内所有的 Segment。文档在文件系统缓存中被解析为一个个 Segment，同时建立倒排索引。这个时候文档是可以被搜索到的，因此减少了后续写入磁盘的大量时间，体现了 Elasticsearch 搜索的近实时性（下图中的绿色和灰色图标分别代表已写入和未写入磁盘的文档）。

![1](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20210120134908.png)

Segment 会消耗文件句柄、内存和 CPU 资源，并且每次搜索请求都必须轮流检查每一个 Segment。因此随着 Segment 越来越多，将导致查询性能越来越差。这时，ElasticSearch 后台会有一个单独线程专门合并 Segment，将零碎的小的 Segment 合并成一个大的 Segment。

![2](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20210120134633.png)
![3](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20210120134642.png)

### Flush

执行 Lucene 的 Commit 操作，将 Segment 写入磁盘中。更新 Commit point 文件并删除文档对应的 Translog 文件。

### Translog

Elasticsearch 在把数据写入到 Index Buffer 的同时，其实还另外记录了一个 Translog 日志。它是以顺序写文件的形式写入到磁盘中的，速度较快。如果发生异常，Elasticsearch 会从 Commit 位置开始，恢复整个 Translog 文件中的记录，保证数据一致性。

## 多个分片的文档写入

计算方式：
> shard = hash(routing) % number_of_primary_shards

每个文档都有一个 routing 参数，默认情况下就使用其_id 值。将其_id 值计算哈希后，对索引的主分片数取余，就得到了文档实际应该存储到的分片。因此索引的主分片数不可以随意修改，一旦主分片数改变，所有文档的存储位置计算结果都会发生改变，索引数据就完全不可读了。

### 同步副本

![1611125606(1)](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/1611125606(1).jpg)

1. 客户端请求发送给 Node 1 节点，注意图中 Node 1 是 Master 节点，实际完全可以不是；
2. Node 1 用文档的 _id 取余计算得到应该将数据存储到 shard 0 上。通过 Cluster State 信息发现 shard 0 的主分片已经分配到了 Node 3 上。Node 1 转发请求数据给 Node 3；
3. Node 3 完成请求数据的索引过程，存入主分片 0。然后并行转发数据给分配有 shard 0 的副本分片的 Node 1 和 Node 2。当收到任一节点汇报副本分片数据写入成功，Node 3 即返回给初始的接收节点 Node 1，宣布数据写入成功。Node 1 返回成功响应给客户端；
4. 当集群中某个节点宕机，该节点上所有分片中的数据全部丢失（既有主分片，又有副分片）。丢失的副分片对数据的完整性没有影响，丢失的主分片在其他节点上的副分片会被选举成主分片；所以整个索引的数据完整性没有被破坏。

注：图中 P 代表主分片（Primary Shard），R 代表副本（Replica Shard）。

## 根据_id 查询文档

```http
GET /[index]/_doc/[_id]
```

![4](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20210120150752.png)

1. Elasticsearch 集群中的任意节点都可以作为协调（Coordinating）节点接受请求，每个节点都知道集群中任一文档位置；
2. 协调节点对文档的_id 进行路由，从而判断该数据在哪个 Shard，然后将请求转发给对应的节点，此时会使用随机轮询算法，在 Primary Shard 和 Replica Shard 中随机选择一个，从而对请求负载均衡；
3. 处理请求的节点返回文档给协调节点；
4. 协调节点返回文档给客户端。

## 根据字段值检索数据

```http
GET /[index]/_search?q=[field]: [value]
```

![5](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20210120154900.png)

1. Elasticsearch 集群中的任意节点都可以作为协调（Coordinating）节点接受请求，每个节点都知道集群中任一文档位置；
2. 协调节点进行分词等操作后，向所有的 shard 节点发送检索请求；
3. ElasticSearch 已建立字段的倒排索引，即可通过字段值检索到所在文档的_id。随后 Shard 将满足条件的数据（_id、排序字段等）信息返回给协调节点；
4. 协调节点将数据重新进行排序，获取到真正需要返回的文档的_id。协调节点再次向对应的 Shard 发起请求（此时已经有_id 了，可以直接定位到对应的 Shard）;
5. Shard 将_id 对应的文档的完整内容返回给协调节点；
6. 协调节点获取到全部检索结果，返回给客户端。

上述流程和根据_id 查询文档相比，只是多了一个从倒排索引中根据字段值寻找文档_id 的过程，其中的 4~6 步与其完全相同。
