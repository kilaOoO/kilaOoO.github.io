---
title: ConcurrentHashMap原理剖析
date: 2019-12-31 10:37:38
tags:
categories: J.U.C
description: 基于JDK1.7、1.8 详细分析了 ConcurrentHashMap
---

ConcurrentHashMap 在 1.7 和 1.8 实现有所不同，分两个版本对比。

## Base 1.7

![__I7__CMDFI4_YIFQJ66YFI.png](https://i.loli.net/2019/12/31/WniKElrMJU2a9fX.png)

ConcurrentHashMap 由 Segment 数组、HashEntry 组成。HashEntry 和 HashMap 一样 数组 + 链表。其中HashEntry 中的 value  用 volatile 修饰，保证可见性。Segment 数组的意义就是将一个大的 table 分割成多个小的table来进行加锁。

### 1. put 实现

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key);
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}
```

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        HashEntry<K,V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else {
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

put 的整个流程：

1. 通过 key 定位到 Segment，如果该Segment还没有初始化，即通过CAS操作进行赋值，之后在对应的 Segment 中进行具体的 put；
2. 尝试自旋获取当前 Segment 的锁；
3. 将当前 Segment 中的 table 通过 key 的 hashcode 定位到 HashEntry；
4. 进行插入操作；
5. 释放锁。

### 2. get 实现

ConcurrentHashMap 的 get 操作跟 HashMap 类似，只是 ConcurrentHashMap 第一次需要经过一次 hash 定位到 Segment 的位置，然后再 hash 定位到指定的 HashEntry，遍历该 HashEntry 下的链表进行对比，成功就返回，不成功就返回 null。

## base 1.8

JDK1.8 的实现已经摒弃了 Segment，而是直接用 Node数组 + 链表 + 红黑树的数据结构来实现，并发控制使用Synchronized 和 CAS 来操作。

![8BYCT8`_0LKH40_M4U2A8SI.png](https://i.loli.net/2019/12/31/RcULndPfHXh95QJ.png)

### 1. put 实现

```java
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
```

put 的整个流程：

1. 如果没有初始化就先调用 initTable（）方法来进行初始化过程；
2. 如果没有hash冲突就直接 CAS 插入；
3. 如果还在进行扩容操作就先进行扩容；
4. 如果存在 hash 冲突，synchronized 锁写入数据，这里有两种情况，一种是链表形式就直接遍历到尾端插入，一种是红黑树就按照红黑树结构插入；
5. 如果数量大于 `TREEIFY_THRESHOLD` 则要转换为红黑树。

### 2. get 实现

1. 根据计算出来的 hashcode 寻址，如果就在桶上那么直接返回值；
2. 如果是红黑树那就按照树的方式获取值；
3. 就不满足那就按照链表的方式遍历获取值。

## 总结

JDK1.8 版本的 ConcurrentHashMap 的数据结构已经接近 HashMap ，相对而言，ConcurrentHashMap 只是增加了同步的操作来控制并发，从 JDK1.7 版本的 ReentrantLock+Segment+HashEntry，到 JDK1.8 版本中synchronized+CAS+HashEntry+红黑树。

1. JDK1.8 的实现降低锁的粒度，JDK1.7版本锁的粒度是基于 Segment 的，包含多个 HashEntry，而 JDK1.8 锁的粒度就是 HashEntry（首节点）；
2. JDK1.8 版本的数据结构变得更加简单，使得操作也更加清晰流畅，因为已经使用 synchronized 来进行同步，所以不需要分段锁的概念，也就不需要Segment这种数据结构了，由于粒度的降低，实现的复杂度也增加了；
3. JDK1.8 使用红黑树来优化链表，基于长度很长的链表的遍历是一个很漫长的过程，而红黑树的遍历效率是很快的。