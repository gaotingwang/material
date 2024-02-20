## 介绍

### 假设和目标

部署在廉价机器，遇到一些问题，如何进行错误检查、快速自动恢复，这是其架构核心
针对批处理/离线处理，注重高吞吐而不是低延迟
大数据集，带宽和大集群
一次写入多次读取，不能对文件的任意位置数据进行修改
移动计算优于移动数据，减少数据在节点间的传递可以有效提高吞吐、降低带宽使用

### 架构（主 / 从）

- NameNode

  一个主节点服务，用于管理文件系统命名空间

  执行文件系统命名空间操作：打开、关闭、重命名文件或路径

  记录数据块对应的DN

- DataNode

  多个存储数据节点，负责读写服务

  一个文件会被拆分成多个block，存储在DN上

  执行NN发起的创建、删除副本的指令

### 副本

block的大小、副本数都是可配置的。NN会监测DN的心跳，DN定期汇报块信息给NN

### 优缺点

- 优点：

  构建在廉价机器

  多副本，高容错

  存大数据

- 缺点：

  不适合存小文件

  支持追加，不支持文件随机修改

  不适合低延迟操作



## 使用

### 部署

- 单节点部署

  目录介绍：

  ```
  hadoop:
  	bin: 客户端脚本
  	sbin: 服务端脚本
  	logs: 相关日志
  	etc/hadoop: 相关配置文件
  	share/hadoop: 官方自带的例子
  ```

  设置用户：

  ```shell
  # 创建hadoop用户
  $ useradd -m hadoop -s /bin/bash
  # 修改密码
  $ passwd hadoop
  # 增加管理员权限
  $ usermod -G root hadoop
  
  #设置管理员或用户组权限
  $ visudo
  方法一
  找到以下 一行 去除 前缀#号
  
  %wheel  ALL=(ALL)       ALL
  
  方法二
  在root 那行增加 hadoop一行，如下所示
  
  root    ALL=(ALL)       ALL
  hadoop    ALL=(ALL)       ALL
  应用设置
  
  方法一: 退出当前用户，改用hadoop登录，用命令su –，即可获得root权限进行操作
  方法二：重新启动系统
  ```

  核心文件配置：

  edit the file etc/hadoop/hadoop-env.sh 设置JAVA_HOME:   

  ```sh
  # set to the root of your Java installation  
  export JAVA_HOME=/usr/java/latest
  ```

  etc/hadoop/core-site.xml:

  ```xml
  <configuration>
      <property>
          <name>fs.defaultFS</name>
          <value>hdfs://localhost:9000</value>
      </property>
      <property>
      	<name>hadoop.tmp.dir</name>
      	<value>/home/hadoop/hadoop/tmp</value>
      </property>
  </configuration>
  ```

  etc/hadoop/hdfs-site.xml:

  ```xml
  <configuration>
      <property>
          <name>dfs.replication</name>
          <value>1</value>
      </property>
      <property>
          <name>dfs.datanode.use.datanode.host</name>
          <value>true</value>
      </property>
  </configuration>
  ```

  启停：

  ```shell
  #格式化目录，仅第一次执行
  $ bin/hdfs namenode -format
  
  # sbin 目录下
  $ ./start-dfs.sh
  
  # 看进程是否启动成功
  $ jpa
  
  $ ./stop-dfs.sh
  ```

  web访问：http://localhost:9870/

- 集群部署

  HDFS: NN	DN

  YARN: RM	NM

  1. 准备三台服务器，新建hadoop用户设置自己密码

     hadoop001：NN	DN	NM

     hadoop002：RM	DN	NM

     hadoop003：DN	NM

  2. 每个机器上配置/etc/hosts

  3. 服务器之间免密码登录

     ```shell
     $ ssh-keygen -t rsa
     $ ssh-copy-id hadoop001
     $ ssh-copy-id hadoop002
     $ ssh-copy-id hadoop003
     ```

  4. 集群部署

  

### 命令行使用

配置环境变量：

```sh
export HADOOP_HOME=/home/hadoop/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
```

命令使用：

```shell
# 建议将HADOOP_HOME/bin加入到环境变量中
# hadoop 文件系统相关命令为 HADOOP_HOME/bin/hadoop fs <args>
$ hadoop fs <args>
# 文件上传
	-moveFromLocal 操作之后客户端本地文件不存在
	-copyFromLocal 客户端本地文件还存在
	-put 同-copyFromLocal
	-appendToFile 追加文件内容到已存在文件末尾
# 文件内容查看
	-cat
	-text
	-head
	-tail
# 文件下载
	-copyToLocal 从 HDFS copy 数据到客户端
	-get 等价
# 其他，与linux常用命令一样
	-ls
	-du 
	...
```

### Java API 使用

HDFS API 编程的入口点为：`FileSystem`

```java
System.setProperty("HADOOP_USER_NAME", "root");

Configuration configuration = new Configuration();
configuration.set("fs.defaultFS", "hdfs://localhost:9000");
configuration.set("dfs.replication", "1");

FileSystem fileSystem = FileSystem.get(configuration);
fileSystem.mkdirs();
...
```

### 安全模式

启动时，NameNode会加载元数据信息，或副本数未达到副本因子要求，会进入安全模式

安全模式下，可以读取，但无法写入



## 源码分析

1. 入口类：

   根据启动脚本找到NameNode入口类为`org.apache.hadoop.hdfs.server.namenode.NameNode`，DataNode入口类为`org.apache.hadoop.hdfs.server.datanode.DataNode`

2. 找到`main`方法

3. 一定先要看类上的注释，对阅读理解很有帮助



