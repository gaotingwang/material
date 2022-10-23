lambda `() -> {}` 表达式在Java中，**本质就是实现特定接口（接口中只能有一个待实现方法）的实例**。一个方法参数中需要该接口，在调用这个方法时，就可以直接写入lambda表达式来作为这个接口的实例，最终达到了可以在方法参数中传递方法的效果。最后实现执行体的方法就行，不再关注具体是哪个对象，形式了函数式编程

lambda表达式重点只需与接口方法的入参和出参相匹配，接口方法的内容是在lambda执行体靠调用者自己来实现的，所以入参和出参匹配了，就是相当于对这个接口在进行实现（系统会自己进行类型推断，将lambda表达式匹配成对应接口）



## 函数接口

| 接口                 | 输入参数 | 返回类型 | 说明                         |
| -------------------- | -------- | -------- | ---------------------------- |
| Predicate\<T>        | T        | boolean  | 断言                         |
| Supplier\<T>         | /        | T        | 提供一个数据                 |
| Consumer\<T>         | T        | void     | 消费一个数据                 |
| Function\<T, R>      | T        | R        | 输入T输出R的函数             |
| UnaryOperator\<>     | T        | T        | 一元函数（输入输出类型相同） |
| BiFunction\<T, U, R> | (T, U)   | R        | 2个输入类型的函数            |
| BinaryOperator\<T>   | (T, T)   | T        | 二元函数（输入输出类型相同） |



## 方法引用

当表达式执行体，只有一个函数调用，且**函数参数与表达式入参一致**，可以缩写为方法引用`::`的方式，如：`Consumer<String> consumer = System.out::println;`

引用类型：

```java
class Dog {
    private String name = "大黄";
    private int food = 10;

    public static void bark(Dog dog) {
        System.out.println(dog);
    }

    public Dog() {
    }

    public Dog(String name) {
        this.name = name;
    }

    public int eat(int num) {
        System.out.println("eat " + num);
        this.food -= num;
        return this.food;
    }

    @Override
    public String toString() {
        return this.name;
    }
}
```

- 静态方法引用

  ```java
  Consumer<Dog> consumer = Dog::bark;
  consumer.accept(dog);
  ```

- 实例方法引用

  ```java
  // 非静态方法
  IntUnaryOperator operator = dog::eat;
  System.out.println(operator.applyAsInt(3) + " left");
  
  // 使用类名方式
  // (非静态方法，默认是会把当前实例做为参数传入，名为this，位置放在第一个)
  // 使用类名方式调用成员方法，会显式地在方法前传入this
  // 对象方式的方法引用，不会显式传入this，此处用dog::eat，没有this，就会入参不匹配
  BiFunction<Dog, Integer, Integer> function = Dog::eat;
  System.out.println(function.apply(dog, 3) + " left");
  ```

- 无参构造函数引用

  ```java
  // 相当于没有参数，返回一个Dog对象
  Supplier<Dog> supplier = Dog::new;
  System.out.println(supplier.get());
  ```

- 带参构造函数引用

  ```java
  // 会匹配参数是String的构造函数
  Function<String, Dog> function = Dog::new;
  System.out.println(function.apply("金毛"));
  ```

总体思想：分析一个函数的输入输出，就能找到对应函数接口来进行匹配，就可以使用方法引用了



## 级联表达式和柯里化

```java
// x + y 中的x相当于引用的上层表达式中的变量，跟在表达式执行体中引用其他其他变量是一样的道理
Function<Integer, Function<Integer, Integer>> function = x -> y -> x + y;
// 2 + 3
System.out.println(function.apply(2).apply(3));
```

柯里化：把多个参数的函数转换为只有一个参数的函数，其目的是为了函数标准化（所有函数都只有一个参数，调用比较灵活）

```java
Function<Integer, Function<Integer, Function<Integer, Integer>>> function = x -> y -> z -> x + y + z;
List<Integer> list = Arrays.asList(1, 2, 3);
Function f = function;
for (Integer i : list) {
  	// 只有一个参数
    Object o = f.apply(i);
    if (o instanceof Function) {
        f = (Function) o;
    } else {
        System.out.println("调用结果为：" + o);
    }
}
```



## Stream流

Stream 是一个高级的迭代器，不是一个数据结构，不是一个集合，不会存放数据，关注点在怎么对数据高效处理（按照流水线方式，一端输入数据，末端输出结果，中间会做一系列操作）

### 创建

|          | 相关方法                                            |
| -------- | --------------------------------------------------- |
| 集合     | Collection.stream/parallelStream                    |
| 数组     | Arrays.stream                                       |
| 数字     | IntStream.range/iterate.. <br />new Random().ints() |
| 自己创建 | Stream.generate/iterate..                           |

注意：IntStream / LongStream 并不是Stream的子类，要转换为Stream时，需要通过`.boxed()`方法进行装箱操作

### 中间操作

|                                    | 相关方法                                                     |
| ---------------------------------- | ------------------------------------------------------------ |
| 无状态操作<br />（不依赖其他元素） | \<R> Stream\<R> map(Function\<? super T, ? extends R> mapper); <br /> mapToXXX |
|                                    | \<R> Stream\<R> flatMap(Function\<? super T, ? extends Stream\<? extends R>> mapper); // 得到所有A元素里的所有B属性集合<br />flatMapToXXX |
|                                    | Stream\<T> filter(Predicate\<? super T> predicate);          |
|                                    | Stream\<T> peek(Consumer\<? super T> action);                |
|                                    | unordered                                                    |
| 有状态操作<br />（依赖其他元素）   | distinct                                                     |
|                                    | sorted                                                       |
|                                    | limit / skip                                                 |

### 终止操作

若不进行终止操作，流是不会执行的

|            | 相关方法                             |
| ---------- | ------------------------------------ |
| 非短路操作 | forEach / forEachOrdered             |
|            | collect / toArray                    |
|            | reduce // 减少，将多个元素合并成一个 |
|            | min / max / count                    |
| 短路操作   | firstFind / findAny                  |
|            | allMatch / anyMatch / noneMatch      |

### 运行机制

1. 所有操作都是链式调用，一个元素只迭代一次
2. 每一个中间操作返回一个新的流，流里面有一个属性sourceStage指向同一个位置，即链表的头Head
3. Head中会存放nextStage，Head -> nextStage -> nextStage ...-> null
4. 有状态操作会把无状态操作阶段单独处理（等之前的无状态操作全部完成，才会进行有状态操作）
5. 并行环境下，有状态的中间操作不一定能并行操作（会有相互依赖）
6. `parallel / sequential `也是中间操作，返回Stream流，但他们不创建流，只修改Head的并行标志(`parallel `)



## Reactive Stream

Java9引入的概念，响应式流，与8中的Stream没有任何关联

### 背压 backprase

是一个交互反馈，发布者与订阅者的互动，订阅者可以告诉发布者需要多少数据，起到调节数据流量作用

### 主要接口

Flow :

- `Publisher<T>`：数据发布者（上游）
- `Subscriber<T>`：数据订阅者（下游）
- `Subscription`：订阅信号操作，实现背压的关键，可以控制请求上游元素数量，请求停止发送数据清除资源
- `Processor<T,R>`：Publisher和Subscriber的结合体，起承上启下作用

响应式流，主要是在消费到一条数据时，根据条件来决定继续消费`subscription.request(long n)`控制消费频率，还是取消消费`subscription.cancel()`



