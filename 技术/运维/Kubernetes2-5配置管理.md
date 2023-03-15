# 配置管理

## Configmap

ConfigMap 是一种 API 对象，用来将非机密性的数据保存到健值对中。使用时可以用作环境变量、命令行参数或者存储卷中的配置文件。ConfigMap 并不提供保密或者加密功能。如果想存储的数据是机密的，请使用 Secret ，或者使用第三方工具来保证你的数据的私密性，而不是用 ConfigMap 。

ConfigMap 定义：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: appvar
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```

ConfigMap 使用：

可以使用四种方式来使用 ConfigMap 配置 Pod 中的容器：

1. 容器 entrypoint 的命令行参数
2. 容器的环境变量
3. 在只读卷里面添加一个文件，让应用来读取
4. 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap

这些不同的方法适用于不同的数据使用方式。对前三个方法，kubelet 使用 ConfigMap 中的数据在Pod 中启动容器  

可以写一个引用 ConfigMap 的 Pod，并根据 ConfigMap 中的数据在该 Pod 中配置容器。这个 Pod 和 ConfigMap 必须要在同一个命名空间中。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
    - name: test
      image: busybox
      command: ["/bin/sh", "-c", "env"]
      # envFrom 可以实现在pod容器中将ConfigMap中所有定义的 key=value 自动生成为环境变量
      envFrom:
        - configMapRef:
            # 指向定义的configmap的名称
            name: appvar
  restartPolicy: Never
```



##  Secret

Secret 对象类型用来保存敏感信息，例如密码、OAuth 令牌和 SSH 密钥。 将这些信息放在 secret 中比放在 Pod 的定义或者容器镜像中来说更加安全和灵活。 

 Secret 定义：

```shell
$ kubectl create secret [flags] [options]
flags Available Commands:
  docker-registry 创建一个给 Docker registry 使用的 secret
  generic         从本地 file, directory 或者 literal value 创建一个 secret
  tls             创建一个 TLS secret
```

```shell
# 创建本例要使用的文件
$ echo -n 'admin' > ./username.txt
$ echo -n '1f2d1e2e67df' > ./password.txt

# secret 创建
$ kubectl create secret generic db-user-pass \
	--from-file=username=./username.txt \
	--from-file=password=./password.txt
	
# secret 也可以为直接录入
$ kubectl create secret generic dev-db-secret \
	--from-literal=username=devuser \
	--from-literal=password='S!B\*d$zDsb='
	
# 查看创建的secret
$ kubectl get secrets
# 查看描述, 默认情况下，kubectl get 和 kubectl describe 避免显示密码的内容。
$ kubectl describe secrets/db-user-pass
Name: db-user-pass
Namespace: default
Labels: <none>
Annotations: <none>
Type: Opaque
Data
====
password.txt: 12 bytes
username.txt: 5 bytes
```

Pod 可以用三种方式之一来使用 Secret：

1. 作为挂载到一个或多个容器上的卷中的文件。
2. 作为容器的环境变量
3. 由 kubelet 在为 Pod 拉取镜像时使用  

