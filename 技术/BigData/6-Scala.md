[Scala官网](https://www.scala-lang.org)

Scala是一种静态类型的编程语言，它运行在Java虚拟机（JVM）上。Scala源代码会被编译成字节码，然后可以在JVM上执行。因此，Scala与JVM之间有着密切的关系，它利用JVM的资源管理和优化，与Java互操作，并能够享受到Java生态系统中的丰富库和工具

## IDEA集成Scala

1. maven 中添加 scala 依赖包：

   ```xml
   <!-- scala -->
   <dependency>
       <groupId>org.scala-lang</groupId>
       <artifactId>scala-library</artifactId>
       <version>2.12.18</version>
   </dependency>
   ```

2. IED 插件需要安装`scala`

3. 在项目上右键`Add Frameworks Support`添加scala

4. 在`main`目录下新增`scala`目录，并将其加为`Sources Root`，效果与`java`目录相同

## 基本语法

### 数据类型

```scala
// val 名称[:类型] = xxx 赋值后不可变
val money:Int = 100
// 大部分场景，数据类型是可以不用显示写出，Scala可以自动推导
val age = 18

// var 名称[:类型] = xxx 赋值后是可以改变的
var name = "zhangsan"
name = "lisi"
println(name)
// 最佳实践：优先使用val，不满足时再使用var
```

![数据类型](https://gtw.oss-cn-shanghai.aliyuncs.com/Scala/%E7%BB%9F%E4%B8%80%E7%B1%BB%E5%9E%8B.png)

在Scala中，所有的值都有类型，包括数值和函数

`Any`是所有类型的超类型，也称为顶级类型。它定义了一些通用的方法如`equals`、`hashCode`和`toString`。`Any`有两个直接子类：`AnyVal`和`AnyRef`。

`AnyRef`代表引用类型。所有非值类型都被定义为引用类型。在Scala中，每个用户自定义的类型都是`AnyRef`的子类型。

`Unit`是不带任何意义的值类型，类似Java中的`Void`，Scala所有的函数必须有返回，所以说有时候`Unit`也是有用的返回类型

`Null`是所有引用类型的子类型（即`AnyRef`的任意子类型），只有一个实例值null，可以给`AnyRef`赋值，不能给`AnyVal`赋值

`Nothing`是所有类型的子类型，也称为底部类型。用途之一是给出非正常终止的信号，如抛出异常、程序退出或者一个无限循环（可以理解为它是一个不对值进行定义的表达式的类型，或者是一个不能正常返回的方法）

![类型转换](https://gtw.oss-cn-shanghai.aliyuncs.com/BigData/Scala/%E7%B1%BB%E5%9E%8B%E8%BD%AC%E6%8D%A2.png)

值类型可以按照上面的方向进行隐式转换，如果想强制转换，使用`.toXXX`方式转换为XXX类型

```scala
// 类型转换
val a = 10
// 隐式转换
val b:Long = a
// 显示转换
val numStr = "123"
val num = numStr.toInt

// 字符串插值
val firstName = "John"
val mi = 'C'
val lastName = "Doe"

// 以s开头 拼接的变量用$开头
println(s"Name: $firstName $mi $lastName")
// 任意表达式嵌入字符串中,用花括号包起来
println(s"2 + 2 = ${2 + 2}")

// 多行字符串，包含在三个双引号内
val aa =
  """哈哈哈
    |  你好呀
    |""".stripMargin
```

### 条件分支

在 Scala 中，if 语句也可以作为表达式使用，它可以返回一个值：

```scala
val num = 10
val result = if (num > 0) {
  "Number is positive."
} else {
  "Number is non-positive."
}
println(result)
```

### 循环

```scala
// 中断循环，需要 import scala.util.control.Breaks._ 使用break()

//  break 功能，循环需要breakable{}包裹
var index = 1
breakable{
  while (index < 10) {
    if (index > 5) {
      break()
    }
    println(s"index is $index")
    index += 1
  }
}
println("exit...")

// continue 功能，执行体用breakable{}包裹
var index = 0
while (index < 10) {
  breakable{
    index += 1
    if (index % 2 == 1) {
      println(s"index is $index")
    }else{
      break()
    }
  }
}
println("exit...")
```

for循环：

```scala
// to 是闭区间 [1, 10]
// until [1, 10)
// 1.to(10) 等价 1 to 10, until 同理
// 1.to(10, 2) 步长为2，相当于i += 2，也可用 1 to 10 by 2
// 1 to 10 reverse, [10, 1]
for(i <- 1 to 10) {
  println(s"index is $i")
}

// 遍历集合
val collection = Seq(1, 2, 3, 4, 5)
for (element <- collection) {
  // 循环体中的代码，element 是集合中的每个元素
}

// 双循环
for(i <- 1 to 9; j <- 1 to i) {
  print(s"$j * $i = ${i * j}\t")
  if(j == i) println()
}

// 循环守卫，只打印奇数
for (i <- 1 to  10 if i % 2 == 1) {
  println(s"index is $i")
}

// 循环返回值，yield关键字
val res = for(i <- 1 to 10) yield i * i
println(res)
```

### 方法定义

在 Scala 中，可以使用以下语法定义方法：

```scala
def methodName(param1: Type1, param2: Type2, ...): ReturnType = {
  // 方法体
  // 可以包含多条语句
  // 最后一条语句的结果将作为返回值
}
```

- def：关键字用于定义方法。

- methodName：方法名，可以根据需求自定义。

- param1: Type1, param2: Type2, ...：参数列表【入参】，每个参数由参数名和类型组成，用逗号分隔。

  方法如果没有入参，在定义和调用时可以省略括号 `println(methodName)`

- ReturnType：返回类型，表示方法执行完成后返回的数据类型。

  返回值并不需要使用return进行返回，默认是**方法体内最后一行作为返回值**

#### 默认参数

可以为方法的参数指定默认值，意味着在调用方法时，如果没有提供该参数的值，将会使用默认值。

```scala
def methodName(param1: Type1, param2: Type2 = defaultValue, ...): ReturnType = {
  // 方法体
}

// 使用默认值作为 param2 的值
methodName(value1) 
```

使用带有默认参数的方法，可以选择省略具有默认值的参数。如果省略了该参数，将使用默认值。如果提供了该参数的值，将使用提供的值而不是默认值。

默认参数的方法时，有几个注意事项需要注意：

1. 默认参数的定义顺序：在方法的参数列表中，如果有多个参数具有默认值，那么这些参数必须按照从左到右的顺序依次声明，默认参数不能跳过。即所有没有默认值的参数必须位于有默认值的参数之前

   ```scala
   // 正确的默认参数定义顺序
   def method(param1: Type1, param2: Type2 = defaultValue, param3: Type3 = defaultValue): ReturnType = {
     // 方法体
   }
   
   // 错误的默认参数定义顺序（编译错误）
   def method(param1: Type1 = defaultValue, param2: Type2, param3: Type3): ReturnType = {
     // 方法体
   }
   ```

2. 混合使用默认参数和非默认参数：当方法同时包含默认参数和没有默认值的参数时，可以根据需要选择性地提供参数值。在方法调用中，可以省略具有默认值的参数，但不能省略没有默认值的参数。

   ```scala
   def method(param1: Type1, param2: Type2 = defaultValue, param3: Type3): ReturnType = {
     // 方法体
   }
   
   method(value1, value3) // 正确，省略了具有默认值的 param2
   
   method(value1, value2, value3) // 正确，提供了所有参数的值
   
   method(value1) // 错误，没有为没有默认值的 param3 提供值
   ```

3. 明确指定参数名称：如果希望在方法调用时只提供某些参数的值，而其他参数使用默认值，可以通过在方法调用时明确指定参数名称来实现。

   ```scala
   def method(param1: Type1, param2: Type2 = defaultValue, param3: Type3): ReturnType = {
     // 方法体
   }
   
   method(param1 = value1, param3 = value3) // 仅提供 param1 和 param3 的值，param2 使用默认值
   ```

#### 命名参数

当调用方法时，实际参数可以通过其对应的形式参数的名称来标记：

```scala
def printName(first: String, last: String): Unit = {
  println(first + " " + last)
}

printName("John", "Smith")  // Prints "John Smith"
printName(last = "Smith", first = "John")  // Prints "John Smith"
```

注意使用命名参数时，顺序是可以重新排列的。 但是，如果某些参数被命名了，而其他参数没有，则未命名的参数要按照其方法签名中的参数顺序放在前面

#### 变长参数

定义一个接受变长参数的方法，需要在参数列表中使用 `*` 修饰符，后跟参数的类型。这个参数将被视为一个序列（Seq）或数组，并可以在方法体内以集合的形式使用

```scala
def methodName(args: Type*): ReturnType = {
  // 方法体
  // 在方法体内部，可以将变长参数视为一个集合，并使用集合相关的方法和操作
}

methodName(arg1, arg2, arg3, ...)
// 使用数组或序列传递参数，需要跟特定的 `:_*`
methodName(args: _*) 
```

### 集合

集合架构：Scala的集合框架可以分为可变集合（mutable collections）和不可变集合（immutable collections）两类

- mutable 

  在`scala.collection.mutable`包下，对于集合可以直接修改，不会返回新对象

- immutable 

  在`scala.collection.immutable`包下，对于集合修改后，会返回新对象，不会破坏之前的对象

Scala集合分为三大类：`Seq、Set、Map`

- Array

  ```scala
      /************ 不可变，在scala包下 ************/
      // val 变量名 = new Array[类型](长度)
      val array = new Array[String](5)
      // 也可通过伴生对象apply方法，这种为常用方式
      val numbers: Array[Int] = Array(1, 2, 3, 4, 5)
      // 需要注意的是，不可变数组本身是不可变的，但数组元素可以是可变的对象。因此，如果数组元素是可变对象，仍然可以修改这些对象的状态
      // 不可变数组在Scala中通过Array类定义，创建后长度不可变
      numbers(0) = 0 // Array(0, 2, 3, 4, 5)
  
      // 新增元素，产生新的Array，不破话原有numbers
      val a1 = numbers :+ 100 // Array(0, 2, 3, 4, 5, 100)
      val a2 = 100 +: numbers // Array(100, 0, 2, 3, 4, 5)
  
      val array2 = Array[Int](7, 8)
      // 数组拼接，产生新的Array
      val a3 = numbers ++ array2 // Array(0, 2, 3, 4, 5, 7, 8)
  
      // 遍历，逆序遍历通过 numbers.reverse
      for(value <- numbers.reverse) {
        printf("%s \t", value)
      }
      for(i <- numbers.length - 1 to 0 by -1) {
        printf("%s \t", numbers(i))
      }
  
      println(
        s"""
          |isEmpty is ${numbers.isEmpty}
          |length is ${numbers.length}
          |max is ${numbers.max}
          |min is ${numbers.min}
          |合 is ${numbers.sum}
          |""".stripMargin)
  
      // 转字符串
      println(numbers.mkString) // 02345
      println(numbers.mkString("-")) //0-2-3-4-5
      println(numbers.mkString("<", "-", ">")) //<0-2-3-4-5>
      
      // 转可变
      numbers.toBuffer
  
      /************ 可变，在scala.collection.mutable.ArrayBuffer包下 ************/
      val c = ArrayBuffer[Int]()
  
      // 添加元素，还是之前的数组，并不产生新的数组
      c += 1
      c += (2,3,4) //  ArrayBuffer(1, 2, 3, 4)
      c.insert(0, 9) // ArrayBuffer(9, 1, 2, 3, 4)
      c ++= Array(6,7,8) // ArrayBuffer(9, 1, 2, 3, 4, 6, 7, 8)
      // b2数组加到b1中去，更新b1： b1 ++= b2
      // b2数组加到b1中去，更新b2： b1 ++=: b2
  
      // 删除元素
      c.remove(1) // 删除当前位置元素 ArrayBuffer(9, 2, 3, 4, 6, 7, 8)
      c.remove(1, 2) // 从当前位置删n个 ArrayBuffer(9, 4, 6, 7, 8)
      c.trimEnd(1) // 移除最后n个元素 ArrayBuffer(9, 4, 6, 7)
      c.trimStart(1) // 移除开头n个元素 ArrayBuffer(4, 6, 7)
      c -= 4 // ArrayBuffer(6, 7)
  
      // 转不可变
      c.toArray
  ```

- Set

  ```scala
      /************ 不可变 ************/
      val a = Set(1, 1, 2, 3, 4)  // Set(1, 2, 3, 4)
  
      // 数组、list去重
      array.toSet
  
      /************ 可变 ************/
      val b = scala.collection.mutable.Set(1, 1, 2, 3, 4)
  
      // 元素添加, 也可以使用+=
      b.add(5)
      // 移除，-=
      b. remove(4)
  
      val s1 = scala.collection.mutable.Set(1, 2, 3, 4, 5)
      val s2 = scala.collection.mutable.Set(3, 4, 5, 6, 7)
      // 并
      s1 ++ s2
      s1 union s2
      s1 | s2
      // 交
      s1.intersect(s2)
      s1.&(s2)
      // 差
      s1 -- s2
      s1 diff s2
      s1 &~ s2
  ```

- List

  ```scala
      /************ 不可变 ************/
      val l = List(1, 2, 3, 4, 5)
      // Nil 是个空数组 List()
      val l2 = 1 :: 2 :: 3 :: 4 :: Nil
  
      // 首元素
      l.head
      // 除了首元素的其他元素
      l.tail // List(2, 3, 4, 5)
      // 返回其他元素，不包括最后一个
      l.init // List(1, 2, 3, 4)
      // 最后一个元素
      l.last
      // 获取前两个元素
      l.take(2)
      // 获取后两个元素
      l.takeRight(2)
  
      // 两个List
      l ::: l2 // List(1, 2, 3, 4, 5, 1, 2, 3, 4)
  
      /************ 可变 ************/
      val l3 = ListBuffer[Int]()
  
      // 元素添加依旧可用 +=、-=、++=
      l3 += (1, 2, 3, 4, 5)
  
      // 移除几个元素，返回的是新的数组对象，不是真的移除，copy剩余元素到新数组里
      l3.drop(2) // ListBuffer(3, 4, 5)
      l3.dropRight(1)
  
      // 同样List可以求交、并、差，与Set同理
  
      // 滑动窗口,第一个参数为窗口大小，第二个参数滑动步长
      // ListBuffer(1, 2) --> ListBuffer(3, 4) --> ListBuffer(5)
      val iterator = l3.sliding(2, 2)
      for(ele <- iterator) {
        println(ele)
      }
  ```

- Tuple

  元组在创建后是不可变的，即元组的元素不能被修改。如果需要修改元组中的元素，需要创建一个新的元组

   元组的索引是从1开始的，而不是从0开始

  在较早的版本中，元组长度的限制是22个元素。在新的版本中，Scala提供了更多元素数量的元组支持

  ```scala
      // 封装了一系列元素的集合，数据类型可以是任意的，最多放22个
      val a = (1, 2, 3, 4, 5, "a", "b")
      // Tuple3 只能放3个元素
      val b = Tuple3("zhangsan", 18, "male")
      // ((Int, String), String) = ((1,one),first)
      val c = 1 -> "one" -> "first"
      // 指定类型
      val d:(Int, String) = (10, "world")
  
      // 获取第一个元素
      a._1
  
      // 遍历
      for(i <- 0 until b.productArity) {
        println(b.productElement(i))
      }
  
      // 解构, 可以将元组的元素赋值给多个变量
      val (name, age, sex) = b
      println(name)
  
      // 转列表、数组
      val tuple = (1, 2, 3)
      val list = tuple.productIterator.toList // 返回 List(1, 2, 3)
      val array = tuple.productIterator.toArray // 返回 Array(1, 2, 3)
  
      // 元组拆分, 用于将包含两个元素的元组的列表拆分为两个独立的列表，分别包含了元组中的第一个和第二个元素
      val listOfTuples = List((1, "a"), (2, "b"), (3, "c"))
      val tuples = listOfTuples.unzip // 返回 (List(1, 2, 3), List("a", "b", "c"))
  ```

- Map

  ```scala
      /************ 不可变 ************/
      val map = Map("zhangsan" -> 18, "lisi" -> 20)
  
      map.contains("lisi")
  
      for((k, v) <- map) {
        println(s"key is $k, value is $v")
      }
  
      // map.keys获取所有key, map.values获取所有value
      for (key <- map.keys) {
        // map(key) 获取key对应的值，key 不存在会抛异常
        // map.getOrElse(key, -9999)
        println(s"key is $key, value is ${map(key)}")
      }
  
  
      /************ 可变 ************/
      val m = scala.collection.mutable.Map[String, Int]()
  
      // 新增
      m += ("wangwu" -> 19)
      m.put("zhaoliu", 25)
  
      // 修改
      m("zhaoliu") = 27
      m.update("wangwu", 29)
  
      // 删除
      m -= ("wangwu", "xiaoming")
      m.remove("zhaoliu")
  ```



## 面向对象

### 类定义和使用

```scala
class User {
  // _ 代表占位符意思，必须要明确数据类型，且不能为val类型，默认为0值
  var name:String = _
  // scala.beans.BeanProperty 编译类class中会生成对应get、set方法
  @BeanProperty
  var age:Int = 30
  // 同java中private
  private val sex: String = "man"

  def sayHello(): Unit = {
    println(s"Hello, my name is $name, age is $age, sex is $sex")
  }
}
```

### 构造器

```scala
// 构造器是类定义的一部分，它的参数列表定义了类的成员变量，并且在类实例化时被初始化
class Person(val name: String, var age: Int) {

  val school = "test"
  /**
   * def this 附属构造器
   * 每个附属构造器，首行必须调用已经存在的主构造器或其他附属构造器
   */
  def this(name: String) {
    this(name, 1)  // 调用主构造器
  }

  def this() {
    this("Unknown")  // 调用其他附属构造器
  }

  println(s"Creating a person with name $name and age $age.")
}
```

### 继承

通过使用 `extends` 关键字来继承类，并使用 `super` 关键字来调用父类的构造器，子类的构造器必须调用父类的构造器来初始化父类的成员变量

```scala
class Student(name: String, age: Int, val major: String) extends Person(name, age) {

  // 重写的属性必须是val类型的才行
  override val school: String = "son school"


  // override关键字代表重写
  override def toString: String = s"$name, $age, $school, $major"
}
```

### 抽象类

```scala
/**
 * 使用abstract来定义抽象类
 * 子类重写父类的属性或者方法，override 关键字可加可不加
 */
abstract class AbstractTest {
  // 类的属性也可以没有完整实现
  val name:String
  def speak
}
```

### 伴生类&伴生对象

在 Scala 中，伴生类（Companion Class）和伴生对象（Companion Object）是一对相互关联的概念。它们共享同一个名称，并且可以相互访问对方的私有成员

- 伴生类：伴生类是一个普通的类，它与同名的伴生对象相关联。伴生类可以包含实例方法、属性和构造器。它的实例可以通过实例化来创建。
- 伴生对象：伴生对象是与同名的伴生类关联的一个对象。它在定义时使用 object 关键字。伴生对象可以包含静态方法、静态属性和其他与类相关的属性和方法。与类不同，伴生对象不能被实例化，因为<font color="red">**它是一个单例对象，只有一个实例**</font>。
- 相互访问：**伴生类和伴生对象可以相互访问对方的私有成员**，使得伴生类和伴生对象之间可以实现一种紧密的协作关系。
- 共享名称：伴生类和伴生对象共享同一个名称，这意味着它们必须在同一个源文件中定义，并且名称要完全相同。
- 静态方法和属性：**伴生对象中的方法和属性可以被类名直接调用，就像静态方法和属性一样**。这是因为伴生对象是一个单例对象，它的方法和属性可以在类级别上直接访问。

```scala
// 伴生类
class Person(val name: String, val age: Int)

// 伴生对象
object Person {
  // apply 方法，用于创建 Person 类的实例
  // 提供一种简洁的语法来实例化伴生类对象，就像 val obj = ClassName()
  def apply(name: String, age: Int): Person = new Person(name, age)

  def printPerson(person: Person): Unit = {
    println(s"Name: ${person.name}, Age: ${person.age}")
  }
}

// 没有new 使用伴生对象的 apply 方法创建对象
val person1 = Person("Alice", 25) 
val person2 = new Person("Bob", 30) // 直接实例化伴生类对象

// 调用伴生对象中的方法
Person.printPerson(person1) 
Person.printPerson(person2)

// 访问伴生类对象的属性
println(person1.name) 
println(person2.age)
```

### case class/object

`case class` 和 `case object` 是特殊的类和对象，它们提供了一些方便的功能和语法糖

```scala
case class Person(name: String, age: Int)

case object EmptyList

// 使用示例
val person1 = Person("Alice", 25)
val person2 = Person("Alice", 25)
println(person1) // 输出: Person(Alice,25)
println(person1 == person2) // 输出: true

val newList = EmptyList
println(newList) // 输出: EmptyList
```

- `case class` vs `class`
  1. 不用new
  2. 重写了`toString`、`equals`、`hashCode`方法
  3. 默认实现了序列化接口
  4. 实例是不可变的（immutable），它们的属性值在创建后不可修改
  5. 每个 `case class` 相关联的还有一个自动生成的伴生对象（companion object）。该伴生对象包含了一些与 `case class` 相关的辅助方法，例如工厂方法 `apply` 和解构方法 `unapply`，这些方法对模式匹配非常有用
  6. 模式匹配支持
- `case class` vs `case object`
  1. `case class`修饰的类必须有参数列表
  2. `case object`修饰的对象没有参数列表
  3. `case class`可以创建多个实例，每个实例都是独立的对象，而`case object`只有一个实例，它是全局唯一的

总的来说，`case class`用于创建不可变的类，并经常用于模式匹配，而`case object`用于创建不可变的单例对象，通常用作特定的标识符。

### traits

`trait` 是一种特殊的概念，可以看作是一种接口（interface）或者是一种可混入的特征（mixin）。`trait` 提供了一种多继承的机制，允许类继承多个 `trait`，从而实现了代码的复用和组合

```scala
// 可以包含抽象方法、具体方法和字段的声明，trait 不能直接被实例化
trait Printable {
  def print(): Unit
}

trait Logger {
  // 可以包含具体方法的实现，这些方法可以直接被继承 trait 的类使用或重写
  def log(message: String): Unit = {
    println(s"print Log: $message")
  }
}

trait Showable {
  // 自身类型,指定自身类型为Logger, 实现了在一个trait调用另一个trait功能，任何混入Showable的类都必须混入Logger特质
  // self: Logger =>
  _: Logger =>
  
  def show(): Unit
  
  def printLog(message: String): Unit = {
    // 调用另一个trait
    // self.log(s"thisis $message")
    this.log(s"thisis $message")
  }
}

// 类可以继承多个 trait，从而获得多个 trait 中定义的方法和字段
class MyClass extends Printable with Showable with Logger {
  override def print(): Unit = println("Printing...")
  override def show(): Unit = println("Showing...")
}

val obj = new MyClass()
obj.print()  // 输出: Printing...
obj.show()   // 输出: Showing...
obj.printLog("hello")


class Person(val name: String)
// 动态混入，在运行时将新的特质（trait）添加到对象中，以改变其行
// 如果多个混入特质具有相同的方法签名，最后混入的特质会覆盖之前混入的特质的实现
val person = new Person("John") with Printable {
  def print(): Unit = {
    println(s"name is $name")
  }
}
```

### 包管理

包导入

```scala
// 可以导入特定的成员
import com.example.myapp.{Util1, Util2}

// 导入包时改名字
import com.example.conversions.{A => MyselfA}
val a = new MyselfA()


```

包对象

```scala
// 包名为com.example.myapp
package com.example

// 包对象是一个特殊的对象，它与包具有相同的名称
// package object中定义的成员对包内的所有其他文件可见，并且可以直接访问，无需导入
// 当在package object中定义的成员与包中其他地方的成员名称冲突时，优先选择package object中的定义
package object myapp {
  // 常量、变量、方法和类型别名的定义
}
```

### 类型相关

类型转换：

```scala
class Animal
class Cat extends Animal
// 向上转型
val animal: Animal = new Cat()
// 向下转型,父类型的对象转换为子类型,需要使用asInstanceOf操作符显式转换
val cat: Cat = animal.asInstanceOf[Cat]

// 类型判断,使用isInstanceOf操作符进行类型判断
if (animal.isInstanceOf[Cat]) {
  println("animal is a Cat")
} else {
  println("animal is not a Cat")
}

```

### 枚举

在Scala中，可以通过继承`Enumeration`类和使用`Value`方法来定义枚举常量

```scala
// 定义枚举
object WeekDay extends Enumeration {
  val Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday = Value

  def isWeekend(day: WeekDay.Value): Boolean = {
    // 枚举常量可以使用==运算符进行比较
    day == Saturday || day == Sunday
  }
}

// 访问枚举值
val day = WeekDay.Monday
println(day)

// 迭代枚举值
for (day <- WeekDay.values) {
  println(day)
}
```



## 匹配模式

在Scala中，模式匹配是一种强大的语言特性，它允许根据输入值的不同模式来执行不同的操作。模式匹配可以用于匹配常量、变量、类型、集合等各种数据结构，基本语法（类似Java中的`switch case`）：

```scala
x match {
  case pattern1 => action1
  case pattern2 => action2
  ...
}
```

### 内容/值匹配

```scala
val x = 5
val condition = true
x match {
  case 1 => println("One")
  case 2 => println("Two")
  // 守卫条件是指在模式后添加 if关键字 和 一个布尔表达式 
  case _ if condition => println("Opss..")
  case _ => println("Other")
}
```

### 类型匹配

```scala
val x: Any = "Hello"
x match {
  // name: Type 进行类型匹配
  case s: String => println(s"String: $s")
  case i: Int => println(s"Int: $i")
  case _ => println("Other")
}
```

### Array/List/Tuple匹配

```scala
// Array匹配
val x = Array("zhangsan", "lisi", "wangwu")
x match {
  // 匹配数组只有一个张三
  case Array("zhangsan") => println("Hi, zhangsan")
  // 匹配数组有两个元素
  case Array(x, y) => println(s"Hi, $x and $y")
  // 匹配数组以zhangsan开头，获取其他所有赋值给name name@_*
  case Array("zhangsan", other@_*) => println(s"Hi, zhangsan and ${other.toList}")
  case _ => println("hello everyone")
}

// Tuple匹配
val myTuple = (1, "Hello", true)
myTuple match {
  // 匹配元组中第一个元素为1，且是三元的，否则会异常
  case (1, str, flag) => println(s"Matched Tuple: $str, $flag")
  case _ => println("Not matched")
}

// List匹配
val x = List("zhangsan", "lisi")
x match {
  // 匹配数组只有一个张三
  case "zhangsan" :: Nil => println("Hi, zhangsan")
  // 匹配数组有两个元素
  case x :: y :: Nil => println(s"Hi, $x and $y")
  // 匹配数组以zhangsan开头
  case "zhangsan" :: tail => println("Hi, zhangsan and other")
  case _ => println("hello everyone")
}
```

### case 匹配

```scala
// class 匹配
class Person(val name: String, val age: Int)
object Person {
  def unapply(person: Person): Option[(String, Int)] = {
    if(person != null) {
      Some(person.name, person.age)
    }else {
      None
    }
  }
}
val person = new Person("zhangsan", 20)

person match {
  // 此处调用的是Person伴生对象的unapply方法
  case Person(name, age) => println(s"$name + $age")
  case _ => println("none")
}

// case class 匹配
case class Person(name: String, age: Int)
val person = Person("zhangsan", 20)

person match {
  case Person(name, age) => println(s"$name + $age")
  case _ => println("none")
}
```

### 异常处理

```scala
try {
  val i = 10 / 0
}catch {
  case e:ArithmeticException => println("发生了算数异常" + e.getMessage)
  case e:Exception => println("发生了Exception")
}
```

### 偏函数

```scala
val add = new PartialFunction[Any, Int] {
  // 获取判断为true的元素
  override def isDefinedAt(x: Any): Boolean = isInstanceOf[Int]

  // 对true结果的元素做处理
  override def apply(v1: Any): Int = asInstanceOf[Int]
}

// 偏函数 PartialFunction ，泛型接受一个输入类型，一个输出类型，即偏爱某一部分
// 花括号中的内容，就是 match 花括号中的内容（match需要有返回值才行）
def toChinese: PartialFunction[String, String] = {
  case "China" => "中国"
  case "America" => "美国"
  case _ => "其他"
}
println(toChinese("China"))

// 等同于
val target = "China"
val result = target match {
  case "China" => "中国"
  case "America" => "美国"
  case _ => "其他"
}
println(result)
```



## 函数式编程

### 函数定义和使用

```scala
/**
 * 函数定义方式一：val/var 函数名 = (参数列表) => {函数体}
 * f1: (Int, Int) => Int = $Lambda$1021/729867689@36c7cbe1
 */
val f1 = (a: Int, b: Int) => {a + b}
println(f1(1, 2))

/**
 * 函数定义方式二：val/var 函数名:(入参类型) => 返回值类型 = (入参引用) =>{函数体}
 * f2: (Int, Int) => Int = $Lambda$1032/39642165@7ea71fc2
 */
val f2:(Int, Int) => Int = (a, b) => {a + b}
println(f2(1, 2))
```

方法转函数：

```scala
def add(a: Int, b: Int): Int = {
  a + b
}

/**
 * 方法一：通过空格+下划线，将方法赋值给函数
 */
val addFunc = add _
println(addFunc(1, 2))

/**
 * 方法二：通过类型声明后，指向方法
 */
val addFunc2:(Int, Int) => Int = add
println(addFunc2(1, 2))
```

### 高阶函数

```scala
def add(x: Int, y: Int) = {
  x + y
}
// 接受函数当入参
def method1(a: Int, b: Int, operator: (Int, Int) => Int): Unit = {
  println(operator(a, b))
}

// 传入add方法，是通过方法二的转函数形式，所以可直接传入add
method1(2, 3, add)
// 匿名函数作为参数
method1(2, 3, (x, y) => x * y)
// 第一个_表示第一个参数，第二个_表示第二个参数，每个参数只能用一次
method1(9, 3, _/_)
```

### 柯里化 currying

```scala
def add1(x: Int, y: Int) = x + y
// 柯里化
def add2(x: Int)(y: Int) = x + y

// 柯里化函数可以“部分应用”，即一次不指定全部参数。待定的参数用“占位符”表示
scala> val onePlus = add2(1)_   // “占位符”表示待定的参数
onePlus: Int => Int = <function1>

scala> onePlus(2)
res7: Int = 3
```

### 自定义算子实现高阶函数

```scala
val array = Array(1, 2, 3, 4, 5, 6, 7, 8, 9)

def foreach(array: Array[Int], op: Int => Unit): Unit = {
  for(ele <- array) {
    op(ele)
  }
}
// foreach(array, x => println(x))
// 执行体只有一行，且函数入参与执行体表达式入参一直，可简写
foreach(array, println)

def filter(array: Array[Int], op: Int => Boolean) = {
  for(ele <- array if op(ele)) yield ele
}
filter(array, _ % 2 == 0).foreach(println)

def map(array: Array[Int], op: Int => Int) = {
  for(ele <- array) yield op(ele)
}
map(array, _ * 10).foreach(println)

def reduce(array: Array[Int], op: (Int, Int) => Int) = {
  var sum = 0
  for(ele <- array) {
    sum = op(sum, ele)
  }
  sum
}
println(reduce(array, _ + _))
```

### 高阶算子

```scala
val l = List(1, 2, 3, 4, 5, 6, 7, 8, 9)
val l2 = List(List(1, 2), List(3, 4), List(5, 6))
val l3 = List("Hello World", "Nihao China", "Hello Chinese")

// 对集合每个元素进行一一映射处理
l.map(_ * 2) // List(2, 4, 6, 8, 10, 12, 14, 16, 18)

// 集合元素过滤，留下结果为true的
l.filter(_ % 2 == 0).foreach(println)

// 扁平化处理
l2.flatten // List(1, 2, 3, 4, 5, 6)
l3.map(_.split(" ")).flatten // List(Hello, World, Nihao, China, Hello, Chinese)

// map + flatten 对集合内部元素做处理，将处理后产生的集合整体拉平
l2.flatMap(_.map(_*2)) // List(2, 4, 6, 8, 10, 12)
l3.flatMap(_.split(" ")) // List(Hello, World, Nihao, China, Hello, Chinese)

// 对集合中元素两两操作
l.reduce(_ - _) // -43
l.reduceLeft(_ - _) // -43
l.reduceRight((x, y) => {
  println(s"$x ===== $y")
  x -y
})

// 柯里化方式，也是元素两两操作，第一个参数代表初始值，从初始值开始操作，不再是从集合中的第一个元素开始
l.fold(0)(_-_) // -45

// 分组
l.groupBy(x => if(x % 2 == 0) "偶数" else "奇数") // Map(奇数 -> List(1, 3, 5, 7, 9), 偶数 -> List(2, 4, 6, 8))
// l3.flatMap(_.split(" ")).groupBy(x => x) 结果： Map(Chinese -> List(Chinese), Nihao -> List(Nihao), China -> List(China), Hello -> List(Hello, Hello), World -> List(World))
l3.flatMap(_.split(" ")).groupBy(x => x).map(x => (x._1, x._2.size)) // Map(Chinese -> 1, Nihao -> 1, China -> 1, Hello -> 2, World -> 1)

// mapValues 是对KV形式数据，对value进行映射处理
l3.flatMap(_.split(" ")).groupBy(x => x).mapValues(_.size) // Map(Chinese -> 1, Nihao -> 1, China -> 1, Hello -> 2, World -> 1)

// 排序
l.sorted // 默认排序，数值升序，字符串字典序
l3.sortBy(x => (-x.length, x)) // 自定义排序，先长度倒序，一样字典序
l.sortWith((x, y) => x > y) // 比较排序，降序排
```



## 隐式转换

实现对现有功能的增强

### 隐式转换函数

```scala
class Man(val name:String)
class SuperMan(val name:String) {
  def fly(): Unit = {
    println(s"$name can fly")
  }
}

/**
 * implicit 关键字来定义隐式转换
 * implicit def a2B(a:A):B = new B(a.*)
 */
implicit def man2SuperMan(man: Man):SuperMan = new SuperMan(man.name)

val man = new Man("小灰灰")
// 没有implicit，man无法调用fly，有了implicit才可以调用
man.fly()
```

### 隐式转换类

```scala
class Man(val name:String)
implicit class RichMan(man: Man) {
  def fly(): Unit = {
    println(s"${man.name} can fly")
  }
}

val man = new Man("小灰灰")
// 有了implicit才可以调用
man.fly()
```

### 隐式参数

```scala
// b,c参数都是隐式的
def add2(a: Int)(implicit b: Int, c: Int) = a + b + c

implicit val x = 100
// 隐式参数没有传递时，会从上下文中找隐式变量，找到了用，找不到或找到个数不匹配会报错
println(add2(2)) // 202
```



## 泛型

```scala
class Person
class User extends Person
class Child extends User

def test01[T](t: T): Unit = {
  println(t)
}

// 同Java中 ? extends T
def test02[T <: User](t: T): Unit = {
  println(t)
}

// 不同于Java中 ? super T，什么都可以传
def test03[T >: User](t: T): Unit = {
  println(t)
}

test01(new User)
test02(new Person) // do not conform to method test02's type parameter bounds [T <: User]
test03(11) // 11
```

视图界定

```scala
// <% 视图界定 使用隐式转换对T类型进行增强，如int2Integer，具备了Comparable功能
class MaxValue[T <% Comparable[T]] (a:T, b: T) {
  def compare = if(a.compareTo(b) > 0) a else b
}

println(new MaxValue[Int](2, 3).compare)
```

逆变和协变

```scala
class Person
class User extends Person
class Child extends User

// Java 中泛型具有不变性，如：List<User> users = new ArrayList<User>(); ArrayList<>只能写User，写Person和Child都是不可以的
// + 协变 父到子
class Test1[+User]
val test1:Test1[User] = new Test1[Child]

// - 逆变 子到父
class Test2[-User]
val test2:Test2[User] = new Test2[Person]
```

