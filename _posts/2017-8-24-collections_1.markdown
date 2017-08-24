---
layout:     post
title:      "Java Collections Framework源码解读(1)——ArrayList"
subtitle:   ""
date:       2017-08-24 18:00:00
author:     "Crow"
#header-img: "img/post-bg-nextgen-web-pwa.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Java
---

ArrayList是List接口**可调整数组大小**的实现。实现所有可选列表操作，并允许放入包括空值在内的所有元素。每个ArrayList都有一个容量（capacity,区别于size），表示**底层数组**的实际大小，容器内存储元素的个数不能多于当前容量。

# 底层实现

**java.util.ArrayList**类的继承关系如下：
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

其中需要注意的是**RandomAccess**接口，这是一个标记接口，没有定义任何具体的内容。在该接口的注释中有这样一段话：
```java
/**
for (int i=0, n=list.size(); i < n; i++) {     
    list.get(i);
}
runs faster than this loop:
for (Iterator i=list.iterator(); i.hasNext(); ) { 
   i.next();
}
**/
```
这说明在数据量很大的情况下，采用迭代器遍历实现了该接口的集合，速度比较慢。

实现了RandomAccess接口的集合有：**ArrayList**, AttributeList, CopyOnWriteArrayList, RoleList, RoleUnresolvedList, **Stack**, **Vector**等。

ArrayList一些重要的字段如下：
```
    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = {};
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    transient Object[] elementData; // non-private to simplify nested class access
    private int size;//底层数组中实际元素个数，区别于capacity
```
可以看到，默认第一次插入元素时创建数组的大小为10。当向容器中添加元素时，如果容量不足，容器会**自动增加50%的容量**。增加容量的函数`grow()`源码如下：
```java
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);//右移一位代表增加50%
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

 private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

**值得注意的是**，由于集合框架用到了编译器提供的语法糖——泛型，而Java泛型的内在实现是通过类型擦除和类型强制转换来进行的，其实存储的数据类型都是Raw Type，因此集合框架的底层数组都是**Object**数组，可以容纳任何对象。

# 数组复制

ArrayList的实现中大量地调用了`Arrays.copyof()`和`System.arraycopy()`方法。在此介绍一下这两个方法。

`System.arraycopy()`方法是一个native方法，调用了系统的C/C++代码，在openJDK中可以看到其源码。该方法最终调用了C语言的`memmove()`函数，因此它可以保证同一个数组内元素的正确复制和移动，比一般的复制方法的实现效率要高很多，很适合用来批量处理数组。Java强烈推荐在**复制大量数组元素**时使用该方法，以取得更高的效率。

`Arrays.copyOf()`方法有很多重载版本，但实现思路都是一样的，其泛型版本源码如下：
```java
public static <T> T[] copyOf(T[] original, int newLength) {  
    return (T[]) copyOf(original, newLength, original.getClass());  
}  
```
其调用了`copyof()`方法：
```java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {  
    T[] copy = ((Object)newType == (Object)Object[].class)  
        ? (T[]) new Object[newLength]  
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);  
    System.arraycopy(original, 0, copy, 0,  
                     Math.min(original.length, newLength));  
    return copy;  
}  
```
该方法实际上是在其内部创建了一个类型为newType、长度为newLength的新数组，调用`System.arraycopy()`方法，将原来数组中的元素复制到新的数组中。


# 非线程安全

ArrayList的实现是**不同步**的，如果多个线程同时访问ArrayList实例，并且至少有一个线程修改list的结构，那么它就必须在外部进行同步。如果没有这些对象， 这个list应该用`Collections.synchronizedList()`方法进行包装。
最好在list的创建时就完成包装，防止意外地非同步地访问list:
```java
List list = Collections.synchronizedList(new ArrayList(...));
```
除了未实现同步之外，ArrayList大致相当于Vector。

`size()`,` isEmpty()`,` get()`,`set()`方法均能在**常数时间**内完成，`add()`方法的时间开销跟插入位置有关(**adding n elements requires O(n) time**)，`addAll()`方法的时间开销**跟添加元素的个数成正比**。其余方法大都是线性时间。

# 常用API

ArrayList常用的`size()`,` isEmpty()`,` get()`,`set()`方法实现都比较简单，读者可自行翻阅源码，它们均能在**常数时间**内完成，性能很高，这也是数组实现的优势所在。`add()`方法的时间开销跟插入位置有关(**adding n elements requires O(n) time**)，`addAll()`方法的时间开销**跟添加元素的个数成正比**。其余方法大都是线性时间。

#### add()方法

```java
public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
}
```
```
public void add(int index, E element) {
        rangeCheckForAdd(index);
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
}
```
前者是在ArrayList尾部插入一个元素，后者是在指定位置插入元素。值得注意的是，将元素的索引赋给elementData[size]时可能会出现数组越界，这里的关键就在于`ensureCapacity(size+1)`的调用，这个方法的作用是确保数组的容量，它的源码如下：

**ensureCapacity()**和**ensureExplicitCapacity()**方法：
```java
public void ensureCapacity(int minCapacity) {
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
            // any size if not default element table
            ? 0
            // larger than default for default empty table. It's already
            // supposed to be at default size.
            : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
}

private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
}
```
其中有一个重要的实例变量**modCount**，它是在**AbstractList**类中定义的，在使用**迭代器遍历**的时候，用modCount来**检查列表中的元素是否发生结构性变化（列表元素数量发生改变）**了，如果modCount值改变，则代表列表中元素发生了结构性变化。

前面说过，ArrayList是**非线程安全**的，modCount主要**在多线程环境下进行安全检查**，防止一个线程正在迭代遍历，另一个线程修改了这个列表的结构。如果在使用迭代器进行遍历ArrayList的时候modCount值改变，则会报**ConcurrentModificationException**异常。

可以看出，直接在数组后面插入一个元素`add(e)`效率也很高，但是如果要按下标来插入元素，则需要调用`System.arraycopy()`方法来移动部分受影响的元素，这会导致性能低下，这也是使用数组实现的**ArrayList的劣势**。

同理，`remove()`方法也会改变modCount的值，效率与按下标插入元素相似，在此不加赘述。

#### addAll()方法

```java
public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
}
```
```java 
public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);

        Object[] a = c.toArray();
        int numNew = a.length;
        ensureCapacityInternal(size + numNew);  // Increments modCount

        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);

        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
}
```
`addAll`方法也分在末尾插入和在指定位置插入，先将入参中的集合c转换成数组，根据转换后数组的程度和ArrayList的size拓展容量，之后调用`System.arraycopy()`方法复制元素到相应位置，调整size。根据返回的内容分析，只要集合c的大小不为空，即转换后的数组长度不为0则返回true。

容易看出，`addAll()`方法的时间开销是**跟添加元素的个数成正比**的。

#### trimToSize()方法

下面来看一个简单但是很有用的方法`trimToSize()`。
```java
public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
}
```
由于`elementData`的长度会被拓展，size标记的是其中包含的元素的个数。所以会出现`size`很小但`elementData.length`很大的情况，将出现空间的浪费。`trimToSize()`将返回一个新的数组给elementData，元素内容保持不变，length和size相同，节省空间。

在实际应用中，考虑这样一种情形，当某个应用需要，一个ArrayList扩容到比如size=10000，之后经过一系列remove操作size=15，在后面的很长一段时间内这个ArrayList的size一直保持在<100以内，那么就造成了很大的空间浪费，这时候建议显式调用一下`trimToSize()`方法，以优化一下内存空间。  
 
或者在一个ArrayList中的容量已经固定，但是由于之前每次扩容都扩充50%，所以有一定的空间浪费，可以调用`trimToSize()`消除这些空间上的浪费。
