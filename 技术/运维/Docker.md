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

