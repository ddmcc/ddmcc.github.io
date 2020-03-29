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


MyBatis 的核心是 SqlSessionFactory 实例。SqlSessionFactory 的实例可以通过 SqlSessionFactoryBuilder 获得。而 SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个已有的 Configuration 实例来构建出 SqlSessionFactory 实例

构建 SqlSessionFactory 实例的方法有两种，`一是从 XML 中构建 SqlSessionFactory，二是可以通过Java 代码的方式来构建`。




### 构建SqlSessionFactory


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
                <property name="url" value="jdbc:mysql://47.107.135.196:3306/user?characterEncoding=utf-8" />
                <property name="username" value="job" />
                <property name="password" value="123456"/>
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

// TODO SqlSessionFactory DefaultSqlSessionFactory SqlSessionManager 类图
    
##### SqlSessionFactoryBuilder#build
    
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


##### XMLConfigBuilder#parse

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

### 如何加载mapper配置文件和接口，并解析成MappedStatement




### 其它

##### SQL注入

- 仅在字符串中发生
- 仅在不使用准备好的语句时发生
- 仅在用户输入时发生


##### ${}与#{}

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



##### Mybatis的参数映射问题

args0,args1...
param1,param2...

@Param注册

JDK1.8后不使用Param注解，直接用参数名映射

