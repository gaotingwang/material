## 介绍

### 核心组件及职责

YARN（RM + NMs）：

- ResourceManager

  整个集群同一时间点只有一个是active，负责集群**资源**的统一管理和调度

  启动和监控AM，如果有AM挂了，RM会负责在NM上启动该AM

  监控NM，接收其心跳信息，NM挂了，该NM上的任务如何处理会告知AM

- NodeManager

  负责各自节点的管理和使用

  周期性向RM汇报本节点上的资源使用情况，及Container的运行状态

  接收并处理RM关于Container的启停命令

  接收处理来自AM的命令

- ApplicationMaster

  每个应用程序一个AM

  为应用程序向RM申请资源，任务分配

  与NM通信，启停Container

  任务的启停、重试都是由AM进行负责

- Container

  所有任务（AM、MT、RT）都是运行在Container中，是对任务运行环境的抽象

  - 任务运行资源：节点、内存、CPU
  - 任务启动命令

- Client

  提交、杀死操作

### 工作原理

![YARN工作原理](https://gtw.oss-cn-shanghai.aliyuncs.com/BigData/Hadoop/YARN%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.png)

1. Client想YARN提交应用
2. RM为该应用分配第一个Container
3. RM与NM通信，要求该NM在这个Container中运行AM
4. AM向RM注册，Client可以通过RM查询该应用运行状态；AM为该应用到RM申请资源（MT、RT）
5. AM根据资源分配情况在指定NM上启动Container执行Task



## 使用

### 环境部署

核心文件配置：

`etc/hadoop/mapred-site.xml`:

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
```

`etc/hadoop/yarn-site.xml`:

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

开启作业历史服务器

`etc/hadoop/mapred-site.xml`:

```xml
<configuration>
    <!-- 历史服务器地址 -->
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>localhost:10020</value>
    </property>
    <!-- 历史服务器web访问地址 -->
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>localhost:19888</value>
    </property>
</configuration>
```

`etc/hadoop/yarn-site.xml`:

```xml
<configuration>
    <!-- 开启日志聚集功能 -->
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
    <!-- 设置日志聚集服务器地址 -->
    <property>
        <name>yarn.log.server.url</name>
        <value>localhost:19888/jobhistory/logs</value>
    </property>
    <!-- 设置日志保留时间为7天 -->
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>604800</value>
    </property>
</configuration>
```

启停：

```shell
# bin 目录下
$ mapred --daemon start historyserver

# sbin 目录下
$ ./start-yarn.sh

$ ./stop-yarn.sh
```

web访问：http://localhost:8088/

### 核心命令

```shell
# 应用程序相关
$ yarn app [options]

# 应用程序日志
$ yarn logs -applicationId <application ID> [options]

# 查看容器状态
$ yarn container [options]

# 查看节点
$ yarn node [options]

# 刷新scheduler队列
$ yarn rmadmin -refreshQueues
```

### 调度器

Client会同一时间向RM提交多个应用程序，应用执行顺序就涉及到调度器

- FIFO Scheduler

  先进先出

- Capacity Scheduler

  适用于多租户共享大集群

  在`yarn-site.xml`配置ResourceManager使用CapacityScheduler：

  ```xml
  	<property>
          <name>yarn.resourcemanager.scheduler.class</name>
          <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
      </property>
  ```

  `capacity-scheduler.xml` is the configuration file for the `CapacityScheduler`：

  ```xml
  <property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <!-- root 下有a,b,c三个子队列 -->  
    <value>a,b,c</value>
    <description>The queues at the this level (root is the root queue).
    </description>
  </property>
  
  <!-- a 队列使用总容量的30% --> 
  <property> 
      <name>yarn.scheduler.capacity.root.a.capacity</name>
      <value>30</value>
      <description>A queue target capacity.</description>
  </property>
  <!-- a 队列使用总容量的最大60% --> 
  <property>
      <name>yarn.scheduler.capacity.root.a.maximum-capacity</name>
      <value>60</value>
      <description>
          The maximum capacity of the A queue.
      </description>
  </property>
  
  <property>
      <name>yarn.scheduler.capacity.root.a.state</name>
      <value>RUNNING</value>
      <description>
          The state of the a queue. State can be one of RUNNING or STOPPED.
      </description>
  </property>
  
  <property>
      <name>yarn.scheduler.capacity.root.default.acl_submit_applications</name>
      <value>*</value>
      <description>
          The ACL of who can submit jobs to the default queue.
      </description>
  </property>
  ```

- Fair Scheduler

指定队列：

```shell
# 在share/hadoop/mapreduce
$ hadoop jar hadoop-mapreduce-examples-3.3.6.jar -Dmapreduce.job.queuename=a pi 2 3
```

