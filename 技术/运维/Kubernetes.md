# 环境准备

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/%E5%AE%9E%E6%88%98%E7%BB%93%E6%9E%84.jpeg" alt="架构" style="zoom:60%;" />

## Virtualbox安装CentOS配置

下载地址：[Downloads – Oracle VM VirtualBox](https://www.virtualbox.org/wiki/Downloads)、[Downloads - CentOS 7](https://mirrors.aliyun.com/centos/7/isos/x86_64/)

安装过程参考：[VirtualBox上安装CentOS7](https://zhuanlan.zhihu.com/p/60408219)

## 环境要求

- 3台虚拟机CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 集群中所有机器之间网络互通
- 可以访问外网，需要拉取镜像
- 禁止swap分区

## 虚拟机配置

- 配置虚拟机双网卡实现固定IP

  首先确认，在Virtualbox中的Network Manager是启用手动配置网卡功能，记录IPv4地址，后续修改静态ip会用到。为虚拟机配置网卡：

  网卡 1： 选择host-only网卡

  网卡 2： 选择网络地址转换(NAT)

- 设置固定ip，外部网络访问虚拟机

  设置静态ip地址，编辑网络配置文件，编辑网络设置文件  

  ```shell
  $ vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
  ```

  需要修改的内容如下：

  ```properties
  # 表示开机启动
  ONBOOT=yes
  # 为静态 IP
  BOOTPROTO=static
  IPADDR=192.168.56.101
  ```

  重启网络：

  ```shell
  $ systemctl restart network
  # 查看 enp0s3 网卡的 ip
  $ ip addr |grep 192
  ```

- 配置 Master 和 work 节点的域名  

  ```shell
  $ vim /etc/hosts
  192.168.56.101 master
  192.168.56.102 node1
  192.168.56.103 node2
  
  # 设置域名解析器服务
  $ vim /etc/resolv.conf
  nameserver 114.114.114.114
  ```

- 基础配置

  ```shell
  # 设置yum源
  $ curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Cent
  
  # 添加依赖
  $ yum install -y yum-utils device-mapper-persistent-data lvm2 wget
  $ yum install -y net-tools ntp git 
  
  # 同步系统时间
  $ ntpdate 0.asia.pool.ntp.org
  
  # 配置Docker, K8S的阿里云yum源
  $ cat >>/etc/yum.repos.d/kubernetes.repo <<EOF
  [kubernetes]
  name=Kubernetes
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirr
  EOF
  
  # 将桥接的IPv4流量传递到iptables
  # 为了让 Linux 节点上的 iptables 能够正确地查看桥接流量，需要确保在 sysctl 配置中将net.bridge.bridge-nf-call-iptables 设置为 1
  $ modprobe br_netfilter
  $ echo "1" >/proc/sys/net/bridge/bridge-nf-call-iptables
  $ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  EOF
  $ sudo sysctl --system
  
  # 关闭防火墙
  $ systemctl status firewalld.service # 查看防火墙
  $ systemctl stop firewalld # 关闭
  $ systemctl disable firewalld # 开机禁用
  
  # 关闭 SeLinux
  $ setenforce 0
  $ sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
  
  # 关闭 swap
  $ swapoff -a
  $ yes | cp /etc/fstab /etc/fstab_bak
  $ vi /etc/fstab
  # 注释下面这一行
  # /dev/mapper/centos-swap swap swap defaults 0 0
  
  ```
  
- 验证检查

  虚拟机可以ping通外网，可以和宿主机互通

## Docker安装环境配置

```shell

# 设置阿里云 Docker 的yum源
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# 查看仓库中所有Docker版本
$ yum list docker-ce --showduplicates | sort -r

# 安装Docker
$ yum -y install docker-ce

# 启动并加入开机启动
$ systemctl start docker
$ systemctl enable docker
```

## Mysql 安装

1. 挂载外部持久化配置和数据目录

   ```shell
   $ mkdir /opt/mysql
   $ mkdir /opt/mysql/conf.d
   $ mkdir /opt/mysql/data/
   $ chmod -R 777 /opt/mysql/
   ```

2. 创建my.cnf配置文件

   ```shell
   $ touch /opt/mysql/my.cnf
   ```

   my.cnf添加如下内容：

   ```
   [mysqld]
   user=mysql
   character-set-server=utf8
   default_authentication_plugin=mysql_native_password
   secure_file_priv=/var/lib/mysql
   expire_logs_days=7
   sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
   max_connections=1000
   
   [client]
   default-character-set=utf8
   
   [mysql]
   default-character-set=utf8
   ```

3. 运行 Mysql Docker 镜像

   ```shell
   # docker run --name mysql57 -p 3306:3306 --network test-network --privileged=true \
   $ docker run --name mysql57 -p 3306:3306 --privileged=true \
    -v /opt/mysql/data:/var/lib/mysql \
    -v /opt/mysql/log:/var/log/mysql \
    -v /opt/mysql/my.cnf:/etc/mysql/my.cnf:rw \
    -e MYSQL_ROOT_PASSWORD=password \
    -d mysql:5.7.30 --default-authentication-plugin=mysql_native_password
   ```

4. 登录 mysql server

   ```shell
   $ mysql -h 127.0.0.1 -uroot -p
   输入密码：password
   ```

5. 创建数据库 blogDB

   ```mysql
   CREATE DATABASE `blogDB` CHARACTER SET utf8 COLLATE utf8_general_ci;
   ```

## 构建博客镜像

下载代码,构建 mvn package

```shell
$ git clone <git url>
$ cd kubeblog/Final
$ mvn clean package
```

构建 Docker 镜像

```shell
$ docker build -t kubeblog:1.0 .
```

查看 docker 镜像

```sh
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
kubeblog            1.0                 f9bb30633155        4 minutes ago       148MB
```

Docker run --link运行博客应用

```shell
$ docker run --name kubeblog -d -p 5000:5000 --link mysql57 kubeblog:1.0
# docker run --name kubeblog -d -p 5000:5000 --network test-network kubeblog:1.0
```

进入容器查看环境变量 evn

```shell
$ docker exec -it kubeblog sh
env | grep MYSQL
MYSQL57_ENV_MYSQL_MAJOR=5.7
MYSQL57_PORT_3306_TCP_ADDR=172.17.0.2
MYSQL57_ENV_MYSQL_ROOT_PASSWORD=password
MYSQL57_ENV_GOSU_VERSION=1.12
MYSQL57_PORT_3306_TCP_PORT=3306
MYSQL57_PORT_3306_TCP_PROTO=tcp
MYSQL57_PORT_33060_TCP_ADDR=172.17.0.2
MYSQL57_PORT=tcp://172.17.0.2:3306
MYSQL57_PORT_3306_TCP=tcp://172.17.0.2:3306
MYSQL57_PORT_33060_TCP_PORT=33060
MYSQL57_ENV_MYSQL_VERSION=5.7.30-1debian10
MYSQL57_PORT_33060_TCP_PROTO=tcp
MYSQL57_NAME=/kubeblog/mysql57
MYSQL57_PORT_33060_TCP=tcp://172.17.0.2:33060
```

查看应用容器的/etc/hosts文件，可以看到`docker run --link`所做的host改动：

```shell
$ cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.17.0.2	mysql57 401b104b930a
172.17.0.3	2028007380c4
```

# 基础及集群搭建

Kubernetes 的最初目标是为应用的容器化编排部署提供一个最小化的平台，包含几个基本功能：

1. 将应用水平扩容到多个集群
2. 为扩容的实例提供负载均衡的策略
3. 提供基本的健康检查和自愈能力
4. 实现任务的统一调度  

## 架构和组件

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/kubernetes%E6%9E%B6%E6%9E%84.png" alt="kubernetes架构" style="zoom:70%;" />

### 主控制节点组件
主控制节点组件对集群做出全局决策(比如调度)，以及检测和响应集群事件（例如，当不满足部署的replicas 字段时，启动新的 pod）

主控制节点组件可以在集群中的任何节点上运行。 然而，为了简单起见，设置脚本通常会在同一个计算机上启动所有主控制节点组件，并且不会在此计算机上运行用户容器。

- apiserver

  主节点上负责提供 Kubernetes API 服务的组件；它是 Kubernetes 控制面的前端组件。

- etcd

  etcd 是兼具一致性和高可用性的键值数据库，可以作为保存 Kubernetes 所有集群数据的后台数据库。

- kube-scheduler

  主节点上的组件，该组件监视那些新创建的未指定运行节点的 Pod，并选择节点让 Pod 在上面运行。

  调度决策考虑的因素包括单个 Pod 和 Pod 集合的资源需求、硬件/软件/策略约束、亲和性和反亲和性规范、数据位置、工作负载间的干扰和最后时限。

- kube-controller-manager

  在主节点上运行控制器的组件。从逻辑上讲，每个控制器都是一个单独的进程，但是为了降低复杂性，它们都被编译到同一个可执行
  文件，并在一个进程中运行。这些控制器包括:

      1. 节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应。
      2. 副本控制器（Replication Controller）: 负责为系统中的每个副本控制器对象维护正确数量的 Pod。
      3. 终端控制器（Endpoints Controller）: 填充终端(Endpoints)对象(即加入 Service 与 Pod)。
      4. 服务帐户和令牌控制器（Service Account & Token Controllers），为新的命名空间创建默认帐户和 API 访问令牌。

### 从节点组件
节点组件在每个节点上运行，维护运行的 Pod 并提供 Kubernetes 运行环境。

- kubelet 

  一个在集群中每个节点上运行的代理。它保证容器都运行在 Pod 中。

  kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。kubelet 不会管理不是由 Kubernetes 创建的容器。

- kube-proxy

  kube-proxy 是集群中每个节点上运行的网络代理,实现 Kubernetes Service 概念的一部分。

  kube-proxy 维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。

- 容器运行时（Container Runtime）

  容器运行环境是负责运行容器的软件。

  Kubernetes 支持多个容器运行环境: Docker、 containerd、cri-o、 rktlet 以及任何实现 Kubernetes CRI (容器运行环境接口)。

### 插件（Addons）

- DNS

  尽管其他插件都并非严格意义上的必需组件，但几乎所有 Kubernetes 集群都应该有集群 DNS， 因为很多示例都需要 DNS 服务。

- Web 界面（仪表盘）

  Dashboard 是K ubernetes 集群的通用的、基于 Web 的用户界面。 它使用户可以管理集群中运行的应用程序以及集群本身并进行故障排除。

- 容器资源监控

  容器资源监控 将关于容器的一些常见的时间序列度量值保存到一个集中的数据库中，并提供用于浏览这些数据的界面。

- 集群层面日志

  集群层面日志 机制负责将容器的日志数据 保存到一个集中的日志存储中，该存储能够提供搜索和浏览接口。

## 部署方案

- 在所有节点上安装Docker和kubeadm，kubelet
- 部署容器网络插件flannel

安装kubelet、kubeadm、kubectl  ：

```shell
$ yum install -y kubelet-1.19.3 kubeadm-1.19.3 kubectl-1.19.3

# 配置系统自动启动 kubelet 
$ systemctl enable kubelet
```

之后制作虚拟机备份 Snapshot，Clone 当前虚拟机，后续 worker node 的初始化可以基于这个 snapshot进行安装。

## 初始化 Master 节点

```shell
# 设置主机名，管理节点设置主机名为master
$ hostnamectl set-hostname master

# 初始化主节点
$ kubeadm init --kubernetes-version=1.19.2 \
--apiserver-advertise-address=192.168.56.101 \  # 配置为master节点IP
--image-repository registry.aliyuncs.com/google_containers \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16

# 如果启动失败提升：kubelet 没有启动，则可以 `systemctl enable kubelet && systemctl start kubelet` 启动 kubelet ，再执行kubeadm reset， init

# 按照提示配置kube
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 配置KUBECONFIG 环境变量
$ echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile

# 安装网络插件 Flannel
# 可以先拉取镜像
# $ docker pull wangqingjiewa/flannel:v0.13.0
# 也可修改flannel.yaml 中image为：registry.cnbeijing.aliyuncs.com/qingfeng666/flannel:v0.13.0
# 使用项目中的yaml文件创建 flannel pod
$ kubectl apply -f kubeblog/docs/Chapter4/flannel.yaml

# 查看是否成功创建flannel网络
$ ifconfig |grep flannel
flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST> mtu 1450

# 等待大约 10 分钟，检查Kubernetes master运行情况
$ kubectl get node
```

## 安装配置 Work 节点

Clone Master 节点，注意 clone snapshot 虚拟机时，选择'Generate new MAC address'。选择Linked 克隆，节省存储空间。  之后启动 work1 节点：

```shell
# 设置 静态ip 地址为 192.168.56.102
$ vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
修改 IPADDR=192.168.56.102

# 重启网络
$ systemctl restart network

# 配置域名
$ hostnamectl set-hostname node1

# 配置端口转发
$ echo 1 > /proc/sys/net/ipv4/ip_forward
$ echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables

# 清理master环境的网络
$ kubeadm reset
$ systemctl stop kubelet
$ systemctl stop docker
$ rm -rf /var/lib/cni
$ rm -rf /var/lib/kubelet/*
$ rm -rf /ect/cni
$ ifconfig cni0 down
$ ifconfig flannel.1 down
$ ifconfig docker0 down
$ ip link delete cni0
$ ip link delete flannel.1
$ systemctl start docker
$ systemctl start kubelet

# 将 master 节点的 admin.conf 拷贝到 node1,在 master 机器上执行：
$ scp /etc/kubernetes/admin.conf root@node1:/etc/kubernetes/

# 配置Worker1 Kubeconfig 环境变量
$ echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
$ source ~/.bash_profile

# 执行kubeadm join
$ kubeadm join 192.168.56.101:6443 --token av4d44.g3qp0llcmo7j5eck \
    --discovery-token-ca-cert-hash sha256:85bf0d4e69ebf1ae07c828253fe1cbe15c7e6d60b9cea6c70c36faafeaa7ccd1

```

默认token的有效期为24小时，当过期之后，该token就不可用了。需要重新生成新的token，在master端执行：

```shell
$ kubeadm token create --print-join-command
```

## 安装错误排查

- 排查kubelet组件是否启动：

  ```shell
  $ systemctl status kubelet
  
  # 若未启动
  $ systemctl start kubelet
  $ systemctl daemon-reload
  ```

- 如果节点状态是 NotReady ，查看pod状态：

  ```shell
  $ kubectl get pod -n kube-system
  ```

  如果有 pod 处于 pending 状态，说明是在下载镜像，等待即可；如果状态是 fail，则很可能是镜像拉取失败，此时需要手动拉取

  ```shell
  # 查看具体pod信息，是否启动成功
  $ kubectl describe pod <NAME> -n kube-system
  
  # 用journalctl查看日志
  $ journalctl -f -u kubelet.service
  $ journalctl -u kube-apiserver
  
  # 如果日志提示：[failed to find plugin "flannel" in path [/opt/cni/bin]]
  # 下载容器插件：https://github.com/containernetworking/plugins/releases/tag/v0.8.6
  $ tar zxvf cni-plugins-linux-amd64-v0.8.6.tgz
  $ cp flannel /opt/cni/bin/
  ```

- 如果提示 ifconfig not found, 则需要在 centos 系统里安装 net-tools  

  ```shell
  $ yum -y install net-tools
  ```

- 如果出现 node ‘master’ not found 错误

  ```shell
  # 检查权限
  $ chmod 777 /run/flannel/subnet.env
  ```

- 执行 kubeadm join 异常问题

  执行 kubeadm reset，并且将 cni0，flannel1.1，docker0等网络规则删除，参考 Worker node 安装

- 镜像拉取不下来

  修改 yaml 文件中的 image路径，改成阿里云镜像地址  

## 剖析Kubeadm安装原理

`kubeadm init` 命令通过执行下列步骤来启动一个 Kubernetes Control Plane节点：

1. 运行一系列的预检项来验证系统状态：一些检查项目仅仅触发警告， 其它的则会被视为错误并且退出 kubeadm，除非问题得到解决或者用户指定了 --ignore-preflight-errors= 参数
2. 生成一个自签名的 CA 证书 (或者使用现有的证书，如果提供的话) 来为集群中的每一个组件建立身份标识。如果用户已经通过 --cert-dir 配置的证书目录（默认为 /etc/kubernetes/pki）提供了他们自己的 CA 证书以及/或者密钥，那么将会跳过这个步骤，正如文档使用自定义证书所述
3. 将 kubeconfig 文件写入 /etc/kubernetes/ 目录，以便 kubelet、控制器管理器和调度器用来连接到 API 服务器，它们每一个都有自己的身份标识，同时生成一个名为 `admin.conf` 的独立的 kubeconfig文件，用于管理操作
4. 为 API 服务器、控制器管理器和调度器生成静态 Pod 的清单文件。静态 Pod 的清单文件被写入到`/etc/kubernetes/manifests` 目录; kubelet 会监视这个目录以便在系统启动的时候创建 Pod。一旦Control Plane的 Pod 都运行起来， kubeadm init 的工作流程就继续往下执行  
5. 对Control Plane节点应用 labels 和 taints 标记，以便不会在它上面运行其它的工作负载 
6. 生成令牌以便其它节点以后可以使用这个令牌向Control Plane节点注册自己  
7. Kubeadm 会创建 configmap，提供添加节点所需要的信息  

## 安装 Dashboard

获取Dashboard yaml文件

修改Service类型为`NodePort`：

```yaml
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 31111
  selector:
  	k8s-app: kubernetes-dashboard
```

部署Dashboard ：

```shell
$ kubectl apply -f kubernetes-dashboard.yaml
```

查看pod，svc状态：

```shell
$ kubectl get pod,svc -n kubernetes-dashboard
```

通过chrome浏览器访问https://192.168.56.102:31111, 此时会返回”Your connection is notprivate“,无法跳转页面到 Dashboard，这是由于 chrome 的安全设置，解决办法：鼠标点击当前页面，直接键盘输入‘thisisunsafe’，然后回车，即可进行页面跳转至 Dashboard登录页  

获取登录 token ：

```shell
$ kubectl -n kube-system describe $(kubectl -n kube-system get secret -n kube-system -o name | grep namespace) | grep token
```

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

# 控制器





# 持久化





# 监控





