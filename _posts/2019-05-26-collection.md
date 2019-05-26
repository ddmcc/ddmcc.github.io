---
layout: post
title:  "集合"
date:   2019-05-26 21:29:52
categories: 面试
tags: 集合 面试
author: ddmcc
---

* content
{:toc}




## 集合

(1) Collection和Collections的区别  

- Collections是个java.util下的类，它包含有各种有关集合操作的静态方法。
  - Collections.sort     对集合排序（需要实现int compareTo） 
  - Collections.synchronizedMap      返回一个线程安全的map 
  - Collections.binarySearch     二分查找一个元素 
  - Collections.shuffle      对集合进行随机排序（就是指每次排序后都不同） 
- Collection是个java.util下的接口，它是各种集合结构的父接口。 
  - List,Set的父接口
  - Collection跟Map没有联系

(2) List 和 Set 区别  

- list：列表，表达形式 [ ]，或者list()，有序，通过索引值进行查找
- set：集合,表达形式set([ ])，无序自动去重。可以做集合的交集，并集，差集

(3) Set内存放的元素为什么不可以重复，内部是如何保证和实现的？ 

- HashSet类实现了Set接口， 其底层其实是包装了一个HashMap去实现的。
HashSet采用HashCode算法来存取集合中的元素，因此具有比较好的读取和查找性能。
首先根据key的hashCode()返回值决定该Entry的存储位置，如果两个key的hash值相同，那么它们的存储位置相同。
如果这个两个key的equals比较返回true。那么新添加的Entry的value会覆盖原来的Entry的value，key不会覆盖

(4) Arraylist 与 LinkedList 区别   

- ArrayList是实现了基于动态数组的数据结构，而LinkedList是基于链表的数据结构；
- 对于随机访问get和set，ArrayList要优于LinkedList，因为LinkedList要移动指针；
- 对于添加和删除操作add和remove，一般大家都会说LinkedList要比ArrayList快，因为ArrayList要移动数据。
但是实际情况并非这样，对于添加或删除，LinkedList和ArrayList并不能明确说明谁快谁慢
> 所以当插入的数据量很小时，两者区别不太大，当插入的数据量大时，大约在容量的1/10之前，LinkedList会优于ArrayList，在其后就劣与ArrayList，且越靠近后面越差。所以个人觉得，一般首选用ArrayList，由于LinkedList可以实现栈、队列以及双端队列等数据结构，所以当特定需要时候，使用LinkedList，当然咯，数据量小的时候，两者差不多，视具体情况去选择使用；当数据量大的时候，如果只需要在靠前的部分插入或删除数据，那也可以选用LinkedList，反之选择ArrayList反而效率更高

(5) Arraylist与LinkedList,Map默认空间是多少； 

- jdk1.6 ArrayList 初始化大小是 10,扩容大小规则是，扩容后的大小= 原始大小+原始大小/2 + 1
- jdk1.7 ArrayList 初始化大小是 0,第一次添加元素时,会将容量设置为10,扩容大小规则是，扩容后的大小= 原始大小+原始大小/2 
- linkedList 是一个双向链表，没有初始化大小，也没有扩容的机制
- HashMap 初始化大小是 16 ，扩容因子默认0.75（可以指定初始化大小，和扩容因子）
  扩容机制.(当前大小 和 当前容量 的比例超过了 扩容因子，就会扩容，扩容后大小为 一倍。例如：初始大小为 16 ，扩容因子 0.75 ，当容量为12的时候，比例已经是0.75 。触发扩容，扩容后的大小为 32.)

(6) ArrayList 与 Vector 区别  

- Vector 是线程安全的，也就是说是它的方法之间是线程同步的，而 ArrayList 是线程
  序不安全
- ArrayList 与 Vector 都有一个初始的容量大小，当存储进它们里面的元素的个数超过
  了容量，就需要增加 ArrayList 与 Vector 的存储空间。Vector 增长原来的一倍， ArrayList 增加原来的0.5倍。

(7) HashSet 和 HashMap 区别  

- HashMap实现Map接口，存储键值对，HashMap使用键（Key）计算Hashcode
- HashSet实现Set接口，底层采用HashMap实现，仅存储对象，HashSet使用成员对象来计算hashcode值，对于两个对象来说hashcode可能相同，所以equals()方法用来判断对象的相等性，如果两个对象不同的话，那么返回false。

(8) HashMap 和 Hashtable 的区别  

- HashMap是非线程安全的，只是用于单线程环境下，多线程环境下可以采用concurrent并发包下的concurrentHashMap。HashMap中，null可以作为键，这样的键只有一个；
- Hashtable也是JDK1.0引入的类，是线程安全的，Hashtable中，key和value都不允许出现null值。
- 遍历方式： Hashtable、HashMap都使用了 Iterator。而由于历史原因，Hashtable还使用了Enumeration的方式 。
- 哈希值的使用不同，HashTable直接使用对象的hashCode。而HashMap重新计算hash值
- 扩容机制和初始化大小： HashTable在不指定容量的情况下的默认容量为11，而HashMap为16，Hashtable不要求底层数组的容量一定要为2的整数次幂，而HashMap则要求一定为2的整数次幂。Hashtable扩容时，将容量变为原来的2倍加1，而HashMap扩容时，将容量变为原来的2倍。 

(9) 谈谈HashMap，哈希表解决hash冲突的方法； 

- JDK1.8 之前 HashMap 底层是 数组和链表 结合在一起使用也就是 链表散列。HashMap 通过 key 的 hashCode 经过扰动函数处理过后得到 hash  值，然后通过 (n - 1) & hash 判断当前元素存放的位置（这里的 n 指的时数组的长度），如果当前位置存在元素的话，就判断该元素与要存入的元素的 hash 值以及 key 是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突。jdk1.8在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间,提高查询速度

(10) HashMap 和 ConcurrentHashMap 的区别  

- 从ConcurrentHashMap代码中可以看出，它引入了一个“分段锁”的概念，具体可以理解为把一个大的Map拆分成N个小的HashTable，根据key.hashCode()来决定把key放到哪个HashTable中。可以提供相同的线程安全，但是效率提升N倍，默认提升16倍。

(13) ConcurrentHashMap 的工作原理及代码实现，如何统计所有的元素个数  

- 这也就是为什么sumCount()中需要遍历counterCells数组，sum累加CounterCell.value值了
  (16) Java Collections和Arrays的sort方法默认的排序方法是什么； 

- java的Collections.sort算法调用的是合并排序
- Arrays.sort() 采用了2种排序算法 -- 基本类型数据使用快速排序法，对象数组使用归并排序

(17) ArrayList和LinkList的删除一个元素的时间复杂度；（ArrayList是O(N)，LinkList是O(1)） 

- ArrayList 是线性表（数组）
  - get() 直接读取第几个下标，复杂度 O(1)
  - add(E) 添加元素，直接在后面添加，复杂度O（1）
  - add(index, E) 添加元素，在第几个元素后面插入，后面的元素需要向后移动，复杂度O（n）
  - remove（）删除元素，后面的元素需要逐个移动，复杂度O（n）

- LinkedList 是链表的操作
  - get() 获取第几个元素，依次遍历，复杂度O(n)
  - add(E) 添加到末尾，复杂度O(1)
  - add(index, E) 添加第几个元素后，需要先查找到第几个元素，直接指针指向操作，复杂度O(n)
  - remove（）删除元素，直接指针指向操作，复杂度O(1)

(18) HashMap在什么时候时间复杂度是O（1），什么时候是O（n），什么时候又是O（logn）； 

- 链表的长度尽可能短，理想状态下链表长度都为1 
- 当 Hash 冲突严重时，如果没有红黑树，那么在桶上形成的链表会变的越来越长，这样在查询时的效率就会越来越低；时间复杂度为O(N)。 
- 采用红黑树之后可以保证查询效率O(logn)