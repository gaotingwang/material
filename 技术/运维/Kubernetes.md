Kubernetes

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/kubernetes/%E6%9E%B6%E6%9E%84.jpeg" alt="架构" style="zoom:60%;" />

# 环境准备

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
  IPADDR=192.168.99.101
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
  192.168.99.101 master
  192.168.99.102 node1
  192.168.99.103 node2
  
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





# 网络实现





# 控制器





# 持久化





# 监控





