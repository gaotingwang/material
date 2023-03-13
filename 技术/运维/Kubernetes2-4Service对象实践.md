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

