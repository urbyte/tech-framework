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
* 类::实例方法名

> * Lambda体中调用方法的参数列表与返回值类型，要与函数式接口中抽象方法的函数列表和返回值类型保持一致
>
> * 若Lambda参数列表中的第一参数是实例方法的调用者，而第二个参数是实例方法的参数时，可以使用ClassName::method

### 四、构造器引用

格式：

​	ClassName::new

> 注意：需要调用的构造器的参数列表要与函数式接口中的抽象方法的参数列表保持一致

### 五、数组引用

格式：

​	Type[]::new





