# PriorityQueue源码分析

> 本文主要解析 Java 标准库中 `java.util.PriorityQueue` 的源码设计，重点介绍它的类继承结构、核心字段和构造函数实现，帮助读者理解其底层原理及设计思路,本文是基于JDK21。

## 目录
- [一、PriorityQueue类简介](#一priorityqueue类简介)
- [二、大小堆的原理基础](#二大小堆的原理基础)
- [三、PriorityQueue类继承结构](#三priorityqueue类继承结构)
- [四、PriorityQueue核心字段](#四priorityqueue核心字段)
- [五、构造函数](#五priorityqueue构造函数)
- [六、源码分析](#六源码分析)
    - [6.1 add方法](#61-add方法)
    - [6.2 poll方法](#62-poll方法)
- [七、总结](#七总结)

## 一、PriorityQueue类简介

`PriorityQueue` 是Java 集合框架中的一个基于优先堆（Heap）实现的无界优先级队列，默认是最小堆，元素的自然顺序或者自定义比较器决定元素优先级。它适合需要快速获取最小/最大元素的场景。

## 二、大小堆的原理基础
### 小顶堆（Min-Heap）

- 特点：**根节点是最小值**。
- 每个节点的值都不大于其子节点。
- 适用于：快速获取最小值，如 Java 的 `PriorityQueue` 默认实现。

### 大顶堆（Max-Heap）

- 特点：**根节点是最大值**。
- 每个节点的值都不小于其子节点。

### Java 中如何实现大顶堆？

通过自定义 `Comparator` 实现反转比较逻辑：

```java
PriorityQueue<Integer> maxHeap = new PriorityQueue<>((a, b) -> b - a);
```

## 三、PriorityQueue类继承结构
```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable 
```
类的继承结构图如下所示:

![img_2.png](..%2Fimg%2Fimg_2.png)

## 四、PriorityQueue核心字段

```java
//底层数组存储堆元素，transient表示不序列化
private transient Object[] queue;
//当前元素个数
private int size = 0;
// 用于自定义元素比较规则
private final Comparator<? super E> comparator;
// 默认容量大小
private static final int DEFAULT_INITIAL_CAPACITY = 11;
```
## 五、PriorityQueue构造函数
**1.默认无参构造函数**
```java
 public PriorityQueue() {
  this(DEFAULT_INITIAL_CAPACITY, null);
}
```
- 使用默认容量 11。

- 使用元素的自然顺序进行排序。

**2. 指定初始容量的构造函数**
```java
public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}
```
- 可以指定队列初始容量大小。

- 使用自然顺序比较。

**3. 指定比较器的构造函数**
```java
 public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
}
```
- 使用默认容量。

- 传入自定义的比较器。

**4.指定容量和比较器的构造函数**
```java
public PriorityQueue(int initialCapacity,Comparator<? super E> comparator) {

    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
```
- 允许同时指定容量和比较器。

- 内部初始化容量和比较器字段。

- 抛出异常保护非法容量。

**5.使用集合构造**
```java
public PriorityQueue(Collection<? extends E> c) {
    if (c instanceof SortedSet<?>) {
      SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
      this.comparator = (Comparator<? super E>) ss.comparator();
      initElementsFromCollection(ss);
    }
    else if (c instanceof PriorityQueue<?>) {
      PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
      this.comparator = (Comparator<? super E>) pq.comparator();
      initFromPriorityQueue(pq);
    }
    else {
      this.comparator = null;
      initFromCollection(c);
    }
}
```
## 六、源码分析
### 6.1 add方法
&nbsp;&nbsp;add方法主要调用了offer方法，该方法的签名如下:
```java
public boolean offer(E e) {
    if (e == null)
     throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
     grow(i + 1);
    siftUp(i, e);
    size = i + 1;
    return true;
}
```
PriorityQueue 不允许插入null元素,如果插入报错空指针异常,modCount自增加1,modCount是继承自AbstractList的一个字段，用于fail-fast机制(如Iterator检测结构变化),
把当前size值赋值给变量i,判断i值是否大于数组长度，如果大于进行扩容操作,底层数组扩容逻辑类似于ArrayList是1.5倍扩容,接着调用了siftUp方法，此方法是上浮方法，如果堆中已有元素，则调用siftUp将新元素向上“浮动”到合适的位置，以维持堆序,
该方法的签名如下:
```java
private void siftUp(int k, E x) {
      if (comparator != null)
          siftUpUsingComparator(k, x, queue, comparator);
      else
          siftUpComparable(k, x, queue);
}
```
此方法取绝于是否传递了comparator比较器，默认是按照自然顺序上浮，调用的是siftUpComparable方法，该方法的签名如下:
```java
private static <T> void siftUpComparable(int k, T x, Object[] es) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = es[parent];
        if (key.compareTo((T) e) >= 0)
            break;
        es[k] = e;
        k = parent;
    }
    es[k] = key;
}
```
k是要插入数据元素的下标，x是要插入的元素，es是数组，把x强制转换为Comparable,由此可以看出要插入的元素必须实现Comparable接口否则会报ClassCastException错误，
第一次插入的时候k=0，那么会直接执行es[k=0]=key，这个比较好理解，关键看while循环中获取parent的逻辑是什么样的，我们知道PriorityQueue是基于堆的设计，它是用数组实现的
完全二叉树，有以下公式，对于任意一个节点在数组中的索引k(k>0),它的左子节点的索引是：2 * k + 1,它的右子节点的索引是：2 * k + 2,它的父节点的索引是：(k - 1) / 2，所以父节点的标准
写法是(k - 1) / 2，但是为了性能可以写成无符号右移(k - 1) >>> 1获取父节点parent。

获取parent下标的元素e,比较传入的元素和e如果大于e则不交互位置，满足小顶堆规则，如果传入的元素小于e则进行交换，把当前下标存储的元素换成父节点的，把父亲节点下标赋值给k,k处节点下标换成当前插入的元素e。

### 6.2 poll方法
&nbsp;&nbsp;该方法的签名如下:
```java
public E poll() {
    final Object[] es;
    final E result;
    if ((result = (E) ((es = queue)[0])) != null) {
        modCount++;
        final int n;
        final E x = (E) es[(n = --size)];
        es[n] = null;
        if (n > 0) {
            final Comparator<? super E> cmp;
            if ((cmp = comparator) == null)
                siftDownComparable(0, x, es, n);
            else
                siftDownUsingComparator(0, x, es, n, cmp);
        }
    }
    return result;
}
```
获取堆顶元素(index 0 位置),同时将queue数组赋值给es,如果堆为空（堆顶为 null），直接返回null,将最后一个元素 es[size - 1] 拿出来，准备替换到堆顶,同时减少size,清除原来的最后一个元素位置，防止内存泄漏,
如果还有元素，调用siftDownComparable() 让最后一个元素下沉，维持最小堆结构,默认调用的是siftDownComparable方法，该方法签名如下:
```java
private static <T> void siftDownComparable(int k, T x, Object[] es, int n) {
    Comparable<? super T> key = (Comparable<? super T>)x;
    int half = n >>> 1;         
    while (k < half) {
        int child = (k << 1) + 1;
        Object c = es[child];
        int right = child + 1;
        if (right < n &&
            ((Comparable<? super T>) c).compareTo((T) es[right]) > 0)
            c = es[child = right];
        if (key.compareTo((T) c) <= 0)
            break;
        es[k] = c;
        k = child;
    }
    es[k] = key;
}
```
k: 当前下沉元素的起始位置(一般是0,即堆顶);x: 要下沉的元素(通常是堆中最后一个元素);es: 堆数组(通常是queue);n: 当前堆的大小(数组中的有效元素数),将传入的元素x转换为Comparable 类型，用于比较优先级。int half = n >>> 1,处理只有
子节点的情况,如果 k >= half，表示是叶子节点了，不需要再往下沉了,获取左子节点的索引下标child,然后获取对应的元素值，获取右子节点的下标值,判断如果有右子节点且右子节点更小，则用右子节点替代当前最小值，比较key和c的值如果key小于c则交换结束，
说明满足最小堆性质，停止下沉，否则把把较小的子节点上移,把child值赋值给k,继续向下查找。

## 七、总结

- Java的PriorityQueue是基于小顶堆实现的优先队列，默认使用元素的自然顺序。

- 支持自定义比较器从而构建大顶堆或其他逻辑顺序。

- 构造函数设计灵活，支持多种初始化方式。

- 源码整体结构简洁高效，适合在各种需要优先队列的场景中使用。