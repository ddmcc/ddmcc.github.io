---

layout: post
title:  "InnoDB存储引擎中的锁"
date:   2021-04-26 16:56:23
categories: mysql
tags:  mysql
author: ddmcc
---

* content
{:toc}




## InnoDB存储引擎中的锁



##  InnoDB存储引擎中的锁



我们都知道事务的4个特性，即 **ACID**。mysql 数据库使用加锁的方式来实现其中的 **I** (Isolation隔离性)。对于 InnoDB 存储引擎来说，它 **支持行锁和表锁** ，而且行锁是由存储引擎通过给索引记录加锁来实现的，并且 InnoDB 默认是加行锁。好处就是锁定颗粒度小，发生锁冲突的概率低，并发度高



#### **锁的类型**

InnoDB 实现了两种行级锁：

- 共享锁(**S**) : 允许事务读一行数据
- 排他锁(**X**)：允许事务删除或更新一行数据



**其中，X 锁与任何的锁都不兼容，而 S 锁仅和 S 锁兼容。**即如果一个事务已经获得了行的 **S锁**，那么另外的事务可以立即获得这行的 **S** 锁，因为读取并没有改变行的数据，称这种情况为锁兼容。 但若有其他的事务想获得行的 **X** 锁，则其必须等待事务释放行上的 **S** 锁



![markdown](https://ddmcc-1255635056.file.myqcloud.com/79cd2235-2d37-45cf-8708-72dbc65aee1b.png)



InnoDB 存储引擎除了行锁以外，还有表锁，通常也称为意向锁，其设计目的主要是为了指示事务稍后对表中的行需要哪种类型的锁（共享或独占）。其支持两种意向锁：

- 意向共享锁（IS Lock），事务想要获得一张表中某几行的共享锁
- 意向排他锁（IX Lock），事务想要获得一张表中某几行的排他锁



如果需要对页上的记录 r 进行上 **X** 锁，那么分别需要对数据库、表、页上 **IX**，最后对记录 r 上 **X** 锁。若中间有任何一个部分导致等待，那么该操作需要等待粗粒度锁释放。 举例来说，在对记录 r 加 **X** 锁之前，已经有事务对表进行了 **S** 表锁，那么表上已存在 **S** 锁，之后事务需要对记录 r 在表上加上 **IX**，由于不兼容，所以该事务需要等待表锁操作的完成



**InnoDB 存储引擎中锁的兼容性：**

![markdown](https://ddmcc-1255635056.file.myqcloud.com/955267a0-f07f-4335-9583-dcaee34290aa.png)

> **设计意向锁的目的何在?**

假设事务1，用 **X** 锁来锁住了页上的几条记录，那么此时表、页上存在 **IX** 锁，即意向排他锁。此时事务2要进行`LOCK TABLE … WRITE`的表级别锁的请求或者事务2进行 DML 操作且未使用索引而升级为表锁时，可以直接根据意向锁是否存在而判断是否有锁冲突。如果没有这个意向锁，那么可能需要遍历整个表记录才能知道表是不是有记录被锁



#### **DML表锁与MDL表锁**

mysql 里面表级别的锁有两种：一种是表锁，一种是元数据锁（metadata lock，MDL)

**表锁**

表锁的语法是 **lock tables … read/write。**可以用 `unlock tables` 主动释放锁，也可以在客户端断开的时候自动释放，`commit` 和 `rollback` 并不会释放锁（准确的说是InnoDB释放了内部表锁，server没释放表锁）

需要注意，lock tables 语法除了会限制别的线程的读写外，也限定了本线程接下来的写。举个例子，如果在某个线程 A 中执行 lock tables t1 read, t2 write; 这个语句，则其他线程写 t1、读写 t2 的语句都会被阻塞。同时，线程 A 在执行 unlock tables 之前，也只能执行读 t1、读写 t2 的操作。写 t1 并不允许，**也不能读写其他表**



**MDL元数据锁(metadata lock)**

MDL 不需要显式使用，在访问一个表的时候会被自动加上。在 `MySQL 5.5` 版本中引入了 MDL，当对一个表做增删改查操作的时候，加 MDL 读锁；当要对表做结构变更操作的时候，加 MDL 写锁

- MDL 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查
- 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行



#### **外键和锁**



#### **自增长与锁**



#### **锁的算法**

InnoDB 通过给索引项加锁来实现行锁，如果没有索引，则通过隐藏的聚簇索引来对记录加锁。如果操作不通过索引条件检索数据，InnoDB 则对表中的所有记录加锁，实际效果就和表锁一样

InnoDB 存储引擎有3种行锁的算法，分别是：

- Record Lock: 单个记录上的锁
- Gap Lock: 间隙锁，锁定一个范围，但不包括本记录
- Next-Key Lock: Gap Lock+Record Lock，锁定一个范围，并且锁定记录本身



![markdown](https://ddmcc-1255635056.file.myqcloud.com/4e3f16e6-5a2e-4bb7-b240-bf9c4ab1005b.png)

例如一个索引有10，30这两个值，InnoDB 可以根据需要使用 `Record Lock` 将10，30两个索引锁住；也可以使用 `Gap Lock` 将(-∞,10)，(10,30)，(30, +∞)三个范围区间锁住；`Next-Key Lock` 类似于上述两种锁的结合，它可以锁住的区间有为(-∞,10]，(10,30]，(30, +∞)，可以看出它即锁定了一个范围，也会锁定记录本身



InnoDB 对于行的查询都是默认采用 `Next-Key lock` 算法。**当条件索引是唯一索引时，InnoDB 存储引擎会进行优化，将其降级为 Record Lock，即锁住索引本身** ，而不是范围，从而提高并发性。若是通过辅助索引查询，**不但会给辅助索引加锁，还会为聚集索引上锁。对于辅助索引加 Next-Key Lock，而聚集索引因为是唯一的，所以只会加 Record Lock**



对于没有显示创建索引的表，则会对 rowId 的聚集索引来加锁。如果操作未使用索引查询，那么会对表中所有记录加锁，实际效果和表锁一样



## 锁与事务隔离级别



#### **Read Committed**

```sql
mysql> show create table t \G\
*************************** 1. row ***************************
       Table: t
Create Table: CREATE TABLE `t` (
  `a` varchar(32) NOT NULL,
  `b` varchar(50) DEFAULT NULL,
  `c` varchar(20) DEFAULT NULL,
  `d` varchar(20) DEFAULT NULL,
  `e` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`a`),
  KEY `IDX_B_C_D` (`b`,`c`,`d`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

mysql> select * from t;
+---+------+------+------+------+
| a | b    | c    | d    | e    |
+---+------+------+------+------+
| 4 | 1    | 12   | 1    | 1    |
+---+------+------+------+------+
1 row in set (0.01 sec)
```



在RC级别中，数据的读取都是不加锁的，但是数据的写入、修改和删除是需要加锁的

为了防止并发过程中的修改冲突，事务 A 给a=4的数据行加锁，并一直不commit（释放锁），那么事务 B 也就一直拿不到该行锁，wait直到超时

上面这种情况是在 a 是有索引情况下，如果是没有索引的其它字段呢？

```sql
update t set a=3 where e = 1; 
```



那么MySQL会给整张表的所有数据行的加行锁。MySQL并不知道哪些数据行是 e = 1 的，如果一个条件无法通过索引快速过滤，存储引擎层面就会将所有记录加锁后返回，再由MySQL Server层进行过滤

但在实际使用过程当中，MySQL做了一些改进，在MySQL Server过滤条件，发现不满足后，会调用unlock_row方法，把不满足条件的记录释放锁 (违背了二段锁协议的约束)。这样做，保证了最后只会持有满足条件记录上的锁，但是每条记录的加锁操作还是不能省略的。（参见《高性能MySQL》中文第三版p181）

这种情况同样适用于MySQL的默认隔离级别RR。所以对一个数据量很大的表做批量修改的时候，如果无法使用相应的索引，MySQL Server过滤数据的时候特别慢，就会出现虽然没有修改某些行的数据，但是它们还是被锁住了的现象



#### **Repeatable Read**

RC（不可重读）模式下的展现

| 事务A                                                        | 事务B                                                      |
| :----------------------------------------------------------- | :--------------------------------------------------------- |
| begin;                                                       | begin;                                                     |
| select id,class_name,teacher_id from class_teacher where teacher_id=1;<br />id \| class_name \| teacher_id<br />1  \| 初三二班     ｜1<br />2  \| 初三一班     ｜1 |                                                            |
|                                                              | update class_teacher set class_name='初三三班' where id=1; |
|                                                              | commit;                                                    |
| select id,class_name,teacher_id from class_teacher where teacher_id=1;<br />id \| class_name \| teacher_id<br />1  \| 初三三班      \| 1<br />2  \| 初三一班      \| 1 <br /><br />读到了事务B修改的数据，和第一次查询的结果不一样，是不可重读的。 |                                                            |
| commit;                                                      |                                                            |



事务B修改id=1的数据提交之后，事务A同样的查询，后一次和前一次的结果不一样，这就是不可重读（重新读取产生的结果不一样）。

这就很可能带来一些问题，那么我们来看看在RR级别中MySQL的表现：

| 事务A                                                        | 事务B                                                        | 事务C                                                        |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| begin;                                                       | begin;                                                       | begin;                                                       |
| select id,class_name,teacher_id from class_teacher where teacher_id=1;<br />id \| class_name \| teacher_id<br />1  \| 初三二班      \| 1<br />2  \| 初三一班      \| 1 |                                                              |                                                              |
|                                                              | update class_teacher set class_name='初三三班' where id=1;<br />commit; |                                                              |
|                                                              |                                                              | insert into class_teacher values (null,'初三三班',1);<br />commit; |
| select id,class_name,teacher_id from class_teacher where teacher_id=1;<br />id \| class_name \| teacher_id<br />1  \| 初三二班      \| 1<br />2  \| 初三一班      \| 1 <br /><br />没有读到事务B修改的数据，和第一次sql读取的一样，是可重复读的<br />没有读到事务C新添加的数据。 |                                                              |                                                              |
| commit;                                                      |                                                              |                                                              |

我们注意到，当teacher_id=1时，事务A先做了一次读取，事务B中间修改了id=1的数据，并commit之后，事务A第二次读到的数据和第一次完全相同。所以说它是可重读的



**悲观锁和乐观锁**

- 悲观锁

在整个数据处理过程中，将数据处于锁定状态。悲观锁的实现，往往依靠数据库提供的锁机制（也只有数据库层提供的锁机制才能真正保证数据访问的排他性，否则，即使在本系统中实现了加锁机制，也无法保证外部系统不会修改数据）

在悲观锁的情况下，为了保证事务的隔离性，就需要 **一致性锁定读**。读取数据时给加锁，其它事务无法修改这些数据。修改删除数据时也要加锁，其它事务无法读取这些数据

- 乐观锁

相对悲观锁而言，乐观锁机制采取了更加宽松的加锁机制。悲观锁大多数情况下依靠数据库的锁机制实现，以保证操作最大程度的独占性

而乐观锁机制在一定程度上解决了这个问题。**乐观锁，大多是基于数据版本（ Version ）记录机制实现**。在基于数据库表的版本解决方案中，一般是通过为数据库表增加一个 “version” 字段来实现。读取出数据时，将此版本号一同读出，之后更新时，对此版本号加一。此时，将提交数据的版本数据与数据库表对应记录的当前版本信息进行比对，如果提交的数据版本号大于数据库表当前版本号，则予以更新，否则认为是过期数据



#### **MVCC在InnoDB中的实现**

在InnoDB中，会在每行数据后添加两个额外的隐藏的值来实现MVCC，这两个值一个记录这行数据何时被创建，另外一个记录这行数据何时过期（或者被删除）。 在实际操作中，存储的并不是时间，而是事务的版本号，每开启一个新事务，事务的版本号就会递增。 在可重读Repeatable reads事务隔离级别下：

- SELECT时，读取创建版本号<=当前事务版本号，删除版本号为空或>当前事务版本号。
- INSERT时，保存当前事务版本号为行的创建版本号
- DELETE时，保存当前事务版本号为行的删除版本号
- UPDATE时，插入一条新纪录，保存当前事务版本号为行创建版本号，同时保存当前事务版本号到原来删除的行

通过MVCC，虽然每行记录都需要额外的存储空间，更多的行检查工作以及一些额外的维护工作，但可以减少锁的使用，大多数读操作都不用加锁，读数据操作很简单，性能很好，并且也能保证只会读取到符合标准的行，也只锁住必要行。

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

[Mysql锁：灵魂七拷问](https://tech.youzan.com/seven-questions-about-the-lock-of-mysql/) 

**《MySQL技术内幕InnoDB存储引擎第2版》**

[Innodb中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)



