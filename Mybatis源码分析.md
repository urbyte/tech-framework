# Mybatis源码分析

## 解析配置文件Configuration

### 入口类`SqlSessionFactoryBuilder`

[mybatis源码解析](https://blog.csdn.net/nmgrd/article/details/54608702)

* db.properties文件

```properties
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/spring_mybatis
jdbc.name=root
jdbc.password=root
```

* mybatis-config.xml配置文件如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 1、mybatis使用properties来引入外部properties配置文件的内容
             resource 引入类路径下资源
             url 引入网络路径或磁盘路径下资源 -->
    <properties resource="db.properties"></properties>
    
    <!--设置mybatis日志类型-->
    <settings>
        <!-- 全局映射器启用缓存:启用二级缓存 -->
        <setting name="cacheEnabled" value="true"/>
        <!-- 配置默认的执行器。SIMPLE 执行器没有什么特别之处。REUSE 执行器重用预处理语句。BATCH 执行器重用语句和批量更新 -->
        <setting name="defaultExecutorType" value="REUSE" />
        <!-- 全局启用或禁用延迟加载。当禁用时，所有关联对象都会即时加载。 -->
        <setting name="lazyLoadingEnabled" value="false" />
        <setting name="aggressiveLazyLoading" value="true" />
         <!--当没有为参数提供特定的 JDBC 类型时，为空值指定 JDBC 类型。 某些驱动需要指定列的 JDBC 类型，多数情况直接用一般类型即可，比如 NULL、VARCHAR 或 OTHER。-->
        <setting name="jdbcTypeForNull" value="NULL"/>
        <!-- 日志框架LOG4J2-->
        <setting name="logImpl" value="LOG4J2"/>
    </settings>
    
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC" />
            <!-- 配置数据库连接信息 -->
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}" />
                <property name="url" value="${jdbc.url}" />
                <property name="username" value="${jdbc.name}" />
                <property name="password" value="${jdbc.password}" />
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="com/pjf/mybatis/mapper/hotelMapper.xml"></mapper>
    </mappers>
</configuration>
```

* 解析配置文件，创建session会话：

```java
//mybatis配置文件
String resource = "mybatis-config.xml";
//通过流将配置文件加载进来
InputStream inputStream = Resources.getResourceAsStream(resource);
//通过配置文件创建会话工厂
SqlSessionFactory sessionFactory = new SqlSessionFactoryBuilder.build(inputStream);
//通过会话工厂获取会话
SqlSession session = sessionFactory.openSession();
```

#### 使用XPath解析mybatis-config.xml配置文件

* 通过`new XMLConfigBuilder(inputStream, environment, properties)`构造一个`XMLConfigBuilder`对象，调用`parse()`解析xml配置文件中的`configuration`节点，返回`Configuration`实例对象

- 解析`settings`节点，并通过反射校验`setting`的name属性对应的key是否有改`setter`方法，如：`cacheEnabled`在`Configuration`类里有`setCacheEnabled`方法

- 解析`typeAliases`节点，首先判断package，扫描包下的类，注册到`TypeAliasRegistry` 类的`TYPE_ALIASES`字段里；然后处理`typeAlias`的type和name，也注册到`TYPE_ALIASES`字段里

  ```xml
  <typeAliases>
  	<!-- 单个别名定义 -->
  	<typeAlias alias="user" type="com.urbyte.pojo.User"/>
  	<!-- 批量别名定义，扫描整个包下的类，别名为类名（首字母大小写都可以） -->
  	<package name="com.urbyte.pojo"/>
  </typeAliases>
  ```

- 解析`plugins`节点，解析链接器，并加入到`InterceptorChain`类的`interceptors`字段里

  ```xml
  <plugins>
  	<plugin interceptor="com.urbyte.mybatis3.interceptor.SQLStatsInterceptor">
          <property name="dialect" value="mysql" />
      </plugin>
  </plugins>
  ```

- 解析`objectFactory`节点，通过`type`属性值生成`ObjectFactory`对象工厂类，设置到`Configuration`的`objectFactory`局部变量里

  ```xml
  <objectFactory type="org.mybatis.example.ExampleObjectFactory">
  	<property name="someProperty" value="100"/>
  </objectFactory>
  ```

- 解析`objectWrapperFactory`节点，通过`type`属性值生成`ObjectWrapperFactory`对象包装工厂类，设置到`Configuration`的`objectWrapperFactory`局部变量里

  ```xml
  <objectWrapperFactory type="com.urbyte.mybatis.MapWrapperFactory"/>
  ```

- 解析`reflectorFactory`节点

- 解析`environments`节点：解析子节点`transactionManager`返回工厂类`TransactionFactory`，解析子节点`dataSource`返回工厂类`DataSourceFactory`，通过建造者模式构建`Environment`对象

- 解析`databaseIdProvider`节点，通过`type`属性值查询`TypeAliasRegistry` 类的`TYPE_ALIASES`局部变量里的`class`，通过`newInstance()`方法返回`DatabaseIdProvider`实例对象；通过`Environment`对象的`dataSource`找到`databaseId`，设置到`Configuration`的`databaseId`局部变量里

  > 注意：sql xml文件中需要增加`databaseId`属性值，如：
  >
  > ```xml
  > <insert id="insert" parameterType="com.urbyte.dao.entity.XXX" databaseId="db2">
  > </insert>
  > ```

  ```xml
  <databaseIdProvider type="VENDOR">
  	<property name="SQL Server" value="sqlserver"/>
  	<property name="DB2" value="db2"/>        
  	<property name="Oracle" value="oracle" />
  </databaseIdProvider>
  ```

- 解析`typeHandlers`节点，首先处理`package`，扫描包下的类；然后处理`typeHandler`，处理`javaType`是从`TypeAliasRegistry#TYPE_ALIASES`中获取`Class<?>`，`jdbcType`属性值为大写，通过枚举类`JdbcType`返回对应的枚举值，处理`handler`是`TypeAliasRegistry#TYPE_ALIASES`中获取`Class<?>`或直接用`Class.forName`加载`Class<?>`；最后注册到`TypeHandlerRegistry#TYPE_HANDLER_MAP`类型为`Map<Type, Map<JdbcType, TypeHandler<?>>>`，`Type`为`JdbcType`，以及注册到`TypeHandlerRegistry#ALL_TYPE_HANDLERS_MAP`类型为`Map<Class<?>, TypeHandler<?>>`，`key`为`handler`对应的值

  ```xml
  <typeHandlers>
        <!-- 
            当配置package的时候，mybatis会去配置的package扫描TypeHandler
            <package name="com.urbyte.demo"/>
         -->
  
        <!-- handler属性直接配置我们要指定的TypeHandler -->
        <typeHandler handler=""/>
  
        <!-- javaType 配置java类型，例如String, 如果配上javaType, 那么指定的typeHandler就只作用于指定的类型 -->
        <typeHandler javaType="" handler=""/>
  
        <!-- jdbcType 配置数据库基本数据类型，例如VARCHAR, 如果配上jdbcType, 那么指定的typeHandler就只作用于指定的类型  -->
        <typeHandler jdbcType="" handler=""/>
  
        <!-- 也可两者都配置 -->
        <typeHandler javaType="" jdbcType="" handler=""/>
    </typeHandlers>
  ```

#### 解析mapper文件 `XMLConfigBuilder#mapperElement` 

* 解析`mappers`节点，首先处理`package`，扫描包下的类（必须为接口），注册到`MapperRegistry#knownMappers`类型为`Map<Class<?>, MapperProxyFactory<?>>`，`key`为接口的`class`，`value`为`MapperProxyFactory`映射器代理工厂，此时并没有产生代理类；

* 接下来处理`resource`，加载`Mapper.xml`文件：配置`namespace`、配置`cacah-ref`、配置`cache`、配置`parameterMap`、配置`resultMap`、配置sql(定义可重用的 SQL 代码段)、配置`select|insert|update|delete`：通过建造者模式添加到`Configuration#mappedStatements`类型为`Map<String, MappedStatement>`

  ```xml
  <mappers>
  	<!-- 第一种方式：通过resource指定 -->
  	<mapper resource="com/urbyte/dao/UserDao.xml"/>
      
  	<!-- 第二种方式， 通过class指定接口，进而将接口与对应的xml文件形成映射关系
               不过，使用这种方式必须保证 接口与mapper文件同名(不区分大小写)， 
               这里接口是UserDao,那么意味着mapper文件为UserDao.xml 
  	<mapper class="com.urbyte.dao.UserDao"/>
      -->
      <!-- 第三种方式，直接指定包，自动扫描，与方法二同理 
      <package name="com.urbyte.dao"/>
      -->
      <!-- 第四种方式：通过url指定mapper文件位置
      <mapper url="file:///var/mappers/UserDao.xml"/>
      -->
  </mappers>
  ```

## 创建会话SqlSession

### 通过SqlSessionFactory工厂类产生SqlSession

* `sqlSession`持有三个对象`configuration`、`executor`、`autoCommit`
* 通过`configuration`的`environment`获取`TransactionFactory`事务工厂类并产出一个事务；通过事务生成一个`Executor`执行器，并实例化一个`DefaultSqlSession`

## mybatis 缓存

### 一级缓存





## spring与mybatis整合

>* **InitializingBean**，一个类实现了该接口，当spring启动初始化bean时，会自动调用afterPropertiesSet方法；如果同时在配置文件中指定了init-method，系统则是先调用afterPropertiesSet方法，然后在反射调用init-method中指定的方法
>
>  `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#invokeInitMethods`
>
>* **DisposableBean**，
>
>* **ApplicationContextAware**，某个bean实现了该接口，Spring容器会在创建该Bean之后，自动调用该Bean的setApplicationContextAware()方法，调用该方法时，会将容器本身ApplicationContext对象作为参数传给该方法
>
>* [**ApplicationListener**](https://blog.csdn.net/ilovejava_2010/article/details/7953419)，在上下文中部署一个实现了ApplicationListener接口的bean,
>
>  那么每当在一个ApplicationEvent发布到 ApplicationContext时， 这个bean得到通知。其实这就是标准的Oberver设计模式
>
>* **BeanNameAware**，Bean获取自己在BeanFactory配置中的名字

