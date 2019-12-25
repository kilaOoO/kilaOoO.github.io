---
title: List原理剖析
date: 2019-12-25 15:36:00
tags:
categories: java
---

## ArrayList

### 1. 概述

ArrayList 底层数据结构是基于 Object 对象数组的，是一个可自动扩容的动态数组。具有数组存取的天然优势，但是对于也具备数组的劣势。

### 2. ArrayList 构造函数

ArryList 提供三种构造函数。

```java
// 1. 指定初始容量
public ArrayList(int initialCapacity)
// 2. 空容量（add时初始化默认为10）
public ArrayList()
// 3. 传入集合参数
public ArrayList(Collection<? extends E> c)

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
public ArrayList() {
    // 初始化时指向一个空的 Object[] 共享类变量
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

### 3. add

当向 ArrayList 添加元素时，先检查空间是否足够，够的话直接添加，不够的话根据原始容量增加一半。

```java
public boolean add(E e) {
    // ensureCapacityInternal 是自动扩容机制的核心函数
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

```java
// 默认初始化容量为 10
private static final int DEFAULT_CAPACITY = 10;
private void ensureCapacityInternal(int minCapacity) {
    // 当不指定初始化大小时，添加第一个元素数组会自动扩容为 10
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    // 判断容量是否足够
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

// grow 方法将数组扩容为原数组的1.5倍，调用 Arrays.copyOf 方法
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

### 3. set 和 get

Array的put和get函数就比较简单了，先做index检查，然后执行赋值或访问操作。

```java
public E set(int index, E element) {
    rangeCheck(index);
    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}
```

### 4. remove

```java
public E remove(int index) {
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
        // 把后面的往前移
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    // 把最后的置null
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}
```

## Vector

Vector 与 ArrayList基本类似，就不详述，唯一不同的是 Vector 大部分方法都加了 synchronized 关键字，是线程安全的。

```java
//Vector和ArrayList一样，底层基于该数组实现
protected Object[] elementData;
//这个相当于ArrayList里面的size
protected int elementCount;
//当Vector的大小大于其容量时，Vector的容量自动增加的量

/*****这里不同于 ArrayList 直接扩容1.5倍，而是指定扩张容量*****/
protected int capacityIncrement;

//带容量和容量自动增加量的参数的构造函数
public Vector(int initialCapacity, int capacityIncrement) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
    //初始化数组
    this.elementData = new Object[initialCapacity];
    this.capacityIncrement = capacityIncrement;
}

//给定容量的构造函数
public Vector(int initialCapacity) {
    this(initialCapacity, 0);
}

//无参构造函数
/*****这里不同于 ArrayList 构造函数中就初始化容量了，而不是第一次 add*****/
public Vector() {
    //初始化数组，大小为10
    this(10);
}
```

## LinkedList

LinkedList 底层用双向链表实现，具有链表存取的优点和缺点。

### 1. node 结构

```java
private static class Node<E> {
        //本身
        E item;
        //下一个元素
        Node<E> next;
        //前一个元素
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

### 2. add()

```java
//添加元素
public boolean add(E e) {
    //把该元素放到链表最后面
    linkLast(e);
    return true;
}


// 第一个元素的引用
transient Node<E> first;

// 最后一个元素的引用
transient Node<E> last;

//在链表尾部创建链接
void linkLast(E e) {
    //获取最后一个元素
    final Node<E> l = last;
    //创建一个一个Node对象，参数（前，本，后）
    //前：指向链表最后一个元素，即新加入的元素的上一个元素
    //本：指的就是新加入元素本身
    //后：因为新加入的元素本身就是在链表最后面加入，所以，后面没有元素，则为null
    final Node<E> newNode = new Node<>(l, e, null);
    //把last引用指向新创建的对象上面
    last = newNode;
    //如果在链表为空的情况下，first=last=null
    if (l == null)
        //那么第一个就是最新创建的元素
        first = newNode;
    else
        //把链表最后元素的next指向创建的新元素的引用
        l.next = newNode;
    //链表大小+1
    size++;
    modCount++;
}
```

### 3. remove()

```java
//根据索引移除元素
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}


//删除链表给定的元素链接
E unlink(Node<E> x) {
    //该元素本身
    final E element = x.item;
    //该元素下一个元素
    final Node<E> next = x.next;
    //该元素上一个元素
    final Node<E> prev = x.prev;

    //如果该元素本身就是第一个元素，即链表头部
    if (prev == null) {
        //那么就把first指向下一个元素引用
        first = next;
    } else {
        //把前一个元素的next指向该元素的下一个元素，即跳过该元素
        //因为该元素马上要被删除掉了
        prev.next = next;
        //把该元素前一个元素引用置空
        x.prev = null;
    }

    //如果该元素本身就是最后一个元素，即链表尾部
    if (next == null) {
        //那么把last指向前一个元素引用
        last = prev;
    } else {
        //把下一个元素的prev指向该元素的上一个元素，也是跳过该元素(即将被删)
        next.prev = prev;
        //把该元素下一个元素引用置空
        x.next = null;
    }

    //把该元素本身置空
    x.item = null;
    //链表大小-1
    size--;
    modCount++;
    //返回该元素
    return element;
}
```

### 4. get 和 set

```java
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

```java
// get 和 set 都使用了此优化的搜索方法
Node<E> node(int index) {
    // assert isElementIndex(index);
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

**判断index是在前半区间还是后半区间，如果在前半区间就从head搜索，而在后半区间就从tail搜索。而不是一直从头到尾的搜索。如此设计，将节点访问的复杂度由O(n)变为O(n/2)。**