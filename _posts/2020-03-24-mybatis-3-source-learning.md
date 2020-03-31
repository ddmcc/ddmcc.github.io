---
layout: post
title:  "mybatis3源码学习"
date:   2020-3-24 23:34:31
categories: mybatis
tags:  mybatis
author: ddmcc
---

* content
{:toc}


## 核心的类

MyBatis有几个核心的类如：`SqlSessionFactoryBuilder`，`SqlSessionFactory`，`SqlSession`，`Configuration` 等


使用 MyBatis 的主要 Java 接口就是 SqlSession。可以通过这个接口来执行命令，获取Mapper接口和管理事务。SqlSessions 是由 SqlSessionFactory 实例创建的。SqlSessionFactory 对象包含创建 SqlSession 实例的各种方法。而 SqlSessionFactory 本身是由 SqlSessionFactoryBuilder 创建的，它可以从 XML、注解或 Java 配置代码来创建 SqlSessionFactory。





### Configuration


在构建 `SqlSessionFactory` 之前先要配置 `Configuration` 实例，Mybatis所有的配置都在这个类里面，在运行时可以通过 **SqlSessionFactory#getConfiguration()** 来获得并检查配置。


> 以下删除了很多配置

```java

public class Configuration {

  // 数据源，事务工厂
  protected Environment environment;

  protected boolean safeRowBoundsEnabled;
  protected boolean safeResultHandlerEnabled = true;
  //  是否开启驼峰命名自动映射，即从经典数据库列名 A_COLUMN 映射到经典 Java 属性名 aColumn。
  protected boolean mapUnderscoreToCamelCase;

  
  // 允许使用方法签名中的名称作为语句参数名称,必须采用 Java 8 编译，并且加上 -parameters 选项。（新增于 3.4.1）
  // 如果开启 在多个参数的接口 可以不用@Param注解或是arg0,param1等，直接用接口的参数名就能替换 
  // 如接口 listBooks(String title, String author) 在sql中直接使用 #{title} #{author}
  protected boolean useActualParamName = true;
  protected boolean returnInstanceForEmptyRow;

  protected String logPrefix;
  protected Class<? extends Log> logImpl;
  protected Class<? extends VFS> vfsImpl;
  
  // 本地缓存范围，默认SqlSession 级别,会缓存会话中执行的所有查询
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;

  // properties 节点
  protected Properties variables = new Properties();
  protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
  protected ObjectFactory objectFactory = new DefaultObjectFactory();

  protected ProxyFactory proxyFactory = new JavassistProxyFactory(); // #224 Using internal Javassist instead of OGNL

  protected String databaseId;
  /**
   * Configuration factory class.
   * Used to create Configuration for loading deserialized unread properties.
   *
   * @see <a href='https://code.google.com/p/mybatis/issues/detail?id=300'>Issue 300 (google code)</a>
   */
  protected Class<?> configurationFactory;

  // mapper注册器，存mapper接口的地方
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  // 拦截器链
  protected final InterceptorChain interceptorChain = new InterceptorChain();
  // TypeHandler注册器，存TypeHandler接口的地方
  protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry(this);
  // 存放类别名与类映射
  protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
  protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();

  // mappedStatements
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
      .conflictMessageProducer((savedValue, targetValue) ->
          ". please check " + savedValue.getResource() + " and " + targetValue.getResource());
  
  // 以下字如其名
  protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
  protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
  protected final Map<String, ParameterMap> parameterMaps = new StrictMap<>("Parameter Maps collection");
  protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<>("Key Generators collection");

  // 已加载的文件
  protected final Set<String> loadedResources = new HashSet<>();
  protected final Map<String, XNode> sqlFragments = new StrictMap<>("XML fragments parsed from previous mappers");


}
```

---
### SqlSessionFactory

#### 1，XML中构建 SqlSessionFactory

下面是Mybatis配置文件（这里只是Mybatis的配置，没有与Spring结合）

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
    PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
    <environments default="development">
        <!-- environment 节点包含事务管理器和数据源的配置 -->
        <environment id="development">
            <transactionManager type="JDBC">
                <property name="" value="" />
            </transactionManager>
            <dataSource type="UNPOOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver" />
                <property name="url" value="" />
                <property name="username" value="" />
                <property name="password" value=""/>
            </dataSource>
        </environment>
    </environments>

    <!-- mappers -->
    <mappers>
        <mapper class="com.ddmcc.UserMapper" />
    </mappers>

</configuration>
```
    
```java
   public static void main(String[] args) throws Exception{
        // 构建SqlSessionFactory对象
        SqlSessionFactory sqlSessionFactory;
        try (Reader reader = Resources.getResourceAsReader("com/ddmcc/mybatis-config.xml")) {
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        }
    }
```
    
 　　读取配置文件创建流，利用SqlSessionFactoryBuilder对象将流构建成`SqlSessionFactory`对象。SqlSessionFactory其实是一个接口，调用build方法返回的
是**DefaultSqlSessionFactory**对象，它是SqlSessionFactory的默认实现。在DefaultSqlSessionFactory中，有一个**Configuration**对象，基本上我们的所有配置属性，mapper接口等都会
保存在这个对象中


---
![KL9ADG7_@TBA~H_JM5_8~ZG.png](https://i.loli.net/2020/03/30/KTJUVS4fQZMygRH.png)
   
   
---
##### build()
    
```java

    public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
        try {
            // 构建解析对象
            XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
            
            // 解析XML，parser.parse()返回的是Configuration对象
            // 再调用build(Configuration对象) 返回一个DefaultSqlSessionFactory
            return build(parser.parse());
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
            ErrorContext.instance().reset();
            try {
                reader.close();
            } catch (IOException e) {
                // Intentionally ignore. Prefer previous error.
            }
        }
    }
```

---
获得配置文件configuration节点下的所有的子节点，逐一解析，赋值到Configuration对象中对应的属性。

可以看到下面解析的方法都是XMLConfigBuilder类内部的方法，所有configuration的属性都是在各个方法里设置的，因为在XMLConfigBuilder中，有一个configuration属性。（parser.parse()方法会判断是否已经解析过了，
如果执行不止一次parse()方法那么两次解析出来的配置就会窜在一起了？？？）

---
##### parse()

```java
    public Configuration parse() {
        // 已解析
        if (parsed) {
            throw new BuilderException("Each XMLConfigBuilder can only be used once.");
        }
        parsed = true;
        // 获取configuration节点下所有子节点
        parseConfiguration(parser.evalNode("/configuration"));
        return configuration;
    }

    private void parseConfiguration(XNode root) {
        try {
            // properties
            // issue #117 read properties first
            propertiesElement(root.evalNode("properties"));
            Properties settings = settingsAsProperties(root.evalNode("settings"));
            loadCustomVfs(settings);
            loadCustomLogImpl(settings);
            typeAliasesElement(root.evalNode("typeAliases"));
            pluginElement(root.evalNode("plugins"));
            objectFactoryElement(root.evalNode("objectFactory"));
            objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
            reflectorFactoryElement(root.evalNode("reflectorFactory"));
            settingsElement(settings);
            // read it after objectFactory and objectWrapperFactory issue #631
            // 数据源，事务管理器
            environmentsElement(root.evalNode("environments"));
            databaseIdProviderElement(root.evalNode("databaseIdProvider"));
            typeHandlerElement(root.evalNode("typeHandlers"));
            // 解析mapper
            mapperElement(root.evalNode("mappers"));
        } catch (Exception e) {
            throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
    }
```

---
#### 2，Java代码构建SqlSessionFactory


下面是直接拷贝的官方文档中Java代码构建SqlSessionFactory的[示例](https://mybatis.org/mybatis-3/zh/java-api.html#sqlSessions)


```java
DataSource dataSource = BaseDataTest.createBlogDataSource();
TransactionFactory transactionFactory = new JdbcTransactionFactory();

Environment environment = new Environment("development", transactionFactory, dataSource);

Configuration configuration = new Configuration(environment);
configuration.setLazyLoadingEnabled(true);
configuration.setEnhancementEnabled(true);
configuration.getTypeAliasRegistry().registerAlias(Blog.class);
configuration.getTypeAliasRegistry().registerAlias(Post.class);
configuration.getTypeAliasRegistry().registerAlias(Author.class);
configuration.addMapper(BoundBlogMapper.class);
configuration.addMapper(BoundAuthorMapper.class);

SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
SqlSessionFactory factory = builder.build(configuration);
```

---
### 对象的作用域

- SqlSessionFactoryBuilder (**局部方法**)

一旦创建了 SqlSessionFactory，就不再需要它了。 因此 SqlSessionFactoryBuilder 实例的最佳作用域是方法作用域（也就是局部方法变量）。 可以重用 SqlSessionFactoryBuilder 来创建多个 SqlSessionFactory 实例，但最好还是不要一直保留着它，以便释放所有的 XML 资源文件

- SqlSessionFactory (**全局，整个应用**)

SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在,因此 SqlSessionFactory 的最佳作用域是应用作用域。 有很多方法可以做到，最简单的就是使用单例模式或者静态单例模式。

- SqlSession (**一次请求，局部方法或者说一个线程**)

每个线程都应该有它自己的 SqlSession 实例。SqlSession 的实例不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是请求或方法作用域。 绝对不能将 SqlSession 实例的引用放在一个类的静态域，甚至一个类的实例变量也不行。 也绝不能将 SqlSession 实例的引用放在任何类型的托管作用域中，比如 Servlet 框架中的 HttpSession。
每次收到 HTTP 请求，就可以打开一个 SqlSession，返回一个响应后，就关闭它。 这个关闭操作很重要，为了确保每次都能执行关闭操作，你应该把这个关闭操作放到 finally 块中。 下面的示例就是一个确保 SqlSession 关闭的标准模式：

>try (SqlSession session = sqlSessionFactory.openSession()) {
     // 你的应用逻辑代码
   }


---
### 解析mappers

解析 `mapper` 的入口方法是 mapperElement ，取得 mappers 节点下的子节点循环添加。子节点有 `package` 和 `mapper` 


```text
    private void mapperElement(XNode parent) throws Exception {
        if (parent != null) {
            for (XNode child : parent.getChildren()) {
                // package节点
                
                if ("package".equals(child.getName())) {
                    String mapperPackage = child.getStringAttribute("name");
                    // 加载包名下的接口
                    configuration.addMappers(mapperPackage);
                } else {
                    
                    // mapper 节点有 resource，url，class 三个属性，只能选择其一，否则抛异常
                    String resource = child.getStringAttribute("resource");
                    String url = child.getStringAttribute("url");
                    String mapperClass = child.getStringAttribute("class");
                    
                    // resource xml文件
                    if (resource != null && url == null && mapperClass == null) {
                        ErrorContext.instance().resource(resource);
                        InputStream inputStream = Resources.getResourceAsStream(resource);
                        XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
                        mapperParser.parse();
                        
                    // url 网络xml文件
                    } else if (resource == null && url != null && mapperClass == null) {
                        ErrorContext.instance().resource(url);
                        InputStream inputStream = Resources.getUrlAsStream(url);
                        XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
                        mapperParser.parse();
                        
                    // class
                    } else if (resource == null && url == null && mapperClass != null) {
                        Class<?> mapperInterface = Resources.classForName(mapperClass);
                        configuration.addMapper(mapperInterface);
                    } else {
                        throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
                    }
                }
            }
        }
    }
```

##### **加载 package**

添加 mapper 接口的操作都是在 configuration 对象的 `mapperRegistry` 对象属性里进行的，解析出来的接口也都是以 key,value 形式存在一个HashMap中


- 会先判断扫出该包下所有超类为 `Object.class` 的class，存在Set列表

- Set列表过滤出所有接口

- 以 key 为该接口类型， value 为该接口的 `MapperProxyFactory` 

- 


---
## 其它

#### **SQL注入**

- 仅在字符串中发生
- 仅在不使用准备好的语句时发生
- 仅在用户输入时发生


#### **${}与#{}**

- ${}

可以看成是动态sql，在Mybatis中，会在获得Statement执行对象之前对sql进行映射替换

如 SELECT * FROM USER WHERE USER_NAME = ${name}      ${name}= "111 OR 1 = 1"

那么在Mybatis解析完sql后，就会变成

SELECT * FROM USER WHERE USER_NAME = 111 OR 1 = 1


- #{}

是静态sql，Mybatis在解析sql的时候会将 #{name} 替换成 占位符 ?

然后用sql生成 PreparedStatement 对象 pstm

再对对象设查询参数 pstm.setStringParameter("111 OR 1 = 1");

这是真正执行的sql 就会变成  **SELECT * FROM USER WHERE USER_NAME = '111 OR 1 = 1'**



#### **Mybatis的参数映射问题**

args0,args1...
param1,param2...

@Param注册

JDK1.8后不使用Param注解，直接用参数名映射

