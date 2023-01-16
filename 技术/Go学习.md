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

  - 数组

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

  - Map

  - 字符串

  

## 面向接口
结构体
duck typing概念
组合思想

## 函数式编程
闭包概念

## 工程化
资源管理，错误处理
测试和文档
性能调优

## 并发编程
goroutine 和 channel
调度器理解