---
title: "Kafka 简介"
date: 2020-11-04T09:19:42+01:00
lastmod: 2020-11-04T09:19:42+01:00
draft: false
menu:
  observability: 
    parent: "ElasticStack"
weight: 1000
---

## 架构

![202103212048](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/202103212048.jpeg)

## 概念

- Broker：Kafka 集群中的一台或多台服务器；
- Topic：逻辑概念。根据消息的类型，将其分为各种主题（Topic），以此区分不同的业务数据；
- Partition：物理概念。每个 Topic 可分为多个分区（Partition），而每个 Partition 都是有序且顺序不变的消息队列；
- Offset：每条消息都会被分配一个连续的、在其 Partition 内唯一的标识来记录顺序，即偏移量（Offset）；

![202103211905](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/202103211905.png)

- Replica：可以为每个 Partition 创建副本（Replica）来实现高可用；
- Leader/Follwer：一个 Leader 副本处理读写请求，多个 Follower 副本同步数据。每台 Broker 上都维护着某些 Partition 的 Leader 副本和某些 Partition 的 Follower 副本，因此集群的负载是均衡的；
- Producer：消息的生产者，将数据主动发送到指定的 Topic；
- Consumer：消息的消费者，从订阅的 Topic 中主动拉取数据；
- Consumer Group：将多个 Consumer 划分为组，组内的 Consumer 可以并行地消费 Topic 中的数据；
- Zookeeper：Kafka 集群中的一个 Broker 会被选举为 Controller，负责管理集群中其他 Broker 的上下线、Partition 副本的分配和 ISR 成员变化、Leader 的选举等工作。而 Controller 的管理工作依赖于 Zookeeper，Broker 必须能通过 Zookeeper 的心跳机制维持其与 Zookeeper 的会话。

## 基本配置

```bash
[root@test-ece-kafka2 kafka_2.11-1.1.1]# ls config/
connect-console-sink.properties    connect-file-sink.properties    connect-standalone.properties  producer.properties     zookeeper.properties
connect-console-source.properties  connect-file-source.properties  consumer.properties            server.properties
connect-distributed.properties     connect-log4j.properties        log4j.properties               tools-log4j.properties
```

Kafka 的基本配置在安装目录中 config 下的 server.properties 文件中：

```json
# The id of the broker. This must be set to a unique integer for each broker.
broker.id=2

# A comma separated list of directories under which to store log files
log.dirs=/data/kafka/logs

# The minimum age of a log file to be eligible for deletion due to age
log.retention.hours=72

# A size-based retention policy for logs. Segments are pruned from the log unless the remaining
# segments drop below log.retention.bytes. Functions independently of log.retention.hours.
#log.retention.bytes=1073741824

# The maximum size of a log segment file. When this size is reached a new log segment will be created.
log.segment.bytes=1073741824

# The interval at which log segments are checked to see if they can be deleted according
# to the retention policies
log.retention.check.interval.ms=300000

# zookeeper cluster
zookeeper.connect=test-ece-zk1:2181,test-ece-zk2:2181,test-ece-zk3:2181
```

## 常用命令

启动

```bash
bin/kafka-server-start.sh config/server.properties
```

创建 Topic

```bash
bin/kafka-topics.sh --create --zookeeper <zookeeper-server> --replication-factor <n> --partitions <n> --topic <topic-name>
```

查看 Topic

```bash
bin/kafka-topics.sh --list --zookeeper <zookeeper-server>
```

```bash
bin/kafka-topics.sh --describe --zookeeper <zookeeper-server> --topic <topic-name>
```

发送消息

```bash
bin/kafka-console-producer.sh --broker-list <broker> --topic <topic-name>
```

消费消息

```bash
bin/kafka-console-consumer.sh --bootstrap-server <bootstrap-server> --topic <topic-name> --from-beginning
```

查看消费者 Offset 与积压（lag）

```bash
bin/kafka-consumer-groups.sh  --describe --bootstrap-server <bootstrap-server>  --group <counsumer-group>
```

## Producer

消息的生产者，将数据主动发送到指定的 Topic。

- 由于只有 Leader 副本处理读写请求，因此 Producer 会将消息发送到 Leader 所在的 Broker 上。为了实现这一功能，所有 Broker 都能响应其请求：哪些 Broker 存活（alive）和 Partition 中的 Leader 在哪台 Broker 上。
- 为了提升性能，Producer 会尝试在内存中汇总数据，并用一次请求批量发送消息。这种处理方式不仅可以指定批量发送的消息数量，也可以指定等待的延迟时间（如 10ms)，这将允许汇总更多的数据后再发送，从而减少在 Broker 端的 IO 操作。

## Partition

Kafka 中的每个 Topic 可分为多个分区（Partition），这样做的目的是：

- 水平扩展：每个单独的 Partition 受限于其所在 Broker 的文件限制，因此可以通过增加 Partition 数量来增大数据量；
- 负载均衡：Partition 由多台 Broker 维护，并发处理请求从而分担读写压力。

Producer 只关心消息发往哪个 Topic，至于消息具体发送到哪一个 Partition 是由分配策略决定的：

- 当指定 Partition 的情况下，直接分配到对应的 Partition；
- 没有指定 Partition 但指定了消息的键值 key，将使用 key 的 hash 值与 Topic 的 Partition 数进行取余得到 Partition 值，即`hash(key) % numPartitions`。因此我们可以指定用户 id 作为 key，那么跟用户有关的所有数据都将发送到同一 Partition 中。；
- 若两者都没有指定，第一次分配消息时会随机生成一个整数（之后再次分配会自增），将此值 Partition 数进行取余得到 Partition 值（即 Round-Robin 算法）。

Partition 中的数据是直接写入磁盘的，其过程是将消息一直追加到文件末端，这样便可以省去大量磁头寻址的过程。

>相比于维护尽可能多的 in-memory cache，并且在空间不足的时候匆忙将数据 flush 到文件系统，我们把这个过程倒过来。所有数据一开始就被写入到文件系统的持久化日志中，而不用在 cache 空间不足的时候 flush 到磁盘。实际上，这表明数据被转移到了内核的 pagecache 中。

Partition 在底层被拆分成了一个个 segment，而每个 segment 则由一个。log 文件和一个。index 文件组成：

```bash
[root@test-ece-kafka2 ~]# ls /data/kafka/logs/<topic-partition_number> 
00000000000000083456.index  00000000000000083456.snapshot   00000000000000126371.index  00000000000000126371.snapshot   leaderer-epoch-checkpoint
00000000000000083456.log    00000000000000083456.timeindex  00000000000000126371.log    00000000000000126371.timeindex
```

- 当起始。log 文件大小超过 Kafka 配置文件中`log.segment.bytes`参数指定的值，就会创建新的。log 文件，即新的 segment；
- .index 和。log 文件以当前 segment 的第一条消息的 Offset 命名；
- .index 文件中保存了每条消息的 Offset 值、在。log 文件中存储的物理偏移地址和大小，因此 Consumer 在消费时可以很快的根据。index 文件找到指定消息在。log 文件中的位置。具体过程如下：Consumer 将当前消费消息的 Offset 值与。index 文件名中的数字对比，找到该消息所在的。index 文件。随后在。index 文件中找到该消息在。log 文件中的起始地址，并根据消息大小确定其在。log 文件中的终止地址，从而拿到完整的目标消息。
- leaderer-epoch-checkpoint 文件保存了 Partition 的 Leader Epoch 信息，详见 [Leader Epoch](/elastic/kafka/hw/#leader-epoch)。

## Consumer

Consumer 采用主动拉取（Pull）的方式从 Broker 中读取数据，因此可以根据 Consumer 的消费能力以适当的速率消费数据。这种方式与 Broker 主动向 Consumer 推送（Push）消息相比，可以防止 Consumer 因不堪重负而出现拒绝服务、网路堵塞等问题。但如果 Topic 中已经没有数据，Consumer 依然会向 Broker 不断发起 Pull 请求。为此 Kafka 引入了一个时长参数`timeout`，即如果当前没有数据可供消费，Consumer 会等待一段时间之后再重新开始拉取数据。

![202103211935](https://cdn.jsdelivr.net/gh/koktlzz/ImgBed@master/202103211935.jpeg)

每个 Topic 中的一个 Partition 只能被一个 Consumer Group 中的一个 Consumer 消费，而 Consumer 到底消费哪一个 Partition 是由 Consumer 客户端的`partition.assignment.strategy`参数，即 Partition 的分配策略所决定的。Kafka 目前支持三种分配策略：

- Range：参数值为`org.apache.kafka.clients.consumer.RangeAssignor`。对于每一个 Topic，RangeAssignor 将 Consumer Group 内所有订阅该 Topic 的 Consumer 名称按照字典顺序排序，然后为每个 Consumer 平均分配 n 个 Partition（n=Partition 数/Consumer 数）。m 个多余的 Partition（m=Partition 数%Consumer 数）将会按序分配给字典顺序靠前的 Consumer；
- RoundRobin：参数值为`org.apache.kafka.clients.consumer.RoundRobinAssignor`。RoundRobinAssignor 将 Consumer Group 内所有 Consumer 以及 Consumer 订阅的所有 Topic 中的 Partition 按照字典顺序排序，然后通过轮询 Consumer 方式逐个将 Partition 分配给每个 Consumer；
- Sticky：参数值为`org.apache.kafka.clients.consumer.StickyAssignor`。StickyAssignor 除了要保证 Partition 均匀分配之外，还会尽可能保证集群变动前后多次 Partition 分配的结果相同。

详见：[Kafka Range、RoundRobin、Sticky 三种分区分配策略区别](https://blog.csdn.net/u010022158/article/details/106271208)

由于 Consumer 在消费过程中可能会出现断电宕机等故障，当其恢复后需要从故障前的位置的继续消费，因此 Consumer 需要实时记录当前消费的 Offset。而 Offset 保存在 Kafka 的内置 Topic 中，即_consumer_offsets。

> 在每一个消费者中唯一保存的元数据是 offset（偏移量）即消费在 log 中的位置。偏移量由消费者所控制：通常在读取记录后，消费者会以线性的方式增加偏移量，但是实际上，由于这个位置由消费者控制，所以消费者可以采用任何顺序来消费记录。例如，一个消费者可以重置到一个旧的偏移量，从而重新处理过去的数据；也可以跳过最近的记录，从"现在"开始消费。
>

## 数据可靠性

Kafka 通过以下机制来保证 Producer 向 Partition 发送数据的可靠性：

- 每个 Partition 在接收到 Producer 发送的消息后，都要通过其所在的 Broker 返回一个 ACK（Acknowledge）数据包。Producer 收到该数据包后才会继续发送消息，否则重新发送消息；
- Leader 中维护了一个动态的 ISR（in-sync replica set）, 即与 Leader 保持同步的 Follower 集合。当 ISR 中的 Follower 完成数据同步后，也会给 Leader 发送 Ack 数据包。若在参数`replica.lag.time.max.ms`规定的时间内未返回 Ack 数据包，该 Follower 将被踢出 ISR；
- `min.insync.replicas`参数指定了 ISR 中的最小副本数，默认值为 1。

Kafka 的吞吐量和可靠性是不可兼得的，我们可以通过调整 ACK 参数来对其进行权衡。对于某些不太重要的数据，其可靠性要求不是很高，能够容忍数据的少量丢失，因此没必要等 ISR 中的 Follower 全部同步完成。

- ACK=0：即使数据还未在某个 Partition 的 Leader 上落盘，Producer 也会认为消息发送成功，不再等待 Broker 返回的 ACK 数据包。若 Leader 所在的 Broker 发生故障，则有可能丢失数据；
- ACK=1（默认值）：只要 Leader 接收到 Producer 发送的消息就返回 ACK 数据包，Producer 认为消息发送成功。如果在 Follower 同步数据前，Leader 所在的 Broker 发生故障，则有可能丢失数据。如果 Follower 同步数据成功，但返回的 ACK 数据包发送失败，Producer 会再次发送相同的消息，从而造成数据重复。
- ACK=-1（all）：只有 ISR 中的所有副本均同步完成，Leader 才会返回 ACK 数据包，Producer 认为消息发送成功。若`min.insync.replicas`值为 1 且 ISR 中只有 Leader 副本，那么即使设置 ACK=-1，也和 ACK=1 的情况相同。

## 幂等性

我们已经可以通过调整 ACK 参数来保证数据不丢失 **At Least Once**（ACK=-1）或者不重复 **At Most Once**（ACK=0）。但对于一些重要信息，如交易数据，Consumer 要求数据既不丢失也不能重复。Kafka 引入幂等性的特性来保证无论 Producer 发送多少条重复数据到 Partition 中，Consumer 都只会消费一条有效信息，即 **Exactly-Once**。要启用幂等性，只需将 Producer 参数中`enable.idompotence`参数设置为 true 即可。为了实现幂等性，Kafka 在底层设计架构中引入了 ProducerID 和 SequenceNumber 两个概念：

- ProducerID：在每个新的 Producer 初始化时，会被分配一个唯一的 ProducerID，这个 ProducerID 对客户端使用者是不可见的；
- SequenceNumber：对于每个 ProducerID，Producer 发往同一 Partition 的不同数据都分别对应了一个从 0 开始单调递增的 SequenceNumber 值。

当 Producer 发送消息 (x2,y2) 给 Broker 时，Broker 接收到消息并将其追加到消息流中。此时，Broker 返回 ACK 信号给 Producer 时，发生异常导致 Producer 接收 ACK 信号失败。对于 Producer 来说，会触发重试机制，将消息 (x2,y2) 再次发送，但是，由于引入了幂等性，在每条消息中附带了 PID（ProducerID）和 SequenceNumber。相同的 PID 和 SequenceNumber 发送给 Broker，而之前 Broker 缓存过之前发送的相同的消息，那么在消息流中的消息就只有一条 (x2,y2)，不会出现重复发送的情况。

![20210304135205](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20210304135205.png)

由于 Producer 重启后 ProducerID 会发生改变，而不同的 Partition 也会有不同的 SequenceNumber，因此 Kafka 无法保证跨会话或跨 Partition 的幂等性。

## 参考文献

[Kafka for Engineers](https://levelup.gitconnected.com/kafka-for-engineers-975feaea6067)
