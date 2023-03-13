编写k8s相关`yaml`文件，可以给编辑器安装`kubernetes`插件，提供了在线动态模板，该模板也可以进行自行修改，输入k接口调用模板，常见配置类型的预定义模板：

```
kcm ：ConfigMap
kdep：Deployment
kpod：Pod
kres：Generic resource
kser：Service
```

# 调度单元Pod

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

