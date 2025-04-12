# ConcurrentHashMap源码分析与实现原理

## 📖 目录

- [一、引言](#一-引言)
- [二、类继承结构](#二类继承结构)
- [三、数据结构](#三数据结构)
  - [3.1 核心成员属性](#31-核心成员属性)
- [四、源码分析](#四源码分析)
  - [4.1 put源码操作源码](#41-put操作核心源码分析)
  - [4.2 get源码操作源码](#42-get操作核心源码分析)
- [六、get 操作源码分析](#六get-操作源码分析)
- [七、扩容机制](#七扩容机制)
- [八、与 HashMap 的区别](#八与-hashmap-的区别)
- [九、常见问题与面试点](#九常见问题与面试点)
- [十、总结](#十总结)

---
## 一、 引言
在多线程开发中，ConcurrentHashMap是我们绕不过的高性能并发容器之一。它在JDK1.8之后全面优化，实现了更优雅的无锁读、分段写、红黑树优化等机制，是Java并发体系中非常核心的一个类。

不过，要真正读懂它的底层源码，建议同志们提前掌握以下几个知识点：

| 技术点                   | 简介|
|-----------------------|------------------|
| CAS(Compare And Swap) | 无锁并发的基础操作，底层通过 Unsafe 实现原子更新|
| volatile 关键字          | 保证变量的可见性和禁止指令重排序|
| synchronized + 锁膨胀机制  | JDK对锁的优化策略，如偏向锁、轻量级锁、重量级锁 |
| 内存模型(JMM)             | 了解happens-before原则，帮助理解并发变量行为 |
| 红黑树结构            | JDK1.8中引入红黑树作为链表优化结构，提升查询性能|
| JavaUnsafe类           | 低层操作的“黑科技”，支持直接内存访问和原子更新|

本文针对JDK21源码进行说明。
## 二、类继承结构

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
        implements ConcurrentMap<K,V>, Serializable
```
![img26.jpg](..%2Fimg%2Fimg26.jpg)


## 三、数据结构

### 3.1 核心成员属性
```java
// 1. 桶数组，存放链表或红黑树节点
transient volatile Node<K,V>[] table;

// 2. 扩容时使用的数组（正在 resize 时使用）
private transient volatile Node<K,V>[] nextTable;

// 3. 当前 map 中 key-value 的数量估算值（因为并发，所以是估算）
private transient volatile long baseCount;

// 4. 控制扩容的线程数（协助 transfer 的线程个数）
private transient volatile int transferIndex;

// 5. 用于控制 size 等统计值的计数数组，防止多线程竞争
private transient volatile long[] counterCells;

// 6. 用于控制表初始化和扩容操作的锁
private transient volatile int sizeCtl;

// 7. 在扩容过程中，每个线程通过 forwardingNode 协同进行迁移操作
private static final int MOVED = -1; // hash 值标识，表示正在转移

// 8. 标记一棵树的根节点的 hash 值
static final int TREEBIN = -2;

// 9. 用于 TreeNode 节点的 hash 值标记（红黑树结构）
static final int TREEHASH = -3;

// 10. 锁定节点，用于树的同步操作
static final int WRITER = -4;
static final int WAITER = -5;
static final int RESIZE_STAMP_SHIFT = 16;
```

## 四、源码分析

### 4.1 put操作核心源码分析

put方法签名如下:
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else if (onlyIfAbsent // check first node without acquiring lock
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
                return fv;
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
                                    pred.next = new Node<K,V>(hash, key, value);
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
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
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
判断如果key是空或者value是空直接抛出空指针异常，这与HashMap的源码不同，HashMap是可以放key是null的,获取key对应的hash值,接着对成员变量table进行一个for死循环操作,把它赋值给
一个变量tab,定义一些变量Node<K,V> f; int n, i, fh; K fk; V fv;判断tab为空，首次添加的时候一定是空的，那就需要初始化tab数组，调用initTable方法，该方法签名如下:
```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield();
            else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
}
```
首先有个while的自旋操作，结束条件是table不为null,判断sc=sizeCtl是否小于0,默认sizeCtl是等于0的,如果小于0表示有其余线程正在初始化或者扩容操作,
因为线程初始化或者扩容时候会通过CAS操作把sizeCtl设置为-1,下面就会看到，有线程正在初始化或扩容，当前线程让出CPU，调用的是Thread.yield()。

如果CAS操作争抢初始化权限把sizeCtl设为-1表示加锁成功，再次检查tab是否初始化过，如果没有设置初始容量(默认16或者由sizeCtl提供）,创建一个Node对象把它
赋值给成员属性table,设置(sc=sizeCtl) = n * 0.75,用于后续扩容判断,最终返回初始化后的table数组。

如果已经初始化过数组，代码逻辑会走到else if ((f = tabAt(tab, i = (n - 1) & hash)) == null)的判断,先获取待插入元素的下标，然后调用了tabAt
方法，该方法的签名如下：
```java
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getReferenceAcquire(tab, ((long)i << ASHIFT) + ABASE);
    }
```
这段代码不像平时那样直接用tab[i]来访问数组，而是通过Unsafe类 + 偏移地址计算来读数组元素。那么这段计算到底做了什么呢？我们逐步拆解。

| 表达式                  | 含义                          |
|----------------------|-----------------------------|
| i                    | 数组下标                        |
| ASHIFT       | 每个数组元素占用的字节大小(log2)         |
| i << ASHIFT | 表示当前下标i对应的内存偏移量  |
| ABASE          | 数组第一个元素在内存中的起始偏移量|

最终计算出的值是：数组中第i个元素在内存中的绝对偏移地址。这里为什么不直接用tab[i]那？因为ConcurrentHashMap是用在多线程环境中，多线程中读取有以下问题:

- **可见性问题：** 普通数组访问无法保证多线程下的内存可见性，可能读到旧值。

- **指令重排序问题：** JVM或CPU可能优化指令顺序，导致意外行为。

- **Unsafe的原子性：** getReferenceAcquire 保证了：
  - **最新值：** 读取时绕过线程缓存，直接读主内存。
  - **内存屏障：** acquire 语义防止后续操作被重排序到读操作之前。

继续前面分析如果得到数组元素的下标位置存储元素为null,调用casTabAt方法，该方法的签名如下:
```java
 static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSetReference(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```
可以看出这段代码是通过CAS操作往数组下标为i的地方放入元素，如果CAS成功直接break退出for死循环。

如果else if ((f = tabAt(tab, i = (n - 1) & hash)) == null)条件不成立会走到else if ((fh = f.hash) == MOVED)这个逻辑判断

&nbsp;&nbsp;这块主要看其他线程是否正在扩容，如果是就协助扩容操作。

如果走到else语句中说明当前桶不为空，且不在扩容状态已经发生了hash冲突的情况

&nbsp;&nbsp;对桶的头结点f(细粒度锁，只锁当前桶，其他桶仍可并发访问）,保证同一时间只有一个线程能修改该桶内的数据，执行检查if (tabAt(tab, i) == f),这段代码主要是用来
检查防止加锁前桶的头节点已被其他线程修改（例如扩容或树化）,比较当前桶的头节点是否仍是之前读取的f，如果不是则放弃操作(外层循环会重试),如果fn大于等于0，说明是链表结构，这是因为
不同类型的hash值代表不同节点类型，以下我画个表格说明:

| hash值          | 节点类型	                 | 用途	                         |
|----------------|-----------------------|-----------------------------|
| ≥ 0            | 普通链表节点(Node)          | 存储键值对，形成链表结构。               |
| = -1 (MOVED)   | 转发节点(ForwardingNode)  | 扩容时标记已迁移的桶，指向新数组。           |
| = -2(TREEBIN)  | 红黑树根节点(TreeBin)       | 代替链表头，管理红黑树操作。              |
| = -3(RESERVED) | 保留节点(ReservationNode) | 用于原子计算操作(如computeIfAbsent)。 |

- 链表情况:

  - 遍历链表，检查是否已存在相同的key：

    - 如果存在，根据 onlyIfAbsent 决定是否更新 value。
    - 如果不存在，在链表末尾插入新节点。
  - binCount 统计链表长度（用于后续判断是否需树化）。  

- 如果是红黑树情况:

&nbsp;&nbsp;binCount从2开始计数,调用TreeBin.putTreeVal方法插入或更新节点。

进行后续处理，判断binCount不等于0，判断链表长度是否大于等于8并且检查数组长度是否达到64，如果没达到优先进行扩容操作，这一点和HashMap一样。
如果是更新操作（oldVal != null）,返回旧值并结束循环。

接着调用addCount方法，该方法的签名是:
```java
private final void addCount(long x, int check) {
    CounterCell[] cs; long b, s;
    if ((cs = counterCells) != null ||
        !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell c; long v; int m;
        boolean uncontended = true;
        if (cs == null || (m = cs.length - 1) < 0 ||
            (c = cs[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n) << RESIZE_STAMP_SHIFT;
            if (sc < 0) {
                if (sc == rs + MAX_RESIZERS || sc == rs + 1 ||
                    (nt = nextTable) == null || transferIndex <= 0)
                    break;
                if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSetInt(this, SIZECTL, sc, rs + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```
该方法主要是统计ConcurrentHashMap的size值,就是元素总数,但由于高并发环境下直接更新单个变量会导致性能问题，ConcurrentHashMap采用了分层计数策略,baseCount 是其中的第一层。以下是详细解析：

**1. baseCount 的作用**

- **存储基础元素数量：**

  当并发竞争较低时，线程通过CAS（Compare-And-Swap） 直接更新baseCount，快速完成计数。

- **与counterCells协同工作：**

  当高并发导致CAS竞争激烈时，退化到分段计数（counterCells），此时baseCount仅作为总数的一部分。

**2. 为什么需要 baseCount?**

**(1) 性能优化**

- **低竞争场景：**

如果所有线程都能无竞争地通过CAS更新baseCount，则无需初始化复杂的counterCells,节省内存和计算开销。

- **高竞争场景**

当 CAS更新baseCount失败(说明多线程竞争)，才启用counterCells分散竞争。

(2) 最终一致性

- size() 的计算：

 最终的元素总数是 baseCount + ∑counterCells[i].value，通过sumCount()合并：

```java
final long sumCount() {
    CounterCell[] cs = counterCells;
    long sum = baseCount;  // 基础值
    if (cs != null) {
        for (CounterCell c : cs) {
            if (c != null) sum += c.value; // 累加分段计数
        }
    }
    return sum;
}
```
好了，知道以上前提下面一步步分析源码，判断CounterCell是否为空或者CAS更新baseCount的值是否成功，这里先讨论条件为真的情况，即CounterCell不为空(说明之前发生过竞争)或者
更新CAS失败的情况(其他线程已修改)，进入分段记时逻辑。

进入的时候还有四个条件判断能否真正进入分段记时逻辑，它们分别是cs == null,(m = cs.length - 1) < 0,(c = cs[ThreadLocalRandom.getProbe() & m]) == null,!(uncontended =U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))),下面我画个
表格来说明它们都是具体什么逻辑:

| 条件             | 含义	                         | 触发后的行为	                   |
|----------------|-----------------------------|---------------------------|
| cs == null            | counterCells 数组未初始化	        | 调用 fullAddCount 初始化并分配槽位             |
| (m = cs.length - 1) < 0  | counterCells 已初始化但长度为0(非法状态，通常不会发生) | 调用 fullAddCount 修复或扩容|
| (c = cs[ThreadLocalRandom.getProbe() & m]) == null | 当前线程的哈希槽位未分配(ThreadLocalRandom.getProbe()生成线程的随机哈希) | 调用 fullAddCount 分配槽位|
| !(uncontended =U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))) | CAS 更新槽位的值失败（说明槽位被其他线程占用）	  | 调用 fullAddCount 分配槽位。 |

下面我们来看看fullAddCount的方法签名:
```java
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;
    for (;;) {
        CounterCell[] cs; CounterCell c; int n; long v;
        if ((cs = counterCells) != null && (n = cs.length) > 0) {
            if ((c = cs[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {            // Try to attach new Cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    if (cellsBusy == 0 &&
                        U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)
                wasUncontended = true;
            else if (U.compareAndSetLong(c, CELLVALUE, v = c.value, v + x))
                break;
            else if (counterCells != cs || n >= NCPU)
                collide = false;
            else if (!collide)
                collide = true;
            else if (cellsBusy == 0 &&
                     U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == cs) 
                        counterCells = Arrays.copyOf(cs, n << 1);
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        else if (cellsBusy == 0 && counterCells == cs &&
                 U.compareAndSetInt(this, CELLSBUSY, 0, 1)) {
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == cs) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        else if (U.compareAndSetLong(this, BASECOUNT, v = baseCount, v + x))
            break;
    }
}
```
首先获取当前线程的初始化探针,如果等于0就初始化当前探针并把值赋值给临时变量h,用于定位counterCells槽位,把wasUncontended设置为true表示有竞争关系,设置collide变量为false,
接着一个for的死循环，定义变量CounterCell[] cs; CounterCell c; int n; long v;接着进行一系列判断。

- **counterCells已初始化情况**

    - 判断如果当前槽位为null

      - 判断cellsBusy是否等于0
        
           临时创建一个CounterCell对象
      
        - 再次判断cellsBusy是否等于0并且CAS是否成功
           
          把boolean created = false,判断counterCells不为null并且数组长度大于0并且当前槽位为null,把当前槽位的值赋值为上面新创建的CounterCell对象,设置created = true,最终设置cellsBusy=0,退出for死循环。

    - 如果当前槽位不为nll

       尝试用CAS更新当前槽位的值,如果更新成功则退出for死循环。

    - 如果超过CPU核心数

       把collide设置为false,表示不在进行扩容。
  
    - 判断代码条件cellsBusy == 0 && U.compareAndSetInt(this, CELLSBUSY, 0, 1)
       
       进行扩容操作，把counterCells容量翻倍，这样可以有效控制冲突的发生。

- **counterCells未初始化**

&nbsp;&nbsp;判断cellsBusy == 0 && counterCells == cs并且CAS自旋成功,创建一个容量是2的CounterCell数组,放置新Cell,成功以后把cellsBusy设置为0,然后结束for循环。

- **如果分段技术尝试都失败时候直接通过CAS更新全局基础计数器baseCount**

&nbsp;&nbsp;这个步骤是什么意思那？CAS将baseCount从当前值v更新为 v + x（x 通常是 +1 或 -1),如果CAS成功直接退出for循环,这是最终的保底方案,无论竞争多么激烈，最终都能通过baseCount确保计数更新不会丢失。

执行到此处fullAddCount方法执行完毕返回返回到addCount方法中，此时调用sumCount方法，该方法的签名如下:
```java
final long sumCount() {
    CounterCell[] cs = counterCells;
    long sum = baseCount;
    if (cs != null) {
        for (CounterCell c : cs)
            if (c != null)
                sum += c.value;
    }
    return sum;
}
```
把baseCount赋值给sum,循环CounterCell数组，累加sum值，最终返回sum,可以看出这段代码在高并发情况下会有计算问题，是线程不安全的，它只是一个估算值并不是准确值。

### 4.2 get操作核心源码分析
&nbsp;&nbsp;该方法的签名如下:
```java
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
```
定义一些临时变量，获取要查找key的hashCode值，判断如果table不为空并且调用tabAt方法获取元素不为空，把获取到元素赋值给变量Node e,比较e.hash是否等于计算出的hash
如果相等并且equals值也相等直接返回e的val值。

如果不相等说明是hash冲突了，判断如果eh的hash值小于0，说明可能是红黑树的TreeBin结构,调用调用其find方法进线查找。

如果以上都不是那么就是链表,一个while的死循环,循环结束条件是e=e.next == null,如果通过链表的next一直能找到相等的hash并且equals相等就返回当前链表的val值,否则就返回null。