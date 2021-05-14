---

layout: post
title:  "mysql日志文件"
date:   2021-04-18 16:56:23
categories: mysql
tags:  mysql
author: ddmcc
---

* content
{:toc}





## bin log

二进制日志 ( binary log) 是 `Server` 层的日志，**记录了对 mysql 数据库执行更改的所有操作**，但是不包括 SELECT和SHOW这类操作，因为这类操作对数据本身并没有修改。然而，若操作本身并没有导致数据库发生变化，那么该操作可能也会写入二进制日志。例如执行下列未更改的sql：



```sql
mysql> UPDATE t SET b = 2 WHERE a = 1;
Query OK, 0 rows affected (0.00 sec)
Rows matched: 0  Changed: 0  Warnings: 0
```



`Changed：0` 说明没有数据被修改。但通过 `show master status` 查看binlog日志文件和 `SHOW BINLOG EVENTS IN 'mysql-bin.000006'` 查看日志内容可以看到在 `binlog` 中的确进行了记录



```shell
mysql> show master status\G;
*************************** 1. row ***************************
             File: mysql-bin.000006
         Position: 482
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)

mysql> SHOW BINLOG EVENTS IN 'mysql-bin.000006'\G
*************************** 1. row ***************************
   Log_name: mysql-bin.000006
        Pos: 4
 Event_type: Format_desc
  Server_id: 123456
End_log_pos: 123
       Info: Server ver: 5.7.31-log, Binlog ver: 4
*************************** 2. row ***************************
   Log_name: mysql-bin.000006
        Pos: 123
 Event_type: Previous_gtids
  Server_id: 123456
End_log_pos: 154
       Info: 
*************************** 3. row ***************************
   Log_name: mysql-bin.000006
        Pos: 154
 Event_type: Anonymous_Gtid
  Server_id: 123456
End_log_pos: 219
       Info: SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
*************************** 4. row ***************************
   Log_name: mysql-bin.000006
        Pos: 219
 Event_type: Query
  Server_id: 123456
End_log_pos: 298
       Info: BEGIN
*************************** 5. row ***************************
   Log_name: mysql-bin.000006
        Pos: 298
 Event_type: Query
  Server_id: 123456
End_log_pos: 402
       Info: use `user`; UPDATE t SET b = 2 WHERE a = 1
*************************** 6. row ***************************
   Log_name: mysql-bin.000006
        Pos: 402
 Event_type: Query
  Server_id: 123456
End_log_pos: 482
       Info: COMMIT
6 rows in set (0.01 sec)
```



`binlog` 日志文件默认并没有开启，需要手动指定参数来启动。通过 **log-bin=filename **可以启动日志，如果不指定filename，则默认日志文件名为主机名

```shell
log_bin=/var/lib/mysql/mysql-bin
binlog_format=mixed # 选择 ROW 模式
server_id=123456 
```



据官方手册中的测试表明，开启binlog会使数据库性能下降1%。下面为binlog相关配置参数：

-  max_binlog_size：单个日志文件最大值（默认1g），如果超过会产生新的文件，后缀名 + 1
- binlog_cache_size：未提交的事务日志会被记录到一个缓存中，等该事务提交时直接将缓存中的日志写入到日志文件，该缓存大小就由该配置控制，默认32k。这个缓存是基于会话（session）的，因此当开始一个事务时，会自动为其分配一个配置大小的缓存
- sync_binlog：`sync_binlog=N` 表示写缓冲多少次就同步磁盘。如果设为1，即同步写磁盘的方式来写日志。但是，即使将sync_ binlog设为1，还是会有一种情况导致问题的发生。当一个事务发出 `COMMIT` 动作之前，因为同步写，因此会将日志立即写人磁盘。如果这时已经写人了日志，但是提交还没有发生，并且此时发生了宕机，那么在数据库下次启动时，由于 `COMMIT` 操作并没有发生，这个事务会被回滚掉。但是日志已经记录了该事务信息，不能被回滚。 这个问题可以通过将参数 innodb_ support_xa设为1来解决，虽然 innodb_ support_xa与XA事务有关，**但它同时也确保了binlog和 INNODB存储引擎数据文件的同步**
- binlog-do-db：表示需要写入哪些库日志，默认全部
- binlog-ignore-db：表示需要忽略哪些库日志，默认无
- log-slave-update ：
- binlog_format：日志记录格式STATEMENT,ROW,MIXED



#### **文件格式**

binlog记录的是 **逻辑日志**，格式分为 `statement`，`row` 以及 `mixed` 三种。在主从同步种一般不建议使用statement，因为有些语句不支持如uuid函数。一般使用 row 模式

- statement：记录具体的执行sql，某些语句和函数如UUID, LOAD DATA INFILE等在复制过程可能导致数据不一致甚至出错
- row：基于行的模式，记录的是行每个字段的变化，以及事件类型等，大小会比其他两种模式大很多
- mixed：混合模式，根据语句来选用是statement还是row模式



#### **作用**

binlog主要用来point-in-time恢复和主/从复制同步，从这里就可以衍生出许多应用场景

- **1.读写分离**

  ![markdown](https://ddmcc-1255635056.file.myqcloud.com/50f38b45-d325-4378-84b0-ffc5c386720a.png)

  - 有一个主库Master，所有的更新操作都在master上进行

  - 同时会有多个Slave，每个Slave都连接到Master上，获取binlog在本地回放，实现数据复制。

  - 在应用层面，需要对执行的sql进行判断。所有的更新操作都通过Master(Insert、Update、Delete等)，而查询操作(Select等)都在Slave上进行。由于存在多个slave，所以我们可以在slave之间做负载均衡。通常业务都会借助一些数据库中间件，如tddl、sharding-jdbc等来完成读写分离功能

    

- **2.数据恢复**

  如果数据误删，或需要恢复到某个时间点，则可以通过较旧时间点的版本+binlog来恢复，或通过全量binlog

  

- **3.数据最终一致性**

  在实际开发中，我们经常会遇到一些需求，在数据库操作成功后，需要进行一些其他操作，如：更新缓存或者更新搜索引擎中的索引等

  **如何保证数据库操作与这些行为的一致性，就成为一个难题**。以数据库与redis缓存的一致性为例：操作数据库成功了，可能会更新redis失败；反之亦然。很难保证二者的完全一致

  这时我们就可以利用binlog。如果数据库操作成功，必然会产生binlog。之后，我们通过一个组件（canal等），来模拟的mysql的slave，拉取并解析binlog中的信息。**通过解析binlog的信息，去异步的更新缓存、索引或者发送MQ消息，保证数据库与其他组件中数据的最终一致**

  ![markdown](https://ddmcc-1255635056.file.myqcloud.com/7ee0475d-a45c-42c2-89dd-d63f0df66b1a.png)

- **4.异地多活，数据同步**

  

#### **写入时机**

binlog只在 **事务提交前** 进行一次写入。什么时候从缓存刷新到磁盘跟参数`sync_binlog`相关


// todo



## **redo log**

重做日志用来实现事务

// todo

#### **文件格式**





#### **作用**

**重做日志用来保证事务的持久性。**为了更好的性能，InnoDB会将数据缓存在内存中（Buffer Pool），对数据的修改也是先修改缓存中的，所以磁盘数据会落后于内存，这时如果进程或机器奔溃，会导致内存数据丢失。为了维护数据库本身的一致性和持久性，InnoDB维护了redo log，修改 `page` 之前需要先将修改的内容记录到redo log中，并保证 redo log早于对应的page落盘，也就是通常说的WAL（Write Ahead Log）。当故障发生导致内存数据丢失后，InnoDB会在重启时，通过重放redo，将page数据恢复到奔溃前的状态



#### 写入时机





## bin log与redo log区别

**1.** redo log是InnoDB引擎特有的；binlog 是 mysql 的 Server 层实现的，所有引擎



## undo log


// todo




## **参考**

[MYSQL实战 45讲](https://time.geekbang.org/column/article/68319) 

**《高性能MySQL第3版》**

