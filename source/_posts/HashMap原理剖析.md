---
title: HashMap 原理剖析
date: 2019-12-26 15:37:48
tags:
categories: java
description: 基于JDK1.8详细分析了 HashMap
---

本文基于JDK1.8,相比于1.7会有区别，后面提到。

### 1. 基本结构

```java
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{
    /**
     * The table, resized as necessary. Length MUST Always be a power of two.
     */
    transient Entry<K,V>[] table;
}
```

HashMap 数据存储在一个 Entry 数组中，Entry 具体结构如下：

```java
static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
 }

```

可以看出 Entry 维护的是一个链表结构。

![1577348067073.png](https://i.loli.net/2019/12/27/NGWSiaY7hTyfLcQ.png)

HashMap 的构造方法如下：

```java
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        // tableSizeFor函数用来生成第一个大于 initialCapacity 的2的平方数
        this.threshold = tableSizeFor(initialCapacity);
    }
```

**initialCapacity** 是 HashMap 的初始化容量（实际的容量是第一个大于 initialCapacity 的2的平方数），默认为16。
**loadFactor** 为负载因子，默认为0.75。
**threshold** 是 HashMap 进行扩容的阀值，当 HashMap 的存放的元素个数超过该值时，会进行扩容，它的值为HashMap 的容量乘以负载因子。

table 不在构造函数中就初始化而是第一次 put 时，这点于 1.7 不同。

### 2. put 的实现

基本流程为：

1. 对 key 的 hashCode() 做 hash,然后计算 index；
2. 如果没有碰撞则直接放 table 里；
3. 如果碰撞了，以链表的形式存放在 table 后；
4. 如果碰撞导致链表过长(大于等于TREEIFY_THRESHOLD)，就把链表转换成红黑树；
5. 如果节点已经存在就替换old value(保证key的唯一性)；
6. 如果bucket满了(超过load factor*current capacity)，就要resize。

代码实现如下：

```java
public V put(K key, V value) {
    // 对key的hashCode()做hash
    return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // tab为空则创建
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 计算index，并对null做处理
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 节点存在
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 该链为树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 该链为链表,1.8 使用尾插法 1.7 为头插法
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 写入
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 超过load factor*current capacity，resize
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### 3. get 的实现

基本流程：

1. 下标直接命中 table 中的唯一节点直接返回；
2. 如果冲突，则通过 key.equals(k) 去查找对应的 entry；若为树，则在树中查找，O(logn)，若为链表，则在链表中查找，O(n)。

代码实现：

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 直接命中
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 未命中
        if ((e = first.next) != null) {
            // 在树中get
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 在链表中get
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```



### 4. hash 的原理

在 get 和 put 的过程中，计算下标时，先对 hashCode 进行 hash 操作，然后再通过 hash 值进一步计算 index。

![293b52fc-d932-11e4-854d-cb47be67949a.png](https://i.loli.net/2019/12/27/inyNCZcsFkWBQz6.png)

对 HashCode 计算 Hash 值的具体实现如下，这步称为**扰动处理**：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

HashCode 高 16bit 不变,低 16bit 和高 16bit 做了一个异或。作者认为在计算 index 时用的 & 运算，高位很容易无法参与，导致容易发生碰撞。于是设计者想了一个顾全大局的方法(综合考虑了速度、作用、质量)，就是把高16bit和低16bit异或了一下。

对 hash 计算 index 的具体实现如下：

```java
(n - 1) & hash
```

使用 & 运算而不是 % 是因为 & 运算更为高效。

为什么跟 length -  1 进行 & 运算？
因为 length 为 2 的幂次方，即一定是偶数，偶数减1，即是奇数，这样保证了（length-1）在二进制中最低位是1，而 & 运算结果的最低位是1还是0完全取决于 hash 值二进制的最低位。如果 length 为奇数，则 length-1 则为偶数，则 length-1 二进制的最低位横为 0，则 & 位运算的结果最低位横为 0，即横为偶数。这样 table 数组就只可能在偶数下标的位置存储了数据，浪费了所有奇数下标的位置，这样也更容易产生 hash 冲突。这也是 HashMap的容量为什么总是 2 的平方数的原因。

### 5. resize 的扩容机制

当 put 时，如果发现目前的 bucket 占用程度已经超过了 Load Factor 所希望的比例，那么就会发生 resize。在resize 的过程，简单的说就是把 bucket 扩充为2倍，之后重新计算 index ，把节点再放到新的 bucket 中。

当超过限制的时候会 resize，然而又因为我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。

怎么理解呢？例如我们从16扩展为32时，具体的变化如下所示：

![ceb6e6ac-d93b-11e4-98e7-c5a5a07da8c4.png](https://i.loli.net/2019/12/27/EyOb4Jn1ZPsU78e.png)

因此元素在重新计算 hash 之后，因为 n 变为 2 倍，那么 n-1 的 mask 范围在高位多 1bit (红色)，因此新的 index就会发生这样的变化：

![519be432-d93c-11e4-85bb-dff0a03af9d3.png](https://i.loli.net/2019/12/27/sYwVcpobiyTtBx8.png)

因此，我们在扩充 HashMap 的时候，不需要重新计算 hash，只需要看看原来的 hash 值新增的那个 bit 是 1 还是 0 就好了，是 0 的话索引没变，是 1 的话索引变成“原索引 + oldCap ”。这样既省去了重新计算 hash 值的时间，而且同时，由于新增的 1bit 是 0 还是 1 可以认为是随机的，因此 resize 的过程，均匀的把之前的冲突的节点分散到新的 bucket 了。

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 超过最大值就不再扩充了，就只好随你碰撞去吧
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 没超过最大值，就扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 计算新的resize上限
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 把每个bucket都移动到新的buckets中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 原索引放到bucket里
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 原索引+oldCap放到bucket里
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

### 6. HashMap JDK 1.7 VS 1.8 

- 1.8 引入了红黑树，用来解决链表过长时的搜索效率；
- 1.7 与 1.8 的 扰动处理方式不同，1.7 四次位运算和5次异或，1.8 一次位运算和一次异或；
- 1.7 在 put 时，采用头插法，1.8 使用尾插法，1.8不会出现链表 **并发下闭环**的情况；
- resize 时， 1.7 需要重新计算 index ,而 1.8 的 index = 原位置 or 原位置 + 旧容量；

### 7. HashMap VS HashTable

- HashTable 线程安全，因为在方法上使用了 `synchronized`,性能较低；HashMap 非线程安全；
- HashTable 不允许 `null` 作为 key 和 value ，而 HashMap 允许；
- HashMap默认初始化大小为16，HashTable为11；HashMap 扩容时是`扩大两倍`，而 HashTable 扩容是`两倍加一`；
- HashTable继承自`Dictionary`类，HashMap 继承 `AbstractMap`类。

### 8. HashMap VS ConcurrentHashMap

后续文章详解 ......