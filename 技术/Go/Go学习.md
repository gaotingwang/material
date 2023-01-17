## 基本语法
- 变量

  定义：使用`var`关键字，也可使用`var( )`来集中定义，编译器可自动决定类型，**在函数内定义变量可使用`:=`省略`var`的方式**

  <font color="red">变量类型写在变量名之后</font>

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

- 选择、循环

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

- 函数、指针

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

- 数组、容器、字符串

  - 数组: go一般不直接使用数组

    ```go
    var arr1 [5]int
    arr2 := [3]int{1, 3, 5}
    // 由编译器决定数组大小
    arr3 := [...]int{2, 4, 6}
    // range 关键字可以获取到集合的元素下标和值
    for i, v := range arr3 {
        fmt.Println(i, v)
    }
    ```

  - 切片

    ```go
    arr := [...]int{0, 1, 2, 3, 4, 5, 6, 7}
    s1 := arr[2:6]
    // s2=[5 6], 不会越界，但s1[4]会越界报错
    s2 := s1[3:5]
    
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

  

## 面向对象

go语言仅支持封装，不支持继承和多态，没有class，只有struct

```go
type TreeNode struct {
	Left, Right *TreeNode
	Value       int
}

// 为结构定义方法:显示定义接收者
func (node TreeNode) getNodeValue() int {
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

封装：

- 命名统一采用驼峰形式
- 首字母大写：public
- 首字母小写：private

包管理：

- 每个目录一个包，包名不一定和目录名相同
- main包包含可执行入口
- 为结构定义的方法必须放在**同一包内**，可以是不同的文件

## 面向接口

结构体
duck typing概念
组合思想

## 函数式编程
闭包概念

## 依赖管理

依赖管理的三个阶段：GOPATH、GOVENDOR、go mod

- GOPATH: 会去`$GOPATH`目录下和`$GOROOT`目录下去加载依赖包（可以为项目配置不同的`$GOPATH`，`$GOPATH`下需要有`src`目录）

- GOVENDOR：每个项目有自己的vendor目录，存放第三方库；会有不同的第三方工具来管理依赖包，将需要的包导入到vendor中（类似maven）

- go mod: 由go 命令统一管理，用户不必关心目录结构

  初始化：`go mod init`，会下项目下生成`go.mod`文件，用来记录依赖的第三方包

  增加依赖：`go get`会去加载依赖包到`$GOPATH/pkg/mod`目录下，同时将依赖记录在`go mod`文件中  / 也可以直接import，在build（`go build ./...`）时候会自动修改mod文件

  更新依赖：`go get 库[@version]` ，`go mod tidy`

## 工程化

资源管理，错误处理
测试和文档
性能调优

## 并发编程
goroutine 和 channel
调度器理解