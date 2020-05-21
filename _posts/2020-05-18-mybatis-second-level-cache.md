---
layout: post
title:  "mybatis3 二级缓存"
date:   2020-05-18 23:27:43
categories: mybatis3
tags:  cache 二级缓存 mybatis
author: ddmcc
---

* content
{:toc}


## 二级缓存的机制与工作模式

在上一篇 [mybatis3 一级缓存](https://ddmcc.cn/2020/05/11/mybatis-first-level-cache/) 中提到一级缓存的最大共享范围是 `SqlSession` ，如果需要多个 `SqlSession`  共享，就需要使用二级缓存。**二级缓存默认是不开启的**，当开启后（ **cacheEnabled=true** ）会使用 `CachingExecutor` 装饰 `Executor` 。`CachingExecutor` 是 `Executor` 的装饰者，以增强**Executor**的功能，使其具有缓存查询的功能。









类图如下：

![7Q4W_LYH92GE2~ET_M_NQ68.png](https://i.loli.net/2020/05/21/MSGcHRwZ86vebyn.png) 

以下是`Configuration`类初始化 `Executor` 对象代码片段：

```java
...........
if (cacheEnabled) {
    executor = new CachingExecutor(executor);
}
executor = (Executor) interceptorChain.pluginAll(executor);
return executor;
```

**注：本文参照3.5.5-SNAPSHOT版本，cacheEnabled 属性是默认开启的，在 XMLConfigBuilder 中 249行**

```
configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
```



开启二级缓存后，`SqlSession` 就使用 `CachingExecutor` 对象来完成操作请求，对于查询的请求`CachingExecutor` 会先去二级缓存查找是否有缓存，如果有，那么直接返回给用户缓存数据，如果没有，则由 ``Executor`` 对象去完成查询操作，再把``Executor`` 查询返回的数据存入到二级缓存中，再返回给用户。



在查询一级缓存之前会先在 `CachingExecutor` 中查询二级缓存中是否有数据，具体查询工作流程如图：

![J07__8BG_QYPE_163E73464.png](https://i.loli.net/2020/05/21/GZHuXFeU68tOqcE.png)

当开启二级缓存后，同一个 `namespace` 下的所有sql影响着同一个 `Cache` ，即同一个 `namespace`下的所有 `MappedStatement` 影响着同一个 `Cache` ，这个 Cache 被多个 `SqlSession` 共享，相当于一个全局变量。

查询的工作流程为 二级缓存 -> 一级缓存 -> 数据库



## 二级缓存的划分 

上面说到 "同一个 namespace 下的所有sql影响着同一个Cache "，意思也就是每个 `mapper` 都有一个自己的 `Cache` 对象，也可以多个mapper共享一个 `Cache` 对象

1. 为mapper配置一个Cache对象： 在mapper.xml中配置 `<cache>` 节点
2. 为多个mapper配置一个Cache对象： 配置 `<cache-ref` 节点







## 使用二级缓存要具备的条件





## 二级缓存的生命周期





## 需要注意的点

