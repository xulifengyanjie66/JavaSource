# ArrayBlockingQueue源码分析

&nbsp;&nbsp;本文将从`ArrayBlockingQueue`的类继承结构、构造方法以及关键源码三个方面进行深入解析，帮助大家全面理解其底层实现原理,本文基于JDK21。

## 目录
- [一、类继承结构](#一类继承结构)
- [二、构造方法](#二构造方法)
- [三、源码分析](#三源码分析)
    - [3.1 成员变量](#31-成员变量)
    - [3.2 入队操作 put/offer](#32-入队操作-putoffer)
    - [3.3 出队操作 take/poll](#33-出队操作-takepoll)
    - [3.4 锁机制](#34-锁机制)
    - [3.5 其他操作](#35-其他操作)
- [四、总结](#四总结)

## 一、类继承结构
```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable
```
类的继承结构图如下所示:

![img_1.png](img_1.png)

## 二、构造方法
```java
public ArrayBlockingQueue(int capacity)
public ArrayBlockingQueue(int capacity, boolean fair)
public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c)
```
- capacity：必须指定容量，不能为负。
- fair：是否使用公平锁，默认不公平（false）。
- 第三个构造器允许初始化时传入集合并填充队列。

## 三、源码分析
### 3.1 成员变量
```java
  final Object[] items;
  final ReentrantLock lock;
  private final Condition notEmpty;
  private final Condition notFull;
  int count;
  int takeIndex;
  int putIndex;
```
- items：实际存储元素的数组。
- lock：可重入锁，控制并发访问。
- notEmpty/notFull：对应两个条件队列，用于线程等待和唤醒。
- count：当前元素数量。
- putIndex/takeIndex：数组中的入队和出队索引。

### 3.2 入队操作 add/offer
&nbsp;&nbsp;add方法主要调用的是offer方法，我们主要看offer方法的签名：
```java
public boolean offer(E e) {
    Objects.requireNonNull(e);
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)
            return false;
        else {
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```
添加的元素不能是null,如果是null会抛出空指针异常,用ReentrantLock进行加锁操作，判断元素的数量是否等于数组的长度,如果等于返回false,否则就调用enqueue方法，该方法签名如下:
```java
private void enqueue(E e) {
    final Object[] items = this.items;
    items[putIndex] = e;
    if (++putIndex == items.length) putIndex = 0;
    count++;
    notEmpty.signal();
}
```
将元素e放入putIndex位置,默认是0,putIndex自增以后判断是否等于数组长度，如果是把putIndex重置为0，这样就形成了一个环状数组，如果有0的位置元素被取走了，新的元素
被重新添加到0的位置，这样避免元素来回移动提高性能，count自增1,notEmpty.signal() 是为了唤醒正在等待队列有元素的线程，通常用于通知take() 操作可以继续了。