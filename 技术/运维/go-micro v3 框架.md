## go-micro v3 框架

组件：

- 注册（Registry）：提供了服务注册发现机制
- 选择器（Selector）：能够实现负载均衡
- 传输（Transport）：服务与服务之间通信接口

- 代理（Broker）：提供异步通信的消息发布/订阅接口
- 编码（Codec）：消息传输到两端时进行编码和解码
- Server 服务端、Client 客户端

技术栈使用：

- 注册中心 & 配置中心

  ```shell
  # 获取consul包
  $ go get github.com/asim/go-micro/plugins/registry/consul/v3
  
  # 获取config包
  $ go get github.com/asim/go-micro/v3/config
  $ go get github.com/asim/go-micro/plugins/config/source/consul/v3
  ```

- 链路追踪

  ```shell
  $ go get github.com/uber/jaeger-client-go
  $ go get github.com/asim/go-micro/plugins/wrapper/trace/opentracing/v3
  ```

- 熔断

  ```shell
  # 熔断
  $ go get github.com/afex/hystrix-go/hystrix
  ```

- 限流

  ```shell
  # 限流
  $ go get github.com/asim/go-micro/plugins/wrapper/ratelimiter/uber/v3
  ```

- 日志中心

- 监控

通过 docker-compose 搭建依赖环境：

```yaml

```

```go
func main() {
	// 1. 注册中心
	consulRegistry := consul.NewRegistry(func(options *registry.Options) {
		options.Addrs = []string{
			"localhost:8500",
		}
	})

	// 创建服务
	service := micro.NewService(
		micro.Name("base"),
		micro.Version("latest"),
		micro.Registry(consulRegistry),
	)

	// 初始化服务
	service.Init()

	// 启动服务
	if err := service.Run(); err != nil {
		//输出启动失败信息
		log.Fatal(err)
	}
}
```



### gRPC和 ProtoBuf

- gRPC是一个高性能、开源、通用的 RPC 框架
- 基于 HTTP2.0协议标准设计开发
- 支持多语言，默认采用 Protocol Buffers 数据序列化协议

定义prtotobuf：

```protobuf
// 限定该文件使用的是proto3的语法
syntax = "proto3";

// 用于引入不同.proto文件中的消息类型
import "myproject/other_protos.proto";

package proto.user;

// 定义消息类型
message SearchRequest {
  // = 在这里不是赋值，是字段标识符
  // 每一个被定义在消息中的字段都会被分配给一个唯一的标量，这些标量用于标识定义在二进制消息格式中的属性，
  // 标量一旦被定义就不允许在使用过程中再次被改变，标量的值在1～15的这个范围里占一个字节编码
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
  // proto允许在定义的消息类型的时候定义枚举类型
  enum Corpus {
    // 必须有一个0作为值
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 4;
}

message SearchResponse {
  //包名.消息名
  repeated string results = 1;
}

// 定义服务
service SearchService {
  //  方法名  方法参数                 返回值
  rpc Search(SearchRequest) returns (SearchResponse);
}
```

根据`protobuf`生成go文件：

```shell
# 获取protoc镜像
$ docker pull rvolosatovs/protoc
# 执行protoc, 
# docker run --rm -v<some-path>:<some-path> -w<some-path> gaotingwang/protoc [OPTION] PROTO_FILES
# --go_out=paths=import:.
# paths参数有两个选项，分别是 import 和 source_relative，默认为 import，表示按照生成的Go代码的包的全路径去创建目录层级，source_relative 表示按照 proto源文件的目录层级去创建Go代码的目录层级，如果目录已存在则不用创建
$ docker run --rm -v ${pwd}:/app -w /app gaotingwang/protoc -I ./ --go_out=./ --micro_out=./ ./proto/user/user.proto
```



### GORM使用介绍

GORM是go语言实现数据库访问的ORM库，可以方便利用面向对象的方法对数据库数据进行CRUD

依赖库：

```shell
# 获取依赖库
$ go get github.com/jinzhu/gorm
# 依赖驱动
$ go get github.com/go-sql-driver/mysql
```

操作数据库：

```go
gorm.Open("mysql", "root:123456@/test?charset=utf8&parseTime=True&loc=Local")
```



### go micro 开发流程

1. 项目目录搭建：使用micro new 生成项目初始目录

   ```shell
   # 拉取镜像
   $ docker pull micro/micro
   # 创建初始化目录（模块名称=user）
   $ docker run --rm -v ${PWD}:/app -w /app micro/micro new user
   .
   ├── main.go
   ├── handler
   │   └── user.go
   ├── proto
   │   └── user.proto
   ├── Makefile
   ├── README.md
   ├── .gitignore
   └── go.mod
   ```

2. 编写proto并自动生成go代码

   ```shell
   # protoc 使用相关命令参数查看
   $ docker run --rm gaotingwang/protoc --help
   ```

3. 编写domain service

   ```go
   # 需要引入数据库驱动依赖
   import _ "github.com/jinzhu/gorm/dialects/mysql"
   
   # 依赖数据库操作，可以通过gorm创建数据库连接
   db, err := gorm.Open("mysql", "root:123456@/micro?charset=utf8&parseTime=True&loc=Local")
   if err != nil {
     fmt.Println(err)
   }
   defer db.Close()
   ```

4. 编写对外暴露的服务Handler，与proto中定义的接口保持一致

5. 在main文件中初始化相关实例，开启服务

### 注册中心 & 配置中心

### 链路追踪

### 熔断、限流、负载均衡

### 性能监控

### 日志中心



