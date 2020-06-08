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

在上一篇 [mybatis3 一级缓存](https://ddmcc.cn/2020/05/11/mybatis-first-level-cache/) 中提到一级缓存的最大共享范围是 `SqlSession` ，如果需要多个 `SqlSession`  共享，就需要使用二级缓存。**二级缓存是默认开启的**，当开启后（ **cacheEnabled=true** ）会使用 `CachingExecutor` 装饰 `Executor` 。CachingExecutor 是 Executor 的装饰者，以增强**Executor**的功能，使其具有缓存查询的功能。









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





---

开启二级缓存后，`SqlSession` 就使用 `CachingExecutor` 对象来完成操作请求，对于查询的请求`CachingExecutor` 会先去二级缓存查找是否有缓存，如果有，那么直接返回给用户缓存数据，如果没有，则由 ``Executor`` 对象去完成查询操作，再把``Executor`` 查询返回的数据存入到二级缓存中，再返回给用户。



在查询一级缓存之前会先在 `CachingExecutor` 中查询二级缓存中是否有数据，具体查询工作流程如图：

![J07__8BG_QYPE_163E73464.png](https://i.loli.net/2020/05/21/GZHuXFeU68tOqcE.png)

查询的工作流程为 **二级缓存 -> 一级缓存 -> 数据库**



当开启二级缓存后，同一个 `namespace` 下的所有sql影响着同一个 `Cache` ，即同一个 `namespace`下的所有 `MappedStatement` 影响着同一个 `Cache` ，这个 Cache 被多个 `SqlSession` 共享，相当于一个全局变量



## 二级缓存的特点

**MyBatis**自身提供了丰富的，并且功能强大的二级缓存的实现，它拥有一系列的**Cache**接口装饰者，可以满足各种对缓存操作和更新的策略。在 `CachingExecuter` 内部有一个用来管理二级缓存的 `TransactionalCacheManager` 对象，`TransactionalCacheManager` 内部只有一个成员变量，key 是 `mapper` 中定义的cache对象，value 是**暂时存缓存数据的对象**。



`TransactionalCache` 装饰的对象就是定义的 `cache` 对象 ：

```java
// key 是 `mapper` 中定义的cache对象，value 是暂时存缓存数据的对象
private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();

// ...... 其它方法

// 如果缓存对象不存在，就创建一个装饰着cache的TransactionalCache对象
private TransactionalCache getTransactionalCache(Cache cache) {
    return transactionalCaches.computeIfAbsent(cache, TransactionalCache::new);
}
```



`TransactionalCache` 也是cache接口的装饰者之一，主要作用是保存SqlSession在事务中需要向某个二级缓存提交的缓存数据（因为事务过程中的数据可能会回滚，所以不能直接把数据就存入二级缓存，而是暂存在TransactionalCache中，在事务提交后再将过程中存放在其中的数据提交到二级缓存，如果事务回滚，则将数据清除掉）。



// TODO 

#### **TransactionalCache** 对象





## 二级缓存的划分 

每个 `mapper` 都有一个自己的 `Cache` 对象，也可以多个mapper共享一个 Cache 对象

1. 为mapper配置一个Cache对象： 在mapper.xml中配置 `<cache>` 节点或在接口添加 `@CacheNamespace` 注解
2. 为多个mapper配置一个Cache对象： 配置 `<cache-ref>` 节点或在接口添加 `@CacheNamespaceRef` 注解



通过 `<cache-ref>` 标签，定义 `namespace` 来指定要引用的缓存的命名空间。这句话有点绕，也就是

```xml
<mapper namespace="com.ddmcc.UserMapper">
    <cache-ref namespace="com.ddmcc.AdminUserMapper" />
</mapper>
```



这时就要求 `AdminUserMapper` 必须定义了 `<cache>` 节点。如果用注解的方式如下：



```java
@CacheNamespace
public interface UserMapper {
    // ...   
}


// 定义要引用的命名空间的类型
@CacheNamespaceRef(AdminUserMapper.class)
public interface UserMapper {
   // ...
}

或者 

// 定义要引用的命名空间的name
@CacheNamespaceRef(name = "AdminUserMapper")
public interface UserMapper {
   // ...
}
```



**通过以上配置，就可以让多个 `Mapper` 公用一个 `Cache` **



## 使用二级缓存要具备的条件

Mybatis二级缓存粒度很细，可以精确到每一条查询语句是否使用缓存



在Mybatis配置中开启了缓存，并且在 `mapper` 中配置了 <cache> 节点，这并不意味着每一条查询语句都会使用到缓存，还需要指定 `select` 语句是否开启缓存 ， `useCache="true"` 声明这条语句开启缓存后，才会使用缓存。



```xml
 <select id="listUser" resultType="com.ddmcc.User" useCache="true">
```



要想使用二级缓存，那么需要满足以下三个条件：



1. **开启二级缓存的总开关：全局配置变量参数  cacheEnabled=true （默认开启）**
2. **为mapper配置了<cache>、<cacheRef> 节点或者mapper接口中配置了 @CacheNamespace、@CacheNamespaceRef注解**
3. **该select语句节点开启了缓存useCache="true" （默认开启）**



**对于我们平常使用来说，如果为一个mapper配置的<cache>节点，那么此mapper中的查询语句将会使用二级缓存**



## 二级缓存的生命周期

#### 回顾一级缓存

在 **一级缓存** 中，数据从数据库查询返回后，就会存入缓存中，SqlSession 执行 `commit`（提交），`close`（关闭），`rollback`（回滚），任何一个 `update` 操作或是 `clearLocalCache` 会清空一级缓存。



#### 二级缓存生命周期

二级缓存则不同，上面说了在缓存创建的时候并不是直接put进缓存对象中，而是会先暂存在 `TransactionalCache` 对象中。当 `sqlSession` 关闭或提交时，再把数据刷入缓存中，如果`rollback` 那么不会把数据刷入缓存中，**但也不会清空已有的二级缓存**。sqlSession 执行任何一个 update 操作时，在事务提交后，都会去清空二级缓存。sqlSession的 `clearCache` 只清除自己会话中的一级缓存，并不会清除二级缓存。

但，并不是 `select` 语句就会存入二级缓存，`update` 就会清除二级缓存。主要有标签的两个参数决定：`useCache`和`flushCache`。select标签默认是 userCache=true 、flushCache=false，所以会将数据存入二级缓存。而 insert|update|delete 标签默认是 userCache=false、flushCache=true，**如果将flushCache改成false，那么也不会去清除缓存**

总的来说：

1. sqlSession调用commit或close才会把二级缓存存入
2. sqlSession执行任一update操作会清除缓存
3. sqlSession回滚不会清除缓存，但本次事务内的查询数据也不会写入缓存中
4. 由userCache、flushCache两个参数决定语句是否使用缓存或清除缓存



## 需要注意的点

1. 同一事务执行多次相同查询或是多个session执行同一mapper下的同一查询语句并不会命中二级缓存，因为二级缓存在 sqlSession close或commit的时候才会写入

![B_C93__7GNKM3UGF_93_@J3.png](https://i.loli.net/2020/06/08/2GDkdl7Mrisv9IH.png)



代码示例：





