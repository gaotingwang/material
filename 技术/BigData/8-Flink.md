Flink 是一个在**有界数据流**和**无界数据流**上进行**有状态计算**的分布式处理引擎和框架。Flink 设计旨在所有常见的集群环境中运行，以任意规模和内存级速度执行计算。

## 架构

Flink 运行时由两种类型的进程组成：一个 `JobManager` 和一个或者多个 `TaskManager`。

![Flink架构](https://gtw.oss-cn-shanghai.aliyuncs.com/BigData/Flink/Flink%E6%9E%B6%E6%9E%84.png)

`Client` 不是程序执行的一部分，而是用于准备数据流并将其发送给 JobManager。之后客户端可以断开连接（分离模式），或保持连接来接收进程报告（附加模式）。客户端可以作为触发执行 Java/Scala 程序的一部分运行，也可以在命令行进程`./bin/flink run ...`中运行。

- JobManager

  具有许多协调 Flink 应用程序进行分布式执行的有关职责：它决定何时调度下一个 task（或一组 task）、对完成的 task 或执行失败做出反应、协调 checkpoint、并且协调从失败中恢复等等

- TaskManagers

  也称为 worker ，执行作业流的 task，并且缓存和交换数据流，必须始终至少有一个 TaskManager。在 TaskManager 中资源调度的最小单位是 `task slot`。TaskManager 中 `task slot` 的数量表示并发处理 task 的数量。请注意一个 `task slot` 中可以执行多个算子



## 部署

standalone模式：

```shell
# 依赖java环境
$ tar -xzf flink-1.20-SNAPSHOT-bin-scala_2.12.tgz
$ cd flink-1.20-SNAPSHOT-bin-scala_2.12

# 启动
$ ./bin/start-cluster.sh

# 运行作业 -c 指定主类 -p 并行度
$ flink run -c com.gtw.xxx.MainClass -p 1 wordCount.jar

# 查看作业 -a 查看所有作业 -r 查看运行中的作业
$ flink list -a

# 取消作业
$ flink cancel <Job ID>

# 停止
$ ./bin/stop-cluster.sh
```

cluster模式：

### CLI Actions

[命令行参数说明](https://nightlies.apache.org/flink/flink-docs-master/docs/deployment/cli/#cli-actions)



## 运行模式

![运行模式](https://gtw.oss-cn-shanghai.aliyuncs.com/BigData/Flink/%E8%BF%90%E8%A1%8C%E6%A8%A1%E5%BC%8F.png)

- Session Mode

  在 Flink Session 集群中，客户端连接到一个预先存在的、长期运行的集群，该集群可以接受多个作业提交。即使所有作业完成后，集群（和 JobManager）仍将继续运行直到手动停止 session 为止。因此，**Flink Session 集群的寿命不受任何 Flink 作业寿命的约束**。

  TaskManager slot 由 ResourceManager 在提交作业时分配，并在作业完成时释放。由于所有作业都共享同一集群，因此在集群资源方面存在一些竞争。 如果 TaskManager 崩溃，则在此 TaskManager 上运行 task 的所有作业都将失败

  拥有一个预先存在的集群可以节省大量时间申请资源和启动 TaskManager。如作业执行时间短并且启动时间长会对端到端的用户体验产生负面的影响 ，希望作业可以使用现有资源快速执行计算的场景

- Application Mode

  Flink Application 集群是专用的 Flink 集群，仅从 Flink 应用程序执行作业，并且 `main()`方法在集群上而不是客户端上运行。

  提交作业是一个单步骤过程：无需先启动 Flink 集群，然后将作业提交到现有的 session 集群；相反，将应用程序逻辑和依赖打包成一个可执行的作业 JAR 中，并且集群入口（`ApplicationClusterEntryPoint`）负责调用 `main()`方法来提取 JobGraph。这允许你像在 Kubernetes 上部署任何其他应用程序一样部署 Flink 应用程序。因此，**Flink Application 集群的寿命与 Flink 应用程序的寿命有关**。

  ResourceManager 和 Dispatcher 作用于单个的 Flink 应用程序，相比于 Flink Session 集群，它提供了更好的隔离。

```shell
# standalone 
# session 方式 start-cluster.sh
$ ./bin/flink run ./examples/streaming/TopSpeedWindowing.jar

# application 方式 standalone-job.sh + taskmanager.sh
# 运行的jar需要在classpath中
$ cp ./examples/streaming/TopSpeedWindowing.jar lib/ 
# 之后可以访问http://localhost:8081
$ ./bin/standalone-job.sh start --job-classname org.apache.flink.streaming.examples.windowing.TopSpeedWindowing 
# 程序需要启动TaskManagers才能正式开始运行
$ ./bin/taskmanager.sh start 


# YARN
# session 方式
# (0) export HADOOP_CLASSPATH
export HADOOP_CLASSPATH=`hadoop classpath`
# (1) Start YARN Session
$ ./bin/yarn-session.sh --detached
# (2) You can now access the Flink Web Interface through the
# URL printed in the last lines of the command output, or through
# the YARN ResourceManager web UI.
# (3) Submit example job
$ ./bin/flink run ./examples/streaming/TopSpeedWindowing.jar
$ ./bin/flink run -t yarn-session -Dyarn.application.id=application_XXXX_YY \
  ./examples/streaming/TopSpeedWindowing.jar
# (4) Stop YARN session (replace the application id based 
# on the output of the yarn-session.sh command)
$ echo "stop" | ./bin/yarn-session.sh -id application_XXXXX_XXX

# application 方式
$ ./bin/flink run-application -t yarn-application ./examples/streaming/TopSpeedWindowing.jar
# List running job on the cluster
$ ./bin/flink list -t yarn-application -Dyarn.application.id=application_XXXX_YY
# Cancel running job
$ ./bin/flink cancel -t yarn-application -Dyarn.application.id=application_XXXX_YY <jobId>
```



## 编程

编程规范：

1. 获取执行环境

   ```java
   // 获取上下文，统一采用StreamExecutionEnvironment上下文
   // 如果在 IDE 中执行或将其作为一般的 Java 程序执行，那么它将创建一个本地环境，该环境将在本地机器上执行程序。
   // 如果基于程序创建了一个 JAR 文件，并通过命令行运行它，Flink 集群管理器将执行程序的 main 方法，同时 getExecutionEnvironment() 方法会返回一个执行环境以在集群上执行程序。
   StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
   // 可以选择模式
   //   env.setRuntimeMode(RuntimeExecutionMode.BATCH); 批处理
   //   env.setRuntimeMode(RuntimeExecutionMode.STREAMING); 流处理
   env.setRuntimeMode(RuntimeExecutionMode.AUTOMATIC); // 自动选择
   ```

2. 加载/创建初始数据

   可以用 `StreamExecutionEnvironment.addSource(sourceFunction)` 将一个 source 关联到程序。

   Flink 自带了许多预先实现的 source functions，不过仍然可以通过实现 `SourceFunction` 接口编写自定义的非并行 source，也可以通过实现 `ParallelSourceFunction` 接口或者继承 `RichParallelSourceFunction` 类编写自定义的并行 sources，以`Rich`开头，其内有`open()`、`close()`生命周期函数

3. 针对数据做处理操作

4. 指定计算结果的存储位置

5. 触发程序执行

   ```java
   env.execute("作业名字");
   ```

### 数据接入

`StreamExecutionEnvironment` 可以访问多种预定义的 stream source：

- 基于文件：`readTextFile(path)`、`readFile(fileInputFormat, path)`
- 基于Socket：`socketTextStream`
- 基于集合：`fromCollection(Collection)`、`fromElements(T ...)`
- 自定义：`addSource` - 关联一个新的 source function

### 数据处理

`DataStream` 表示的是Flink程序中要执行的数据集合（不可变，数据可重复，可以有界或无界），可以添加额外的`DataStream`

通过`Operators transform`能将一个或多个 DataStream 转换成新的 DataStream，在应用程序中可以将多个数据转换合并成一个复杂的数据流拓扑

```java
// map(): DataStream → DataStream
// filter()：DataStream → DataStream
// flatMap()：DataStream → DataStream，输入一个元素同时产生零个、一个或多个元素
// keyBy()：DataStream → KeyedStream，分组聚合，之后可对KeyedStream继续sum() 、 reduce()等操作
dataStream.keyBy(value -> value.getSomeKey());
// union()：DataStream* → DataStream，将多个数据流联合来创建一个包含所有流中数据的新流，要求流中的数据类型必须一致
dataStream.union(otherStream1, otherStream2, ...);
// connect()：DataStream,DataStream → ConnectedStream，两个流中数据类型可不一样
ConnectedStreams<Integer, String> connectedStreams = someStream.connect(otherStream);
connectedStreams.map(new CoMapFunction<Integer, String, Boolean>() {
    // 对流1数据做处理
    @Override
    public Boolean map1(Integer value) {
        return true;
    }

    // 对流2数据做处理
    @Override
    public Boolean map2(String value) {
        return false;
    }
});
```

自定义分区：

```java
public class AccessPartitioner implements Partitioner<String> {
    @Override
    public int partition(String key, int partitionNum) {
        if("a".equals(key)) {
            return 0;
        }else if("b".equals(key)) {
            return 1;
        }else {
            return 2;
        }
    }
}

// 分区器，相同分区交给同一线程进行处理
source.partitionCustom(new AccessPartitioner(), x -> x.getDomain());
```

分流操作：除了由 `DataStream` 操作产生的主要流之外，还可以产生任意数量的旁路输出结果流。结果流中的数据类型不必与主要流中的数据类型相匹配，并且不同旁路输出的类型也可以不同。

```
[读取数据]---->[处理逻辑1]---------->[数据输出]---->[HDFS]
                       \--------->[数据输出]----->[mysql]
                        \-------->[数据输出]----->[clickhouse]
```

旁路输出是在不复制数据流的情况下。把一个数据流分割成多个流，并且不同的流进行不同的处理。通过这种方式来同一个流进行不同的处理非常方便，也是非常常用的一种流处理策略。

```java
// 分流操作, OutputTag 标识旁路输出流
OutputTag<Access> outputTag1 = new OutputTag<Access>("分流1"){};
OutputTag<Access> outputTag2 = new OutputTag<Access>("分流2"){};

// 放入不同的Tag中
SingleOutputStreamOperator<Access> processSource = source.process(new ProcessFunction<Access, Access>() {
    @Override
    public void processElement(Access value, ProcessFunction<Access, Access>.Context ctx, Collector<Access> out) throws Exception {
        if("qq.com".equals(value.getDomain())) {
            ctx.output(outputTag1, value);
        }else if("baidu.com".equals(value.getDomain())) {
            ctx.output(outputTag2, value);
        }else {
            out.collect(value);
        }
    }
});

// 从指定tag中获取
processSource.print("主流");
processSource.getSideOutput(outputTag1).print("1分流");
processSource.getSideOutput(outputTag2).print("2分流");
```

### 结果输出

Data sinks 使用 DataStream 并将它们转发到文件、套接字、外部系统或打印它们。多种预定义的 Sink：

- 文件：`writeAsText()/writeAsCsv()/`
- Socket：`writeToSocket`
- 控制台：`print()` / `printToErr()`
- 自定义：`addSink` - 调用自定义 sink function



## Window

### 时间语义

- process time

  数据处理的时间，与执行机器的当前系统时间有关

  优点：性能好，低延迟

  缺点：结果不确定（网络抖动、运行效率等因素，数据被运行时间可能滞后）

- event time

  数据真正产生的时间，一旦产生不会改变，与watermark有关，结果具有确定性

### Window分类

1. 是否keyBy

   - Keyed Windows

     ```java
     stream
            .keyBy(...)               <-  keyed versus non-keyed windows
            .window(...)              <-  required: "assigner"
           [.trigger(...)]            <-  optional: "trigger" (else default trigger)
           [.evictor(...)]            <-  optional: "evictor" (else no evictor)
           [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
           [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
            .reduce/aggregate/apply()      <-  required: "function"
           [.getSideOutput(...)]      <-  optional: "output tag"
     ```

   - Non-Keyed Windows

     ```java
     stream
            .windowAll(...)           <-  required: "assigner"
           [.trigger(...)]            <-  optional: "trigger" (else default trigger)
           [.evictor(...)]            <-  optional: "evictor" (else no evictor)
           [.allowedLateness(...)]    <-  optional: "lateness" (else zero)
           [.sideOutputLateData(...)] <-  optional: "output tag" (else no side output for late data)
            .reduce/aggregate/apply()      <-  required: "function"
           [.getSideOutput(...)]      <-  optional: "output tag"
     ```

2. 按照时间或数量划分

   - 按时间划分 Time-based Windows，根据时间对数据流切片，如每隔30s一个窗口
   - 按数量划分 Count-based Windows，按照元素个数对数据流切片，如3个元素构成一个窗口

### Window Assigners职责

可以在 `window(...)`或 `windowAll(...)` 中指定一个 `WindowAssigner`， `WindowAssigner` 负责将 stream 中的每个数据分发到一个或多个窗口中

基于时间的窗口用 *start timestamp*（包含）和 *end timestamp*（不包含）描述窗口的大小。 在代码中，Flink 处理基于时间的窗口使用的是 `TimeWindow`， 它有查询开始和结束 timestamp 以及返回窗口所能储存的最大 timestamp 的方法 `maxTimestamp()`。

内置的assigner：

- Tumbling Windows 滚动窗口

  将数据依据固定窗口长度进行切分，窗口大小固定，不重叠，元素只属于一个窗口

- Sliding Windows 滑动窗口

  将固定窗口进行滑动，需要指定滑动步长slide，slide < 窗口大小时，会出现窗口重叠。滚动窗口是特殊的滑动窗口，slide = 窗口大小

- Session Windows 会话窗口

  在一段时间内没有接收到数据，该窗口就会关闭。超出该时间段，会话窗口被关闭，数据会被分发到新的窗口上

- Global Windows 全局窗口

- 自定义 extend `WindowAssigner`

### Window Function

- 采用增量方式，效率高

  ReduceFuntion 

  AggregateFunction

- 全量方式，对窗口内所有数据进行遍历，效率低，使用更灵活

  ProcessWindowFunction

- 全量和增量是可以配合使用的

## 延迟乱序数据处理 Watermark

由于网路抖动，数据到达会有延迟、乱序问题，引入watermark用来衡量event进展，处理时是根据event-time来的

watermark是单调递增的向前推荐时间的一种机制，可以从源头产生的时候带着，也可以在operator上带着

### 产生策略

```java
stream.
  // Watermark 指定
  .assignTimestampsAndWatermarks(
  // forMonotonousTimestamps 0容忍度，forBoundedOutOfOrderness(Duration.ofSeconds(2)) 可传入容忍延迟时间
  WatermarkStrategy.<Access>forMonotonousTimestamps().withTimestampAssigner(new SerializableTimestampAssigner<Access>() {
    // 指定 event-time
    @Override
    public long extractTimestamp(Access access, long l) {
      return access.getTime();
    }
  })
)
```

### 结合EventTime & Time

窗口大小是[window_start, window_end)，左闭右开，当wartermark >= window_end就会触发执行

```java
public class BoundedOutOfOrdernessWatermarks<T> implements WatermarkGenerator<T> {
    private long maxTimestamp;
    private final long outOfOrdernessMillis;

    public BoundedOutOfOrdernessWatermarks(Duration maxOutOfOrderness) {
        this.outOfOrdernessMillis = maxOutOfOrderness.toMillis();
        // 初始化时，maxTimestamp为Long最小值 + 1
        this.maxTimestamp = Long.MIN_VALUE + this.outOfOrdernessMillis + 1L;
    }
		
  	// 每次事件触发：maxTimestamp取event-time和maxTimestamp的最大值
    public void onEvent(T event, long eventTimestamp, WatermarkOutput output) {
        this.maxTimestamp = Math.max(this.maxTimestamp, eventTimestamp);
    }

  	// Watermark 就是maxTimestamp - 1
  	// 每次事件触发，获取到的就是最大的event-time，Watermark本质相当于event-time - 1
    public void onPeriodicEmit(WatermarkOutput output) {
        output.emitWatermark(new Watermark(this.maxTimestamp - this.outOfOrdernessMillis - 1L));
    }
}
```

### 数据乱序&延迟

方案一：采用容忍度

```java
/**
 * 支持容忍度，延迟窗口触发条件
 * 1000,a,1
 * 1999,a,1
 * 5000,b,1 ---> 本来是在这里触发窗口执行，延迟2s到7000时才执行，这样可以让延迟到来的4000数据可以加入前窗口执行
 * 4000,a,1
 * 7000,c,1
 * Access{domain='a', traffic=3.0}
 */
WatermarkStrategy.<Access>forBoundedOutOfOrderness(Duration.ofSeconds(2))
```

方案二：允许延迟

```java
/**
 * 允许延迟方式
 * 1000,a,1
 * 1999,a,1
 * 5000,b,1
 * Access{domain='a', traffic=2.0}
 *
 * 4000,a,1
 * Access{time=1000, domain='a', traffic=3.0}
 *
 * 4999,a,1
 * Access{time=1000, domain='a', traffic=4.0}
 *
 * 7000,c,1
 * 4300,a,1 --------->大于2s后的数据，被舍弃了
 * 8000,a,1
 * 10000,c,1
 * Access{domain='c', traffic=1.0}
 * Access{domain='a', traffic=1.0}
 * Access{domain='b', traffic=1.0}
 */
stream
       .keyBy(...)             
       .window(...)          
  		 // 允许延迟2s
       .allowedLateness(Time.seconds(2))
```

方案三：延迟数据做SideOutput，对延迟数据可做重新处理等方式

```java
/**
 * 边路输出延迟数据
 * 1000,a,1
 * 1999,a,1
 * 5000,b,1
 * Access{domain='a', traffic=2.0}
 *
 * 4000,a,1
 * late data is :> Access{domain='a', traffic=1.0}
 *
 * 4999,a,1
 * late data is :> Access{domain='a', traffic=1.0}
 *
 * 7000,c,1
 * 4300,a,1
 * late data is :> Access{domain='a', traffic=1.0}
 *
 * 8000,a,1
 * 10000,c,1
 * Access{domain='c', traffic=1.0}
 * Access{domain='a', traffic=1.0}
 * Access{domain='b', traffic=1.0}
 */
 
OutputTag<Access> outputTag = new OutputTag<Access>("late data"){};

stream
       .keyBy(...)             
       .window(...)          
  		 // 延迟数据走sideOutput
       .sideOutputLateData(outputTag);

DataStream<Access> sideOutput = result.getSideOutput(outputTag);
sideOutput.print("late data is :");
```



## 状态管理

当前计算流程需要依赖之前的计算结果，那么之前的计算结果就是Stateful。

Flink State分为两大类：

- KeyedState

- OperatorState

### StateTTL

所有状态类型都支持单元素的 TTL。 这意味着列表元素和映射元素将独立到期

在使用状态 TTL 前，需要先构建一个`StateTtlConfig` 配置对象。 然后把配置传递到 state descriptor 中启用 TTL 功能：

```java
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    // TTL 的更新策略
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    // 数据在过期但还未被清理时的可见性
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
```

默认情况下，过期数据会在读取的时候被删除，例如 `ValueState#value`，同时会有后台线程定期清理。可以通过`.cleanupXXX()`来指定具体的清理方式

### Checkpoint机制

Flink 中的每个方法或算子都能够是**有状态的**。 状态化的方法在处理单个 元素/事件 的时候存储数据，让状态成为使各个类型的算子更加精细的重要部分。 

为了让状态容错，Flink 需要为状态添加 **checkpoint（检查点）**。Checkpoint 使得 Flink 能够恢复状态和在流中的位置，从而向应用提供和无故障执行时一样的语义。

默认情况下 [checkpoint](https://nightlies.apache.org/flink/flink-docs-master/zh/docs/dev/datastream/fault-tolerance/checkpointing/) 是禁用的。通过调用 `StreamExecutionEnvironment` 的 `enableCheckpointing(n)` 来启用 checkpoint，里面的 *n* 是进行 checkpoint 的间隔，单位毫秒。

```java
Configuration configuration = new Configuration();
// 记录savepoint位置，重启后依赖该文件从最新checkpoint的位置开始继续执行
configuration.setString("execution.savepoint.path", path + "{jobId}/chk-4");
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(configuration);
env.setParallelism(1);

// 打开checkpoint，周期性生成快照，防止作业挂了后，可以从最新的checkpoint进行继续作业
env.enableCheckpointing(5000, CheckpointingMode.EXACTLY_ONCE);
// 也可以通过state.checkpoints.dir: hdfs:///checkpoints/ 进行统一配置
env.getCheckpointConfig().setCheckpointStorage(path);
```

使用checkpoint进行恢复：

```shell
$ bin/flink run -s :checkpointMetaDataPath [:runArgs]
```

#### checkpoint vs savepoint

checkpoint 是自动触发生成的，savepoint是通过手动生成的，如在程序升级，需要启停，可以通过savepoint存储数据

### Task重启策略

开启Checkpoint 后，程序异常后默认是重启Integer的最大数。

```java
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 5));
```

### State Backend

Flink提供了多种支持存储State的方式

```java
env.setStateBackend(new HashMapStateBackend());
```



## Flink Table & SQL API

离线数据处理整个过程（输入、处理、输出），它是一个静态表，数据都是明确固定的。Flink相比较离线处理来说，输入、处理、输出都是持续进行的，数据会源源不断的追加进来，是一个动态表

### 编程范式

创建 TableEnvironment：

```java
// 通过EnvironmentSettings创建
EnvironmentSettings settings = EnvironmentSettings.newInstance().build();
TableEnvironment tableEnv = TableEnvironment.create(settings);

// 从现有的 StreamExecutionEnvironment 创建一个 StreamTableEnvironment 与 DataStream API 互操作
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tEnv = StreamTableEnvironment.create(env);
```

DataStream和Table相互转换

```java
StreamExecutionEnvironment streamEnv = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = StreamTableEnvironment.create(streamEnv);

SingleOutputStreamOperator<ClickLog> outputStream = streamEnv....

// 将DateStream转Table
Table table = tableEnv.fromDataStream(outputStream);
Table result = table.select($("user"), $("url"), $("time")).where($("user").isEqual("Mary"));

// 将Table转DateStream
DataStream<ClickLog> dataStream = tableEnv.toDataStream(result, ClickLog.class);
dataStream.print();
```

创建Table&表&视图：

临时表：是存储在内存中的，当前会话持续期间存在，对于其它会话是不可见的。它们不与任何 catalog 或者数据库绑定。

永久表：一旦被创建，它将对任何连接到 catalog 的 Flink 会话可见且持续存在，直至被明确删除

```java
// get a TableEnvironment
TableEnvironment tableEnv = ...; 

// table is the result of a simple projection query 
Table projTable = tableEnv.from("X").select(...);

// 创建视图
tableEnv.createTemporaryView("projectedTable", projTable);

// 创建表
TableDescriptor sourceDescriptor = TableDescriptor.forConnector("datagen")
        .schema(Schema.newBuilder()
                .column("column1", DataTypes.STRING())
                .build())
        .option(DataGenConnectorOptions.ROWS_PER_SECOND, 100L)
        .build();
tableEnv.createTable("SourceTableA", sourceDescriptor);
tableEnv.createTemporaryTable("SourceTableB", sourceDescriptor);
```

Table API：

```java
Table orders = tableEnv.from("Orders");
// compute revenue for all customers from France
Table revenue = orders
  .filter($("cCountry").isEqual("FRANCE"))
  .groupBy($("cID"), $("cName"))
  .select($("cID"), $("cName"), $("revenue").sum().as("revSum"));
```

SQL：[DDL & DML](https://nightlies.apache.org/flink/flink-docs-master/zh/docs/dev/table/sql/overview/)

```java
// 将查询结果作为 Table 对象返回
Table revenue = tableEnv.sqlQuery(
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// 将查询的结果插入到已注册的表中
tableEnv.executeSql(
    "INSERT INTO RevenueFrance " +
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );
```

时间语义在DDL中定义 

```sql
-- 使用process time
CREATE TABLE user_actions (
  user_name STRING,
  data STRING,
  user_action_time AS PROCTIME() -- 就是系统的process time
) WITH (
  ...
);

-- event time
CREATE TABLE user_actions (
  user_name STRING,
  data STRING,
  user_action_time TIMESTAMP(3),
  -- 声明 user_action_time 是event time属性，并且用延迟 5 秒的策略来生成 watermark
  WATERMARK FOR user_action_time AS user_action_time - INTERVAL '5' SECOND
) WITH (
  ...
);
```

### Connector

[Table API Connector](https://nightlies.apache.org/flink/flink-docs-master/zh/docs/connectors/table/overview/)

Upsert Kafka 连接器支持以 upsert 方式从 Kafka topic 中读取数据并将数据写入 Kafka topic。

- 作为 source，upsert-kafka 连接器生产 changelog 流，其中每条数据记录代表一个更新或删除事件。更准确地说，如果有这个 key（如果不存在相应的 key，则该更新被视为 INSERT），数据记录中的 value 被解释为同一 key 的最后一个 value 的 UPDATE。因为任何具有相同 key 的现有行都被覆盖。另外，value 为空的消息将会被视作为 DELETE 消息。

- 作为 sink，upsert-kafka 连接器可以消费 changelog 流。它会将 INSERT/UPDATE_AFTER 数据作为正常的 Kafka 消息写入，并将 DELETE 数据以 value 为空的 Kafka 消息写入（表示对应 key 的消息被删除）。

  Flink 将根据主键列的值对数据进行分区，从而保证主键上的消息有序，因此同一主键上的更新/删除消息将落在同一分区中。

JDBC Connector，如果在 DDL 中定义了主键，JDBC sink 将使用 upsert 语义而不是普通的 INSERT 语句。由于 upsert 没有标准的语法，如在MySQL中，使用`INSERT .. ON DUPLICATE KEY UPDATE ..`

[对输入数据格式支持](https://nightlies.apache.org/flink/flink-docs-master/zh/docs/connectors/table/formats/overview/)

### UDF函数

函数分为4类：

1. 临时性系统函数
2. 系统函数
3. 临时性 Catalog 函数
4. Catalog 函数

系统函数始终优先于 Catalog 函数解析，临时函数始终优先于持久化函数解析

[内置函数](https://nightlies.apache.org/flink/flink-docs-master/zh/docs/dev/table/functions/systemfunctions/)

[自定义函数及使用](https://nightlies.apache.org/flink/flink-docs-master/zh/docs/dev/table/functions/udfs/#%E8%A1%A8%E5%80%BC%E5%87%BD%E6%95%B0)

### SQL-Client

SQL-Client 的目的是提供一种简单的方式来编写、调试和提交表程序到 Flink 集群上，而无需写一行 Java 或 Scala 代码。SQL Client CLI 能够在命令行中检索和可视化分布式应用中实时产生的结果

```shell
# 需要先启动flink集群
$ ./bin/start-cluster.sh

# 启动sql client
$ ./bin/start-cluster.sh
```

SQL使用滚动Window举例：

```sqlite

Flink SQL> SELECT * FROM Bid;
+------------------+-------+------+
|          bidtime | price | item |
+------------------+-------+------+
| 2020-04-15 08:05 |  4.00 | C    |
| 2020-04-15 08:07 |  2.00 | A    |
| 2020-04-15 08:09 |  5.00 | D    |
| 2020-04-15 08:11 |  3.00 | B    |
| 2020-04-15 08:13 |  1.00 | E    |
| 2020-04-15 08:17 |  6.00 | F    |
+------------------+-------+------+

Flink SQL> SELECT * FROM TABLE(
   -- TUMBLE 开滚动窗口，三个参数:第一个拥有时间属性列的表;第二个列描述符，决定数据的哪个时间属性列应该映射到窗口;第三个窗口大小开10min窗口
   TUMBLE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '10' MINUTES)
);
+------------------+-------+------+------------------+------------------+-------------------------+
|          bidtime | price | item |     window_start |       window_end |            window_time  |
+------------------+-------+------+------------------+------------------+-------------------------+
| 2020-04-15 08:05 |  4.00 | C    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:07 |  2.00 | A    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:09 |  5.00 | D    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:17 |  6.00 | F    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
+------------------+-------+------+------------------+------------------+-------------------------+

-- apply aggregation on the tumbling windowed table
Flink SQL> SELECT window_start, window_end, SUM(price) AS total_price
  FROM TABLE(
    TUMBLE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '10' MINUTES))
  GROUP BY window_start, window_end;
+------------------+------------------+-------------+
|     window_start |       window_end | total_price |
+------------------+------------------+-------------+
| 2020-04-15 08:00 | 2020-04-15 08:10 |       11.00 |
| 2020-04-15 08:10 | 2020-04-15 08:20 |       10.00 |
+------------------+------------------+-------------+

```

SQL使用滑动Window举例：

```sql
> SELECT * FROM TABLE(
    -- HOP开滑动窗口，窗口大小10min，滑动步长5min
    HOP(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '5' MINUTES, INTERVAL '10' MINUTES)
);
+------------------+-------+------+------------------+------------------+-------------------------+
|          bidtime | price | item |     window_start |       window_end |           window_time   |
+------------------+-------+------+------------------+------------------+-------------------------+
| 2020-04-15 08:05 |  4.00 | C    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 | -- 数据会属于多个窗口
| 2020-04-15 08:05 |  4.00 | C    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:07 |  2.00 | A    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:07 |  2.00 | A    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:09 |  5.00 | D    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:09 |  5.00 | D    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:17 |  6.00 | F    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:17 |  6.00 | F    | 2020-04-15 08:15 | 2020-04-15 08:25 | 2020-04-15 08:24:59.999 |
+------------------+-------+------+------------------+------------------+-------------------------+

-- apply aggregation on the hopping windowed table
> SELECT window_start, window_end, SUM(price) AS total_price
  FROM TABLE(
    HOP(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '5' MINUTES, INTERVAL '10' MINUTES))
  GROUP BY window_start, window_end;
+------------------+------------------+-------------+
|     window_start |       window_end | total_price |
+------------------+------------------+-------------+
| 2020-04-15 08:00 | 2020-04-15 08:10 |       11.00 |
| 2020-04-15 08:05 | 2020-04-15 08:15 |       15.00 |
| 2020-04-15 08:10 | 2020-04-15 08:20 |       10.00 |
| 2020-04-15 08:15 | 2020-04-15 08:25 |        6.00 |
+------------------+------------------+-------------+
```

