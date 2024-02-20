## 启动

```shell
# 修改zk/conf 目录下zoo.cfg 数据存储目录
dataDir=/home/hadoop/zk/tmp

# bin下
$ zkServer.sh start
```



## 数据模型

是一个层次化的目录数据（树形结构），一旦确定节点类型，就不能修改

znode两种类型:

- 临时 ephemera

  客户端会话结束，节点就会被删除，该类型节点下不能有子节点

- 永久 persistent

  只要创建一直存在，还有一种是顺序节点

zonde特点：

- zonde数据有各自的版本号，可以通过代码或命令来显示
- 数据变化，版本号会增加；数据不宜过大（几K）
- 可以设置ACL
- 可以设置watcher



## 命令行

```shell
# 进入zk客户端
$ zkCli.sh

# 查询
# 列表查询
$ ls /zookeeper
# 数据查询
$ get /zookeeper

# 创建
# 节点及数据
$ create /tw hello-world
# 顺序节点
$ create -s /tw/seq test
Created /tw/seq0000000001
# 临时节点
$ create -e /tw/tmp test
$ get -s /tw/tmp
test
cZxid = 0x3 当前节点事务ID
ctime = Tue Jan 23 17:22:29 CST 2024
mZxid = 0x3 修改的事务ID
mtime = Tue Jan 23 17:22:29 CST 2024
pZxid = 0x3 最后更新子节点的事务ID
cversion = 0 子节点版本
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x100011edb8b0000
dataLength = 4
numChildren = 0

# 修改
$ set /tw new-world

# 删除
$ delete /tw/seq0000000001
# 带版本号删除
$ delete -v 1 /tw/seq0000000002
```



## 监听器

针对每个znode可以设置监听器，当监控的znode发生变化，会触发watcher

watcher是一次性的，触发后立即销毁

父子节点之间也可以进行监听

```shell
# 节点监测(NodeCreated、NodeDataChanged、NodeDeleted)
$ stat -w /aa

# 子目录监测(NodeChildrenChanged)，子目录新增、删除触发，修改不触发
$ ls -w /aa
```



## 集群

修改zk00*的conf：

```shell
dataDir=/home/hadoop/zk/tmp/zk001
clientPort=2181

# 2888与leader通信端口，3888选举端口
server.1=hadoop000:2888:3888
server.2=hadoop000:2889:3889
server.3=hadoop000:2890:3890
```

在各自dataDir目录中，`vim myid`存入各自序号

```shell
$ vim zk001/myid
1
```

启动各自`bin`目录下的`./zkServer.sh start`可以通过`./zkServer.sh status`查看集群每个节点状态

客户端连接集群：`./zkCli.sh -server hadoop000:2181,hadoop000:2182,hadoop000:2183`

