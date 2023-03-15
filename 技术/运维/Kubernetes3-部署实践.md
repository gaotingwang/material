# 部署博客项目

## StatefulSet 部署 Mysql

StatefulSet 为它们的每个 Pod 维护了一个固定的 ID，无论怎么调度，每个 Pod 都有一个永久不变的 ID

StatefulSets 使用场景：

- 稳定的、唯一的网络标识符。
- 稳定的、持久的存储。
- 有序的、优雅的部署和缩放。
- 有序的、自动的滚动更新。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql57
  labels:
    app: mysql
spec:
  ports:
    - port: 3306
      name: mysql
      nodePort: 30306
  type: NodePort
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql # has to match .spec.template.metadata.labels
  serviceName: "mysql"
  replicas: 1 # by default is 1
  template:
    metadata:
      labels:
        app: mysql # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mysql
          image: registry.cn-beijing.aliyuncs.com/qingfeng666/mysql:5.7
          ports:
            - containerPort: 3306
              name: mysql
          volumeMounts:
            - name: host-path
              mountPath: /var/lib/mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "password"
      volumes:
        - name: host-path
          hostPath:
            path: /tmp/mysql
            type: DirectoryOrCreate
```

```shell
# 宿主机连接mysql
$ mysql -h192.168.56.102 -P30306 -uroot -p

# 创建database
MySQL > create database blogDB DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
```



## 编写博客应用的Service，Deployment 文件

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeblog
spec:
  selector:
    app: kubeblog
  type: NodePort
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30002
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeblog
spec:
  selector:
    matchLabels:
      app: kubeblog
  template:
    metadata:
      labels:
        app: kubeblog
    spec:
      containers:
        - name: kubeblog
          image: registry.cn-beijing.aliyuncs.com/qingfeng666/kubeblog:1.3
          ports:
            - containerPort: 5000
          env:
            - name: MYSQL_PORT
              value: "30306"
            - name: MYSQL_SERVER
              value: "192.168.56.102"
            - name: MYSQL_DB_NAME
              value: "blogDB"
            - name: MYSQL_USER_TEST
              value: "root"
            - name: MYSQL_PASSWORD_TEST
              value: "password"
```

部署成功后，浏览器可以正常访问 [http://192.168.56.102:30002/](http://192.168.56.102:30002/) 

## 进行配置分离

### 创建 mysql 秘钥 secret

```shell
$ kubectl create secret generic mysql-password-test --from-literal=MYSQL_PASSWORD_TEST=password
```

### 在 Deployment 中使用该秘钥

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeblog
spec:
  selector:
    app: kubeblog
  type: NodePort
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30002
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeblog
spec:
  selector:
    matchLabels:
      app: kubeblog
  template:
    metadata:
      labels:
        app: kubeblog
    spec:
      containers:
        - name: kubeblog
          image: registry.cn-beijing.aliyuncs.com/qingfeng666/kubeblog:1.3
          ports:
            - containerPort: 5000
          env:
            - name: MYSQL_PORT
              value: "30306"
            - name: MYSQL_SERVER
              value: "192.168.56.102"
            - name: MYSQL_DB_NAME
              value: "blogDB"
            - name: MYSQL_USER_TEST
              value: "root"
            - name: MYSQL_PASSWORD_TEST
              # 从secret中获取
              valueFrom:
                secretKeyRef:
                  key: MYSQL_PASSWORD_TEST
                  name: mysql-password-test
```

## 空间隔离

通过namespace可进行环境隔离，避免资源相互干扰

```shell
# 查看kubernetes的命名空间
$ kubectl get ns
NAME                   STATUS   AGE
default                Active   6d11h
ingress-nginx          Active   4d11h
kube-node-lease        Active   6d11h
kube-public            Active   6d11h
kube-system            Active   6d11h
kubernetes-dashboard   Active   6d6h

# 创建 Test和 Prod namespace
$ kubectl create namespace test
$ kubectl create namespace prod

# 创建secret通过 -n 参数来指定 namespace
$ kubectl create secret generic mysql-password-test --from-literal=MYSQL_PASSWORD_TEST=password -n test

# 修改之前的yaml, 在metadata中加入namespace来指定命名空间
```

# 使用Helm部署应用

## 安装helm

```shell
# master安装helm
$ wget https://get.helm.sh/helm-v3.4.1-linux-amd64.tar.gz -O helm-v3.4.1-linux-amd64.tar.gz
$ tar -zxvf helm-v3.4.1-linux-amd64.tar.gz
$ mv linux-amd6/helm /usr/local/bin/helm

$ helm ls
NAME	NAMESPACE	REVISION	UPDATED	STATUS	CHART	APP VERSION
```

## 部署一个 Helm Chart

```shell
# 会在当前目录下创建nginx目录，该目录下回预置k8s相关模板
$ helm create nginx
# Chart.yaml 定义了helm chart的名称和版本，template 目录防止模板，valuse.yaml 定义变量值
$ cd nginx && ls
charts  Chart.yaml  templates  values.yaml

# 在 nginx 文件同级目录打包
$ helm package nginx && ls
Successfully packaged chart and saved it to: /opt/applications/helm/nginx-0.1.0.tgz
nginx  nginx-0.1.0.tgz

# 部署Chart
$ helm install nginx nginx-0.1.0.tgz
$ helm ls
NAME 	NAMESPACE	REVISION	UPDATED                               	STATUS  	CHART      	APP VERSION
nginx	default  	1       	2023-03-15 11:23:01.07909629 +0800 CST	deployed	nginx-0.1.0	1.16.0
$ kubectl get po
NAME                                      READY   STATUS      RESTARTS   AGE
nginx-764c6bcc78-6vwlb                    1/1     Running     0          107s

# 删除部署
$ helm delete nginx
```

## 创建博客应用的 Helm chart

### 初始化

```shell
$ helm create kubeblog

$ cd kubeblog
# charts 目录下为第三方依赖的组件，这里目前用不到为空

```

### 编辑模板

编辑模板放入templates目录下：

- kubeblog-deployment.yaml

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: {{ .Release.Name }}
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: {{ .Release.Name }}
    template:
      metadata:
        labels:
          app: {{ .Release.Name }}
      spec:
        containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
          - containerPort: {{ .Values.service.port }}
          env:
            - name: MYSQL_PORT
              value: "30306"
            - name: MYSQL_SERVER
              value: "192.168.56.102"
            - name: MYSQL_DB_NAME
              value: "{{ .Values.mysql.dbName }}"
            - name: MYSQL_USER_TEST
              value: "root"
            - name: MYSQL_PASSWORD_TEST
              valueFrom:
                  secretKeyRef:
                    name: mysql-password-test
                    key: MYSQL_PASSWORD_TEST
        imagePullSecrets:
        - name: regcred-local
  ```

- mysql.yaml

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: mysql57
    labels:
      app: mysql
  spec:
    ports:
    - port: 3306
      name: mysql
      nodePort: {{ .Values.mysql.port }}
    type: NodePort
    selector:
      app: mysql
  ---
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: mysql
  spec:
    selector:
      matchLabels:
        app: mysql # has to match .spec.template.metadata.labels
    serviceName: "mysql"
    replicas: 1 # by default is 1
    template:
      metadata:
        labels:
          app: mysql # has to match .spec.selector.matchLabels
      spec:
        terminationGracePeriodSeconds: 10
        containers:
        - name: mysql
          image: registry.cn-beijing.aliyuncs.com/qingfeng666/mysql:5.7
          ports:
          - containerPort: 3306
            name: mysql
          volumeMounts:
          - name: host-path
            mountPath: /var/lib/mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "password"
        volumes:
          - name: host-path
            hostPath:
                path: /tmp/mysql
                type: DirectoryOrCreate
  ```

- service.yaml

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: {{ .Release.Name }}
    labels:
      app: {{ .Release.Name }}
  spec:
    ports:
      - name: port2
        protocol: TCP
        port: {{ .Values.service.port }}
        nodePort: {{ .Values.service.nodePort }}
    selector:
      app: {{ .Release.Name }}
    type: NodePort
  ```

将变量抽取出来放入values.yaml文件中：

```yaml
# Default values for discovery.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: registry.cn-beijing.aliyuncs.com/qingfeng666/kubeblog
  tag: 1.3
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 5000
  nodePort: 30002

mysql:
  port: 30306
  dbName: blogDB


resources: {}
  # We usually recommend not to specify default resources and to leave this as a conscious
  # choice for the user. This also increases chances charts run on environments with little
  # resources, such as Minikube. If you do want to specify resources, uncomment the following
  # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
  # limits:
  #  cpu: 100m
  #  memory: 128Mi
  # requests:
  #  cpu: 100m
#  memory: 128Mi

nodeSelector: {}

tolerations: []

affinity: {}

```

```
.
├── kubeblog
│   ├── charts
│   ├── Chart.yaml
│   ├── templates
│   │   ├── _helpers.tpl
│   │   ├── kubeblog-deployment.yaml
│   │   ├── mysql.yaml
│   │   ├── NOTES.txt
│   │   └── service.yaml
│   └── values.yaml
```

### 部署

```shell
$ helm package kubeblog
$ helm install kubeblog kubeblog-0.1.0.tgz
```

## 升级回滚

将kubeblog副本数改为 2 `spec: replicas: 2`， 修改 kubeblog 目录下 Chart.yaml 文件版本为 0.1.1 ，执行`helm package kubeblog`，生成 kubeblog-0.1.1.tgz。

```shell
# 部署新版本 Chart
$ helm upgrade kubeblog helm/kubeblog --version 0.1.1

# 回滚版本到0.1.0
$ helm rollback kubeblog 1
```

## 多环境部署

```shell
# 通过 -f 来指定不同变量文件，不同环境可以使用不同的变量文件
# -n 来指定命名空间
$ helm install kubeblog kubeblog-0.1.1.tgz -f kubeblog/values-test.yaml -n test
```

# 最佳实践总结

## 普通配置

- 定义配置时，请指定最新的稳定 API 版本。 
- 在推送到集群之前，配置文件应存储在版本控制中。 这样在必要时快速回滚配置更改，它还有助于集群重新创建和恢复。 
- 使用 YAML 而不是 JSON 编写配置文件。虽然这些格式几乎可以在所有场景中互换使用，但 YAML 往往更加用户友好。 
- 只要有意义，就将相关对象分组到一个文件中。 一个文件通常比几个文件更容易管理

## Deployment

- 如果可能，不要使用独立的 Pods（即，未绑定到 ReplicaSet 或 Deployment 的 Pod）。 如果节点发生故障，将不会重新调度独立的 Pods。
- Deployment 会创建一个 ReplicaSet 以确保所需数量的 Pod 始终可用，并指定替换 Pod 的策略（例如 RollingUpdate)， 除了一些显式的restartPolicy: Never 场景之外，几乎总是优先考虑直接创建 Pod。 Job 也可能是合适的。  

## 服务

在创建相应的后端工作负载（Deployment 或 ReplicaSet），以及在需要访问它的任何工作负载之前创建服务， 否则将环境变量不会生效，DNS 没有此限制。

- 一个可选（尽管强烈推荐）的集群插件是 DNS 服务器。DNS 服务器为新的 Services 监视Kubernetes API，并为每个创建一组 DNS 记录。 如果在整个集群中启用了 DNS，则所有 Pods 应该能够自动对 Services 进行名称解析。  
- 当不需要 kube-proxy 负载均衡时，使用无头服务(ClusterIP 被设置为 None)以便于服务发现

## 使用标签  

- 定义并使用标签来识别应用程序 或 Deployment 的 语义属性，例如{ app: myapp, tier: frontend, phase: test, deployment: v3 }。 可以使用这些标签为其他资源选择合适的 Pod； 例如，一个选择所有 tier: frontend Pod 的服务，或者 app: myapp 的所有 phase: test 组件。
- 通过从选择器中省略特定发行版的标签，可以使服务跨越多个 Deployment。 Deployment 可以在不停机的情况下轻松更新正在运行的服务。
- Deployment 描述了对象的期望状态，并且如果对该规范的更改被成功应用， 则 Deployment 控制器以受控速率将实际状态改变为期望状态。  

## 容器镜像

- 在生产中部署容器时应避免使用 `:latest` 标记，因为这样更难跟踪正在运行的镜像版本，并且更难以正确回滚  
- 底层镜像驱动程序的缓存语义能够使即便 imagePullPolicy: Always 的配置也很高效。 例如，对于 Docker，如果镜像已经存在，则拉取尝试很快，因为镜像层都被缓存并且不需要下载  

## 使用 kubectl

- 使用 `kubectl apply -f` 。 它在中的所有 .yaml、.yml 和 .json 文件中查找 Kubernetes 配置，并将其传递给 apply。
- 使用标签选择器进行 get 和 delete 操作，而不是特定的对象名称  



