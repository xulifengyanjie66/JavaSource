# CompletableFuture源码分析

- [一、简介](#一简介)

- [二、初始化状态与成员变量](#二初始化状态与成员变量)

- [三、类继承关系](#三类继承关系)

- [四、设计精髓](#四设计精髓)

- [五、实际应用示例](#五实际应用示例)

- [六、例子方法源码分析](#六例子方法源码分析)

- [七、总结](#七总结)

## 一、简介
&nbsp;&nbsp;Java8 引入了CompletableFuture，大大增强了Future接口的能力，支持异步编排、链式调用和异常恢复, 它既是`Future`的实现类，也是 `CompletionStage`接口的实现,
本文将深入分析其源码结构，了解其核心机制、线程模型和异常处理等设计亮点,本文基于JDK版本：JDK21(源码中已引入VarHandle替代Unsafe)。

## 二、初始化状态与成员变量

- 初始化空状态，result == null 表示尚未完成。

**成员变量**
- volatile Object result：任务结果或异常封装体 AltResult
- stack：依赖任务链栈顶节点
- 使用 VarHandle 对这些字段进行原子操作

## 三、类继承关系
&nbsp;&nbsp; `CompletableFuture<V>` 继承自Object，实现了Future和CompletionStage两大接口,CompletionStage是Java8引入的新接口，代表一个可组合的、异步的计算阶段。
它是 CompletableFuture 的真正亮点，支持链式编程、回调编排、异常处理,这里我画一个表格说明一下它的主要方法。

| 类型        |                示例方法                 |     说明     |
|:----------|:---------------------------------:|:----------:|
| 转换类	      | thenApply, thenApplyAsync | 将结果转换为另一个值 |
| 消费类       |          thenAccept, thenAcceptAsync| 处理结果但无返回	  |
| 组合类 |           thenCombine, thenCompose, runAfterBoth|    组合两个 CompletionStage 	      |
| 异常处理类 |           exceptionally, handle, whenComplete|    异常处理机制	      |
| 控制类 |           toCompletableFuture()|    返回自身	      |

**CompletableFuture中的特点：**

- 每一个方法都创建一个内部回调节点(如 UniApply, BiApply)，挂载在 stack 上。

- 默认使用ForkJoinPool.commonPool() 执行异步方法，除非显式指定 Executor。

类的继承结构图如下:

![img_6.png](..%2Fimg%2Fimg_6.png)

## 四、设计精髓

&nbsp;&nbsp;CompletableFuture的设计是Java并发编程的重要里程碑，其精妙之处体现在多个方面：

**1. 分阶段异步编排模型**

CompletableFuture基于CompletionStage 接口，支持将复杂的异步逻辑拆分为“阶段”执行，并通过链式 API 表达清晰的任务依赖关系。其设计理念可总结为：

- 每个阶段完成后，才会触发下一个阶段
- 可以选择是否使用线程池执行下一个阶段（同步/异步）

这种模型使得开发者不再需要手动控制线程与状态，而是专注于数据流。

**2. Completion树与非阻塞机制**

- 内部用 `Completion` 链表（或树）结构保存后续任务
- 任务完成时，会自动触发这些回调逻辑，避免轮询
- 使用 `VarHandle` + `CAS` 实现高效、无锁的任务完成通知

这使得 `CompletableFuture` 可以被多个线程等待，又不会像传统 `Future` 那样阻塞。

**3.灵活的线程调度策略**

- 默认线程池为 `ForkJoinPool.commonPool`
- 也可以传入自定义线程池，更适合微服务场景
- `thenApply()` 为同步执行，`thenApplyAsync()` 为异步执行

这种设计让用户可以选择在“当前线程”还是“线程池”中执行下一个阶段，极大增强了灵活性。

**4. 对异常的天然支持**

与传统 try-catch 相比，CompletableFuture 支持链式异常捕获：

- `exceptionally`：统一兜底处理
- `handle`：可拿到异常或正常结果
- `whenComplete`：无论成功失败都执行回调

这是一种“响应式”的异常模型，非常适合现代并发系统。

**5. 高并发场景下的可组合性**

CompletableFuture 强调的是组合 —— 各种任务之间可以：

- `thenCombine` 合并两个结果
- `allOf` / `anyOf` 等待多个任务完成
- `thenCompose` 实现嵌套异步

这些都是构建大型异步系统的基础能力。

## 五、实际应用示例
```java
import java.util.concurrent.*;

public class CompletableFutureDemo {

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        CompletableFuture<Void> orderFlow = CompletableFuture
            .supplyAsync(() -> checkStock("iPhone15"), executor)
            .thenApply(stock -> createOrder("iPhone15", stock))
            .thenAccept(orderId -> sendSms(orderId))
            .exceptionally(ex -> {
                System.out.println("下单失败，原因：" + ex.getMessage());
                return null;
            });

        // 等待完成
        orderFlow.get();

        executor.shutdown();
    }

    // 模拟库存检查
    private static int checkStock(String product) {
        System.out.println("检查库存...");
        sleep(500);
        if ("iPhone15".equals(product)) {
            return 5; // 有 5 件
        } else {
            throw new RuntimeException("商品不存在");
        }
    }

    // 模拟创建订单
    private static String createOrder(String product, int stock) {
        System.out.println("创建订单...");
        sleep(300);
        return "ORDER-" + System.currentTimeMillis();
    }

    // 模拟发送短信
    private static void sendSms(String orderId) {
        System.out.println("发送短信通知，订单号：" + orderId);
    }

    private static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException ignored) {}
    }
}
```
这个示例模拟了商品检查库存、创建订单、发送短信通知的全过程，它首先创建了一个自定义固定线程池，线程池里面线程数量是3,然后调用supplyAsync异步去
检查库存，检查库存成功后调用链式thenApply方法去下单,下单成功后调用链式thenAccept方法发送短信，如果有异常统一处理异常信息，然后主线程调用get方法阻塞获取结果，最后关闭线程池,
下面我分析的源码主要按照这个主线进行,还会有其它的源码分析，搞清楚这个就很容易理解其它的源码。

## 六、例子方法源码分析

&nbsp;&nbsp;这块我结合上面给出的例子一步步分析源码，先从主线程的supplyAsync方法分析一步步串联分析，然后再分析线程池执行逻辑，supplyAsync方法的源码的签名如下:
```java
 static <U> CompletableFuture<U> asyncSupplyStage(Executor e,Supplier<U> f) {
        if (f == null) throw new NullPointerException();
        CompletableFuture<U> d = new CompletableFuture<U>();
        e.execute(new AsyncSupply<U>(d, f));
        return d;
}
```
该方法主要是创建一个CompletableFuture，然后把Supplier和CompletableFuture对象封装为一个AsyncSupply对象，用线程池执行AsyncSupply对象，
然后返回创建的CompletableFuture对象,接着我们先沿着主线程逻辑分析，稍后在分析线程池执行逻辑。

接着调用thenApply方法,该方法的签名如下:
```java
public <U> CompletableFuture<U> thenApply(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}
```
里面调用的是uniApplyStage方法，该方法的签名如下:
```java
private <V> CompletableFuture<V> uniApplyStage(
      Executor e, Function<? super T,? extends V> f) {
      if (f == null) throw new NullPointerException();
      Object r;
      if ((r = result) != null)
          return uniApplyNow(r, e, f);
      CompletableFuture<V> d = newIncompleteFuture();
      unipush(new UniApply<T,V>(e, d, this, f));
      return d;
  }
```
判断返回的CompletableFuture对象的result是否为空，如果不为空说明线程池已经执行完获取到结果了，它是用volatile修饰的，那么此时主线程可以看到，立即调用
uniApplyNow方法执行thenApply方法逻辑，uniApplyNow方法签名如下:
```java
private <V> CompletableFuture<V> uniApplyNow(
        Object r, Executor e, Function<? super T,? extends V> f) {
        Throwable x;
        CompletableFuture<V> d = newIncompleteFuture();
        if (r instanceof AltResult) {
            if ((x = ((AltResult)r).ex) != null) {
                d.result = encodeThrowable(x, r);
                return d;
            }
            r = null;
        }
        try {
            if (e != null) {
                e.execute(new UniApply<T,V>(null, d, this, f));
            } else {
                @SuppressWarnings("unchecked") T t = (T) r;
                d.result = d.encodeValue(f.apply(t));
            }
        } catch (Throwable ex) {
            d.result = encodeThrowable(ex);
        }
        return d;
}
```
创建一个新的CompletableFuture对象,判断上一个CompletableFuture对象结果是否是AltResult对象，关于AltResult对象是什么我在分析线程池时候具体说，
如果AltResult对象的ex不为空说明有异常发生，调用encodeThrowable处理异常，该方法的签名如下:
```java
static Object encodeThrowable(Throwable x, Object r) {
    if (!(x instanceof CompletionException))
        x = new CompletionException(x);
    else if (r instanceof AltResult && x == ((AltResult)r).ex)
        return r;
    return new AltResult(x);
}
```
判断Throwable是否是CompletionException类型，如果不是，把Throwable封装为CompletionException对象然后用AltResult包装返回，如果是
CompletionException类型，判断r类型是否是AltResult类型并且是AltResult的异常类型直接返回AltResult，否则也包装成AltResult类型返回，异常是其他类型。

此时返回新创建AltResult的对象，uniApplyNow方法执行完成。

如果没有发生异常并且r是AltResult对象，把r=null，代表上一个CompletableFuture对象计算结果是null, 然后会执行这一小段代码，T t = (T) r;d.result = d.encodeValue(f.apply(t));其中t是执行supplyAsync方法返回的结果，
f.apply就是执行我们的代码createOrder的逻辑，执行完成以后把结果传入到新创建的CompletableFuture对象的encodeValue方法中，该方法的签名如下：
```java
  final Object encodeValue(T t) {
      return (t == null) ? NIL : t;
  }
```
判断createOrder的执行结果是否是null,如果是返回NIL,它是一个AltResult对象，里面包装的是null值，如果不是null返回t的值，同样如果执行
createOrder过程中发生异常调用encodeThrowable处理方法，无论是否发生异常最终都返回新创建的CompletableFuture对象,到此为止uniApplyNow执行结束。

此时回到调用者方法判断如果reslut == null会执行这一小段代码，代码如下:
```java
  CompletableFuture<V> d = newIncompleteFuture();
  unipush(new UniApply<T,V>(e, d, this, f));
  return d;
```
此处还是新创建一个CompletableFuture对象，把它包装成为一个UniApply对象，UniApply是CompletableFuture的一个静态内部类，它表示一个包装了Function<T, V> 的任务，
它会在CompletableFuture<T> 计算完成后，把结果T应用到 Function<T, V> 上，得到新结果 V，并设置给下一个 CompletableFuture<V>。它的成员变量主要有Executor executor线程池对象、
CompletableFuture<V> dep当前新创建的CompletableFuture对象、CompletableFuture<T> src当前调用该方法的CompletableFuture对象，也就是上一步的CompletableFuture对象，它可能包含计算结果,
Function<? super T,? extends V> fn表示要计算的函数式对象。

接着调用unipush方法传入上面包装成的UniApply对象，该方法的签名如下:
```java
final void unipush(Completion c) {
      if (c != null) {
          while (!tryPushStack(c)) {
              if (result != null) {
                  NEXT.set(c, null);
                  break;
              }
          }
          if (result != null)
              c.tryFire(SYNC);
      }
  }
```
这个方法又调用了tryPushStack方法传入上面包装成的UniApply对象，该方法的签名如下：
```java
final boolean tryPushStack(Completion c) {
    Completion h = stack;
    NEXT.set(c, h);  
    return STACK.compareAndSet(this, h, c);
}
```
把原始对象的CompletableFuture的成员属性Stack赋值给变量h,然后把它设置给UniApply的Next成员属性，通过CAS原子操作把stack从h设置为传入的UniApply对象，如果设置成功返回true,否则返回false,
适用于多线程安全的操作，为啥要这么做那？ 因为希望把当前传入的UniApply对象设置到栈顶上能先执行。

此时返回到unipush方法继续执行，如果是CAS设置失败会在while循环里面执行判断之前执行结果如果不为null,设置Next为null，跳出while死循环;如果是CAS执行成功此时退出while循环，此时判断result如果不为null就
调用UniApply的tryFire方法执行Function<? super T,? extends V>逻辑，否则就什么都不做，等待结果执行完成也会自动调用UniApply的tryFire方法执行，这块的逻辑我到线程池异步执行那块在分析。

执行到这里thenApply方法执行完成它也返回新的CompletableFuture对象了，然后又链式调用thenAccept方法，这里面的逻辑基本和thenApply一样，不一样的地方就是封装的是UniAccept对象，它和上面说的
UniApply对象的执行逻辑不太一样等到分析那块时候再说。

沿着主线程的顺序继续执行此时执行到exceptionally方法，调用此方法的对象是thenAccept创建的新的CompletableFuture对象，这个方法是统一处理异常的方法，它的方法签名如下:
```java
public CompletableFuture<T> exceptionally(
        Function<Throwable, ? extends T> fn) {
        return uniExceptionallyStage(null, fn);
}
```
它又调用了uniExceptionallyStage方法，该方法的签名如下：
```java
private CompletableFuture<T> uniExceptionallyStage(
    Executor e, Function<Throwable, ? extends T> f) {
    if (f == null) throw new NullPointerException();
    CompletableFuture<T> d = newIncompleteFuture();
    Object r;
    if ((r = result) == null)
        unipush(new UniExceptionally<T>(e, d, this, f));
    else if (e == null)
        d.uniExceptionally(r, f, null);
    else {
        try {
            e.execute(new UniExceptionally<T>(null, d, this, f));
        } catch (Throwable ex) {
            d.result = encodeThrowable(ex);
        }
    }
    return d;
}
```
还是首先判断上一步CompletableFuture的计算结果result如果等于null,封装为UniExceptionally对象放入到CompletableFuture的属性stack栈头上。

判断如果是线程池e==null并且上一步计算结果resul的不为null,调用本方法新创建的CompletableFuture对象的uniExceptionally方法,该方法的签名如下:

```java
final boolean uniExceptionally(Object r,
        Function<? super Throwable, ? extends T> f,
        UniExceptionally<T> c) {
  Throwable x;
  if (result == null) {
      try {
          if (c != null && !c.claim())
              return false;
          if (r instanceof AltResult && (x = ((AltResult)r).ex) != null)
              completeValue(f.apply(x));
          else
              internalComplete(r);
      } catch (Throwable ex) {
          completeThrowable(ex);
      }
  }
  return true;
}
```
主要还是判断上一步是否有异常发生，如果有调用completeValue方法处理自己的逻辑，我的例子就是打印这句话`System.out.println("下单失败，原因：" + ex.getMessage());`
如果没有异常就设置正常返回的结果到result属性上。

exceptionally方法执行完成返回给main线程新创建的CompletableFuture对象，此时调用get方法进行阻塞等待线程池返回结果，该方法的签名如下：
```java
public T get() throws InterruptedException, ExecutionException {
      Object r;
      if ((r = result) == null)
          r = waitingGet(true);
      return (T) reportGet(r);
  }
```
判断如果result为null,说明线程池还没执行完成，此时就调用waitingGet方法阻塞等待，该方法的签名如下:
```java
private Object waitingGet(boolean interruptible) {
    if (interruptible && Thread.interrupted())
        return null;
    Signaller q = null;
    boolean queued = false;
    Object r;
    while ((r = result) == null) {
        if (q == null) {
            q = new Signaller(interruptible, 0L, 0L);
            if (Thread.currentThread() instanceof ForkJoinWorkerThread)
                ForkJoinPool.helpAsyncBlocker(defaultExecutor(), q);
        }
        else if (!queued)
            queued = tryPushStack(q);
        else if (interruptible && q.interrupted) {
            q.thread = null;
            cleanStack();
            return null;
        }
        else {
            try {
                ForkJoinPool.managedBlock(q);
            } catch (InterruptedException ie) { // currently cannot happen
                q.interrupted = true;
            }
        }
    }
    if (q != null) {
        q.thread = null;
        if (q.interrupted)
            Thread.currentThread().interrupt();
    }
    postComplete();
    return r;
}
```
传入interruptible为true表示可以被中断，然后判断线程是否被外部真的进行中断了，如果是该方法返回null,什么都不做，否则就继续等待，进入while死循环等待，前提是result是null,
初始化一个当前线程的阻塞器Signaller,它是是CompletableFuture内部自定义的阻塞对象，实现了ForkJoinPool.ManagedBlocker,用于在ForkJoinPool或普通线程中等待结果,如果支持中断并且线程被中断，就清除阻塞栈并返回null


```java
if (Thread.currentThread() instanceof ForkJoinWorkerThread)
    ForkJoinPool.helpAsyncBlocker(defaultExecutor(), q);
```
如果当前线程是ForkJoinPool的工作线程，就交给ForkJoinPool来帮忙管理阻塞,否则就把当前阻塞器入栈用于唤醒,通过 ForkJoinPool.managedBlock(q) 进行阻塞等待。如果抛出 InterruptedException，就设置 interrupted = true，下一轮循环会处理。

如果result不为null说明已经获取到了结果清理q.thread，避免内存泄露,如果中断过，重新设置当前线程的中断状态,调用postComplete() 做收尾处理,返回异步任务的结果。

到这里，主线程的执行逻辑就已经分析完成。主要流程是为各个阶段的 CompletableFuture 实例构建执行链，这些执行链以栈（stack）的形式挂载在每个 CompletableFuture 对象上。当某个阶段的计算结果完成时，会触发回调机制，依次调用其 stack 中的节点（如 UniApply, UniAccept, BiApply 等），将结果向后传递并继续计算。

这些回调的执行通常是通过异步线程池（如 ForkJoinPool.commonPool() 或自定义 Executor）来调度完成的。下面，我将重点分析异步线程池是如何调度这些任务的执行过程。

前面分析asyncSupplyStage方法时候我说过创建了一个AsyncSupply线程对象，它是异步执行的关键，下面我来重点分析这一块的源码。

**AsyncSupply类源码分析**

AsyncSupply继承自ForkJoinTask类同时实现了Runnable,用线程池时候会执行其run方法，要重点看一下它的实现，源码如下:
```java
public void run() {
    CompletableFuture<T> d; Supplier<? extends T> f;
    if ((d = dep) != null && (f = fn) != null) {
        dep = null; fn = null;
        if (d.result == null) {
            try {
                d.completeValue(f.get());
            } catch (Throwable ex) {
                d.completeThrowable(ex);
            }
        }
        d.postComplete();
    }
}
```
定义两个变量CompletableFuture<T> d; Supplier<? extends T> f;把成员变量CompletableFuture dep赋值给d,把成员变量Supplier fn赋值给f,把dep,fn指向null,可以使得
JVM先AsyncSupply没有时候能自动回收没有回收的对象同时也防止任务重复执行时引用未清空造成副作用,接着判断result是否为null,为null说明任务还没执行完调用CompletableFuture对象的completeValue方法，该
方法的签名如下:
```java
final boolean completeValue(T t) {
      return RESULT.compareAndSet(this, null, (t == null) ? NIL : t);
}
```
其中RESULT = l.findVarHandle(CompletableFuture.class, "result", Object.class);它是CompletableFuture的成员变量result类型是Object,可以利用它进行CAS
赋值操作，NIL是AltResult NIL = new AltResult(null),AltResult的成员变量Throwable ex,如果ex为空说明没有发生异常，所以这段代码表示如果CompletableFuture
的成员属性result如果为null的前提下，如果t==null，设置为NIT,否则就设置为Supplier的get的值，如果设置成功返回true,否则返回false。

接着调用CompletableFuture对象的postComplete方法,该方法的签名如下:

```java
final void postComplete() {
    CompletableFuture<?> f = this; Completion h;
    while ((h = f.stack) != null ||
           (f != this && (h = (f = this).stack) != null)) {
        CompletableFuture<?> d; Completion t;
        if (STACK.compareAndSet(f, h, t = h.next)) {
            if (t != null) {
                if (f != this) {
                    pushStack(h);
                    continue;
                }
                NEXT.compareAndSet(h, t, null);
            }
            f = (d = h.tryFire(NESTED)) == null ? this : d;
        }
    }
}
```
f是当前要处理的CompletableFuture 对象，h是它的stack(回调栈),有一个while的循环不断处理stack中的Completion，如果CAS原子成功设置栈顶节点为当前节点的下一个
节点就调用Completion对象的tryFire方法，如果CAS原子设置失败就行下一个while循环直到设置成功为止。

现在我分析第一个Completion的实现类的UniApply(前面已经分析过)的tryFire方法,该方法的签名如下:
```java
final CompletableFuture<V> tryFire(int mode) {
          CompletableFuture<V> d; CompletableFuture<T> a;
          Object r; Throwable x; Function<? super T,? extends V> f;
          if ((a = src) == null || (r = a.result) == null
              || (d = dep) == null || (f = fn) == null)
              return null;
          tryComplete: if (d.result == null) {
              if (r instanceof AltResult) {
                  if ((x = ((AltResult)r).ex) != null) {
                      d.completeThrowable(x, r);
                      break tryComplete;
                  }
                  r = null;
              }
              try {
                  if (mode <= 0 && !claim())
                      return null;
                  else {
                      @SuppressWarnings("unchecked") T t = (T) r;
                      d.completeValue(f.apply(t));
                  }
              } catch (Throwable ex) {
                  d.completeThrowable(ex);
              }
          }
          src = null; dep = null; fn = null;
          return d.postFire(a, mode);
      }
}
```
把成员属性src赋值给变量a，sr成员属性是上一步产生的CompletableFuture对象，把成员属性dep赋值给变量d，dep成员属性是在调用thenApply方法时候生成的CompletableFuture对象，这里的
dep要依赖src产生的结果，r是src的计算结果值，判断r是否是AltReslut，如果是可能代表上一步发生了异常或者返回结果为null,如果发生异常调用dep的completeThrowable方法设置异常信息，否则把结果设置为null,
接着调用f.apply(t)设置dep的值,f在本例中是stock -> createOrder("iPhone15", stock),t是src的产生值。如果执行发生异常还是调用dep的completeThrowable方法设置异常信息。

接着把src、dep、fn的引用指向null,方便gc收集器收集上一步产生的无用对象，此时调用新的CompletableFuture对象得postFire,该方法的签名如下:
```java
final CompletableFuture<T> postFire(CompletableFuture<?> a, int mode) {
      if (a != null && a.stack != null) {
          Object r;
          if ((r = a.result) == null)
              a.cleanStack();
          if (mode >= 0 && (r != null || a.result != null))
              a.postComplete();
      }
      if (result != null && stack != null) {
          if (mode < 0)
              return this;
          else
              postComplete();
      }
      return null;
  }
```
该方法的主要逻辑就是返回调用此方法的CompletableFuture对象。

此时执行逻辑又返回到原始第一个CompletableFuture对象的postComplete方法执行下一次while循环,此时这一小段代码条件成立`f != this && (h = (f = this).stack) != null)`
此时h变量是新的CompletableFuture得Completion成员属性按照我的例子来说就是UniAccept对象，它的tryFire方法如下:
```java
final CompletableFuture<Void> tryFire(int mode) {
    CompletableFuture<Void> d; CompletableFuture<T> a;
    Object r; Throwable x; Consumer<? super T> f;
    if ((a = src) == null || (r = a.result) == null
        || (d = dep) == null || (f = fn) == null)
        return null;
    tryComplete: if (d.result == null) {
        if (r instanceof AltResult) {
            if ((x = ((AltResult)r).ex) != null) {
                d.completeThrowable(x, r);
                break tryComplete;
            }
            r = null;
        }
        try {
            if (mode <= 0 && !claim())
                return null;
            else {
                @SuppressWarnings("unchecked") T t = (T) r;
                f.accept(t);
                d.completeNull();
            }
        } catch (Throwable ex) {
            d.completeThrowable(ex);
        }
    }
    src = null; dep = null; fn = null;
    return d.postFire(a, mode);
}
```
可以看出它只消费上一步产生的值并且不返回值，可以从d.completeNull()方法看出，它设置结果是AltResult对象。

如此再继续执行会执行到UniExceptionally(前面分析过),它主要是统一处理异常的逻辑，这块之前已经说过了这里就不在详细说明了。

好了，如果执行完调用exceptionally方法的CompletableFuture对象以后接着调用d.postFire时候会调用Signaller对象的tryFire方法，为什么？还记得前面分析过主线程执行
CompletableFuture的get方法时候会往stack里面添加Signaller对象把，它会判断如果当前CompletableFuture对象的result为空会阻塞等待。那现在异步线程池已经执行完了就会唤醒
主线程的阻塞，那我们看看到底是怎么唤醒的，Signaller对象的tryFire方法签名如下：
```java
final CompletableFuture<?> tryFire(int ignore) {
      Thread w;
      if ((w = thread) != null) {
          thread = null;
          LockSupport.unpark(w);
      }
      return null;
  }
```
很简单把，就是调用阻塞线程的unpark方法唤醒线程使得主线程可以继续处理逻辑。

**CompletableFuture的allOf源码分析**

&nbsp;&nbsp;allOf方法用于等待多个异步任务全部完成后再继续执行，适用于“并发任务汇聚”的典型场景,它返回一个新的CompletableFuture对象,下面我给出一个简单示例介绍一下
allOf方面怎么用，然后依据这个例子做源码分析，例子的代码如下:
```java
 public static void main(String[] args) throws Exception {

      // 模拟任务1：1秒后完成
      CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
      sleep(1);
      System.out.println("任务1完成");
      return "结果1";
      });

      // 模拟任务2：2秒后完成
      CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
      sleep(2);
      System.out.println("任务2完成");
      return "结果2";
      });

      // 模拟任务3：3秒后完成
      CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
      sleep(3);
      System.out.println("任务3完成");
      return "结果3";
      });
    // 使用 allOf 等待全部任务完成
    CompletableFuture<Void> allOf = CompletableFuture.allOf(future1, future2, future3);
    // allOf 本身不携带结果，可以在 thenRun 中统一处理逻辑
    allOf.thenRun(() -> {
    try {
      // 取出各个任务的返回值
      String result1 = future1.get();
      String result2 = future2.get();
      String result3 = future3.get();
      System.out.println("所有任务完成，结果为：");
      System.out.println(result1 + ", " + result2 + ", " + result3);
    } catch (Exception e) {
      e.printStackTrace();
    }
    });
    // 阻塞主线程，避免主线程提前退出
    allOf.get();
}
```
allOf的方法签名如下:
```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {
      return andTree(cfs, 0, cfs.length - 1);
}
```
可以看出它又调用了andTree方法，该方法的签名如下:
```java
static CompletableFuture<Void> andTree(CompletableFuture<?>[] cfs,int lo, int hi) {
    CompletableFuture<Void> d = new CompletableFuture<Void>();
    if (lo > hi) 
        d.result = NIL;
    else {
        CompletableFuture<?> a, b; Object r, s, z; Throwable x;
        int mid = (lo + hi) >>> 1;
        if ((a = (lo == mid ? cfs[lo] :
                  andTree(cfs, lo, mid))) == null ||
            (b = (lo == hi ? a : (hi == mid+1) ? cfs[hi] :
                  andTree(cfs, mid+1, hi))) == null)
            throw new NullPointerException();
        if ((r = a.result) == null || (s = b.result) == null)
            a.bipush(b, new BiRelay<>(d, a, b));
        else if ((r instanceof AltResult
                  && (x = ((AltResult)(z = r)).ex) != null) ||
                 (s instanceof AltResult
                  && (x = ((AltResult)(z = s)).ex) != null))
            d.result = encodeThrowable(x, z);
        else
            d.result = NIL;
    }
    return d;
}
```
cfs变量代表多个待组合的CompletableFuture数组,本例子是future1、future2、future3。lo, hi当前递归处理的索引区间 [lo, hi],第一次执行时候lo=0,hi=cfs.length-1即是2,
新创建一个CompletableFuture对象，如果lo > hi表示如果传入的是空区间，则说明没有任务需要组合，直接标记d为已完成（使用内部常量 NIL表示）。

开始使用递归构造左右子树将数组分成左右两部分(mid为中点),递归调用andTree构建左子树a和右子树b,通过这种方式，把cfs数组的多个CompletableFuture 按照二叉树组合成多个 "小组合任务"，最终组合成一个总的。递归的结束条件是lo==mid或者
hi == mid+1。

第一次执行mid=1，lo=0,lo != mid,继续递归调用andTree(第二次)方法,此时lo=0,hi=1,mid=0,此时lo == mid,得到a=cfs[0],b=cfs[1],判断a的result或者b的result是否有一个为null,
如果有封装为BiRelay对象,传入新创建的CompletableFuture对象、a、b。然后调用a.bipush方法传入BiRelay对象，该方法的签名如下：
```java
final void bipush(CompletableFuture<?> b, BiCompletion<?,?,?> c) {
    if (c != null) {
        while (result == null) {
            if (tryPushStack(c)) {
                if (b.result == null)
                    b.unipush(new CoCompletion(c));
                else if (result != null)
                    c.tryFire(SYNC);
                return;
            }
        }
        b.unipush(c);
    }
}
```
- 判断a对象的result如果等于null,把c压入到a对象的stack栈上,如果b对象的result也为null,把c也压入到b对象的stack栈上，否则如果当前对象result不为null就执行c的tryFire方法

- 判断a对象的result如果不等于null,直接把c压入到b对象的stack栈上。

此时第二次andTree方法执行完成返回到第一次andTree的执行处。会执行这段逻辑`(b = (lo == hi ? a : (hi == mid+1) ? cfs[hi] :andTree(cfs, mid+1, hi))) == null)`
此时hi == mid+1 那么b=cfs[2],a等于第二次andTree返回的CompletableFuture对象，此时又去执行bipush方法如此反复执行。

画个简图表示一下这个过程:

![img_7.png](..%2Fimg%2Fimg_7.png)

下一步就是分析一下BiRelay的tryFire执行逻辑,该方法的签名如下:
```java
final CompletableFuture<Void> tryFire(int mode) {
            CompletableFuture<Void> d;
    CompletableFuture<T> a;
    CompletableFuture<U> b;
    Object r, s, z; Throwable x;
    if (   (a = src) == null || (r = a.result) == null
        || (b = snd) == null || (s = b.result) == null
        || (d = dep) == null)
        return null;
    if (d.result == null) {
        if ((r instanceof AltResult
             && (x = ((AltResult)(z = r)).ex) != null) ||
            (s instanceof AltResult
             && (x = ((AltResult)(z = s)).ex) != null))
            d.completeThrowable(x, z);
        else
            d.completeNull();
    }
    src = null; snd = null; dep = null;
    return d.postFire(a, b, mode);
}
```
a,b是要依赖的左右子树对象，它们类型都是CompletableFuture类型，判断只要有一个未完成就返回null,只有都完成才能设置CompletableFuture的结果为完成状态,完成以后还是调用
CompletableFuture的postFire方法，依次去给要依赖的对象result对象赋值。

## 七、总结

&nbsp;&nbsp;本文围绕CompletableFuture的核心方法如supplyAsync、thenApply、thenAccept、exceptionally、allOf、get 展开了系统源码分析。通过对源码的逐层解构，可以发现它们背后都依赖于统一的基础机制：将逻辑封装为Completion对象，并通过无锁的栈式结构 (stack) 实现异步回调的注册与调度。

可以说，Completion是理解整 CompletableFuture 执行链的“骨架”，掌握了它的构建与触发机制，几乎等于掌握了 CompletableFuture 的精髓。

尽管CompletableFuture提供了大量的方法看似繁多，但其内部逻辑具有高度统一性。无论是组合多个异步任务（如 allOf）、捕获异常（如 exceptionally），还是串行转换（如 thenApply），本质上都是在构建依赖图，然后通过 tryFire 方法驱动执行链逐步推进。

最后，通过对这些核心路径的掌握，不仅可以轻松理解其它未分析的方法逻辑，还能够建立对异步流式模型更深层次的认知，比如：线程调度、安全性保障、任务取消的传播路径等。
