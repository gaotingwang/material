## 容器交互模式

`docker container`
`docker ps`
`docker run --rm`   Automatically remove the container when it exits
`docker start`
`docker stop`
`docker rm`
`docker exec`
`docker logs`

```sh
# 输出日志 -f 追加式日志输出
$ docker logs xx 
# 保持开一个伪终端，可进入交互模式
$docker container -it 
# 创建容器并进入命令行，命令行交互退出，容器也就停止了
$ docker run -it ubuntu sh 
# 在已运行容器执行一个额外command，命令行交互退出，仅退出交互，容器未停止
$ docker exec -it 容器ID sh 
```



## 镜像管理

`docker rmi/ docker image rm`
`docker pull / docker image pull`
`docker image inspect imageId`  查看镜像详情
构建和分享：
	`docker image build -t hello .`  根据 Dockerfile 生成镜像，-t指定镜像名称，`.` 指定执行路径在当前目录下
	`docker image tag hello gtw/hello:1.0` 给镜像生成新的镜像名称
	`docker image push gtw/hello:1.0` 推送镜像到 dockerhub，镜像名称前缀必须为自己的ID
从容器生成镜像：
	`docker [container] commit containerId hello/nginx` 根据一个容器来生成新的镜像
离线方式导出导入：
	`docker image save nginx:1.20.0 -o nginx.image` 在当前目录生成一个离线的镜像文件 （`-o output`）
	`docker image load -i .\nginx.image` 导入指定目录下的镜像文件



## Dockerfile

Dockerfile 是用于构建镜像的文件，包含了构建镜像需要的指令

```dockerfile
# 指定基础镜像
FROM ubuntu:23.04

# 切换目录，目录不存在会自动创建，后续文件操作若不指定目录，默认就在WORKDIR下
WORKDIR /app

# 文件复制，目标目录不存在会自动创建
# ADD 与 COPY 功能相似，ADD 比 COPY 高级在于：如果复制文件为gzip等压缩文件，可以自动解压缩
ADD index.html /usr/share/nginx/html/index.html

# 构建参数(只能在构建镜像时使用，可在build时动态指定，`docker build -t m/ubuntu --build-arg VERSION=2.0.0 .` )
ARG VERSION=2.0.1
# 环境变量(可以在构建镜像时使用，同时会将环境变保存在镜像中，在容器中可以继续使用)
ENV VERSIONS=2.0.0

# 执行命令(每个RUN命令都会产生image layer, 导致镜像臃肿，可将多行命令使用一个RUN来执行)
RUN apt-get update && \
	apt-get install -y wget && \
	wget https://github.com/ipinfo/cli/releases/download/ipinfo-${VERSION}/ipinfo_${VERSION}_linux_amd64.tar.gz && \
	tar zxf ipinfo_${VERSION}_linux_amd64.tar.gz && \
	mv ipinfo_${VERSION}_linux_amd64 /usr/bin/ipinfo && \
	rm -rf ipinfo_${VERSION}_linux_amd64.tar.gz

# 向外暴露端口
# EXPOSE 5000

# 启动命令, 与CMD不同，ENTRYPOINT命令肯定会被执行，不会被覆盖
# 可与CMD联合使用，ENTRYPOINT设置命令，CMD设置参数
ENTRYPOINT ["sh", "-c"]
# 设置容器启动时，默认会执行的命令（如果在 docker run 时指定了其他命令，cmd命令会被忽略）
# 定义多个CMD，只有最后一个会被执行
# 如果执行的命令要使用变量，可以通过shell脚本的形式，`sh -c echo hello $VERSIONS`
CMD ["echo hello $VERSIONS"]

```

```sh
# -f Name of the Dockerfile (Default is 'PATH/Dockerfile')
$ docker build -f dockerfile.good -t hello .
```

基础镜像选择原则：

1. 优先官方镜像，其次 Dockerfile 开源；
2. 固定tag版本；
3. 尽量选体积小的镜像

技巧：

1. 合理使用缓存，将不经常发送变化的内容往前放，会变化的内容往后放，提高镜像构建速度

2. 利用 `.dockerignore` 减少向docker daemon中发送的build content内容

3. 多阶段构建，来降低最终镜像文件的大小

   ```dockerfile
   # 选择gcc这个image去构建文件
   FROM gcc:9.4 AS builder
   COPY hello.c /src/hello.c
   WORKDIR /src
   RUN gcc --static -o hello hello.c
   
   
   # 编译完以后，并不需要这样一个大的GCC环境，一个小的alpine镜像就可以
   FROM alpine:3.13.5
   COPY --from=builder /src/hello /src/hello
   ENTRYPOINT [ "/src/hello" ]
   CMD []
   ```

4. 尽量使用非root用户

   使用非root用户来构建这个镜像，名字叫 `flask-no-root` Dockerfile 如下：

   - 通过`groupadd`和`useradd`创建一个flask的组和用户
   - 通过`USER`指定后面的命令要以flask这个用户的身份运行

   ```dockerfile
   FROM python:3.9.5-slim
   
   RUN pip install flask && \
       groupadd -r flask && useradd -r -g flask flask && \
       mkdir /src && \
       chown -R flask:flask /src
   
   USER flask
   
   COPY app.py /src/app.py
   
   WORKDIR /src
   ENV FLASK_APP=app.py
   
   EXPOSE 5000
   
   CMD ["flask", "run", "-h", "0.0.0.0"]
   ```




## Docker 存储

主要提供了两种方式做数据的持久化

### Data Volume

由Docker管理，(/var/lib/docker/volumes/ Linux)， 持久化数据的最好方式

在 Dockerfile 中通过`VOLUME` 来指定要持久化的目录

```dockerfile
# 将指定目录进行持久化，容器启动不指定-v，则持久化目录名称是随机的
# 不是必须在Dockerfile里通过VOLUME定义，可以在容器启动时通过-v指定
VOLUME ["/app"]
```

在容器启动时，可以指定volume到的名称

```sh
# -v cron-data:/app 指定宿主机的/var/lib/docker/volumes/cron-data/_data 目录来存储容器中/app下的内容
$ docker run -d -v cron-data:/app my-cron
# 查看volume列表
$ docker volume ls
# 查看具体的volume详情
$ docker volume inspect xxxxxx
```

### Bind Mount

由用户指定存储的数据具体mount在系统什么位置

```sh
# 在容器启动时候-v也可以直接指定目录来存储容器目录中的内容，linux和mac使用$(pwd)
$ docker run -d -v ${pwd}:/app my-cron
```

此时通过`docker volume ls`命令是查看不到对应的volume信息，只能到用户指定目录下去看

### 多机器间的数据共享

Docker的volume支持多种driver，默认创建的volume driver都是local

```sh
$ docker volume inspect vscode
[
    {
        "CreatedAt": "2021-06-23T21:33:57Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/vscode/_data",
        "Name": "vscode",
        "Options": null,
        "Scope": "local"
    }
]
```

1. 以sshfs的driver举例，安装plugin：`docker plugin install --grant-all-permissions vieux/sshfs`

2. 创建volume：

   ```sh
   $ docker volume create --driver vieux/sshfs \
                             -o sshcmd=vagrant@192.168.200.12:/home/vagrant \
                             -o password=vagrant \
                             sshvolume
   $ docker volume ls
   DRIVER               VOLUME NAME
   vieux/sshfs:latest   sshvolume
   $ docker volume inspect sshvolume
   [
       {
           "CreatedAt": "0001-01-01T00:00:00Z",
           "Driver": "vieux/sshfs:latest",
           "Labels": {},
           "Mountpoint": "/mnt/volumes/f59e848643f73d73a21b881486d55b33",
           "Name": "sshvolume",
           "Options": {
               "password": "vagrant",
               "sshcmd": "vagrant@192.168.200.12:/home/vagrant"
           },
           "Scope": "local"
       }
   ]
   ```

3. 容器挂载volume：`docker run -it -v sshvolume:/app busybox sh`



## Docker 网络

常用命令：

- ip地址查看：`ifconfig`
- 网路连通性：`ping 192.168.178.1`
- 端口的连通性：`telnet www.baidu.com 80`
- 路径探测跟踪：`tracepath www.baidu.com`
- 请求web服务：`curl www.baidu.com`

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/docker/docker-network.png" alt="docker-network" style="zoom:70%;" />

```sh
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
1847e179a316   bridge    bridge    local
a647a4ad0b4f   host      host      local
fbd81b56c009   none      null      local
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "1847e179a316ee5219c951c2c21cf2c787d431d1ffb3ef621b8f0d1edd197b24",
        "Created": "2021-07-01T15:28:09.265408946Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            # docker0 network 相当于网关地址
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        # 容器网络信息
        "Containers": {
            "03494b034694982fa085cc4052b6c7b8b9c046f9d5f85f30e3a9e716fad20741": {
                "Name": "box1",
                "EndpointID": "072160448becebb7c9c333dce9bbdf7601a92b1d3e7a5820b8b35976cf4fd6ff",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "4f3303c84e5391ea37db664fd08683b01decdadae636aaa1bfd7bb9669cbd8de": {
                "Name": "box2",
                "EndpointID": "4cf0f635d4273066acd3075ec775e6fa405034f94b88c1bcacdaae847612f2c5",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

自定义bridge：`docker network create -d bridge mybridge`

容器启动时`--network`使用指定的network：`docker run -d -e MYSQL_ROOT_PASSWORD=root --network mybridge -p 3307:3306 --name mysql-network mysql:5.7`

给指定容器绑定网络：`docker network connect mybridge some-mysql` 取消使用`disconnect`



## Docker Compose

Compose 用于定义和运行多容器 Docker 应用程序的工具。通过 Compose，可以使用 YML 文件来配置应用程序需要的所有服务。然后，使用一个命令，就可以从 YML 文件配置中创建并启动所有服务。

### 安装

Windows和Mac在默认安装了docker desktop以后，docker-compose随之自动安装

```powershell
PS C:\Users\TW\docker.tips> docker-compose --version
Docker Compose version v2.15.1
```

Linux用户需要自行安装，最新版本号可以在这里查询 [Docker Compose 版本号](https://github.com/docker/compose/releases)

```sh
$ sudo curl -L "https://github.com/docker/compose/releases/download/2.15.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
$ docker-compose --version
docker-compose version 1.29.2, build 5becea4c
```

### 文件结构

```yaml
version: "3.8"

# 定义容器
services:
  # 容器1，服务名字，这个名字也是内部 bridge网络可以使用的 DNS name
  frontend:
    image: awesome/webapp # 镜像的名字
    ports: # 可选，相当于 docker run里的 -p
      - "443:8043"
    networks: # 可选，相当于 docker run里的 --network
      - front-tier
      - back-tier
    configs:
      - httpd-config
    secrets:
      - server-certificate
    environment: # 可选，相当于 docker run里的 --env
      - COMPOSE_PROJECT_NAME
    command: echo "I'm running ${COMPOSE_PROJECT_NAME}" # 可选，如果设置，则会覆盖默认镜像里的 CMD命令
    volumes: # 可选，相当于docker run里的 -v 
  # 容器2
  flask:
  	build: # 相当于docker build 指定build content目录，及Dockerfile文件名
      context: ./flask
      dockerfile: Dockerfile
    image: flask-demo:latest
    depends_on:
      - redis-server
    volumes:
      - db-data:/etc/data
    networks:
      - back-tier
  # 容器3
  redis-server:
    image: redis:latest
    command: redis-server --requirepass ${REDIS_PASSWORD}
    networks:
      - back-tier

# 可选，相当于 docker volume create
volumes:
  db-data:
    driver: flocker
    driver_opts:
      size: "10GiB"

configs:
  httpd-config:
    external: true

secrets:
  server-certificate:
    external: true

# 可选，相当于 docker network create
networks:
  front-tier: {}
  back-tier: {}

```

### 使用

Compose 使用的三个步骤：

- 使用 Dockerfile 定义应用程序的环境。
- 使用 `docker-compose.yml` 定义构成应用程序的服务，这样它们可以在隔离环境中一起运行。
- 最后，执行 `docker-compose up` 命令来启动并运行整个应用程序。（`docker-compose`执行，需要在`docker-compose.yml` 所在文件夹执行）

 `docker-compose build` 可以根据Dockfile来构建镜像，`docker-compose pull` 可以拉取镜像和构建镜像

`docker-compose up -d` 可以进行后台运行， `docker-compose logs -f` 可以持续动态的查看日志

`docker-compose ps` 查看应用进程，`docker-compose stop` 停止所有应用

上述文件里的`${REDIS_PASSWORD}` 变量，可在 `docker-compose.yml` 同目录下创建`.env`文件来配置：

```properties
REDIS_PASSWORD=123
```

也可手动指定变量的配置文件，在每次命令使用时，通过`docker-compose --env-file .\myenv config`这种方式指定配置文件（`docker-compose` 都需要带着`--env-file`参数）

### 服务更新

1. 修改镜像文件，需要重新build镜像，可以通过`docker-compose up -d --build`
2. 移除没有定义在Compose中的service `docker-compose up -d --remove-orphans`
3. 容器重启，`docker-compose restart`

### 水平扩展

`docker-compose up -d --scale flask=3` 通过`--scale`参数，可以实现对flask服务的扩缩容，同时支持负载均衡

### 健康检查

容器本身有一个健康检查的功能，但是需要在Dockerfile里通过`HEALTHCHECK`关键字来定义，或者在执行docker container run 的时候，通过下面的一些参数指定

```sh
--health-cmd string              Command to run to check health
--health-interval duration       Time between running the check
                                (ms|s|m|h) (default 0s)
--health-retries int             Consecutive failures needed to
                                report unhealthy
--health-start-period duration   Start period for the container to
                                initialize before starting
                                health-retries countdown
                                (ms|s|m|h) (default 0s)
--health-timeout duration        Maximum time to allow one check to
```

Docker Compose通过`healthcheck`来监测容器状态，`depends_on.condition` 来监测依赖服务是否启动成功

```yaml
version: "3.8"

services:
  php:
    tty: true
    build:
      context: .
      dockerfile: tests/Docker/Dockerfile-PHP
      args:
        version: cli
    volumes:
      - ./src:/var/www/src
      - ./tests:/var/www/tests
      - ./build:/var/www/build
      - ./phpunit.xml.dist:/var/www/phpunit.xml.dist
    depends_on:
      mysql:
      	# 依赖服务健康状态
        condition: service_healthy
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
  mysql:
    image: mysql
    environment:
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
      - MYSQL_ROOT_PASSWORD=
      - MYSQL_DATABASE=cache
    # 健康检查
    healthcheck:
      test: ["CMD", "mysql" ,"-h", "mysql", "-P", "3306", "-u", "root", "-e", "SELECT 1", "cache"]
      interval: 1s
      timeout: 3s
      retries: 30
  postgresql:
    image: postgres
    environment:
      - POSTGRES_PASSWORD=
      - POSTGRES_DB=cache
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 1s
      timeout: 3s
      retries: 30
  redis:
    image: redis
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 1s
      timeout: 3s
      retries: 30
```



## Docker Swarm

Docker Compose只适用在单机情况下使用，在生产环境中，需要考虑跨机器做scale横向扩展，容器失败退出时如何新建容器确保服务正常运行，需要管理密码、Key等敏感数据等问题，Docker Swarm就是用于做容器编排管理的。

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/docker/docker-swarm.png" style="zoom:67%;" />

docker engine默认没有激活swarm模式，可以通过`docker swarm init` 来激活，`docker node ls` 可以查看集群节点信息，`docker swarm leave --force`来退出swarm模式。

To add a worker to this swarm, run the following command:

```sh
$ docker swarm join --token SWMTKN-1-3y26cfwg7ccslfk5ijktbsowdm3issum8r1yeim6lmlk6kqd9d-8c2h0w50napk2oucwxv3y0zvw 192.168.65.4:2377
```

当执行`docker swarm init` 主要是PKI和安全相关的自动化：

- 创建swarm集群的根证书
- manager节点的证书
- 其它节点加入集群需要的tokens
- 创建Raft数据库用于存储证书，配置，密码等数据

### service 操作

```sh
# 创建服务（进行镜像拉取，容器创建）, 须注意docker service 不能进行镜像的构建，只能拉取
$ docker service create --name web --replicas 2 nginx

# 设置服务的实例个数
$ docker service update web --replicas 3 (同 docker service scale web=3)

# 查看现有服务
$ docker service ls
ID             NAME      MODE         REPLICAS   IMAGE          PORTS
oivpfwcvtnvh   web       replicated   1/1        nginx:latest

# 查看具体服务下的各实例信息
$ docker service ps oivp
ID             NAME      IMAGE          NODE             DESIRED STATE   CURRENT STATE            ERROR     PORTS
9ntgshptwhqi   web.1     nginx:latest   docker-desktop   Running         Running 8 minutes ago
fow0yj6s2squ   web.2     nginx:latest   docker-desktop   Running         Running 37 seconds ago
xat3lo9v7t7a   web.3     nginx:latest   docker-desktop   Running         Running 37 seconds ago


# 删除service
$ docker service rm web

```

### overlay 网络

在swarm集群里的服务，如何对外进行访问，这部分分为两块:

- 第一，`东西向流量` ，也就是不同swarm节点上的容器之间如何通信（不同容器可能在不同机器上），swarm通过 `overlay` 网络来解决；
- 第二，`南北向流量` ，也就是swarm集群里的容器如何对外访问，比如互联网，这个是 `Linux bridge + iptables NAT` 来解决的

```sh
# 创建 overlay 网络
$ docker network create -d overlay mynet

# 服务连接到这个 overlay网络
$ docker service create --network mynet --name test --replicas 2 busybox ping 8.8.8.8

$ docker container ls
$ docker container exec -it cac sh
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    valid_lft forever preferred_lft forever
24: eth0@if25: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue
    link/ether 02:42:0a:00:01:08 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.8/24 brd 10.0.1.255 scope global eth0
    valid_lft forever preferred_lft forever
26: eth1@if27: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth1
    valid_lft forever preferred_lft forever
/ #
```

这个容器有两个接口 eth0和eth1， 其中eth0是连到了mynet这个网络，eth1是连到docker_gwbridge这个网络

### ingress 网络

外部如何访问部署运行在swarm集群内的服务，在swarm里通过 `ingress` 来解决。docker swarm的ingress网络又叫 `Ingress Routing Mesh`，主要是为了实现把service的服务端口对外发布出去，让其能够被外部网络访问到。

```sh
$ docker service create --name web --network mynet -p 8080:80 --replicas 2 containous/whoami

$ docker service ls
ID             NAME      MODE         REPLICAS   IMAGE                      PORTS
a9cn3p0ovg5j   web       replicated   2/2        containous/whoami:latest   *:8080->80/tcp

$ curl 192.168.200.10:8080
$ curl 192.168.200.11:8080
$ curl 192.168.200.12:8080
```

会在每个swarm节点的IP加端口8080，每个节点IP:8080都可以访问，并且回应的容器是不同的（hostname），也就是有负载均衡的效果

### stack部署多service应用

1. 准备工作

   ```sh
   # 使用compose构建镜像
   $ docker-compose build
   
   # 提交镜像到dockerhub
   $ docker login
   $ docker-compose push
   ```

2. 通过stack启动服务

   ```yaml
   version: "3.9"
   
   services:
     vote:
       image: dockersamples/examplevotingapp_vote
       ports:
         - 5000:80
       networks:
         - frontend
       # 指定deploy 实例的个数
       deploy:
         replicas: 2
         
     worker:
       image: dockersamples/examplevotingapp_worker
       networks:
         - frontend
         - backend
       deploy:
         placement:
         	constraints: [node.role == manager] # 部署在manager节点
   ```

   

   ```sh
   # 使用compose文件开始部署
   $ docker stack deploy --compose-file docker-compose.yml flask-demo
   
   # 查看stack列表
   $ docker stack ls
   NAME         SERVICES   ORCHESTRATOR
   flask-demo   2          Swarm
   
   # 查看具体stack内容
   $ docker stack ps flask-demo
   ID             NAME                        IMAGE                            NODE            DESIRED STATE   CURRENT STATE
   ERROR     PORTS
   lzm6i9inoa8e   flask-demo_flask.1          xiaopeng163/flask-redis:latest   swarm-manager   Running         Running 23 seconds ago
   ejojb0o5lbu0   flask-demo_redis-server.1   redis:latest                     swarm-worker2   Running         Running 21 seconds ago
   
   # 查看具体stack对应的service列表
   $ docker stack services flask-demo
   ID             NAME                      MODE         REPLICAS   IMAGE                            PORTS
   mpx75z1rrlwn   flask-demo_flask          replicated   1/1        xiaopeng163/flask-redis:latest   *:8080->5000/tcp
   z85n16zsldr1   flask-demo_redis-server   replicated   1/1        redis:latest
   
   $ docker service ls
   ID             NAME                      MODE         REPLICAS   IMAGE                            PORTS
   mpx75z1rrlwn   flask-demo_flask          replicated   1/1        xiaopeng163/flask-redis:latest   *:8080->5000/tcp
   z85n16zsldr1   flask-demo_redis-server   replicated   1/1        redis:latest
   
   $ curl 127.0.0.1:8080
   Hello Container World! I have been seen 1 times and my hostname is 21d63a8bfb57.
   ```

### 使用 secret

1. 创建

   ```sh
   # 从标准的输入读取
   $ echo hello | docker secret create mysql_pass -
   
   # 从文件读取
   $ echo world > pass.txt
   $ docker secret create pass_word pass.txt
   
   # 查看secret
   $ docker secret ls
   ID                          NAME         DRIVER    CREATED          UPDATED
   px38wqko11jqhfwbpxt0ojw8g   mysql_pass             3 minutes ago    3 minutes ago
   zzpk7lhy5ayc16eto976ja4m9   pass_word              15 seconds ago   15 seconds ago
   $ docker secret inspect mysql_pass
   ```

2. 使用

   ```sh
   # 通过--secret 会将mysql_pass存放在容器/run/secrets/mysql_pass文件中
   $ docker service create --name mysql-demo --secret mysql_pass --env MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mysql_pass mysql:5.7
   ```

3. 在Compose中使用

   ```yaml
   version: "3.9"
   
   services:
      db:
        image: mysql:latest
        environment:
          MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
          MYSQL_DATABASE: wordpress
          MYSQL_USER: wordpress
          MYSQL_PASSWORD_FILE: /run/secrets/db_password
        # 导入secret到/run/secrets目录下的对应文件中
        secrets:
          - db_root_password
          - db_password
   
   # 会创建 secrets
   secrets:
      db_password:
        file: db_password.txt
      db_root_password:
        file: db_root_password.txt
   ```

### 使用 local volume

```yaml
version: "3.8"

services:
  db:
    image: mysql:5.7
    environment:
      - MYSQL_ROOT_PASSWORD_FILE=/run/secrets/mysql_pass
    secrets:
      - mysql_pass
    volumes:
      # 将data volume挂载到本地/var/lib/mysql目录下
      - data:/var/lib/mysql

# volume create name=data
volumes:
  data:

secrets:
  mysql_pass:
    file: mysql_pass.txt
```



## 多架构支持

```sh
$ docker buildx create --name mybuilder --use

$ docker buildx ls

# 会在一个容器内进行build, 构建完成 docker images 并不会查到创建好的镜像
# --push 会直接push到dochub
$ docker buildx build --push --platform linux/arm/v7,linux/arm64/v8,linux/amd64 -t gaotingwang/flask-redis:latest .
```



## CI / CD

持续集成，可利用Git Actions进行，参考文档[Actions](https://docs.github.com/zh/actions)



## Docker 小技巧

批量操作: 

```sh
$ docker rm $(docker ps -aq)
# 删除所有已退出的容器
$ docker system prune -f 
# 删除未使用的镜像
$ docker image prune -a
# volume 清除
$ docker volume prune -f
```



## 私有仓库搭建

### 容器镜像中心

- 官方Dockerhub
- 其他镜像仓库，如阿里云镜像仓库

```shell
# 配置国内 Docker 镜像源
$ vi /etc/docker/daemon.json
{
    "registry-mirrors": ["https://registry.docker-cn.com"],
    "live-restore": true
}

# 重启 docker 服务
$ systemctl restart docker
```

### 搭建私有镜像中心

#### 安装 JFrog Container Registry(JCR) 

```shell
# 在master节点执行
# 增加 JFROG_HOME 环境变量
$ vi ~/.bash_profile
export JFROG_HOME=/etc
$ source ~/.bash_profile

# 创建挂载目录
$ mkdir -p $JFROG_HOME/artifactory/var/etc/
$ cd $JFROG_HOME/artifactory/var/etc/
$ touch ./system.yaml
$ chown -R 1030:1030 $JFROG_HOME/artifactory/var
$ chmod -R 777 $JFROG_HOME/artifactory/var

# 启动 JCR
$ docker run --name artifactory-jcr -v $JFROG_HOME/artifactory/var/:/var/opt/jfrog/artifactory -d -p 8081:8081 -p 8082:8082 releases-docker.jfrog.io/jfrog/artifactory-jcr:latest

# 访问 192.168.56.101:8081（master节点ip） 可以看到初始化页面
```

#### 配置私有镜像中心

```shell
# 在 Master，node 节点上分别配置 JCR 的本地域名解析
$ vi /etc/hosts
192.168.56.101 master art.local
192.168.56.102 node1
192.168.56.103 node2

# 配置Docker insecure registry
$ vi /etc/docker/daemon.json
{
    "registry-mirrors":[
        "https://registry.docker-cn.com",
        "https://dockerhub.azk8s.cn",
        "https://reg-mirror.qiniu.com",
        "http://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn"
    ],
    "insecure-registries":[
        "art.local:8081"
    ]
}

$ systemctl restart docker

# 访问地址 http://192.168.56.101:8081 在页面上初始化 JCR，设置登录账号密码

# 执行 docker login
# docker login art.local:8081
```

#### 配置 JCR 仓库

登录 JCR 并创建 5 个仓库：docker-local, docker-test, docker-release, docker-remote 和 docker virtual 仓库

<img src="https://gtw.oss-cn-shanghai.aliyuncs.com/DevOps/docker/JCR%E4%BB%93%E5%BA%93.png" style="zoom:60%;" />

#### 镜像推送到私有镜像中心

```shell
# 登录镜像中心，并上传镜像
$ docker login art.local:8081 admin/password
$ docker tag registry.cn-beijing.aliyuncs.com/qingfeng666/kubeblog:1.0 art.local:8081/docker-local/kubeblog:1.0
$ docker push art.local:8081/docker-local/kubeblog:1.0

# 登录 JCR 查看推送的镜像
```

