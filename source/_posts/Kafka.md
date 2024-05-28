---
title: Kafka
---
<style type="text/css">
* {font-family: YaHei Consolas Hybrid, Consolas;}
.wrap * {white-space: normal; word-break: break-word;}
</style>

# 参考文章

[Kafka介绍](https://zhuanlan.zhihu.com/p/649938603)

[Kafka 学习笔记](https://my.oschina.net/jallenkwong/blog/4449224#h2_1)

[Kafka为什么吞吐量大、速度快？](https://blog.csdn.net/yangbindxj/article/details/125275597)

[读懂消息队列：Kafka与RocketMQ](https://blog.csdn.net/fuzhongmin05/article/details/105205124)

# 文件布局

![Kafka 文件布局](img/Kafka/Kafka%20%E6%96%87%E4%BB%B6%E5%B8%83%E5%B1%80.png?raw=true)

- `Broker`：Kafka 服务实例。每个服务器上可以有一个或多个 Broker。Kafka 集群内每个 Broker 都有一个唯一的编号。
- `Topic`：消息的主题。一个 Broker 上可以创建多个 Topic。
- `Partition`：Topic 的分区。一个 Topic 可以有多个 Partition，Partition 的作用是支撑**负载均衡**与**提高吞吐量**。同一个 Topic 的不同 Partition 中的数据是不重复的，每个 Partition 的数据存储于一个文件夹中。
- `Replication`：每个 Partition 都有多个副本备份，分布在不同的 Broker 中。当主副本（Leader）故障时，会选择一个备份副本（Follower）作为新的 Leader。备份副本不对外提供读写服务。
- `Consumer Group`：一个消费者组可以包含多个消费者，可以消费多个 Topic。为提高吞吐量，一个消费者组消费的所有 Topic 的所有 Paritition 会被分配给各个消费者，每个 Partition 只会被分配给一个消费者。Kafka 提供了默认分配策略 range 和 round-robin，可自行开发分配策略。
- `Zookeeper`：Kafka 集群依赖 Zookeeper 来保存集群的元信息，保证系统的可用性。

Partition 是实际物理存储结构，每个 Partition 对应的文件夹中可存储多个 Segment，每个 Segment 包含多个文件：

- 0000000000xxxxxxxxxx.log：日志文件，大小达到`log.segment.bytes`后创建新 Segment
- 0000000000xxxxxxxxxx.index：偏移量稀疏索引，相对偏移量 -> 在 log 文件中的物理地址
- 0000000000xxxxxxxxxx.timeindex：时间戳稀疏索引，时间戳 -> 在 index 文件中的相对偏移量

# 消费模型

对于 Kafka 而言，pull 模式更合适，它可简化 Broker 的设计，Consumer 可自主控制消费消息的速率以及消费方式（可批量消费/逐条消费），同时还能选择不同的提交方式从而实现不同的传输语义。

pull 模式不足之处是，如果 Kafka 没有数据，消费者可能会陷入循环，一直等待数据到达。为了避免这种情况，在拉请求中有参数，允许消费者请求在等待数据到达的“长轮询”中进行阻塞（并且可选地等待到给定的字节数，以确保足够的传输数据量）。

# 特点

优点：

- 与周边生态系统的集成和兼容程度非常高
- 消息批量处理，批量压缩，零拷贝收发性能非常高
   - 处理 Producer 发来的消息时，通过 mmap 将数据顺序写入 Partition
   - 处理 Consumer 的读请求时，通过 sendfile 将磁盘文件读到内核缓冲区后，直接转到 socket buffer 进行网络发送

缺点：

- 异步批量的设计导致消息延迟略高，不适合实时性要求高的业务
- 由于每个 Partition 对应一个文件夹，当 Topic 数达到几百时，写文件的 IO 操作就是变得零散，类似于随机 IO，丧失了 Partition 顺序写的优势，性能会大幅下降

# Controller

## 参考文章

[今天想和你聊聊Kafka的Controller（控制器）](https://cloud.tencent.com/developer/article/2005100)

## 说明

Broker 在启动时，会尝试去 Zookeeper 中创建`/controller`节点，第一个创建成功的 Broker 会被指定为控制器。

Controller Broker 的主要职责：

- 创建/删除 Topic，增加 Partition
- 集群 Broker 管理：监听其他 Broker 的新增/主动关闭/故障事件，调整 Leader 分区，更新 ISR
- 元数据同步

# Group Coordinator

## 参考文章

[Kafka的Group Coordinator主要负责什么，消费者选择Coordinator的算法是如何实现](https://www.bilibili.com/read/cv28285172/)

## 说明

Kafka Cluster 会在它的第一个 Consumer 启动时自动创建`__consumer_offset`主题，分区数为`offsets.topic.num.partitions`（默认 50），副本数为`offsets.topic.replication.factor`（默认 3）。`__consumer_offset`中存储键值对：`<group.id, topic, partition> -> <offset, partition_leader_epoch, metadata, timestamp>`。ID 为`group.id`的消费者组中所有 Consumer 的 offset 信息存储于分区号为`hash(group.id) % offsets.topic.num.partitions`的`__consumer_offset`分区，该分区的 Leader 副本所在 Broker 即为该消费者组的 Group Coordinator。

消费者组启动步骤：

1. findCoordinator
   1. 每个 Consumer 向任意 Broker 发送包含其 group.id 的 findCoordinator 请求
   2. Broker 返回包含 group.id 对应的 Group Coordinator Broker 编号的响应
2. joinGroup
   1. Consumer 向 Group Coordinator 发送包含其订阅 Topic 的 joinGroup 请求
   2. Group Coordinator 将收到的第一个 joinGroup 请求的发送者定为 Leader Consumer。Group Coordinator 为每个发送请求的 Consumer 生成 member_id，在响应中返回 member_id、leader_id（Leader Consumer 的 member_id）、members（组中所有 Consumer 及其消费的 Topic，仅 Leader Consumer 会收到此信息）
3. syncGroup
   1. Consumer 向 Group Coordinator 发送包含其 member_id 的 syncGroup 请求，其中 Leader Consumer 发送的请求中还包含其按照本地配置的[分区分配策略](#partition.assignment.strategy)为组内每个 Consumer 制定的分区分配方案
   2. Group Coordinator 按照 member_id 返回包含其分配到的分区的响应
4. 开始消费及心跳

# HW、LEO

## 参考文章

[Kafka HW和LEO](https://www.cnblogs.com/larry1024/p/17593615.html)

## 说明

LEO（log end offset）称为日志末端位移，代表日志文件中下一条待写入消息的 offset。分区 ISR 集合中的每个副本都会维护自身的 LEO。当 Leader 副本收到生产者的一条消息，LEO 通常会自增 1，而 Follower 副本需要从 Leader 副本 fetch 到数据后，才会增加它的 LEO。

HW（high watermark）称为高水位，ISR 集合中最小的 LEO 即为分区的 HW，消费者只能拉取到这个 offset 之前的消息，即成功复制到 ISR 中所有副本的最后一条消息。

位移值大于 HW 的消息对消费者是不可见的，即这些消息不对用户作承诺。当 Leader 副本所在 Broker 下线，发生故障转移时，Follower 副本需要获取新的 Leader 副本的 LEO，若自身的 LEO 大于 Leader 的 LEO，则需要进行日志截断（删除不存在于 Leader 的数据），之后才能从 Leader 拉取数据进行同步。

0.11.0.0 版本引入 leader epoch 机制解决了高水位同步机制的缺陷，即 Leader 切换时可能存在的数据丢失与数据不一致问题。

# 配置（2.8.x）

## 参考文章

[Kafka 相关配置参数](https://www.cnblogs.com/hongdada/p/16920904.html)

[Documentation > Configuration](https://kafka.apache.org/28/documentation.html#configuration)

## Broker

更新模式：

- R：read-only，每个 Broker 自定义的配置，重启 Broker 才能生效
- P：per-broker，每个 Broker 自定义的配置，可动态更新
- C：cluster-wide，可以在每个 Broker 自定义配置，也可批量应用于集群作为默认值，可动态更新

> 格式：
> - 属性名 | 类型/取值范围 | 默认值 | 更新模式 | Topic 级别配置（如果有）\
详细说明
> 
> 【注】
> - 重要属性的**属性名**加粗
> - 部分配置可以被 Topic 级别配置覆盖，相对地，Topic 级别未显式配置时，默认使用对应的 Broker 配置

Broker 自身属性：

- **broker.id** | int | -1 | R\
Broker 的 id。若没有配置此值，且开启了自动生成，则从`reserved.broker.max.id + 1`开始递增生成 id。
- reserved.broker.max.id | int | 1000 | R\
`broker.id`可以配置的最大值。
- broker.id.generation.enable | boolean | true | R\
是否启用`broker.id`自动生成。

Topic 相关：

- num.partitions | $[1, 2^{31} - 1]$ || R | --partitions\
自动创建的 Topic 的默认分区数。
- default.replication.factor | int | 1 | R | --replication-factor\
自动创建的 Topic 的默认副本数，建议配置为3。
- **min.insync.replicas** | $[1, 2^{31} - 1]$ | 1 | C | --config min.insync.replicas\
当 Producer 配置中的`acks`为 all 或 -1 时生效，若某 Partition 的同步中的副本（ISR）数小于此值，则 Producer 向该 Partition 发送消息时将抛出异常。
- message.max.bytes | int | 1048576（1MB） | C | --config max.message.bytes\
允许记录的记录（如果启用了压缩，则指压缩后的记录）大小上限。

Replica 同步相关：

- replica.lag.time.max.ms | long | 30000（30s） | R\
若 Follower 副本没有在指定时间内向 Leader 发送 fetch 请求，或没有跟上 Leader 的 LEO，就将该副本从 ISR 中移出。被移出 ISR 的副本，在同步速度跟上后，可能被重新加入 ISR。
- replica.fetch.max.bytes | int | 1048576（1MB） | R\
副本同步单次拉取的最大字节数。建议设置为不小于`message.max.byte`的值，否则可能导致性能偏低。

线程数相关：

- num.network.threads | $[1, 2^{63} - 1]$ | 3 | C\
用于接收和发送网络数据的线程数。网络线程的处理涉及内存拷贝，吃 CPU，建议设置为`CPU数 + 1`。
- num.io.threads | $[1, 2^{63} - 1]$ | 8 | C\
用于处理请求的线程数，所有写 log 的操作都由该线程触发，建议设置为`CPU数 × 2 + 1`。
- num.replica.fetchers | int | 1 | C\
从 Leader 副本获取数据的线程数，建议设置为`CPU数 + 1`。

日志相关：

- log.segment.bytes | $[14, 2^{63} - 1]$ | 1073741824（1GB） | C | --config segment.bytes\
单个 log 文件大小上限，达到上限后将创建新的 log 文件。
- log.retention.bytes | long | -1（即不限制保留大小） | C | --config retention.bytes\
删除前单个 Partition 目录最多保留的字节数。
- log.retention.hours | int | 168（7d） | R\
log 文件删除前保留时间（小时）。仅在未设置`log.retention.ms`和`log.retention.minutes`时生效。
- log.retention.minutes | int | null | R\
log 文件删除前保留时间（分钟）。仅在未设置`log.retention.ms`时生效。
- log.retention.ms | long | null | R | --config retention.ms\
log 文件删除前保留时间（毫秒）。设置为`-1`代表不限制保留时间。
- log.retention.check.interval.ms | $[1, 2^{63} - 1]$ | 300000（5min） | R\
log 清除任务执行周期。

Consumer 相关：

- group.max.session.timeout.ms | int | 1800000（30min） | R\
注册消费者时允许的最大`session.timeout.ms`。
- group.min.session.timeout.ms | int | 6000（6s） | R\
注册消费者时允许的最小`session.timeout.ms`。
- offsets.retention.minutes | $[1, 2^{31} - 1]$ | 10080（7d） | R\
消费者组中所有 Consumer 都下线后，其 offset 数据保留的时间。

## Producer

> 格式：
> - 属性名 | 类型/取值范围 | 默认值\
详细说明
> 
> 【注】重要属性的**属性名**加粗

- **bootstrap.servers** | list | ""\
Broker 地址，格式`host1:port1,host2:port2,...`。仅用于初始连接以自动发现 Kafka 集群中所有成员身份，所以无须指定所有 Broker，但为避免部分 Broker 宕机，可以指定多个 Broker 地址。
- **key.serializer** | class | \
键序列化器（`org.apache.kafka.common.serialization.Serializer`）实现类。
- **value.serializer** | class | \
值序列化器（`org.apache.kafka.common.serialization.Serializer`）实现类。
- **acks** | string | 1\
Producer 向 Broker 发送消息的持久化机制。取值：
   - `0`：不需要 Broker 的任何确认，直接视为已发送。此配置下，重试配置不生效，且发送结果中的偏移量始终为`-1`
   - `1`：只需要等待 Leader 将数据写入本地 log。此配置下，若 Leader 在*接收记录后且任意 Follower 复制完成之前*失败，则丢失该记录
   - `all`（或`-1`）：需要等待数据写入所有 ISR 的本地 log。此配置是对数据持久性的最强保证
- buffer.memory | long | 33554432（32MB）\
Producer 本地缓冲区大小。
- batch.size | int | 16384（16KB）\
当多个记录被发送到同一个 Partition 时，Producer 会尝试将这些记录合并为一个批次进行发送，批次大小为`batch.size`字节。
- linger.ms | long | 0\
Producer 发送批次之间的最长等待时间，一般设置 10（ms）左右。**批次被发送的条件：批次数据量达到`batch.size`，或批次中最老的数据年龄达到`linger.ms`**。
- request.timeout.ms | int | 30000（30s）\
客户端等待请求响应的时间上限。若客户端超时未接收到响应，会尝试进行重试；若时间（`delivery.timeout.ms`）或重试次数（`retries`）耗尽，则请求失败。
- delivery.timeout.ms | int | 120000（2min）\
调用`send()`后返回（成功或失败）结果的时间上限。该配置限制了发送前等待、发送及重试的总时间，Kafka 建议以该配置取代`retries`配置。该配置值应不小于`linger.ms + request.timeout.ms`
- retries | int | 2147483647\
发送失败重试次数上限。
- retry.backoff.ms | long | 100\
发送失败重试时间间隔（毫秒）。
- max.in.flight.requests.per.connection | $[1, 2^{31} - 1]$ | 5\
客户端可以异步同时发送的请求数量上限。
   - 如果该值大于`1`，则在发送失败重试（如果允许）时可能导致消息顺序改变
   - 此值设置得过小可能导致吞吐量严重降低
- partitioner.class | class | org.apache.kafka.clients.producer.internals.DefaultPartitioner\
分区器（`org.apache.kafka.clients.producer.Partitioner`）实现类，记录中未指定 Partition 时调用该对象来获取目标 Partition。默认实现逻辑：若记录没有 key，则轮询指定 Topic 的分区，否则对 key 进行 hash 后取模算出目标分区。

## Consumer

> 格式：
> - 属性名 | 类型/取值范围 | 默认值\
详细说明
> 
> 【注】重要属性的**属性名**加粗

- **bootstrap.servers** | list | ""\
见 Producer 配置参数。
- **key.deserializer** | class | \
键反序列化器（`org.apache.kafka.common.serialization.Deserializer`）实现类。
- **value.deserializer** | class | \
值反序列化器（`org.apache.kafka.common.serialization.Deserializer`）实现类。
- group.id | string | null\
Consumer 所属的消费者组 id。
- enable.auto.commit | boolean | true\
是否在后台周期性自动提交 offset。自动提交的情况下，若 Consumer 在提交 offset 后，消费消息之前异常崩溃，则会导致消息漏消费。
- auto.commit.interval.ms | int | 5000\
offset自动提交周期（毫秒）。
- max.poll.records | $[1, 2^{31} - 1]$ | 500\
单次调用`poll()`返回的最大记录数。
- max.poll.interval.ms | $[1, 2^{31} - 1]$ | 300000（5min）\
使用消费者组管理时，调用`poll()`之间的最大时间间隔。超时的消费者将被视为处理能力太弱，消费者组会移除该消费者并进行 rebalance，以便将分区重新分配给另一个消费者。
- session.timeout.ms | int | 10000\
Consumer 心跳超时时间（毫秒）。若组协调器在指定时间内未检测到某 Consumer 发送的心跳，则判定该 Consumer 发生故障，将其移出消费者组并 rebalance。
- heartbeat.interval.ms | int | 3000\
Consumer 向组协调器发送心跳的时间间隔，必须小于`session.timeout.ms`，一般设置为不超过`session.timeout.ms`的 1/3。组协调器会对 Consumer 的心跳进行响应，发生 rebalance 时响应中会包含`REBALANCE_IN_PROGRESS`标识。
- auto.offset.reset | string | latest\
当 Kafka 中没有初始 offset（如新建消费者组）或者当前 offset 在服务器上不再存在（如数据被删除）时的处理策略：
   - latest：Consumer 从最新记录开始读取
   - earliest：Consuemr 从头读取所有记录
   - none：Consumer 抛出异常
- <span id="partition.assignment.strategy">partition.assignment.strategy</span> | list | org.apache.kafka.clients.consumer.RangeAssignor\
分区分配策略列表，按优先级从高到低列出，取值：
   - `org.apache.kafka.clients.consumer.RangeAssignor`：将消费者按`member_id`排序，对于每个 Topic，设`n = 分区数 / 消费者数`，则`前分区数 % 消费者数`个消费者消费`n + 1`个分区，其余消费者消费`n`个分区\
   ![Kafka partition.assignment.strategy Range](img/Kafka/Kafka%20partition%20assignment%20strategy%20Range.jpg?raw=true)
   - `org.apache.kafka.clients.consumer.RoundRobinAssignor`：所有主题的所有分区都按轮询的方式分配给 Consumer\
   ![Kafka partition.assignment.strategy RoundRobin](img/Kafka/Kafka%20partition%20assignment%20strategy%20RoundRobin.jpg?raw=true)
   - `org.apache.kafka.clients.consumer.StickyAssignor`：是 RoundRobin 的变种，会尽量保证各消费者分配到数量接近的 Partition；且 rebalance 时会尽量减少分配方案的变化，仅做必要的重分配\
   ![Kafka partition.assignment.strategy Sticky](img/Kafka/Kafka%20partition%20assignment%20strategy%20Sticky.jpg?raw=true)
   - `org.apache.kafka.clients.consumer.CooperativeStickyAssignor`：原理与 Sticky 相同，但使用的是 COOPERATIVE 协议。前面三种策略使用 EAGER 协议，在 rebalance 时会发生全局 STW，而 COOPERATIVE 协议则不会
   - 实现`org.apache.kafka.clients.consumer.ConsumerPartitionAssignor`的自定义分区策略

# 最佳实践

## 防止消息丢失

Broker 配置：

```properties
default.replication.factor=3
# 为提高 Kafka 集群可用性，允许集群在最多一个副本下线时依然可用
min.insync.replicas={default.replication.factor - 1}
```

Topic 配置（自动创建 Topic 时不需要配置）：

    --replication.factor=3 --config min.insync.replicas={replication.factor - 1}

Producer 配置：

```properties
# 需要等待数据写入所有 ISR 的本地 log
acks=all
```

Consumer 配置：

```properties
# 关闭 offset 自动提交，消息消费成功后主动提交
enable.auto.commit=false
```

## Topic 全局顺序消费

Topic 配置：

    --partitions 1 

Kafka 只能保证分区有序，要实现 Topic 全局有序，只能将分区数设置为 1。

Producer 配置：

```properties
# 生产者线程应当保证同步有序生产消息，需要捕获推送失败的情况并做处理
acks=1 或 all
# 禁止异步同时发送多个请求，否则失败重试时可能导致消息顺序改变
max.in.flight.requests.per.connection=1
```

> 【注】Kafka不适合 Topic 全局顺序消费场景：
> - Producer吞吐量低
> - 单分区会限制处理效率，只能有一个 Consumer，且只能单线程消费
> 
> RocketMQ更适用于此场景。

## 解决消息堆积

若消费瓶颈不在 Consumer，则做对应优化。

若消费瓶颈只在 Consumer，且发生消息堆积的 Topic 对全局有序没有要求：

- 若出现分区分配不均的情况，调整 Consumer 配置中的分区分配策略
   ```shell
   # 查看分区分配情况
   bin/kafka-consumer-groups.sh --bootstrap-server <broker-list> --describe --group <consumer-group-id>
   ```
- 若 Consumer 数量少于分区数，导致一个 Consumer 要处理多个 Partition，则增加该消费者组中的 Consumer
- 若 Topic 对分区有序没有要求：
   - 若 Consumer 的网络、CPU 等消费处理所需资源尚未到达瓶颈，可在 Consumer 增加多线程逻辑
   - 可创建一个拥有更多 Partition 的新 Topic，原 Topic 的消费逻辑变更为将消息直接转发到新 Topic，组建消费能力比之前更强的消费者组对新 Topic 使用原消费逻辑进行消费
   - 可在不同的消费者组新增同样逻辑的消费者，在幂等性处理中忽略对已处理消息的重复处理，以加快消费速度
- 若 Topic 对分区有序有要求：
   - 若存在一个不可再拆分的有序集合，任意 Consumer 消费它时都会达到瓶颈：
      - 变更实现方式或业务逻辑
      - 提升 Consumer 的硬件配置
   - 否则设`n = 按照对应粒度可拆分出的最多的有序集合数`，若当前分区数小于`n`：
      - 若 Consumer 的网络、CPU等消费处理所需资源尚未到达瓶颈，可在 Consumer 增加多线程逻辑（同一有序集合中的元素应由同一线程处理）
      - 可创建一个拥有更多（不超过`n`个）Partition 的新 Topic，原 Topic 的消费逻辑变更为将消息直接转发到新 Topic（同一有序集合中的元素应被发送到同一分区），组建消费能力比之前更强的消费者组对新 Topic 使用原消费逻辑进行消费

## 延时队列

Producer 将消息按延迟时间分组发送到对应 Topic，通过 Consumer（`enable.auto.commit=false`）控制消费来达到延时的效果。

例如某 Topic 用于存储延时`n`分钟消费的消息，则 Consumer 消费轮询 poll 得到的消息时，只消费`当前时间 - n`之前创建的消息（消费完成手动提交 offset），忽略剩余消息。

# 常见问题

## 消费效率偏低

表现：

- Consumer 客户端报错：
   <span class="wrap">
   ```log
   org.apache.kafka.clients.consumer.CommitFailedException: Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms, which typically implies that the poll loop is spending too much time message processing. You can address this either by increasing the session timeout or by reducing the maximum size of batches returned in poll() with max.poll.records.
   ```
   </span>
- Broker server.log 日志频繁出现`group xxx offset提交失败`与`(Re-)joining group xxx`

分析：

1. Consumer 无法在`max.poll.interval.ms`时间内消费完一次 poll 拉取的数据，被 Group Coordinator 移除 Consumer Group
2. 发生 rebalance，导致被移出的 Consumer 无法提交 offset
3. 稍后，被移出的 Consumer 又尝试加入消费者组（发生 rebalance）
4. 重复循环

解决方案：

- 增大`max.poll.interval.ms`
- 减小`max.poll.records`

# 性能

## 参考文章

[Kafka集群调优+能力探底](https://www.cnblogs.com/xijiu/p/17878078.html)

[【kafka性能测试脚本详解、性能测试、性能分析与性能调优】](https://blog.csdn.net/qq_45442178/article/details/131172336)

## 说明

下表为 3 Broker 集群压测数据。

流量指标：

- `A`：单副本，`acks=1`，多 Topic 或多 Partition，Broker 压满 CPU 情况下的单机流量
- `B`：3 副本，`acks=all`情况下的单机流量（不含副本同步流量）
- `C`：3 副本，`acks=1`情况下的单机流量

<table>
   <tr>
      <th rowspan="2">机器规格</th>
      <th colspan="3">流量（MB/s）</th>
      <th rowspan="2">单机磁盘写<br />（MB/s）</th>
      <th rowspan="2">峰值磁盘写<br />CPU 占用</th>
   </tr>
   <tr>
      <th>A</th>
      <th>B</th>
      <th>C</th>
   </tr>
   <tr>
      <th>2C4G</th>
      <td>1000</td>
      <td>320</td>
      <td>550</td>
      <td>1105</td>
      <td>200%</td>
   </tr>
   <tr>
      <th>4C8G</th>
      <td>2250</td>
      <td>550</td>
      <td>1100</td>
      <td>2580</td>
      <td>400%</td>
   </tr>
   <tr>
      <th>8C16G</th>
      <td>3100</td>
      <td>950</td>
      <td>1920</td>
      <td>3674</td>
      <td>800%</td>
   </tr>
   <tr>
      <th>16C32G</th>
      <td>未知</td>
      <td>1350</td>
      <td>2720</td>
      <td>4080</td>
      <td>950%</td>
   </tr>
</table>

流量超过`B`时，Follower 将落后于 Leader，等到流量低于`B`时 Follower 才能追赶进度。

> 【注】压测时，若增加 Producer 也无法再提高流量，但 CPU 却无法压到磁盘写峰值时的占用率：
> - 检查 Broker 的线程数相关配置是否合理
> - 增加各 Broker 中的 Partition 数（增加 Partition 或 Topic）

# 版本

## 参考文章

[kafka系列之kafka各个版本的区别](https://blog.csdn.net/qq_21451945/article/details/103085232)

## 说明

各版本中新增功能通常不稳定，尽量避免使用。

- 0.7\
差异：没有副本机制。\
使用：不建议。
- 0.8\
差异：引入副本机制，0.8.2.0 版本引入**新版本 Producer API**（需要指定 Broker 地址）。\
使用：如果不能升级到 0.9，至少升级到 0.8.2.2（该版本开始，**老版本 Consumer API** 比较稳定）。
- 0.9\
差异：增加了基础的安全认证/权限功能，重写了**新版本 Consumer API**，引入了 Kafka Connect 组件用于实现高性能的数据抽取。\
使用：**新版本 Producer API** 比较稳定，可用。
- 0.10\
差异：引入 Kafka Streams。\
使用：如果不能升级到 0.11，至少升级到 0.10.2.2（该版本开始，**新版本 Consumer API** 比较稳定，且修复了一个可能导致 Producer 性能降低的 Bug）。
- 0.11\
差异：提供幂等性 Producer API 以及事务（Transaction）API，对 Kafka 消息格式做了重构。\
使用：消息引擎功能已经相当完善。
- 1.x/2.x\
差异：消息引擎方面未引入新的重大功能特性，主要是 KafKa Streams 方面的改进。
- 3.x\
差异：放弃对 Java8 的支持。
