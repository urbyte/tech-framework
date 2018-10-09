# Mybatis源码分析

> Mybatis支持动态 SQL 、ORM 映射和提供一级二级缓存等常用功能，支持 XML 和注解两种配置方式，屏蔽了近乎所有的 JDBC 代码、 参数设置、结果集处理等。
>
> 源码分析基于mybatis版本3.2.x

> Mybatis涉及到的设计模式：
>
> * 工厂模式：SqlSessionFactory、DataSourceFactory、MapperProxyFactory
> * 建造者模式：SqlSessionFactoryBuilder
> * 动态代理模式：MapperProxy
> * 装饰器模式：CachingExecutor
> * 适配器模式：日志模块
> * 单例模式：VFS
> * 策略模式：

## 配置文件Configuration

> `org.apache.ibatis.session.Configuration`，MyBatis全局配置信息类

### 基本结构

* configuration -- 根元素
  - properties —— 定义配置外在化
  - settings —— 一些全局性的配置
  - typeAliases —— 为一些类定义别名
  - typeHandlers —— 定义类型处理，也就是定义java类型与数据库中的数据类型之间的转换关系
  - objectFactory —— 对象工厂
  - plugins —— Mybatis的插件，插件可以修改Mybatis内部的运行规则
  - environments —— 配置Mybatis的环境
    - environment
      - transactionManager —— 事务管理器
      - dataSource —— 数据源
  - databaseIdProvider
  - mappers —— 指定映射文件或映射类

### 入口类`SqlSessionFactoryBuilder`

> `org.apache.ibatis.session.SqlSessionFactory`接口，操作SqlSession的工厂接口，默认实现类是`DefaultSqlSessionFactory`

[mybatis源码解析](https://blog.csdn.net/nmgrd/article/details/54608702)、[mybatis技术内幕](https://my.oschina.net/zudajun?tab=newest&catalogId=3532897)

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

### 使用XPath解析mybatis-config.xml配置文件

* 通过`new XMLConfigBuilder(inputStream, environment, properties)`构造一个`XMLConfigBuilder`对象
  * `XMLConfigBuilder(XPathParser parser, String environment, Properties props)`构造函数，调用父类构造，参数直接实例`Configuration`
  * 调用`parse()`解析xml配置文件中的`configuration`节点，返回`Configuration`实例对象

```java
public Configuration() {
    typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
    typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);

    typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class);
    typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
    typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class);

    typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class);
    typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
    typeAliasRegistry.registerAlias("LRU", LruCache.class);
    typeAliasRegistry.registerAlias("SOFT", SoftCache.class);
    typeAliasRegistry.registerAlias("WEAK", WeakCache.class);

    typeAliasRegistry.registerAlias("DB_VENDOR", VendorDatabaseIdProvider.class);

    typeAliasRegistry.registerAlias("XML", XMLLanguageDriver.class);
    typeAliasRegistry.registerAlias("RAW", RawLanguageDriver.class);

    typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
    typeAliasRegistry.registerAlias("COMMONS_LOGGING", JakartaCommonsLoggingImpl.class);
    typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
    typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
    typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
    typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
    typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);

    typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
    typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);

    languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
    languageRegistry.register(RawLanguageDriver.class);
  }
```

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

* `typeHandler`使用：TODO

* 解析`mappers`节点，首先处理`package`，扫描包下的类（必须为接口），注册到`MapperRegistry#knownMappers`类型为`Map<Class<?>, MapperProxyFactory<?>>`，`key`为接口的`class`，`value`为`MapperProxyFactory`映射器代理工厂，此时并没有产生代理类；

* 接下来处理`resource`，加载`Mapper.xml`文件：

  ```xml
  <mappers>
  	<!-- 第一种方式：通过resource指定 -->
  	<mapper resource="com/urbyte/dao/UserDao.xml"/>
      
  	<!-- 
  		第二种方式， 通过class指定接口，进而将接口与对应的xml文件形成映射关系
          不过，使用这种方式必须保证 接口与mapper文件同名(不区分大小写)， 
          这里接口是UserDao,那么意味着mapper文件为UserDao.xml 
  		<mapper class="com.urbyte.dao.UserDao"/>
      -->
      <!-- 
  		第三种方式，直接指定包，自动扫描，与方法二同理 
      	<package name="com.urbyte.dao"/>
      -->
      <!-- 
  		第四种方式：通过url指定mapper文件位置
      	<mapper url="file:///var/mappers/UserDao.xml"/>
      -->
  </mappers>
  ```

### 解析mapper文件 `XMLMapperBuilder#configurationElement` 

* 配置`namespace`、配置`cacah-ref`、配置`cache`、配置`parameterMap`、配置`resultMap`、配置sql(定义可重用的 SQL 代码段)、配置`select|insert|update|delete`：通过建造者模式添加到`Configuration#mappedStatements`类型为`Map<String, MappedStatement>`
* 解析`select|insert|update|delete`节点用`XMLStatementBuilder#parseStatementNode`，通过助手类`MapperBuilderAssistant#addMappedStatement`将二级缓存`currentCache`组装到`MappedStatement`对象里，构造一个`MappedStatement`对象，这里用到建造者模式。

```java 
//建造者模式
MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType);
statementBuilder.resource(resource);
statementBuilder.fetchSize(fetchSize);
statementBuilder.statementType(statementType);
statementBuilder.keyGenerator(keyGenerator);
statementBuilder.keyProperty(keyProperty);
statementBuilder.keyColumn(keyColumn);
statementBuilder.databaseId(databaseId);
statementBuilder.lang(lang);
statementBuilder.resultOrdered(resultOrdered);
statementBuilder.resulSets(resultSets);
setStatementTimeout(timeout, statementBuilder);

setStatementParameterMap(parameterMap, parameterType, statementBuilder);

setStatementResultMap(resultMap, resultType, resultSetType, statementBuilder);
/**
currentCache 为二级缓存，根据<cache eviction="LRU" flushInterval="60000" size="512" readOnly="true"/>，eviction属性值对应的缓存回收策略，详情见二级缓存
*/
setStatementCache(isSelect, flushCache, useCache, currentCache, statementBuilder);

MappedStatement statement = statementBuilder.build();

configuration.addMappedStatement(statement);
return statement;
```

## 动态SQL原理

> 处理动态SQL入口：`XMLMapperBuilder`类的`sqlElement`和`buildStatementFromContext`方法

### 配置sql代码片段

* `XMLMapperBuilder#sqlElement`循环处理/mapper/sql节点，*用namespace+id作为key，XNode的context作为value*，存入`Map<String, XNode> sqlFragments`中

  > 注意：sqlFragments包括相同的key，原XNode的context属性databaseId不为空，则直接不替换原值，否则覆盖原值

  ```java
  private void sqlElement(List<XNode> list, String requiredDatabaseId) throws Exception {
      for (XNode context : list) {
          String databaseId = context.getStringAttribute("databaseId");
          String id = context.getStringAttribute("id");
          //将id增加namespace空间，即id = namespace + id
          id = builderAssistant.applyCurrentNamespace(id, false);
          //将sql片段放入hashmap,context为：<sql id=""> xxx </sql>
          if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
              sqlFragments.put(id, context);
          }
      }
  }
  
  private boolean databaseIdMatchesCurrent(String id, String databaseId, String requiredDatabaseId) {
      if (requiredDatabaseId != null) {
          //requiredDatabaseId与databaseId相同，则直接返回true，无论是否存在相同的id直接用新的SQL
          if (!requiredDatabaseId.equals(databaseId)) {
              return false;
          }
      } else {
          if (databaseId != null) {
              return false;
          }
          //判断是否存在相同的id
          if (this.sqlFragments.containsKey(id)) {
              XNode context = this.sqlFragments.get(id);
              //原sql节点有databaseId，则false，不覆盖；否则返回true，覆盖老的sql
              if (context.getStringAttribute("databaseId") != null) {
                  return false;
              }
          }
      }
      return true;
  }
  ```

### 配置select|insert|update|delete语句

* `XMLMapperBuilder#buildStatementFromContext`循环处理mapper下的select|insert|update|delete节点，实例化`XMLStatementBuilder`类，调用方法`XMLStatementBuilder#parseStatementNode`解析相应节点

* fetchSize：批量返回的结果行数
* timeout：超时时间
* parameterType：参数类型，如果值为class（比如model对象），直接用`Resources.classForName(class)`，因此可以直接使用`java.lang.Integer`、`java.lang.Long`等类型；如果值为typeAlias时，通过`TypeAliasRegistry#TYPE_ALIASES`获取Class<T>，该Class是在解析configuration配置时已经装配好，详见代码：`XMLConfigBuilder#typeAliasesElement`

```java
private void typeAliasesElement(XNode parent) {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            if ("package".equals(child.getName())) {
                String typeAliasPackage = child.getStringAttribute("name");
                //扫描包下的类，然后注册别名(有@Alias注解则用，没有则取类的simpleName)
                configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
            } else {//处理配置typeAlias
                String alias = child.getStringAttribute("alias");
                String type = child.getStringAttribute("type");
                try {
                    Class<?> clazz = Resources.classForName(type);
                    //根据Class名字来注册类型别名
                    //（二）调用TypeAliasRegistry.registerAlias
                    if (alias == null) {
                        //alias可以省略
                        typeAliasRegistry.registerAlias(clazz);
                    } else {
                        typeAliasRegistry.registerAlias(alias, clazz);
                    }
                } catch (ClassNotFoundException e) {
                    throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
                }
            }
        }
    }
}
```

```xml
<!-- typeAlias配置 -->   
<typeAliases>
  <typeAlias alias="user" type="com.urbyte.entity.User"/>
  <typeAlias alias="userLog" type="com.urbyte.entity.UserLog"/>
</typeAliases>
<!-- package配置 -->   
<typeAliases>
  <package name="com.urbyte.entity"/>
</typeAliases>
```

* resultMap：引用外部返回结果
* resultType：返回类型，与parameterType类似
* lang：脚本语言，值为XML、RAW，不设置默认为用`XMLLanguageDriver`

```java
//XMLLanguageDriver处理xml的sql，RawLanguageDriver处理静态sql
typeAliasRegistry.registerAlias("XML", XMLLanguageDriver.class);
typeAliasRegistry.registerAlias("RAW", RawLanguageDriver.class);

//默认注册XMLLanguageDriver
languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
languageRegistry.register(RawLanguageDriver.class);
```

* resultSetType：结果集类型，值为：FORWARD_ONLY|SCROLL_SENSITIVE|SCROLL_INSENSITIVE，默认为FORWARD_ONLY

       FORWARD_ONLY：结果集的游标只能向下滚动

       SCROLL_INSENSITIVE：结果集的游标可以上下移动，当数据库变化时，当前结果集不变 

       SCROLL_SENSITIVE：返回可滚动的结果集，当数据库变化时，当前结果集同步改变

* statementType：语句类型，值为：STATEMENT|PREPARED|CALLABLE，默认为PREPARED

* sqlCommandType：sql命令类型，获取select|insert|update|delete节点名称判断是否为select类型，如果是，isSelect为true，否则为false
* flushCache：是否刷新缓存标识，默认为!isSelect
* useCache：是否缓存select结果标识，默认为isSelect
* resultOrdered：针对嵌套结果 select 语句适用：如果为 true，就是假设包含了嵌套结果集或是分组了，这样的话当返回一个主结果行的时候，就不会发生有对前面结果集的引用的情况。这就使得在获取嵌套的结果集的时候不至于导致内存不够用。默认值：`false`

### 解析include代码片段

* 实例化`XMLIncludeTransformer` XML include转换器类，调用`XMLIncludeTransformer#applyIncludes`方法处理include
* 首先处理元素节点，遍历元素节点下的子节点
* 然后处理include节点，通过refid+namespace生成ID，从`Map<String, XNode> sqlFragments`获取sql片段节点



## 事务Transaction





## 类型处理器TypeHandler





## 创建会话SqlSession

> `org.apache.ibatis.session.SqlSession`接口，执行sql，管理事务的接口，默认实现类是`DefaultSqlSession`

### 通过SqlSessionFactory工厂类产生SqlSession

* `sqlSession`持有三个对象`configuration`、`executor`、`autoCommit`

* 通过`configuration`的`environment`获取`TransactionFactory`事务工厂类并产出一个事务；通过事务生成一个`Executor`执行器，并实例化一个`DefaultSqlSession`

  * `org.apache.ibatis.session.Configuration#newExecutor`，如果`cacheEnabled`为true，则实例化`CachingExecutor`对象，即：开启二级缓存，二级缓存用`CachingExecutor`执行器，用到装饰器模式

  ```java
  if (cacheEnabled) {
  	executor = new CachingExecutor(executor);
  }
  ```

>* SqlSession 的实例不是线程安全的,是不能被共享的，每个线程都应该有自己的 SqlSession 实例，在没有Transaction的情况下生命周期：请求或方法级别，在有Transaction情况下，是Transaction范围内的
>
>* SqlSessionFactory生命周期：应用，是单例与工厂模式，如Spring来生成该实例
>* SqlMapper创建绑定映射语句的接口，其实例从SqlSession获得，因此生命周期与SqlSession相同，即：请求或方法

## MapperMethod

* `org.apache.ibatis.binding.MapperProxyFactory#newInstance`



## mybatis 缓存

### 一级缓存

>基于**PerpetualCache** 的 **HashMap**本地缓存，其**存储作用域为** **Session**，当 **Session flush** **或** **close** 之后，该**Session中的所有 Cache 就将清空**

* `DefaultSqlSessionFactory#openSession`实例化`DefaultSqlSession`时，需要执行器`Executor`参数。产生`Executor`对象：

```java
if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
} else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
} else {
    executor = new SimpleExecutor(this, transaction);
}
```

* 实例化执行器`Executor`调用父类`BaseExecutor`构造方法，同时实例化一级缓存`PerpetualCache`：

```java
protected BaseExecutor(Configuration configuration, Transaction transaction) {
	this.transaction = transaction;
    this.deferredLoads = new ConcurrentLinkedQueue<DeferredLoad>();
    this.localCache = new PerpetualCache("LocalCache");
    this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
    this.closed = false;
    this.configuration = configuration;
    this.wrapper = this;
}
```

* 一级缓存生命周期与`sqlSession`相同，实例化`sqlSession`才实例化`PerpetualCache`

```java
this.localCache = new PerpetualCache("LocalCache");
```

* 查看`DefaultSqlSession#selectList(String statement, Object parameter, RowBounds rowBounds)`源码：

```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        MappedStatement ms = configuration.getMappedStatement(statement);
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

* 生成一级缓存的key，规则为：TODO

```java
@Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    //MyBatis 对于其 Key 的生成采取规则为：[mappedStementId + offset + limit + SQL + queryParams + environment]生成一个哈希码
    cacheKey.update(ms.getId());
    cacheKey.update(Integer.valueOf(rowBounds.getOffset()));
    cacheKey.update(Integer.valueOf(rowBounds.getLimit()));
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    //模仿DefaultParameterHandler的逻辑,不再重复，请参考DefaultParameterHandler
    for (int i = 0; i < parameterMappings.size(); i++) {
      ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```

* 接下来调用`BaseExecutor#query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) `方法，`BaseExecutor#localCache`属性为一级缓存

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        queryStack++;
        //根据cachekey从一级缓存localCache查询数据
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            clearLocalCache();
        }
    }
    return list;
}
```

* 保存数据到一级缓存：

TODO

* `org.apache.ibatis.session.defaults.DefaultSqlSession#close`清空`PerpetualCache`实例对象，即清空一级缓存
* `sqlSession`执行update、insert、delete时，都会调用`org.apache.ibatis.executor.BaseExecutor#update`方法，同时清空一级缓存

```java
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    //清空一级缓存
    clearLocalCache();
    return doUpdate(ms, parameter);
}
```

```java
@Override
public void clearLocalCache() {
    if (!closed) {
        //清空PerpetualCache#cache
        localCache.clear();
        localOutputParameterCache.clear();
    }
}
```

### 二级缓存

> **二级缓存**与一级缓存其机制相同，默认也是采用 PerpetualCache，HashMap存储，不同在于其**存储作用域为 Mapper(Namespace)**，并且**可自定义存储源**，如 Ehcache

> 二级缓存相关配置：
>
> * 全局配置文件中的setting中的cacheEnabled需要为true(默认为true)
> * mapper配置文件中需要加入<cache>节点
> * mapper配置文件中的select节点需要加上属性useCache需要为true(默认为true)

* 开启二级缓存：

```xml
<settings>	
	<setting name="cacheEnabled" value="true"/>
</settings>
```

* 在 mapper 映射文件中触发该Mapper(Namespace)开启二级缓存

```xml
<!-- 开启当前mapper的namespace下的二级缓存 -->
<!--
 	type:指定cache接口的实现类的类型；不写type属性，mybatis默认使用PerpetualCache；
	eviction：回收策略；
	flushInterval：刷洗缓存，单位毫秒，默认不自动刷新；
	size：缓存被应用的数目，默认1024次查询的结果；
    readOnly：只读属性，默认是false
	blocking：指定为true时将采用BlockingCache进行封装，默认是false
-->
<cache eviction="LRU" flushInterval="60000" size="512" readOnly="true" blocking="false"/>
```

* cache-ref节点：用来指定其它Mapper.xml中定义的Cache，有的时候可能我们多个不同的Mapper需要共享同一个缓存的，是希望在MapperA中缓存的内容在MapperB中可以直接命中的，这个时候我们就可以考虑使用cache-ref，这种场景只需要保证它们的缓存的Key是一致的即可命中，二级缓存的Key是通过`Executor`接口的createCacheKey()方法生成的，其实现`BaseExecutor#createCacheKey`

```xml
<!--PersonMapper.xml需要引用UserMapper.xml的缓存，就直接用UserMapper.xml的namespace-->
<cache-ref namespace="com.urbyte.mybatis.dao.UserMapper"/>
```

* `org.apache.ibatis.builder.xml.XMLMapperBuilder#cacheElement`解析mapper文件中的缓存配置

  * 解析cache节点的`eviction`属性，通过回收策略属性值在`org.apache.ibatis.type.TypeAliasRegistry#resolveAlias`中查询对应的class

  ```java
  typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class);
  typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
  typeAliasRegistry.registerAlias("LRU", LruCache.class);
  typeAliasRegistry.registerAlias("SOFT", SoftCache.class);
  typeAliasRegistry.registerAlias("WEAK", WeakCache.class);
  ```

  * 通过建造者模式构建以namespace为id的`Cache`实例对象
  * 将`Cache`实例对象保存到`org.apache.ibatis.session.Configuration#caches`中，以cache的id作为key，cache实例对象作为value存储；将cache实例对象赋值给`MapperBuilderAssistant#currentCache`

* 如果二级缓存开启(cacheEnabled=true)，则`DefaultSqlSession`实例对象中的执行器为`CachingExecutor`(二级缓存执行器)，`CachingExecutor#query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)`中会从二级缓存中取

```java 
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
    //从MappedStatement中获取缓存回收策略，同时缓存回收策略中包含缓存对象(Cache)；mapper文件如果没有配置cache节点，则ms.getCache()为空，直接委托给实际执行器(SimpleExecutor)处理,SimpleExecutor中的query方法会去查询一级缓存，如果一级缓存没有则去数据库查询
    Cache cache = ms.getCache();
    if (cache != null) {
        flushCacheIfRequired(ms);
        if (ms.isUseCache() && resultHandler == null) {
            ensureNoOutParams(ms, parameterObject, boundSql);
            @SuppressWarnings("unchecked")
            //从二级缓存中取
            List<E> list = (List<E>) tcm.getObject(cache, key);
            if (list == null) {
                list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                //保存到二级缓存中
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            return list;
        }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```

* 清空二级缓存数据：执行update、insert、delete时都会调用`CachingExecutor#update`

```java
@Override
public int update(MappedStatement ms, Object parameterObject) throws SQLException {
    //刷新缓存，清空二级缓存
    flushCacheIfRequired(ms);
    return delegate.update(ms, parameterObject);
}
private void flushCacheIfRequired(MappedStatement ms) {
    Cache cache = ms.getCache();
    if (cache != null && ms.isFlushCacheRequired()) {      
        tcm.clear(cache);
    }
}
```

* 保持二级缓存数据：`DefaultSqlSession#commit`->`CachingExecutor#commit`->`TransactionalCacheManager#commit`->`TransactionalCache#commit`->`TransactionalCache#flushPendingEntries`

```java
private void flushPendingEntries() {
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
        /**
         * delegate为PerpetualCache、FifoCache、LruCache、SoftCache、WeakCache
         * FifoCache、LruCache、SoftCache、WeakCache通过装饰器模式最终调用   PerpetualCache#putObject
        **/
        delegate.putObject(entry.getKey(), entry.getValue());
    }
    for (Object entry : entriesMissedInCache) {
        if (!entriesToAddOnCommit.containsKey(entry)) {
            delegate.putObject(entry, null);
        }
    }
}
```

* 存在的问题：TODO

### Cache接口

> `org.apache.ibatis.cache.Cache`是MyBatis的缓存接口，自定义的缓存需要实现这个接口
>
> Cache接口的实现类使用了装饰器设计模式

* 接口实现类：
* [图片处理](https://blog.csdn.net/u010158659/article/details/61197893)TODO

![image-20181005161532991](/var/folders/fr/4dpl599d5gj7gpt4csyc3x5m0000gn/T/abnerworks.Typora/image-20181005161532991.png)

* LRU – 最近最少使用:移除最长时间不被使用的对象

  FIFO – 先进先出:按对象进入缓存的顺序来移除它们

  SOFT – 软引用:移除基于垃圾回收器状态和软引用规则的对象

  WEAK – 弱引用:更积极地移除基于垃圾收集器状态和弱引用规则的对象

  LoggingCache – 输出缓存命中的日志信息

  ScheduledCache – 调度缓存，负责定时清空缓存

  SerializedCache – 缓存序列化和反序列化存储

  SynchronizedCache – 同步的缓存装饰器，用于防止多线程并发访问

* 二级缓存引用顺序：BlockingCache-->SynchronizedCache-->LoggingCache-->SerializedCache-->ScheduledCache-->LruCache/FifoCache/SoftCache/WeakCache-->PerpetualCache

  ```java
  public Cache build() {
      setDefaultImplementations();
      //new一个base的cache(PerpetualCache)
      Cache cache = newBaseCacheInstance(implementation, id);
      //设置属性
      setCacheProperties(cache);
      // 装饰器缓存不能应用到自定义缓存
      if (PerpetualCache.class.equals(cache.getClass())) {
          for (Class<? extends Cache> decorator : decorators) {
              cache = newCacheDecoratorInstance(decorator, cache);
              setCacheProperties(cache);
          }
          //附加上标准的装饰者
          cache = setStandardDecorators(cache);
      } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
          //自定义缓存，且不是日志，要加日志缓存
          cache = new LoggingCache(cache);
      }
      return cache;
  }
  //附加上标准的装饰者
  private Cache setStandardDecorators(Cache cache) {
      try {
          MetaObject metaCache = SystemMetaObject.forObject(cache);
          if (size != null && metaCache.hasSetter("size")) {
              metaCache.setValue("size", size);
          }
          if (clearInterval != null) {
              cache = new ScheduledCache(cache);
              ((ScheduledCache) cache).setClearInterval(clearInterval);
          }
          if (readWrite) {
              cache = new SerializedCache(cache);
          }
          //日志缓存
          cache = new LoggingCache(cache);
          cache = new SynchronizedCache(cache);
          if (blocking) {
              cache = new BlockingCache(cache);
          }
          return cache;
      } catch (Exception e) {
          throw new CacheException("Error building standard cache decorators.  Cause: " + e, e);
      }
  }
  ```

* 自定义缓存接口，比如配置mybatis使用redis作为自定义缓存

  > type为自定义缓存时，装饰器上的eviction对应的回收策略不能应用到自定义缓存中

  * 实现`org.apache.ibatis.cache.Cache`接口
  * 开启二级缓存配置

  ```xml
  <setting name="cacheEnabled" value="true"/>
  ```

  * Mapper.xml配置文件中加入cache节点

  ```xml
  <cache type="org.mybatis.caches.redis.RedisCache"/>
  ```

  * 用Redis作为自定义缓存源码：

  ```xml
  <dependency>
      <groupId>org.mybatis.caches</groupId>
      <artifactId>mybatis-redis</artifactId>
      <version>1.0.0-beta2</version>
  </dependency>
  ```

## 执行器Executor

> `org.apache.ibatis.executor.Executor`接口，sql执行器，SqlSession执行sql最终是通过该接口实现的，常用的实现类有SimpleExecutor和CachingExecutor,这些实现类都使用了装饰者设计模式

* ![image-20181005163449385](/var/folders/fr/4dpl599d5gj7gpt4csyc3x5m0000gn/T/abnerworks.Typora/image-20181005163449385.png)
* 

### 键值生成器KeyGenerator



### 结果集处理器ResultSetHandler



### 语句处理器StatementHandler





## 日志接口Log



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

### XML方式

`org.mybatis.spring.mapper.MapperScannerConfigurer`





### Annotation方式

`org.mybatis.spring.annotation.MapperScannerRegistrar`