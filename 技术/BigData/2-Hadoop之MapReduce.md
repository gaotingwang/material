## 介绍

通过并行方式，处理大数据量（T级别）的分布式计算框架

### 优缺点

- 优点

  - 编码角度：实现MR提供的抽象类或接口即可
  - 扩展角度：计算资源吃紧可以通过加机器进行水平扩展
  - 容错角度：Node挂了，MR框架会将Node上作业转移到其他节点上去
  - 海量数据的离线计算/批处理

- 缺点

  不适合实时计算

  不适合迭代次数多的计算（job1 -> job2 -> job3）

### 核心思想

MR程序一般有两个阶段：

1. Map阶段：MapTask运行是可以并行的，数量是可以设置的
2. Reduce阶段：ReduceTask的数量是可以设置的，数据来源上游的MapTask

(input) `<k1, v1> ->` **map** `-> <k2, v2> ->` **combine** `-> <k2, v2> ->` **reduce** `-> <k3, v3>` (output)

![MapReduce流程](https://gtw.oss-cn-shanghai.aliyuncs.com/BigData/Hadoop/MapReduce%E6%B5%81%E7%A8%8B.jpg)

MRAppMaster 进程负责整个MR作业（N个MapTask + N个ReduceTask）的调度/协调工作



## 代码编写

### 数据类型

String	->	Text
Null	   ->	NullWritable
Int	     ->	IntWritable
xxx	    ->	xxxWritable

### 序列化

特点：

- 紧凑
- 快速
- 实现扩展性，支持多种协议
- 互操性，多语言之间

`Writable`是基于 `java.io.DataInput` / `DataOutput` 来实现简单，高效，支持序列化协议的可序列化对象

MR操作的`<key, value>`键值对，要求`key` `value`都实现`Writable`接口，`key`必须实现`WritableComparable`，便于框架排序

### 编程规范

1. 开发自定义的Mapper

   ```java
   public static class TokenizerMapper
           extends Mapper<Object, Text, Text, IntWritable>{
   
       private final static IntWritable one = new IntWritable(1);
       private Text word = new Text();
   
       /**
        * 重写map方法，自己的业务逻辑
        * Object, Text：Mapper输入数据的key类型和value类型
        * Text, IntWritable：Mapper输出数据的key类型和value类型
        */
       public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
           StringTokenizer itr = new StringTokenizer(value.toString());
           while (itr.hasMoreTokens()) {
               word.set(itr.nextToken());
               context.write(word, one);
           }
       }
   }
   ```

2. 开发自定义的Reducer

   ```java
   public static class IntSumReducer
           extends Reducer<Text,IntWritable,Text,IntWritable> {
       private IntWritable result = new IntWritable();
   
       /**
        * 重写map方法，自己的业务逻辑
        * Text,IntWritable：reducer输入数据的key类型和value类型
        *                 也是Mapper输出数据的key类型和value类型
        * Text,IntWritable：reducer输出数据的key类型和value类型
        *
        * Iterable<IntWritable> values
        * 
        * hello,hello,hello
        * banana,banana
        * 
        * 相同key经过shuffle后，会在同一个reducer中进行聚合
        * <hello,1><hello,1><hello,1> ==> <hello,<1,1,1>>
        * <banana,1><banana,1>        ==> <banana,<1,1>>
        */
       public void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
           int sum = 0;
           for (IntWritable val : values) {
               sum += val.get();
           }
           result.set(sum);
           context.write(key, result);
       }
   }
   ```

3. 自定义的Driver

   ```java
   // Driver把这个MR作业串起来
   public static void main(String[] args) throws Exception {
       Configuration conf = new Configuration();
       // 获取Job
       Job job = Job.getInstance(conf, "word count");
       // 设置作业Class
       job.setJarByClass(WordCount.class);
       // 设置自定义的Mapper
       job.setMapperClass(TokenizerMapper.class);
       // 设置自定义的Reducer
       job.setCombinerClass(IntSumReducer.class);
       // 设置自定义的Reducer
       job.setReducerClass(IntSumReducer.class);
       // 设置Mapper输出的Key,Value类型
       job.setOutputKeyClass(Text.class);
       job.setOutputValueClass(IntWritable.class);
       // 作业的输入目录
       FileInputFormat.addInputPath(job, new Path(args[0]));
       // 作业的输出目录
       FileOutputFormat.setOutputPath(job, new Path(args[1]));
       // 作业提交
       System.exit(job.waitForCompletion(true) ? 0 : 1);
   }
   ```

### InputFormat & InputSplit

HDFS中是以`Block`为单位进行存储，在MR中是以`InputSplit`为单位(是一个逻辑上的概念)交给MapTask来运行

数据由`InputFormat`读取进来：使用`FileInputFormat`，如果一个文件很小，则一个文件对应一个`InputSplit`，否则其大小由`fs.local.block.size`参数决定，默认大小32M；使用`NLineInputFormat`由参数`mapreduce.input.lineinputformat.linespermap`决定多少行，为一个`InputSplit`

```java
// 使用特殊的InputFormat需要显式指定
job.setInputFormatClass(NLineInputFormat.class);
```

MapTask的并行度与`InputSplit`数量有关

### Partitioner

根据指定条件将结果输出到不同的文件中，默认采用的是`HashPartitioner`

**reduce数量决定了最终文件输出的个数**：

```java
// 设置自定义分区规则
job.setPartitionerClass(MyPartitioner.class);
/**
 * reduce数量和分区数量的关系
 * 1. reduce 数量N = partition数量N N个文件
 * 2. reduce 数量N > partition数量M N个文件，多出来的文件为空
 * 3. reduce 数量 = 1，产生一个文件，数据合并在一起
 * 4. 1 < reduce 数量 < partition数量，报错
 */
job.setNumReduceTasks(3);
```

### Combiner

```java
// 设置Combiner,本质就是一个Reducer,只是运行在MapTask上
// 使用前MapperTask产出的数据：
// 		<hello,1><hello,1><hello,1>
// 		<hello,1><hello,1>
// 使用后MapperTask产出的数据：
// 		<hello,3>
// 		<hello,2>
job.setCombinerClass(WordCountReducer.class);
```

预聚合是分布式计算框架中非常常见的一个优化手段，在MR中称为Combiner。

Combiner是一个本地的Reducer操作，实际的Reducer是在接收到所有的Mapper结果之后进行处理，而Combiner是在每个MapTask上运行的，是对MapTask输出结果做汇总，来减少在Shuffle时的网络传输量

Combiner使用前提：不能影响最终的业务逻辑，如求平均值

### OutputFormat

OutputFormat 指定了最终作业的输出

```java
// 可以自定义作业结果的输出方式，不指定采用默认的TextOutputFormat
job.setOutputFormatClass(MyOutputFormat.class);
```

### Shuffle

从Mapper中拷贝数据到内存中，内存中溢写出的数据写到磁盘，之后将磁盘文件合并成一个文件，按照key进行分组，方便后续reducer获取

### 调优

- `mapreduce.task.io.sort.mb` 环型缓冲区大小，默认100M
- `mapreduce.map.sort.spill.percent` 缓冲区使用百分比，达到百分比后会将数据写入磁盘形成溢写文件，默认0.8
- `mapreduce.task.io.sort.factor` 对达到指定个数的溢写文件进行合并，默认10
- `mapreduce.map.memory.mb` map执行时内存大小，默认100M
- `mapreduce.map.cpu.vcores` map执行时的cpu数，默认1
- `mapreduce.reduce.shuffle.parallelcopies` 在Shuffle 拷贝数据阶段并行传输数量，默认5
- `mapreduce.reduce.shuffle.input.buffer.percent` 默认0.7
- `mapreduce.reduce.shuffle.merge.percent` 默认0.66
- `mapreduce.reduce.memory.mb`
- `mapreduce.reduce.cpu.vcores`

### 压缩

core-site.xml:

```xml
<property>
    <name>io.compression.codecs</name>
    <value>
    	org.apache.hadoop.io.compress.BZip2Codec,
        org.apache.hadoop.io.compress.DefaultCodec,
        org.apache.hadoop.io.compress.GzipCodec,
        org.apache.hadoop.io.compress.SnappyCodec
    </value>
</property>
```

mapreduce-site.xml

```xml
<!-- map端输出是否压缩，及压缩格式 -->
<property>
    <name>mapreduce.map.output.compress</name>
    <value>true</value>
</property>
<property>
    <name>mapreduce.map.output.compress.codec</name>
    <value>org.apache.hadoop.io.compress.BZip2Codec</value>
</property>
<!-- reducer端输出是否压缩，及压缩格式 -->
<property>
    <name>mapreduce.output.fileoutputformat.compress</name>
    <value>true</value>
</property>
<property>
    <name>mapreduce.output.fileoutputformat.compress.codec</name>
    <value>org.apache.hadoop.io.compress.BZip2Codec</value>
</property>
```

### 场景题

- group by

  WordCount功能相当于分组统计

- distinct

  MR框架按照key来进行数据的shuffle，相同的key肯定在同一个reduce中，只需要关注key

- join

  `select a.xx, b.xx from a join b on a.id = b.aid`

  - ReduceJoin：按照key进行shuffle，在reduce端完成join操作

    ![ReduceJoin](https://gtw.oss-cn-shanghai.aliyuncs.com/BigData/Hadoop/ReduceJoin.jpg)

    因为相同的key会放在同一个reduce中，所以将`on`的条件就是map端的输出

    要读取两份数据，对数据加一个标志位，用来区分数据来自于哪个表

    缺点：join操作是在reduce端完成的，map端仅是打了标记然后将数据分发。若某个key的数据量很大，数据都集中在一个reduce中，产生数据倾斜。

  - MapJoin：前提是大表和小表的join，将小表的数据加载到缓存中

    首先，将小表广播到分布式缓存中

    之后，读取大表数据时，每条数据和缓存中的数据做匹配，匹配上就是join上了

    不存在数据倾斜，所有匹配操作都是在Map端完成的

### 总结

1. InputFormat

    `FileInputFormat、TextInputFormat`

    `KeyValueTextInputFormat` 默认分割符`\t`

    `NLineInputFormat` N行进行切片

    `DBInputFormat` 数据库操作

2. Mapper

     四个泛型，第一个泛型`LongWritable`该行数据的`offset`，不是行号

3. Comparable

     key,value需要实现`Writable`接口

     key需要排序，需要实现`WritableComparable`接口

4. Partitioner

     按照key的规则进行分区，默认`HashPartitioner`

     需要注意分区数量和reducer数量关系

5. Combiner

     是一个本地的Reducer，运行在Map端来实现预聚合功能

     要么重写一个Reducer，要么复用已有的Reducer

6. Reducer

     四个泛型

     Mapper端相同的key，会进入同一Reducer

7. OutputFormat

     `FileOutputFormat、TextOutputFormat`，可以自定义输出格式
