Prometheus 相比于其他传统监控工具主要有以下几个特点：

- 具有由 metric 名称和键/值对标识的时间序列数据的多维数据模型
- 有一个灵活的查询语言
- 不依赖分布式存储，只和本地磁盘有关
- 通过 HTTP 的服务拉取时间序列数据
- 也支持推送的方式来添加时间序列数据
- 还支持通过服务发现或静态配置发现目标
- 多种图形和仪表板支持  

## 安装部署

### 1. 准备

在每个节点上(Master, node 都需要)同步系统时间：

```shell
$ yum -y install ntp
$ systemctl enable ntpd
$ ntpdate time1.aliyun.com
```

### 2. 创建NameSpace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

```shell
$ kubectl apply -f monitoring-ns.yml
```

所有Prometheus相关的对象都部署在NameSpace“monitoring”之下  

### 3. 部署node-exporter 

node-exporter用于提供Unix内核的硬件以及系统指标，采集服务器层面的运行指标，包括机器的loadavg、filesystem、meminfo等  

- 部署node-exporter daemonset

  ```yaml
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
    name: prometheus-node-exporter
    namespace: monitoring
    labels:
      app: prometheus
      component: node-exporter
  spec:
    selector:
      matchLabels:
        app: prometheus
        component: node-exporter
    template:
      metadata:
        name: prometheus-node-exporter
        labels:
          app: prometheus
          component: node-exporter
      spec:
        containers:
        # 原始docker image是prom/node-exporter:v0.14.0
        - image: prom/node-exporter:v0.14.0
          name: prometheus-node-exporter
          ports:
          - name: prom-node-exp
            #^ must be an IANA_SVC_NAME (at most 15 characters, ..)
            containerPort: 9100
            hostPort: 9100
        hostNetwork: true
        hostPID: true
  ```

  ```shell
  $ kubectl apply -f node-exporter-daemonset.yml
  ```

- 部署node-exporter service  

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    annotations:
      prometheus.io/scrape: 'true'
    name: prometheus-node-exporter
    namespace: monitoring
    labels:
      app: prometheus
      component: node-exporter
  spec:
    # 使用Cluster_IP，不能直接访问
    type: ClusterIP
    clusterIP: None
    ports:
      # 使用宿主机网络，开放宿主机端口9100，可以直接通过<node_ip>:9100访问
      - name: prometheus-node-exporter
        port: 9100
        protocol: TCP
    selector:
      app: prometheus
      component: node-exporter
  ```
  
  ```shell
  $ kubectl apply -f node-exporter-service.yml
  ```

### 4. 部署kube-state-metrics

kube-state-metrics关注于获取k8s各种资源的最新状态，如deployment、daemonset，将k8s的运行状态在内存中做个快照，并获取新的指标

- 创建对应的ServiceAccount  

  ```yaml
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: kube-state-metrics
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: kube-state-metrics
  subjects:
  - kind: ServiceAccount
    name: kube-state-metrics
    namespace: monitoring
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: kube-state-metrics
  rules:
  - apiGroups: [""]
    resources:
    - nodes
    - pods
    - services
    - resourcequotas
    - replicationcontrollers
    - limitranges
    verbs: ["list", "watch"]
  - apiGroups: ["apps"]
    resources:
    - daemonsets
    - deployments
    - replicasets
    verbs: ["list", "watch"]
  ---
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: kube-state-metrics
    namespace: monitoring
  ```

  ```shell
  $ kubectl apply -f kube-state-metrics-ServiceAccount.yml
  ```

  

- 部署kube-state-metrics deployment  

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: kube-state-metrics
    namespace: monitoring
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: kube-state-metrics
    template:
      metadata:
        labels:
          app: kube-state-metrics
      spec:
        serviceAccountName: kube-state-metrics
        containers:
        - name: kube-state-metrics
  #       image: gcr.io/google_containers/kube-state-metrics:v0.5.0
          image: registry.cn-beijing.aliyuncs.com/qua-io-coreos/kube-state-metrics:v1.3.0
          ports:
          - containerPort: 8080
  
  ```

    ```shell
    $ kubectl apply -f kube-state-metrics-deploy.yml
    ```

- 部署kube-state-metrics service  

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    annotations:
      prometheus.io/scrape: 'true'
    name: kube-state-metrics
    namespace: monitoring
    labels:
      app: kube-state-metrics
  spec:
    ports:
    - name: kube-state-metrics
      port: 8080
      protocol: TCP
    selector:
      app: kube-state-metrics
  ```

    ```shell
    $ kubectl apply -f kube-state-metrics-service.yml
    ```

  

### 5. 部署node disk monitor

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-directory-size-metrics
  namespace: monitoring
  annotations:
    description: |
      This `DaemonSet` provides metrics in Prometheus format about disk usage on the nodes.
      The container `read-du` reads in sizes of all directories below /mnt and writes that to `/tmp/metrics`. It only reports directories larger then `100M` for now.
      The other container `caddy` just hands out the contents of that file on request via `http` on `/metrics` at port `9102` which are the defaults for Prometheus.
      These are scheduled on every node in the Kubernetes cluster.
      To choose directories from the node to check, just mount them on the `read-du` container below `/mnt`.
spec:
  selector:
    matchLabels:
      app: node-directory-size-metrics
  template:
    metadata:
      labels:
        app: node-directory-size-metrics
      annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/port: '9102'
        description: |
          This `Pod` provides metrics in Prometheus format about disk usage on the node.
          The container `read-du` reads in sizes of all directories below /mnt and writes that to `/tmp/metrics`. It only reports directories larger then `100M` for now.
          The other container `caddy` just hands out the contents of that file on request on `/metrics` at port `9102` which are the defaults for Prometheus.
          This `Pod` is scheduled on every node in the Kubernetes cluster.
          To choose directories from the node to check just mount them on `read-du` below `/mnt`.
    spec:
      containers:
      - name: read-du
        image: giantswarm/tiny-tools:latest
        imagePullPolicy: IfNotPresent
        command:
        - fish
        - --command
        - |
          touch /tmp/metrics-temp
          while true
            for directory in (du --bytes --separate-dirs --threshold=100M /mnt)
              echo $directory | read size path
              echo "node_directory_size_bytes{path=\"$path\"} $size" \
                >> /tmp/metrics-temp
            end
            mv /tmp/metrics-temp /tmp/metrics
            sleep 300
          end
        volumeMounts:
        - name: host-fs-var
          mountPath: /mnt/var
          readOnly: true
        - name: metrics
          mountPath: /tmp
      - name: caddy
        image: dockermuenster/caddy:0.9.3
        command:
        - "caddy"
        - "-port=9102"
        - "-root=/var/www"
        ports:
        - containerPort: 9102
        volumeMounts:
        - name: metrics
          mountPath: /var/www
      volumes:
      - name: host-fs-var
        hostPath:
          path: /var
      - name: metrics
        emptyDir:
          medium: Memory
```

```shell
$ kubectl apply -f monitor-node-disk-daemonset.yml
```

### 6. 部署Prometheus

- 创建对应的ServiceAccount

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    name: prometheus
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: prometheus
  subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: monitoring
  ---
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: prometheus
  rules:
  - apiGroups: [""]
    resources:
    - nodes
    - nodes/proxy
    - services
    - endpoints
    - pods
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources:
    - configmaps
    verbs: ["get"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
  ---
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: prometheus-k8s
    namespace: monitoring
  ```

  ```shell
  $ kubectl apply -f prometheus-k8s-ServiceAccount.yml
  ```

- 创建配置相关的configmap

  可以到 https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config 查看相关设置的定义

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    creationTimestamp: null
    name: prometheus-core
    namespace: monitoring
  data:
    prometheus.yaml: |
      global:
        scrape_interval: 10s
        scrape_timeout: 10s
        evaluation_interval: 10s
      rule_files:
        - "/etc/prometheus-rules/*.rules"
      scrape_configs:
  
        # https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L37
        - job_name: 'kubernetes-nodes'
          scheme: https
          tls_config:
            ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
          kubernetes_sd_configs:
            - role: node
          relabel_configs:
            #- source_labels: [__address__]
            #  regex: '(.*):10250'
            #  replacement: '${1}:10255'
            #  target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics
        # https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L79
        - job_name: 'kubernetes-endpoints'
          kubernetes_sd_configs:
            - role: endpoints
          relabel_configs:
            - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
              action: keep
              regex: true
            - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
              action: replace
              target_label: __scheme__
              regex: (https?)
            - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
              action: replace
              target_label: __metrics_path__
              regex: (.+)
            - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
              action: replace
              target_label: __address__
              regex: (.+)(?::\d+);(\d+)
              replacement: $1:$2
            - action: labelmap
              regex: __meta_kubernetes_service_label_(.+)
            - source_labels: [__meta_kubernetes_namespace]
              action: replace
              target_label: kubernetes_namespace
            - source_labels: [__meta_kubernetes_service_name]
              action: replace
              target_label: kubernetes_name
  
        # https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L119
        - job_name: 'kubernetes-services'
          metrics_path: /probe
          params:
            module: [http_2xx]
          kubernetes_sd_configs:
            - role: service
          relabel_configs:
            - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
              action: keep
              regex: true
            - source_labels: [__address__]
              target_label: __param_target
            - target_label: __address__
              replacement: blackbox
            - source_labels: [__param_target]
              target_label: instance
            - action: labelmap
              regex: __meta_kubernetes_service_label_(.+)
            - source_labels: [__meta_kubernetes_namespace]
              target_label: kubernetes_namespace
            - source_labels: [__meta_kubernetes_service_name]
              target_label: kubernetes_name
  
        # https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml#L156
        - job_name: 'kubernetes-pods'
          kubernetes_sd_configs:
            - role: pod
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
              action: keep
              regex: true
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
              action: replace
              target_label: __metrics_path__
              regex: (.+)
            - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
              action: replace
              regex: (.+):(?:\d+);(\d+)
              replacement: ${1}:${2}
              target_label: __address__
            - action: labelmap
              regex: __meta_kubernetes_pod_label_(.+)
            - source_labels: [__meta_kubernetes_namespace]
              action: replace
              target_label: kubernetes_namespace
            - source_labels: [__meta_kubernetes_pod_name]
              action: replace
              target_label: kubernetes_pod_name
            - source_labels: [__meta_kubernetes_pod_container_port_number]
              action: keep
              regex: 9\d{3}
  
        - job_name: 'kubernetes-cadvisor'
          scheme: https
          tls_config:
            ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
          kubernetes_sd_configs:
            - role: node
          relabel_configs:
            - action: labelmap
            - action: labelmap
              regex: __meta_kubernetes_node_label_(.+)
            - target_label: __address__
              replacement: kubernetes.default.svc:443
            - source_labels: [__meta_kubernetes_node_name]
              regex: (.+)
              target_label: __metrics_path__
              replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
  ```

    ```shell
    $ kubectl apply -f prometheus-config-configmap.yml
    ```

- 创建警告规则相关的configmap  

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    creationTimestamp: null
    name: prometheus-rules
    namespace: monitoring
  data:
    cpu-usage.rules: |
      ALERT NodeCPUUsage
        IF (100 - (avg by (instance) (irate(node_cpu{name="node-exporter",mode="idle"}[5m])) * 100)) > 75
        FOR 2m
        LABELS {
          severity="page"
        }
        ANNOTATIONS {
          SUMMARY = "{{$labels.instance}}: High CPU usage detected",
          DESCRIPTION = "{{$labels.instance}}: CPU usage is above 75% (current value is: {{ $value }})"
        }
    instance-availability.rules: |
      ALERT InstanceDown
        IF up == 0
        FOR 1m
        LABELS { severity = "page" }
        ANNOTATIONS {
          summary = "Instance {{ $labels.instance }} down",
          description = "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute.",
        }
    low-disk-space.rules: |
      ALERT NodeLowRootDisk
        IF ((node_filesystem_size{mountpoint="/root-disk"} - node_filesystem_free{mountpoint="/root-disk"} ) / node_filesystem_size{mountpoint="/root-disk"} * 100) > 75
        FOR 2m
        LABELS {
          severity="page"
        }
        ANNOTATIONS {
          SUMMARY = "{{$labels.instance}}: Low root disk space",
          DESCRIPTION = "{{$labels.instance}}: Root disk usage is above 75% (current value is: {{ $value }})"
        }
  
      ALERT NodeLowDataDisk
        IF ((node_filesystem_size{mountpoint="/data-disk"} - node_filesystem_free{mountpoint="/data-disk"} ) / node_filesystem_size{mountpoint="/data-disk"} * 100) > 75
        FOR 2m
        LABELS {
          severity="page"
        }
        ANNOTATIONS {
          SUMMARY = "{{$labels.instance}}: Low data disk space",
          DESCRIPTION = "{{$labels.instance}}: Data disk usage is above 75% (current value is: {{ $value }})"
        }
    mem-usage.rules: |
      ALERT NodeSwapUsage
        IF (((node_memory_SwapTotal-node_memory_SwapFree)/node_memory_SwapTotal)*100) > 75
        FOR 2m
        LABELS {
          severity="page"
        }
        ANNOTATIONS {
          SUMMARY = "{{$labels.instance}}: Swap usage detected",
          DESCRIPTION = "{{$labels.instance}}: Swap usage usage is above 75% (current value is: {{ $value }})"
        }
  
      ALERT NodeMemoryUsage
        IF (((node_memory_MemTotal-node_memory_MemFree-node_memory_Cached)/(node_memory_MemTotal)*100)) > 75
        FOR 2m
        LABELS {
          severity="page"
        }
        ANNOTATIONS {
          SUMMARY = "{{$labels.instance}}: High memory usage detected",
          DESCRIPTION = "{{$labels.instance}}: Memory usage is above 75% (current value is: {{ $value }})"
        }  
  ```

    ```shell
    $ kubectl apply -f prometheus-rules-configmap.yml
    ```

- 创建Prometheus的缺省用户及密码  

  ```yaml
  apiVersion: v1
  kind: Secret
  data:
    admin-password: YWRtaW4=
    admin-username: YWRtaW4=
  metadata:
    name: grafana
    namespace: monitoring
  type: Opaque
  ```

    ```shell
    $ kubectl apply -f prometheus-secret.yml
    ```

  缺省用户/密码为 admin/admin： `echo "YWRtaW4=" | base64 -D`  

- 部署Prometheus Server的Deployment  

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: prometheus-core
    namespace: monitoring
    labels:
      app: prometheus
      component: core
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: prometheus
        component: core
    template:
      metadata:
        name: prometheus-main
        labels:
          app: prometheus
          component: core
      spec:
        serviceAccountName: prometheus-k8s
        containers:
        - name: prometheus
          image: prom/prometheus:v1.7.0
          args:
            - '-storage.local.retention=12h'
            - '-storage.local.memory-chunks=500000'
            - '-config.file=/etc/prometheus/prometheus.yaml'
            - '-alertmanager.url=http://alertmanager:9093/'
          ports:
          - name: webui
            containerPort: 9090
          resources:
            requests:
              cpu: 500m
              memory: 500M
            limits:
              cpu: 500m
              memory: 500M
          volumeMounts:
          - name: config-volume
            mountPath: /etc/prometheus
          - name: rules-volume
            mountPath: /etc/prometheus-rules
        volumes:
        - name: config-volume
          configMap:
            name: prometheus-core
        - name: rules-volume
          configMap:
            name: prometheus-rules
  ```
  
    ```shell
    $ kubectl apply -f prometheus-deploy.yml
    ```
  
- 部署Prometheus Server的Service  

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: prometheus
    namespace: monitoring
    labels:
      app: prometheus
      component: core
    annotations:
      prometheus.io/scrape: 'true'
  spec:
    # 类型是NodePort，由K8s自由分配，通过 kubectl get service -n monitoring 可以查到分配的端
    type: NodePort
    ports:
      - port: 9090
        protocol: TCP
        name: webui
        nodePort: 30091
    selector:
      app: prometheus
      component: core
  ```
  
    ```shell
    $ kubectl apply -f prometheus-service.yml  
    ```

### 7. 部署 Grafana

用于图表显示

1. 创建Grafana Dashboard模版相关的configmap

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     creationTimestamp: null
     name: grafana-import-dashboards
     namespace: monitoring
   data:
     grafana-net-2-dashboard.json: |
       {
         "__inputs": [{
           "name": "DS_PROMETHEUS",
           "label": "Prometheus",
           "description": "",
           "type": "datasource",
           "pluginId": "prometheus",
           "pluginName": "Prometheus"
         }],
         "__requires": [{
           "type": "panel",
           "id": "singlestat",
           "name": "Singlestat",
           "version": ""
         }, {
           "type": "panel",
           "id": "text",
           "name": "Text",
           "version": ""
         }, {
           "type": "panel",
           "id": "graph",
           "name": "Graph",
           "version": ""
         }, {
           "type": "grafana",
           "id": "grafana",
           "name": "Grafana",
           "version": "3.1.0"
         }, {
           "type": "datasource",
           "id": "prometheus",
           "name": "Prometheus",
           "version": "1.0.0"
         }],
         "id": null,
         "title": "Prometheus Stats",
         "tags": [],
         "style": "dark",
         "timezone": "browser",
         "editable": true,
         "hideControls": true,
         "sharedCrosshair": false,
         "rows": [{
           "collapse": false,
           "editable": true,
           "height": 178,
           "panels": [{
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": false,
             "colors": ["rgba(245, 54, 54, 0.9)", "rgba(237, 129, 40, 0.89)", "rgba(50, 172, 45, 0.97)"],
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 1,
             "editable": true,
             "error": false,
             "format": "s",
             "id": 5,
             "interval": null,
             "links": [],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "50%",
             "prefix": "",
             "prefixFontSize": "50%",
             "span": 3,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "(time() - process_start_time_seconds{job=\"prometheus\"})",
               "intervalFactor": 2,
               "refId": "A",
               "step": 4
             }],
             "thresholds": "",
             "title": "Uptime",
             "type": "singlestat",
             "valueFontSize": "80%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current",
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "rangeMaps": [{
               "from": "null",
               "to": "null",
               "text": "N/A"
             }],
             "mappingType": 1,
             "gauge": {
               "show": false,
               "minValue": 0,
               "maxValue": 100,
               "thresholdMarkers": true,
               "thresholdLabels": false
             }
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": false,
             "colors": ["rgba(50, 172, 45, 0.97)", "rgba(237, 129, 40, 0.89)", "rgba(245, 54, 54, 0.9)"],
             "datasource": "${DS_PROMETHEUS}",
             "editable": true,
             "error": false,
             "format": "none",
             "id": 6,
             "interval": null,
             "links": [],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "50%",
             "prefix": "",
             "prefixFontSize": "50%",
             "span": 3,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": true
             },
             "targets": [{
               "expr": "prometheus_local_storage_memory_series",
               "intervalFactor": 2,
               "refId": "A",
               "step": 4
             }],
             "thresholds": "1,5",
             "title": "Local Storage Memory Series",
             "type": "singlestat",
             "valueFontSize": "70%",
             "valueMaps": [],
             "valueName": "current",
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "rangeMaps": [{
               "from": "null",
               "to": "null",
               "text": "N/A"
             }],
             "mappingType": 1,
             "gauge": {
               "show": false,
               "minValue": 0,
               "maxValue": 100,
               "thresholdMarkers": true,
               "thresholdLabels": false
             }
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": true,
             "colors": ["rgba(50, 172, 45, 0.97)", "rgba(237, 129, 40, 0.89)", "rgba(245, 54, 54, 0.9)"],
             "datasource": "${DS_PROMETHEUS}",
             "editable": true,
             "error": false,
             "format": "none",
             "id": 7,
             "interval": null,
             "links": [],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "50%",
             "prefix": "",
             "prefixFontSize": "50%",
             "span": 3,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": true
             },
             "targets": [{
               "expr": "prometheus_local_storage_indexing_queue_length",
               "intervalFactor": 2,
               "refId": "A",
               "step": 4
             }],
             "thresholds": "500,4000",
             "title": "Internal Storage Queue Length",
             "type": "singlestat",
             "valueFontSize": "70%",
             "valueMaps": [{
               "op": "=",
               "text": "Empty",
               "value": "0"
             }],
             "valueName": "current",
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "rangeMaps": [{
               "from": "null",
               "to": "null",
               "text": "N/A"
             }],
             "mappingType": 1,
             "gauge": {
               "show": false,
               "minValue": 0,
               "maxValue": 100,
               "thresholdMarkers": true,
               "thresholdLabels": false
             }
           }, {
             "content": "<img src=\"http://prometheus.io/assets/prometheus_logo_grey.svg\" alt=\"Prometheus logo\" style=\"height: 40px;\">\n<span style=\"font-family: 'Open Sans', 'Helvetica Neue', Helvetica; font-size: 25px;vertical-align: text-top;color: #bbbfc2;margin-left: 10px;\">Prometheus</span>\n\n<p style=\"margin-top: 10px;\">You're using Prometheus, an open-source systems monitoring and alerting toolkit originally built at SoundCloud. For more information, check out the <a href=\"http://www.grafana.org/\">Grafana</a> and <a href=\"http://prometheus.io/\">Prometheus</a> projects.</p>",
             "editable": true,
             "error": false,
             "id": 9,
             "links": [],
             "mode": "html",
             "span": 3,
             "style": {},
             "title": "",
             "transparent": true,
             "type": "text"
           }],
           "title": "New row"
         }, {
           "collapse": false,
           "editable": true,
           "height": 227,
           "panels": [{
             "aliasColors": {
               "prometheus": "#C15C17",
               "{instance=\"localhost:9090\",job=\"prometheus\"}": "#C15C17"
             },
             "bars": false,
             "datasource": "${DS_PROMETHEUS}",
             "editable": true,
             "error": false,
             "fill": 1,
             "grid": {
               "threshold1": null,
               "threshold1Color": "rgba(216, 200, 27, 0.27)",
               "threshold2": null,
               "threshold2Color": "rgba(234, 112, 112, 0.22)"
             },
             "id": 3,
             "legend": {
               "avg": false,
               "current": false,
               "max": false,
               "min": false,
               "show": true,
               "total": false,
               "values": false
             },
             "lines": true,
             "linewidth": 2,
             "links": [],
             "nullPointMode": "connected",
             "percentage": false,
             "pointradius": 2,
             "points": false,
             "renderer": "flot",
             "seriesOverrides": [],
             "span": 9,
             "stack": false,
             "steppedLine": false,
             "targets": [{
               "expr": "rate(prometheus_local_storage_ingested_samples_total[5m])",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "{{job}}",
               "metric": "",
               "refId": "A",
               "step": 2
             }],
             "timeFrom": null,
             "timeShift": null,
             "title": "Samples ingested (rate-5m)",
             "tooltip": {
               "shared": true,
               "value_type": "cumulative",
               "ordering": "alphabetical",
               "msResolution": false
             },
             "type": "graph",
             "yaxes": [{
               "show": true,
               "min": null,
               "max": null,
               "logBase": 1,
               "format": "short"
             }, {
               "show": true,
               "min": null,
               "max": null,
               "logBase": 1,
               "format": "short"
             }],
             "xaxis": {
               "show": true
             }
           }, {
             "content": "#### Samples Ingested\nThis graph displays the count of samples ingested by the Prometheus server, as measured over the last 5 minutes, per time series in the range vector. When troubleshooting an issue on IRC or Github, this is often the first stat requested by the Prometheus team. ",
             "editable": true,
             "error": false,
             "id": 8,
             "links": [],
             "mode": "markdown",
             "span": 2.995914043583536,
             "style": {},
             "title": "",
             "transparent": true,
             "type": "text"
           }],
           "title": "New row"
         }, {
           "collapse": false,
           "editable": true,
           "height": "250px",
           "panels": [{
             "aliasColors": {
               "prometheus": "#F9BA8F",
               "{instance=\"localhost:9090\",interval=\"5s\",job=\"prometheus\"}": "#F9BA8F"
             },
             "bars": false,
             "datasource": "${DS_PROMETHEUS}",
             "editable": true,
             "error": false,
             "fill": 1,
             "grid": {
               "threshold1": null,
               "threshold1Color": "rgba(216, 200, 27, 0.27)",
               "threshold2": null,
               "threshold2Color": "rgba(234, 112, 112, 0.22)"
             },
             "id": 2,
             "legend": {
               "avg": false,
               "current": false,
               "max": false,
               "min": false,
               "show": true,
               "total": false,
               "values": false
             },
             "lines": true,
             "linewidth": 2,
             "links": [],
             "nullPointMode": "connected",
             "percentage": false,
             "pointradius": 5,
             "points": false,
             "renderer": "flot",
             "seriesOverrides": [],
             "span": 5,
             "stack": false,
             "steppedLine": false,
             "targets": [{
               "expr": "rate(prometheus_target_interval_length_seconds_count[5m])",
               "intervalFactor": 2,
               "legendFormat": "{{job}}",
               "refId": "A",
               "step": 2
             }],
             "timeFrom": null,
             "timeShift": null,
             "title": "Target Scrapes (last 5m)",
             "tooltip": {
               "shared": true,
               "value_type": "cumulative",
               "ordering": "alphabetical",
               "msResolution": false
             },
             "type": "graph",
             "yaxes": [{
               "show": true,
               "min": null,
               "max": null,
               "logBase": 1,
               "format": "short"
             }, {
               "show": true,
               "min": null,
               "max": null,
               "logBase": 1,
               "format": "short"
             }],
             "xaxis": {
               "show": true
             }
           }, {
             "aliasColors": {},
             "bars": false,
             "datasource": "${DS_PROMETHEUS}",
             "editable": true,
             "error": false,
             "fill": 1,
             "grid": {
               "threshold1": null,
               "threshold1Color": "rgba(216, 200, 27, 0.27)",
               "threshold2": null,
               "threshold2Color": "rgba(234, 112, 112, 0.22)"
             },
             "id": 14,
             "legend": {
               "avg": false,
               "current": false,
               "max": false,
               "min": false,
               "show": true,
               "total": false,
               "values": false
             },
             "lines": true,
             "linewidth": 2,
             "links": [],
             "nullPointMode": "connected",
             "percentage": false,
             "pointradius": 5,
             "points": false,
             "renderer": "flot",
             "seriesOverrides": [],
             "span": 4,
             "stack": false,
             "steppedLine": false,
             "targets": [{
               "expr": "prometheus_target_interval_length_seconds{quantile!=\"0.01\", quantile!=\"0.05\"}",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "{{quantile}} ({{interval}})",
               "metric": "",
               "refId": "A",
               "step": 2
             }],
             "timeFrom": null,
             "timeShift": null,
             "title": "Scrape Duration",
             "tooltip": {
               "shared": true,
               "value_type": "cumulative",
               "ordering": "alphabetical",
               "msResolution": false
             },
             "type": "graph",
             "yaxes": [{
               "show": true,
               "min": null,
               "max": null,
               "logBase": 1,
               "format": "short"
             }, {
               "show": true,
               "min": null,
               "max": null,
               "logBase": 1,
               "format": "short"
             }],
             "xaxis": {
               "show": true
             }
           }, {
             "content": "#### Scrapes\nPrometheus scrapes metrics from instrumented jobs, either directly or via an intermediary push gateway for short-lived jobs. Target scrapes will show how frequently targets are scraped, as measured over the last 5 minutes, per time series in the range vector. Scrape Duration will show how long the scrapes are taking, with percentiles available as series. ",
             "editable": true,
             "error": false,
             "id": 11,
             "links": [],
             "mode": "markdown",
             "span": 3,
             "style": {},
             "title": "",
             "transparent": true,
             "type": "text"
           }],
           "title": "New row"
         }, {
           "collapse": false,
           "editable": true,
           "height": "250px",
           "panels": [{
             "aliasColors": {},
             "bars": false,
             "datasource": "${DS_PROMETHEUS}",
             "decimals": null,
             "editable": true,
             "error": false,
             "fill": 1,
             "grid": {
               "threshold1": null,
               "threshold1Color": "rgba(216, 200, 27, 0.27)",
               "threshold2": null,
               "threshold2Color": "rgba(234, 112, 112, 0.22)"
             },
             "id": 12,
             "legend": {
               "alignAsTable": false,
               "avg": false,
               "current": false,
               "hideEmpty": true,
               "max": false,
               "min": false,
               "show": true,
               "total": false,
               "values": false
             },
             "lines": true,
             "linewidth": 2,
             "links": [],
             "nullPointMode": "connected",
             "percentage": false,
             "pointradius": 5,
             "points": false,
             "renderer": "flot",
             "seriesOverrides": [],
             "span": 9,
             "stack": false,
             "steppedLine": false,
             "targets": [{
               "expr": "prometheus_evaluator_duration_milliseconds{quantile!=\"0.01\", quantile!=\"0.05\"}",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "{{quantile}}",
               "refId": "A",
               "step": 2
             }],
             "timeFrom": null,
             "timeShift": null,
             "title": "Rule Eval Duration",
             "tooltip": {
               "shared": true,
               "value_type": "cumulative",
               "ordering": "alphabetical",
               "msResolution": false
             },
             "type": "graph",
             "yaxes": [{
               "show": true,
               "min": null,
               "max": null,
               "logBase": 1,
               "format": "percentunit",
               "label": ""
             }, {
               "show": true,
               "min": null,
               "max": null,
               "logBase": 1,
               "format": "short"
             }],
             "xaxis": {
               "show": true
             }
           }, {
             "content": "#### Rule Evaluation Duration\nThis graph panel plots the duration for all evaluations to execute. The 50th percentile, 90th percentile and 99th percentile are shown as three separate series to help identify outliers that may be skewing the data.",
             "editable": true,
             "error": false,
             "id": 15,
             "links": [],
             "mode": "markdown",
             "span": 3,
             "style": {},
             "title": "",
             "transparent": true,
             "type": "text"
           }],
           "title": "New row"
         }],
         "time": {
           "from": "now-5m",
           "to": "now"
         },
         "timepicker": {
           "now": true,
           "refresh_intervals": ["5s", "10s", "30s", "1m", "5m", "15m", "30m", "1h", "2h", "1d"],
           "time_options": ["5m", "15m", "1h", "6h", "12h", "24h", "2d", "7d", "30d"]
         },
         "templating": {
           "list": []
         },
         "annotations": {
           "list": []
         },
         "refresh": false,
         "schemaVersion": 12,
         "version": 0,
         "links": [{
           "icon": "info",
           "tags": [],
           "targetBlank": true,
           "title": "Grafana Docs",
           "tooltip": "",
           "type": "link",
           "url": "http://www.grafana.org/docs"
         }, {
           "icon": "info",
           "tags": [],
           "targetBlank": true,
           "title": "Prometheus Docs",
           "type": "link",
           "url": "http://prometheus.io/docs/introduction/overview/"
         }],
         "gnetId": 2,
         "description": "The  official, pre-built Prometheus Stats Dashboard."
       }
     grafana-net-737-dashboard.json: |
       {
         "__inputs": [{
           "name": "DS_PROMETHEUS",
           "label": "prometheus",
           "description": "",
           "type": "datasource",
           "pluginId": "prometheus",
           "pluginName": "Prometheus"
         }],
         "__requires": [{
           "type": "panel",
           "id": "singlestat",
           "name": "Singlestat",
           "version": ""
         }, {
           "type": "panel",
           "id": "graph",
           "name": "Graph",
           "version": ""
         }, {
           "type": "grafana",
           "id": "grafana",
           "name": "Grafana",
           "version": "3.1.0"
         }, {
           "type": "datasource",
           "id": "prometheus",
           "name": "Prometheus",
           "version": "1.0.0"
         }],
         "id": null,
         "title": "Kubernetes Pod Resources",
         "description": "Shows resource usage of Kubernetes pods.",
         "tags": [
           "kubernetes"
         ],
         "style": "dark",
         "timezone": "browser",
         "editable": true,
         "hideControls": false,
         "sharedCrosshair": false,
         "rows": [{
           "collapse": false,
           "editable": true,
           "height": "250px",
           "panels": [{
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": true,
             "colors": [
               "rgba(50, 172, 45, 0.97)",
               "rgba(237, 129, 40, 0.89)",
               "rgba(245, 54, 54, 0.9)"
             ],
             "datasource": "${DS_PROMETHEUS}",
             "editable": true,
             "error": false,
             "format": "percent",
             "gauge": {
               "maxValue": 100,
               "minValue": 0,
               "show": true,
               "thresholdLabels": false,
               "thresholdMarkers": true
             },
             "height": "180px",
             "id": 4,
             "interval": null,
             "isNew": true,
             "links": [],
             "mappingType": 1,
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "50%",
             "prefix": "",
             "prefixFontSize": "50%",
             "rangeMaps": [{
               "from": "null",
               "text": "N/A",
               "to": "null"
             }],
             "span": 4,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "sum (container_memory_working_set_bytes{id=\"/\",instance=~\"^$instance$\"}) / sum (machine_memory_bytes{instance=~\"^$instance$\"}) * 100",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "",
               "refId": "A",
               "step": 2
             }],
             "thresholds": "65, 90",
             "timeFrom": "1m",
             "timeShift": null,
             "title": "Memory Working Set",
             "transparent": false,
             "type": "singlestat",
             "valueFontSize": "80%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current"
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": true,
             "colors": [
               "rgba(50, 172, 45, 0.97)",
               "rgba(237, 129, 40, 0.89)",
               "rgba(245, 54, 54, 0.9)"
             ],
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "format": "percent",
             "gauge": {
               "maxValue": 100,
               "minValue": 0,
               "show": true,
               "thresholdLabels": false,
               "thresholdMarkers": true
             },
             "height": "180px",
             "id": 6,
             "interval": null,
             "isNew": true,
             "links": [],
             "mappingType": 1,
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "50%",
             "prefix": "",
             "prefixFontSize": "50%",
             "rangeMaps": [{
               "from": "null",
               "text": "N/A",
               "to": "null"
             }],
             "span": 4,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "sum(rate(container_cpu_usage_seconds_total{id=\"/\",instance=~\"^$instance$\"}[1m])) / sum (machine_cpu_cores{instance=~\"^$instance$\"}) * 100",
               "interval": "10s",
               "intervalFactor": 1,
               "refId": "A",
               "step": 10
             }],
             "thresholds": "65, 90",
             "timeFrom": "1m",
             "timeShift": null,
             "title": "Cpu Usage",
             "type": "singlestat",
             "valueFontSize": "80%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current"
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": true,
             "colors": [
               "rgba(50, 172, 45, 0.97)",
               "rgba(237, 129, 40, 0.89)",
               "rgba(245, 54, 54, 0.9)"
             ],
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "format": "percent",
             "gauge": {
               "maxValue": 100,
               "minValue": 0,
               "show": true,
               "thresholdLabels": false,
               "thresholdMarkers": true
             },
             "height": "180px",
             "id": 7,
             "interval": null,
             "isNew": true,
             "links": [],
             "mappingType": 1,
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "50%",
             "prefix": "",
             "prefixFontSize": "50%",
             "rangeMaps": [{
               "from": "null",
               "text": "N/A",
               "to": "null"
             }],
             "span": 4,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "sum(container_fs_usage_bytes{id=\"/\",instance=~\"^$instance$\"}) / sum(container_fs_limit_bytes{id=\"/\",instance=~\"^$instance$\"}) * 100",
               "interval": "10s",
               "intervalFactor": 1,
               "legendFormat": "",
               "metric": "",
               "refId": "A",
               "step": 10
             }],
             "thresholds": "65, 90",
             "timeFrom": "1m",
             "timeShift": null,
             "title": "Filesystem Usage",
             "type": "singlestat",
             "valueFontSize": "80%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current"
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": false,
             "colors": [
               "rgba(50, 172, 45, 0.97)",
               "rgba(237, 129, 40, 0.89)",
               "rgba(245, 54, 54, 0.9)"
             ],
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "format": "bytes",
             "gauge": {
               "maxValue": 100,
               "minValue": 0,
               "show": false,
               "thresholdLabels": false,
               "thresholdMarkers": true
             },
             "height": "1px",
             "hideTimeOverride": true,
             "id": 9,
             "interval": null,
             "isNew": true,
             "links": [],
             "mappingType": 1,
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "20%",
             "prefix": "",
             "prefixFontSize": "20%",
             "rangeMaps": [{
               "from": "null",
               "text": "N/A",
               "to": "null"
             }],
             "span": 2,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "sum(container_memory_working_set_bytes{id=\"/\",instance=~\"^$instance$\"})",
               "interval": "10s",
               "intervalFactor": 1,
               "refId": "A",
               "step": 10
             }],
             "thresholds": "",
             "timeFrom": "1m",
             "title": "Used",
             "type": "singlestat",
             "valueFontSize": "50%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current"
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": false,
             "colors": [
               "rgba(50, 172, 45, 0.97)",
               "rgba(237, 129, 40, 0.89)",
               "rgba(245, 54, 54, 0.9)"
             ],
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "format": "bytes",
             "gauge": {
               "maxValue": 100,
               "minValue": 0,
               "show": false,
               "thresholdLabels": false,
               "thresholdMarkers": true
             },
             "height": "1px",
             "hideTimeOverride": true,
             "id": 10,
             "interval": null,
             "isNew": true,
             "links": [],
             "mappingType": 1,
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "50%",
             "prefix": "",
             "prefixFontSize": "50%",
             "rangeMaps": [{
               "from": "null",
               "text": "N/A",
               "to": "null"
             }],
             "span": 2,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "sum (machine_memory_bytes{instance=~\"^$instance$\"})",
               "interval": "10s",
               "intervalFactor": 1,
               "refId": "A",
               "step": 10
             }],
             "thresholds": "",
             "timeFrom": "1m",
             "title": "Total",
             "type": "singlestat",
             "valueFontSize": "50%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current"
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": false,
             "colors": [
               "rgba(50, 172, 45, 0.97)",
               "rgba(237, 129, 40, 0.89)",
               "rgba(245, 54, 54, 0.9)"
             ],
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "format": "none",
             "gauge": {
               "maxValue": 100,
               "minValue": 0,
               "show": false,
               "thresholdLabels": false,
               "thresholdMarkers": true
             },
             "height": "1px",
             "hideTimeOverride": true,
             "id": 11,
             "interval": null,
             "isNew": true,
             "links": [],
             "mappingType": 1,
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": " cores",
             "postfixFontSize": "30%",
             "prefix": "",
             "prefixFontSize": "50%",
             "rangeMaps": [{
               "from": "null",
               "text": "N/A",
               "to": "null"
             }],
             "span": 2,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "sum (rate (container_cpu_usage_seconds_total{id=\"/\",instance=~\"^$instance$\"}[1m]))",
               "interval": "10s",
               "intervalFactor": 1,
               "refId": "A",
               "step": 10
             }],
             "thresholds": "",
             "timeFrom": "1m",
             "timeShift": null,
             "title": "Used",
             "type": "singlestat",
             "valueFontSize": "50%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current"
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": false,
             "colors": [
               "rgba(50, 172, 45, 0.97)",
               "rgba(237, 129, 40, 0.89)",
               "rgba(245, 54, 54, 0.9)"
             ],
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "format": "none",
             "gauge": {
               "maxValue": 100,
               "minValue": 0,
               "show": false,
               "thresholdLabels": false,
               "thresholdMarkers": true
             },
             "height": "1px",
             "hideTimeOverride": true,
             "id": 12,
             "interval": null,
             "isNew": true,
             "links": [],
             "mappingType": 1,
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": " cores",
             "postfixFontSize": "30%",
             "prefix": "",
             "prefixFontSize": "50%",
             "rangeMaps": [{
               "from": "null",
               "text": "N/A",
               "to": "null"
             }],
             "span": 2,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "sum (machine_cpu_cores{instance=~\"^$instance$\"})",
               "interval": "10s",
               "intervalFactor": 1,
               "refId": "A",
               "step": 10
             }],
             "thresholds": "",
             "timeFrom": "1m",
             "title": "Total",
             "type": "singlestat",
             "valueFontSize": "50%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current"
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": false,
             "colors": [
               "rgba(50, 172, 45, 0.97)",
               "rgba(237, 129, 40, 0.89)",
               "rgba(245, 54, 54, 0.9)"
             ],
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "format": "bytes",
             "gauge": {
               "maxValue": 100,
               "minValue": 0,
               "show": false,
               "thresholdLabels": false,
               "thresholdMarkers": true
             },
             "height": "1px",
             "hideTimeOverride": true,
             "id": 13,
             "interval": null,
             "isNew": true,
             "links": [],
             "mappingType": 1,
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "50%",
             "prefix": "",
             "prefixFontSize": "50%",
             "rangeMaps": [{
               "from": "null",
               "text": "N/A",
               "to": "null"
             }],
             "span": 2,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "sum(container_fs_usage_bytes{id=\"/\",instance=~\"^$instance$\"})",
               "interval": "10s",
               "intervalFactor": 1,
               "refId": "A",
               "step": 10
             }],
             "thresholds": "",
             "timeFrom": "1m",
             "title": "Used",
             "type": "singlestat",
             "valueFontSize": "50%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current"
           }, {
             "cacheTimeout": null,
             "colorBackground": false,
             "colorValue": false,
             "colors": [
               "rgba(50, 172, 45, 0.97)",
               "rgba(237, 129, 40, 0.89)",
               "rgba(245, 54, 54, 0.9)"
             ],
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "format": "bytes",
             "gauge": {
               "maxValue": 100,
               "minValue": 0,
               "show": false,
               "thresholdLabels": false,
               "thresholdMarkers": true
             },
             "height": "1px",
             "hideTimeOverride": true,
             "id": 14,
             "interval": null,
             "isNew": true,
             "links": [],
             "mappingType": 1,
             "mappingTypes": [{
               "name": "value to text",
               "value": 1
             }, {
               "name": "range to text",
               "value": 2
             }],
             "maxDataPoints": 100,
             "nullPointMode": "connected",
             "nullText": null,
             "postfix": "",
             "postfixFontSize": "50%",
             "prefix": "",
             "prefixFontSize": "50%",
             "rangeMaps": [{
               "from": "null",
               "text": "N/A",
               "to": "null"
             }],
             "span": 2,
             "sparkline": {
               "fillColor": "rgba(31, 118, 189, 0.18)",
               "full": false,
               "lineColor": "rgb(31, 120, 193)",
               "show": false
             },
             "targets": [{
               "expr": "sum (container_fs_limit_bytes{id=\"/\",instance=~\"^$instance$\"})",
               "interval": "10s",
               "intervalFactor": 1,
               "refId": "A",
               "step": 10
             }],
             "thresholds": "",
             "timeFrom": "1m",
             "title": "Total",
             "type": "singlestat",
             "valueFontSize": "50%",
             "valueMaps": [{
               "op": "=",
               "text": "N/A",
               "value": "null"
             }],
             "valueName": "current"
           }, {
             "aliasColors": {},
             "bars": false,
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "fill": 1,
             "grid": {
               "threshold1": null,
               "threshold1Color": "rgba(216, 200, 27, 0.27)",
               "threshold2": null,
               "threshold2Color": "rgba(234, 112, 112, 0.22)",
               "thresholdLine": false
             },
             "height": "200px",
             "id": 32,
             "isNew": true,
             "legend": {
               "alignAsTable": true,
               "avg": true,
               "current": true,
               "max": false,
               "min": false,
               "rightSide": true,
               "show": true,
               "sideWidth": 200,
               "sort": "current",
               "sortDesc": true,
               "total": false,
               "values": true
             },
             "lines": true,
             "linewidth": 2,
             "links": [],
             "nullPointMode": "connected",
             "percentage": false,
             "pointradius": 5,
             "points": false,
             "renderer": "flot",
             "seriesOverrides": [],
             "span": 12,
             "stack": false,
             "steppedLine": false,
             "targets": [{
               "expr": "sum(rate(container_network_receive_bytes_total{instance=~\"^$instance$\",namespace=~\"^$namespace$\"}[1m]))",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "receive",
               "metric": "network",
               "refId": "A",
               "step": 240
             }, {
               "expr": "- sum(rate(container_network_transmit_bytes_total{instance=~\"^$instance$\",namespace=~\"^$namespace$\"}[1m]))",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "transmit",
               "metric": "network",
               "refId": "B",
               "step": 240
             }],
             "timeFrom": null,
             "timeShift": null,
             "title": "Network",
             "tooltip": {
               "msResolution": false,
               "shared": true,
               "sort": 0,
               "value_type": "cumulative"
             },
             "transparent": false,
             "type": "graph",
             "xaxis": {
               "show": true
             },
             "yaxes": [{
               "format": "Bps",
               "label": "transmit / receive",
               "logBase": 1,
               "max": null,
               "min": null,
               "show": true
             }, {
               "format": "Bps",
               "label": null,
               "logBase": 1,
               "max": null,
               "min": null,
               "show": false
             }]
           }],
           "showTitle": true,
           "title": "all pods"
         }, {
           "collapse": false,
           "editable": true,
           "height": "250px",
           "panels": [{
             "aliasColors": {},
             "bars": false,
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 3,
             "editable": true,
             "error": false,
             "fill": 0,
             "grid": {
               "threshold1": null,
               "threshold1Color": "rgba(216, 200, 27, 0.27)",
               "threshold2": null,
               "threshold2Color": "rgba(234, 112, 112, 0.22)"
             },
             "height": "",
             "id": 17,
             "isNew": true,
             "legend": {
               "alignAsTable": true,
               "avg": true,
               "current": true,
               "hideEmpty": true,
               "hideZero": true,
               "max": false,
               "min": false,
               "rightSide": true,
               "show": true,
               "sideWidth": null,
               "sort": "current",
               "sortDesc": true,
               "total": false,
               "values": true
             },
             "lines": true,
             "linewidth": 2,
             "links": [],
             "nullPointMode": "connected",
             "percentage": false,
             "pointradius": 5,
             "points": false,
             "renderer": "flot",
             "seriesOverrides": [],
             "span": 12,
             "stack": false,
             "steppedLine": false,
             "targets": [{
               "expr": "sum(rate(container_cpu_usage_seconds_total{image!=\"\",name=~\"^k8s_.*\"}[1m])) by (pod_name)",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "{{ pod_name }}",
               "metric": "container_cpu",
               "refId": "A",
               "step": 240
             }],
             "timeFrom": null,
             "timeShift": null,
             "title": "Cpu Usage",
             "tooltip": {
               "msResolution": true,
               "shared": false,
               "sort": 2,
               "value_type": "cumulative"
             },
             "transparent": false,
             "type": "graph",
             "xaxis": {
               "show": true
             },
             "yaxes": [{
               "format": "none",
               "label": "cores",
               "logBase": 1,
               "max": null,
               "min": null,
               "show": true
             }, {
               "format": "short",
               "label": null,
               "logBase": 1,
               "max": null,
               "min": null,
               "show": false
             }]
           }, {
             "aliasColors": {},
             "bars": false,
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "fill": 0,
             "grid": {
               "threshold1": null,
               "threshold1Color": "rgba(216, 200, 27, 0.27)",
               "threshold2": null,
               "threshold2Color": "rgba(234, 112, 112, 0.22)"
             },
             "id": 33,
             "isNew": true,
             "legend": {
               "alignAsTable": true,
               "avg": true,
               "current": true,
               "hideEmpty": true,
               "hideZero": true,
               "max": false,
               "min": false,
               "rightSide": true,
               "show": true,
               "sideWidth": null,
               "sort": "current",
               "sortDesc": true,
               "total": false,
               "values": true
             },
             "lines": true,
             "linewidth": 2,
             "links": [],
             "nullPointMode": "null",
             "percentage": false,
             "pointradius": 5,
             "points": false,
             "renderer": "flot",
             "seriesOverrides": [],
             "span": 12,
             "stack": false,
             "steppedLine": false,
             "targets": [{
               "expr": "sum (container_memory_working_set_bytes{image!=\"\",name=~\"^k8s_.*\",instance=~\"^$instance$\",namespace=~\"^$namespace$\"}) by (pod_name)",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "{{ pod_name }}",
               "metric": "",
               "refId": "A",
               "step": 240
             }],
             "timeFrom": null,
             "timeShift": null,
             "title": "Memory Working Set",
             "tooltip": {
               "msResolution": false,
               "shared": false,
               "sort": 2,
               "value_type": "cumulative"
             },
             "type": "graph",
             "xaxis": {
               "show": true
             },
             "yaxes": [{
               "format": "bytes",
               "label": "used",
               "logBase": 1,
               "max": null,
               "min": null,
               "show": true
             }, {
               "format": "short",
               "label": null,
               "logBase": 1,
               "max": null,
               "min": null,
               "show": false
             }]
           }, {
             "aliasColors": {},
             "bars": false,
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "fill": 1,
             "grid": {
               "threshold1": null,
               "threshold1Color": "rgba(216, 200, 27, 0.27)",
               "threshold2": null,
               "threshold2Color": "rgba(234, 112, 112, 0.22)"
             },
             "id": 16,
             "isNew": true,
             "legend": {
               "alignAsTable": true,
               "avg": true,
               "current": true,
               "hideEmpty": true,
               "hideZero": true,
               "max": false,
               "min": false,
               "rightSide": true,
               "show": true,
               "sideWidth": 200,
               "sort": "avg",
               "sortDesc": true,
               "total": false,
               "values": true
             },
             "lines": true,
             "linewidth": 2,
             "links": [],
             "nullPointMode": "null",
             "percentage": false,
             "pointradius": 5,
             "points": false,
             "renderer": "flot",
             "seriesOverrides": [],
             "span": 12,
             "stack": false,
             "steppedLine": false,
             "targets": [{
               "expr": "sum (rate (container_network_receive_bytes_total{image!=\"\",name=~\"^k8s_.*\",instance=~\"^$instance$\",namespace=~\"^$namespace$\"}[1m])) by (pod_name)",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "{{ pod_name }} < in",
               "metric": "network",
               "refId": "A",
               "step": 240
             }, {
               "expr": "- sum (rate (container_network_transmit_bytes_total{image!=\"\",name=~\"^k8s_.*\",instance=~\"^$instance$\",namespace=~\"^$namespace$\"}[1m])) by (pod_name)",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "{{ pod_name }} > out",
               "metric": "network",
               "refId": "B",
               "step": 240
             }],
             "timeFrom": null,
             "timeShift": null,
             "title": "Network",
             "tooltip": {
               "msResolution": false,
               "shared": false,
               "sort": 2,
               "value_type": "cumulative"
             },
             "type": "graph",
             "xaxis": {
               "show": true
             },
             "yaxes": [{
               "format": "Bps",
               "label": "transmit / receive",
               "logBase": 1,
               "max": null,
               "min": null,
               "show": true
             }, {
               "format": "short",
               "label": null,
               "logBase": 1,
               "max": null,
               "min": null,
               "show": false
             }]
           }, {
             "aliasColors": {},
             "bars": false,
             "datasource": "${DS_PROMETHEUS}",
             "decimals": 2,
             "editable": true,
             "error": false,
             "fill": 1,
             "grid": {
               "threshold1": null,
               "threshold1Color": "rgba(216, 200, 27, 0.27)",
               "threshold2": null,
               "threshold2Color": "rgba(234, 112, 112, 0.22)"
             },
             "id": 34,
             "isNew": true,
             "legend": {
               "alignAsTable": true,
               "avg": true,
               "current": true,
               "hideEmpty": true,
               "hideZero": true,
               "max": false,
               "min": false,
               "rightSide": true,
               "show": true,
               "sideWidth": 200,
               "sort": "current",
               "sortDesc": true,
               "total": false,
               "values": true
             },
             "lines": true,
             "linewidth": 2,
             "links": [],
             "nullPointMode": "null",
             "percentage": false,
             "pointradius": 5,
             "points": false,
             "renderer": "flot",
             "seriesOverrides": [],
             "span": 12,
             "stack": false,
             "steppedLine": false,
             "targets": [{
               "expr": "sum(container_fs_usage_bytes{image!=\"\",name=~\"^k8s_.*\",instance=~\"^$instance$\",namespace=~\"^$namespace$\"}) by (pod_name)",
               "interval": "",
               "intervalFactor": 2,
               "legendFormat": "{{ pod_name }}",
               "metric": "network",
               "refId": "A",
               "step": 240
             }],
             "timeFrom": null,
             "timeShift": null,
             "title": "Filesystem",
             "tooltip": {
               "msResolution": false,
               "shared": false,
               "sort": 2,
               "value_type": "cumulative"
             },
             "type": "graph",
             "xaxis": {
               "show": true
             },
             "yaxes": [{
               "format": "bytes",
               "label": "used",
               "logBase": 1,
               "max": null,
               "min": null,
               "show": true
             }, {
               "format": "short",
               "label": null,
               "logBase": 1,
               "max": null,
               "min": null,
               "show": false
             }]
           }],
           "showTitle": true,
           "title": "each pod"
         }],
         "time": {
           "from": "now-3d",
           "to": "now"
         },
         "timepicker": {
           "refresh_intervals": [
             "5s",
             "10s",
             "30s",
             "1m",
             "5m",
             "15m",
             "30m",
             "1h",
             "2h",
             "1d"
           ],
           "time_options": [
             "5m",
             "15m",
             "1h",
             "6h",
             "12h",
             "24h",
             "2d",
             "7d",
             "30d"
           ]
         },
         "templating": {
           "list": [{
             "allValue": ".*",
             "current": {},
             "datasource": "${DS_PROMETHEUS}",
             "hide": 0,
             "includeAll": true,
             "label": "Instance",
             "multi": false,
             "name": "instance",
             "options": [],
             "query": "label_values(instance)",
             "refresh": 1,
             "regex": "",
             "type": "query"
           }, {
             "current": {},
             "datasource": "${DS_PROMETHEUS}",
             "hide": 0,
             "includeAll": true,
             "label": "Namespace",
             "multi": true,
             "name": "namespace",
             "options": [],
             "query": "label_values(namespace)",
             "refresh": 1,
             "regex": "",
             "type": "query"
           }]
         },
         "annotations": {
           "list": []
         },
         "refresh": false,
         "schemaVersion": 12,
         "version": 8,
         "links": [],
         "gnetId": 737
       }
     prometheus-datasource.json: |
       {
         "name": "prometheus",
         "type": "prometheus",
         "url": "http://prometheus:9090",
         "access": "proxy",
         "basicAuth": false
       }
   
   ```

   ```shell
   $ kubectl apply -f grafana-net-2-dashboard-configmap.yml
   ```

   可以到 https://grafana.com/grafana/dashboards 查看相关Dashboard模版的设置

2. 部署Grafana Server的 Deployment

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: grafana-core
     namespace: monitoring
     labels:
       app: grafana
       component: core
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: grafana
         component: core
     template:
       metadata:
         labels:
           app: grafana
           component: core
       spec:
         containers:
   #     - image: grafana/grafana:4.2.0
         - image: grafana/grafana:4.2.0
           name: grafana-core
           imagePullPolicy: IfNotPresent
           # env:
           resources:
             # keep request = limit to keep this container in guaranteed class
             limits:
               cpu: 100m
               memory: 100Mi
             requests:
               cpu: 100m
               memory: 100Mi
           env:
             # The following env variables set up basic auth twith the default admin user and admin password.
             - name: GF_AUTH_BASIC_ENABLED
               value: "true"
             - name: GF_SECURITY_ADMIN_USER
               valueFrom:
                 secretKeyRef:
                   name: grafana
                   key: admin-username
             - name: GF_SECURITY_ADMIN_PASSWORD
               valueFrom:
                 secretKeyRef:
                   name: grafana
                   key: admin-password
             - name: GF_AUTH_ANONYMOUS_ENABLED
               value: "false"
             # - name: GF_AUTH_ANONYMOUS_ORG_ROLE
             #   value: Admin
             # does not really work, because of template variables in exported dashboards:
             # - name: GF_DASHBOARDS_JSON_ENABLED
             #   value: "true"
           readinessProbe:
             httpGet:
               path: /login
               port: 3000
             # initialDelaySeconds: 30
             # timeoutSeconds: 1
           volumeMounts:
           - name: grafana-persistent-storage
             mountPath: /var/lib/grafana
         volumes:
         - name: grafana-persistent-storage
           emptyDir: {}
   ```

      ```shell
      $ kubectl apply -f grafana-deploy.yml
      ```

3. 部署Grafana Server的 Service  

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: grafana
     namespace: monitoring
     labels:
       app: grafana
       component: core
   spec:
     type: NodePort
     ports:
       - port: 3000
     selector:
       app: grafana
       component: core
   ```

      ```shell
      $ kubectl apply -f grafana-service.yml
      ```

4. 为Grafana添加数据源，并创建Dashboard  

   登录 Grafana 进行 Prometheus 数据源配置
   注意：必须在创建Grafana Service之后再运行  

## 应用暴露监控数据

1. springboot 应用添加依赖配置：

   ```xml
   <!--配置Prometheus的监控-->
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-actuator</artifactId>
   </dependency>
   <dependency>
       <groupId>io.micrometer</groupId>
       <artifactId>micrometer-registry-prometheus</artifactId>
   </dependency>
   ```

2. 重新打包 kubeblog.jar

3. 重新打包上传镜像

4. 更新 Helm Chart  

   在 values 文件中修改镜像为新的版本  

## 添加应用监控配置项

