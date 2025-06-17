# ForkJoinPool 源码分析与示例

## 目录

- [ForkJoinPool 简介](#一forkjoinpool简介)
- [ForkJoinPool 设计理念](#二forkjoinpool设计理念)
- [核心类和继承关系](#三继承关系)
- [ForkJoinPool 主要成员字段和构造函数](#四主要成员字段和构造函数)
- [工作窃取算法（Work Stealing）简介](#五工作窃取算法work-stealing简介)
- [ForkJoinPool 任务执行流程](#六forkjoinpool任务执行流程)
- [ForkJoinTask 概述](#七forkjointask概述)
- [RecursiveTask 与 RecursiveAction 区别](#八recursivetask与recursiveaction区别)
- [示例：ForkJoinSumExample 数组求和](#九示例forkjoinsumexample-数组求和)
  - [示例代码详解](#91-示例代码详解)
  - [fork() 与 join() 的作用](#92-fork与join的作用)
  - [工作线程的执行机制](#93-工作线程的执行机制)
- [Invoke方法源码解析](#十invoke方法源码解析)
- [总结](#十一-总结)

## 一、ForkJoinPool简介

- ForkJoinPool是Java引入的高效并行任务执行框架，基于分治思想。

- 适合将大任务拆分为小任务并行处理。

- 利用多核CPU，提升计算密集型任务性能。

- 兼顾任务拆分、调度和负载均衡。

- 内置工作窃取算法(work-stealing)。

- 本文基于JDK21进行源码分析。

## 二、ForkJoinPool设计理念

- 任务分割(fork)和合并(join)

- 工作窃取(work stealing)算法实现线程间负载均衡

- 每个工作线程维护自己的任务双端队列(deque)

- 线程空闲时窃取其他线程队列尾部任务，避免线程饥饿

- 减少线程切换开销和资源竞争

## 三、继承关系
```java
public class ForkJoinPool extends AbstractExecutorService
```
它的类的继承结构图如下：

![img_5.png](..%2Fimg%2Fimg_5.png)

- ForkJoinPool继承自AbstractExecutorService

- 核心执行任务的线程类是 ForkJoinWorkerThread

- 任务基类是 ForkJoinTask(抽象类)

- 两个常用子类：

  - RecursiveTask<V>：有返回值任务

  - RecursiveAction：无返回值任务

线程池管理工作线程，任务通过 ForkJoinTask提交执行

## 四、主要成员字段和构造函数
| 参数名                    |               类型                |              作用              |
  |:-----------------------|:-------------------------------:|:----------------------------:|
| parallelism	 |               int               |        表示并行度，也就是目标线程数，通常建议设置为 Runtime.getRuntime().availableProcessors()        |
| factory          |  ForkJoinWorkerThreadFactory	   | 用于自定义工作线程的创建方式，比如加日志、命名等 
| handler | Thread.UncaughtExceptionHandler | 如果任务抛出未捕获异常，如何处理（比如记录日志） |
| asyncMode                     |            boolean	             |      是否启用异步模式，默认 false 是 LIFO（后进先出），用于递归任务更快合并；设置为 true 是 FIFO，更像普通线程池      |
| corePoolSize                     |              int	               |      （高级用法）控制线程池核心线程数量，仅在“托管模式”才会生效（常用构造函数不会暴露这个）      |
| maximumPoolSize                     |              int	               |      最大线程数限制（托管池中才用到）     |
| minimumRunnable                     |              int	               |      	保证多少个任务是 runnable 状态，用于动态扩容线程（托管池相关）     |
| saturate                     |              Predicate<ForkJoinPool>	               |      	判断是否“饱和”，触发线程扩展的策略     |
| keepAliveTime                     |              long	               |      	线程空闲多久后被回收（适用于托管模式）    |
| unit                     |              TimeUnit	               |      	keepAliveTime 的时间单位   |

说明一下:这里的corePoolSize看似像普通线程池的中的核心线程数，但是这里它并不是普通线程池那个参数，它是jdk21中的托管模式采用的，实际上控制线程数量的参数是
parallelism，这里知道一下就可以了，不必深究。

&nbsp;&nbsp; 构造函数主要是根据当前计算的Runtime.getRuntime().availableProcessors()初始化工作队列容量，假如Runtime.getRuntime().availableProcessors()=16，通过计算
int size = 1 << (33 - Integer.numberOfLeadingZeros(p - 1));size值是32,工作队列WorkQueue数组容量就是32

## 五、工作窃取算法(Work Stealing)简介

- 每个工作线程拥有自己的任务双端队列

- 线程只从队列头部执行任务

- 保障高CPU利用率，避免线程空闲

- 减少同步开销，提升并行效率

## 六、ForkJoinPool任务执行流程

- 任务通过submit()、invoke()或execute()进入线程池

- 任务被放入当前线程或工作线程的任务队列

- 工作线程从队列头部取任务执行

- 任务执行时可拆分成子任务，递归调用fork()和join()

- 线程完成任务后继续执行队列中或窃取任务

## 七、ForkJoinTask概述

- 抽象基类，代表可分割的任务

- 任务拆分执行核心

- 提供fork()、join()、compute()方法

- 支持任务取消、完成状态管理

- 关联ForkJoinPool线程执行

## 八、RecursiveTask与RecursiveAction区别

- RecursiveTask<V>：有返回值任务，重写compute()返回结果

- RecursiveAction：无返回值任务，重写compute()无返回值

- 适合不同场景，区别使用

## 九、示例：ForkJoinSumExample 数组求和

**以这个例子来分析源码**

```java
public class ForkJoinSumExample {

    static class SumTask extends RecursiveTask<Long> {
        private final long[] arr;
        private final int start, end;
        private static final int THRESHOLD = 10_000; // 阈值，分割任务大小

        SumTask(long[] arr, int start, int end) {
            this.arr = arr;
            this.start = start;
            this.end = end;
        }

        @Override
        protected Long compute() {
            int length = end - start;
            if (length <= THRESHOLD) {
                // 任务足够小，直接计算
                long sum = 0;
                for (int i = start; i < end; i++) {
                    sum += arr[i];
                }
                return sum;
            } else {
                // 任务过大，拆分为两个子任务
                int mid = start + length / 2;
                SumTask leftTask = new SumTask(arr, start, mid);
                SumTask rightTask = new SumTask(arr, mid, end);

                // fork() 异步执行左边任务
                leftTask.fork();

                // 当前线程同步计算右边任务
                long rightResult = rightTask.compute();

                // 等待左边任务执行完成，获得结果
                long leftResult = leftTask.join();

                return leftResult + rightResult;
            }
        }
    }

    public static void main(String[] args) {
        long[] array = new long[1_000_000];
        for (int i = 0; i < array.length; i++) {
            array[i] = i + 1; // 赋值 1 到 1,000,000
        }

        ForkJoinPool forkJoinPool = new ForkJoinPool();
        SumTask task = new SumTask(array, 0, array.length);

        long result = forkJoinPool.invoke(task);

        System.out.println("Sum: " + result); // 期望结果：500000500000
    }
}
```
### 9.1 示例代码详解
&nbsp;&nbsp;写一个任务SumTask继承RecursiveTask代表有返回值，实现RecursiveTask的compute方法，通过end-start值是否小于阈值判断是否返回，如果小于就返回，如果大于就
继续fork、join、compute进行递归调用拆分任务直到小于阈值为止。

### 9.2 fork()与join()的作用

- fork()异步提交子任务执行

- join()等待子任务执行完成并获取结果

- 当前线程利用compute()同步计算另一半任务

### 9.3 工作线程的执行机制

- ForkJoinPool线程池分配线程执行任务

- 利用工作窃取保证线程不空闲

- 实现高效并行执行

## 十、Invoke方法源码解析

首先调用的是invoke方法，该方法的签名如下：
```java
 public <T> T invoke(ForkJoinTask<T> task) {
      poolSubmit(true, task);
      return task.join();
  }
```
这个方法又调用了poolSubmit方法，该方法的签名如下:
```java
private <T> ForkJoinTask<T> poolSubmit(boolean signalIfEmpty,
                                       ForkJoinTask<T> task) {
    WorkQueue q; Thread t; ForkJoinWorkerThread wt;
    U.storeStoreFence();
    if (task == null) throw new NullPointerException();
    if (((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) &&
        (wt = (ForkJoinWorkerThread)t).pool == this)
        q = wt.workQueue;
    else {
        task.markPoolSubmission();
        q = submissionQueue(true);
    }
    q.push(task, this, signalIfEmpty);
    return task;
}
```
执行main方法时候指定不是ForkJoinWorkerThread线程，所以会走到else语句，这里面我主要挑重点的源码讲，调用的是submissionQueue方法,该方法签名如下:
```java
 final WorkQueue submissionQueue(boolean isSubmit) {
    int r;
    ReentrantLock lock = registrationLock;
    if ((r = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();     
        r = ThreadLocalRandom.getProbe();
    }
    if (lock != null) { 
        for (int id = r << 1;;) { 
            int n, i; WorkQueue[] qs; WorkQueue q;
            if ((qs = queues) == null || (n = qs.length) <= 0)
                break;
            else if ((q = qs[i = (n - 1) & id]) == null) {
                WorkQueue w = new WorkQueue(null, id | SRC);
                w.array = new ForkJoinTask<?>[INITIAL_QUEUE_CAPACITY];
                lock.lock();
                if (queues == qs && qs[i] == null)
                    qs[i] = w;
                lock.unlock();
            }
            else if (q.getAndSetAccess(1) != 0) 
                id = (r = ThreadLocalRandom.advanceProbe(r)) << 1;
            else if (isSubmit && runState != 0) {
                q.access = 0;
                break;
            }
            else
                return q;
        }
    }
    throw new RejectedExecutionException();
}
```
给当前线程初始化一个探针值，然后左移1位得到id值，计算数组下标值i = (n - 1) & id,创建一个WorkQueue，初始化里面的arry属性，该值是一个
ForkJoinTask数组，初始化容量是64，根据计算的下标值放入到WorkQueue数组相应位置,最终返回创建的WorkQueue。

此时方法执行返回到poolSubmit,会调用q.push(task, this, signalIfEmpty)方法，该方法的签名如下:
```java
final void push(ForkJoinTask<?> task, ForkJoinPool pool,boolean signalIfEmpty) {
    boolean resize = false;
    int s = top++, b = base, cap, m; ForkJoinTask<?>[] a;
    if ((a = array) != null && (cap = a.length) > 0) {
        if ((m = (cap - 1)) == s - b) {
            resize = true; 
            int newCap = (cap < 1 << 24) ? cap << 2 : cap << 1;
            ForkJoinTask<?>[] newArray;
            try {
                newArray = new ForkJoinTask<?>[newCap];
            } catch (Throwable ex) {
                top = s;
                access = 0;
                throw new RejectedExecutionException(
                    "Queue capacity exceeded");
            }
            if (newCap > 0) { 
                int newMask = newCap - 1, k = s;
                do { 
                    newArray[k-- & newMask] = task;
                } while ((task = getAndClearSlot(a, k & m)) != null);
            }
            array = newArray;
        }
        else
            a[m & s] = task;
        getAndSetAccess(0);
        if ((resize || (a[m & (s - 1)] == null && signalIfEmpty)) &&
            pool != null)
            pool.signalWork();
    }
}
```
top++：递增top指针(任务尾部)，表示要在队列“尾部”放入任务,base是任务头部,cap为当前数组容量，m = cap - 1 用作掩码，用于环形索引,a是当前当前任务数组。

```java
if ((m = (cap - 1)) == s - b) {
    resize = true;
    int newCap = (cap < 1 << 24) ? cap << 2 : cap << 1;
```
这一小段代码首先判断队列是否已满，如果已满设置resize = true：标记需要扩容。

扩容策略主要是:

- 小于16M的数组4倍扩容；
- 否则2倍扩容(防止太大)。

```java
ForkJoinTask<?>[] newArray;
try {
    newArray = new ForkJoinTask<?>[newCap];
} catch (Throwable ex) {
    top = s;
    access = 0;
    throw new RejectedExecutionException("Queue capacity exceeded");
}
```
这一小段代码如果分配失败则回滚top并抛出拒绝执行异常。

```java
if (newCap > 0) {
    int newMask = newCap - 1, k = s;
    do {
        newArray[k-- & newMask] = task;
    } while ((task = getAndClearSlot(a, k & m)) != null);
}
array = newArray;
```
这一小段代码执行逻辑是:

- 将旧队列中的任务逐个迁移到新队列。
- getAndClearSlot()：取出旧数组中的任务并清除引用。

如果不需要扩容执行a[m & s] = task;直接把任务放到数组中的 s（top）位置上。

```java
getAndSetAccess(0);
if ((resize || (a[m & (s - 1)] == null && signalIfEmpty)) && pool != null)
    pool.signalWork();
```
这一小段代码的逻辑是:

- 如果刚刚扩容完毕，或者任务 push 前是空队列，signalIfEmpty = true，就需要唤醒线程。

- signalWork() 是 ForkJoinPool 的方法，用于唤醒等待的 worker 线程。

下面我将分析signalWork的源码，该方法源码签名如下:
```java
final void signalWork() {
    int pc = parallelism, n;
    long c = ctl;
    WorkQueue[] qs = queues;
    if ((short)(c >>> RC_SHIFT) < pc && qs != null && (n = qs.length) > 0) {
        for (;;) {
            boolean create = false;
            int sp = (int)c & ~INACTIVE;
            WorkQueue v = qs[sp & (n - 1)];
            int deficit = pc - (short)(c >>> TC_SHIFT);
            long ac = (c + RC_UNIT) & RC_MASK, nc;
            if (sp != 0 && v != null)
                nc = (v.stackPred & SP_MASK) | (c & TC_MASK);
            else if (deficit <= 0)
                break;
            else {
                create = true;
                nc = ((c + TC_UNIT) & TC_MASK);
            }
            if (c == (c = compareAndExchangeCtl(c, nc | ac))) {
                if (create)
                    createWorker();
                else {
                    Thread owner = v.owner;
                    v.phase = sp;
                    if (v.access == PARKED)
                        LockSupport.unpark(owner);
                }
                break;
            }
        }
    }
}
```
parallelism：并行度配置，也就是核心线程数量;ctl：ForkJoinPool的核心状态变量，封装了线程数量、空闲线程栈等信息;queues：所有 WorkQueue数组，表示工作线程的任务队列。

```java
if ((short)(c >>> RC_SHIFT) < pc && qs != null && (n = qs.length) > 0)
```
这段代码主要是把c右移48位(获取高16位结果)获取活动线程数量是否小于并行度，如果线程池中还有可创建/唤醒的空间，就继续下面的逻辑，否则就什么都不做。

```java
for (;;){
  boolean create=false;
  int sp=(int)c&~INACTIVE; 
  WorkQueue v=qs[sp&(n-1)];
}
```
接着这一小段代码是一个for的死循环，sp变量主要是获取栈顶指针stackPred，去除INACTIVE 标志，然后找到栈顶的空闲线程。

```java
int deficit = pc - (short)(c >>> TC_SHIFT); // 并行度 - 总线程数
long ac = (c + RC_UNIT) & RC_MASK, nc;
```
deficit: 判断还能不能创建线程;ac: 增加活跃线程计数后的新值

```java
  if (sp != 0 && v != null)
      nc = (v.stackPred & SP_MASK) | (c & TC_MASK);
```
如果栈顶有空闲线程，则弹栈，准备唤醒线程。

```java
  else if (deficit <= 0)
      break;
  else {
      create = true;
      nc = ((c + TC_UNIT) & TC_MASK);
  }
```
- 如果不能再创建线程（deficit <= 0），就退出。

- 否则设置 create = true，表示要创建新线程。

```java
    if (c == (c = compareAndExchangeCtl(c, nc | ac))) {
```
compareAndExchangeCtl() 尝试CAS设置新ctl值,如果create值是true,调用createWorker方法创建新线程，新创建的线程对象是ForkJoinWorkerThread,否则就是唤醒。

我们主要看ForkJoinWorkerThread的构造函数都做了什么，已经他有哪些成员属性，构造函数方法签名如下:
```java
ForkJoinWorkerThread(ThreadGroup group, ForkJoinPool pool,
                     boolean useSystemClassLoader,
                     boolean clearThreadLocals) {
    super(group, null, pool.nextWorkerThreadName(), 0L, !clearThreadLocals);
    UncaughtExceptionHandler handler = (this.pool = pool).ueh;
    this.workQueue = new ForkJoinPool.WorkQueue(this, 0);
    if (clearThreadLocals)
        workQueue.setClearThreadLocals();
    super.setDaemon(true);
    if (handler != null)
        super.setUncaughtExceptionHandler(handler);
    if (useSystemClassLoader)
        super.setContextClassLoader(ClassLoader.getSystemClassLoader());
}
```
这里主要分析一下重点的属性，有个ForkJoinPool.WorkQueue成员属性，即每个线程都有自己的队列，队列成员属性同样维护了ForkJoinWorkerThread对象,下面我将分析ForkJoinWorkerThread的run
方法，该方法是线程执行的主要逻辑，代码签名如下:
```java
public void run() {
        Throwable exception = null;
        ForkJoinPool p = pool;
        ForkJoinPool.WorkQueue w = workQueue;
        if (p != null && w != null) {
            try {
                p.registerWorker(w);
                onStart();
                p.runWorker(w);
            } catch (Throwable ex) {
                exception = ex;
            } finally {
                try {
                    onTermination(exception);
                } catch (Throwable ex) {
                    if (exception == null)
                        exception = ex;
                } finally {
                    p.deregisterWorker(this, exception);
                }
            }
        }
}
```
可以看出这个方法先调用了registerWorker方法,该方法的签名如下:
```java
final void registerWorker(WorkQueue w) {
      ThreadLocalRandom.localInit();
      int seed = ThreadLocalRandom.getProbe();
      ReentrantLock lock = registrationLock;
      int cfg = config & FIFO;
      if (w != null && lock != null) {
          w.array = new ForkJoinTask<?>[INITIAL_QUEUE_CAPACITY];
          cfg |= w.config | SRC;
          w.stackPred = seed;
          int id = (seed << 1) | 1; 
          lock.lock();
          try {
              WorkQueue[] qs; int n;
              if ((qs = queues) != null && (n = qs.length) > 0) {
                  int k = n, m = n - 1;
                  for (; qs[id &= m] != null && k > 0; id -= 2, k -= 2);
                  if (k == 0)
                      id = n | 1;
                  w.phase = w.config = id | cfg;

                  if (id < n)
                      qs[id] = w;
                  else { 
                      int an = n << 1, am = an - 1;
                      WorkQueue[] as = new WorkQueue[an];
                      as[id & am] = w;
                      for (int j = 1; j < n; j += 2)
                          as[j] = qs[j];
                      for (int j = 0; j < n; j += 2) {
                          WorkQueue q;
                          if ((q = qs[j]) != null)
                              as[q.config & am] = q;
                      }
                      U.storeFence();
                      queues = as;
                  }
              }
          } finally {
              lock.unlock();
          }
      }
}
```
这个方法主要是做了初始化当前线程的probe值(随机数种子)，用于计算WorkQueue的初始索引（避免冲突）,然后分配当前线程队列的任务数组,
设置stackPred:记录前一个栈帧位置（链表栈结构),id作WorkQueue在数组中的位置（奇数位，偶数位用于 external 提交线程）,接着调用lock.lock()进线加锁注册保证线程安全
```java
int k = n, m = n - 1;
for (; qs[id &= m] != null && k > 0; id -= 2, k -= 2);
```
这一小段代码的作用主要是:

- id &= m：保证 id 在 [0, n-1] 范围

- id -= 2：跳过偶数位(给external用)

- 如果找不到空位，就准备扩容

如果id < n就把当前队列对象注册到总的WorkQueue数组上,如果数组满了，就扩容queues,容量进行翻倍,将新WorkQueue写入新的索引位置,将旧队列复制到新数组中然后发布新队列。

下面我将分析ForkJoinWorkerThread的runWorker方法,该方法的签名如下:
```java
final void runWorker(WorkQueue w) {
    if (w != null) { 
        int r = w.stackPred, src = 0;
        do {
            r ^= r << 13; r ^= r >>> 17; r ^= r << 5;
        } while ((src = scan(w, src, r)) >= 0 ||
                 (src = awaitWork(w)) == 0);
        w.access = STOP;
    }
}
```
- stackPred是当前线程注册时生成的伪随机种子，用于确定工作窃取的扫描起点。

- src是上一次从哪个线程偷的任务，用于记录扫描路径。

```java
r ^= r << 13;
r ^= r >>> 17;
r ^= r << 5;
```
这是典型的XorShift随机数算法，用于生成一个随机数 r：

- 用于扰乱scan顺序，避免多个线程从同一个地方开始窃取任务，造成资源竞争；
- 保证工作分布更均匀。

接着走到while循环调用scan方法，这个方法作用：

- 尝试从自己的队列或其他工作线程中获取一个任务来执行(包括窃取)。

- 返回目标任务来源索引(>=0 表示有任务)。

scan方法签名如下:
```java
 private int scan(WorkQueue w, int prevSrc, int r) {
    WorkQueue[] qs = queues;
    int n = (w == null || qs == null) ? 0 : qs.length;
    for (int step = (r >>> 16) | 1, i = n; i > 0; --i, r += step) {
        int j, cap; WorkQueue q; ForkJoinTask<?>[] a;
        if ((q = qs[j = r & (n - 1)]) != null &&
            (a = q.array) != null && (cap = a.length) > 0) {
            int src = j | SRC, b = q.base;
            int k = (cap - 1) & b, nb = b + 1, nk = (cap - 1) & nb;
            ForkJoinTask<?> t = a[k];
            U.loadFence(); 
            if (q.base != b) 
                return prevSrc;
            else if (t != null && WorkQueue.casSlotToNull(a, k, t)) {
                q.base = nb;
                w.source = src;
                if (src + (src << SWIDTH) != prevSrc &&
                    q.base == nb && a[nk] != null)
                    signalWork();
                w.topLevelExec(t, q);
                return src + (prevSrc << SWIDTH);
            }
            else if (q.array != a || a[k] != null || a[nk] != null)
                return prevSrc; 
        }
    }
    return -1;
}
```
拿到当前池中所有的工作队列qs,n是队列数量。如果当前工作线程或队列为空，则无法进行扫描，直接 n=0。

```java
for (int step = (r >>> 16) | 1, i = n; i > 0; --i, r += step) {
```
这一小段代码主要做的事情主要有以下几点:

- r >>> 16是扰动值右移16位，获得一个“步长 step”

- | 1 保证步长是奇数，奇数可以遍历整个环形数组，不遗漏队列

- r += step 让扫描顺序看起来是“随机的”

- i = n 是控制最多扫描n个队列（不重复）

接下来获取下标j = r & (n - 1),用于在环形数组中取模,如果通过下标j获取得WorkQueue不为空并且里面得任务不为空，这里定义一些遍历，它们代表得含义是:

- src是来源队列的标识

- b是当前队列的base 指针(任务从这里出队)

- k是base在数组中的索引(环形队列),k=(cap - 1) & b可知偷队列是从0依次往后增加，这就是LIFO的偷取策略。

- t = a[k]是拿到准备偷的任务

- U.loadFence() 是内存屏障，防止指令重排

```java
if (q.base != b)  
    return prevSrc;
else if (t != null && WorkQueue.casSlotToNull(a, k, t)) {
    q.base = nb;
    w.source = src;
```
这一小端代码的主要作用是:

- 判断之前拿到的b值是否还等于WorkQueue的base值，如果不等于说明别的线程抢先偷走了，退出这轮尝试。

- 否则CAS设置该槽为 null，代表偷到了任务。

- 更新base指针,把src记为当前任务来源。

```java
if (src + (src << SWIDTH) != prevSrc &&
    q.base == nb && a[nk] != null)
    signalWork(); 

w.topLevelExec(t, q); 
return src + (prevSrc << SWIDTH);
```
- 如果前后来源不同，并且还有下一个任务（a[nk] != null），可能还需要叫醒别的线程来抢活干。

- topLevelExec执行偷来的任务。

- 返回新来源 src + (prevSrc << SWIDTH) 记录窃取来源轨迹。

这里我将主要分析topLevelExec方法,该方法的签名如下:
```java
final void topLevelExec(ForkJoinTask<?> task, WorkQueue src) {
    int cfg = config, fifo = cfg & FIFO, nstolen = 1;
    while (task != null) {
        task.doExec();
        if ((task = nextLocalTask(fifo)) == null &&
            src != null && (task = src.tryPoll()) != null)
            ++nstolen;
    }
    nsteals += nstolen;
    source = 0;
    if ((cfg & CLEAR_TLS) != 0)
        ThreadLocalRandom.eraseThreadLocals(Thread.currentThread());
}
```
主要是调用task的doExec方法，该方法的签名如下:
```java
final int doExec() {
      int s; boolean completed;
      if ((s = status) >= 0) {
          try {
              completed = exec();
          } catch (Throwable rex) {
              s = trySetException(rex);
              completed = false;
          }
          if (completed)
              s = setDone();
      }
      return s;
}
```
判断ForkJoinTask的status是否大于等于0,如果是说明任务还没执行完成,如果是小于0的说明任务是执行完成的。现在主要分析任务没执行完成
的情况，它是调用的exec方法,exec方法是抽象方法，它的实现类是RecursiveTask,它实现了exec方法，该方法源码如下:
```java
protected final boolean exec() {
    result = compute();
    return true;
}
```
可以看出这里调用的是一个抽象方法compute，这个方法是需要我们业务自己实现的，我上面的例子SumTask就实现此抽象方法，那么就会调用到这里面来，我再把
这个方法的实现贴出来一下。
```java
protected Long compute() {
    int length = end - start;
    if (length <= THRESHOLD) {
        System.out.println("计算区间: " + start + "~" + end + "，线程：" + Thread.currentThread().getName());
        // 任务足够小，直接计算
        long sum = 0;
        for (int i = start; i < end; i++) {
            sum += arr[i];
        }
        return sum;
    } else {
        // 任务过大，拆分为两个子任务
        int mid = start + length / 2;
        SumTask leftTask = new SumTask(arr, start, mid);
        SumTask rightTask = new SumTask(arr, mid, end);

        // fork() 异步执行左边任务
        leftTask.fork();

        // 当前线程同步计算右边任务
        long rightResult = rightTask.compute();

        // 等待左边任务执行完成，获得结果
        long leftResult = leftTask.join();

        return leftResult + rightResult;
    }
}
```
从这个方法可以看出计算end-start值如果小于指定阈值就会返回结束此方法，但这不是重点，重点是分析它们是如何拆分任务的，如果走到else语句就会折半计算
中间值重新拆分左任务和右任务，然后调用leftTask的fork方法，该方法的签名如下:
```java
 public final ForkJoinTask<V> fork() {
    Thread t; ForkJoinWorkerThread wt;
    ForkJoinPool p; ForkJoinPool.WorkQueue q;
    U.storeStoreFence();
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) {
        p = (wt = (ForkJoinWorkerThread)t).pool;
        q = wt.workQueue;
    }
    else
        q = (p = ForkJoinPool.common).submissionQueue(false);
    q.push(this, p, true);
    return this;
}
```
判断如果当前线程是ForkJoinWorkerThread就把任务放入到直接的WorkQueue的任务队列里面去，通过压栈的方式，放到最顶部。

此时继续返回到compute方法中执行，执行到rightTask的compute方法，可以看出这块是一个递归操作，都需要等待被操作的方法返回时候才能陆续向前返回，
假如此时已经返回，继续执行leftTask.join()，join方法的签名如下:
```java
public final V join() {
    int s;
    if ((s = status) >= 0)
        s = awaitDone(s & POOLSUBMIT, 0L);
    if ((s & ABNORMAL) != 0)
        reportException(s);
    return getRawResult();
}
```
根据任务的状态进行逻辑判断，如果大于0说明是没执行完同步调用awaitDone方法等待执行完成，如果执行异常调用reportException方法，最终通过
getRawResult方法返回计算后的值，这里我分析一下awaitDone方法,该方法的签名如下:
```java
private int awaitDone(int how, long deadline) {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool p;
    ForkJoinPool.WorkQueue q = null;
    boolean timed = (how & TIMED) != 0;
    boolean owned = false, uncompensate = false;
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) {
        owned = true;
        q = (wt = (ForkJoinWorkerThread)t).workQueue;
        p = wt.pool;
    }
    else if ((p = ForkJoinPool.common) != null && (how & POOLSUBMIT) == 0)
        q = p.externalQueue();
    if (q != null && p != null) {
        if (this instanceof CountedCompleter)
            s = p.helpComplete(this, q, owned, timed);
        else if ((how & RAN) != 0 ||
                 (s = q.tryRemoveAndExec(this, owned)) >= 0)
            s = (owned) ? p.helpJoin(this, q, timed) : 0;
        if (s < 0)
            return s;
        if (s == UNCOMPENSATE)
            uncompensate = true;
    }
    Aux node = null;
    long ns = 0L;
    boolean interrupted = false, queued = false;
    for (;;) {
        Aux a;
        if ((s = status) < 0)
            break;
        else if (node == null)
            node = new Aux(Thread.currentThread(), null);
        else if (!queued) {
            if (((a = aux) == null || a.ex == null) &&
                (queued = casAux(node.next = a, node)))
                LockSupport.setCurrentBlocker(this);
        }
        else if (timed && (ns = deadline - System.nanoTime()) <= 0) {
            s = 0;
            break;
        }
        else if (Thread.interrupted()) {
            interrupted = true;
            if ((how & POOLSUBMIT) != 0 && p != null && p.runState < 0)
                cancelIgnoringExceptions(this);
            else if ((how & INTERRUPTIBLE) != 0) {
                s = ABNORMAL;
                break;
            }
        }
        else if ((s = status) < 0)
            break;
        else if (timed)
            LockSupport.parkNanos(ns);
        else
            LockSupport.park();
    }
    if (uncompensate)
        p.uncompensate();

    if (queued) {
        LockSupport.setCurrentBlocker(null);
        if (s >= 0) {
            outer: for (Aux a; (a = aux) != null && a.ex == null; ) {
                for (Aux trail = null;;) {
                    Aux next = a.next;
                    if (a == node) {
                        if (trail != null)
                            trail.casNext(trail, next);
                        else if (casAux(a, next))
                            break outer;
                        break;
                    } else {
                        trail = a;
                        if ((a = next) == null)
                            break outer;
                    }
                }
            }
        }
        else {
            signalWaiters();
            if (interrupted)
                Thread.currentThread().interrupt();
        }
    }
    return s;
}
```
可以看出这段代码很复杂，它的主要功能有：

- 辅助执行(帮助自己或子任务);
- 阻塞等待(自旋 + park);
- 超时与中断;
- 线程间唤醒与清理。

由于很复杂我只分析一下它的主要逻辑，包括自己执行方法、帮助别人执行方法、线程等待逻辑。

- how变量，它是一个 标志位，表示等待的策略（是否可中断、是否限时、是否是外部线程等），是一个位掩码；
- deadline: 如果有超时，则表示等待的截止时间（System.nanoTime() 单位）。

判断当前线程是否是ForkJoinPool线程，如果是把owned设置为true,表示是“池中线程”,获取当前线程的workQueue;如果不是则尝试使用common pool提交队列。
接下来执行判断(how & RAN) != 0 ||(s = q.tryRemoveAndExec(this, owned)) >= 0的逻辑，先分析一下tryRemoveAndExec方法，该方法签名如下:
```java
final int tryRemoveAndExec(ForkJoinTask<?> task, boolean owned) {
    ForkJoinTask<?>[] a = array;
    int p = top, s = p - 1, d = p - base, cap;
    if (task != null && d > 0 && a != null && (cap = a.length) > 0) {
        for (int m = cap - 1, i = s; ; --i) {
            ForkJoinTask<?> t; int k;
            if ((t = a[k = i & m]) == task) {
                if (!owned && getAndSetAccess(1) != 0)
                    break;
                else if (top != p || a[k] != task ||
                         getAndClearSlot(a, k) == null) {
                    access = 0;
                    break;
                }
                else {
                    if (i != s && i == base)
                        base = i + 1;
                    else {
                        for (int j = i; j != s;)
                            a[j & m] = getAndClearSlot(a, ++j & m);
                        top = s;
                    }
                    releaseAccess();
                    return task.doExec();
                }
            }
            else if (t == null || --d == 0)
                break;
        }
    }
    return 0;
}
```
这段代码是尝试从自己的队列中取出任务并且立即执行，是LIFO模式(后进先出)，从栈顶取任务，下面详细分析一下它的执行流程。
task参数是希望立即执行的任务（通常是自己 join() 的目标任务）,owned: 当前线程是否拥有这个队列（是否是本线程自己的队列）：
- true：代表是队列所有者线程
- false：代表是其他线程在尝试从这个队列移除任务(需要加锁)

```java
for (int m = cap - 1, i = s; ; --i)
```
这一小段代码的逻辑是i从top - 1开始，往前扫描,查找目标task,如果下标命中当前目标task, 再次验证是否未被其他线程篡改,双重检查确保这task还在原地,如果不在原地需要退出
```java
 if (i != s && i == base)
     base = i + 1;
```
这一小段代码表示如果任务在队头把base值加1，如果是在中间位置需要移动任务元素保持数组连续性，最后调用task.doExec()又去执行业务逻辑，返回status状态，如果执行完成一定返回负数，如果返回0代表当前线程没有任务可能被别的
线程偷走了任务，这块至关重要，因为根据是否大于0去调用helpJoin方法，只有大于等于0才去调用helpJoin方法，说明当前线程没有这个任务，去帮助别的任务进行join,那接着分析
helpJoin方法，该方法的签名如下:
```java
final int helpJoin(ForkJoinTask<?> task, WorkQueue w, boolean timed) {
    if (w == null || task == null)
        return 0;
    int wsrc = w.source, wid = (w.config & SMASK) | SRC, r = wid + 2;
    long sctl = 0L; 
    for (boolean rescan = true;;) {
        int s; WorkQueue[] qs;
        if ((s = task.status) < 0)
            return s;
        if (!rescan && sctl == (sctl = ctl)) {
            if (runState < 0)
                return 0;
            if ((s = tryCompensate(sctl, timed)) >= 0)
                return s;
        }
        rescan = false;
        int n = ((qs = queues) == null) ? 0 : qs.length, m = n - 1;
        scan: for (int i = n >>> 1; i > 0; --i, r += 2) {
            int j, cap; WorkQueue q; ForkJoinTask<?>[] a;
            if ((q = qs[j = r & m]) != null && (a = q.array) != null &&
                (cap = a.length) > 0) {
                for (int src = j | SRC;;) {
                    int sq = q.source, b = q.base;
                    int k = (cap - 1) & b, nb = b + 1;
                    ForkJoinTask<?> t = a[k];
                    U.loadFence();
                    boolean eligible = true;
                    for (int d = n, v = sq;;) {
                        WorkQueue p;
                        if (v == wid)
                            break;
                        if (v == 0 || --d == 0 || (p = qs[v & m]) == null) {
                            eligible = false;
                            break;
                        }
                        v = p.source;
                    }
                    if (q.source != sq || q.base != b)
                        ;      
                    else if ((s = task.status) < 0)
                        return s;  
                    else if (t == null) {
                        if (a[k] == null) {
                            if (!rescan && eligible &&
                                (q.array != a || q.top != b))
                                rescan = true;  
                            break;
                        }
                    }
                    else if (t != task && !eligible)
                        break;
                    else if (WorkQueue.casSlotToNull(a, k, t)) {
                        q.base = nb;
                        w.source = src;
                        t.doExec();
                        w.source = wsrc;
                        rescan = true;
                        break scan;
                    }
                }
            }
        }
    }
}
```
这段代码的主要流程就是工作窃取从别的队列偷，它是FIFO模型，先进入的被先拿出来，和LIFO顺序正好相反，下面我将详细分析其执行流程。

首先写了一个for死循环，判断当前任务的状态是否小于0，如果小于说明任务执行完成直接返回状态码，判断ctl值是否发生变化并且判断rescan == false说明什么都没干成尝试执行
tryCompensate表示当前线程准备进入等待状态。

WorkQueue[] qs = queues; 扫描整个线程池的任务队列，循环判断是否有可以偷的任务，这段代码很重要int k = (cap - 1) & b, nb = b + 1;这样写表示从先进入的地方开始获取，这正是
FIFO模型的体现
```java
for (int d = n, v = sq;;) {
    WorkQueue p = qs[v & m];
    v = p.source;
}
```
这一小段代码主要是为了看任务t的创建线程是否间接创建了我要等待的task,只有在创建链中，任务t才是有助于完成我等待的任务的任务,否则就是无关任务，不帮。
```java
if (WorkQueue.casSlotToNull(a, k, t)) {
    q.base = nb;
    t.doExec();      // 执行任务
    rescan = true;   // 继续找下一个任务
}
```
这一小段代码主要是成功抢占后就调用doExec() 执行；然后rescan = true，表示继续重新扫描是否还有别的任务可以抢。

## 十一、 总结
&nbsp;&nbsp;ForkJoinPool源码非常复杂，我只是分析了其中的一部分重要的源码，但也足以可以了解其中的设计理念和精髓，理解工作窃取和任务拆分是重点，最好
结合我的示例进行手动调试一下比较好，不同版本的jdk代码差异比较大，大家可以结合自己实际情况看一下。