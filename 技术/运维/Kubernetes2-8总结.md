## 架构原理



## 核心组件

- 集群管理入口：kube-apiserver

- 管理控制中心：kube-controller-manager

- 调度器：kube-scheduler

  承接Controller创建的Pod，为其安排可运行的目标node

- 配置中心：etcd

- 节点pod管家：kubelet

- 服务外部代理：kube-porxy

- 集群管理工具：kubectl

API Server 架构：

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/API%20Server.png" style="zoom:80%;" />

k8s 应用创建流程和监听机制：
<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/k8s%20%E5%BA%94%E7%94%A8%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B.png" style="zoom:80%;" />

Controller Manager：

- 副本控制器：Replication Controller
- 节点控制器：Node Controller
- 资源控制器：ResourceQuota Controller
  - 容器级别：可以对CPU和Memory进行限制
  - Pod级别：可以对一个Pod内所有容器的可用资源进行限制
  - Namespace级别：为Namespace级别的资源限制，包括：POD、RC、Service、ResourceQuota、Secret、PV数量
- 命名空间控制器：Namespace Controller
- Endpoints 控制器：Endpoints Controller
- 服务控制器：Service Controller

## Kubectl 常用命令

kubectl 语法：`kubectl [command] [TYPE] [NAME] [flags]`

命令分类：

- 命令式资源管理
  - 创建：create 创建资源、expose 暴露资源
  - 更新：scale 扩展资源、annotate 添加备注、label 标签
  - 删除：delete 删除资源
- 资源查看
  - get：显示一个或多个资源详细信息
  - describe：查看详情，聚合了相关资源的信息并输出
- 容器管理
  - log：查看容器log
  - exec：进入容器执行命令
  - cp：用于从容器与物理机文件的拷贝

k8s常用资源类型缩写：

- ing: ingress
- no: nodes
- ns: namespace
- rs: replicasets
- svc: services
- ep: endpoints
- po: pods

## Kompose 介绍

是一个将docker-compose文件快速转换为k8s能够部署文件的工具

使用条件：

- 必须具备一个已经安装好的k8s集群
- kubectl CTL工具必须能连接到已搭建好的k8s集群上

安装：

```shell
# Linux
curl -L https://github.com/kubernetes/kompose/releases/download/v1.28.0/kompose-linux-amd64 -o kompose

# macOS
curl -L https://github.com/kubernetes/kompose/releases/download/v1.28.0/kompose-darwin-amd64 -o kompose

chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose
```

使用：

```shell
# 转换docker-compose文件
$ kompose convert -f docker-compose.yaml

# 部署资源文件
$ kubectl apply -f *.yaml
```

