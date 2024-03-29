---
layout: post
title:  "mysql执行过程"
date:   2021-04-14 16:56:23
categories: mysql
tags:  mysql
toc: true
---


## 一条SQL查询语句是如何执行的

下面是 `mysql` 查询的基本执行路径示意图：

<!-- more -->

![markdown](https://ddmcc-1255635056.file.myqcloud.com/40fd5017-8a29-4da8-bfe7-d3bb19ec0599.png)

大体来说，**mysql可以分为 Server 层和存储引擎层两部分：**

- **Server** 层包括：连接器、查询缓存、语法解析器、优化器、执行器等大部分核心服务功能，以及所有的内置函数（如日期、时间、数学加密函数等）。所有跨存储引擎的功能都在这一层实现：如存储过程、触发器、视图等

- **存储引擎** 层负责数据的存储和提取。其架构是插件式的，支持 `InnoDB` , `MyISAM` , `Memory` 等多个存储引擎。从 `mysql 5.5.5` 版本开始默认引擎为 `InnoDB`

  

> 插件式的也就意味着存储引擎是可插拔的。我们可以在表 a 使用 InnoDB，在表 b 使用 memory 。不同存储引擎的表数据存取方式不同，支持的功能也不同。我们也可以自定义存储引擎 [15.9示例存储引擎](http://www.searchdoc.cn/rdbms/mysql/dev.mysql.com/doc/refman/5.7/en/example-storage-engine.com.coder114.cn.html)



从上面图中可以看出，不同的存储引擎公用一个 `Server ` 层，也就是从 `连接器` 到 `执行器` 的部分。接下来看看每个组件都做了什么：



#### **连接器**

首先需要连接到数据库，这时接待你的就是连接器。连接器负责跟客户端建立连接、校验账号、获取权限、维持和管理连接。每个客户端都会在服务器中拥有一个线程，这个连接的查询只会在这个单独的线程中执行。从 `mysql 5.5` 开始支持线程池，所以服务器不需要为每一个新建的连接创建或销毁线程



连接命令一般这么写：

```shell
mysql -h $ip -P $port -u $user -p
```



在连接上后，连接器就会对其进行身份认证。认证基于用户名、密码和原始主机信息。如果使用了安全套接字（ssl）的方式连接，还可以使用证书认证。如果用户名、密码不对或主机没有权限，就会收到 `Access denied for user 'xxx@ip'` 的错误。一旦连接成功，连接器会继续查出该连接所拥有的权限（例如：是否允许对 user 库的 t 表执行 select 语句）。**这个连接后续的权限判断都依赖于此时读到的权限，这也就意味着一个用户成功连接后，即使修改了它的权限，也不会受影响，只有重新新建连接才会使用新的权限设置**



连接完成后，如果没有做后续的动作，这个连接就处于空闲的状态，通过 `show processlist` 命令可以看到 `Command` 为 sleep 的就是空闲连接。如果空闲一定长的时间，连接器就会自动将它断开。这个时间通过参数 `wait_timeout` 控制，默认为28800秒=8小时



#### **查询缓存**

在连接建立完成后，就可以开始执行查询语句了。这时来到第二步：`查询缓存`，这边所指的“缓存”，指的是完整的查询结果。当接收到一个查询请求后，会先到查询缓存看看，当查询命中缓存，会立刻返回结果，跳过了 **解析、优化、和执行阶段**。如果未命中，就会继续后面执行阶段，执行完成后，执行结果会被存入查询缓存中



查询缓存以 key-value 的形式被直接缓存在内存中，这个key其实是一个大小写敏感哈希值，这个哈希值包括了语句本身、查询的数据库、参数等一些其它会影响查询结果的信息，即使只有一个字节的不同都会导致缓存不命中。当查询中包含任何用户自定义函数、存储函数、用户变量、临时表、mysq库中的系统表，或者任何包含列级别权限的表，都不会被缓存。



查询缓存还会跟踪查询中涉及的每个表，如果这些表发生变化，那么和这个表相关的所有缓存数据都将失效，即使数据表变化时对缓存中的结果可能并没有影响。这也就是**大多数情况下会建议不要使用查询缓存**的原因，对于更新频繁的数据库来说，查询缓存命中率会非常低，除非是静态表，很少才有更新数据。mysql团队也意识到查询缓存的问题，自 mysql 5.6（2013 年）以来，查询缓存已被默认禁用，**在 mysql8.0 版本中查询缓存模块直接被移除了** [MySQL 8.0：退出查询缓存支持](https://mysqlserverteam.com/mysql-8-0-retiring-support-for-the-query-cache/)



#### **语法解析器和预处理**



如果没有命中查询缓存，这时就会到解析器，开始真正的执行语句。解析器会先使用 mysql 语法规则验证和解析查询，例如，它会验证是否使用错误的关键字，或者关键字使用顺序问题，再或者引号、括号前后能否正确匹配，之后生成一颗对应的 “解析树”。如果语句不对，就会收到 “You have an error in your SQL syntax” 的错误提醒

预处理器则根据一些规则进行进一步的检查解析树是否合法，例如检查数据表、数据列是否都存在，解析名字和别名是否有歧义。最后会验证权限



#### **查询优化器**



现在解析树被认为是合法的了，现在将由优化器将其转化为执行计划。一条语句可以有很多种执行的方式，虽然最后的查询结果都是相同的，优化器的作用就是找到这其中最好的方式



`mysql` 使用基于成本的优化器，它会 **预测** 一个查询使用某种执行计划时的成本，并选择其中成本最小的一个 [更多关于优化器内容](todo)



#### **执行器**



在优化器阶段完成后，查询对应的执行计划就已经生成好了，执行器则根据这个执行计划来完成整个查询。开始执行的时候，会先判断这个表是否有权限，如果没有就直接返回权限错误（上查询缓存阶段，如果命中缓存，在返回缓存结果的时候也会做权限校验。在优化器之前也会做precheck权限校验）。执行器简单的根据执行计划的指令逐步执行，在执行过程中，有大量的操作需要通过调用储存引擎 api 来完成，这些接口被称为 “handler API" 。存储引擎接口有着非常丰富的功能，但是底层接口却只有几十个。例如,有一个査询某个索引的第一行的接口，再有个査询某个索引条目的下一个条目的功能，有了这两个功能我们就可以完成全索引扫描的操作了。这种简单的接口模式，让  mysql 的存储引擎插件式架构成为可能



```sql
SELECT * FROM T WHERE ID = 10；
```



比如我们这个例子中的表 T 中，ID 字段没有索引，那么执行器的执行流程是这样的：

- 调用 InnoDB引擎取这个表的第一行的接口，判断 ID 值是不是 10，如果不是则跳过，如果是则将这行存在结果集中；

- 调用引擎接口取“下一行”，重复相同的判断逻辑，直到取到这个表的最后一行

- 执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端

  

对于有索引的表，执行的逻辑也差不多，第一次调用的是“取满足条件的第一行”这个接口，之后循环取“满足条件的下一行”这个接口，这些接口都是引擎中已经定义好的



查询最后一步是将结果集返回给客户端，即使查询不需要返回结果集给客户端，mysql 仍然会返回一些查询的信息，如查询影响到的行数。如果查询可以被缓存，那么在这个阶段也会将结果存放到查询缓存中。你会在数据库的慢查询日志中看到一个 rows_examined 的字段，表示这个语句执行过程中扫描了多少行。这个值就是在执行器每次调用引擎获取结果集的时候累加的。在有些场景下，执行器调用一次，在引擎内部则扫描了多行，因此引擎扫描行数跟 rows_examined 并不是完全相同的

最后 mysql 将结果集返回给客户端是一个增量，逐步返回的过程。 在执行器得到第一条结果时，就可以向客户端逐步返回结果集了。这样的好处是：

- 服务端无须存储太多的结果，也就不会因为要返回太多结果而消耗太多内存
- 让客户端第一时间获得返回结果



结果集中的每一行都会以一个满足 mysql 客户端/服务器通信协议的封包发送，再通过TCP协议进行传输，在TCP传输的过程中,可能对。mysql 的封包进行缓存然后批量传输



## 一条更新查询语句是如何执行的

更新语句前面的步骤和查询一致，解析器通过解析知道这是一条更新语句，然后执行器选择最优执行计划去执行



- 执行器会先open table，如果该表上有 `MDL（X）` （元数据排他锁），则等待。如果没有则在该表上加 `MDL（S）` （元数据共享锁）
- 进入到引擎层，首先会去 `innodb_buffer_pool` 里的 `data dictionary` (元数据信息，是InnoDB自己管理的表缓存) 得到表信息，通过元数据信息，去 `lock info` 里查出是否会有相关的锁信息，并把这条update语句需要的锁信息写入到 `lock info` 里
- 然后涉及的旧数据以快照的形式存储到缓冲池中的 `undo page` 里，并在 redo log 中记录 undo log（undo log持久化）
- 对数据页进行修改，并把数据页的物理修改记录到 redo log buffer里。由于一个事务会涉及到多个页面的修改，所以redo log buffer里会有多条页面的修改信息。并且由于 group_commit 的原因，本次事务所产生的redo log buffer可能会跟其它事务一同刷新到磁盘上
- 同时修改的信息，会按照 `event` 的格式，以不同的 event type 记录到 binlog_cache 中，在 **事务 commit 后** dump 线程会从 binlog_cache 里把 event 发送给 slave 的 I/O 线程
- 之后把还需要在二级索引上做的修改，写入到 change buffer page，等到下次有读取该二级索引页时，再去与二级索引页做 merge
- commit操作，由于存储引擎层与 server 层之间采用的是内部 XA (保证两个事务的一致性，这里主要保证redo log和binlog的原子性)，所以提交分为prepare阶段与commit阶段
  - 引擎将新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态
  - 执行器将 binlog_cache 里的日志进行刷新到磁盘，并进行同步操作
  - 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成






## **参考**

[MYSQL实战 45讲](https://time.geekbang.org/column/article/68319) 

**《高性能MySQL第3版》**

