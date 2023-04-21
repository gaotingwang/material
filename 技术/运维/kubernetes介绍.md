开始 Kubernetes 之旅，更好地了解其内部原理

## 容器

在不知道容器是什么的情况下谈论容器编排器（Kubernetes）是没有意义的！

![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/%E5%AE%B9%E5%99%A8.png)

“容器”是一个用来存放，放入的所有物品的容器。像应用程序代码，依赖库以及它的依赖关系一直到内核。

这里的关键概念是隔离。将所有内容与其余内容隔离开，以便更好地控制它们。

容器提供三种隔离类型：

- **工作区隔离（流程，网络）**
- **资源隔离（CPU，内存）**
- **文件系统隔离（联合文件系统）**

考虑一下像 VM 一样的容器。它们精简，快速（启动）且体积小。而且，所有这些都没有构建起来。

取而代之的是，它们使用 Linux 系统中存在的结构（例如 cgroups，namespaces），在其上构建了一个不错的抽象。

现在知道什么是容器了，很容易理解为什么它们很受欢迎。不仅可以分发应用程序的二进制/代码，还可以以实用的方式交付运行应用程序所需的整个环境。

因为可以将容器构建为非常小的单元，解决“在我的机器上工作”问题的完美解决方案。

## 什么时候使用 Kubernetes

容器一切都很好，软件开发人员的生活现在要好很多。那么，为什么需要另一项技术，如 Kubernetes 这样的容器编排工具呢？

当进入某个状态时，需要用到它来管理众多容器：

**问：**我的前端容器在哪里，我要运行几个？
**答：**很难说，使用容器编排工具。

**问：**如何使前端容器与新创建的后端容器对话？
**答：**对 IP 进行硬编码，或者，使用容器编排工具。

**问：**如何进行滚动升级？
**答：**在每个步骤中手动握住，或者，使用容器编排工具。

## 为什么选择 Kubernetes

有很多容器编排工具，例如 Docker Swarm，Mesos 和 Kubernetes。这里选择是 Kubernetes（因此有了本文），因为 Kubernetes 是……

![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/%E5%AE%B9%E5%99%A8%E7%BC%96%E6%8E%92.png)

就像乐高积木一样，它不仅具有大规模运行容器编排所需的组件，而且还具有使用自定义组件交换内部和外部不同组件的灵活性。

想要拥有一个自定义的调度程序，也很方便。需要具有新的资源类型，编写一个 CRD。此外，社区非常活跃，并且工具迅速发展。

## Kubernetes 架构

![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/kubernetes%20HA.png)

每个 Kubernetes 集群都有两种类型的节点，主节点和工作节点。顾名思义，主节点是在工作程序运行有效负载（应用程序）的地方控制和监视群集。

集群可以与单个主节点一起工作，但是最好拥有三个以实现高可用性（称为 HA 群集）。

让我们仔细看一下主节点及其组成：

![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/k8s%E4%B8%BB%E8%8A%82%E7%82%B9.png)

1. **etcd：**数据库，用于存储有关 Kubernetes 对象，其当前状态，访问信息和其他集群配置信息的所有数据。

2. **API Server：**RESTful API 服务器，公开端点以操作整个集群。主节点和工作节点中的几乎所有组件都与该服务器通信以执行其职责。

3. **调度程序：**负责决定哪个有效负载需要在哪台机器上运行。

4. **控制管理器：**这是一个控制循环，它监视集群的状态（通过调用 API 服务器来获取此数据）并采取措施将其置于预期状态。

   ![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/k8s%20worker%E8%8A%82%E7%82%B9.png)

   1. **kubelet：**是工作节点的心脏。它与主节点 API 服务器通信并运行为其节点安排的容器。

   2. **kube-proxy：**使用 IP 表/IPVS 处理 Pod 的网络需求。

   3. **Pod：**运行所有容器的 Kubernetes 的功劳。如果没有 Pod 的抽象，就无法在 Kubernetes 中运行容器。Pod 添加了对容器之间的 Kuberenetes 联网方式至关重要的功能。

      ![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/k8s%20pod.png)

      一个 Pod 可以有多个容器，并且在这些容器中运行的所有服务器都可以将彼此视为本地主机。

      这使得将应用程序的不同方面分离为单独的容器，并将它们全部作为一个容器加载在一起非常方便。

      有多种不同的 Pod 模式，例如 sidecar、proxy 和ambassador，可以满足不同的需求。查看[这篇文章](https://matthewpalmer.net/kubernetes-app-developer/articles/multi-container-pod-design-patterns.html)可以了解有关它们的更多信息。

      Pod 网络接口提供了一种将其与同一节点和其他工作节点中的其他 Pod 通信的机制。

      ![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/pod.png)

      而且，每个 Pod 都将分配有自己的 IP 地址，kube-proxy 将使用该 IP 地址来路由流量，而且此 IP 地址仅在群集中可见。

      所有容器也都可以看到安装在容器内的卷，有时可以使用这些卷在容器之间进行异步通信。

      例如，假设你的应用是照片上传应用（例如 Instagram），它可以将这些文件保存在一个卷中，而同一 Pod 中的另一个容器可以监视该卷中的新文件，并开始对其进行处理以创建多种尺寸，将它们上传到云存储。

## 控制器

在 Kubernetes 中，有很多控制器，例如 ReplicaSet，Replication Controllers，Deployments，StatefulSets 和 Service。

这些是以一种或另一种方式控制 Pod 的对象。让我们看一些重要的。

### ReplicaSet

![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/replicaset.png)

*ReplicaSet 做自己擅长的事情，复制 Pod*

该控制器的主要职责是创建给定 Pod 的副本，如果 Pod 因某种原因死亡，则会通知该控制器，并立即跳入操作以创建新的 Pod。

### Deployment

![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/deployment.png)

*试图控制 ReplicaSet 的部署（头发凌乱）*

部署是一个高阶对象，它使用 ReplicaSet 来管理副本。它通过放大新的 ReplicaSet 和缩小（最终删除）现有的 ReplicaSet 来提供滚动升级。

### Service

![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/service.png)

*表示为无人机的服务，将数据包传递到相应的 Pod*

服务是一个控制器对象，其主要职责是在将“数据包”分发到相应节点时充当负载平衡器。

基本上，它是一种控制器构造，用于在工作节点之间对相似的 Pod（通常由 Pod 标签标识）进行分组。

假设你的“前端”应用程序想与“后端”应用程序通信，则每个应用程序可能有许多正在运行的实例。

你不必担心对每个后端 Pod 的 IP 进行硬编码，而是将数据包发送到后端服务，然后由后端服务决定如何进行负载平衡并相应地转发。

**PS：**请注意，服务更像是一个虚拟实体，因为所有数据包路由均由 IP 表/IPVS/CNI 插件处理。

它只是使它更容易被视为一个真正的实体，让它们脱颖而出以了解其在 Kubernetes 生态系统中的作用。

### Ingress

![图片](https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/ingress.png)

*进入一个浮动平台，所有数据包都通过该平台流入集群*

入口控制器是与外界联系的单点，可以与集群中运行的所有服务进行对话。这使我们可以轻松地在单个位置设置安全策略，监视甚至记录日志。

**PS：**Kubernetes 中还有很多其他控制器对象，例如 DaemonSets，StatefulSets 和 Jobs。



原文链接：

```
https://medium.com/tarkalabs/know-kubernetes-pictorially-f6e6a0052dd0
```