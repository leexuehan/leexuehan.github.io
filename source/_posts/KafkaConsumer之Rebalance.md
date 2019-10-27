---
title: KafkaConsumer之Rebalance
date: 2019-08-15 23:03:17
tags: Kafka
categories: 技术
---

本文主要就 Kafka 的消费者端的分区分配策略进行简单探讨。

<!--more-->

## 分区策略

分组订阅机制算是 Kafka 消息引擎的一大特点了。

简言之，就是消费者与消费者可以组成一个 Group 去消费一个 Topic 的消息。

其示意图如下。

<div align=center>![consumer](./consumer.png)

从图中可以看出：生产者将消息发送到指定 Topic 的 Partition 之后，消费者会以**组**为单位，对每个 Partition 进行消费，也就是说：**每个 Partition 中的消息必然会到达每个消费者组**。

上图中会产生一个疑问：Consumer Group1 中有两个消费者： Consumer1 和 Consumer2。而 Partition1到Partition4的消息都会到达这个组。这样会产生一个现象：`2` 个消费者消费 `4` 个 Partition 中的消息。

那么它们的分配方式是什么呢？分区策略又是在哪里执行的呢，是 Kafka Consumer 端还是 Kafka Broker 端呢？

先回答第一个问题：Kafka 的分区策略主要分为两种：Range 和 RoundRobin。

### Range

Range 策略的执行步骤如下：

1. 对同一 Topic 下的 Partition 按照序号进行排序，以上图为例，则排好序的分区序号依次为：0、1、2、3.

2. 对 Group 里面的消费者也按照序号排序，以上图为例，则排好序的消费者序号依次为：0、1。

3. Consumer的个数为 2，Partition 的个数为 4，所以按照算法 4/2 = 2，每个消费者消费 2 个 Partition。注意是连续的两个 Partition： Consumer0 消费 Partition-0 和 Partition-1，Consumer1 消费 Partition-2 和 Partition-3.

   【注】：如果不能整除怎么办？答：还是顺序消费，多出来的 Partition 会从头开始顺延派发。假如 Partition 的个数为 5， Consumer 的个数为 2， 则 Consumer0 会消费 Partition：0、1、2，Consumer1会消费 Partition：3、4。以此类推。

### RoundRobin

将所有的分区组成TopicAndPartition 列表，然后依照 hashcode 进行排序。以上图为例，假如排好序的 TopicAndPartition 列表如下：

T-3

T-1					

T-2

T-0

则 Consumer0 会消费 T-3 和 T-2，Consumer1 会消费 T-1 和 T-0。

与 Range 消费连续 Partition 不同， RoundRobin 采用类似“等间隔采样”的思想。

### 指定 Partition

除了以上两种策略外，在 Consumer 端可以**消费指定的** Partition。但是注意：目前还不能指定 Partition 的分配策略。

以上解答了第一个问题：Partition 与 Consumer 的分配关系。

第二个问题：Partition 策略在哪儿实现的？是在服务端完成的还是客户端完成的？

这就涉及到了 Rebalance 机制。

## Rebalance 机制

关于 Rebalance 机制，Kafka 的设计先后经历了三种方案。

### 方案一

方案一是采用在 zookeeper 上添加 Watcher 来实现的，如图。

<div align=center>![zookeeper](./zookeeper.png)

- /brokers/ids/broker 记录了host、port 以及在此 broker 上的 topic 分区列表
- /broker/topics/[topic_name] 记录了每个Partition的Leader和 ISR 信息
- /consumer/group_id/ids 记录了消费者的信息
- /consumer/group_id/offsets 记录了 consumer group 在某个 partition 上的消费位置
- /consumer/group_id/owners 记录了 partition 与 consumer 的对应关系。

可以看出来，**消费者通过监听 zookeeper 路径下的变化便可感知到消费者组的变化情况**。

但是这个方案有两个问题：

1. “羊群效应”

   任何 consumer 或者 broker  的加入都会触发 Watcher 通知 consumer，导致大量的通知被发送给客户端。

2. 脑裂问题

   由于 zookeeper 只保证最终一致性。所以当不同的 consumer 连接不同的 zookeeper 节点的时候，会造成短时间内看到的元数据不一样，引起不正确的 Rebalance 尝试。

### 方案二

针对方案一中的两个问题，方案二分别作出了改进。

原来的羊群效应是因为发送的通知太多，在方案二里引入了一个代理，代理监听 zookeeper 节点的变化，节点变化后只要通知代理即可，这个代理就是 GroupCoordinator。

在本方案里，所有的 Consumer 组被划分成为几个子集，每个子集由一个 GroupCoordinator 代理。

此时就变成了：Consumer 依赖 GroupCoordinator， GroupCoordinator 依赖 zookeeper。

**Consumer 加入或者退出 Group 时，还是会修改 Zookeeper 上保存的元数据，但是此时 zookeeper 只发送通知给 GroupCoordinator，由 GroupCoordinator 进行 Rebalance 操作。**

下面简单图示出来，一个 Consumer 加入一个 Group 的过程：

<div align=center>![JoinGroup](./JoinGroup.png)

1. Consumer 向 Kafka Broker 集群中的任一节点**获取元数据**请求，请求中包含了自身所在的 GroupId，节点收到请求之后，返回的响应中就包含了 GroupCoordinator  的信息；
2. Consumer 向 GroupCoordinator  周期性发送 HeartbeatRequest 请求，目的是告诉 GroupCoordinator 自己正常在线。
3. GroupCoordinator 通过在 HeartbeatResponse 中封装 IllegalGeneration 异常告诉 Consumer，发起Rebalance 操作；
4. Consumer 向 GroupCoordinator 发送 JoinGroupRequest **加入 Group **。
5. GroupCoordinator 根据 zookeeper 存储的元数据信息和请求制定分配方案，并将方案写入 zookeeper 中
6. GroupCoordinator 将制定好的分区分配方案写入 JoinGroupResponse 中，返回到 Consumer 。
7. Consumer 解析收到的响应中包含的 GroupCoordinator 已经制定好的分区方案，按照分区开始消费
8. 继续执行步骤 2.

方案的时序图如下所示：

<div align=center>![方案二执行时序图](./Solution2.png)

但是此方案也有其缺点： 分区策略由服务端的 GroupCoordinator  来制定，一旦需要更改分区策略，就得重启服务器，不太灵活。

### 方案三

针对方案二由服务器端制定分区分配方案这一不灵活的缺陷，方案三改为由 Consumer 端制定分区分配方案。

大体来说，Consumer 加入一个组总共分为两阶段：JoinGroup 阶段和 Synchronizing Group State 阶段。

#### JoinGroup 阶段

当 Consumer 请求加入组时，它会向 GroupCoordinator  发送 JoinGroup 请求，在该请求中包含了 Consumer 订阅的 Topic。

收集好全部成员的 JoinGroup 请求之后，GroupCoordinator  会从这些 Consumer 中选择一个担任这个 Group 的 Leader。

通常情况下：第一个发送 JoinGroup 请求的会自动称为 Group Leader。

选出来 Leader 之后，GroupCoordinator   会将 Consumer 的全部订阅信息封装到 JoinGroup 响应中，发送给 Group Leader，由 Group Leader 制定分区方案；其它 Consumer 也会收到 JoinGroup 响应，但是响应中没有订阅信息。

此时进入下一个阶段。

#### Synchronizing Group State 阶段

在此阶段，Group Leader 向 GroupCoordinator 发送 SyncGroup 请求，请求中封装了制定好的分区方案；而其他 Consumer 发送的 SyncGroup 请求中并没有实际内容。

当 Group 内所有成员都成功接收到分配方案后，Consumer 即开始正常消费消息。

#### 典型场景

下面从 Broker 端的角度开始分析几个主要的场景的流程：

场景一：新成员入组

组处于 stable 状态时，如果有新成员加入。

<div align=center>![新成员加入](./member_add.png)

场景二：组成员主动离组

Consumer 实例所在的进程或者线程调用 close 方法主动通知 GroupCoordinator 它要退出。

<div align=center>![主动离组](./member_leave.png)

场景三：组成员崩溃离组

Consumer 实例发生了严重故障，突然宕机导致离组。此时 GroupCoordinator 需要等待超时才能感知。

<div align=center>![崩溃离组](./member_crush.png)

场景四：重平衡时协调者对组内成员提交位移的处理。

每个组内成员都会定期汇报位移给 GroupCoordinator ，当 Rebalance 开启时，GroupCoordinator 会给予成员一段缓冲时间，要求每个成员必须在这段时间快速上报自己的位移信息，然后开启正常 JoinGroup/SyncGroup 请求发送。

<div align=center>![提交位移](./offset_commit.png)

#### 如何找到 GroupCoordinator

方案三中 Consumer 不再需要从 Broker 节点中获取 GroupCoordinator 所在的位置信息。每个 Consumer Group 都有其对应的 GroupCoordinator，但具体是由哪个 GroupCoordinator 负责与 group.id 的 hash 值有关，通过这个 **abs(GroupId.hashCode()) % NumPartitions** 来计算出一个值（其中，NumPartitions 是 `__consumer_offsets` 的 partition 数，默认是50个），这个值代表了 `__consumer_offsets` 的一个 partition，而这个 partition 的 leader 即为这个 Group 要交互的 GroupCoordinator 所在的节点。

## Rebalance 缺点

Rebalance 的缺点就在于消费者组内实例数多的情况下，会造成一次 Rebalance 比较耗时。而且 Rebalance 时，所有的 Consumer 需要放下正在进行的消费工作，等待分配方案分配完成。

所以针对此，最好的优化就是：尽量避免 Rebalance 的发生。

（全文完）



## 参考资料

1. 《Apache Kafka 源码剖析》徐郡明
2. 《Kafka核心技术与实战》胡夕
3. [https://www.iteblog.com/archives/2209.html](https://www.iteblog.com/archives/2209.html)