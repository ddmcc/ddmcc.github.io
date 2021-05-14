---
layout: post
title:  "mybatis3 一级缓存"
date:   2020-05-11 21:32:57
categories: mybatis3
tags:  cache 一级缓存 mybatis
author: ddmcc
---

* content
{:toc}


## mybatis 一级缓存

mybatis有一级，二级缓存机制，**一级缓存是默认开启的本地缓存，且不可关闭** 本文主要介绍一级缓存。通过本文你将了解：




- 什么是一级缓存？使用一级缓存的好处
- 一级缓存是如何设计的？
- Cache接口的设计以及CacheKey的定义
- 一级缓存的生命周期
- 使用一级缓存值得注意的点


## 什么是一级缓存？使用一级缓存的好处

说到 `一级缓存` 那就不得不说 `SqlSession` 对象。顾名思义，`session` 代表与数据库的会话。每当我们使用MyBatis执行sql时，`MyBatis` 会创建出一个 `SqlSession` 对象表示一次数据库会话。

在一次会话中，我们有可能会很多的语句，或反复地执行完全相同的语句。对于反复执行相同的语句且返回的结果是相同的话就没必要每次都去查询数据库了，这么做不但效率低且浪费资源。

为了解决这一问题，减少资源的浪费，MyBatis会在表示会话的 `SqlSession` 对象中建立一个简单的缓存，将每次查询到的结果结果缓存起来，当下次查询的时候，如果判断先前有个完全一样的查询，会直接从缓存中直接将结果取出，返回给用户，不需要再进行一次数据库查询了。

这个缓存是一个本地缓存(local cache)，存储在 `SqlSession` 对象中。对于每一次查询，都会尝试根据查询的条件去本地缓存中查找是否在缓存中，如果在缓存中，就直接从缓存中取出，然后返回给用户；否则，从数据库读取数据，将查询结果存入缓存并返回给用户

**对于会话（Session）级别的数据缓存，我们称之为一级缓存**


## 一级缓存是如何设计的？


简单示意图：



![markdown](https://ddmcc-1255635056.file.myqcloud.com/a69a0708-b5d8-4a31-8dc7-8676ab37f16e.png)

---



当创建新的 `SqlSession`时，Mybatis也会为这个 `SqlSession` 创建一个 `Executor` 执行器，它是实际执行数据库操作的对象。一级缓存就维护在 `Executor` 对象中。而对缓存和缓存相关的操作，Mybatis将它封装在 `Cache`接口中。

 所以`SqlSession`,`Executor`,`Cache`三者的关系类图如下：



![markdown](https://ddmcc-1255635056.file.myqcloud.com/e8f2bf39-ffad-4759-aec8-7a7d4cd4ba46.png)

---

如上述的类图所示，`Executor` 接口的实现类 `BaseExecutor` 中拥有一个 `Cache` 接口的实现类 `PerpetualCache` ，所以它将使用 `PerpetualCache` 对象维护缓存。


综上，`SqlSession`，`Executor`,`Cache` 三个对象之间的关系图如下：



![markdown](https://ddmcc-1255635056.file.myqcloud.com/4e91b475-98f4-4f84-8768-1581c23bfcc8.png)

---



#### **工作流程**


一级缓存执行的时序图，如下图所示。


![markdown](https://ddmcc-1255635056.file.myqcloud.com/0b669d40-30ff-4108-bf06-faf0bf0ced4c.png)

---


我们已经知道 **一级缓存就是 `PerpetualCache` 对象维护的** ，那么 `PerpetualCache` 如何实现的也就是 `一级缓存` 的原理了




#### **PerpetualCache 对象**


一级缓存内部实现其实就是用 `HashMap` 来实现的，以 `Key,Value` 形式维护缓存，下面是 `PerpetualCache` 提供的一些接口，对一级缓存的操作实则是对HashMap的操作。



![markdown](https://ddmcc-1255635056.file.myqcloud.com/3638bd86-bbe1-495e-9b58-b2010cb956bd.png)


## Cache接口的设计以及CacheKey的定义

`Cache` 接口有很多的实现，一级缓存只会涉及到这一个 `PerpetualCache` 子类。通过阅读 `PerpetualCache` 源码我们知道，缓存内部使用 `Map`
来维护的，`key` 是本次查询的特征值， `value` 是本次查询的查询结果。 什么是 `本次查询特征值`？ 也就是能代表本次查询的，它不能单单是查询的sql，也不能是查询的参数，它应该是本次查询所有条件的集合！ 
**所以如何确定本次查询的特征值就是一级缓存的重点**，也就是如何确定两次查询是否是一样的？

Mybatis认为，对于是两次查询是否是相同的，需要满足以下的条件：

- **相同的statementId**

- **结果集中的要求的结果范围 （结果的范围通过rowBounds.offset和rowBounds.limit表示）相同**

- **经过参数解析过后的字符串（boundSql.getSql()）要相同**

- **给java.sql.Statement设置的参数值**



分别解释上述四个条件：

1. 对于Mybatis而言，你要执行哪个接口，哪条sql，必须传入对应的statementId

2. MyBatis自身提供的分页功能是通过RowBounds来实现的，它通过rowBounds.offset和rowBounds.limit来过滤查询出来的结果集，这种分页功能是基于查询结果的再过滤，而不是进行数据库的物理分页。
如果查询有传入分页条件，那么两次查询分页条件也必须一致，因为也会造成结果不一致。

3. Mybatis底层还是调用的JDBC API去访问数据库。对于JDBC而言，两次查询的sql和参数都一致，那么结果也可以认为是一致的。

4. 第四点就是保证参数要一致



>boundSql.getSql() 返回的是经过映射参数解析过的sql，比如#{}会解析成？，${}则已经把参数替换上。然后MyBatis拿着这个sql创建JDBC的PreparedStatement对象，对于这个PreparedStatement对象，
还需要对它设置参数，调用setXXX()来完成设值，第4条，就是要求对设置JDBC的PreparedStatement的参数值也要完全一致。
---


综上所述，就是要满足 **调用JDBC API的时候，传入的SQL语句要完全相同，传递的参数值也要完全相同，如果有用RowBounds分页，那么分页参数也要一致**，`CacheKey` 也就由 **statementId  + RowBounds  + 传递给JDBC的SQL  + 传递给JDBC的参数值** 决定。

`CacheKey` 作为本次查询的 `特征值` 它的作用就是在查询的时候去缓存 `Map` 中查找缓存，如果查找到缓存，那么直接返回，如果缓存中没查到，那么就去数据库查询，
查询后将这个 CacheKey 作为 key，查询结果作为 value 存储到 缓存Map中。


#### **CacheKey**

`CacheKey` 的构建方法在 `BaseExecutor` 中，源码如下：


```java
@Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    ............
    CacheKey cacheKey = new CacheKey();
    // statementId 
    cacheKey.update(ms.getId());
    // rowBounds.offset 分页参数
    cacheKey.update(rowBounds.getOffset());
    // rowBounds.limit  分页参数
    cacheKey.update(rowBounds.getLimit());
    // boundSql.getSql() SQL语句
    cacheKey.update(boundSql.getSql());
    
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        .........
        // 每一个参数值
        cacheKey.update(value);
      }
    }
    // 运行环境
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```



#### **CacheKey hashcode算法**

一级缓存内部实现本质还是用 `Map<K,V>` 来实现的，而构建 `CacheKey` 目的也就是作为 `Map` 的key，所以构建 `CacheKey` 的过程也可以看成是
构建 `hashcode` 的过程（map 的 key值取得是hashcode）


下面是构建 hashcode 的过程：

```java
public void update(Object object) {
    // 得到对象的hashcode
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);
    // 更新计数++
    count++;
    // 所有的baseHashCode相加的值
    checksum += baseHashCode;
    // baseHashCode乘以 count倍
    baseHashCode *= count;
    // hashcode = 拓展因子（默认37）* 当前hashcode * baseHashCode
    hashcode = multiplier * hashcode + baseHashCode;

    updateList.add(object);
  }
```


`CacheKey` 重写的 `equals` 方法

```java
@Override
  public boolean equals(Object object) {
    if (this == object) {
      return true;
    }
    if (!(object instanceof CacheKey)) {
      return false;
    }

    final CacheKey cacheKey = (CacheKey) object;

    if (hashcode != cacheKey.hashcode) {
      return false;
    }
    if (checksum != cacheKey.checksum) {
      return false;
    }
    if (count != cacheKey.count) {
      return false;
    }

    for (int i = 0; i < updateList.size(); i++) {
      Object thisObject = updateList.get(i);
      Object thatObject = cacheKey.updateList.get(i);
      if (!ArrayUtil.equals(thisObject, thatObject)) {
        return false;
      }
    }
    return true;
  }
```


## 一级缓存的生命周期

从上面内容我们知道一级缓存是维护在 SqlSession 对象里的 Executor 对象中，那么它的最大生命周期也就是 **PerpetualCache <= Executor <= SqlSession**，


1. MyBatis在开启一个数据库会话时，会创建一个新的SqlSession对象；当会话结束时，SqlSession对象及其内部的Executor对象还有PerpetualCache对象也一并释放掉。


2. 如果SqlSession调用了close()方法，会释放掉一级缓存PerpetualCache对象
3. 如果SqlSession调用了rollback()方法，会清空PerpetualCache对象中的数据，但是该对象并未释放；
4. 如果SqlSession调用了clearCache()，会清空PerpetualCache对象中的数据，但是该对象并未释放；
5. 如果SqlSession调用了commit()，会清空PerpetualCache对象中的数据，但是该对象并未释放；
6. SqlSession中执行了任何一个update操作(update()、delete()、insert()) ，都会清空PerpetualCache对象的数据，但是该对象并未释放掉；在查询操作中，如果该 `MappedStatement` 设置了强制刷新缓存 (flushCache=true)，那么在去查询缓存map之前，会
   先清空PerpetualCache对象的数据；Mybatis一级缓存默认的 `scope` 是 SESSION 级别的， **如果设置成 `STATEMENT` 级别** ，那么在每次查询之后都会去清除缓存数据（相当于关闭一级缓存）。


>在Spring中使用Mybatis，SqlSession的生命周期是线程级别的，SqlSessionUtils 类中会将获取的SqlSession绑定到当前上下文中（内部使用ThreadLocal），所以SqlSession中的一级缓存最大生命周期也就是当前线程



## 使用一级缓存值得注意的点

1. 一级缓存没有更新缓存的概念，在查询中，只要命中缓存，那么直接返回缓存中的结果，不会再去数据库中查询。一级缓存也不会过期，如不清除缓存数据或缓存对象一直未被释放，那么它会一直存在。


2. 对于更新频繁的，并且需要高时效准确性的数据，使用 `SqlSession` 查询的时候，要控制好对象生存时间，生存时间越长，它其中缓存的数据有可能就越旧，
从而造成和真实数据的误差；同时对于这种情况，可以手动地适时清空SqlSession中的缓存，或设置强制刷新缓存，或设置一级缓存为STATEMENT


3. **一级缓存直接返回对象的唯一引用，如果直接修改将会影响缓存中的值** ![markdown](https://ddmcc-1255635056.file.myqcloud.com/3c44b99f-956e-4aa2-82e0-070cb6683190.png)


4. **由于一级缓存的范围是 `SqlSession` 的，所以当有多个 `SqlSession` 同时进行读写操作，可以会读取到脏数据**
![markdown](https://ddmcc-1255635056.file.myqcloud.com/49bd0787-ba67-4912-b1d4-99b95d1514b7.png)



## 其它

Mybatis文档对本地缓存的介绍：

>Mybatis 使用到了两种缓存：本地缓存（local cache）和二级缓存（second level cache）。每当一个新 session 被创建，MyBatis 就会创建一个与之相关联的本地缓存。任何在 session 执行过的查询结果都会被保存在本地缓存中，所以，当再次执行参数相同的相同查询时，就不需要实际查询数据库了。本地缓存将会在做出修改、事务提交或回滚，以及关闭 session 时清空。
 默认情况下，本地缓存数据的生命周期等同于整个 session 的周期。由于缓存会被用来解决循环引用问题和加快重复嵌套查询的速度，所以无法将其完全禁用。但是你可以通过设置 localCacheScope=STATEMENT 来只在语句执行时使用缓存。
 注意，如果 localCacheScope 被设置为 SESSION，对于某个对象，MyBatis 将返回在本地缓存中唯一对象的引用。对返回的对象（例如 list）做出的任何修改将会影响本地缓存的内容，进而将会影响到在本次 session 中从缓存返回的值。因此，不要对 MyBatis 所返回的对象作出更改，以防后患。