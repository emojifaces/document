# kafka
## kafka 的特点
1. 可靠性：具有副本及容错机制。
2. 可扩展性：kafka无需停机即可扩展节点及节点上线。
3. 持久性：数据存储到磁盘上，持久性保存。
4. 性能：kafka具有高吞吐量。达到TB级的数据，也有非常稳定的性能。
5. 速度快：顺序写入和零拷贝技术使得kafka延迟控制在毫秒级。

## kafka 主要组件
### broker
kafka集群中包含一个或者多个服务实例（节点），这种服务实例被称为broker（一个broker就是一个节点/一个服务器
### producer 生产者
producer主要是用于生产消息，是kafka当中的消息生产者，生产的消息通过topic进行归类，保存到kafka的broker里面去。
### consumer 消费者
consumer是kafka当中的消费者，主要用于消费kafka当中的数据，消费者一定是归属于某个消费组中的。
### consumer group 消费者组
消费者组由一个或者多个消费者组成，同一个组中的消费者对于同一条消息只消费一次。

每个消费者都属于某个消费者组，如果不指定，那么所有的消费者都属于默认的组。

每个消费者组都有一个ID，即group ID。组内的所有消费者协调在一起来消费一个订阅主题( topic)的所有分区(partition)。当然，每个分区只能由同一个消费组内的一个消费者(consumer)来消费，可以由不同的消费组来消费。

partition数量决定了每个consumer group中并发消费者的最大数量

某一个主题下的分区数，对于消费该主题的同一个消费组下的消费者数量，应该小于等于该主题下的分区数。

分区数越多，同一时间可以有越多的消费者来进行消费，消费数据的速度就会越快，提高消费的性能。
### topic 主题
1. kafka将消息以topic为单位进行归类；
2. topic特指kafka处理的消息源（feeds of messages）的不同分类；
3. topic是一种分类或者发布的一些列记录的名义上的名字。kafka主题始终是支持多用户订阅的；也就是说，一 个主题可以有零个，一个或者多个消费者订阅写入的数据；
4. 在kafka集群中，可以有无数的主题；
5. 生产者和消费者消费数据一般以主题为单位。更细粒度可以到分区级别。
### partition 分区
kafka当中，topic是消息的归类，一个topic可以有多个分区（partition），每个分区保存部分topic的数据，所有的partition当中的数据全部合并起来，就是一个topic当中的所有的数据。

每一个分区内的数据是有序的，但全局的数据不能保证是有序的。
### segment
一个partition当中存在多个segment文件段，每个segment分为两部分，.log文件和 .index 文件，其中 .index 文件是索引文件，主要用于快速查询， .log 文件当中数据的偏移量位置；
### replication 副本
Kafka 保证数据高可用的方式，Kafka 同一 Partition 的数据可以在多 Broker 上存在多个副本，通常只有主副本对外提供读写服务，当主副本所在 broker 崩溃或发生网络一场，Kafka 会在 Controller 的管理下会重新选择新的 Leader 副本对外提供读写服务。
## kafka 数据不丢失机制
### 生产者生产数据不丢失
#### 发送消息方式
生产者发送给kafka数据，可以采用同步方式或异步方式
#### 同步方式
发送一批数据给kafka后，等待kafka返回结果：

1. 生产者等待10s，如果broker没有给出ack响应，就认为失败。
2. 生产者重试3次，如果还没有响应，就报错。
#### 异步方式
发送一批数据给kafka，只是提供一个回调函数：

1. 先将数据保存在生产者端的buffer中。buffer大小是2万条 。
2. 满足数据阈值或者数量阈值其中的一个条件就可以发送数据。
3. 发送一批数据的大小是500条。

#### ack机制（确认机制）
生产者数据发送出去，需要服务端返回一个确认码，即ack响应码；ack的响应有三个状态值0,1，-1

0：生产者只负责发送数据，不关心数据是否丢失，丢失的数据，需要再次发送

1：partition的leader收到数据，不管follow是否同步完数据，响应的状态码为1

-1：所有的从节点都收到数据，响应的状态码为-1

如果broker端一直不返回ack状态，producer永远不知道是否成功；producer可以设置一个超时时间10s，超过时间认为失败。
### broker中数据不丢失 
在broker中，保证数据不丢失主要是通过副本因子（冗余），防止数据丢失。
### 消费者消费数据不丢失 
在消费者消费数据的时候，只要每个消费者记录好offset值即可，就能保证数据不丢失。也就是需要我们自己维护偏移量(offset)，可保存在 Redis 中。

## kafka 中的zookeeper
Broker 注册：Broker 是分布式部署并且之间相互独立，Zookeeper 用来管理注册到集群的所有 Broker 节点。
Topic 注册： 在 Kafka 中，同一个 Topic 的消息会被分成多个分区并将其分布在多个 Broker 上，这些分区信息及与 Broker 的对应关系也都是由 Zookeeper 在维护
生产者负载均衡：由于同一个 Topic 消息会被分区并将其分布在多个 Broker 上，因此，生产者需要将消息合理地发送到这些分布式的 Broker 上。
消费者负载均衡：与生产者类似，Kafka 中的消费者同样需要进行负载均衡来实现多个消费者合理地从对应的 Broker 服务器上接收消息，每个消费者分组包含若干消费者，每条消息都只会发送给分组中的一个消费者，不同的消费者分组消费自己特定的 Topic 下面的消息，互不干扰。

## Kafka Rebalance
rebalance 本质上是一种协议，规定了一个 consumer group 下的所有 consumer 如何达成一致来分配订阅 topic 的每个分区。比如某个 group 下有 20 个 consumer，它订阅了一个具有 100 个分区的 topic。正常情况下，Kafka 平均会为每个 consumer 分配 5 个分区。这个分配的过程就叫 rebalance。
### 什么时候 rebalance？
1. 组成员发生变更（新 consumer 加入组、已有 consumer 主动离开组或已有 consumer 崩溃了——这两者的区别后面会谈到）
2. 订阅主题数发生变更
3. 订阅主题的分区数发生变更
### 如何进行组内分区分配？
Kafka 默认提供了两种分配策略：Range 和 Round-Robin。当然 Kafka 采用了可插拔式的分配策略，你可以创建自己的分配器以实现不同的分配策略。

## kafka 推拉模式
push-and-pull

Kafka Producer 向 Broker 发送消息使用 Push 模式，Consumer 消费采用的 Pull 模式。拉取模式，让 consumer 自己管理 offset，可以提供读取性能

## 分区与副本
在分布式数据系统中，通常使用分区来提高系统的处理能力，通过副本来保证数据的高可用性。多分区意味着并发处理的能力，这多个副本中，只有一个是 leader，而其他的都是 follower 副本。仅有 leader 副本可以对外提供服务。 多个 follower 副本通常存放在和 leader 副本不同的 broker 中。通过这样的机制实现了高可用，当某台机器挂掉后，其他 follower 副本也能迅速”转正“，开始对外提供服务。
### 为什么 follower 副本不提供读服务？
这个问题本质上是对性能和一致性的取舍。试想一下，如果 follower 副本也对外提供服务那会怎么样呢？首先，性能是肯定会有所提升的。但同时，会出现一系列问题。类似数据库事务中的幻读，脏读。 比如你现在写入一条数据到 kafka 主题 a，消费者 b 从主题 a 消费数据，却发现消费不到，因为消费者 b 去读取的那个分区副本中，最新消息还没写入。而这个时候，另一个消费者 c 却可以消费到最新那条数据，因为它消费了 leader 副本。Kafka 通过 WH 和 Offset 的管理来决定 Consumer 可以消费哪些数据，已经当前写入的数据。
### 只有 Leader 可以对外提供读服务，那如何选举 Leader
kafka 会将与 leader 副本保持同步的副本放到 ISR 副本集合中。当然，leader 副本是一直存在于 ISR 副本集合中的，在某些特殊情况下，ISR 副本中甚至只有 leader 一个副本。 当 leader 挂掉时，kakfa 通过 zookeeper 感知到这一情况，在 ISR 副本中选取新的副本成为 leader，对外提供服务。 但这样还有一个问题，前面提到过，有可能 ISR 副本集合中，只有 leader，当 leader 副本挂掉后，ISR 集合就为空，这时候怎么办呢？这时候如果设置 unclean.leader.election.enable 参数为 true，那么 kafka 会在非同步，也就是不在 ISR 副本集合中的副本中，选取出副本成为 leader。
### 副本的存在就会出现副本同步问题
Kafka 在所有分配的副本 (AR) 中维护一个可用的副本列表 (ISR)，Producer 向 Broker 发送消息时会根据ack配置来确定需要等待几个副本已经同步了消息才相应成功，Broker 内部会ReplicaManager服务来管理 flower 与 leader 之间的数据同步。
## 性能优化
- partition 并发
- 顺序读写磁盘
- page cache：按页读写
- 预读：Kafka 会将将要消费的消息提前读入内存
- 高性能序列化（二进制）
- 内存映射
- 无锁 offset 管理：提高并发能力
- Java NIO 模型
- 批量：批量读写
- 压缩：消息压缩，存储压缩，减小网络和 IO 开销

## 问题
什么是分布式消息中间件？通信，队列，分布式，生产消费者模式。

消息中间件的作用是什么？ 解耦、峰值处理、异步通信、缓冲。

消息中间件的使用场景是什么？ 异步通信，消息存储处理。

消息中间件选型？语言，协议、HA、数据可靠性、性能、事务、生态、简易、推拉模式。

Kafka 有哪些命令行工具？你用过哪些？/bin目录，管理 kafka 集群、管理 topic、生产和消费 kafka

Kafka Producer 的执行过程？拦截器，序列化器，分区器和累加器

Kafka Producer 有哪些常见配置？broker 配置，ack 配置，网络和发送参数，压缩参数，ack 参数

如何让 Kafka 的消息有序？Kafka 在 Topic 级别本身是无序的，只有 partition 上才有序，所以为了保证处理顺序，可以自定义分区器，将需顺序处理的数据发送到同一个 partition

Producer 如何保证数据发送不丢失？ack 机制，重试机制

如何提升 Producer 的性能？批量，异步，压缩

如果同一 group 下 consumer 的数量大于 part 的数量，kafka 如何处理？多余的 Part 将处于无用状态，不消费数据

Kafka Consumer 是否是线程安全的？不安全，单线程消费，多线程处理

讲一下你使用 Kafka Consumer 消费消息时的线程模型，为何如此设计？拉取和处理分离

Kafka Consumer 的常见配置？broker, 网络和拉取参数，心跳参数

Consumer 什么时候会被踢出集群？奔溃，网络异常，处理时间过长提交位移超时

当有 Consumer 加入或退出时，Kafka 会作何反应？进行 Rebalance

什么是 Rebalance，何时会发生 Rebalance？topic 变化，consumer 变化

Kafka 如何保证高可用？

通过副本来保证数据的高可用，producer ack、重试、自动 Leader 选举，Consumer 自平衡

Kafka 的交付语义？

交付语义一般有at least once、at most once和exactly once。kafka 通过 ack 的配置来实现前两种。

Replic 的作用？

实现数据的高可用

什么是 AR，ISR？

AR：Assigned Replicas。AR 是主题被创建后，分区创建时被分配的副本集合，副本个 数由副本因子决定。

ISR：In-Sync Replicas。Kafka 中特别重要的概念，指代的是 AR 中那些与 Leader 保 持同步的副本集合。在 AR 中的副本可能不在 ISR 中，但 Leader 副本天然就包含在 ISR 中。关于 ISR，还有一个常见的面试题目是如何判断副本是否应该属于 ISR。目前的判断 依据是：Follower 副本的 LEO 落后 Leader LEO 的时间，是否超过了 Broker 端参数 replica.lag.time.max.ms 值。如果超过了，副本就会被从 ISR 中移除。

Leader 和 Flower 是什么？

Kafka 中的 HW 代表什么？

高水位值 (High watermark)。这是控制消费者可读取消息范围的重要字段。一 个普通消费者只能“看到”Leader 副本上介于 Log Start Offset 和 HW（不含）之间的 所有消息。水位以上的消息是对消费者不可见的。

Kafka 为保证优越的性能做了哪些处理？

partition 并发、顺序读写磁盘、page cache 压缩、高性能序列化（二进制）、内存映射 无锁 offset 管理、Java NIO 模型


