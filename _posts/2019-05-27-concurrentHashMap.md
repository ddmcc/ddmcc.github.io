---
layout: post
title:  "ConcurrentHashMap"
date:   2019-05-27 20:11:52
categories: 并发编程
tags: ConcurrentHashMap
author: ddmcc
---

* content
{:toc}




## JDK1.8之前ConcurrentHashMap

- concurrentHashMap在JDK1.5出现的，为了解决HashMap线程不安全问题和Hashtable使用synchronized导致并发性能低问题。

- 在1.7中使用分段锁来提升map的并发性能。在 `ConcurrentHashMap` 有一个 **$\color{red}{Segment}$** 数组，
(**Segment<K,V>** 是ConcurrentHashMap内部类)，将HashMap分成多个段(默认分成16个Segment，通过hash来定位Segment的位置),将锁的颗粒度降低至一个段(即一个Segment数组)。

- **$\color{red}{Segment}$** 继承了 **$\color{red}{ReentrantLock}$** 表示Segment是一个可重入锁,
ConcurrentHashMap通过可重入锁来实现分段锁机制。

- 在Segment类中有一个volatile HashEntry<K,V>[] table桶数组(HashEntry也是ConcurrentHashMap的一个内部类,用作储存键值数据的节点代表一个桶),而每个桶又是一个单向链表。


结构如图:

![](https://ws3.sinaimg.cn/large/005BYqpggy1g3g6375ewdj30x00ig3zq.jpg)


   源码如下:
    

    public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
            implements ConcurrentMap<K, V>, Serializable {
    
        final Segment<K,V>[] segments;
        static final class Segment<K,V> extends ReentrantLock implements Serializable {
            transient volatile HashEntry<K,V>[] table;
            transient int count;
        }
       
        // 桶
        static final class HashEntry<K,V> {
            final int hash;
            final K key;
            volatile V value;
            volatile HashEntry<K,V> next;
        }
    }


## JDK1.8的ConcurrentHashMap

> 1.8中主要有两点不同

- 抛弃了分段锁,利用数组+链表+红黑树来实现,对数组的每个元素来加锁,将锁的颗粒度降至每个节点(即每个桶)。ConcurrentHashMap中有一个volatile Node<K,V>[] table数组,Node<K,V>是ConcurrentHashMap的一个内部类,继承Map.Entry<K,V>。

- 增加了红黑树来保存数据。尽管好的hash算法能降低冲突,但在大的扩容因子和大数据量下,也会提高冲突的几率(扩容因子小会影响性能,因为扩容很消耗性能),
当每个桶的Node节点大于等于8时,将单向链表转为红黑树来提高查询效率(如果数组长度>64才会转),单向链表查询的复杂度为O(n),红黑树的查询复杂度为O(logn),
所以能提高查询的效率。

- 当删除红黑树时,如果数量<=6,则会转回单向链表。

结构如图:

![](https://ws3.sinaimg.cn/large/005BYqpggy1g3g7bx3j8sj30cl0bqdg2.jpg)


### 链表转为红黑树

    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
	    // 如果table < MIN_TREEIFY_CAPACITY,则不转,直接扩容,MIN_TREEIFY_CAPACITY默认为64
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);

            // 通过CAS得到指定位置Node节点,加锁转换
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                    new TreeNode<K,V>(e.hash, e.key, e.val,
                                            null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }



### 红黑树转为链表

    static <K,V> Node<K,V> untreeify(Node<K,V> b) {
        Node<K,V> hd = null, tl = null;
        for (Node<K,V> q = b; q != null; q = q.next) {
            Node<K,V> p = new Node<K,V>(q.hash, q.key, q.val, null);
            if (tl == null)
                hd = p;
            else
                tl.next = p;
            tl = p;
        }
        return hd;
    }



## 常用方法

### get方法

    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }


获取key的hash值,再通过 `speed()` 方法,对高位也进行hash,然后在查询对应的桶。

    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }


然后通过tabAt()放法获取链表或红黑树的第一个节点,然后遍历通过key的hash值查询相应的value。
tabAt()方式是通过Unsafe类的getObjectVolatile方法来获取值,volatile可以保证值的可见性,从而保证读到的值是最新的。


### put方法

    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }

- 1,数组为空,是否有数据,为空则先初始化table数组,然后再次进入循环。

- 2,判断目标桶上首节点是否为空,为空则直接不加锁新增节点。

- 3,判断目标桶是否在扩容操作(MOVED为正在转移的节点的hash值，值为-1),返回扩容后的table数组,重新进入循环。

- 4,对桶进行加锁(即对单向链表或红黑树),判断链表或红黑树进行新增Node节点,当onlyIfAbsent为false时会替换原来的值。

- 5,新增后判断binCount是否需要转换结构,oldVal不为空直接返回oldVal,oldVal不为空一种可能是本来就是红黑树结构了,还有就是链表结构,但是没有新增节点。

- 6,调用addCount增加table中Node的数量,有可能会触发扩容操作。



#### size方法

有size()和mappingCount()两个方法能获取元素的个数。在添加和删除元素时，会通过CAS操作更新ConcurrentHashMap的baseCount属性值来统计元素个数。但是CAS操作可能会失败，因此，ConcurrentHashMap又定义了一个CounterCell数组来记录CAS操作失败时的元素个数

    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }

    // 只返回int个数
    public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }


    public long mappingCount() {
        long n = sumCount();
        return (n < 0L) ? 0L : n; // ignore transient negative values
    }