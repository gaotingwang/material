# 持久化

Kubernetes提供了众多的volume类型，包括emptyDir、hostPath、nfs、glusterfs、cephfs、ceph等

## emptyDir

当 Pod 指定到某个节点上时，首先创建的是一个 emptyDir 卷，并且只要 Pod 在该节点上运行，卷就一直存在。 就像它的名称表示的那样，卷最初是空的。 尽管 Pod 中的容器挂载 emptyDir 卷的路径可能相同也可能不同，但是这些容器都可以读写 emptyDir 卷中相同的文件。 当 Pod 因为某些原因被从节点上删除时，emptyDir 卷中的数据也会永久删除。

说明： 容器崩溃并不会导致 Pod 被从节点上移除，因此容器崩溃时 emptyDir 卷中的数据是安全的。

emptyDir 的一些用途：

- 缓存空间，例如基于磁盘的归并排序。
- 为耗时较长的计算任务提供检查点，以便任务能方便地从崩溃前状态恢复执行。
- 在 Web 服务器容器服务数据时，保存内容管理器类型容器获取的文件  

## hostPath

hostPath卷能将主机节点文件系统上的文件或目录挂载到Pod中。虽然这不是大多数Pod需要的，但是它为一些应用程序提供了强大的持久化能力。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
    - name: test-container
      image: nginx
      volumeMounts:
        # 容器中要挂载的目录
        - mountPath: /test-nginx
          # 根据名称，会找到对应的volumes进行挂载
          name: myhostpath
  volumes:
    # 节点上的挂载目录
    - name: myhostpath
      # 使用 hostPath
      hostPath:
        path: /tmp/nginx
        type: DirectoryOrCreate
```

hostPath的局限性在于，节点之间数据的共享，同时如果节点不存在了，则持久化的文件也就丢失了

## NFS 卷

### 安装 NFS

```shell
# 在 Master 和 所有Worker node 安装 nfs 服务
$ yum install -y nfs-utils rpcbind

# 修改 master 节点配置
$ vi /etc/exports
/nfsdata *(rw,sync,no_root_squash)

# 在 master 节点系统自动启动
$ systemctl enable --now rpcbind
$ systemctl enable --now nfs

# 重新激活配置
$ exportfs -r

# 查看 nfs 挂载效果
$ showmount -e master
Export list for master:
/nfsdata *
```

### Pod 引用 NFS 存储

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: nginx
      volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: nfs-volume
  volumes:
    - name: nfs-volume
      # 使用 nfs 挂载
      nfs:
        path: /nfsdata
        # nfs 服务在master节点
        server: master
```

在容器中，挂载目录下的变更，会同步到 master 节点的`/nfsdata`目录下

nfs适用于文件不大、数量不多的挂载，如果是海量文件还是适合适用对象存储

## 持久化存储 PersistantVolume

持久卷（PersistentVolume，PV）是集群中的一块存储，可以由管理员事先供应，或者使用存储类（Storage Class）来动态供应。 持久卷是集群资源，就像节点也是集群资源一样。PV 持久卷和普通的Volume 一样，也是使用卷插件来实现的，**只是它们拥有独立于任何使用 PV 的 Pod 的生命周期**。 

### 持久卷

`PersistentVolume  ` API 对象中记述了存储的实现细节，无论其背后是 NFS、iSCSI 还是特定于云平台的存储系统  

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-demo
spec:
  capacity:
    # 存储空间为5G
    storage: 5Gi
  accessModes:
    # 一次只允许一个节点进行读写
    - ReadWriteOnce
  # 可进行回收
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /nfsdata
    server: master
```

```shell
$ kubectl get pv pv-demo
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM               STORAGECLASS       REASON   AGE
pv-demo    5Gi        RWO            Recycle          Available                                                   90s
```

### 持久卷申领

表达的是用户对存储的请求，PVC 申领会耗用 PV 资源。PVC 申领可以请求特定的大小和访问模式 （例如，可以要求 PV 卷能够以 ReadWriteOnce、
ReadOnlyMany 或 ReadWriteMany 模式之一来挂载）。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-demo
spec:
  accessModes:
    - ReadWriteOnce
  # 可以请求特定数量的资源。在这个上下文中，请求的资源是存储。卷和Claim都使用相同的资源模型
  resources:
    requests:
      storage: 2Gi
  # 设置标签选择算符来进一步过滤卷集合。只有标签与选择算符相匹配的卷能够绑定到Claim上
  selector:
    # 卷必须包含带有此值的标签
    matchLabels:
      release: "stable"
    matchExpressions:
      - { key: environment, operator: In, values: [ dev ] }
```

```shell
$ kubectl get pvc
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
pvc-demo    Bound    pv-demo    5Gi        RWO                               56s

# pv 状态变为 bound
$ kc get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS       REASON   AGE
pv-demo    5Gi        RWO            Recycle          Bound    default/pvc-demo                                33m

```

### StorageClass

一个大规模的Kubernetes集群里，可能有成千上万个PVC，这就意味着运维人员必须实现创建出这个多个PV。此外，随着项目的需要，会有新的PVC不断被提交，那么运维人员就需要不断的添加新的，满足要求的PV，否则新的Pod就会因为PVC绑定不到PV而导致创建失败。而且通过 PVC 请求到一定的存储空间也很有可能不足以满足应用对于存储设备的各种需求，不同的应用程序对于存储性能的要求可能也不尽相同，比如读写速度、并发性能等。

集群管理员需要能够提供不同性质的 PersistentVolume，并且这些 PV 卷之间的差别不仅限于卷大小和访问模式，同时又不能将卷是如何实现的这些细节暴露给用户。 为了满足这类需求，就有了存储类（StorageClass） 资源。  

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/storage-class.png" style="zoom:80%;" />

Kubernetes 提供了一套可以自动创建 PV 的机制 , 即 :Dynamic Provisioning。这个机制的核心在于:StorageClass这个API对象。通过 StorageClass 的定义，管理员可以将存储资源定义为某种类型的资源，比如快速存储、慢速存储等，用户根据 StorageClass 的描述，可以非常直观的知道各种存储资源的具体特性了，这样就可以根据应用的特性去申请合适的存储资源了

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
# 这里的名称要和provisioner配置文件中的环境变量PROVISIONER_NAME保持一致
provisioner: qgg-nfs-storage
parameters:
  archiveOnDelete: "false"
```

1. 创建用户和角色 RBAC

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     # 根据实际环境设定namespace,下同
     namespace: default        
   ---
   # 集群级别角色
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: nfs-client-provisioner-runner
   rules:
     - apiGroups: [""]
       resources: ["persistentvolumes"]
       verbs: ["get", "list", "watch", "create", "delete"]
     - apiGroups: [""]
       resources: ["persistentvolumeclaims"]
       verbs: ["get", "list", "watch", "update"]
     - apiGroups: ["storage.k8s.io"]
       resources: ["storageclasses"]
       verbs: ["get", "list", "watch"]
     - apiGroups: [""]
       resources: ["events"]
       verbs: ["create", "update", "patch"]
   ---
   kind: ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: run-nfs-client-provisioner
   subjects:
     - kind: ServiceAccount
       name: nfs-client-provisioner
       # replace with namespace where provisioner is deployed
       namespace: default
   roleRef:
     kind: ClusterRole
     name: nfs-client-provisioner-runner
     apiGroup: rbac.authorization.k8s.io
   ---
   # 应用级别角色
   kind: Role
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     namespace: default
   rules:
     - apiGroups: [""]
       resources: ["endpoints"]
       verbs: ["get", "list", "watch", "create", "update", "patch"]
   ---
   kind: RoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-client-provisioner
   subjects:
     - kind: ServiceAccount
       name: nfs-client-provisioner
       # replace with namespace where provisioner is deployed
       namespace: default
   roleRef:
     kind: Role
     name: leader-locking-nfs-client-provisioner
     apiGroup: rbac.authorization.k8s.io
   ```

   

2. 创建 Persistent Volume Provisioner

   通过为该容器定义ENV 变量：PROVISIONER_NAME为qgg-nfs-storage，该容器会动态生产一个名为qgg-nfs-storage 的持久卷，从而能够实现持久卷的动态创建，无需手工创建

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nfs-client-provisioner
     labels:
       app: nfs-client-provisioner
     # replace with namespace where provisioner is deployed
     namespace: default  #与RBAC文件中的namespace保持一致
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: nfs-client-provisioner
     strategy:
       type: Recreate
     template:
       metadata:
         labels:
           app: nfs-client-provisioner
       spec:
         serviceAccountName: nfs-client-provisioner
         containers:
           - name: nfs-client-provisioner
             image: registry.cn-beijing.aliyuncs.com/qingfeng666/nfs-client-provisioner:v3.1.0
             volumeMounts:
               - name: nfs-client-root
                 mountPath: /persistentvolumes
             env:
               - name: PROVISIONER_NAME
                 # provisioner名称,请确保该名称与 nfs-StorageClass.yaml文件中的provisioner名称保持一致
                 value: qgg-nfs-storage
               - name: NFS_SERVER
                 # NFS Server IP地址
                 value: master
               - name: NFS_PATH
                 #NFS 挂载卷
                 value: /nfsdata
         volumes:
           - name: nfs-client-root
             nfs:
               # NFS Server IP地址
               server: master
               # NFS 挂载卷
               path: /nfsdata
   ```

3. 创建Storage Class

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: managed-nfs-storage
   # 这里的名称要和provisioner配置文件中的环境变量PROVISIONER_NAME保持一致
   provisioner: qgg-nfs-storage
   parameters:
     archiveOnDelete: "false"
   ```

4. 创建 PVC

   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: test-claim
     annotations:
       # 与nfs-StorageClass.yaml metadata.name保持一致
       volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
   spec:
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 1Mi
   ```

5. 验证创建 Pod，引用这个 Storage Class  

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-pod
   spec:
     containers:
       - name: test-pod
         image: nginx
         command:
           - "/bin/sh"
         args:
           - "-c"
           - "touch /mnt/SUCCESS && exit 0 || exit 1"   # 创建一个SUCCESS文件后退出
         volumeMounts:
           - name: nfs-pvc
             mountPath: "/mnt"
     restartPolicy: "Never"
     volumes:
       - name: nfs-pvc
         persistentVolumeClaim:
           # 与PVC名称保持一致
           claimName: test-claim
   ```

   登录 master 的`/nfsdata` 目录，可以看到该目录下为 test-pod 自动创建了一个文件夹，里面创建一个SUCCESS 文件  







