# 控制器 Controller

多种控制器管理 Pod 的生命周期

## ReplicaSet管理副本

ReplicaSet 的目的是维护一组在任何时候都处于运行状态的 Pod 副本的稳定集合。 因此，它通常用来保证给定数量的、完全相同的 Pod 的可用性。

工作原理：RepicaSet 是通过一组字段来定义的，包括一个用来识别可获得的 Pod 的集合的选择符、一个用来标明应该维护的副本个数的数值、一个用来指定创建新 Pod 以满足副本个数条件时要使用的 Pod 模板等等。每个 ReplicaSet 都通过根据需要创建和删除 Pod 以使得副本个数达到期望值，进而实现其存在价值。

ReplicaSet 通过 Pod 上的 metadata.ownerReferences 字段连接到附属 Pod，该字段给出当前对象的属主资源。 ReplicaSet 所获得的 Pod 都在其ownerReferences 字段中包含了属主 ReplicaSet 的标识信息。正是通过这一连接，ReplicaSet 知道它所维护的 Pod 集合的状态， 并据此计划其操作行为。

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
spec:
  # 副本个数
  replicas: 3
  # pod 选择符
  selector:
    matchLabels:
      app: nginx
  # pod 模板
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 80
```

```shell
$ kubectl get all
NAME              READY   STATUS              RESTARTS   AGE
pod/nginx-lf9lh   1/1     Running             0          6s
pod/nginx-mvqjl   0/1     ContainerCreating   0          6s
pod/nginx-s862n   0/1     ContainerCreating   0          6s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   2d4h

NAME                    DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx   3         3         1       6s
```

## 深入理解 Deployment

一个 Deployment 控制器为 Pods 和 ReplicaSets 提供声明式的更新能力。

根据 Deployment 中目标状态的描述，Deployment 控制器可以更改实际状态， 使其变为期望状态 。 可以定义 Deployment 以创建新的 ReplicaSet ， 或删除现有 Deployment ， 并通过新的Deployment 适配其资源。

以下是 Deployments 的典型使用场景：

- 创建 Deployment 以 ReplicaSet 上线。 ReplicaSet 在后台创建 Pods。 检查 ReplicaSet 的上线状态，查看其是否成功。
- 通过更新 Deployment 的 PodTemplateSpec，声明 Pod 的新状态 。 新的 ReplicaSet 会被创建，Deployment 以受控速率将 Pod 从旧 ReplicaSet 迁移到新 ReplicaSet。 每个新的 ReplicaSet 都会更新 Deployment 的修订版本。
- 如果 Deployment 的当前状态不稳定，回滚到较早的 Deployment 版本。 每次回滚都会更新Deployment 的修订版本。
- 扩大 Deployment 规模以承担更多负载。
- 暂停 Deployment 以应用对 PodTemplateSpec 所作的多项修改， 然后恢复其执行以启动新的上线版本。
- 使用 Deployment 状态来判定上线过程是否出现停滞。
- 清理较旧的不再需要的 ReplicaSet 。  

当使用 Deployment 时，不必担心还要管理它们创建的 ReplicaSet。Deployment 会拥有并管理它们的 ReplicaSet：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  # 副本数
  replicas: 2
  # pod 选择符
  selector:
    matchLabels:
      run: nginx-label
  # pod 模板
  template:
    metadata:
      labels:
        run: nginx-label
    spec:
      containers:
        - name: my-nginx
          image: nginx
          ports:
            - containerPort: 80
```

```shell
$ kubectl get all
NAME                            READY   STATUS    RESTARTS   AGE
pod/my-nginx-76d99854c5-2rg6f   1/1     Running   0          16s
pod/my-nginx-76d99854c5-dbrhn   1/1     Running   0          16s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   2d4h

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-nginx   2/2     2            2           16s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/my-nginx-76d99854c5   2         2         2       17s
```

## 有状态的应用 StatefulSets

StatefulSet 是用来管理有状态应用的工作负载 API 对象，可以管理 Deployment 和扩展一组 Pod，并且能为这些 Pod 提供序号和唯一性保证

和 Deployment 相同的是，StatefulSet 管理了基于相同容器定义的一组 Pod。但和 Deployment 不同的是，StatefulSet 为它们的每个 Pod 维护了一个固定的 ID。这些 Pod 是基于相同的声明来创建的，不能相互替换：无论怎么调度，每个 Pod 都有一个永久不变的 ID

StatefulSets 使用场景：

- 稳定的、唯一的网络标识符。
- 稳定的、持久的存储。
- 有序的、优雅的部署和缩放。
- 有序的、自动的滚动更新。

**稳定意味着 Pod 调度或重调度的整个过程是有持久性的**。如果应用程序不需要任何稳定的标识符或有序的部署、删除或伸缩，则应该使用由一组无状态的副本控制器提供的工作负载来部署应用程序，比如 Deployment 或者 ReplicaSet 可能更适用于无状态应用部署需要  

使用限制：

- 给定 Pod 的存储必须由 PersistentVolume 驱动基于所请求的 storage class 来提供，或者由管理员预先提供。

- 删除或者收缩 StatefulSet 并不会删除它关联的存储卷。这样做是为了保证数据安全，它通常比自动清除 StatefulSet 所有相关的资源更有价值。

- StatefulSet 当前需要`headless`服务来负责 Pod 的网络标识

  headless使用场景：有时候创建的服务不想走负载均衡，想直接通过pod-ip链接后端，可以使用`headless service`接可以解决。`headless service` 是将service的发布文件中的`clusterip=none` ，不让其获取`clusterip` ， **DNS解析的时候直接走pod**
  
- 当删除 StatefulSets 时，StatefulSet 不提供任何终止 Pod 的保证。

- 为了实现 StatefulSet 中的 Pod可以有序和优雅的终止，可以在删除之前将 StatefulSet 缩放为 0  

```yaml
# 先创建持久卷
kind: PersistentVolume
apiVersion: v1
metadata:
  name: datadir1
  labels:
    type: local
spec:
  storageClassName: my-storage-class
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/data1"
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: datadir2
  labels:
    type: local
spec:
  storageClassName: my-storage-class
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/data2"
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: datadir3
  labels:
    type: local
spec:
  storageClassName: my-storage-class
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/data3"
```

```yaml
apiVersion: v1
  kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      name: web
  # 使用Headless Service 用来控制网络域名
  # Stateful clusterIp=none，不让其获取clusterIP ， DNS解析的时候直接走pod
  clusterIP: None

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
template:
  metadata:
    labels:
      app: nginx # has to match .spec.selector.matchLabels
  spec:
    terminationGracePeriodSeconds: 10
    containers:
      - name: nginx
        image: registry.cn-beijing.aliyuncs.com/qingfeng666/nginx
        ports:
          - containerPort: 80
            name: web
        volumeMounts:
          - name: www
            mountPath: /usr/share/nginx/html
# 通过 PersistentVolumes 来提供稳定的存储
volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

```shell
# 查看 pod
$ kc get all -o wide
NAME        READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
pod/web-0   1/1     Running   0          29s   10.244.1.119   node1   <none>           <none>
pod/web-1   1/1     Running   0          24s   10.244.1.120   node1   <none>           <none>
pod/web-2   1/1     Running   0          21s   10.244.1.121   node1   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   8d    <none>
service/nginx        ClusterIP   None         <none>        80/TCP    29s   app=nginx

NAME                   READY   AGE   CONTAINERS   IMAGES
statefulset.apps/web   3/3     29s   nginx        registry.cn-beijing.aliyuncs.com/qingfeng666/nginx

# 每个副本都使用独立的存储data1-3目录， 并挂载在/usr/share/nginx/html路径
# 存储独立性验证
$ kubectl exec -it web-0 sh
$ cd /usr/share/nginx/html

# 在/tmp/data1 目录下创建一个文件1.txt，再回到 web-0 pod的/usr/share/nginx/html目录，看到目录下自动多了 1.txt 文件
# 登录 web-1 pod，进入相同目录，会发现该目录并没有 1.txt 文件，此时就证明了 statefulset 的 pod存储是互相独立的

# 访问该 nginx 服务
$ curl 10.244.1.119
```

## DeamonSet 后台任务

后台支撑型服务，主要是用来部署守护进程。后台支撑型服务的核心关注点在K8S集群中的节点(物理机或虚拟机)，可以保证每个节点上都有一个此类Pod运行。节点可能是所有集群节点，也可能是通过 nodeSelector 选定的一些特定节点。DaemonSet 确保所有符合条件的节点都运行该 Pod 的一个副本

典型的后台支撑型服务包括：存储、日志和监控等。在每个节点上支撑K8S集群运行的服务

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      containers:
        - name: fluentd-elasticsearch
          image: registry.cn-beijing.aliyuncs.com/qingfeng666/fluentd:v2.5.2
      terminationGracePeriodSeconds: 30
      # 在 DaemonSet 中的 Pod 模板必须具有一个值为 Always 的 RestartPolicy。 当该值未指定时，默认是 Always
      restartPolicy: Always
      # 如果指定了 .spec.template.spec.nodeSelector ，DaemonSet 控制器将在能够与 Node 选择器匹配的节点上创建 Pod
      # 指定 .spec.template.spec.affinity ，之后 DaemonSet 控制器将在能够与节点亲和性 匹配的节点上创建 Pod
      # 如果都没有指定，则 DaemonSet Controller 将在所有节点上创建 Pod
```

```shell
$ kubectl -n kube-system get po -o wide | grep fluentd
NAME                             READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
fluentd-elasticsearch-5z5jb      1/1     Running   0          20s   10.244.1.122     node1    <none>           <none>
fluentd-elasticsearch-cn6sv      1/1     Running   0          20s   10.244.0.39      master   <none>           <none>
```

与 DaemonSet 中的 Pod 进行通信的几种可能模式如下：

- NodeIP 和已知端口：DaemonSet 中的 Pod 可以使用 `hostPort` ，从而可以通过节点 IP 访问到Pod。客户端能通过某种方法获取节点 IP 列表，并且基于此也可以获取到相应的端口。
- DNS：创建具有相同 Pod 选择器的无头服务通过使用 `endpoints` 资源或从 DNS 中检索到多个 A 记录来发现 DaemonSet。
- Service：创建具有相同 Pod 选择器的服务，并使用该服务随机访问到某个节点上的守护进程（没有办法访问到特定节点）。  

删除一个 DaemonSet  时，如果使用 `kubectl` 并指定 `--cascade=false` 选项， 则 Pod 将被保留在节点上。接下来如果创建使用相同选择器的新 DaemonSet， 新的 DaemonSet 会收养已有的 Pod。 如果有 Pod 需要被替换，DaemonSet 会根据其 updateStrategy 来替换  

## Job 任务

Job是K8S中用来控制批处理型任务的API对象。批处理业务与长期伺服业务的主要区别就是批处理业务的运行有头有尾，而长期伺服业务在用户不停止的情况下永远运行。

Job管理的Pod根据用户的设置把任务成功完成就自动退出了。成功完成的标志根据不同的 spec.completions 策略而不同：单Pod型任务有一个Pod成功就标志完成；定数成功行任务保证有N个任务全部成功；工作队列性任务根据应用确定的全局成功而标志成功。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
        - name: pi
          image: registry.cn-beijing.aliyuncs.com/google_registry/perl:5.26
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(100)"]
      restartPolicy: Never
  # 任务失败重试次数，默认6
  backoffLimit: 4
```

```shell
$ kubectl get all
NAME           READY   STATUS      RESTARTS   AGE
pod/pi-gjc88   0/1     Completed   0          8m3s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.1.0.1     <none>        443/TCP   2d7h

NAME           COMPLETIONS   DURATION   AGE
job.batch/pi   1/1           5m21s      8m3s

# 选择算符与 Job 的选择算符相同。 --output=jsonpath 
$ pods=$(kubectl get pods --selector=job-name=pi --output=jsonpath='{.items[*].metadata.name}')
$ kubectl logs $pods
```

CronJob 用来执行定时任务，每次执行完，就会多出一个Completed的pod

