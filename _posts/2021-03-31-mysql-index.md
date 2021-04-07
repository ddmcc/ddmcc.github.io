---

layout: post
title:  "mysql索引"
date:   2021-03-31 16:56:23
categories: mysql
tags:  mysql
author: ddmcc
---

* content
{:toc}





## 索引原理

和字典、通讯录原理一样，索引的目的在于提高查询效率，通过不断的缩小数据的查找范围来筛选出最终想要的结果。比如要查找 "张三" 这个人，会先定位在姓 “张” 的人中，然后再从中查找 “三” 这个名。如果没有索引，那么可能需要把整个通讯录都翻一遍才能找到想要的。

在 mysql 中存储引擎用类似的方法使用索引，其先通过值在索引中找到匹配的索引记录，然后根据索引记录来找到对应的数据行。又因为是 `B+` 树索引，按照顺序存储数据，所以可以用来做 `ORDER BY` 和 `GROUP BY` 操作。最后，因为索引存储了实际的列值，所以某些查询只使用索引就能够完成全部查询。据此特性总结索引有以下优点：

- 索引大大减少服务器需要扫描的数据量

- 索引可以帮助服务器避免排序和临时表

- 索引可以将随机IO变为顺序IO

  

> 这边要注意的是B+树索引并不能找到一个给定键值的具体行，能找到的只是被查找数据行所在的页。然后数据库通过把页从磁盘读入到内存，再在内存中进行查找，最后得到要査找的数据。



## **磁盘IO与预读**



在介绍索引之前，先了解一下磁盘IO，数据库数据文件是存在磁盘上的，磁盘读取数据靠的是机械运动，每次读取数据花费的时间可以分为寻道时间、旋转延迟、传输时间三个部分。

### **寻道时间**

Tseek是指将读写磁头移动至正确的磁道上所需要的时间。寻道时间越短，I/O操作越快，目前磁盘的平均寻道时间一般在3-15ms，主流磁盘一般在5ms以下

### **旋转延迟**

Trotation是指盘片旋转将请求数据所在的扇区移动到读写磁盘下方所需要的时间。通常用磁盘旋转一周所需时间的1/2表示，比如一个磁盘7200转表示：每分钟能转7200次，也就是说1秒钟能转120次，旋转延迟就是1 / 120 / 2 = 4.17ms

### **传输时间**

指的是从磁盘读出或将数据写入磁盘的时间，一般在零点几毫秒，相对于前两个时间可以忽略不计。



那么访问一次磁盘的时间，即一次磁盘IO的时间约等于5+4.17 = 9ms左右。考虑到磁盘IO是非常高昂的操作，计算机操作系统做了一些优化，当一次IO时，不光把当前磁盘地址的数据，而是把相邻的数据也都读取到内存缓冲区内，因为局部预读性原理告诉我们，当计算机访问一个地址的数据的时候，与其相邻的数据也会很快被访问到。每一次IO读取的数据我们称之为一页(page)。具体一页有多大数据跟操作系统有关，一般为4k或8k，也就是我们读取一页内的数据时候，实际上才发生了一次IO



## **索引数据结构**

在 mysql 中，索引是 `存储引擎` 层实现的，并没有统一的标准。不同的存储引擎的索引实现并不一样。即使多个存储引擎支持同一种类型的索引，其底层的实现也可能不同。如：`InnoDB` 中根据主键引用被索引的行，而 `MyISAM` 则通过数据的物理位置引用被索引的行



在 `InnoDB` 中，表中数据都是根据主键顺序存放的，这种存储方式的表叫 **索引组织表**。而 `聚簇索引` 就是按照每张表的主键构造的一棵 `B+` 树，树中同时保存了索引和数据行，数据行被存放在索引的叶子页中。也将 `聚簇索引` 的叶子节点称为 `数据页`。因为无法同时把数据行存放在两个不同的地方，所以一个表只能有一个 `聚簇索引`。如果没有定义主键，`InnoDB` 会选择一个唯一非空索引字段代替，如果没有这样的索引，那么 `InnoDB`会隐式定义一个 `6` 字节的 `rowId` 来作为 `聚簇索引`



> 唯一非空索引：是唯一索引（unique key）并且该字段的值not null



并且 **每一个索引 `InnoDB` 都会为其单独维护一颗 `B+树`**。



假设我们有这么一张表：

```sql
CREATE TABLE `user` (
  `id` bigint(20) NOT NULL,
  `name` varchar(50) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `IDX_AGE` (`age`) USING BTREE
) ENGINE=InnoDB;
```



那么 `InnoDB` 就会为主键 `id` 维护一个 `聚簇索引` ，为索引 `age` 维护一个非主键索引，也叫 `二级索引`。



![markdown](https://ddmcc-1255635056.file.myqcloud.com/8c6a15ad-cbe5-4489-8def-479b9f84416f.png)



![markdown](https://ddmcc-1255635056.file.myqcloud.com/0c60cc55-8414-428b-8325-08f650b8ce83.png)



在上图中可以看出，**二级索引的叶子页存的是主键的值，而聚簇索引叶子页存的是整行数据，聚簇索引的叶子节点也称为数据页**。非叶子节不存储数据行，只存储指向下层叶子的指针和索引键值的虚记录，如19并不真实存在于数据表中



### **一页16kb**

就如上面所说的，磁盘IO是非常高昂的操作。为了减少磁盘IO次数，存储引擎也做了很多的优化。比如会整页读取数据并把一些热数据放在缓冲池中。不同存储引擎缓存单位并不相同，`InnoDB`在 **默认情况下是 16kb** ，如果 `InnoDB`做一个单行查找需要读取磁盘，就需要把包含该行的整个页面读入缓冲池进行缓存。假设要随机访问100字节的行， `InnoDB` 将用掉很多缓冲池中额外的内存来缓存这些行，因为每一次都必须读取和缓存一个完整的 `16kb` 页面。而`InnoDB` 索引页大小也是 `16kb` ，意味着访问一个100字节的行可能一共要使用`32kb`的缓存空间（有可能更多，取决于索引树有多深） 



`B+` 索引在数据库中有一个特点是 `高扇出性` ，因此在数据库中，`B+` 树的高度一般都在2 ～ 4层，也就是说查找某一键值的行记录时最多只需要2 ～ 4次IO。前面分析了一次IO大概9ms，意味着查询时间只需0.02 ～ 0.04秒。

> 扇出：是指该模块直接调用的下级模块的个数



以一个整数字段（bigint）索引为例，字段长度为`8b` ，另外每个索引还跟着 `6b` 指向子树的指针，则每页可以存放索引 16kb * 1024 / 14 b = 1170。这棵树高是 4 的时候，就可以存 1170 的 3 次方个值，这已经 16亿了。考虑到树根的数据块总是在内存中的，那么一个 10 亿行的表上一个整数字段的索引，查找一个值最多只需要3 次磁盘IO。这也就是 **索引字段越小越好** 的原因。因为磁盘块的大小也就是一个数据页的大小，是固定的（默认`16k`），那么在总数据量固定的情况下，如果数据项占的空间越小，则每页能存的数据项数量越多，那么树的高度越低。这也是为什么 `b+` 树要求把真实的数据放到叶子节点而不是内层节点，一旦放到内层节点，每个磁盘块的数据项会大幅度下降，导致树增高



### **索引页、数据页逻辑连续**

每一层的页通过双向链表链接（Page Header中的PAGE_PREV和PAGE_NEXT记录上、下页的位置），页按照索引键的顺序排序；另外每个页中的记录也是通过单向链表进行维护的（Recorder Header的最后两个字节记录下一行偏移量）。按照索引键排序的好处就是对于索引键的排序查找和范围查找非常快。如要查找年龄最大的两个人id，由于`b+树`索引是双向链表的，可以快速找到最后一个数据页，并取最后两条数据



```sql
mysql> EXPLAIN SELECT id FROM user ORDER BY id LIMIT 2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user
   partitions: NULL
         type: index
possible_keys: NULL
          key: PRIMARY
      key_len: 8
          ref: NULL
         rows: 2
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```



可以看到 Extra 信息中并没有 Using filesort，说明并没有进行额外的排序操作。如果排序字段不是索引字段，那么就会在内存中进行额外的排序操作



未使用索引进行排序：

```sql
mysql> EXPLAIN SELECT ID FROM user ORDER BY name LIMIT 2\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: user
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 9
     filtered: 100.00
        Extra: Using filesort
1 row in set, 1 warning (0.00 sec)
```



### **通过二分+遍历查找**

`B+` 树页内索引使用类似 **跳表** 查找。在定位到了页之后，通过页目录（Page Directory）来进行 `二分查找` ，定位到距离数据较近的槽点（Slot）

// todo 描述页内查找过程及图



## **索引的查找**



基于聚簇索引和普通二级索引的查询有什么区别呢？



如下面语句，根据 `id` 查找：

```sql
SELECT * FROM user WHERE id = 1;
```



即主键查询的方式，则只需要搜索 **id** 这棵 `B+` 树。在多数情况下，查询优化器倾向于采用聚簇索引，因为聚簇索引索引能够在 `B+` 树索引的叶子节点上直接找到数据。此外，由于定义了数据的逻辑顺序，聚簇索引能够很快地访问针对范围值的查询。查询优化器能够快速发现某一段范围的数据页需要扫描



再如下面语句，根据 `age` 来查找：

```sql
SELECT * FROM user WHERE age = 15;
```



即普通索引查询方式，则需要先搜索 `age`索引树，得到 `id` 的值为 33，再拿着33到 `id` 索引树搜索一次，这个过程称为回表

**也就是说，基于非主键索引的查询需要多扫描一棵索引树。因此，我们在应用中应该尽量使用主键查询**。当然也不是每次用非主键索引查询都会回表，如果要查询的字段都在这棵索引书上，就不需要回表操作



## **索引的维护** 

![markdown](https://ddmcc-1255635056.file.myqcloud.com/b6615bde-2f14-42d4-949d-b5ebec377a15.png)



#### **页分裂**

`B+` 树为了维护索引有序性，在插入新值的时候需要做必要的维护。以上面这个图为例，**第一种情况：** 如果插入点的新值为6，则只需要4后面插入一条新的记录。**第二种情况：** 如果插入新的值为20，这时无法直接在25后面直接插入，需要逻辑上挪动后面的数据，空出位置。更糟的情况是，如果所在的页已经满了，这时就要新建一个新的页，并把部分数据挪过去，这个过程称为页分裂。在这种情况下，不管性能还是空间利用率都会受到影响



**所以通常都会建议使用 `AUTO_INCREMENT` 自增列作为主键** ，这样可以保证数据行是按顺序写入的。最好避免随机的聚簇索引，比如使用 `UUID`作为主键，这样使得聚簇索引的插入变得完全随机。

如果使用自增列作为主键，每次插入都是追加操作，就如上面说的第一种情况。当达到页的最大填充因子时（InnoDB默认最大填充因子是页大小的 15/16，留出部分空间用于后续修改），下一条记录就会写入新的页中。不涉及到挪动其它记录，也不会触发叶子节点的分裂



而有业务逻辑的字段做主键或者使用UUID，则不能保证有序插入，这样写入数据的成本相对较高。总结以下是一些缺点：

- 写入目标页可能已经刷到磁盘上并从缓存中清除，或者是还没有被加载到缓存中，InnoDB在插入之前不得不先找到并从磁盘读取目标页到内存。这将导致大量的随机I/O
- 因为写入是乱序的，InnoDB不得不频繁地做页分裂操作，以便为新的行分配空间，页分裂会导致移动大量的数据，一次插入最少需要修改三个页而不是一个页
- 由于频繁的页分裂，页会变得稀疏并被不规则的填充，最终数据会有碎片



而且通常业务主键都比较大，比如用身份证号做主键，那么每个二级索引的叶子节点都需要存储，这也大大的增加了索引的空间占用。而如果使用整型做主键，则只需4个字节，长整型（bigint）也只需8字节。所以，从性能和存储空间方面考量，自增主键往往是更合理的选择





#### **页合并**

删除记录时，不会实际删除该记录。相反，它将记录标记为已删除，并且它所使用的空间可以被其它记录声明使用



![图5](https://www.percona.com/blog/wp-content/uploads/2017/04/Locality_3.png)

<center style="font-size:14px;color:#ddd;text-decoration:underline">图1</center> 

当页中删除的记录达到`MERGE_THRESHOLD`（默认页体积的50%），InnoDB会开始寻找最靠近的页（前或后）看看是否可以将两个页合并以优化空间使用



![图6](https://www.percona.com/blog/wp-content/uploads/2017/04/Locality_4.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">图2</center> 



在本例中，页面 `图2` 占用的空间不足一半。`图1` 又有足够的删除数量，现在使用率也不到50%。从InnoDB的角度来看，它们是可合并的



![](https://www.percona.com/blog/wp-content/uploads/2017/04/Locality_5.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">图3</center> 



合并操作的结果（图3）是图1包含它以前的数据加上图2的数据，图2变成一个空页，可以接纳新数据



![](https://www.percona.com/blog/wp-content/uploads/2017/04/Locality_6.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">合并后的图2</center> 



当我们进行UPDATE操作，并且使页中记录数量低于阈值时，InnoDB也会进行一样的操作

规则是：页合并发生在删除或更新操作中，关联到当前页的相邻页。如果合并操作成功，在`INFOMATION_SCHEMA.INNODB_METRICS`中的`index_page_merge_successful`将会增加



[了解更多关于页合并与分裂](https://www.percona.com/blog/2017/04/10/innodb-page-merging-and-page-splitting/)



## **建索引的原则和使用时注意点**



以下表结构来说明：

```sql
CREATE TABLE `t` (
  `a` varchar(32) NOT NULL,
  `b` varchar(50) DEFAULT NULL,
  `c` varchar(20) DEFAULT NULL,
  `d` varchar(20) DEFAULT NULL,
  `e` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`a`),
  KEY `IDX_B_C_D` (`b`,`c`,`d`) USING BTREE
) ENGINE=InnoDB;
```



### **原则**



#### **最左前缀匹配原则**

mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如b = 2 and c > 3 and d = 4 如果建立(b,c,d)顺序的索引，d是用不到索引的，如果建立(b,d,c)的索引则都可以用到，b,d的顺序可以任意调整，因为查询优化器会帮你优化成索引可以识别的形式

**总结为以下三点：**

- **如果不是按照索引的最左列开始查找，则无法使用索引**

  

下面查询语句，不是最左列开始查找：

```shell
mysql> EXPLAIN SELECT * FROM t WHERE c = 'bbb'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 3
     filtered: 33.33
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```



可以看到并没有使用索引。如果从最左列开始查找，并且使用右模糊，也是可以使用索引进行匹配的，因为可以按左前缀字符查找

```shell
mysql> EXPLAIN SELECT * FROM t WHERE b LIKE 'aa%'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t
   partitions: NULL
         type: range
possible_keys: IDX_B_C_D
          key: IDX_B_C_D
      key_len: 153
          ref: NULL
         rows: 1
     filtered: 100.00
        Extra: Using index condition
1 row in set, 1 warning (0.01 sec)
```





- **不能跳过索引中的列**

也就是说上面的索引无法用于查找 b = 'xxx' and d = 'xxx' 的记录。如果不指定b的值，则 `mysql` 只能使用索引的第一列。在索引 `IDX_B_C_D ` 中匹配到所有 b = 'xxx' 的记录，再拿着这些记录的主键去聚簇索引中查找



- **如果查询中有某个列是范围查询，则其右边所有列都无法使用索引**

例如查询 b = 'xxx' and c >/</between/like 'xxx' and d = 'xxx' 这个查询只能用前面两列，因为列c是一个范围查询。但是存储引擎会用d列进行查询优化（索引下推）。如果范围查询的列值有限，那么可以通过使用多个等于条件来代替范围条件



#### **尽量选择区分度高的列作为索引**

区分度的公式是 **count(distinct col) / count(*)**，表示字段不重复的比例，比例越大查询效率越高，唯一键的区分度是1。而一些状态、性别字段可能在大数据面前区分度就是0

存在非等号和等号混合判断条件时，在建索引时，等号条件的列前置。如：where c > ? and d = ? 那么即使 c 的区分度更高，也必须把 d 放在索引的最前列，即建立组合索引 idx_d_c。**当然这也只是“经验法则”，还需要考虑到索引的复用能力**

根据[美团技术团队博客文章](https://tech.meituan.com/2014/06/30/mysql-index.html)，需要join的字段这个值一般都要求是0.1以上，即平均1条扫描10条记录



#### **尽量的扩展索引，不要新建索引**

- 空间：`InnoDB` 会为每个索引都建立 `B+` 树索引，所以会占用更多的空间
- 性能：每次修改数据时要对索引进行维护，多棵索引树无疑增加了维护成本。并且在查询上，多列索引有机会使用 `覆盖索引` 和 `索引下推` 来提升查询效率



### **使用时注意点**

- 索引列不能参与计算：比如 from_unixtime(create_time) = ’2014-05-29’  或 left(code,  6) = '010108'  或 score + 1 = 80 就不能使用到索引。所以语句应该写成create_time = unix_timestamp(’2014-05-29’)  和 code LIKE '010108%'
- 索引列与参数类型不匹配：比如字符串字段索引 phone = 13024532432
- 最左前缀匹配：比如左模糊查询
- ...



## **索引的使用**



#### **联合索引**

`联合索引` 也是一棵 `B+` 树，不同的是联合索引的键值数量不是 1，而是大于等于 2。并且和单个键值的 `B+` 树一样，键值都是排序的，通过叶子节点可以逻辑上读出所有的数据。

如有以下表：

```sql
CREATE TABLE t1 ( 
a INT, 
b INT, 
c INT, 
PRIMARY KEY ( a ), 
KEY IDX_B_C ( b, c ) ) 
ENGINE = INNODB;
```



索引先按字段b排序，如果b字段值相等，再按c字段排序，即(1, 1)、(1, 2)、(2, 1)、(2, 4)、(3, 1)、(3, 2) 数据按(b, c) 的顺序进行了存放。因为对第二字段进行了排序，在很多场景下可以利用这个特性来避免多一次排序操作。如查询一个人的最近3笔订单，则可以建立(user_id,  create_time) 的联合索引，建表、查询语句为：

```sql
CREATE TABLE `buy_order` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `IDX_USER_CREATE_TIME` (`user_id`,`create_time`) USING BTREE,
  KEY `IDX_USER` (`user_id`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=6;

INSERT INTO `user`.`buy_order`(`id`, `user_id`, `create_time`) VALUES (1, 1, '2021-04-04 23:14:27');
INSERT INTO `user`.`buy_order`(`id`, `user_id`, `create_time`) VALUES (2, 1, '2021-04-07 23:14:35');
INSERT INTO `user`.`buy_order`(`id`, `user_id`, `create_time`) VALUES (3, 1, '2021-04-07 23:14:43');
INSERT INTO `user`.`buy_order`(`id`, `user_id`, `create_time`) VALUES (4, 1, '2021-04-30 23:14:52');
INSERT INTO `user`.`buy_order`(`id`, `user_id`, `create_time`) VALUES (5, 2, '2021-04-14 23:15:49');
```



为了做比较还加了一个单独的IDX_USER索引，先查询用户为1的所有订单，这个语句应该是两个索引都可以使用：

```sql
mysql> EXPLAIN SELECT * FROM buy_order WHERE user_id = 1\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: buy_order
   partitions: NULL
         type: ref
possible_keys: IDX_USER_CREATE_TIME,IDX_USER
          key: IDX_USER_CREATE_TIME
      key_len: 9
          ref: const
         rows: 4
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```



可以看到上面的执行计划，优化器最终选择了单个索引的IDX_USER，**因为该索引的节点包含单个键值，所以理论上一个页能存放更多的记录** 



接着假设要查询用户为1的最近三个订单：

```sh
mysql> EXPLAIN SELECT * FROM buy_order WHERE user_id = 1 ORDER BY create_time DESC LIMIT 3\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: buy_order
   partitions: NULL
         type: ref
possible_keys: IDX_USER_CREATE_TIME
          key: IDX_USER_CREATE_TIME
      key_len: 9
          ref: const
         rows: 4
     filtered: 100.00
        Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)
```



同样的，对于这个查询既可以使用 IDX_USER 索引，也可以使用 IDX_USER_CREATE_TIME 索引。但是这次优化器使用了 IDX_USER_CREATE_TIME 的联合索引，**因为在这个联合索引中 create_time 已经排序好了，根据该联合索引取出数据，无须再对create_time 做一次额外的排序操作**，若强制使用 IDX_USER索引，则执行计划如下图：



```shell
mysql> EXPLAIN SELECT * FROM buy_order FORCE INDEX(IDX_USER) WHERE user_id = 1 ORDER BY create_time DESC LIMIT 3\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: buy_order
   partitions: NULL
         type: ref
possible_keys: IDX_USER
          key: IDX_USER
      key_len: 9
          ref: const
         rows: 4
     filtered: 100.00
        Extra: Using index condition; Using filesort
1 row in set, 1 warning (0.00 sec)
```



在Extra选项中可以看到 `Using filesort`，即需要额外的一次排序操作才能完成查询。而这次显然需要对列 create_time 排序，因为索引 IDX_USER 中的 create_time 是未排序的



正如前面所介绍的那样，联合索引(b, c)其实是根据列b、c进行排序，因此下列语句可以直接使用联合索引得到结果：

```sql
SELECT * FROM TABLE WHERE a = ? ORDER BY b
```



然而对于联合索引(a,b,c)来说，下列语句同样可以直接通过联合索引得到结果：

```sql
SELECT * FROM TABLE WHERE a = ? ORDER BY b
SELECT * FROM TABLE WHERE a = ? AND b = ? ORDER BY c
```





#### **覆盖索引**



如果执行的语句是 :

```sql
SELECT ID FROM t WHERE k BETWEEN 3 AND 5;
```



这时只需要查 ID 的值，而 ID 的值已经在 k 索引树上了，因此可以直接提供查询结果，不需要回表。也就是说从二级索引就可以得到查询的字段，而不需要查询聚簇索引中的记录（mysql5.0或以下版本不支持）



使用覆盖索引另一个好处是针对某些统计，如下面查询语句：

```sql
SELECT COUNT(*) FROM t WHERE b = ?;
```



假设 b 字段也是表的主键，并且b字段上还有其它的索引，InnoDB存储引擎并不会通过聚簇索引来进行统计。因为二级索引远小于聚簇索引，选择二级索引可以减少IO操作

此外，通常情况下，如有索引(a, b)的联合索引，是不可以选择作为列b的查询索引。但是如果是统计操作，并且是覆盖索引，则优化器会选择它

比如以下查询语句：

```sql
SELECT COUNT(a) FROM t WHERE b = ?;
```



#### **索引下推**（ICP）　

从 `mysql5.6` 开始支持的一种根据索引进行查询的优化方式



如有索引(a, b)的联合索引和以下查询语句：

```sql
SELECT * FROM t WHERE a LIKE 'a%' AND b = ?;
```

在 `mysql5.6` 之前，根据最左前缀匹配规则，存储引擎只能查询出符合条件 `LIKE 'a%'` 的记录，然后把结果集返回给 `Server` 层，再拿着一个个主键去聚簇索引查找，然后在`Server` 层过滤条件 `b = ?` 

在引入下推优化之后（mysql5.6），如果部分WHERE条件能使用索引中的字段，`Server` 层会把这部分（上面b条件）下推到引擎层（调用引擎接口的时候把这部分也传过去）。存储引擎在二级索引遍历过程中对 `b` 字段先做判断，直接过滤掉不满足条件的记录。`ICP` 减少了 `Server` 层访问存储引擎的次数和引擎层访问聚簇索引的次数（减少回表次数）。总之是 ` ICP` 的优化在引擎层就能够过滤掉大量的数据，减少 `IO` 次数，提高查询语句性能



> 控制参数：SET @@optimizer_switch="index_condition_pushdown=on"



#### **MRR优化**（Multi-range Read）

从 `mysql5.6` 开始支持 `MRR` 优化，`MRR` 优化的目的是为了减少磁盘的随机访问，并将随机访问转化为较为顺序的数据访问。

在不使用 `MRR` 时，优化器根据二级索引返回的记录来进行回表，因为这个记录是根据二级索引键排序的，一般会有较多的随机IO。比如在二级索引中查询到的数据集为：(key=1, pk=99)，(key=2, pk=88)，(key=14, pk=875)，(key=15, pk=76)，这时如果按这个顺序去聚簇索引查找，因为主键无序，可能会有更多的随机IO和缓冲区（buffer pool）中的页被替换出，然后又不断地被读入缓冲区，若是按照主键顺序进行访问，则可以将此重复行为降到最低



**使用MRR时，SQL语句的执行过程是这样的：**

1)  优化器将二级索引查询到的记录放到一块缓冲区中

2)  如果二级索引查询完成或者缓冲区已满，则使用快速排序对缓冲区中的内容按照主键进行排序

3)  根据主键的排序来访问聚簇索引

4)  当缓冲区中的列表取完数据后，则继续调用过程 2) 3)，对超过缓存区大小的部分继续排序查询，直至扫描结束

通过上述过程，优化器将二级索引随机的 IO 进行排序，转化为主键的有序排列，从而实现了随机 IO 到顺序 IO 的转化，提升性能





> 控制参数：
>
> 用optimizer_switch 的标记来控制是否使用MRR.设置mrr=on时，表示启用MRR优化。
>
> mrr_cost_based表示是否通过cost base的方式来启用MRR.
>
> 当mrr=on,mrr_cost_based=on,则表示cost base的方式还选择启用MRR优化,当发现优化后的代价过高时就会不使用该项优化
>
> 当mrr=on,mrr_cost_based=off,则表示总是开启MRR优化
>
> ```
> SET @@optimizer_switch='mrr=on,mrr_cost_based=on';
> ```

> 缓冲区参数：
>
> 参数 read mnd_ bufter_size 用来控制键值的缓冲区大小，当大于该值时，则执行器对已经缓存的数据根据主键进行排序，并通过 主键来取得行数据。该值默认为256K



#### **怎么给字符串字段加索引**

有时需要给较长的字符串字段加索引，这会使得索引变得大且慢。通常，可以定义字符串一部分前缀作为索引，这样可以大大节约索引空间，从而提高索引效率。但这样也会降低索引的区分度，还可能使查询变得更慢

> 为什么说减小索引大小就能提高查询效率呢？首先较长的字符串在索引排序和值比较时势必会更慢。还有索引页大小都是固定的，较小的话每一页能够存放更多的索引，这样也能减少IO次数。重要的是节省空间



假设，现在维护一个支持邮箱登录的系统，用户表是这么定义的：



```shell
mysql> create table SUser(
Id bigint unsigned primary key,
email varchar(64) 
) engine=innodb; 
```



我们可以直接给 `email` 字段添加索引：

```shell
alter table SUser add index index1(email);
```



这样创建的索引键值包含整个 `email` 字符串。也可以指定只取前面几个字符创建索引，如：

```shell
mysql> alter table SUser add index index2(email(6));
```



这样两种定义方式的索引的数据结构和存储是这样的：

![markdown](https://ddmcc-1255635056.file.myqcloud.com/5a745e89-5287-48c0-958b-b684ec490b5e.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">index1</center> 



![markdown](https://ddmcc-1255635056.file.myqcloud.com/cfb104cc-8da7-4307-ab25-5b4a9d426584.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">index2</center> 



从图中可以看到，由于 `email(6)` 这个索引结构中邮箱只取前6个字符，所以占用空间会更小，这就是前缀索引的优势。但同时带来的问题是，**可能额外增加回表次数**



根据下面这条查询语句，来分析上面两种索引的执行过程：

```sql
mysql> select id,name,email from SUser where email='aaaaaab@gmail.com';
```



如果使用的是 **index1**（即整个字符串），执行顺序是这样的：

1. 从二级索引中找到满足值是 `aaaaaab@gmail.com` 的记录，取得 id = 200
2. 到聚簇索引上找到 id 为 200 的行





#### **普通索引和唯一索引怎么选择**



## **其它索引**



#### **哈希索引**



#### **全文索引**







## 参考

**《高性能MySQL第3版》**

**《MySQL技术内幕InnoDB存储引擎第2版》**

[MySQL索引原理及慢查询优化](https://tech.meituan.com/2014/06/30/mysql-index.html)

[磁盘I/O那些事](https://tech.meituan.com/2017/05/19/about-desk-io.html)

