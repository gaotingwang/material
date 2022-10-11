## 安装

```shell
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

用`java -jar`的方式启动：

```shell
java -jar arthas-boot.jar

[INFO] Found existing java process, please choose one and hit RETURN.
* [1]: 79952 cn.test.MobileApplication
  [2]: 93872 org.jetbrains.jps.cmdline.Launcher
```

然后输入数字，选择想要监听的应用，回车即可

默认情况下， arthas server侦听的是 `127.0.0.1` 这个IP，如果希望远程可以访问，可以使用`--target-ip`的参数：

```shell
java -jar arthas-boot.jar --target-ip
```

## JVM查看

### dashboard 命令可以查看当前系统的实时数据面板

```shell
[arthas@163]$ dashboard
ID  NAME                     GROUP        PRIORIT STATE   %CPU     DELTA_T TIME    INTERRUP DAEMON  
-1  C1 CompilerThread0       -            -1      -       0.0      0.000   0:0.833 false    true    
-1  C2 CompilerThread0       -            -1      -       0.0      0.000   0:0.801 false    true    
23  arthas-NettyHttpTelnetBo system       5       RUNNABL 0.0      0.000   0:0.117 false    true    
1   main                     main         5       TIMED_W 0.0      0.000   0:0.113 false    false   
-1  VM Thread                -            -1      -       0.0      0.000   0:0.095 false    true    
-1  VM Periodic Task Thread  -            -1      -       0.0      0.000   0:0.053 false    true    
11  Attach Listener          system       9       RUNNABL 0.0      0.000   0:0.036 false    true    
24  arthas-command-execute   system       5       TIMED_W 0.0      0.000   0:0.033 false    true    
16  arthas-NettyHttpTelnetBo system       5       RUNNABL 0.0      0.000   0:0.031 false    true    
-1  Sweeper thread           -            -1      -       0.0      0.000   0:0.010 false    true    
25  Timer-for-arthas-dashboa system       5       RUNNABL 0.0      0.000   0:0.003 false    true    
Memory               used    total  max    usage  GC                                                
heap                 20M     26M    177M   11.25% gc.copy.count            15                       
tenured_gen          15M     18M    122M   12.62% gc.copy.time(ms)         64                       
eden_space           3M      7M     49M    7.45%                           1                        
survivor_space       895K    896K   6272K  14.29% gc.marksweepcompact.time 15                       
nonheap              26M     30M    -1     86.43% (ms)                                              
codeheap_'non-nmetho 1M      2M     5M     21.63%                                                   
ds'                                                                                                 
Runtime                                                                                             
os.name                                           Linux                                             
os.version                                        5.4.0-126-generic                                 
java.version                                      15-ea                                             

```

注意：
	ID: Java级别的线程ID，注意这个ID不能跟jstack中的nativeID对应 

### jvm 查看当前JVM信息

`heapdump` 类似 jmap 命令的 heap dump 功能 `heapdump /tmp/dump.hprof`

`--live`只 dump live 对象:

```bash
[arthas@58205]$ heapdump --live /tmp/dump.hprof
Dumping heap to /tmp/dump.hprof...
Heap dump file created
```

### memory 查看 JVM 内存信息

监控到 JVM 的实时运行状态

- 怎样直接从 JVM 内查找某个类的实例

## 线程查看

`thread` 命令可以查询所有线程信息

| 参数名称      | 参数说明                                                |
| :------------ | :------------------------------------------------------ |
| *id*          | 线程 id                                                 |
| [n:]          | 指定最忙的前 N 个线程并打印堆栈                         |
| [b]           | 找出当前阻塞其他线程的线程                              |
| [i `<value>`] | 指定 cpu 使用率统计的采样间隔，单位为毫秒，默认值为 200 |
| [--all]       | 显示所有匹配的线程                                      |

查看线程ID 1的栈：

```shell
[arthas@86]$ thread 1 | grep 'main('
    at app//demo.MathGame.main(MathGame.java:17)
```

查看5秒内的CPU使用率top n线程栈：

```shell
[arthas@86]$ thread -n 3 -i 5000
```

查找线程是否有阻塞：

```
thread -b
```

## 线上问题排查

### jad 反编译指定类的源码

有时候，版本发布后，代码竟然没有执行，代码是最新的吗，这时可以使用jad反编译相应的class：

```shell
[arthas@86]$ jad cn.test.mobile.controller.order.OrderController
```

仅编译指定的方法：

```shell
[arthas@86]$ jad cn.test.mobile.controller.order.OrderController getOrderInfo

ClassLoader:
@RequestMapping(value={"getOrderInfo"}, method={RequestMethod.POST})
public Object getOrderInfo(HttpServletRequest request, @RequestBody Map map) {
    ResponseVo responseVo = new ResponseVo();
    ... ... ...  ...
```

### sc 查看JVM已加载的类信息

“Search-Class” 的简写，有的时候只记得类的部分关键词，可以用sc获取完整名称，当碰到“ClassNotFoundException”或者“ClassDefNotFoundException”，可以用这个命令验证下

| 参数名称         | 参数说明                                                     |
| :--------------- | :----------------------------------------------------------- |
| *class-pattern*  | 类名表达式匹配                                               |
| *method-pattern* | 方法名表达式匹配                                             |
| [d]              | 输出当前类的详细信息，包括这个类所加载的原始文件来源、类的声明、加载的ClassLoader等详细信息。如果一个类被多个ClassLoader所加载，则会出现多次 |

模糊搜索

```shell
[arthas@86]$ sc *OrderController*
cn.test.mobile.controller.order.OrderController
```

打印类的详细信息 ：`sc -d`

```shell
[arthas@86]$ sc -d cn.test.mobile.controller.order.OrderController

 class-info        cn.test.mobile.controller.order.OrderController
 code-source       /F:/IDEA-WORKSPACE-TEST-qyb/trunk/BE/mobile/target/classes/
 name              cn.test.mobile.controller.order.OrderController
 isInterface       false
 isAnnotation      false
 isEnum            false
 isAnonymousClass  false
 isArray           false
 isLocalClass      false
 isMemberClass     false
 isPrimitive       false
 isSynthetic       false
 simple-name       OrderController
 modifier          public
 annotation        org.springframework.web.bind.annotation.RestController,org.springframework.web.bind.annotation.Requ
                   estMapping
 interfaces
 super-class       +-cn.test.mobile.controller.BaseController
                     +-java.lang.Object
 class-loader      +-sun.misc.Launcher$AppClassLoader@18b4aac2
                     +-sun.misc.Launcher$ExtClassLoader@480bdb19
 classLoaderHash   18b4aac2
```

> 与之相应的还有 sm( “Search-Method”  )，查看已加载类的方法信息

查看String里的方法

```shell
[arthas@86]$ sm java.lang.String
java.lang.String <init>([BII)V
java.lang.String <init>([BLjava/nio/charset/Charset;)V
java.lang.String <init>([BLjava/lang/String;)V
java.lang.String <init>([BIILjava/nio/charset/Charset;)V
java.lang.String <init>([BIILjava/lang/String;)V
... ... ... ...
```

查看String中toString的详细信息

```shell
[arthas@86]$ sm -d java.lang.String toString
declaring-class  java.lang.String
 method-name      toString
 modifier         public
 annotation
 parameters
 return           java.lang.String
 exceptions
 classLoaderHash  null
```

### <font color="red">watch 监测一个方法的入参和返回值</font>

有些问题线上会出现，本地重现不了，这时这个命令就有用了

| 参数名称            | 参数说明                                                 |
| :------------------ | :------------------------------------------------------- |
| *class-pattern*     | 类名表达式匹配                                           |
| *method-pattern*    | 方法名表达式匹配                                         |
| *express*           | 观察表达式，默认值：`{params, target, returnObj}`        |
| *condition-express* | 条件表达式                                               |
| [b]                 | 在**方法调用之前**观察                                   |
| [e]                 | 在**方法异常之后**观察                                   |
| [s]                 | 在**方法返回之后**观察                                   |
| [f]                 | 在**方法结束之后**(正常返回和异常返回)观察，**默认选项** |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配                     |
| [x:]                | 指定输出结果的属性遍历深度，默认为 1                     |

*express* 实际上是一个`ognl`表达式，它支持一些内置对象不同变量的含义：

|    变量名 | 变量解释                                                     |
| --------: | :----------------------------------------------------------- |
|    loader | 本次调用类所在的 ClassLoader                                 |
|     clazz | 本次调用类的 Class 引用                                      |
|    method | 本次调用方法反射引用                                         |
|    target | 本次调用类的实例                                             |
|    params | 本次调用参数列表，这是一个数组，如果方法是无参方法则为空数组 |
| returnObj | 本次调用返回的对象。当且仅当 `isReturn==true` 成立时候有效，表明方法调用是以正常返回的方式结束。如果当前方法无返回值 `void`，则值为 null |
|  throwExp | 本次调用抛出的异常。当且仅当 `isThrow==true` 成立时有效，表明方法调用是以抛出异常的方式结束。 |
|  isBefore | 辅助判断标记，当前的通知节点有可能是在方法一开始就通知，此时 `isBefore==true` 成立，同时 `isThrow==false` 和 `isReturn==false`，因为在方法刚开始时，还无法确定方法调用将会如何结束。 |
|   isThrow | 辅助判断标记，当前的方法调用以抛异常的形式结束。             |
|  isReturn | 辅助判断标记，当前的方法调用以正常返回的形式结束。           |

观察getOrderInfo的出参和返回值，出参就是方法结束后的入参

```shell
[arthas@86]$ watch cn.test.mobile.controller.order.OrderController getOrderInfo "{params,returnObj}" -x 2

Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 456 ms.
ts=2019-11-13 15:30:18; [cost=18.48307ms] result=@ArrayList[
    @Object[][  # 这个就是函数出参，params
        @RequestFacade[org.apache.catalina.connector.RequestFacade@1d81dbd7],
        @LinkedHashMap[isEmpty=false;size=2], # 把遍历深度x改为3就可以查看map里的值了
    ],
    @ResponseVo[ # 这个就是返回值 returnObj
        log=@Logger[Logger[cn.test.db.common.vo.ResponseVo]],
        success=@Boolean[true],
        message=@String[Ok],
        count=@Integer[0],
        code=@Integer[1000],
        data=@HashMap[isEmpty=false;size=1],
    ],
]
```

如果需要捕捉异常的话，使用`throwExp`，如`{params,returnObj,throwExp}`

`watch`命令支持按请求耗时进行过滤，比如：

```
watch com.example.demo.arthas.user.UserController * '{params, returnObj}' '#cost>200'
```

### tt 方法执行数据的时空隧道

即 TimeTunnel，它可以记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测

`watch` 虽然很方便和灵活，但需要提前想清楚观察表达式的拼写，这对排查问题而言要求太高，因为很多时候我们并不清楚问题出自于何方，只能靠蛛丝马迹进行猜测。

这个时候如果能记录下当时方法调用的所有入参和返回值、抛出的异常会对整个问题的思考与判断非常有帮助。

对于一个最基本的使用来说，就是记录下当前方法的每次调用环境现场：

```shell
[arthas@86]$ tt -t demo.MathGame primeFactors
Press Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 66 ms.
 INDEX   TIMESTAMP            COST(ms)  IS-RET  IS-EXP   OBJECT         CLASS                          METHOD
-------------------------------------------------------------------------------------------------------------------------------------
 1000    2018-12-04 11:15:38  1.096236  false   true     0x4b67cf4d     MathGame                       primeFactors
 1001    2018-12-04 11:15:39  0.191848  false   true     0x4b67cf4d     MathGame                       primeFactors
 1002    2018-12-04 11:15:40  0.069523  false   true     0x4b67cf4d     MathGame                       primeFactors
 1003    2018-12-04 11:15:41  0.186073  false   true     0x4b67cf4d     MathGame                       primeFactors
 1004    2018-12-04 11:15:42  17.76437  true    false    0x4b67cf4d     MathGame                       primeFactors
```

命令参数解析：

- `-t`

  tt 命令有很多个主参数，`-t` 就是其中之一。这个参数的表明希望记录下类 `*Test` 的 `print` 方法的每次执行情况。

- `-n 3`

  当你执行一个调用量不高的方法时可能你还能有足够的时间用 `CTRL+C` 中断 `tt` 命令记录的过程，但如果遇到调用量非常大的方法，瞬间就能将 JVM 内存撑爆。

  此时可以通过 `-n` 参数指定需要记录的次数，当达到记录次数时 Arthas 会主动中断 `tt` 命令的记录过程，避免人工操作无法停止的情况。

#### 检索调用记录

当用 `tt` 记录了一大片的时间片段之后，希望能从中筛选出自己需要的时间片段，这个时候需要对现有记录进行检索：

```bash
$ tt -s 'method.name=="primeFactors"'
 INDEX   TIMESTAMP            COST(ms)  IS-RET  IS-EXP   OBJECT         CLASS                          METHOD
-------------------------------------------------------------------------------------------------------------------------------------
 1000    2018-12-04 11:15:38  1.096236  false   true     0x4b67cf4d     MathGame                       primeFactors
 1001    2018-12-04 11:15:39  0.191848  false   true     0x4b67cf4d     MathGame                       primeFactors
 1002    2018-12-04 11:15:40  0.069523  false   true     0x4b67cf4d     MathGame                       primeFactors
 1003    2018-12-04 11:15:41  0.186073  false   true     0x4b67cf4d     MathGame                       primeFactors
 1004    2018-12-04 11:15:42  17.76437  true    false    0x4b67cf4d     MathGame                       primeFactors
                              9
 1005    2018-12-04 11:15:43  0.4776    false   true     0x4b67cf4d     MathGame                       primeFactors
Affect(row-cnt:6) cost in 607 ms.
```

需要一个 `-s` 参数。同样的，搜索表达式的核心对象依旧是 `Advice` 对象。

#### 查看调用信息查看调用信息

对于具体一个时间片的信息而言，可以通过 `-i` 参数后边跟着对应的 `INDEX` 编号查看到他的详细信息。

```bash
$ tt -i 1003
 INDEX            1003
 GMT-CREATE       2018-12-04 11:15:41
 COST(ms)         0.186073
 OBJECT           0x4b67cf4d
 CLASS            demo.MathGame
 METHOD           primeFactors
 IS-RETURN        false
 IS-EXCEPTION     true
 PARAMETERS[0]    @Integer[-564322413]
 THROW-EXCEPTION  java.lang.IllegalArgumentException: number is: -564322413, need >= 2
                      at demo.MathGame.primeFactors(MathGame.java:46)
                      at demo.MathGame.run(MathGame.java:24)
                      at demo.MathGame.main(MathGame.java:16)

Affect(row-cnt:1) cost in 11 ms.
```

#### 重做一次调用

当做了一些调整之后，可能需要前端系统重新触发一次调用。`tt` 命令由于保存了当时调用的所有现场信息，所以可以自己主动对一个 `INDEX` 编号的时间片自主发起一次调用，从而解放沟通成本。此时需要 `-p` 参数。通过 `--replay-times` 指定 调用次数，通过 `--replay-interval` 指定多次调用间隔(单位 ms, 默认 1000ms)

```bash
$ tt -i 1004 -p
 RE-INDEX       1004
 GMT-REPLAY     2018-12-04 11:26:00
 OBJECT         0x4b67cf4d
 CLASS          demo.MathGame
 METHOD         primeFactors
 PARAMETERS[0]  @Integer[946738738]
 IS-RETURN      true
 IS-EXCEPTION   false
 COST(ms)         0.186073
 RETURN-OBJ     @ArrayList[
                    @Integer[2],
                    @Integer[11],
                    @Integer[17],
                    @Integer[2531387],
                ]
Time fragment[1004] successfully replayed.
Affect(row-cnt:1) cost in 14 ms.
```

发现结果虽然一样，但调用的路径发生了变化，由原来的程序发起变成了 Arthas 自己的内部线程发起的调用了

需要强调的点：

1. **ThreadLocal 信息丢失**

   很多框架偷偷的将一些环境变量信息塞到了发起调用线程的 ThreadLocal 中，由于调用线程发生了变化，这些 ThreadLocal 线程信息无法通过 Arthas 保存，所以这些信息将会丢失。

   一些常见的 CASE 比如：鹰眼的 TraceId 等。

2. **引用的对象**

   需要强调的是，`tt` 命令是将当前环境的对象引用保存起来，但仅仅也只能保存一个引用而已。如果方法内部对入参进行了变更，或者返回的对象经过了后续的处理，那么在 `tt` 查看的时候将无法看到当时最准确的值。这也是为什么 `watch` 命令存在的意义。

### stack 输出当前方法被调用的路径

很多时候我们都知道一个方法被执行，但是有很多地方调用了它，你并不知道是谁调用了它，此时需要的是 stack 命令。

| 参数名称         | 参数说明         |
| :--------------- | :--------------- |
| *class-pattern*  | 类名表达式匹配   |
| *method-pattern* | 方法名表达式匹配 |

```shell
[arthas@79952]$ stack com.baomidou.mybatisplus.extension.service.IService getOne
Press Q or Ctrl+C to abort.
Affect(class-cnt:202 , method-cnt:209) cost in 10761 ms.
ts=2019-11-13 11:49:13;thread_name=http-nio-8801-exec-6;id=2d;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@a6c54c3
    @com.baomidou.mybatisplus.extension.service.impl.ServiceImpl.getOne()
        at com.baomidou.mybatisplus.extension.service.IService.getOne(IService.java:230)
        ...... ......
        at cn.test.mobile.controller.order.OrderController.getOrderInfo(OrderController.java:500)
```

可以看到`OrderController.java`的第500行调用了这个`getOne`接口

### trace 输出方法内部调用路径，和路径上每个节点的耗时

可以通过这个命令，查看哪些方法耗性能，从而找出导致性能缺陷的代码，这个耗时还包含了arthas执行的时间哦。

| 参数名称            | 参数说明                             |
| :------------------ | :----------------------------------- |
| *class-pattern*     | 类名表达式匹配                       |
| *method-pattern*    | 方法名表达式匹配                     |
| *condition-express* | 条件表达式                           |
| [E]                 | 开启正则表达式匹配，默认为通配符匹配 |
| `[n:]`              | 命令执行次数                         |
| `#cost`             | 方法执行耗时                         |

输出getOrderInfo的调用路径

```shell
[arthas@79952]$ trace -j cn.test.mobile.controller.order.OrderController getOrderInfo

Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 92 ms.
---ts=2019-11-13 15:46:59;thread_name=http-nio-8801-exec-4;id=2b;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@a6c54c3
    ---[15.509011ms] cn.test.mobile.controller.order.OrderController:getOrderInfo()
        +---[0.03584ms] cn.test.db.common.vo.ResponseVo:<init>() #472
        +---[0.00992ms] java.util.HashMap:<init>() #473
        +---[0.02176ms] cn.test.mobile.controller.order.OrderController:getUserInfo() #478
        +---[0.024ms] java.util.Map:get() #483
        +---[0.00896ms] java.lang.Object:toString() #483
        +---[0.00864ms] java.lang.Integer:parseInt() #483
        +---[0.019199ms] com.baomidou.mybatisplus.core.conditions.query.QueryWrapper:<init>() #500
        +---[0.135679ms] com.baomidou.mybatisplus.core.conditions.query.QueryWrapper:allEq() #500
        +---[12.476072ms] cn.test.db.service.IOrderMediaService:getOne() #500
        +---[0.0128ms] java.util.HashMap:put() #501
        +---[0.443517ms] cn.test.db.common.vo.ResponseVo:setSuccess() #503
        `---[0.03488ms] java.util.Map:put() #504
```

输出`getOrderInfo`的调用路径，且cost大于10ms，`-j`是指过滤掉`jdk`中的方法，可以看到输出少了很多:

```shell
[arthas@79952]$ trace -j cn.test.mobile.controller.order.OrderController getOrderInfo '#cost > 10'

Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 96 ms.
---ts=2019-11-13 15:53:42;thread_name=http-nio-8801-exec-2;id=29;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@a6c54c3
    ---[13.803743ms] cn.test.mobile.controller.order.OrderController:getOrderInfo()
        +---[0.01312ms] cn.test.db.common.vo.ResponseVo:<init>() #472
        +---[0.01408ms] cn.test.mobile.controller.order.OrderController:getUserInfo() #478
        +---[0.0128ms] com.baomidou.mybatisplus.core.conditions.query.QueryWrapper:<init>() #500
        +---[0.303998ms] com.baomidou.mybatisplus.core.conditions.query.QueryWrapper:allEq() #500
        +---[12.675431ms] cn.test.db.service.IOrderMediaService:getOne() #500
        `---[0.409917ms] cn.test.db.common.vo.ResponseVo:setSuccess() #503
```

### ognl 可以动态执行代码

| 参数名称              | 参数说明                                                     |
| :-------------------- | :----------------------------------------------------------- |
| *express*             | 执行的表达式                                                 |
| `[c:]`                | 执行表达式的 ClassLoader 的 hashcode，默认值是 SystemClassLoader |
| `[classLoaderClass:]` | 指定执行表达式的 ClassLoader 的 class name                   |
| [x]                   | 结果对象的展开层次，默认值 1                                 |

对于只有唯一实例的ClassLoader可以通过`--classLoaderClass`指定class name，使用起来更加方便，而`-c <hashcode>`是动态变化的：

```shell
[arthas@79952]$ ognl --classLoaderClass org.springframework.boot.loader.LaunchedURLClassLoader  @org.springframework.boot.SpringApplication@logger
@Slf4jLocationAwareLog[
    FQCN=@String[org.apache.commons.logging.LogAdapter$Slf4jLocationAwareLog],
    name=@String[org.springframework.boot.SpringApplication],
    logger=@Logger[Logger[org.springframework.boot.SpringApplication]],
]
```



### jobs 执行后台异步任务

线上有些问题是偶然发生的，这时就需要使用异步任务，把信息写入文件。

**使用 & 指定命令去后台运行**，使用 > 将结果重写到日志文件，以trace为例

```shell
[arthas@79952]$ trace -j cn.test.mobile.controller.order.OrderController getOrderInfo > test.out &
```

`jobs`——列出所有异步job：

```shell
[arthas@79952]$ jobs
[76]*  
       Running           trace -j cn.test.mobile.controller.order.OrderController getOrderInfo >> test.out &
       execution count : 0
       start time      : Wed Nov 13 16:13:23 CST 2019
       timeout date    : Thu Nov 14 16:13:23 CST 2019
       session         : f4fba846-e90b-4234-959e-e78ad0a5db8c (current)
```

job id是76, \* 表示此job是当前session创建，状态是Running，execution count是执行次数，timeout date是超时时间

异步执行时间，默认为1天，如果要修改，使用`options`命令,

```shell
[arthas@79952]$ options job-timeout 2d
```

`options`可选参数 1d, 2h, 3m, 25s，分别代表天、小时、分、秒

`kill`——强制终止任务

```shell
[arthas@79952]$ kill 76
kill job 76 success
```

最多同时支持8个命令使用重定向将结果写日志

请勿同时开启过多的后台异步命令，以免对目标JVM性能造成影响

### redefine 在不重启项目的情况下，热更新类

假设想修改`OrderController`里的某几行代码，然后热更新至JVM：

a. 反编译`OrderController`，默认情况下，反编译结果里会带有`ClassLoader`信息，通过--source-only选项，可以只打印源代码。方便和`mc/redefine`命令结合使用

```shell
[arthas@79952]$ jad --source-only cn.test.mobile.controller.order.OrderController > OrderController.java
```

生成的`OrderController.java`在哪呢，执行`pwd`就知道在哪个目录了

b. 查找加载`OrderController`的`ClassLoader`

```shell
[arthas@79952]$ sc -d cn.test.mobile.controller.order.OrderController | grep classLoaderHash
classLoaderHash   18b4aac2
```

c. 修改保存好`OrderController.java`之后，使用`mc(Memory Compiler)`命令来编译成字节码，并且通过`-c`参数指定`ClassLoader`

```shell
[arthas@79952]$ mc -c 18b4aac2 OrderController.java -d ./
```

d. 热更新刚才修改后的代码

```shell
[arthas@79952]$ redefine -c 18b4aac2 OrderController.class
redefine success, size: 1
```

然后代码就更新成功了。

高级用法 [获取 spring context 调用 bean 方法](https://github.com/alibaba/arthas/issues/482)

## 应用热点，火焰图

[生成火焰图](https://arthas.aliyun.com/doc/profiler.html)

## 退出

用 `exit` 或者 `quit` 命令可以退出Arthas。

退出Arthas之后，还可以再次用 `java -jar arthas-boot.jar` 来连接。

`exit/quit`命令只是退出当前session，arthas server还在目标进程中运行。想完全退出Arthas，可以执行 `stop` 命令。