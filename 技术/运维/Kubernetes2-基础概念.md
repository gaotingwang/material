# 调度单元Pod

编写`yaml`文件，可以给编辑器安装`kubernetes`插件，提供了在线动态模板，该模板也可以进行自行修改，输入k接口调用模板，常见配置类型的预定义模板：

```
kcm ：ConfigMap
kdep：Deployment
kpod：Pod
kres：Generic resource
kser：Service
```

## 创建 Nginx Pod

- 编写nginx pod的yaml

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: my-nginx
    labels:
      app: my-nginx
  spec:
    containers:
      - name: my-nginx
        image: nginx
        # 为容器的生命周期事件设置处理函数
        lifecycle:   	
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
          preStop:
            exec:
              command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
          - containerPort: 80
  ```

- 创建nginx pod

  ```shell
  $ kubectl apply -f nginx.yaml
  ```

- 查看 Pod 情况  

  ```shell
  $ kubectl get pod
  
  # -w 参数可以监控pod状态
  $ kubectl get pod -w
  
  # 查看容器启动情况
  $ kubectl describe pod my-nginx
  
  # 进入容器查看执行
  $ kubectl exec -it my-nginx sh
  
  # 查看pod日志
  $ kubectl logs myapp
  
  # -c 查看pod中容器的日志
  $ kubectl logs myapp -c my-nginx
  
  # 查看容器分配的IP
  $ kubectl get pods -l run=nginx-label -o yaml | grep podIP
  ```

## Pod实现原理

- 什么是 Pod

  Pod 的共享上下文包括一组 Linux 名字空间、控制组（cgroup）和可能一些其他的隔离方面，即用来隔离 Docker 容器的技术。 在 Pod 的上下文中，每个独立的应用可能会进一步实施隔离。 

  就 Docker 概念的术语而言，Pod 类似于共享名字空间和文件系统卷的一组 Docker 容器。 

  说明： 除了 Docker 之外，Kubernetes 支持很多其他容器运行时， Docker 是最有名的运行时， 使用 Docker 的术语来描述 Pod 会很有帮助。 

- Pod 生命期

  和一个个独立的应用容器一样，Pod 也被认为是相对临时性（而不是长期存在）的实体。 

  Pod 会被创建、赋予一个唯一的 ID（UID）， 并被调度到节点，并在终止（根据重启策略）或删除之前一直运行在该节点。 如果一个节点死掉了，调度到该节点 的 Pod 也被计划在预定超时期限结束后删除。

- 使用 Pod

  通常不需要直接创建 Pod，甚至单实例 Pod。 相反，会使用诸如 Deployment 或 Job 这类工作负载资源来创建 Pod。如果 Pod 需要跟踪状态， 可以考虑`StatefulSet` 资源

- Pod 网络  

  每个 Pod 都在每个地址族中获得一个唯一的 IP 地址。 Pod 中的每个容器共享网络名字空间，包括IP 地址和网络端口。 Pod 内的容器可以使用 localhost 互相通信。 当 Pod 中的容器与 Pod 之外的实体通信时，它们必须协调如何使用共享的网络资源 （例如端口）  

Kubernetes 集群中的 Pod 主要有两种用法：

1. 运行单个容器的 Pod。“每个 Pod 一个容器” 模型是最常见的 Kubernetes 用例； 在这种情况下，可以将 Pod 看作单个容器的包装器，并且 Kubernetes 直接管理 Pod，而不是容器。
2. 运行多个协同工作的容器的 Pod。 Pod 可能封装由多个紧密耦合且需要共享资源的共处容器组成的应用程序。 这些位于同一位置的容器可能形成单个内聚的服务单元 —— 一个容器将文件从共享卷提供给公众， 而另一个单独的“边车”（sidecar）容器则刷新或更新这些文件。 Pod 将这些容器和存储资源打包为一个可管理的实体。

  ## 生命周期

- 容器阶段 Phase

  1. Pending（挂起）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。 
  2. Running（运行中）：Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。 
  3. Succeeded（成功）：Pod 中的所有容器都已成功终止，并且不会再重启。 
  4. Failed（失败）：Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止。 
  5. Unknown（未知）：因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败

- 容器状态 Status

  一旦调度器将 Pod 分派给某个节点，kubelet 就通过容器运行时开始为 Pod 创建容器。 容器的状态有三种：Waiting（等待）、Running（运行中）和 Terminated（已终止）。  

  ```shell
  $ kubectl describe pod <pod 名称>
  ```

  1. Waiting （等待）：如果容器并不处在 Running 或 Terminated 状态之一，它就处在 Waiting 状态。 处于 Waiting 状态的容器仍在运行它完成启动所需要的操作：例如，从某个容器镜像仓库拉取容器镜像，或者向容器应用 Secret 数据等等。 当使用 kubectl 来查询包含 Waiting 状态的容器的 Pod 时，也会看到一个 Reason 字段，其中给出了容器处于等待状态的原因。
  2. Running（运行中）：Running 状态表明容器正在执行状态并且没有问题发生。 如果配置了 postStart 回调，那么该回调已经执行完成。 如果使用 kubectl 来查询包含 Running 状态的容器的 Pod 时，会看到关于容器进入 Running 状态的信息。
  3. Terminated（已终止）：处于 Terminated 状态的容器已经开始执行并且或者正常结束或者因为某些原因失败。 如果使用kubectl 来查询包含 Terminated 状态的容器的 Pod 时，会看到容器进入此状态的原因、退出代码以及容器执行期间的起止时间  

## 探针检查健康

探针是由 kubelet 对容器执行的定期诊断。 要执行诊断，kubelet 调用由容器实现的 Handler （处理程序）。

有三种类型的处理程序： 

- ExecAction： 在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。 
- TCPSocketAction： 对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功 的。
- HTTPGetAction： 对容器的 IP 地址上指定端口和路径执行 HTTP Get 请求。如果响应的状态码大于等于 200 且小于 400，则诊断被认为是成功的。 

每次探测都将获得以下三种结果之一：

- Success（成功）：容器通过了诊断。
- Failure（失败）：容器未通过诊断。
- Unknown（未知）：诊断失败，因此不会采取任何行动

### 何时使用

对于所包含的容器需要较长时间才能启动就绪的 Pod 而言，启动探针是有用的。 不再需要配置一个较长的存活态探测时间间隔，只需要设置另一个独立的配置选定， 对启动期间的容器执行探测，从而允许使用远远超出存活态时间间隔所允许的时长。

如果容器启动时间通常超出 initialDelaySeconds + failureThreshold × periodSeconds 总值，应该设置一个启动探测，对存活态探针所使用的同一端点执行检查。 periodSeconds 的默认值是 30 秒。

应该将其 failureThreshold 设置得足够高， 以便容器有充足的时间完成启动，并且避免更改存活态探针所使用的默认值。 这一设置有助于减少死锁状况的发生。

### HTTP 请求探针

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness.http
  labels:
    test: liveness
spec:
  containers:
    - name: liveness
      image: mirrorgooglecontainers/liveness
      args:
        - /server
      # 存活探针，如果处理程序返回失败代码，则 kubelet 会杀死这个容器并且重新启动它
      livenessProbe:
        httpGet:
          port: 8080
          path: /healthz
          httpHeaders:
            - name: Custom-Header
              value: Awesome
        # 在执行第一次探测前应该等待 3 秒
        initialDelaySeconds: 3
        # 指定了 kubelet 每隔 3 秒执行一次存活探测
        periodSeconds: 3

```

```shell
# 10 秒之后，通过看 Pod 事件来检测存活探测器已经失败了并且容器被重新启动了
$ kubectl describe pod liveness-http
```

## 创建包含Init容器的Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
    - name: myapp-container
      image: busybox:1.28
      command: ['sh', '-c', 'date && sleep 3600']
  initContainers:
    - name: init-container
      image: busybox:1.28
      command: ['sh', '-c', 'date && sleep 10']
```

每个 Pod 中可以包含多个容器， 应用运行在这些容器里面，同时 Pod 也可以有一个或多个先于应用容器启动的 Init 容器。Init 容器与普通的容器非常像，除了如下两点：

1. 它们总是运行到完成。

2. 每个都必须在下一个启动之前成功完成。

   如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。 然而，如果 Pod 对应的 restartPolicy 值为 Never，Kubernetes 不会重新启动 Pod。

3. 与普通容器的不同之处：

   Init 容器支持应用容器的全部属性和特性，包括资源限制、数据卷和安全设置。同时 Init 容器不支持`lifecycle`、`livenessProbe`、`readinessProbe` 和`startupProbe`， 因为它们必须在 Pod 就绪之前运行完成

## 容器启动命令设置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dependent-envars-demo
spec:
  containers:
    - name: dependent-envars-demo
      # 设置启动时要执行的命令
      # 如果在配置文件中设置了容器启动时要执行的命令及其参数，那么容器镜像中自带的命令与参数将会被覆盖而不再执行。
      command:
        - sh
        - -c
      # 设置命令的参数
      # 如果配置文件中只是设置了参数，却没有设置其对应的命令，那么容器镜像中自带的命令会使用该新参数作为其执行时的参数
      args:
        - printf UNCHANGED_REFERENCE=$UNCHANGED_REFERENCE'\n'; printf SERVICE_ADDRESS=$SERVICE_ADDRESS'\n'; printf ESCAPED_REFERENCE=$ESCAPED_REFERENCE
      image: busybox
      # 可以将环境变量作为命令的参数
      # 意味着可以将那些用来设置环境变量的方法应用于设置命令的参数，其中包括了 ConfigMaps 与 Secrets
      env:
        - name: SERVICE_PORT
          value: "80"
        - name: SERVICE_IP
          value: "172.17.0.1"
        - name: UNCHANGED_REFERENCE
          # 依赖环境变量PROTOCOL，必须先定义出来才行
          value: "$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
        - name: PROTOCOL
          value: "https"
        - name: SERVICE_ADDRESS
          # 引用变量需要加上$()，类似于 "$(VAR)" ，这是在 command 或 args 字段使用变量的格式要求
          value: "$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
        - name: ESCAPED_REFERENCE
          # $(VAR_NAME) 这样的值可以用两个 $ 转义，既： $$(VAR_NAME)。无论引用的变量是否定义，转义的引用永远不会展开。
          value: "$$(PROTOCOL)://$(SERVICE_IP):$(SERVICE_PORT)"
```

## 管理容器配额

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
    - name: myapp
      image: registry.cn-beijing.aliyuncs.com/qingfeng666/stress
      # 配置请求资源
      resources:
        # 最多限制使用 1 个 CPU
        limits:
          cpu: "1"
          memory: "128Mi"
        # 容器将会请求 0.5 个 CPU
        requests:
          cpu: "0.5"
          memory: "128Mi"
      ports:
        - containerPort: 80
      # -cpus "2" 参数告诉容器尝试使用 2 个 CPU，在有resources: limit时，此配置不生效
      args:
        - -cpus
        - "2"
```

## 使用亲和性调度节点

```shell
# 列出集群中的节点及其标签
$ kubectl get nodes --show-labels

# 选择一个节点，添加一个标签
$ kubectl label nodes <your-node-name> disktype=ssd
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    # 指定节点的亲和性
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        # 选取node类型：匹配label disktype = ssd的node节点
        nodeSelectorTerms:
          - matchExpressions:
            - key: disktype
              operator: In
              values:
                - ssd
  containers:
    - name: nginx
      image: nginx
      imagePullPolicy: IfNotPresent
```

## 将ConfigMap数据注入容器

```yaml
apiVersion: v1
# 配置ConfigMap
kind: ConfigMap
metadata:
  name: special-config
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm

---
apiVersion: v1
kind: Pod
metadata:
  name: config-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "env"]
      # 指定环境变量来源
      envFrom:
        - configMapRef:
           # ConfigMap 名称
           name: special-config
```

```shell
# 命令查看ConfigMap
$ kubectl get configmap
```

## 非root用户运行

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      # 大多数容器默认实际上以root用户身份运行,但在容器里不是真的具有root用户的所有权限
      # 通过privileged=true开启特权用户，就像：docker run -it --privileged busybox sh
      securityContext:
        privileged: true
      command: [ "/bin/sh", "-c", "env" ]
```

要为 Pod 设置安全性设置，可在 Pod 规约中包含 securityContext 字段。为 Pod 所设置的安全性配置会应用到 Pod 中所有 Container 上：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    # 指定 Pod 中的所有容器内的进程都使用用户 ID 1000 来运行
    runAsUser: 1000
    # 指定所有容器中的进程都以主组 ID 3000 来运行，忽略此字段，则容器的主组 ID 将是root（0）
    runAsGroup: 3000
    # 所有创建的文件也会划归用户 1000 和组 3000
    fsGroup: 2000
  volumes:
    - name: sec-ctx-vol
      emptyDir: {}
  containers:
    - name: sec-ctx-demo
      image: busybox
      command: [ "sh", "-c", "sleep 1h" ]
      volumeMounts:
        - name: sec-ctx-vol
          mountPath: /data/demo
      securityContext:
        # 是否允许权限提升
        allowPrivilegeEscalation: false
```

# Service对象实践

每个 Pod 都有自己的 IP 地址，但是在 Deployment 中，在同一时刻运行的 Pod 集合可能与稍后运行该应用程序的 Pod 集合不同。

这导致了一个问题： 如果一组 Pod（称为“后端”）为群集内的其他 Pod（称为“前端”）提供功能， 那么前端如何找出并跟踪要连接的 IP 地址，以便前端可以使用后端部分？

Service 定义：将运行在一组 Pods 上的应用程序公开为网络服务的抽象方法。即kubernetes通过Service暴露Pod对外的服务地址， Kubernetes 为 Pods 提供
自己的 IP 地址，并为一组 Pod 提供相同的 DNS 名， 并且可以在它们之间进行负载均衡

Service 在 Kubernetes 中是一个 REST 对象，和 Pod 类似。 像所有的 REST 对象一样，Service 定义可以基于 POST 方式，请求 API server 创建新的实例。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  # 假定有一组 Pod，它们对外暴露了 9376 端口，同时还被打上 app=MyApp 标签
  selector:
    app: myapp
  ports:
    - protocol: TCP
      # service对外暴露80端口
      port: 80
      # 默认情况下，targetPort 与 port 字段相同的值
      targetPort: 9376
```

```shell
# 查看service
$ kubectl get svc

# 查看指定service的详细信息
$ kubectl describe svc my-nginx
```

##  集群内 Pod 通信机制

Kubernetes 支持两种基本的服务发现模式 —— 环境变量和 DNS  

- 环境变量

  当 Pod 运行在 Node 上，kubelet 会为每个活跃的 Service 添加一组环境变量。 它同时支持 Docker links 、 简单的 `{SVCNAME}_SERVICE_HOST` 和 `{SVCNAME}_SERVICE_PORT` 变量。 这里 Service 的名称需大写，横线被转换成下划线。

  举个例子，一个名称为 “my-nginx” 的 Service 暴露了 TCP 端口 80， 同时给它分配了 Cluster IP 地址 10.1.153.168。这个 Service 生成了如下环境变量： 

  ```properties
  MY_NGINX_PORT=tcp://10.1.153.168:80
  MY_NGINX_SERVICE_HOST=10.1.153.168
  MY_NGINX_SERVICE_PORT=80
  MY_NGINX_PORT_80_TCP_ADDR=10.1.153.168
  MY_NGINX_PORT_80_TCP_PORT=80
  MY_NGINX_PORT_80_TCP_PROTO=tcp
  MY_NGINX_PORT_80_TCP=tcp://10.1.153.168:80
  ```

  **说明：**在使用环境变量方法将端口和群集 IP 发布到客户端Pod 时，必须在客户端 Pod 创建之前创建服务。 否则，这些客户端 Pod 将不会设定其环境变量。即需要先创建Service，后创建Pod才会生效

- DNS

  Kubernetes DNS 在群集上调度 DNS Pod 和服务，并配置 kubelet 以告知各个容器使用 DNS 服务的 IP 来解析 DNS 名称。

  - A/AAAA 记录

    “普通” 服务会以 `my-svc.my-namespace.svc.cluster-domain.example` 这种名字的形式被分配一个 DNS A 或 AAAA 记录，取决于服务的 IP 协议族。 该名称会解析成对应服务的集群 IP。

  - Pods A/AAAA 记录

    经由 Deployment 或者 DaemonSet 所创建的所有 Pods 都会有`pod-ip-address.deployment-name.my-namespace.svc.cluster-domain.example.` DNS 解析项与之对应

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: busybox
  spec:
    selector:
      app: busybox
    ports:
      - port: 80
  
  ---
  apiVersion: v1
  kind: Pod
  metadata:
    name: busybox
    labels:
      app: busybox
  spec:
    # Pod 规约中包含一个可选的 hostname 字段，可以用来指定 Pod 的主机名。 当这个字段被设置时，它将优先于 Pod 的名字成为该 Pod 的主机名
    hostname: busybox-1
    # 还有一个可选的 subdomain 字段，可以用来指定 Pod 的子域名
    subdomain: default-subdomain
    containers:
      - name: busybox
        image: busybox
        command: ["sh", "-c", "sleep 1h"]
        ports:
          - containerPort: 80
  
  ```

  Pod 将看到自己的域名为：`busybox-1.default-subdomain.default.svc.cluster.local`。DNS 会为此名字提供一个 A 记录或 AAAA 记录，指向该 Pod 的 IP。

## 从集群外部访问Service

Service 暴露方式：`ExternalName`、`ClusterIP`、`NodePort`、 and `LoadBalancer`，默认为`ClusterIP`

- ClusterIP

  仅仅使用一个集群内部（指pod集群）的IP地址，选择这个值意味着只能在这个服务在集群内部才可以被访问到

- NodePort

  在集群内部IP的基础上，在每一个节点的端口上开放这个服务，可以在任意`NodePort`地址上访问到这个服务。

- LoadBalancer

  在使用一个集群内部IP地址和在NodePort上开放一个Service的基础上，还可以向云提供者申请一个负载均衡器，将流量转发到已经以NodePort形式开发的Service上。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
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
  selector:
    matchLabels:
      app: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  # service暴露方式选择NodePort
  type: NodePort
  ports:
      # service的端口
    - port: 80
      # 目标Pod端口
      targetPort: 80
      # 节点上对外的端口，nodePort需要大于30000
      nodePort: 30001
```

```shell
# 当前节点30001端口映射到service的80端口
$ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.1.0.1      <none>        443/TCP        37h
nginx        NodePort    10.1.245.61   <none>        80:30001/TCP   3m47s

# 访问节点地址对应端口，可以请求到数据
$ curl 192.168.56.102:30001
```

## Ingress 控制器

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/ingress.png" style="zoom:45%;" />

Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。

Ingress-nginx 本质是网关，当请求 abc.com/service/a, Ingress 会进行响应转发到对应service，底层运行了一个 nginx。但是Kubernetes不直接使用 nginx 原因是：Kubernetes 也需要把转发的路由规则纳入它的配置管理，变成 ingress 对象，所有才有 ingress 这个资源对象。

Ingress 公开了从集群外部到集群内服务的 HTTP 和 HTTPS 路由，流量路由由 Ingress 资源上定义的规则控制。

### 部署 Ingress 控制器

`ingress-nginx`实际就是运行在kubernetes上的一组Pod，通过Pod来接管外部到worker node上的请求

ingress-nginx-controller.yaml：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  clusterIP: 10.1.211.240
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 31686
    port: 80
    protocol: TCP
    targetPort: http
  - name: https
    nodePort: 30036
    port: 443
    protocol: TCP
    targetPort: https
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  sessionAffinity: None
  type: NodePort
---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: registry.aliyuncs.com/google_containers/nginx-ingress-controller:0.26.1
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown

---

```

```shell
# TODO 按照
$ kubectl apply -f ingress-nginx-controller.yaml

# 查看部署状态
$ kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --watch

# 检查部署版本
$ POD_NAMESPACE=ingress-nginx
$ POD_NAME=$(kubectl get pods -n $POD_NAMESPACE -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}')
$ kubectl exec -it $POD_NAME -n $POD_NAMESPACE -- /nginx-ingress-controller --version

$ kubectl get all -n ingress-nginx
```

### 创建 Ingress 资源

每个 HTTP 规则都包含以下信息：

1. 可选的 host：若未指定 host，该规则适用于通过指定 IP 地址的所有入站 HTTP 通 信。 如果提供了 host(例如 foo.bar.com)，则 rules 适用于该 host。

2. 路径列表 paths（例如，/testpath）：每个路径都有一个由 serviceName 和 servicePort 定义的关联后端。 在负载均衡器将流量定向到引用的服务之前，主机和路径都必须匹配传入请求的内容。
3. backend(后端)：是 Service 文档中所述的服务和端口名称的组合。 与规则的 host 和 path 匹配的对 Ingress 的 HTTP(和 HTTPS )请求将发送到列出的backend。

```shell
# 创建一个 Deployment
$ kubectl create deployment web --image=registry.cn-beijing.aliyuncs.com/qingfeng666/hello-app:1.0
deployment.apps/web created

# 将 Deployment 暴露出来
$ kubectl expose deployment web --type=NodePort --port=8080

# 验证 Service 已经创建
$ kubectl get service web

# 使用节点端口信息访问服务
$ curl node2:32739
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: hello-world.info
      http:
        paths:
          # 所有访问 hello-world.info 域名根目录的请求都会转发到 web 这个service
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080
```

```shell
# 创建 Ingress 资源
$ kubectl apply -f example-ingress.yaml

# 查看ingress资源
$ kubectl get ing
NAME              CLASS    HOSTS              ADDRESS        PORTS   AGE
example-ingress   <none>   hello-world.info   10.1.211.240   80      99s

# 在 /etc/hosts 文件添加
10.1.211.240 hello-world.info

$ curl hello-world.info
```

### 在集群内部暴露服务 

上面的实例实现在在宿主机上暴露的 Ingress 服务，但在宿主机外部并无法访问。想要在集群级别暴露服务，在 ingress 控制器的Deployment 定义中加入：`hostNetwork: true`，即可让Ingress 服务对外暴露80端口：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  ...
spec:
  replicas: 1
  selector:
    ...
    spec:
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      hostNetwork: true
      containers:
```

重新部署Ingress控制器，从其他节点`curl node1  `可看到结果返回

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

  有时候创建的服务不想走负载均衡，想直接通过pod-ip链接后端，可以使用`headless service`接可以解决。`headless service` 是将service的发布文件中的
  `clusterip=none` ，不让其获取`clusterip` ， **DNS解析的时候直接走pod**

- 当删除 StatefulSets 时，StatefulSet 不提供任何终止 Pod 的保证。

- 为了实现 StatefulSet 中的 Pod可以有序和优雅的终止，可以在删除之前将 StatefulSet 缩放为 0  

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
$ kubectl get po -o wide
NAME    READY   STATUS    RESTARTS   AGE     IP            NODE    NOMINATED NODE   READINESS GATES
web-0   1/1     Running   0          3m43s   10.244.1.56   node1   <none>           <none>
web-1   1/1     Running   0          2m45s   10.244.1.57   node1   <none>           <none>
web-2   1/1     Running   0          2m41s   10.244.1.58   node1   <none>           <none>

# 每个副本都使用独立的存储data1-3目录， 并挂载在/usr/share/nginx/html路径
# 存储独立性验证
$ kubectl exec -it web-0 sh
$ cd /usr/share/nginx/html

# 在/tmp/data1 目录下创建一个文件1.txt，再回到 web-0 pod的/usr/share/nginx/html目录，看到目录下自动多了 1.txt 文件
# 登录 web-1 pod，进入相同目录，会发现该目录并没有 1.txt 文件，此时就证明了 statefulset 的 pod存储是互相独立的

# 访问该 nginx 服务
$ curl 10.244.1.67
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
fluentd-elasticsearch-7vncc      1/1     Running   0          2m2s   10.244.0.14      master   <none>           <none>
fluentd-elasticsearch-r8fc8      1/1     Running   0          2m2s   10.244.1.55      node1    <none>           <none>
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

# 持久化





# 监控





