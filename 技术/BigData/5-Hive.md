## 介绍

由Facebook开源，为了解决**海量结构化**的日志数据统计问题

底层支持多种不同的执行引擎（MapReduce、Spark、Tez），可以通过参数切换

构建在Hadoop之上的数据仓库：

- HDFS：Hive的数据可以存放在HDFS上
- MR：分布式执行引擎，Hive作业通过以MR方式运行
- YARN：统一的资源管理和调度

定义了一种类SQL语言：Hive QL，将SQL**翻译**成底层引擎对应的作业，并提交运行

支持压缩、存储格式、自定义函数

### 优缺点

适用场景：离线 / 批计算，延迟较大

- 优点：

  类SQL，易上手

  比MR编程简单

  内置非常多函数，也可以自定义函数

- 缺点：

  SQL表述能力有限

  作业延迟比较大，处理少量数据可能也会花费比较多时间

### 架构

![Hive架构](https://gtw.oss-cn-shanghai.aliyuncs.com/BigData/Hive/Hive%E6%9E%B6%E6%9E%84.png)

SQL ==> 作业 ==> YARN运行

访问方式：

- CLI：命令行
- JDBC/ODBC：代码方式 (Hive Server 2)
- WebUI：HUE/Zeppelin

Driver：

- 解析器：SQL(String) ==> AST(抽象语法树)
- 编译器：AST ==> 逻辑执行计划
- 优化器：逻辑执行计划进行优化
- 执行器：执行计划 ==> 底层引擎的作业（MR/Spark/Tez）

MetaSotre：元数据存储，Hive数据是存放在HDFS上的，Hive元数据是存放在MySQL中的

### Hive VS RDBMS

- 都是面向SQL
- 延时性
- 都支持事务，支持insert/update/delete（Hive 0.14版本后支持）
- 支持分布式
- 数据量：Hive PB级



## 部署

[MySQL安装参照](https://developer.aliyun.com/article/931051)

在`$HIVE_HOME/conf`目录下创建`hive-site.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false&amp;useUnicode=true&amp;characterEncoding=UTF-8&amp;allowPublicKeyRetrieval=true</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>root</value>
    </property>
</configuration>
```

给`$HIVE_HOME/lib`目录中放入mysql驱动包`mysql-connector-java.jar`

在MySQL中初始schema:

```shell
# bin目录下
$ ./schematool -dbType mysql -initSchema
```

与Hadoop log4j冲突:

```shell
# lib目录下
$ mv log4j-slf4j-impl-2.17.1.jar log4j-slf4j-impl-2.17.1.jar-bak
```

CLI方式访问：

```shell
# bin目录下
$ ./hive

# 提示Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient，metadata服务未开启导致
$ ./hive --service metastore &

# hive参数
# -e 可以带sql语句执行
$ hive -e "select * from t_user"
# -f sql语句过大过长，可以指定文件
$ hive -f test.sql
# -i 初始化sql文件，如udf
$ hive -i test.sql
```

HS2&beeline方式：

hiveserver2增加了权限控制，需要在hadoop的`core-site.xml`配置文件中配置，重启hadoop：

```xml
<property>
    <!-- 使用hadoop用户，就是hadoop.proxyuser.hadoop -->
    <name>hadoop.proxyuser.hadoop.hosts</name>
    <value>*</value>
</property>
<property>
    <name>hadoop.proxyuser.hadoop.groups</name>
    <value>*</value>
</property>
```

`$HIVE_HOME/conf/hive-site.xml`增加配置：

```xml
  <property>
      <name>hive.server2.enable.doAs</name>
      <value>false</value>
  </property>
```

```shell
# $HIVE_HOME/bin下
$ nohup hiveserver2 2>&1 &
$ beeline -u jdbc:hive2://hadoop000:10000 -n hadoop
```

### 参数设置

`set key;` 查看key值，`set key=value;` 设置key值，这种设置方式仅对当前session有效

也可在启动hive时指定：`hive --hiveconf hive.cli.print.header=true `

设置全局有效，在`$HIVE_HOME/conf/hive-site.xml`文件中配置即可

命令行参数：

- hive.cli.print.header 打印查询列的名字
- hive.cli.print.current.db 显示当前数据库名字

存储参数：

- hive.metastore.warehouse.dir 数据存储位置，默认在HDFS存储目录下的/user/hive/warehouse



## DDL & DML

### 数据模型

Hive数据是存放在HDFS上的，默认在HDFS存储目录下的/user/hive/warehouse目录下

数据存储是以文件夹方式进行组织，文件夹下存储的就是Hive表数据

`user/hive/warehouse/{database.db}/{table}/{partition}/{bucket}/000000_0` 如果没有分区或桶，则没有该级目录

### 表创建

```sql
-- 语法
CREATE TABLE [EXTERNAL] table_name
  col_name data_type COMMENT
  like table_name:复制表结构，不复制数据
  as: 复制表结构及数据
COMMENT: 表注释
PARTITIONED BY：指定分区
CLUSTERED BY： 4大by
ROW FORMAT： 可以指定分隔符
  DELIMITED FIELDS TERMINATED BY ','
STORED AS： 存储格式
  text
  orc
  parquet
LOCATION hdfs_path：指定表路径
```

### 内部表 VS 外部表

- 内部表：MANAGED_TABLE

  删除时，数据和元数据都会被删除

- 外部表：EXTERNAL_TABLE

  删除时，元数据会被删除，数据还在

```sql
-- 内外部表转换'EXTERNAL'='FALSE'
ALTER TABLE table_name SET TBLPROPERTIES('EXTERNAL'='TRUE');
```

`DROP TABLE table_name` 是删除表，`TRUNCATE TABLE table_name`是清除表数据（外部表无法执行）

### 加载数据方式

- 从文件加载

  ```sql
  -- 指定LOCAL则从本地加载，否则从HDFS加载
  -- 指定OVERWRITE表示覆盖，否则是追加
  LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)]
  ```

- CTAS：`CREATE TABLE table_name as SELECT * FROM other_tabel;`

- `INSERT INTO/OVERWRITE TABLE tablename1 select statement1 FROM from_statement;`此种方式要求表结构必须存在

- `insert into values`，此种方式不推荐，会生成MR作业，每次的插入都会生成小文件

导出数据文件：

```sql
-- 需要注意会将指定目录下内容全部清空，若有其他文件，也会被清除，然后写入数据文件
INSERT OVERWRITE [LOCAL] DIRECTORY '/home/hadoop/data' SELECT * FROM t_user;
```

集群之间的数据迁移：

```sql
-- 导出数据，会将表的数据及元数据导出到HDFS指定目录中
EXPORT TABLE table_name to 'PATH';
-- 导入数据，会根据元数据生成表加载数据
IMPORT TABLE target_table FROM 'PATH';
```

### 分区表

```sql
-- 注意，分区字段不能出现在建表的普通字段中
CREATE TABLE [EXTERNAL] table_name
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
...

-- 导入分区数据,其他加载数据方式同理
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename PARTITION (day='20240126');

-- 查询时条件最好带着分区字段
select * from table_name where day='20240126';

-- HDFS中分区数据存在对应分区目录下
-- 手动导入数据到分区目录下，需要刷新元数据信息
msck repair table table_name;

-- 查询表分区信息
show partitions table_name;

-- 删除分区
alter table table_name drop partition(col_name='value');
```

动态分区：

```sql
-- PARTITION()中只需要指定分区字段，不需要显示指定具体值
-- SELECT最后的字段必须是分区字段
-- hive.exec.dynamic.partition.mode=nonstrict,严格模式下要求必须有一个静态分区
INSERT OVERWRITE TABLE tablename1 PARTITION(col_name)
select statement1,col_name FROM from_statement;
```

### with...as...

`with...as...`需要定义一个sql片段，会将这个片段产生的结果集保存在内存中，后续的sql均可以访问这个结果集和,作用与视图或临时表类似

```sql
-- 同级的多个temp之间用
with temp1 as (
    select * from xxx
),temp2 as (
    select * from xxx
)

-- 嵌套
with temp2 as (
    with temp1 as (
        select * from xxx
    )
    select * from temp1
)
select * from temp2;

-- with...as...只能在一条sql中使用
with temp1 as (
    select * from xxx
)
select * from temp1;
select xxx from temp1; -- error! no table named temp1;
```



Hive函数
---

### 复杂数据类型

```
array_type
  : ARRAY < data_type >
 
map_type
  : MAP < primitive_type, data_type >
 
struct_type
  : STRUCT < col_name : data_type [COMMENT col_comment], ...>
 
union_type
   : UNIONTYPE < data_type, data_type, ... >  -- (Note: Available in Hive 0.7.0 and later)
```

- Array

  ```mysql
  create table hive_array(
  	name string,
  	location ARRAY<string>
  )
  -- 指定字段分隔符，集合分隔符
  ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' COLLECTION ITEMS TERMINATED BY ',';
  
  -- 集合数组获取
  select name,location[0] from hive_array ;
  -- sort_arry 对集合排序
  select * from hive_array where array_contains(location, 'beijing');
  ```

- Map

  ```sql
  create table hive_map(
  id int,
  name string,
  info MAP<string,string>,
  age int
  )
  -- 指定字段分隔符，集合分隔符, K-V分割符
  ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' COLLECTION ITEMS TERMINATED BY '#' MAP KEYS TERMINATED BY ':';
  
  -- Map中信息获取
  select id, name, info['father'] father, info['brother'] brother from hive_map;
  -- 获取所有key,同理value map_values
  select id, name, map_keys(info) from hive_map;
  ```

- Struct

  ```sql
  create table hive_struct(
  ip string,
  info STRUCT<name:string,age:int>
  )
  ROW FORMAT DELIMITED FIELDS TERMINATED BY '#' COLLECTION ITEMS TERMINATED BY ':';
  
  select ip,info.name from hive_struct;
  ```

### 内置函数

[内置函数](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF)

```sql
-- 查看内置函数
show functions;
-- 查看函数使用方式
desc function function_name;
desc function extended function_name;

-- 常用函数类别：日期时间、字符串、json处理（json_tuple、get_json_object）、parse_url_tuple、条件函数（nvl、isnull、if、case when）

-- 行转列
-- 使用聚合函数 collect_set 生成set集合，再通过contat_ws字符串拼接
select dept_no,contat_ws('|', collect_set(name)) from dept group by dept_no;

-- 列转行
-- explode(col) 将一列中复杂的array拆分成多行
-- lateral view udtf(exp) tablealias as columnalias
-- lateral view 配合UDTF函数使用，主要功能是将原本汇总在一条（行）的数据拆分成多条（行）成虚拟表，再与原表进行笛卡尔积，从而得到明细表。
select name, locat
from hive_arrays 
lateral view explode(location, '分隔符')) locations_table as locat;
```

### UDF(User Defined Function)函数

- UDF：一进一出
- UDAF(User Defined Aggregation Function)：多进一出
- UDTF(User Defined Table-Generating Function)：一进多出

定义UDF函数：

```java
public class UDFHelloNew extends GenericUDF {
    /**
     * 初始化
     */
    @Override
    public ObjectInspector initialize(ObjectInspector[] arguments) throws UDFArgumentException {
        // 参数校验
        if (arguments.length != 1) {
            throw new UDFArgumentLengthException("requires 1 argument, got " + arguments.length);
        }
        // todo 类型判断
        return PrimitiveObjectInspectorFactory.javaStringObjectInspector;
    }
    /**
     * 处理业务逻辑
     */
    @Override
    public Object evaluate(DeferredObject[] arguments) throws HiveException {
        String val = arguments[0].get().toString();
        return "Hello: " + val;
    }
    /**
     * 很少使用
     */
    @Override
    public String getDisplayString(String[] children) {
        return super.getStandardDisplayString("helloNew", children);
    }
}
```

如果定义UDTF函数，`extends GenericUDTF`

临时函数（会话中有效，会话重开后，函数失效）：

```sql
-- 将jar包scp至服务器目录后
add jar /home/hadoop/lib/pk-hadoop-1.0.jar
CREATE TEMPORARY FUNCTION function_name AS class_name;
```

永久函数：

```sql
-- 将udf函数打包成jar传入HDFS后, 创建永久函数
CREATE FUNCTION [db_name.]say_hello AS "com.gtw.hadoop.test.hive.udf.UDFHelloNew"
  USING JAR "hdfs://hadoop000:9000/lib/pk-hadoop-1.0.jar";
  
-- 会把这个函数注册到元数据中去，在FUNCS表中，可到mysql中查看
 
-- hive中试用
select ename,say_hello(ename) from emp;
```

官方函数入口点，都是在`org.apache.hadoop.hive.ql.exec.FunctionRegistry`进行注册的

### 窗口分析函数

[窗口分析函数](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+WindowingAndAnalytics)

`over()`生成一个窗口，期内可以指定窗口规格，简单理解就像一个滑动窗口，窗口一行一行增加记录变大，窗口内的计算结果也随之变化

```sql
select domain,times,traffic,
sum(traffic) over(partition by domain order by times) as pv1,
sum(traffic) over(partition by domain rows between unbounded preceding and current row) as pv2,
sum(traffic) over(partition by domain) as pv3,
sum(traffic) over(partition by domain order by times rows between 3 preceding and current row) as pv4,
sum(traffic) over(partition by domain order by times rows between 3 preceding and 1 following) as pv5,
sum(traffic) over(partition by domain order by times rows between current row and unbounded following) as pv6
from win01 order by domain,times;
```

ROWS与RANGE区别：
`RANGE`表示的是具体的值，比这个值小 value 的行，比这个值大 value 的行。
没有`ORDER BY`：那么就是当前分区的所有行都包含在框架中，因为所有行都会成为当前行的相同行

分析函数使用场景：

- NTILE

  对窗户内的数据切成几片，切分不均匀时，多出来的数据会先放入前面的分片中

  切分成两片：`select domain,times,traffic,NTILE(2) OVER(partition by domain order by times) from win_traffic order by domain,times;`

- ROW_NUMBER

  给窗口内分组后的数据，从1开始，生成一个递增序号

  `select domain,times,traffic,ROW_NUMBER() OVER(partition by domain order by traffic) from win_traffic order by domain;`

- RANK

  与`ROW_NUMBER`相似，当排名相同时，生成的序号相同，下次不同时会留有空位（1，2，3，3，5，6），序号最大数`=`总数

- DENSE_RANK

  与`RANK`相似，当排名相同时，生成的序号相同，不同时递增（1，2，3，3，4，5），所以序号最大数 `<=` 总数

- CUME_DIST

  `CUME_DIST` ：小于等于当前行的行数 / 分组内总行数

- PERCENT_RANK

  `PERCENT_RANK` ：分组内当前行的`RANK()` - 1 / 分组内总行数 - 1

窗口函数使用场景（函数需要传入column）：

- LAG

  LAG(column, N, default)用于统计窗口内往上取第N行的值，取不到则未default值，不指定为NULL

  `select domain,times,traffic,LAG(times, 2, '1970-01-01 00:00:00') OVER(partition by domain order by traffic) from win_traffic;`

- LEAD

  窗口内往下取第N行的值

- FIRST_VALUE

  `FIRST_VALUE(traffic)`窗口内第一个值，返回的都是一样的

- LAST_VALUE

  `LAST_VALUE(traffic)`窗口内最后一个值，返回是不一样的，因为窗口是不断变大的，最后一个值是不断变化的



调优
---

[配置参数文档](https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties)

### 开启压缩

Hive Query Parameters（也可配置在`hive-site.xml`中）：

```
SET hive.exec.compress.output=true
SET mapreduce.output.fileoutputformat.compress=true
SET mapreduce.output.fileoutputformat.compress.codec=com.hadoop.compression.lzo.LzoCodec
```

### 存储格式

[ORC存储格式](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ORC)

在创建表时`CREATE TABLE ... STORED AS ...`来指定存储格式：

- `CREATE TABLE ... STORED AS ORC [tblproperties ("orc.compress"="NONE")]` 可以通过`tblproperties`来指定orc是否压缩相关属性
- `ALTER TABLE ... [PARTITION partition_spec] SET FILEFORMAT ORC`
- `SET hive.default.fileformat=Orc`

### 抓取策略

是否交由YARN跑作业，由`hive.fetch.task.conversion`参数控制：

- `none`：禁用，都会跑MR作业
- `minimal`：SELECT *，在分区列上的 FILTER（WHERE 和 HAVING 子句），`limit` 这三个不会跑MR
- `more`：SELECT（including UDFs），FILTER ，仅限 LIMIT（包括 TABLESAMPLE、虚拟列）(UDTFs and lateral views are not yet supported – see [HIVE-5718](https://issues.apache.org/jira/browse/HIVE-5718).)

### 本地模式

`hive.exec.mode.local.auto=true`，让 Hive 自己决定是否以本地模式运行，默认为`false`，Hive判断依据：

- 最大输入数据量：`hive.exec.mode.local.auto.inputbytes.max=134217728`
- 最大task数量：`hive.exec.mode.local.auto.tasks.max=4`
- 最大输入文件数：`hive.exec.mode.local.auto.input.files.max=4`

### 严格模式

为了预防一些有风险的查询，如全表查询：order by不带limit、分区表不带分区条件等，线上推荐`hive.mapred.mode=strict`（2.x之后默认为strict）

### 四大By

[4大by总结官方文档](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+SortBy)

- order by

  全局排序，只能有一个ReduceTask来完成，不管是否设置了reduce个数

  与严格模式有关系

- sort by

  分区内有序，可以设置多个ReduceTask，仅保证每个reduce出来的结果有序

  与严格模式没关系

- distribute by

  ```sql
  -- 会按照reduce数量生成n份文件，文件按照distribute by分发而成
  insert overwrite local directory '/home/hadoop/hive_tmp/dist' row format delimited fields terminated by '\t' select * from temperature distribute by year sort by year,temp;
  ```

  控制map端如何分发数据到reduce端，类似于Partitioner的功能

  采用hash取模的方式

- cluster by

  ```sql
  insert overwrite local directory '/home/hadoop/hive_tmp/cluster2' row format delimited fields terminated by '\t' select * from temperature cluster by year;
  ```

  = distribute by xxx sort by xxx，同时具备了distribute by和sort by的功能

  但cluster by 不能指明降序

### 并行执行

`hive.exec.parallel=true`可开启并行作业，默认为false

开启并行作业的场景：如多表union的子查询，多个join的子查询，会被翻译成多个mr作业，多个mr作业之间并不相互依赖的情况下。从Hive 0.14开始，也适用于可以并行运行的移动任务，例如在多插入期间将文件移动到插入目标。

还有个设置多少个作业并行的参数：`hive.exec.parallel.thread.number=8`

### 推测执行

整个作业跑完所需要耗费的时间是由最慢的一个task 来决定的，决定task耗时影响的有：

- Node可能跑的有其他作业，负载较高
- 作业数据倾斜，key 的数据量特别大

不管是MR、Hive、Spark 都有推测式执行的概念：假设task 在Node1上已经跑了一定时间(有阈值)，这时会在另一个节点Node2上也启动一个相同的task，此时两个节点上的container 里面跑的task 是一样的，谁先跑完就以谁的结果为准，然后干掉另外一个没有跑完的task 。

`hive.mapred.reduce.tasks.speculative.execution=true`开启推测式执行

但这个功能也有一定的适用场景：如果某个task 数据倾斜，在另外一个节点再启一个，同样也是一样的，该挂还是要挂。

### MapTask个数设置

在MR中，MapTask 的数量 是在 `FileInputFormat` 中算出来的

影响MapTask个数的因素：

- 文件个数
- 文件大小
- block大小

可以通过`mapreduce.input.fileinputformat.split.maxsize`来设置切片大小的值，来控制Mapper数量。Mapper数过多，会导致JVM进程数过多，增加节点负载，一般情况下Mapper的数量不会动的

### ReduceTask个数设置

Reducer个数决定了输出文件的个数，reducer数量少可能reduceTask处理任务量多，reducer数量多跑起来慢，需要结合场景取舍

一般通过`mapred.reduce.tasks`来指定数量，也可以通过`hive.exec.reducers.bytes.per.reducer`来设定每个reducer执行作业的大小

ReduceTask个数是根据跑出来文件大小结果反推出来，设置一个合适的个数，不是盲目设置

### 数据倾斜

某个或者某几个key 对应的数据量过大或者过于集中，由于相同的 key 都是在相同ReduceTask 中运行的，所以会导致这些 RT 运行时间过长，**在shuffle 时**，才会产生数据倾斜。

通过`explain` 方式可以查看执行计划，一个`Stage` 可以理解为一个MR 作业

- group by场景

  解决思路：在 map 出来的 key 中，可以加个随机数前缀，然后 reducer 算完后，再用一个map ，得到真正的key ，最后再来一个reduce

  `hive.groupby.skewindata` 这个参数决定数据存在倾斜是否优化 group by 查询，默认不优化

  还有一个`hive.map.aggr` ，是否在 Hive Group By 查询中使用映射端聚合

- count(distinct)场景

  `select count(distinct city) from page_views;` 是由一个reduce作业处理的，直接修改reducer 数量没有作用

  通过SQL改写：`select count(1) from (select city from page_views group by city) t;`

- join场景

  `hive.auto.convert.join` 参数指定是否开启了根据输入文件大小将`common join`转为`mapjoin`的优化

  `hive.smalltable.filesize` or  `hive.mapjoin.smalltable.filesize` 来指定文件大小的阈值