kafka 具有高吞吐、高性能、实时性高可靠等特点

## Topic 和 Partion



## 核心API解读

### 客户端

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

### 偏移量

### Topic订阅与故障发现



## 设计原理

### 持久性

### 高效率

### 消息传递保障

### 副本集



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



## 集群监控

### kafka manager



## 经验之谈

### 面试点

### 推荐配置方式

