# DelayQueue源码解析

## 目录
- [1. 什么是DelayQueue？](#1-什么是-delayqueue)
- [2. 类继承结构](#2-类继承结构)
- [3. 构造函数解析](#3-构造函数解析)
- [4. 核心属性与内部结构](#4-核心属性与内部结构)
- [5. 核心方法源码分析](#5-核心方法源码分析)
    - [5.1 offer方法](#51-offer方法)
    - [5.2 poll方法](#52-poll方法)
    - [5.3 take方法](#53-take方法)
- [6. 总结与思考](#6-总结与思考)

---

## 1. 什么是 DelayQueue？
`DelayQueue` 是Java并发包 `java.util.concurrent` 提供的一个**无界阻塞队列**，元素必须实现 `Delayed` 接口。它的特点是：

- 元素带有“延迟时间”，只有延迟到期后，元素才能被获取。
- 基于最小堆(优先队列)实现，内部使用 `PriorityQueue` 来排序。
- 线程安全，适合调度、任务过期处理等场景。

> 实现类位于 `java.util.concurrent.DelayQueue`。

## 2. 类继承结构
```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E>
    implements BlockingQueue<E> 
```
类的继承结构图如下:

![img_3.png](..%2Fimg%2Fimg_3.png)

**继承和实现说明：**

- 继承了 AbstractQueue：继承了通用队列行为

- 实现了 BlockingQueue 接口：支持阻塞特性，如 take()

- 泛型限定为 Delayed：传入的元素必须实现 Delayed 接口，才能参与比较和延迟判断

## 3. 构造函数解析
```java
public DelayQueue() {
}

public DelayQueue(Collection<? extends E> c) {
    this.addAll(c);
}
```
**说明：**

- 默认构造函数创建一个空队列，使用 PriorityQueue 作为底层容器。

- 第二个构造函数可以通过已有集合初始化队列，但集合中的元素必须是 Delayed 类型。

- PriorityQueue 会根据 getDelay 值自动排序。

## 4. 核心属性与内部结构
```java
private final transient ReentrantLock lock = new ReentrantLock();
private final PriorityQueue<E> q = new PriorityQueue<>();
private final Condition available = lock.newCondition();
```
**关键属性说明：**

| 属性名  |                作用                 |  
|:-----|:---------------------------------:|
| lock	 | 线程互斥锁，保障线程安全 | 
| q |          实际存储元素的优先队列	           |
| available |           条件变量，用于线程等待、唤醒机制	           |

此外，还有一个特殊的属性：
```java
private Thread leader;
```
leader 机制是关键性能优化点，表示当前正在等待最近到期元素的线程，避免所有线程都使用 timed wait，提升效率。

## 5. 核心方法源码分析
### 5.1 offer方法
&nbsp;&nbsp;该方法的签名如下:
```java
public boolean offer(E e) {
      final ReentrantLock lock = this.lock;
      lock.lock();
      try {
          q.offer(e);
          if (q.peek() == e) {
              leader = null;
              available.signal();
          }
          return true;
      } finally {
          lock.unlock();
      }
  }
```
首先拿到一个ReentrantLock的锁对象，DelayQueue是线程安全的,用ReentrantLock来控制并发访问,调用PriorityQueue对象的offer方法添加元素，PriorityQueue上文我已经分析
过了它的数据结构默认是最小堆，按照实现了Comparable接口逻辑实现来排序比较大小，DelayQueue实际上是按照getDelay()排序，越早过期越靠前，判断新加入的元素是否是新的队首元素(最早过期的）。
peek()返回的是当前队列中最早到期的元素。如果刚插入的元素是最早的，那么需要通知等待线程，因为有可能原来没有可取的元素，现在有了。
return true代表总是返回成功，不会因为容量限制失败因为DelayQueue是没有容量限制的,最终释放锁。

### 5.2 poll方法
&nbsp;&nbsp;该方法的签名如下:
```java
  public E poll() {
      final ReentrantLock lock = this.lock;
      lock.lock();
      try {
          E first = q.peek();
          return (first == null || first.getDelay(NANOSECONDS) > 0)
              ? null
              : q.poll();
      } finally {
          lock.unlock();
      }
  }
```
首先还是获取到ReentrantLock锁，进行加锁，调用PriorityQueue对象的peek方法获取第一个元素，判断第一个元素是否为空或者还没到过期时间，如果成立返回null,否则返回第一个到期时间的元素，最后在
finally中进行解锁操作。

### 5.3 take方法
&nbsp;&nbsp;该方法的签名如下:
```java
public E take() throws InterruptedException {
    
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            E first = q.peek();
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0L)
                    return q.poll();
                first = null;
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock();
    }
}
```
首先还是获取ReentrantLock锁，可响应中断操作，意思是可以不需要线程一直阻塞获取元素，死循环for为了获取首个元素，里面的逻辑是调用PriorityQueue的peek方法查看队首元素，
但是不移除，如果first为null,调用available.await方法等待其他线程插入元素（比如通过 offer()），避免忙等。如果不为空说明有元素获取它的过期时间，如果delay小于等于0
说明是过期的了，调用PriorityQueue的poll方法取出，如果没到过期时间把first=null,判断leader如果不是null,当前已有线程在等待下一个元素过期(领导者线程)，那就乖乖排队，等待别人唤醒。
如果leader=null,说明没有领导者线程，那当前线程自己做领导，精确等待delay纳秒。等待时间到了后会被唤醒，重新进入循环判断是否真正到期。在for里面判断leader == thisThread
如果是退出领导位置，为后续线程腾位。在最外层的finally函数里面判断如果当前没有领导者，且队列中还有元素，则唤醒一个等待线程，让它成为新的领导者继续判断。

## 6. 总结与思考
- DelayQueue是Java并发容器中极具代表性的阻塞队列，适用于定时调度类业务

- 结合 PriorityQueue + Condition + leader 线程，实现高效的等待与唤醒机制

- 适合中小规模的延迟任务调度；大规模或分布式场景建议使用更专业组件如 Quartz、xxl-job、RocketMQ 延迟队列
