Flume是一个针对日志数据进行收集汇总的框架

数据两大场景：

- 离线

  data ==> Flume ==> HDFS ==> 大数据分布式计算引擎

- 实时

  data ==> Flume ==> Kafka ==> Stream Engine

## 三大核心组件

![Flume](https://flume.apache.org/_images/DevGuide_image00.png)

Flume Agent 是一个JVM进程，由Source、Channel和Sink构成

### 收集：Source

负责收集数据到Flume

- Avro Source
- Exec Source
- Spooling Directory Source
- Taildir Source
- Kafka Source

### 聚合：Channel

是介于Source和Sink之间的缓冲区，临时存储的地方

- Memory Channel
- Kafka Channel
- File Channel

### 移动：Sink

读取Channel的数据，然后推送到不同的目的地

- HDFS Sink
- Hive Sink
- Logger Sink
- Avro Sink
- Kafka Sink
- ES/HBase Sink



## 部署

需要修改下`$FLUME_HOME/conf`目录下`flume-env.sh`中的`JAVA_HOME`



## 配置编写

使用Flume最关键的就是Agent配置文件的编写

```properties
# agent 的名字为 a1
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# 配置 source
a1.sources.r1.type = netcat
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 44444

# 配置 channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# 配置 sink
a1.sinks.k1.type = logger

# 三大组件的流向组织
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

启动：

```shell
$ flume-ng agent --name a1 --conf $FlUME_HOME/conf --conf-file /home/hadoop/config/flume/example.conf -Dflume.root.logger=INFO,console
# test
$ telnet localhost 44444
```



## 日志收集

### 文件数据收集到HDFS

Source采用Exec Source，通过`tail -f`命令执行获取日志内容

```properties
# agent name
exec-memory-hdfs.sources = exec-source
exec-memory-hdfs.sinks = hdfs-sink
exec-memory-hdfs.channels = memory-channel

# config source
exec-memory-hdfs.sources.exec-source.type = exec
exec-memory-hdfs.sources.exec-source.command = tail -F ~/data/logs/data.log
exec-memory-hdfs.sources.exec-source.shell = /bin/sh -c

# config channel
exec-memory-hdfs.channels.memory-channel.type = memory
exec-memory-hdfs.channels.memory-channel.capacity = 1000
exec-memory-hdfs.channels.memory-channel.transactionCapacity = 100

# config sink
exec-memory-hdfs.sinks.hdfs-sink.type = hdfs
# 可以按照发生时间或机器属性等对数据进行分区，通过 % + 参数
exec-memory-hdfs.sinks.hdfs-sink.hdfs.path = hdfs://hadoop000:9000/data/flume/tail/%Y%m%d%H%M
exec-memory-hdfs.sinks.hdfs-sink.hdfs.useLocalTimeStamp = true
# 是否按时间戳滚动生成，如1min生成一个文件夹
spool-memory-hdfs.sinks.hdfs-sink.hdfs.round = true
spool-memory-hdfs.sinks.hdfs-sink.hdfs.roundValue = 1
spool-memory-hdfs.sinks.hdfs-sink.hdfs.roundUnit = minute
# 压缩流需要指定压缩格式
exec-memory-hdfs.sinks.hdfs-sink.hdfs.fileType = CompressedStream
exec-memory-hdfs.sinks.hdfs-sink.hdfs.codeC = gzip
exec-memory-hdfs.sinks.hdfs-sink.hdfs.filePrefix = event-
# 防止HDFS上出现小文件，按照300s间隔， 要么128M，要么10万条才进行滚动
spool-memory-hdfs.sinks.hdfs-sink.hdfs.rollInterval = 300
spool-memory-hdfs.sinks.hdfs-sink.hdfs.rollSize = 134217728
spool-memory-hdfs.sinks.hdfs-sink.hdfs.rollCount = 1000000

# Bind the source and sink to the channel
exec-memory-hdfs.sources.exec-source.channels = memory-channel
exec-memory-hdfs.sinks.hdfs-sink.channel = memory-channel
```

测试：

```shell
# 启动
$ flume-ng agent -n exec-memory-hdfs -c $FlUME_HOME/conf \
	-f /home/hadoop/config/flume/exec-memory-hdfs.conf -Dflume.root.logger=INFO,console

# 脚本写入日志测试
for i in {1..200}; do echo "hello $i" >> /home/hadoop/data/logs/data.log; sleep 0.1; done
```

### 文件夹数据收集到HDFS

对文件夹的监听，可以采用Spooling Directory Source

- 文件处理完，就会标记一个`COMPLETED`（`a.log.COMPLETED`），以表示该文件已经处理完
- 当文件被放入Spooling Directory中后，文件被再次写入Flume会报错并停止处理
- 放入的文件名要是重复，Flume会打印错误日志并停止处理；为了避免此问题，建议文件名带唯一标识，如时间戳

### TAILDIR方式收集

监听多个文件夹内容，支持断点续传

```properties
# config source
taildir-way.sources.taildir-source.type = TAILDIR
# 配置多个文件组，用空格隔开
taildir-way.sources.taildir-source.filegroups = f1 f2
# 指定文件组的监听文件
taildir-way.sources.taildir-source.filegroups.f1 = /home/hadoop/data/logs/taillog1/a.log
taildir-way.sources.taildir-source.filegroups.f2 = /home/hadoop/data/logs/taillog2/.*log.*
# 记录处理过数据的offset
# 断点续传，如果flume挂了，重启后，flume会读取文件中的offset，开始续传
taildir-way.sources.taildir-source.positionFile = ~/.flume/taildir_position.json
```

### avrosink配合avrosource使用

![avro](https://flume.apache.org/releases/content/1.11.0/_images/UserGuide_image03.png)

为了在多个代理活跃点之间传输数据，前一个代理的接收器和当前活跃点的源需要是 avro 类型，接收器**指向源的主机名（或 IP 地址）和端口**。

```properties
# 上游 Agent 配置avro的sink
exec-memory-avro.sinks.avro-sink.type = avro
exec-memory-avro.sinks.avro-sink.hostname = 0.0.0.0
exec-memory-avro.sinks.avro-sink.port = 44445

# 下游 Agent 配置avro的source
avro-memory-logger.sources.avro-source.type = avro
avro-memory-logger.sources.avro-source.bind = 0.0.0.0
avro-memory-logger.sources.avro-source.port = 44445

# 上下游的hostname和port一定要对应，一定先启动下游Agent，再启动上游Agent
```

### Channel Selector实战

![Agent数据流向](https://gtw.oss-cn-shanghai.aliyuncs.com/BigData/Flume/agent%E6%95%B0%E6%8D%AE%E6%B5%81%E5%90%91.png)

通过Channel Selector来选择不同的channel从而实现不同目的地的写入，一对多的分发形式

![](https://gtw.oss-cn-shanghai.aliyuncs.com/BigData/Flume/agent%E5%A4%9A%E7%9B%AE%E7%9A%84%E5%9C%B0%E5%86%99%E5%85%A5.png)

```properties
a1.sources = r1
a1.sinks = k1 k2
a1.channels = c1 c2

# 配置多个channel
a1.channels.c1.type = memory
a1.channels.c2.type = memory

a1.sinks.k1.type = logger

a1.sinks.k2.type = hdfs
...

# Channel Selector有三种，relicating、multiplexing、custom
# replicating 这种选择器会以复制的方式将数据写到不同的Channel中
a1.sources.r1.selector.type = replicating
a1.sources.r1.channels = c1 c2
a1.sinks.k1.channel = c1
a1.sinks.k2.channel = c2
```

### Sink Processor实战

同一个channel的sink也可以有多个（同一时间同一个channel只有一个sink在工作），这样就有了HA效果，不同的sink 可以配置不同的优先级，会先往优先级高的sink 写

- load_balance：多个sink随机 / 负载均衡的输出
- failover：同一时刻只有一个sink 工作，当这个sink 挂了，会自动切换到另外一个sink

```properties
agent1.sources = r1
agent1.sinks = k1 k2
agent1.channels = c1

# 配置sinkgroup
agent1.sinkgroups = g1
agent1.sinkgroups.g1.sinks = k1 k2
# failover：一个sink失败自动切换到另一个
agent1.sinkgroups.g1.processor.type = failover
# 设置优先级，值越大优先级越高
agent1.sinkgroups.g1.processor.priority.k1 = 15
agent1.sinkgroups.g1.processor.priority.k2 = 10
agent1.sinkgroups.g1.processor.maxpenalty = 10000

...
# 配置sink1
agent1.sinks.k1.type = avro
agent1.sinks.k1.hostname = 0.0.0.0
agent1.sinks.k1.port = 55551
# 配置sink1
agent1.sinks.k2.type = avro
agent1.sinks.k2.hostname = 0.0.0.0
agent1.sinks.k2.port = 55552
...

```

