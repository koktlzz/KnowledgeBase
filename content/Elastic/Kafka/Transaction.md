---
title: "事务"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  elastic: 
    parent: "Kafka"
weight: 1200
---

Kafka 引入事务以实现：

- 跨会话或跨 Partition 的 **Exactly-Once**：Producer 将消息发送到多个 Topic/Partition 时，可以封装在一个事务中，形成一个**原子操作**：多条消息要么都发送成功，要么都发送失败。所谓的失败是指消息对事务型 Consumer 不可见，而 Consumer 只读取成功提交事务（Committed）的消息。
- **consume-process-produce** 场景的 **Exactly-Once**：所谓 **consume-process-produce** 场景，是指 Kafka Stream 从若干源 Topic 消费数据，经处理后再发送到目标 Topic 中。Kafka 可以将整个流程全部封装在一个事务中，并维护其原子性：要么源 Topic 中的数据被消费掉且处理后的结果正确写入了目标 topic，要么数据完全没被处理，还能从源 Topic 里读取到。

事务无法实现跨系统的 **Exactly-Once**：如果数据源和目的地不只是 Kafka 的 Topic（如各种数据库），那么操作的原子性便无法保证。

## 启用事务

- 对于 Producer，需将`transactional.id`参数设置为非空，并将`enable.idempotence`参数设置为`true`；
- 对于 Consumer，需将`isolation.level`参数设置为`read_committed`，即 Consumer 只消费已提交事务的消息。除此之外，还需要设置`enable.auto.commit`参数为`false`来让 Consumer 停止自动提交 Offset。

## Producer 事务

Kafka 将事务状态存储在内部的 topic：`__transaction_state`中，并由 Transaction Coordinator 组件维护。每个事务型 Producer 都拥有唯一标识 Transaction ID，它与 [ProducerID](/elastic/kafka/intro/#幂等性) 相绑定。Producer 重启后会向 Transaction Coordinator 发送请求，根据 Transaction ID 找到原来的 ProducerID 以及事务状态。

另外，为了拒绝僵尸实例（Zombie Instance），每个 Producer 还会被分配一个递增的版本号 Epoch。Kafka 会检查提交事务的 Proudcer 的 Epoch，若不为最新版本则拒绝该实例的请求。

>在分布式系统中，一个 instance 的宕机或失联，集群往往会自动启动一个新的实例来代替它的工作。此时若原实例恢复了，那么集群中就产生了两个具有相同职责的实例，此时前一个 instance 就被称为“僵尸实例（Zombie Instance）”。

## Consumer 事务

两种方案：
读取消息 -> 更新（提交 Offset -> 处理消息：若处理消息时发生故障，接管的 Consumer 从更新后的 Offset 读数据，则缺数据，类似于 **At Most Once**
读取消息 -> 处理消息 -> 更新（提交）Offset：若更新 Offset 时发生故障，接管的 Consumer 重新读之前的数据，数据重复，类似于 **At Least Once**
选取第二种方案，因为只要在消息中添加唯一主键，便可以让第二次写入的数据覆盖重复数据，从而做到 **Exactly-Once**

## 参考文献

[Transactions in Apache Kafka](https://www.confluent.io/blog/transactions-apache-kafka/)
