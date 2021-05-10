---

layout: post
title:  "锁与事务隔离级别"
date:   2021-05-10 16:56:23
categories: mysql
tags:  mysql
author: ddmcc
---

* content
{:toc}





## **悲观锁和乐观锁**

- 悲观锁

在整个数据处理过程中，将数据处于锁定状态。悲观锁的实现，往往依靠数据库提供的锁机制（也只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据）

在悲观锁的情况下，为了保证事务的隔离性，就需要 **一致性锁定读**。读取数据时给加锁，其它事务无法修改这些数据。修改删除数据时也要加锁，其它事务无法读取这些数据

- 乐观锁(**一致性非锁定读**)

相对悲观锁而言，乐观锁机制采取了更加宽松的加锁机制。悲观锁大多数情况下依靠数据库的锁机制实现，以保证操作最大程度的独占性

而乐观锁机制在一定程度上解决了这个问题。**乐观锁，大多是基于数据版本（ Version ）记录机制实现**。在基于数据库表的版本解决方案中，一般是通过为数据库表增加一个 “version” 字段来实现。读取出数据时，将此版本号一同读出，之后更新时，对此版本号加一。此时，将提交数据的版本数据与数据库表对应记录的当前版本信息进行比对，如果提交的数据版本号大于数据库表当前版本号，则予以更新，否则认为是过期数据



## **Read Committed**



在RC级别中，数据的读取都是不加锁的（**采用一致性非锁定读**），但是数据的写入、修改和删除是需要加锁的（**仅使用Record Lock**）

为了防止并发过程中的修改冲突，事务 A 给 `a = 4` 的数据行加锁，并一直不commit（释放锁），那么事务 B 也就一直拿不到该行锁，wait直到超时

上面这种情况是在 a 是有索引情况下，如果是没有索引的 e 字段呢？

```sql
update t set a=3 where e = 1; 
```



那么MySQL会给整张表的所有数据行的加行锁。MySQL并不知道哪些数据行是 e = 1 的，如果一个条件无法通过索引快速过滤，存储引擎层面就会将所有记录加锁后返回，再由MySQL Server层进行过滤

但在实际使用过程当中，MySQL做了一些改进，在MySQL Server过滤条件，发现不满足后，会调用 `unlock_row` 方法，把不满足条件的记录释放锁 (违背了二段锁协议的约束)。这样做，保证了最后只会持有满足条件记录上的锁，但是每条记录的加锁操作还是不能省略的。（参见《高性能MySQL》中文第三版p181）

这种情况同样适用于MySQL的默认隔离级别RR。所以对一个数据量很大的表做批量修改的时候，如果无法使用相应的索引，MySQL Server过滤数据的时候特别慢，就会出现虽然没有修改某些行的数据，但是它们还是被锁住了的现象



## **Repeatable Read**

RC（不可重读）模式下的展现

| 事务A                                                        | 事务B                                                      |
| :----------------------------------------------------------- | :--------------------------------------------------------- |
| begin;                                                       | begin;                                                     |
| select id,class_name,teacher_id from class_teacher where teacher_id=1;<br />id \| class_name \| teacher_id<br />1  \| 初三二班     ｜1<br />2  \| 初三一班     ｜1 |                                                            |
|                                                              | update class_teacher set class_name='初三三班' where id=1; |
|                                                              | commit;                                                    |
| select id,class_name,teacher_id from class_teacher where teacher_id=1;<br />id \| class_name \| teacher_id<br />1  \| 初三三班      \| 1<br />2  \| 初三一班      \| 1 <br /><br />读到了事务B修改的数据，和第一次查询的结果不一样，是不可重读的。 |                                                            |
| commit;                                                      |                                                            |



事务 B 修改 `id = 1` 的数据提交之后，事务 A 同样的查询，后一次和前一次的结果不一样，这就是不可重读（重新读取产生的结果不一样）

这就很可能带来一些问题，那么我们来看看在RR级别中MySQL的表现：

| 事务A                                                        | 事务B                                                        | 事务C                                                        |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| begin;                                                       | begin;                                                       | begin;                                                       |
| select id,class_name,teacher_id from class_teacher where teacher_id=1;<br />id \| class_name \| teacher_id<br />1  \| 初三二班      \| 1<br />2  \| 初三一班      \| 1 |                                                              |                                                              |
|                                                              | update class_teacher set class_name='初三三班' where id=1;<br />commit; |                                                              |
|                                                              |                                                              | insert into class_teacher values (null,'初三三班',1);<br />commit; |
| select id,class_name,teacher_id from class_teacher where teacher_id=1;<br />id \| class_name \| teacher_id<br />1  \| 初三二班      \| 1<br />2  \| 初三一班      \| 1 <br /><br />没有读到事务B修改的数据，和第一次sql读取的一样，是可重复读的<br />没有读到事务C新添加的数据。 |                                                              |                                                              |
| commit;                                                      |                                                              |                                                              |



我们注意到，当 `teacher_id=1` 时，事务 A 先做了一次读取，事务 B 中间修改了` id=1` 的数据，并commit之后，事务A第二次读到的数据和第一次完全相同。所以说它是可重读的



**在 RR 隔离级别下，对于读 InnoDB 存储引擎同样使用 `一致性非锁定读` ，但加锁上却和RC不同，其使用 Next-Key Lock** 



## **MVCC在InnoDB中的实现**

在InnoDB中，会在每行数据后添加两个额外的隐藏的值来实现 MVCC，这两个值一个记录这行数据创建事务id（**DB_TRX_ID**），另外一个记录这行回滚指针（**DB_ROLL_PTR**）

>DB_TRX_ID：记录插入或更新该行的事务的事务ID。
>
>DB_ROLL_PTR：回滚指针，指向 undo log 记录。每次对某条记录进行改动时，该列会存一个指针，可以通过这个指针找到该记录修改前的信息 。当某条记录被多次修改时，该行记录会存在多个版本，通过DB_ROLL_PTR 链接形成一个类似版本链的概念。
>
>



每开启一个新事务，事务的版本号就会递增。 在可重读Repeatable reads事务隔离级别下：

- SELECT时，读取行创建版本号<=当前事务版本号，并且删除版本号为空或>当前事务版本号
- INSERT时，保存当前事务版本号为行的创建版本号
- DELETE时，保存当前事务版本号为行的删除版本号
- UPDATE时，插入一条新纪录，保存当前事务版本号为行创建版本号，同时保存当前事务版本号到原来删除的行作为删除版本号

通过MVCC，虽然每行记录都需要额外的存储空间，更多的行检查工作以及一些额外的维护工作，但可以减少锁的使用，大多数读操作都不用加锁，读数据操作很简单，性能很好，并且也能保证只会读取到符合标准的行，也只锁住必要行



**MVCC 在mysql 中的实现依赖的是 undo log 与 read view **

 [**undo log**](http://ddmcc.cn/2021/04/18/mysql-log-files/)

根据行为的不同，undo log分为两种： `insert undo log` 和 `update undo log`



#### **insert undo log：**
  insert 操作中产生的undo log，因为insert操作记录只对当前事务本身课件，对于其他事务此记录不可见，所以 insert undo log 可以在事务提交后直接删除而不需要进行purge操作。

> purge的主要任务是将数据库中已经 mark del 的数据删除，另外也会批量回收undo pages



数据库 Insert 时的数据初始状态：



![markdown](https://ddmcc-1255635056.file.myqcloud.com/e8ec94cf-9413-4367-b80d-2d04be0d4e8d.png)



#### **update undo log：**

update 或 delete 操作中产生的 undo log。 因为会对已经存在的记录产生影响，为了提供 MVCC机制，因此update undo log 不能在事务提交时就进行删除，而是将事务提交时放到入 history list 上，等待 purge 线程进行最后的删除操作。


数据第一次被修改时：

![markdown](https://ddmcc-1255635056.file.myqcloud.com/fad0465e-d79a-400b-897d-d4a0e56d5db7.png)



当另一个事务第二次修改当前数据：



![markdown](https://ddmcc-1255635056.file.myqcloud.com/857ac367-fb87-4543-8788-4e531d1b41e7.png)



#### **ReadView**



在 Innodb 中每个SQL语句执行前都会得到一个read view。 **主要保存了当前数据库系统中正处于活跃（没有commit）的事务的ID号**，其实简单的说这个副本中保存的是系统中当前不应该被本事务看到的其他事务id列表



**Read view 的几个重要属性**

- **trx_ids:** 当前系统活跃(未提交)事务版本号集合。

- **low_limit_id:** 创建当前read view 时“当前系统最大**事务版本号**+1”。

- **up_limit_id:** 创建当前read view 时“系统正处于**活跃事务**最小版本号”

- **creator_trx_id:** 创建当前read view的事务版本号；



**ReadView 匹配条件**



**（1）数据事务ID < up_limit_id 则显示**

如果数据事务ID小于read view中的最小活跃事务ID，则可以肯定该数据是在当前事务启之前就已经存在了的，所以可以显示

**（2）数据事务ID >= low_limit_id 则不显示**

如果数据事务ID大于 read view 中的当前系统的最大事务ID，则说明该数据是在当前read view 创建之后才产生的，所以数据不予显示

**（3） up_limit_id <** 数据事务ID < **low_limit_id 则与活跃事务集合 **trx_ids **里匹配**

如果数据的事务ID大于最小的活跃事务ID，同时又小于等于系统最大的事务ID，这种情况就说明这个数据有可能是在当前事务开始的时候还没有提交的。

所以这时候我们需要把数据的事务ID与当前read view 中的活跃事务集合trx_ids 匹配:

**情况1:** 如果事务ID不存在于trx_ids 集合（则说明read view产生的时候事务已经commit了），这种情况数据则可以显示。

**情况2：** 如果事务ID存在trx_ids则说明read view产生的时候数据还没有提交，但是如果数据的事务ID等于creator_trx_id ，那么说明这个数据就是当前事务自己生成的，自己生成的数据自己当然能看见，所以这种情况下此数据也是可以显示的

**情况3：** 如果事务ID既存在trx_ids而且又不等于creator_trx_id那就说明read view产生的时候数据还没有提交，又不是自己生成的，所以这种情况下此数据不能显示



**（4）不满足read view条件时候，从undo log里面获取数据**

当数据的事务ID不满足read view条件时候，从undo log里面获取数据的历史版本，然后数据历史版本事务号回头再来和read view 条件匹配 ，直到找到一条满足条件的历史数据，或者找不到则返回空结果；





## todo 流程




RR级别是可重复读的，但无法解决幻读，而只有在Serializable级别才能解决幻读。于是我就加了一个事务C来展示效果。在事务C中添加了一条teacher_id=1的数据commit，RR级别中应该会有幻读现象，事务A在查询teacher_id=1的数据时会读到事务C新加的数据。但是测试后发现，在MySQL中是不存在这种情况的，在事务C提交后，事务A还是不会读到这条数据。可见在MySQL的RR级别中，是解决了幻读的读问题的。参见下图

![innodb_lock_1](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/6eb5d3b1.png)



读问题解决了，根据MVCC的定义，并发提交数据时会出现冲突，那么冲突时如何解决呢？我们再来看看InnoDB中RR级别对于写数据的处理。



#### **在RC和RR隔离级别下MVCC的差异**

在事务隔离级别 READ COMMITTED和 REPEATABLE READ( INNODB存储引整 的默认事务隔离级别)下, INNODB存储引擎使用非锁定一致性读。然而,对于快照数据的定义却不相同。在 READ COMMITTED事务隔离级别下,对于快照数据,非一致性读总是读取被锁定行的最新一份快照数据。而在 REPEATABLE READ事务隔离级别下,对于快照数据,非一致性读总是读取事务开始时的行数据版本





#### **MVCC解决不可重复读问题**





#### **Next-Key Lock 解决幻读问题**

在默认隔离级别 `REPEATABLE READ` 下，InnoDB 中行锁默认使用算法 Next-Key Lock，只有当查询的索引是唯一索引或主键时，InnoDB会对 Next-Key Lock 进行优化，将其降级为 Record Lock，即仅锁住索引本身，而不是范围

当查询的索引为辅助索引时，InnoDB则会使用Next-Key Lock进行加锁。InnoDB对于辅助索引有特殊的处理，不仅会锁住辅助索引值所在的范围，还会将其下一键值加上Gap Lock



```sql
CREATE TABLE e4 (a INT, b INT, PRIMARY KEY(a), KEY(b));
INSERT INTO e4 SELECT 1,1;
INSERT INTO e4 SELECT 3,1;
INSERT INTO e4 SELECT 5,3;
INSERT INTO e4 SELECT 7,6;
INSERT INTO e4 SELECT 10,8;
```



然后执行下面的语句：

```text
SELECT * FROM e4 WHERE b=3 FOR UPDATE;
```



因为通过辅助索引b来进行查询，所以 InnoDB 会使用 Next-Key Lock 进行加锁，并且还会对主键索引a进行加锁。对于主键索引a，仅仅对值为5的索引加上 Record Lock。而对于索引b，需要加上 Next-Key Lock 索引，锁定的范围是(1,3]。除此之外，还会对其下一个键值加上Gap Lock，即还有一个范围为(3,6)的锁

再新开一个会话，执行下面的SQL语句，会发现都会被阻塞：

```text
SELECT * FROM e4 WHERE a = 5 FOR UPDATE; # 主键a被锁
INSERT INTO e4 SELECT 4,2; # 插入行b的值为2，在锁定的(1,3]范围内
INSERT INTO e4 SELECT 6,5; # 插入行b的值为5，在锁定的(3,6)范围内
```



若此时没有 Gap Lock 锁定(3,6) ，虽然会话A锁住了 b = 3 这条记录，但是会话B可以插入一条值为4的记录，这会导致会话A中再次执行查询时会返回不同的记录，即导致幻读问题

InnoDB 存储引擎采用 Next-Key Lock 来解决幻读问题。因为 Next-Key Lock 是锁住一个范围，所以就不会产生幻读问题。但是需要注意的是，**InnoDB 只在 Repeatable Read 隔离级别下使用该机制**



## **参考**

**《MySQL技术内幕InnoDB存储引擎第2版》**

[Innodb中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)

[MySQL事务与MVCC如何实现的隔离级别](https://blog.csdn.net/qq_35190492/article/details/109044141)

[Innodb MVCC实现原理](https://zhuanlan.zhihu.com/p/52977862)