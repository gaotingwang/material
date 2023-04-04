[Micro Getting Started](https://micro.dev/getting-started)

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

- 链路追踪

- 熔断

- 限流

- 日志中心

- 监控

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
$ docker run --rm -v ${pwd}:/app -w /app gaotingwang/protoc:v3 -I ./ --go_out=./ --micro_out=./ ./proto/user/user.proto
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

### Micro API 网关

启动micro api作为代理，进行后端请求转发

```shell
$ docker run --rm -p 8080:8080 ghcr.io/micro/micro api --handler=api --registry=consul --registry-address=10.64.86.100:8500 
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
   $ docker run --rm gaotingwang/protoc:v3 --help
   ```

3. 编写domain service

   ```go
   // 需要引入数据库驱动依赖
   import _ "github.com/jinzhu/gorm/dialects/mysql"
   
   // 依赖数据库操作，可以通过gorm创建数据库连接
   db, err := gorm.Open("mysql", "root:123456@/micro?charset=utf8&parseTime=True&loc=Local")
   if err != nil {
     fmt.Println(err)
   }
   defer db.Close()
   ```

4. 编写对外暴露的服务Handler，与proto中定义的接口保持一致

5. 在main文件中初始化相关实例，开启服务

### 注册中心 & 配置中心

```shell
# 启动运行consul服务
# http://localhost:8500/
$ docker run -d -p 8500:8500 consul

# 获取consul包
$ go get github.com/asim/go-micro/plugins/registry/consul/v3

# 获取config包
$ go get github.com/asim/go-micro/v3/config
$ go get github.com/asim/go-micro/plugins/config/source/consul/v3
```

```go
// 设置配置中心
func GetConsulConfig(host string, port int64, prefix string) (config.Config, error) {
	consulSource := consul.NewSource(
		// 设置配置中心地址
		consul.WithAddress(host+":"+strconv.FormatInt(port, 10)),
		// 设置前缀，不设置默认：/micro/config
		consul.WithPrefix(prefix),
		consul.StripPrefix(true),
	)
	conf, err := config.NewConfig()
	if err != nil {
		return conf, err
	}
	err = config.Load(consulSource)
	return conf, err
}

// 从配置中心获取配置
func GetMysqlFromConsul(config config.Config, path ...string) *MysqlConfig {
	mysqlConfig := &MysqlConfig{}
	config.Get(path...).Scan(mysqlConfig)
	return mysqlConfig
}

-------------
func main() {
	// 注册中心
	consulRegistry := consul.NewRegistry(func(options *registry.Options) {
		options.Addrs = []string{
			"localhost:8500",
		}
	})
    
    // 配置中心
	conf, err := common.GetConsulConfig("localhost", 8500, "/micro/config")
	if err != nil {
		fmt.Println(err)
	}
    
    // 使用
    service := micro.NewService(
		micro.Name("base"),
		micro.Version("latest"),
         // 设置服务地址和需要暴露的端口
         micro.Address("127.0.0.1:8082"),
		// 添加consul 作为注册中心
		micro.Registry(consulRegistry),
	)
    
    // 加载配置
	mysqlConfig := common.GetMysqlFromConsul(conf, "mysql")
	// 使用配置初始化数据库
    // db, err := gorm.Open("mysql", "root:123456@/micro?charset=utf8&parseTime=True&loc=Local")
	db, err := gorm.Open("mysql", mysqlConfig.User+":"+mysqlConfig.Pwd+"@/"+mysqlConfig.Database+"?charset=utf8&parseTime=True&loc=Local")
	if err != nil {
		fmt.Println(err)
	}
    
    ...
}
```

### 链路追踪

jaeger 重要组件：

1. jaeger-client（客户端库）

2. Agent（客户端代理）

3. Collector（数据收集处理）

4. Data Store（数据存储）

5. UI（数据查询和前端展示）

术语Span包含对象：

- operation name：操作名称（span name）
- start timestamp：起始时间
- finish timestamp：结束时间
- span tag：一组键值对，构成span标签集合
- span log：span 日志集合
- spanContext：span 上下文
- References：span间的关系，相关0~n个Span

```shell
# 启动运行jaeger服务
# http://localhost:16686/
$ docker run -d -p 6831:6831/udp -p 16686:16686 jaegertracing/all-in-one

# jaeger 客户端
$ go get github.com/uber/jaeger-client-go
# 微服务链路追踪
$ go get github.com/asim/go-micro/plugins/wrapper/trace/opentracing/v3
```

```go
// 创建jaeger链路追踪实例
func NewTracer(serviceName string, addr string) (opentracing.Tracer, io.Closer, error) {
	cfg := &jaegercfg.Configuration{
		ServiceName: serviceName,
		Sampler: &jaegercfg.SamplerConfig{
			Type:  jaeger.SamplerTypeConst,
			Param: 1,
		},
		Reporter: &jaegercfg.ReporterConfig{
			LogSpans:            true,
			BufferFlushInterval: 1 * time.Second,
			LocalAgentHostPort:  addr,
		},
	}
	tracer, closer, err := cfg.NewTracer()
	return tracer, closer, err
}

----
func main() {
	...
	// 添加链路追踪
	tracer, io, err := common.NewTracer("base", "localhost:6831")
	if err != nil {
		fmt.Println(err)
	}
	defer io.Close()
	opentracing.SetGlobalTracer(tracer)

	// 创建服务
	service := micro.NewService(
		...
		// 链路追踪
		micro.WrapHandler(opentracing2.NewHandlerWrapper(opentracing.GlobalTracer())),
		micro.WrapClient(opentracing2.NewClientWrapper(opentracing.GlobalTracer())),
		...
	)
	...
}
```

### 熔断、限流、负载均衡

```shell
# 安装运行hystrix看板
# http://localhost:9002/hystrix
$ docker run -d -p 9002:9002 --name hystrix-dashboard mlabouardy/hystrix-dashboard

# 客户端熔断
$ go get github.com/afex/hystrix-go/hystrix
# 服务端限流
$ go get github.com/asim/go-micro/plugins/wrapper/ratelimiter/uber/v3
# 负载均衡
$ go get github.com/asim/go-micro/plugins/wrapper/select/roundrobin/v3
```

```go
type clientWrapper struct {
	client.Client
}

// 相当于一个包装类，对Client进行包装，增强了Client功能
// 熔断逻辑
func (c *clientWrapper) Call(ctx context.Context, req client.Request, rsp interface{}, opts ...client.CallOption) error {
	// hystrix.Do() 同步API，第一个参数是command, 应该是与当前请求一一对应的一个名称，如入“GET-/test”。
	// 第三个参数传入一个函数，函数包含处理错误逻辑，当请求失败时应该返回error, hystrix会根据失败率执行熔断策略
	return hystrix.Do(req.Service()+"."+req.Endpoint(), func() error {
		//正常执行
		fmt.Println(req.Service() + "." + req.Endpoint())
		return c.Client.Call(ctx, req, rsp, opts...)
	}, func(e error) error {
		//走熔断逻辑,每个服务可不一样
		fmt.Println(req.Service() + "." + req.Endpoint() + "的熔断逻辑")
		return e
	})
}

func NewClientHystrixWrapper() client.Wrapper {
	return func(i client.Client) client.Client {
		return &clientWrapper{i}
	}
}

func main() {
	...
    
    // 添加熔断器
	hystrixStreamHandler := hystrix.NewStreamHandler()
	hystrixStreamHandler.Start()
	//启动监听程序
	go func() {
		// 通过9092端口上报hystrix，http://宿主机ip:9092/turbine/turbine.stream
		err = http.ListenAndServe(net.JoinHostPort("10.64.86.100", "9092"), hystrixStreamHandler)
		if err != nil {
			fmt.Println(err)
		}
	}()

	//服务参数设置
	srv := micro.NewService(
		...
         // 客户端添加熔断
		micro.WrapClient(NewClientHystrixWrapper()),
		// 服务端添加限流
		micro.WrapHandler(ratelimit.NewHandlerWrapper(1000)),
         // 添加负载均衡
		micro.WrapClient(roundrobin.NewClientWrapper()),
	)
    ...
}
```

### 监控

prometheus 重要组件：

- Prometheus Server：用于收集和存储时间序列数据
- Client Library：客户端库生成相应的metrics并暴露给Prometheus Server
- Push Gateway：主要用于短期的jobs
- Exporters：用于暴露已有的第三方服务的metrics给Prometheus
- AlertManager：从Prometheus server端接收到alerts，会进行去重、分组，并路由到对应的接收方式发出报警

prometheus 工作流程：

1. Prometheus Server定期从配置好的jobs/exporters/push gateway中拉取数据
2. Prometheus Server记录数据并且根据报警规则推送alert数据
3. AlertManager根据配置文件，对接收到的报警进行处理，发出告警
4. 在图形界面，可视化采集数据

prometheus 相关概念：

- metric（指标）类型
  - Counter类型：一种累加的指标，如：请求的个数，出现错误数等
  - Gauge类型：可以任意加减，如：温度，运行的goroutines个数
  - Histogram类型：可以对观察结果采样，分组及统计，如：柱状图
  - Summary类型：提供观测值的count和sum功能，如：请求持续时间
- instance：一个单独监控的目标，一般对应于一个进程
- jobs：一组同类型的instances（主要用于保证可扩展性和可靠性）

```shell
$ docker network create prometheus-network --driver bridge
# 安装prometheus http://localhost:9100/ 
$ docker run -d -p 9090:9090 --network prometheus-network -v ./prometheus.yml:/etc/prometheus/prometheus.yml bitnami/prometheus
# grafana 看板适合用于数据可视化展示，http://localhost:3000/ 
$ docker run -d -p 3000:3000 --name=grafana --network prometheus-network grafana/grafana:8.5.22

$ go get github.com/asim/go-micro/plugins/wrapper/monitoring/prometheus/v3
```

```go
func PrometheusBoot(port int) {
	// 上报监控状态
	http.Handle("/metrics", promhttp.Handler())
	// 启动web 服务
	go func() {
		err := http.ListenAndServe("0.0.0.0:"+strconv.Itoa(port), nil)
		if err != nil {
			log.Fatal("启动失败")
		}
		log.Info("监控启动,端口为：" + strconv.Itoa(port))
	}()
}

----
func main() {
	...

	// 暴露端口给prometheus采集信息
	PrometheusBoot(9093)

	//服务参数设置
	srv := micro.NewService(
		...
		// 添加监控
		micro.WrapHandler(prometheus.NewHandlerWrapper()),
	)
	...
}

```

### 日志中心

ELK ：

- Elasticsearch：分布式搜索引擎，提供搜索、分析、存储三大功能
- Logstash：主要用来日志的搜集、分析、过滤的工具
- Kibana：提供web界面，帮助汇总、分析和搜索数据
- Beats：轻量级日志收集处理工具
  - Packetbeat：收集网络流量数据
  - Metricbeat：收集系统、进程和文件系统级别的数据
  - Filebeat：收集文件数据
  - Winlogbeat：收集windows事件日志数据
  - Auditbeat：收集审计数据
  - Heartbeat：运行时间监控，收集系统运行时的的数据

FileBeat 基本组成：

- Prospector（勘测者）：负责管理Harvester并找到所有读取源
- Harvester（收割机）：负责读取单个文件内容，每个文件启动一个

```yaml
# elk 相关 docker compose
version: "3.8"

services:
  elasticsearch:
    image: elasticsearch:7.17.9
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - ./config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      ELASTIC_PASSWORD: elastic
      discovery.type: single-node
      network.publish_host: _eth0_
  logstash:
    image: logstash:7.17.9
    ports:
      - "5044:5044"
      - "5000:5000"
      - "9600:9600"
    volumes:
      - ./config/logstash.yml:/usr/share/logstash/config/logstash.yml
      - ./config/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
  kibana:
    image: kibana:7.17.9
    ports:
      - "5601:5601"
    volumes:
      - ./config/kibana.yml:/usr/share/kibana/config/kibana.yml
```



```go
// 日志工具集成
// go get go.uber.org/zap
// go get gopkg.in/natefinch/lumberjack.v2 用来拆分控制日志大小

func init() {
	//日志文件名称
	fileName := "micro.log"
	syncWriter := zapcore.AddSync(
		&lumberjack.Logger{
			Filename: fileName, //文件名称
			MaxSize:  512,      //MB
			//MaxAge:     0,
			MaxBackups: 0, //最大备份
			LocalTime:  true,
			Compress:   true, //是否启用压缩
		})
	//编码
	encoder := zap.NewProductionEncoderConfig()
	//时间格式
	encoder.EncodeTime = zapcore.ISO8601TimeEncoder
	core := zapcore.NewCore(
		// 编码器
		zapcore.NewJSONEncoder(encoder),
		syncWriter,
		//
		zap.NewAtomicLevelAt(zap.DebugLevel))

	log := zap.New(
		core,
		zap.AddCaller(),
		zap.AddCallerSkip(1))

	logger = log.Sugar()
}

func Debug(args ...interface{}) {
	logger.Debug(args)
}
...
```

filebeat收集日志：

filebeat 下载地址：https://www.elastic.co/cn/downloads/past-releases/filebeat-7-17-9

```yaml
# filebeat.yml
filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

processors:
- add_cloud_metadata: ~

output.elasticsearch:
  hosts: '${ELASTICSEARCH_HOSTS:elasticsearch:9200}'
  username: '${ELASTICSEARCH_USERNAME:}'
  password: '${ELASTICSEARCH_PASSWORD:}'
```

```shell
# 启动filebeat，注意和elk版本保持一致
$ docker pull elastic/filebeat:7.17.9
$ docker run --name=filebeat -v ${pwd}/filebeat.yml:/usr/share/filebeat/filebeat.yml elastic/filebeat:7.17.9
```

Filebeat 默认收集的是 Docker 容器启动后输出到`docker logs`的日志，但有些服务除了这些日志外，还会把其他日志存放到容器的里面。比如`egg.js`框架会把错误日志和一些 web 日志放到`/root/logs/yourserver/`下面。那么要如何收集这些额外的日志呢？

解决方法是把额外日志的路径映射到 Docker 的输出控制台，可以在 Dockerfile 里面这样设置：

```
Dockerfile# 将普通日志输出到 stdout
RUN ln -sf /dev/stdout /root/logs/web-server/web-server-web.log
# 将错误日志输出到 stderr
RUN ln -sf /dev/stderr /root/logs/web-server/common-error.log
```

这样容器启动后就会将容器里日志传输到控制台上，而 Filebeat 也就可以收集该日志了

Filebeat 官方推荐的一种收集 Docker 容器的方法，其实还有其他方案，比如说在每个容器里面安装一个 Filebeat 客户端来分别收集各自服务的日志，或者是把每个 Docker 容器的日志路径都映射到宿主机上的某个目录下面，然后在宿主机上安装一个 Filebeat 客户端来统一收集。



## 开发步骤

### 1. 服务端开发

1. 创建初始化目录（模块名称=user）

   ```shell
   $ docker run --rm -v ${PWD}:/app -w /app micro/micro new user
   ```

2. 开发 domain - model

3. 开发 domain - repository

4. 开发 domain - service

5. 编写proto并自动生成go代码

   ```shell
   $ docker run --rm -v ${pwd}:/app -w /app gaotingwang/protoc:v3 -I ./ --go_out=./ --micro_out=./ ./proto/user/user.proto
   ```

6. 开发 对外暴露的服务Handler

   7. 编写 main.go 

   8. 编译

      ```sh
      # 交叉编译
      $ CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o 替换成服务名称 *.go
      
      # 编写 dockerfile, 打包 docker 镜像
      ```

   9. docker运行

      

### 2. 开发对外暴露的接口（启动API网关）

1. 初始化项目工程目录

2. 编写 API proto 文件，生成go代码

3. 编写对外暴露的 API 接口

4. 编写 main.go

5. 打包docker镜像，运行

   

### 3. 启动网关

建立api-gateway网关：

```sh
$ docker run -d -p 8080:8080 cap1573/cap-api-gateway --registry=consul --registry_address=替换成注册中心地址:8500 api --handler=api
```

