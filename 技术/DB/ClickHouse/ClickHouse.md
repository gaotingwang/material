## 初始化

参考链接：https://www.cnblogs.com/dreamfly2016/p/16087569.html

安装部署：

```shell
$ docker pull clickhouse/clickhouse-server:<tag>

启动clickhouse-server：
- 容器中配置目录: /etc/clickhouse-server
- 需要提前准备好配置放到挂载目录，否则会有启动报错： Configuration file '/etc/clickhouse-server/config.xml' isn't readable by user with id '101'
$ docker run -d --name clickhouse-server --ulimit nofile=262144:262144 \
  -p 18123:8123 -p19000:9000 \
  -v $(pwd)/clickhouse/data:/var/lib/clickhouse/ \
  -v $(pwd)/clickhouse/logs:/var/log/clickhouse-server/ \
  -v $(pwd)/clickhouse/conf:/etc/clickhouse-server/clickhouse-server \
  --platform linux/amd64 \
  clickhouse/clickhouse-server:<tag>

启动clickhouse-client：
clickhouse-client命令参数：
- --host
- --port
- -m 运行输入多行SQL语句，不加-m，SQL只能写在一行
- -q 可以直接跟SQL语句
$ docker exec -it clickhouse-server clickhouse-client

$ docker exec -it 容器ID sh 
```

核心目录：

```
conf:
	/etc/clickhouse-server/
        └── config.xml

lib:
	/var/lib/clickhouse/
        └── data 数据
            └── /test_database 数据库
            	└── /test_table 表
                    ├── size.json
                    └── 以列为单位的数据文件.bin
        └── metadata 元数据
            ├── test_database.sql
            └── /test_database 
            	└── /test_table.sql

log:
	/var/log/clickhouse-server/

bin:
	/usr/bin
```



## 数据类型

ClickHouse 所支持的表名都在 `system.data_type_families` 表中

- 数值类型

  - 整型

  - 浮点型

  - Decimal

    `Decimal(P,S)`, `Decimal32(S)`：P代表总共有多少位，S代表小数点后多少位

    精度变化：1. 加减法：`S = max(S1, S2)`；2. 乘法：`S = S1 + S2`；3.除法：`S = S1`

    数值发生溢出时：小数中的过多数字被丢弃（不是舍入的）；整数中的过多数字将导致异常

- 布尔类型

  没有单独的类型来存储布尔值。可以使用 UInt8 类型，取值限制为 0 或 1

- 字符串类型

  - String

  - FixedString

    使用定长的长度，一个中文占三个字符长度

  - UUID

    可使用`generateUUIDv4()`来生成UUID

- 日期时间类型

  - 日期

    日期中没有存储时区信息

  - 日期时间

    `DateTime`精确到秒，使用启动客户端或服务器时的系统时区

    `DateTime64(3, 'Asia/Istanbul')`，第一个参数指定时间秒的精度到小数点后几位，默认为3；第二个是指定时区

- Array类型

  数组索引是从1开始，不是0

  获取数据第一个元素：`select a[1] from t_table;`

- Tuple类型

  tuple中的元素类型可以是不同类型，至少包含一个元素，

  获取第一个元素：`select t.1 from t_table`

- Map类型

  `INSERT INTO table_map VALUES ({'key3':100}), ({});`
  `SELECT a['key3'] FROM table_map;`



## 内置函数

- 比较函数

  比较函数始终返回0或1，对于不同组的类型间不能够进行比较，必须进行转换

- 取整函数

  `floor(x, N)` 向下取整，N保留小数点后N位，N为负整数位；`ceil` 向上取整；`round` 四舍五入

- 类型转换函数

  toXXX()，可以转为对应类型

  toTypeName()， 获取类型

- 字符串函数

- 日期时间函数



## DDL & DML

### DML

- 插入数据：与`MySQL`相同

- 修改数据：`ALTER TABLE [db.]table [ON CLUSTER cluster] UPDATE column1 = expr1 [, ...] [IN PARTITION partition_id] WHERE filter_expr`

- 删除数据：

  `ALTER TABLE [db.]table [ON CLUSTER cluster] DELETE WHERE filter_expr`

  `DELETE FROM [db.]table [ON CLUSTER cluster] WHERE expr`，`DELETE FROM` requires the `ALTER DELETE` privilege:

  ```sql
  grant ALTER DELETE ON db.table to username;
  ```

  `TRUNCATE TABLE [IF EXISTS] [db.]name [ON CLUSTER cluster]`，The `TRUNCATE` query is not supported for View, File, URL, Buffer and Null table engines.

### DDL

与`MySQL`类似

### 分区

- 创建

  创建表时指定分区：

  ```sql
  CREATE OR REPLACE TABLE test
  (
      id UInt64,
      name String,
      birthday DateTime
  )
  ENGINE = MergeTree
  PARTITION BY toDate(birthday) -- 指定分区
  ORDER BY id;
  ```

  **可以通过`optimize table table_name final;` 来合并不同时间插入的相同分区数据**

  分区信息查询`select database,table,name,partition_id from system.parts;`

- 删除

  `ALTER TABLE table_name drop partition partition_id;`

- 复制

  `CREATE TABLE table_name_bak as table_name;` 根据`table_name`来创建一个备份表

  `ALTER TABLE table_name_bak REPLACE PARTITION '20210910' FROM table_name; ` 将`table_name`的`20210910`分区数据copy到`table_name_bak`中



## 引擎

不同引擎，存储方式不一样，支持功能不一样

在创建表的时候需要显示指定引擎

### Log Family

这些引擎是为了需要写入许多小数据量的表（少于一百万行），之后整体读出的场景而开发的

共性：

- 数据存储在磁盘上。
- 写入时将数据追加在文件末尾。
- 并发访问支持锁（进行`INSERT`查询，会进行表锁）
- **不支持`mutations`(修改、删除等)操作。**
- 不支持索引（意味着 `SELECT` 在范围查询时效率不高）。
- 非原子地写入数据

差异性：

- TinyLog（无数据标记）

  适合一次写入，多次读取，无法并发读取

  数据存储：每一个字段，一个文件，文件名为字段名.bin；sizes.json，使用json格式记录每个bin文件对应的数据大小信息

- StripeLog（所有数据都在一个数据文件中，可并行读取）

  数据全部被写入到data.bin文件中

  index.mrk：数据标记文件，保存了数据在data.bin中的位置信息，标记包含了已插入的每个数据块中每列的偏移量。

  带标记的文件使得可以并行的读取数据，这意味着 `SELECT` 请求返回行的顺序是不可预测的。可以使用 `ORDER BY` 子句对行进行排序

- Log（有数据标记、按字段进行文件存储）

  数据存储：每一个字段，一个文件，文件名为字段名.bin；

  与 `TinyLog` 的不同之处在于，有`__marks.mrk`标记文件，这些标记写在每个数据块上，并且包含offset，这些offset指示从哪里开始读取文件以便跳过指定的行数，这使得可以在多个线程中读取表数据，提升查询性能；

### Intergrations Engine

提供了多种方式来与外部系统集成，包括表引擎。像所有其他的表引擎一样，使用`CREATE TABLE`或`ALTER TABLE`查询语句来完成配置。配置的集成看起来像查询一个正常的表，但对它的查询是代理给外部系统的

```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name
(
    columns list...
)
ENGINE = JDBC(datasource_uri, external_database, external_table);


CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
) ENGINE = MySQL('host:port', 'database', 'table', 'user', 'password'[, replace_query, 'on_duplicate_clause'])
SETTINGS
    [ connection_pool_size=16, ]
    [ connection_max_tries=3, ]
    [ connection_wait_timeout=5, ]
    [ connection_auto_close=true, ]
    [ connect_timeout=10, ]
    [ read_write_timeout=300 ]
;
```

### Special Engine

- File

  File引擎以`data.{format}`格式存储数据，常见`format`格式有`CSV`,`TSV`,`JSONEachRow`

  `create table tableName (col1 type, col2 type...)engine =File(format);` 数据存储在对应的数据库表下:`/var/lib/clickhouse/data/{库名}/{表名}/data.{fomat}`

  在`data.{format}`文件中写入数据，可以通过`select `查询出；也可以通过`insert`写入数据到文件中。

  也可以在服务器文件系统中手动创建这些子文件夹和文件，然后通过 ATTACH（`attach table importCSV2(id Int8,name String,date Date)engine =File (CSV)`） 将其创建为具有对应名称的表，这样你就可以从该文件中查询数据了

- Merge

  自身并不存储数据，数据来自于库中符合条件的其他表中

  `CREATE TABLE ... Engine=Merge(db_name, tables_regexp)`

- Memory

  数据是存储在内存当中

### MergeTree Family

高性能：列式存储、自定义分区、稀疏主索引、次要数据滑动索引等

- MergeTree

  通过分片方式来插入大量数据，之后在后台合并分片数据（optimize），该方法比在插入过程中不断重写存储中的数据要高得多

  ```sql
  CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
  (
      name1 [type1] [[NOT] NULL] [DEFAULT|MATERIALIZED|ALIAS|EPHEMERAL expr1] [COMMENT ...] [CODEC(codec1)] [TTL expr1] [PRIMARY KEY],
      name2 [type2] [[NOT] NULL] [DEFAULT|MATERIALIZED|ALIAS|EPHEMERAL expr2] [COMMENT ...] [CODEC(codec2)] [TTL expr2] [PRIMARY KEY],
      ...
      INDEX index_name1 expr1 TYPE type1(...) [GRANULARITY value1],
      INDEX index_name2 expr2 TYPE type2(...) [GRANULARITY value2],
      ...
      PROJECTION projection_name_1 (SELECT <COLUMN LIST EXPR> [GROUP BY] [ORDER BY]),
      PROJECTION projection_name_2 (SELECT <COLUMN LIST EXPR> [GROUP BY] [ORDER BY])
  ) ENGINE = MergeTree()
  ORDER BY expr -- 必填字段，按照该字段排序，与分区没有关系 order by(id, name)
  [PARTITION BY expr]
  	-- 分区，可以是单字段，也可是多字段
  	-- 没有分区，文件名以all开头
  	-- 有分区，文件名以分区结果开头
  	-- 202106_1_4_2;202106为分区ID；1为MinBlockNum；4为MaxBlockNum；2为Level层级，即合并次数
  	-- 分区信息查询 select database,table,name,partition_id from system.parts;
  [PRIMARY KEY expr]
  	-- 主键，默认与order by相同，primary key 必须是 order by 中字段
  	-- 主键不具有去重功能
  [SAMPLE BY expr]
  	-- 抽样表达式
  [TTL expr
  	-- 过期数据
      [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx' [, ...] ]
      [WHERE conditions]
      [GROUP BY key_expr [SET v1 = aggr_func(v1) [, v2 = aggr_func(v2) ...]] ] ]
  [SETTINGS name=value, ...]
  	-- 设置额外参数
  ```

- ReplacingMergeTree

  可以根据`order by ` 对**同一分区**的数据进行去重，但去重时间不确定，可以通过`optimize` 手动触发

  可以通过`ReplacingMergeTree([ver])`来指定保留的版本，没有指定，保留同一组重复数据的最后一行；存在指定，保留同一组重复数据中ver字段最大的一行

- SummingMergeTree

  根据`order by ` 对**同一分区中**的数据进行求和，类似于`select id,sum(age) from table group by id;`



## 元数据管理

相关元数据信息在`system` database 中：

- 表信息：`tables`

  ```sql
  select 
  	database,name,engine,is_temporary,data_paths,metadata_path,create_table_query,partition_key,
  	sorting_key,primary_key,total_rows,total_bytes 
  from 
  	system.tables where database='test_database' and name='test_table';
  ```

- 列信息：`columns`

  ```sql
  select 
  	database,table,name,type,position
  from
  	system.columns where database='test_database' and table='test';
  ```

- 分区信息

  ```sql
  select database,table,name,partition_id from system.parts 
  	where database='test_database' and table='test';
  select partition,name,partition_id,engine,path,column,type,column_position from system.parts_columns 
  	where database='test_database' and table='test';
  select * from detached_parts
  ```

- 执行相关元数据

  ```sql
  -- 查询操作记录
  select *from processes;
  
  -- 查询日志
  select *from query_log limit 2;
  
  -- mutation相关操作
  select *from mutations
  ```

- 内置元数据

  ```sql
  -- 数据类型
  select * from data_type_families;
  
  -- 函数、表函数、聚合
  select * from functions;
  select * from table_functions;
  select * from aggregate_function_combinators;
  
  -- 文件format
  select * from formats;
  
  -- 时区
  select * from time_zones;
  
  -- 指标
  select * from metrics;
  ```

- 用户角色

  ```sql
  users、roles、role_grants、privileges、grants、current_roles、enabled_roles
  ```

- 其他信息

  - 磁盘：`select *from disks;`
  - 配额（分配每个用户使用资源大小）：`select *from quota_*;`
  - 集群：`select *from clusters;`

