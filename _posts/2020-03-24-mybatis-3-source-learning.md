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

### 怎么启动的？


### SqlSessionFactory 对象是如何被构建出来的
    
 　　读取配置文件创建流，利用SqlSessionFactoryBuilder对象将流构建成`SqlSessionFactory`对象。SqlSessionFactory其实是一个接口，调用build方法返回的
是**DefaultSqlSessionFactory**对象，它是SqlSessionFactory的默认实现。在DefaultSqlSessionFactory中，有一个**Configuration**对象，基本上我们的所有配置属性，mapper接口等都会
保存在这个对象中
    
```java
    SqlSessionFactory sqlSessionFactory;
    try (Reader reader = Resources.getResourceAsReader("com/ddmcc/mybatis-config.xml")) {
        sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
    }


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

