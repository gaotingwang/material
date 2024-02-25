Kafak 是一个分布式的基于发布订阅的消息队列，也是一个分布式事件流平台：数据管道、数据集成、流分析

kafka 具有高吞吐、高性能、实时性高可靠等特点

部署启动
---

kafka的部署依赖zk，需要先有zk服务

```shell
$ cd kafka_2.13-3.6.1
$ bin/kafka-server-start.sh -daemon config/server.properties
```



## Topic 和 Partion

![](https://gtw.oss-cn-shanghai.aliyuncs.com/Kafka/Topic%20VS%20ConsumerGroup.png)

关于topic操作：创建、删除、修改、查看：

```shell
$ ./kafka-topics.sh 
--bootstrap-server broker的地址：hostname:port
--create 创建topic --topic topic名称 --config <String: name=value> topic配置
--delete 删除topic
--alter 修改topic
--list 展示所有能用的topic
--describe 查看topic详情
--partitions 分区数

# 创建
$ ./kafka-topics.sh --bootstrap-server hadoop000:9092 --create --topic test-topic --partitions 1
# 查看
$ ./kafka-topics.sh --bootstrap-server hadoop000:9092 --list
$ ./kafka-topics.sh --bootstrap-server hadoop000:9092 --describe --topic test-topic
```



## 核心API解读

### Broker

相关配置：

```properties
# 必要配置
broker.id
log.dirs
zookeeper.connect

# 自动创建topic
auto.create.topics.enable = true
# 是否可以删除topic
delete.topic.enable = true
# 数据文件保留多少h(7天)
log.retention.hours = 168
# 单个文件大小，默认 1G
log.segment.bytes = 1073741824
```

### Producer

采用批量发送的方式，可以指定发送间隔`linger_ms`和每批次发送数据量`batch_size`

- 异步发送
- 阻塞发送（获取每次发送结果）
- 异步回调发送

`acks` ：

为0：表示producer不需要等待任何确认收到的信息。副本将立即加到socket buffer并认为已经发送。没有任何保障可以保证此种情况下 server已经成功接收数据，同时重试配置不会发生作用（因为客户端不知道是否失败）回馈的offset会总是设置为－1；
为1：意味着至少要等待leader已经成功将数据写入本地log，但并没有等待所有follower是否写入成功。这种情况下，如果 follower 没有成功备份数据，而此时leader又挂掉，则消息会丢失；
为all：意味着leader需要等待所有备份都成功写入日志，这种策略会保证只要有一个备份存活就不会丢失数据。这是最强的保证。

消息传递保障：

- 最多一次（0~1）
- 至少一次（1~n）
- 正好一次（1）

### Consumer

单个分区的消息只能有ConsumerGroup中的某个Consumer消费（一个partition不能被多个consumer消费 ，但一个consumer可以消费多个partition）

Consumer从Partition中消费消息是顺序消费，默认从头消费

单个ConsumerGroup会消费所有Partition中的消息

### Stream



### Connect

流式计算的一部分，用来与其他中间件建立流式通道，负责加载数据到kafka中，和将kafka中的数据转移出去



## 底层实现

### 日志存储机制

存储目录在`config/server.properties`配置文件中由`log.dirs`进行指定。

1. 日志以partition为单位进行保存，每个partition只支持顺序读写

2. 每个 Partition 都为一个目录，而每一个目录又被平均分配成N个大小相等的 Segment File(但每个segment中消息数量不一定相等)，当segment达到一定阈值会flush到磁盘上。

   Segment File又由 index file（".index"）和 data file（".log"）组成，他们总是成对出现。index中记录了消息的offset，来在log文件中进行检索。

   index 文件中并没有为数据文件中的每条 Message 建立索引，而是采用了稀疏存储的方式，每隔一定字节的数据建立一条索引。这样避免了索引文件占用过多的空间，从而可以将索引文件保留在内存中。

3. Partition 会为每个 Consumer Group 保存一个偏移量，记录 Group 消费到的位置

### 偏移量

消费者的offset会提交到kafka特殊的topic中：`__consumer_offsets`，该topic默认分区数由`offsets.topic.num.partitions`参数控制，默认数为50

### Topic订阅与故障发现

consumer和coordinator之间的心跳为3秒，超时检查`session.timeout.ms=45000`

消费者处理消息时间为`max.poll.interval.ms=300000` 5分钟，超时后会触发rebalance



## 设计原理

### 高效率

### 消息传递保障

### 副本集

AR：分区中所有副本称为 AR

ISR：所有与主副本保持一定程度同步的副本（包括主副本）称为 ISR

OSR：与主副本滞后过多的副本组成 OSR

AR = ISR + OSR

### 日志压缩



## 集群配置

- Broker: 一般指kafka的部署节点
- Leader: 用于处理消息的接收和消费等请求
- Follower: 主要用于备份消息数据

### 拓扑结构

![拓扑结构](https://gtw.oss-cn-shanghai.aliyuncs.com/Kafka/kafka%E6%8B%93%E6%89%91%E7%BB%93%E6%9E%84.jpeg)

### Leader选举

 并没有采用投票多数来选举leader的机制（各个节点可能同步数据量不一样）

采用动态维护一组Leader数据的副本（ISR），kafka会在ISR中选择一个速度比较快的设为leader。

如果ISR中副本全部宕机，kafka会进行unclean leader选举，提供了两种不同方式处理该部分内容：1. 等ISR恢复，在ISR中选出一个leader；2. 在ISR之外的Follower中选取一个leader。（生产建议禁用"unclean leader"选举，手动来指定最小ISR）

### zk在kafka中的应用

kafka信息在zk上的存储：

`/brokers/ids` 存储broker对应信息

`/brokers/topics` 存储所有topic信息

`/admin/delete_topics`被删除的topic信息

`/config` 配置信息

`/consumer` 消费者信息



## 集群监控

### kafka manager



## 经验之谈

### 面试点

### 推荐配置方式

