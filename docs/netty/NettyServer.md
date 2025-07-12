# Netty主要组件和服务器启动源码分析
- [1. Netty 服务端启动代码](#1-netty服务端启动代码)
- [2. 核心组件结构解析和继承结构图](#22-nioeventloopgroup继承结构图)
    - [2.1 NioEventLoopGroup是什么](#21-nioeventloopgroup是什么)
    - [2.2 NioEventLoopGroup](#22-nioeventloopgroup继承结构图)
    - [2.3 NioEventLoop是什么](#23-nioeventloop是什么)
    - [2.4 NioEventLoop继承结构图](#24-nioeventloop继承结构图)
    - [2.5 NioServerSocketChannel是什么](#25-nioserversocketchannel是什么)
    - [2.6 ChannelPipeline是什么](#26-channelpipeline是什么)
    - [2.7 Unsafe是什么](#27-unsafe是什么)
    - [2.8 ByteBuf是什么](#28-bytebuf是什么)
    - [2.9 Future Promise是什么](#29-future-promise是什么)
- [3. NioEventLoopGroup构造函数源码分析](#3nioeventloopgroup构造函数源码分析)
- [4.NioServerSocketChannel构造函数源码分析](#4nioserversocketchannel构造函数源码分析)
- [5.DefaultChannelPipeline构造函数源码分析](#5defaultchannelpipeline构造函数源码分析)
- [6.NioEventLoop构造函数源码分析](#6nioeventloop构造函数源码分析)
- [7.ServerBootstrap源码分析](#7serverbootstrap源码分析)
- [8.总结](#8总结)

## 1. Netty服务端启动代码
```java
public class NettyServer {
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1); // 只处理 accept
        EventLoopGroup workerGroup = new NioEventLoopGroup(); // 处理读写

        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                     .channel(NioServerSocketChannel.class)
                     .childHandler(new ChannelInitializer<SocketChannel>() {
                         @Override
                         public void initChannel(SocketChannel ch) {
                             ch.pipeline().addLast(new SimpleServerHandler());
                         }
                     });

            ChannelFuture future = bootstrap.bind(8080).sync();
            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```
## 2. 核心组件结构解析和继承结构图
### 2.1 NioEventLoopGroup是什么

&nbsp;&nbsp;NioEventLoopGroup是Netty的线程组抽象，本质是一个线程池，管理多个NioEventLoop。

- bossGroup:用于接收连接（注册 NioServerSocketChannel）

- workerGroup:处理读写请求（绑定 NioSocketChannel）

每个 NioEventLoop 是一个线程，绑定多个 channel，负责事件循环。

### 2.2 NioEventLoopGroup继承结构图


![img_8.png](..%2Fimg%2Fimg_8.png)

从继承类图上看，NioEventLoopGroup通过继承EventExecutorGroup实现了Iterable<EventExecutor> 和 ScheduledExecutorService 接口，因此它是一个可迭代的事件执行器组，并支持定时任务和延迟任务调度功能，用于事件循环线程模型中。

### 2.3 NioEventLoop是什么

- 实现了 Runnable,每个线程在run()中使用Selector.select() 轮询事件。

- 内部维护一个任务队列(runAllTasks() 执行)

职责：

  - 监听 selector 的事件（OP_ACCEPT, OP_READ 等）

  - 执行 IO 操作（读、写）

  - 执行任务队列（如异步 execute() 的任务）

### 2.4 NioEventLoop继承结构图

![img_9.png](..%2Fimg%2Fimg_9.png)

这个类本质上就是一个线程,它实现各个类主要有以下这些

**1.AbstractEventExecutor**

- 是EventExecutor的基础实现

- 提供默认的 schedule()、next() 等通用方法

- 支持异步执行和定时任务接口

**2. SingleThreadEventExecutor**

- 真正管理线程执行循环的地方。

- 封装了Thread、BlockingQueue<Runnable> 等核心成员。

- 实现了任务队列、线程启动、线程终止等控制逻辑。

- 提供核心的runAllTasks()、execute()、shutdown()等方法。

**3.SingleThreadEventLoop**

- 是EventLoop的基础实现，添加了和Channel绑定的逻辑。

- 每个EventLoop绑定多个 Channel，控制生命周期。

- 提供register(Channel) 方法。

**4.NioEventLoop**

- 最终实现类，专门用于 Java NIO 的事件处理（使用 Selector）

- 线程循环中调用 selector.select() 检查就绪事件（如 OP_ACCEPT、OP_READ）

- 处理IO操作，并执行 runAllTasks() 执行任务队列

### 2.5 NioServerSocketChannel是什么

&nbsp;&nbsp;NioServerSocketChannel是Netty提供的服务端Channel类型,主要用于在服务器端监听客户端连接请求的通道,本质上是对java.nio.channels.ServerSocketChannel的封装。

### 2.6 ChannelPipeline是什么

&nbsp;&nbsp;ChannelPipeline 是 Netty 中的核心组件之一，可以理解为：

- 一个责任链模式(链式处理器结构)，用来管理和调度所有的ChannelHandler，从而对Channel中的数据进行编码、解码、业务处理等操作。

**它的核心职责主要是:**

- 管理Handler维护一个双向链表结构的多个ChannelHandlerContext。
- 分发事件把Netty I/O 事件(读、写、连接等）分发给对应的 ChannelHandler。
- 支持链式操作addLast, addFirst, remove, replace等动态操作。
- 分离 I/O 和业务	把编码解码逻辑、业务逻辑分散在不同 Handler 中，解耦合

**数据进出流程**

**入站（Inbound）事件传播：**
比如：收到数据，触发 channelRead():调用的顺序是HeadContext → Decoder → BusinessHandler → TailContext,每个Handler都可以调用`ctx.fireChannelRead(msg)`把事件传播下去。

**出站（Outbound）事件传播：**
比如：调用 ctx.write() 或 channel.write()：调用顺序是TailContext → Encoder → BusinessHandler → HeadContext,写数据一般从尾部逆向传播，直到HeadContext调用底层写操作。

### 2.7 Unsafe是什么

Channel.Unsafe是Netty提供的底层通道操作接口，封装了 I/O 原语、Selector 注册、读写、关闭等敏感操作。它是Channel的一个内部接口，只有 Netty自己框架内部使用，用户不应该直接调用。

它的主要作用有这些，我画个表格说明:

| 方法                     |         功能说明          | 
  |:-----------------------|:---------------------:|
| register()	 | 向 Selector 注册 Channel |
| bind()        |         绑定端口	         |  
| connect() |      客户端连接远程地址	       | 
| beginRead()                     |         注册读事件         |
| write() / flush()                    |        执行底层写操作        |
| close()                    |         关闭连接          |

### 2.8 ByteBuf是什么

所有数据收发都基于ByteBuf，而非传统的Java NIO ByteBuffer，下面给出一个表格说明一下:

| 类名                |          描述           |
|:------------------|:---------------------:|
| ByteBuf	       | Netty 自定义缓冲区，替代 ByteBuffer|
| Unpooled           |         创建非池化的 ByteBuf	         |  
| PooledByteBufAllocator         |      池化内存分配器（性能更优）       |

### 2.9 Future Promise是什么

Netty 所有操作都是异步的，比如 bind(), write() 都返回 ChannelFuture,下面给出一个表格说明一下:

| 类名                |          描述           |
|:------------------|:---------------------:|
| ChannelFuture	       | 异步操作的结果，比如 bind、connect|
| ChannelPromise           |         ChannelFuture 的子类，允许你设置结果（成功/失败）         |  
| GenericFutureListener         |      回调监听器，可以添加在 Future 上异步通知完成       |


## 3.NioEventLoopGroup构造函数源码分析

`NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);`

这里面传递1代表boss线程数量只有1个，它调用了父类的构造函数而且调用的很深，这里主要分析主要的

```java
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,
                           final SelectStrategyFactory selectStrategyFactory) {
      super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
  }
```
走到这里开始调用父类的构造函数，主要参数nThreads代表线程数量本例中是1,如果不传线程数量是当前电脑的核心线程数*2,executor代表线程池对象，这里是null,SelectorProvider是JavaNIO中的一个类，用于提供底层 Selector、Channel、Pipe 等组件的工厂,它是跨平台的，
RejectedExecutionHandler是线程池拒绝对象，如果队列已满拒绝始终返回RejectedExecutionException异常。

接着调用父类的构造方法，主要代码如下:
```java
 protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
      super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
  }
```
把上面的selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject()变成了可变参数args,继续调用父类的构造方法，代码如下:
```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
    this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}
```
这里面新增一个DefaultEventExecutorChooserFactory对象，它是Netty中的“线程选择器工厂”，创建不同的线程选择策略对象（Chooser）用于负载均衡。接着调用父类的构造函数，代码如下；
```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,EventExecutorChooserFactory chooserFactory, Object... args) {
        checkPositive(nThreads, "nThreads");

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        chooser = chooserFactory.newChooser(children);

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```
如果没有指定线程池，默认使用一个线程工厂为每个任务创建一个线程。根据传入的线程数量构造EventExecutor数组,EventExecutor的实现是NioEventLoop,每个
children[i] 就是一个独立的线程/执行器,newChild代码如下:
```java
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
}
```
主要是把args可变参数拆分传递到NioEventLoop的构造函数中。

接下来创建线程选择器Chooser `chooser = chooserFactory.newChooser(children);`

- 使用DefaultEventExecutorChooserFactory根据线程数是否为2的幂来选择：

 - PowerOfTwoEventExecutorChooser：位运算轮询

 - GenericEventExecutorChooser：模运算轮询

用途：用于next()方法中负载均衡选择一个线程。

接下来设置终止监听器 terminationListener `final FutureListener<Object> terminationListener = ...`

- 用于监听所有 EventExecutor终止的回调（优雅关闭时使用）

- 每个子线程关闭后执行 terminatedChildren.incrementAndGet()

最后构建只读的children集合,提供外部只读视图，防止对内部数组children[]的修改。

## 4.NioServerSocketChannel构造函数源码分析

NioServerSocketChannel的构造函数源码如下:

```java
 public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```
newSocket获取原生java.nio的ServerSocketChannel，然后调用重载构造函数,它的源码如下:
```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```
SelectionKey.OP_ACCEPT的值是16,代表它是接收事件,继续调用父类的构造函数,它的源码如下:
```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();
    this.readInterestOp = readInterestOp;
    try {
      ch.configureBlocking(false);
    } catch (IOException e) {
    try {
      ch.close();
    } catch (IOException e2) {
      logger.warn(
      "Failed to close a partially initialized socket.", e2);
    }
}
```
由此可见又给成员变量unsafe赋值一个NioMessageUnsafe类型的对象，又给成员变量pipeline赋值一个DefaultChannelPipeline类型对象，设置readInterestOp的值等于16，并把
原生的ServerSocketChannel对象设置为非阻塞。

## 5.DefaultChannelPipeline构造函数源码分析
在上面说过创建一个NioServerSocketChannel对象时候会初始化一个ChannelPipeline对象，newChannelPipeline创建一个DefaultChannelPipeline并把NioServerSocketChannel对象传入
```java
  protected DefaultChannelPipeline(Channel channel){
        this.channel=ObjectUtil.checkNotNull(channel,"channel");
        succeededFuture=new SucceededChannelFuture(channel,null);
        voidPromise=new VoidChannelPromise(channel,true);
        
        tail=new TailContext(this);
        head=new HeadContext(this);
        
        head.next=tail;
        tail.prev=head;
}
```
之后创建了TailContext和HeadContext对象，都把当前的DefaultChannelPipeline对象传入到自己的构造方法中。TailContext、HeadContext都继承了AbstractChannelHandlerContext实现ChannelHandlerContext接口
，它是一个双向链表结构有next,prev属性。

## 6.NioEventLoop构造函数源码分析
```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
                 EventLoopTaskQueueFactory queueFactory) {
  super(parent, executor, false, newTaskQueue(queueFactory), newTaskQueue(queueFactory),
          rejectedExecutionHandler);
  this.provider = ObjectUtil.checkNotNull(selectorProvider, "selectorProvider");
  this.selectStrategy = ObjectUtil.checkNotNull(strategy, "selectStrategy");
  final SelectorTuple selectorTuple = openSelector();
  this.selector = selectorTuple.selector;
  this.unwrappedSelector = selectorTuple.unwrappedSelector;
}
```
首先调用父类构造方法SingleThreadEventLoop,父类的构造函数源码简化如下:
```java
 protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                      boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                      RejectedExecutionHandler rejectedHandler) {
      super(parent);
      this.addTaskWakesUp = addTaskWakesUp;
      this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
      this.executor = ThreadExecutorMap.apply(executor, this);
      this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
      this.rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
      this.provider = selectorProvider;
      this.selectStrategy = strategy;
      final SelectorTuple selectorTuple = openSelector();
      this.selector = selectorTuple.selector;
      this.unwrappedSelector = selectorTuple.unwrappedSelector;
  }
```
这个做了以下几件事：

- 设置线程执行器executor(通常是ThreadPerTaskExecutor)。
- 初始化队列taskQueue：处理普通任务(I/O、业务任务)。
- 设置线程拒绝策略。
- 保存SelectorProvider：Java NIO 提供的Selector工厂（默认是 SelectorProvider.provider())。
- SelectStrategy：定义select的策略，比如阻塞、非阻塞、忙轮询（Netty提供默认策略)。
- 初始化Java NIO的Selector。

## 7.ServerBootstrap源码分析

按照我前面的例子创建了一个ServerBootstrap对象，调用group方法设置成员变量bossGroup、workerGroup，分别代表接收线程和工作线程，接着调用
channel方法，传入一个NioServerSocketChannel的Class类,该方法源码如下:
```java
public B channel(Class<? extends C> channelClass) {
      return channelFactory(new ReflectiveChannelFactory<C>(
              ObjectUtil.checkNotNull(channelClass, "channelClass")
      ));
  }
```
创建一个反射工厂类ReflectiveChannelFactory用于在后面反射创建NioServerSocketChannel对象，同样把这个反射工厂类对象ReflectiveChannelFactory设置
到ServerBootstrap成员属性上,创建一个匿名的ChannelHandler对象设置到ServerBootstrap成员属性上。

接着调用ServerBootstrap的bind方法传入端口号8888,最终调用到doBind方法,该方法的源码如下:
```java
private ChannelFuture doBind(final SocketAddress localAddress) {
  final ChannelFuture regFuture = initAndRegister();
  final Channel channel = regFuture.channel();
  if (regFuture.cause() != null) {
      return regFuture;
  }

  if (regFuture.isDone()) {
      ChannelPromise promise = channel.newPromise();
      doBind0(regFuture, channel, localAddress, promise);
      return promise;
  } else {
      final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
      regFuture.addListener(new ChannelFutureListener() {
          @Override
          public void operationComplete(ChannelFuture future) throws Exception {
              Throwable cause = future.cause();
              if (cause != null) {
                  promise.setFailure(cause);
              } else {
                  promise.registered();

                  doBind0(regFuture, channel, localAddress, promise);
              }
          }
      });
      return promise;
  }
}
```
首先调用了initAndRegister方法用于注册，该方法的源码如下:
```java
final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                channel.unsafe().closeForcibly();
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }
        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        return regFuture;
}
```
主要是通过反射工厂ReflectiveChannelFactory反射创建NioServerSocketChannel对象，然后调用init方法传入刚创建的NioServerSocketChannel对象，
init方法源码如下:
```java
void init(Channel channel) {
      setChannelOptions(channel, newOptionsArray(), logger);
      setAttributes(channel, newAttributesArray());

      ChannelPipeline p = channel.pipeline();

      final EventLoopGroup currentChildGroup = childGroup;
      final ChannelHandler currentChildHandler = childHandler;
      final Entry<ChannelOption<?>, Object>[] currentChildOptions = newOptionsArray(childOptions);
      final Entry<AttributeKey<?>, Object>[] currentChildAttrs = newAttributesArray(childAttrs);

      p.addLast(new ChannelInitializer<Channel>() {
          @Override
          public void initChannel(final Channel ch) {
              final ChannelPipeline pipeline = ch.pipeline();
              ChannelHandler handler = config.handler();
              if (handler != null) {
                  pipeline.addLast(handler);
              }
              ch.eventLoop().execute(new Runnable() {
                  @Override
                  public void run() {
                      pipeline.addLast(new ServerBootstrapAcceptor(
                              ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                  }
              });
          }
      });
}
```
调用NioServerSocketChannel的pipeline方法获取ChannelPipeline对象，然后调用addLast方法添加一个匿名的ChannelHandler对象。

执行到这里init方法执行完成返回到initAndRegister方法继续往下执行`ChannelFuture regFuture = config().group().register(channel)`
获取到config().group()即代码中的bossGroup接收线程池组调用其register方法,该方法源码如下:
```java
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```
经过一路的调用其父类的方法，最终执行这段代码
```java
public EventExecutor next() {
      return chooser.next();
}
```
chooser的实现类是PowerOfTwoEventExecutorChooser,上面说过，调用它的next方法获取一个EventExecutor对象，看看它是怎么取得EventExecutor对象的，
```java
public EventExecutor next() {
    return executors[idx.getAndIncrement() & executors.length - 1];
}
```
采用位运算轮询从数组中获取EventExecutor对象，这个算法的前提是只有当executors.length是2的幂时才能使用这种位运算，前面已经说过。

接着调用register方法，该方法的源码如下:
```java
 public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}
```
创建一个DefaultChannelPromise对象，该对象包装了NioServerSocketChannel对象和线程池对象NioEventLoop(上面返回的EventExecutor)，DefaultChannelPromise对象用于异步通知Channel操作的结果（成功 / 失败）,支持
添加回调函数，接着调用重载的register方法，该方法的源码如下:
```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    ObjectUtil.checkNotNull(eventLoop, "eventLoop");
    if (isRegistered()) {
        promise.setFailure(new IllegalStateException("registered to an event loop already"));
        return;
    }
    if (!isCompatible(eventLoop)) {
        promise.setFailure(
                new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
        return;
    }

    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) {
        register0(promise);
    } else {
        try {
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```
把获取到的EventLoop对象绑定到当前的NioServerSocketChannel上代码判断是`AbstractChannel.this.eventLoop = eventLoop`
此时会调用到eventLoop.execute方法执行任务即register0方法，该方法的源码如下:
```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;
        pipeline.invokeHandlerAddedIfNeeded();
        safeSetSuccess(promise);
        pipeline.fireChannelRegistered();
        if (isActive()) {
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```
这里面首先调用了doRegister方法,该方法的源码如下:
```java
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```
这里可以看出调用了java.nio里面的方法进行注册。

返回到register0方法继续执行代码片段`pipeline.invokeHandlerAddedIfNeeded()`一路跟踪调用栈会执行到上面的init方法的添加的匿名类这块，
```java
 p.addLast(new ChannelInitializer<Channel>() {
          @Override
          public void initChannel(final Channel ch) {
              final ChannelPipeline pipeline = ch.pipeline();
              ChannelHandler handler = config.handler();
              if (handler != null) {
                  pipeline.addLast(handler);
              }
              ch.eventLoop().execute(new Runnable() {
                  @Override
                  public void run() {
                      pipeline.addLast(new ServerBootstrapAcceptor(
                              ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                  }
              });
          }
 });
```
看是否设置了ServerBootstrap的ChannelHandler,如果设置了添加到NioServerSocketChannel的pipeline上，接着又添加了一个ServerBootstrapAcceptor类型的
ChannelHandler，该类的构造函数主要是NioServerSocketChannel类、NioEventLoopGroup(工作线程组)、ChannelHandler(处理连接请求的Handler)。

设置promise的状态为执行成功然后调用fireChannelRegistered方法通知各个ChannelHandler的channelRegistered方法，关于这个方法执行流程也是分析完
这个整体流程再作分析,接着判断是否是isActive状态并且是已经注册如果是调用pipeline的fireChannelActive方法表示前Channel已经处于active（激活）状态，也就是说建立成功等待
读取数据了,但是此时是注册阶段isActive是判断端口是否绑定了，绑定逻辑还没分析所以这块根本不会执行。

好了，到现在为止register执行完成返回到initAndRegister方法然后返回到doBind方法继续执行，此时执行的代码逻辑是:
```java
final ChannelFuture regFuture = initAndRegister();
final Channel channel = regFuture.channel();
  if (regFuture.cause() != null) {
      return regFuture;
  }
if (regFuture.isDone()) {
    ChannelPromise promise = channel.newPromise();
    doBind0(regFuture, channel, localAddress, promise);
    return promise;
} else {
    final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
    regFuture.addListener(new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            Throwable cause = future.cause();
            if (cause != null) {
                promise.setFailure(cause);
            } else {
                promise.registered();

                doBind0(regFuture, channel, localAddress, promise);
            }
        }
    });
    return promise;
}
```
返回的ChannelFuture对象是Promise对象，代表的异步对象，这是主线程调用Promise对象的isDone方法判断是否注册完成，如果完成调用doBind0方法进行绑定
如果没完成添加一个Listener对象，如果异步执行完成可以回调这个对象，然后也是调用doBind0方法，下面我们看看这个方法的执行逻辑：
```java
private static void doBind0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress localAddress, final ChannelPromise promise) {
    
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```
最终会调用到底层绑定方法bind,该方法的源码如下:
```java
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }
    if (Boolean.TRUE.equals(config().getOption(ChannelOption.SO_BROADCAST)) &&
        localAddress instanceof InetSocketAddress &&
        !((InetSocketAddress) localAddress).getAddress().isAnyLocalAddress() &&
        !PlatformDependent.isWindows() && !PlatformDependent.maybeSuperUser()) {
        logger.warn(
                "A non-root user can't receive a broadcast packet if the socket " +
                "is not bound to a wildcard address; binding to a non-wildcard " +
                "address (" + localAddress + ") anyway as requested.");
    }
    boolean wasActive = isActive();
    try {
        doBind(localAddress);
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        closeIfClosed();
        return;
    }
  
    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }
  
    safeSetSuccess(promise);
}
```
其中最终调用的核心方法是doBind,这里是调用java.nio里面的进行绑定端口
```java
protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

## 8.总结
本文系统性地介绍了Netty框架中各个核心组件的作用与协作机制，深入剖析了诸如 Channel、EventLoop、ChannelPipeline、ChannelHandler、ByteBuf、Future/Promise 等关键类的职责与内部设计。同时，也详细分析了 Netty 中一些重要类的构造函数实现，解释了它们在框架初始化与运行过程中的关键角色。

除此之外，本文还对Netty服务器端的启动源码流程进行了深入讲解，涵盖了从ServerBootstrap初始化配置、线程模型建立，到NioServerSocketChannel注册到Selector，再到服务端端口绑定的完整流程。启动过程被清晰地划分为两个关键阶段：

  - 注册阶段：将服务端Channel注册到Selector 上，建立事件监听机制；

  - 绑定阶段：将 Channel 绑定到指定端口，并触发 channelActive 等生命周期事件，为接收客户端连接做好准备。

通过源码层面的追踪与分析，为进一步掌握Netty的使用与优化打下了坚实基础，下篇文章我将分析请求过程的源码，欢迎大家继续关注。