### Lambda表达式语法

> Lambda表达式分成两部分：
>
> 左侧：Lambda表达式的参数列表
>
> 右侧：Lambda表达式中所需执行的功能，即Lambda体

#### 语法格式一：无参数，无返回值

```Java
() -> System.out.println("Hello Lambda!");
```

#### 语法格式二：有一个参数，并且无返回值

```java 
(x) -> System.out.println(x);
```

#### 语法格式三： 若只有一个参数，小括号可以省略不写

```Java
x -> System.out.println(x);
```

#### 语法格式四： 有两个以上的参数，有返回值，并且Lambda体中有多条语句

```Java
Comparator<Integer> com = (x, y) -> {
    System.out.println("函数式接口");
    return Integer.compare(x, y);
}
```

#### 语法格式五：若Lambda体中只有一个语句，return和大括号都可以省略不写

```java
Comparator<Integer> com = (x, y) -> Integer.compare(x, y);
```

#### 语法格式六：Lambda表达式的参数列表的数据类型可以省略不写，因为JVM编译器通过上下文推断出数据类型，即“类型推断”

```java
(Integer x, Integer y) -> Integer.compare(x, y);
```

### 二、Lambda表达式需要“函数式接口”的支持

> 函数式接口：接口中只有一个抽象方法的接口，称为函数式接口。可以使用注解@FunctionalInterface修饰

#### Java 8 内置的四大核心函数式接口

```Java
Comsumer<T> : 消费型接口
    void accept(T t);
Supplier<T> : 供给型接口
    T get();
Function<T, R> : 函数型接口
    R apply(T t);
Predicate<T> : 断言型接口
    boolean test(T t);
```

### 三、方法引用

> 方法引用：若Lambda体中的内容有方法已经实现了，我们可以使用“方法引用”，可以理解为方法引用是Lambda表达式的另外一种表现形式

主要有三种语法格式：

* 对象::实例方法名
* 类::静态方法名
* 类::实例方法名（对象方法引用）

> * 对象::实例方法名、类::静态方法名：
>
>   Lambda体中调用方法的参数列表与返回值类型，要与函数式接口中抽象方法的函数列表和返回值类型保持一致
>
> * 类::实例方法名（对象方法引用）：
>
>   若Lambda参数列表中的第一参数是实例方法的调用者，而第二个参数是实例方法的参数时，可以使用ClassName::instanceMethod

### 四、构造器引用

格式：

​	ClassName::new

> 注意：需要调用的构造器的参数列表要与函数式接口中的抽象方法的参数列表保持一致

### 五、数组引用

格式：

​	Type[]::new

### Stream操作

> Stream操作：
>
> * 创建Stream
> * 中间操作
> * 终止操作

#### 创建Stream

* 通过Collection系列集合提供的stream()或parallelStream()

  ```java 
  List<String> list = new ArrayList<>();
  Stream<String> stream1 = list.stream();
  ```

* 通过Arrays中的静态方法stream()获取数组流

  ```java
  Employee[] emps = new Employee[10];
  Stream<Employee> stream2= Arrays.stream(emps);
  ```

* 通过stream类中的静态方法of()

  ```java
  Stream<String> stream3 = Stream.of("a","b","c");
  ```

* 创建无限流，迭代

  ```java
  Stream<Integer> stream4 = Stream.iterate(0, (x) -> x + 2);
  stream4.limit(10).forEach(System.out::println);
  ```

* 创建无限流，生成

  ```java
  Stream.generate(() -> Math.random())
      .limit(5)
      .forEach(System.out::println);
  ```

#### 中间操作

##### 筛选与切片

* filter--接收Lambda，从流中排除某些元素
* limit--截断流，使其元素不超过给定数量
* skip(n) -- 跳过元素，返回一个扔掉了前n个元素的流，若流中元素不足n个，则返回一个空流。与limit(n)互补
* distinct--筛选，通过流所生成元素的hashCode()和equals()去除重复元素

##### 映射

* map -- 接收Lambda，将元素转换成其他形式或提取信息。接收一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素
* flatMap -- 接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有的流连接成一个流

##### 排序

* sorted -- 自然排序（Comparable）
* sorted(Comparator com) --定制排序（Comparator）

#### 终止操作

##### 查找与匹配

* allMatch -- 检查是否匹配所有元素
* anyMatch -- 检查是否至少匹配一个元素
* noneMatch -- 检查是否没有匹配所有元素
* findFirst -- 返回第一个元素
* findAny -- 返回当前流中的任意元素
* count -- 返回流中元素的总个数
* max -- 返回流中最大值
* min -- 返回流中最小值

##### 规约与收集

* 规约：reduce(T identity, BinaryOperator bo)、reduce(BinaryOperator bo)，将流中元素反复结合起来，得到一个值。
* 收集：collect -- 将流转换为其他形式，接收一个Collector接口的实现，用于给Stream中元素做汇总的方法

### 并行流与顺序流

* 并行流把一个内容分成多个数据块，并用不同的线程分别处理每个数据块的流
* 并行流：parallel()，顺序流：sequential()

### Optional类

> Optional<T> 类是一个容器类，代表一个值存在或不存在，原来用null表示一个值不存在，现在Optional更好表达这个概念，并且可以避免空指针异常。

#### 常用方法：

* Optional.of(T t) ：创建一个Optional实例
* Optional.empty()：创建一个空的Optional实例
* Optional.ofNullable(T t)：若t不为null，创建Optional实例，否则创建空实例
* isPresent()：判断是否包含值
* orElse(T t)：如果调用对象包含值，返回该值，否则返回t
* orElseGet(Supplier s)：如果调用对象包含值，返回该值，否则返回s获取的值
* map(Function f)：如果有值对其处理，并返回处理后的Optional，否则返回Optional.empty()
* flatMap(Function mapper)：与map类似，要求返回值必须是Optional