## 基本语法
### 变量

定义：使用`var`关键字，也可使用`var( )`来集中定义，编译器可自动决定类型，**在函数内定义变量可使用`:=`省略`var`的方式**，<font color="red">变量类型写在变量名之后</font>

```go
// zero value
var a int
var b string
fmt.Printf("%d %q\n", a, b)

// init value
var a1, a2 int = 3, 4
var b1 string = "abc"
fmt.Println(a1, a2, b1)

// type deduction
var c, d, e, f = 1, 2, true, "edf"
fmt.Println(c, d, e, f)

// defined shorter
c1, d1, e1, f1 := 3, 4, false, "hhh"
d1 = 6
fmt.Println(c1, d1, e1, f1)

// package variable
fmt.Println(aa, bb, cc)

var (
    aaa int
    bbb string = "ddd"
)
```

类型：

- `bool`, `string`
- `(u)int`, `(u)int8`, `(u)int16`, `(u)int32`, `(u)int64`, `uintptr`
- `byte`, `rune` (`rune`理解为`char`)
- `float32`, `float64`, `complex64`, `complex128`, 

- 常量、枚举

  常量使用`const`关键字表示，也可用`const( )`来集中定义

  枚举可以通过`const`块的方式来定义，可以通过`iota`关键字来实现自增：

  ```go
  const (
      aa = iota
      bb
      cc
      dd
  )
  fmt.Println(aa, bb, cc, dd) // 0 1 2 3
  ```

### 数组、容器、字符串

- 数组: go一般不直接使用数组

  ```go
  var arr1 [5]int
  arr2 := [3]int{1, 3, 5}
  // 由编译器决定数组大小
  arr3 := [...]int{2, 4, 6}
  arr3 := []int{2, 4, 6}
  // range 关键字可以获取到集合的元素下标和值
  for i, v := range arr3 {
      fmt.Println(i, v)
  }
  ```

- 切片

  ```go
  arr := [...]int{0, 1, 2, 3, 4, 5, 6, 7}
  s1 := arr[2:6] // [2 3 4 5]
  s2 := s1[3:5] // s2=[5 6], 不会越界，但s1[4]会越界报错
  
  // append元素
  a1 := append(s2, 10) // [5 6 10]
  a2 := append(a1, 11) // [5 6 10 11]
  // arr = [0 1 2 3 4 5 6 10]
  
  // 创建初始化的slice
  s3 := make([]int, 10) // len=10, cap = 10
  s4 := make([]int, 10, 32) // len = 10, cap = 32，初始值都为0
  
  // s2元素copy到s3，s3=[5 6 0 0 0 0 0 0 0 0]
  copy(s3, s2)
  ```

  > 1. `s[i]` 不可以超越`len(s)`，向后扩展不可以超越底层数组`cap(s)`，也不可向前扩展
  > 2. 向slice添加元素，未超过cap，会替换原有数组的内容；超过cap，会重新分配新的底层数组

- Map

  定义：map[K]V，map[K1]map[K2]V2

  ```go
  // 定义，除了slice、map、function的内建类型都可以作为key，struct不包含上述类型也可作为key
  m := make(map[string]int)
  m1 := map[string]string{
      "name": "zhangsan",
      "city": "北京",
  }
  // 遍历，不保证顺序，如需排序，需手动对key排序，可以把key放在slice中
  for k, v := range m1 {
      fmt.Println(k, v)
  }
  // 取值，key不存在，获取value类型的初始值
  if name, exist := m1["nn"]; exist {
      fmt.Println(name)
  } else {
      fmt.Println("key not exist")
  }
  // 删除元素
  delete(m1, "name")
  ```

- 字符串

  ```go
  str := "你好美女"
  // len获取字节长度，一个中文占3个字节
  fmt.Println(len(str))
  // []byte获得字节
  fmt.Println([]byte(str))
  // utf8.RuneCountInString 获取字符数
  fmt.Println(utf8.RuneCountInString(str))
  // 中文长度占3个长度，i = 0,3,6,9，v 就是一个rune，是一个unicode编码
  for i, v := range str {
      fmt.Printf("(%d, %d)", i, v)
  }
  // 直接转为[]rune,会新开空间存储 i = 0,1,2,3
  for i, v := range []rune(str) {
      fmt.Printf("(%d, %c)", i, v)
  }
  ```

  > 字符串其他相关操作都在`strings`包下

### 选择、循环

- if 的条件里不需要括号，条件中可以赋值，条件里赋值的变量作用域只在if 块中

  ```go
  const fileName = "abc.txt"
  if contents, err := ioutil.ReadFile(fileName); err != nil {
      fmt.Println(err)
  } else {
      fmt.Printf("%s\n", contents)
  }
  ```

- switch 会自动break，不需要在每个case中单独加；switch之后可以没有表达式，在case放置条件

  ```go
  grade := ""
  switch {
  case score < 0 || score > 100:
      panic(fmt.Sprintf("Wrong Score: %d", score))
  case score < 60:
      grade = "D"
  case score < 80:
      grade = "C"
  case score < 90:
      grade = "B"
  case score <= 100:
      grade = "A"
  default:
      panic(fmt.Sprintf("Wrong Score: %d", score))
  }
  return grade
  ```

- for 条件不需要括号，可以省略初始条件，结束条件，递增条件

  ```go
  sum := 0
  for i := 0; i < 100; i++ {
      sum += i
  }
  fmt.Println(sum)
  ```

- 没有while关键字，省略初始条件和递增条件，就是while循环；省略结束条件就是死循环

  ```go
  for age < 18 {
      ...
  }
  ```

### 函数、指针

与Java都是反的，go函数的**函数名在前，返回值在后**；**变量名在前，类型在后**；**可以有多个返回值**

```go
func eval(a, b int, op string) (int, error) {
    
}
// 如果函数返回值有多个，但只需要其中一个返回值可以通过_来忽略返回值
result,_ := evel(3, 4, "*")

// 可变长参数
func test(a ...int) int {
}
```

指针：在Go语言中，指针比较简单，指针不可运算。它的使用方式：

1. 定义指针变量：它的表示也是反的，**通过\* + 类型来表示一个指针类型**；
2. 为指针赋值：Go 语言的取地址符是 &，放到一个变量前使用就会返回相应变量的内存地址；
3. 访问指针变量中指向地址的值：在指针类型前面加上 * 号（前缀）来获取指针所指向的内容

```go
var x int = 8
// 声明指针变量
var p *int
// 为指针变量赋值
p = &x
fmt.Println("p is ", p)
// 访问指针变量中指向地址的值
fmt.Println("*p is ", *p)
*p = 9
fmt.Println(x)
```



## 面向对象

go语言仅支持封装，不支持继承和多态，没有class，只有struct

```go
type TreeNode struct {
	Left, Right *TreeNode
	Value       int
}

// 为结构定义方法:显示定义接收者
func (node TreeNode) GetNodeValue() int {
	return node.Value
}
// 只有指针才可以改变结构体内容
func (node *TreeNode) setNodeValue(value int) {
	node.Value = value
}

// 结构没有构造函数，可以通过工厂方法
func createNode(value int) *TreeNode {
    return &TreeNode{Value: value}
}

// 对象初始化
var root TreeNode
root = TreeNode{Value: 3}
root.Left = &TreeNode{}
root.Right = &TreeNode{nil, nil, 5}
// new 关键字返回对象地址
root.Right.Left = new(TreeNode)

// 可以通过定义别名方式来扩展原有结构体的方法
type MyNode TreeNode
```

扩展已有类型：可以通过组合、定义别名、内嵌的方式来扩展已有类型，内嵌与继承有点类似，但不是真的继承如没有多态表示等，只是个语法糖

```go
type myNode struct {
    // 内嵌方式
	*test.TreeNode // 理解为myNode结构中有一个属性叫TreeNode
}

// 通过内嵌的方式，可以直接使用原结构的属性和方法，也可以对其进行重载和扩展
node := myNode{&test.TreeNode{Value: 3}}
node.Left = &test.TreeNode{Value: 4}
node.Right = &test.TreeNode{Value: 5}
fmt.Println(node.GetNodeValue())
```

封装：

- 命名统一采用驼峰形式
- 首字母大写：public
- 首字母小写：private

包管理：

- 每个目录一个包，包名不一定和目录名相同
- main包包含可执行入口
- 为结构定义的方法必须放在**同一包内**，可以是不同的文件



## 面向接口

接口定义：

```go
type Retriever interface {
    // 接口方法，不需要显式指定func
	Get(url string) string
}
```

接口组合：

```go
// 接口同时具有Retriever、Poster的接口方法
type RetrieverPoster interface {
    Retriever
    Poster
}
```

接口实现：

- 接口实现是隐式的
- 只要实现接口里的方法

```go
package mock

// 结构体不需要声明实现哪个接口
type Retriever struct {
	Msg string
}

// 只需要实现方法与接口中定义的保持一样
// 值接收者，使用时候指针和值方式都可，如：r = mock.Retriever{} 或 r = &mock.Retriever{}
func (receiver Retriever) Get(url string) string {
	return receiver.Msg
}
-------------------
package real

type Retriever struct {
	Header  string
	TimeOut time.Duration
}

// 采用了指针接收者，在使用的时候只能使用指针方式，见main()中的 r = &real.Retriever{}
func (r *Retriever) Get(url string) string {
	...
	return r.Header
}
```

指针变量：包含实现者的类型（type switch）和实现者的值（Type assertion）

```go
func main() {
	var r Retriever // 接口变量
	r = mock.Retriever{}
	fmt.Printf("%T %v\n", r, r)
	// 接口变量自带指针，同样采用值传递，几乎不使用接口的指针
	r = &real.Retriever{}
    
    // .(type) 为 type switch，在switch语句中使用
	switch v := r.(type) {
	case mock.Retriever:
		fmt.Println("It is mock retriever: ", v.Msg)
	case *real.Retriever:
		fmt.Println("It is mock retriever", v.Header)
	}

    // Type assertion，通过`接口对象.`方式来获取到接口变量中的对象值
	c, ok := r.(mock.Retriever)
	fmt.Println(c, ok)
}
```



## 函数式编程

闭包概念：一个函数的内部函数为匿名函数，匿名函数优越性在于可以直接使用这个函数内的变量，不必申明。当一个外部函数返回结果为其内部函数时，相关参数和变量都保存在返回的函数中，这种称为“闭包(Closure)”的程序结构拥有极大的威力。

返回闭包时牢记的一点就是：返回函数不要引用任何循环变量，或者后续会发生变化的变量：

```go
// 方法返回的是一个函数数组
func count() []func() int {
	var arr = make([]func() int, 3)
	for i := 0; i < 3; i++ {
		arr[i] = func() int {
			return i * i
		}
	}
	return arr
}

func main() {
	result := count()
	f1 := result[0]
	f2 := result[1]
	f3 := result[2]
	// 返回的函数引用了变量i，但它并非立刻执行，等到3个函数都返回时，它们所引用的变量i已经变成了3，因此最终结果为9。
	fmt.Println(f1()) // 9
	fmt.Println(f2()) // 9
	fmt.Println(f3()) // 9
}
```

go语言中闭包使用：

- 没有lambda表达式，有匿名函数
- 更为自然，不需要修饰如何访问自由变量

为函数实现接口：函数在go中不光可以作为参数、类型、返回值，同时还可以实现接口

```go
// 定义函数
type fbiGen func() int

// 函数式编程，返回结果为函数，其内包含了相关参数和局部变量a, b都保存在返回的函数中
func fbiInitFunction() fbiGen {
	a, b := 0, 1
	return func() int {
		a, b = b, a+b
		return a
	}
}

// 为函数fbiGen 实现 io.Reader 接口
func (f fbiGen) Read(p []byte) (n int, err error) {
	next := f()
	if next > 10000 {
		return 0, io.EOF
	}

	s := fmt.Sprintf("%d\n", next)
	return strings.NewReader(s).Read(p)
}

// 定义接收io.Reader参数的方法
func printContents(reader io.Reader) {
	scanner := bufio.NewScanner(reader)
	for scanner.Scan() {
		fmt.Println(scanner.Text())
	}
}

func main() {
	f := fbiInitFunction()
    // 函数就可以传入接收io.Reader的方法中
	printContents(f)
}
```



## 依赖管理

依赖管理的三个阶段：GOPATH、GOVENDOR、go mod

- GOPATH: 会去`$GOPATH`目录下和`$GOROOT`目录下去加载依赖包（可以为项目配置不同的`$GOPATH`，`$GOPATH`下需要有`src`目录）

- GOVENDOR：每个项目有自己的vendor目录，存放第三方库；会有不同的第三方工具来管理依赖包，将需要的包导入到vendor中（类似maven）

- go mod: 由go 命令统一管理，用户不必关心目录结构

  初始化：`go mod init`，会下项目下生成`go.mod`文件，用来记录依赖的第三方包

  增加依赖：`go get`会去加载依赖包到`$GOPATH/pkg/mod`目录下，同时将依赖记录在`go mod`文件中  / 也可以直接import，在build（`go build ./...`）时候会自动修改mod文件

  设置代理：
  
  ```shell
  # 开启 Go Module
  go env -w GO111MODULE=on
  # 设置国内镜像
  go env -w GOPROXY=https://goproxy.cn,direct
  # 可以对私有仓库不设置代理
  go env -w GOPRIVATE=*.xxx.com
  # goimports
  # go get 内部使用https的clone命令，默认只支持公有仓库
  go get -v golang.org/x/tools/cmd/goimports
  ```
  
  ​	国内可以设置代理 `go env -w GOPROXY=https://goproxy.cn,direct`， `-w` 标记 要求一个或多个形式为 NAME=VALUE 的参数且覆盖默认的设置
  
  更新依赖：`go get 库[@version]` ，`go mod tidy`

### go module导入本地包

现在有文件目录结构如下：

```bash
├── p1
│   ├── go.mod
│   └── main.go
└── p2
    ├── go.mod
    └── p2.go
```

`p1/main.go`中想要导入`p2.go`中定义的函数。

`p2/go.mod`内容如下：

```go
module liwenzhou.com/q1mi/p2

go 1.14
```

`p1/main.go`中按如下方式导入

```go
import (
	"fmt"
	"liwenzhou.com/q1mi/p2"
)
func main() {
	p2.New()
	fmt.Println("main")
}
```

因为并没有把`liwenzhou.com/q1mi/p2`这个包上传到`liwenzhou.com`这个网站，现在只是想导入本地的包，这个时候就需要用到`replace`这个指令：`go mod edit -replace pack=../pack`。

`p1/go.mod`内容如下：

```go
module github.com/q1mi/p1

go 1.14

require "liwenzhou.com/q1mi/p2" v0.0.0
replace "liwenzhou.com/q1mi/p2" => "../p2"
```

此时就可以正常编译`p1`这个项目了



## 工程化

### 资源管理，错误处理

`defer` 关键字：

- 可以确保调用在函数结束时发生
- `defer`列表为先进后出
- 参数在执行`defer`语句时计算

错误处理：

- `panic(err)`可以打印出错信息，停止当前函数执行，一直向上返回，执行每一层的`defer`，如果没有遇见`recover()`程序就会退出

- `recover()`仅在`defer`调用中使用，获取`panic`值，如果无法处理，可重新`panic`

  ```go
  func main() {
  	defer func() {
  		// r 为panic中内容
  		r := recover()
  		if err, ok := r.(error); ok {
  			fmt.Println("Error occurred: ", err)
  		} else {
  			panic("I know nothing")
  		}
  	}()
  
  	panic(errors.New("test error"))
  }
  ```

### 测试

编写测试和函数很类似，其中有一些规则

- 程序需要在一个名为 `xxx_test.go` 的文件中编写
- 测试函数的命名必须以单词 `Test` 开始
- 测试函数接受一个参数 `t *testing.T`
- TestMain 的参数是 `*testing.M` 类型

类型为 `*testing.T` 的变量 `t` 是在测试框架中的 hook（钩子），当想让测试失败时可以执行 `t.Fail()` 之类的操作

```go
func TestAdd(t *testing.T) {
    // 表格驱动测试，将测试内容和结果，放在一起来校验
	test := []struct{ a, b, c int }{
		{1, 2, 3},
		{2, 3, 6},
		{5, 6, 11},
	}

	for _, tt := range test {
		if result := Add(tt.a, tt.b); result != tt.c {
			t.Errorf("Add Test(%d, %d) result is %d, expect is %d", tt.a, tt.b, result, tt.c)
		}
	}
}

// 如果测试文件中包含函数 TestMain，那么生成的测试将调用 TestMain(m)，而不是直接运行测试
func TestMain(m *testing.M) {
    //  m.Run() 触发所有测试用例的执行
	code := m.Run()
    // 使用 os.Exit() 处理返回的状态码，如果不为0，说明有用例失败
	os.Exit(code)
}
```

`go test .` 命令可以运行当前目录下的测试文件

```shell
# 覆盖率
go test -race -cover  -coverprofile=./coverage.out -timeout=10m -short -v ./...
go tool cover -func ./coverage.out
```

性能测试：

- 函数名必须以 `Benchmark` 开头，后面一般跟待测试的函数名
- 参数为 `b *testing.B`

```go
func BenchmarkAdd(b *testing.B) {
	p1, p2 := 1, 2
	ans := 8
    
    // 重置定时器
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		result := Add(p1, p2)
		if result != ans {
			b.Errorf("Add Test(%d, %d) result is %d, expect is %d", p1, p2, result, ans)
		}
	}
}

func BenchmarkParallel(b *testing.B) {
	templ := template.Must(template.New("test").Parse("Hello, {{.}}!"))
    // RunParallel 测试并发性能
	b.RunParallel(func(pb *testing.PB) {
		var buf bytes.Buffer
		for pb.Next() {
			// 所有 goroutine 一起，循环一共执行 b.N 次
			buf.Reset()
			templ.Execute(&buf, "World")
		}
	})
}
```

http测试：
针对 http 开发的场景，使用标准库 `net/http/httptest` 进行测试更为高效:

```go
-------- to be tested
func helloHandler(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("hello world"))
}

-------- test code
import (
	"io/ioutil"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestConn(t *testing.T) {
	req := httptest.NewRequest("GET", "http://example.com/foo", nil)
	w := httptest.NewRecorder()
	helloHandler(w, req)
	bytes, _ := ioutil.ReadAll(w.Result().Body)

	if string(bytes) != "hello world" {
		t.Fatal("expected hello world, but got", string(bytes))
	}
}
```

### 文档

1. 注释内容最终可以用来生成文档
2. 在测试中加入`Example`前缀，来生成文档中的示例
3. 使用`go doc` / `godoc -http :6000` 来查看生成文档



## 并发编程

### goroutine

`go`关键字可以定义开启一个协程去运行相应函数：

- 任何函数只需要加上`go` 就能给调度器来运行
- 不需要在定义时，区分是否为异步函数
- 调度器会在合适点进行切换
- 使用`-race`来检测数据访问冲突，eg：`go run -race goroutine.go`

```go
func main() {
	for i := 0; i < 1000; i++ {
		go func(i int) {
			for {
				fmt.Println("test goruntine %d", i)
			}
		}(i)
	}
	time.Sleep(time.Millisecond)
}
```

协程 coroutine：
- 轻量级“线程”
- 非抢占式多任务处理，由协程主动交出控制权
- 编译器/解释器/虚拟机层面的多任务，非操作系统层面
- 多个协程可能在一个或多个线程上运行

goroutine 切换点：

- I/O，select
- channel
- 等待锁
- 函数调用（有时）
- runtime.Gosched()  手动提供的一个切换点
- 只是参考，不能保证切换，不能保证在其他地方不切换

### channel

单纯地将函数并发执行是没有意义的，函数与函数间需要交换数据才能体现并发执行函数的意义。虽然可以使用共享内存进行数据交换，但是共享内存在不同的goroutine中容易发生竞态问题。为了保证数据交换的正确性，必须使用互斥量对内存进行加锁，这种做法势必造成性能问题。

Go语言的并发模型是CSP（Communicating Sequential Processes），**提倡通过通信共享内存而不是通过共享内存而实现通信**。

如果说goroutine是Go程序并发的执行体，channel就是它们之间的连接。<font color="red">**channel是可以让一个goroutine发送特定值到另一个goroutine的通信机制**</font>，所以需要一个goroutine向channel发，另一个goroutine从channel接。通道（channel）是一种特殊的类型，像一个传送带或者队列，总是遵循先入先出（First In First Out）的规则，保证收发数据的顺序。

每一个通道都是一个具体类型的导管，也就是声明channel的时候需要为其指定元素类型。

```go
// channel是一种类型，一种引用类型，声明格式：var 变量 chan 元素类型
// 通道是引用类型，通道类型的空值是nil
var ch1 chan int // 声明一个传递整型的通道

// 声明的通道后需要使用make函数初始化之后才能使用, make(chan 元素类型, [缓冲大小])
ch2 := make(chan int)

// 单向通道使用：在函数传参及任何赋值操作中将双向通道转换为单向通道是可以的，但反过来是不可以的
// chan<- int是一个只能发送的通道，可以发送但是不能接收
func counter(out chan<- int) {
    for i := 0; i < 100; i++ {
        out <- i
    }
    close(out)
}

// <-chan int是一个只能接收的通道，可以接收但是不能发送
func printer(in <-chan int) {
    for i := range in {
        fmt.Println(i)
    }
}
```

通道有发送（send）、接收(receive）和关闭（close）三种操作：

```go
// 1. 发送
ch1 <- 10 // 把10发送到ch中

// 2. 接收
x := <-ch1 // 从ch中接收值并赋值给变量x
<-ch1      // 从ch中接收值，忽略结果

// 3. 关闭
close(ch1) // 通过调用内置的close函数来关闭通道

// 判断通道是否关闭
for {
    // ok通道中是否还有值
    i, ok := <-ch1 
    if !ok {
        break
    }
}

// 通道关闭后无值会退出for range循环
for i := range ch2 { 
    fmt.Println(i)
}
```

关闭后的通道有以下特点：

1. 对一个关闭的通道再发送值就会导致panic。
2. 对一个关闭的通道进行接收会一直获取值，直到通道为空。
3. 对一个关闭的并且没有值的通道执行接收操作，会得到对应类型的零值。
4. 关闭一个已经关闭的通道会导致panic。

缓冲通道：

```go
------------ 无缓冲
func recv(c chan int) {
    ret := <-c
    fmt.Println("接收成功", ret)
}

func main() {
    // 无缓冲的通道，无缓冲的通道只有在有人接收值的时候才能发送值
    // 无缓冲通道上的发送操作会阻塞，直到另一个goroutine在该通道上执行接收操作
    ch := make(chan int)
    // 如果没有通道接收，会产生deadLock
    go recv(ch) // 启用goroutine从通道接收值
    ch <- 10
    fmt.Println("发送成功")
}

--------------- 有缓冲通道

func main() {
    // 只要通道的容量大于零，那么该通道就是有缓冲的通道，通道的容量表示通道中能存放元素的数量
    ch := make(chan int, 1) // 创建一个容量为1的有缓冲区通道
    ch <- 10
    fmt.Println("发送成功")
}
```

### select调度
Go内置了`select`关键字，可以同时响应多个通道的操作

```go
// select的使用类似于switch语句，它有一系列case分支和一个默认的分支
// select可以同时监听一个或多个channel
// 每个case会对应一个通道的通信（接收或发送）过程。select会一直等待，直到某个case的通信操作完成时，就会执行case分支对应的语句
select {
case <-chan1:
	// 如果chan1成功读到数据，则进行该case处理语句
case chan2 <- 1: // 若 chan2 = nil channel 不会执行
	// 如果成功向chan2写入数据，则进行该case处理语句
default:
	// 如果上面都没有成功，则进入default处理流程，有default相当于非阻塞式的，无是阻塞式的
}
```

### 定时器

```go
finish := time.After(10 * time.Second)    // 多长时间后，返回值为channel
tick := time.Tick(800 * time.Millisecond) // 定时器，返回值为channel
for {
	select {
	case <-time.After(1000 * time.Millisecond):
		fmt.Println("timeout")
	case <-tick:
		fmt.Println("定时执行")
	case <-finish:
		fmt.Println("bye")
		return

	}
}
```

### 传统锁

```go
var x int64
var wg sync.WaitGroup // 可以等待多任务执行结束
var lock sync.Mutex // Go语言中使用sync包的Mutex类型来实现互斥锁

func add() {
    for i := 0; i < 5000; i++ {
        lock.Lock() // 加锁
        defer lock.Unlock() // 解锁
        x = x + 1
    }
    wg.Done() // 等待的任务完成，数量会-1
}
func main() {
    wg.Add(2) // 设置等待数量
    go add()
    go add()
    wg.Wait() // 开始等待，直到全部完成
    fmt.Println(x)
}
```



## 反射机制

反射是指在程序运行期对程序本身进行访问和修改的能力

- reflect包封装了反射相关的方法
- 获取类型信息：`reflect.TypeOf`，是静态的
- 获取值信息：`reflect.ValueOf`，是动态的

```go
type User struct {
	Id   int
	Name string
	Age  int
}

func (u User) Hello(name string) {
	fmt.Println("Hello：", name)
}

func main() {
    u := User{1, "5lmh.com", 20}
	reflectGet(u)
	reflectSet(&u)
}

func reflectGet(o interface{}) {
	// 获取类型
	t := reflect.TypeOf(o)
	fmt.Println("type is :", t)
	fmt.Println("type name is：", t.Name()) // 缩略名

	// 获取值
	v := reflect.ValueOf(o)
	fmt.Println("value is :", v)

	// 获取结构体字段个数
	fmt.Println("field num is :", t.NumField())
	// 获取字段值t.Field(i)
	f := t.Field(0)
	fmt.Printf("name is %s : type value is %v\n", f.Name, f.Type)
	// Interface()：获取字段对应的值
	val := v.Field(0).Interface()
	fmt.Println("val :", val)

	// 获取方法
	m := t.Method(0)
	fmt.Println(m.Name)
	fmt.Println(m.Type)
}

func reflectSet(o interface{}) {
	v := reflect.ValueOf(o)

	// 获取指针指向的元素
	e := v.Elem()
	// 取字段
	fn := e.FieldByName("Name")
	if fn.Kind() == reflect.String {
		fn.SetString("kuteng")
	}

	// 获取方法
	cm := e.MethodByName("Hello")
	// 构建一些参数
	args := []reflect.Value{reflect.ValueOf("6666")}
	// 没参数的情况下：var args2 []reflect.Value
	// 调用方法，需要传入方法的参数
	cm.Call(args)
}

```



## HTTP标准库

Go语言内置的`net/http`包提供了HTTP客户端和服务端的实现

### 服务端

`server.ListenAndServe` 使用指定的监听地址和处理器启动一个HTTP服务端

```go
import (
	"fmt"
	"net/http"
)

// net/http包是对net包的进一步封装，专门用来处理HTTP协议的数据
func main() {
	// Handle和HandleFunc函数可以向DefaultServeMux添加处理器。
	//http.Handle("/foo", fooHandler)
	http.HandleFunc("/", sayHello)
	// 使用指定的监听地址和处理器启动一个HTTP服务端,处理器参数通常是nil，这表示采用包变量DefaultServeMux作为处理器。
	err := http.ListenAndServe("127.0.0.1:9090", nil)
	if err != nil {
		fmt.Printf("http server failed, err:%v\n", err)
		return
	}
}

func sayHello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello 美女！")
}
```

要管理服务端的行为，可以创建一个自定义的Server：

```go
s := &http.Server{
    Addr:           ":8080",
    Handler:        myHandler,
    ReadTimeout:    10 * time.Second,
    WriteTimeout:   10 * time.Second,
    MaxHeaderBytes: 1 << 20,
}
log.Fatal(s.ListenAndServe())
```

### 客户端

```go
resp, err := http.Get("http://www.baidu.com/")
...
resp, err := http.Post("http://www.baidu.com/upload", "image/jpeg", &buf)
...
resp, err := http.PostForm("http://www.baidu.com/form",
    url.Values{"key": {"Value"}, "id": {"123"}})
```

程序在使用完response后必须关闭回复的主体：

```go
resp, err := http.Get("http://www.baidu.com/")
if err != nil {
    // handle error
}
defer resp.Body.Close()
body, err := ioutil.ReadAll(resp.Body)
...
```

带参数请求：

```go
-------- GET
apiUrl := "http://127.0.0.1:9090/get"
u, err := url.ParseRequestURI(apiUrl)
if err != nil {
	fmt.Printf("parse url requestUrl failed,err:%v\n", err)
}

// URL param
data := url.Values{}
data.Set("name", "枯藤")
data.Set("age", "18")
u.RawQuery = data.Encode() // URL encode
fmt.Println(u.String())

resp, err := http.Get(u.String())
if err != nil {
	fmt.Println("post failed, err:%v\n", err)
	return
}
defer resp.Body.Close()

b, err := ioutil.ReadAll(resp.Body)
if err != nil {
	fmt.Println("get resp failed,err:%v\n", err)
	return
}
fmt.Println(string(b))
----------- POST
url := "http://127.0.0.1:9090/post"
// 表单数据
contentType := "application/json"
data := `{"name":"张三","age":18}`
resp, err := http.Post(url, contentType, strings.NewReader(data))
if err != nil {
    fmt.Println("post failed, err:%v\n", err)
    return
}
defer resp.Body.Close()

b, err := ioutil.ReadAll(resp.Body)
if err != nil {
    fmt.Println("get resp failed,err:%v\n", err)
    return
}
fmt.Println(string(b))
```

httpClient控制请求头：

```go
request, err := http.NewRequest(http.MethodGet, "http://www.baidu.com", nil)
request.Header.Add("Content-Type", "application/json")

// httpClient
client := http.Client{
    CheckRedirect: func(req *http.Request, via []*http.Request) error {
        fmt.Println("Redirect: ", req)
        return nil
    },
}
resp, err := client.Do(request)
if err != nil {
    panic(err)
}
defer resp.Body.Close()

// httputil简化工作
s, err := httputil.DumpResponse(resp, true)
if err != nil {
    panic(err)
}
fmt.Printf("%s\n", s)
```



