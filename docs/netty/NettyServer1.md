# Netty服务端请求处理源码
- [1. Netty线程启动源码](#1-netty线程启动源码)
- [2. Netty线程处理非io事件任务源码](#2-netty线程处理非IO事件任务源码)
- [3. Netty线程处理非io事件任务源码](#3-netty线程处理IO事件任务源码)
   - [3.1 连接接收accept](#31-连接接收accept)
   - [3.2 数据读取read](#32-数据读取read)
- [4. 补充selector空轮询bug与修复](#4补充selector空轮询bug与修复)
- [5. 总结](#5-总结)

## 1. Netty线程启动源码
上文说到在注册时候会启动一个线程进行异步注册，代码是这样的
```java
eventLoop.execute(new Runnable() {
    @Override
    public void run() {
    register0(promise);
    }
});
```
eventLoop的实现类是NioEventLoop,调用其execute方法，该方法的源码如下:
```java
public void execute(Runnable task) {
    ObjectUtil.checkNotNull(task, "task");
    execute(task, !(task instanceof LazyRunnable) && wakesUpForTask(task));
}
```
然后调用了重载方法execute,该方法的源码如下:
```java
private void execute(Runnable task, boolean immediate) {
    boolean inEventLoop = inEventLoop();
    addTask(task);
    if (!inEventLoop) {
        startThread();
        if (isShutdown()) {
            boolean reject = false;
            try {
                if (removeTask(task)) {
                    reject = true;
                }
            } catch (UnsupportedOperationException e) {
            }
            if (reject) {
                reject();
            }
        }
    }
    if (!addTaskWakesUp && immediate) {
        wakeup(inEventLoop);
    }
}
```
把任务添加到线程池的队列中去，如果当前线程不是EventLoop线程调用startThread方法，这里面主要调用了doStartThread方法，该方法源码
如下:
```java
private void doStartThread() {
assert thread == null;
executor.execute(new Runnable() {
    @Override
    public void run() {
        thread = Thread.currentThread();
        if (interrupted) {
            thread.interrupt();
        }
        boolean success = false;
        updateLastExecutionTime();
        try {
            SingleThreadEventExecutor.this.run();
            success = true;
        } catch (Throwable t) {
            logger.warn("Unexpected exception from an event executor: ", t);
        } finally {
            for (;;) {
                int oldState = state;
                if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                        SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                    break;
                }
            }
            if (success && gracefulShutdownStartTime == 0) {
                if (logger.isErrorEnabled()) {
                    logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                            SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must " +
                            "be called before run() implementation terminates.");
                }
            }
            try {
                for (;;) {
                    if (confirmShutdown()) {
                        break;
                    }
                }
                for (;;) {
                    int oldState = state;
                    if (oldState >= ST_SHUTDOWN || STATE_UPDATER.compareAndSet(
                            SingleThreadEventExecutor.this, oldState, ST_SHUTDOWN)) {
                        break;
                    }
                }
                confirmShutdown();
            } finally {
                try {
                    cleanup();
                } finally {
                    FastThreadLocal.removeAll();
                    STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                    threadLock.countDown();
                    int numUserTasks = drainTasks();
                    if (numUserTasks > 0 && logger.isWarnEnabled()) {
                        logger.warn("An event executor terminated with " +
                                "non-empty task queue (" + numUserTasks + ')');
                    }
                    terminationFuture.setSuccess(null);
                }
            }
        }
    }
  });
}
```
由此可以看出主要执行SingleThreadEventExecutor的run方法。

## 2. Netty线程处理非IO事件任务源码

可以看出主要调用的是SingleThreadEventExecutor的run方法，它的run方法源码如下:
```java
protected void run() {
        int selectCnt = 0;
        for (;;) {
            try {
                int strategy;
                try {
                    strategy = selectStrategy.calculateStrategy(selectNowSupplier, hasTasks());
                    switch (strategy) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.BUSY_WAIT:
                    case SelectStrategy.SELECT:
                        long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                        if (curDeadlineNanos == -1L) {
                            curDeadlineNanos = NONE;
                        }
                        nextWakeupNanos.set(curDeadlineNanos);
                        try {
                            if (!hasTasks()) {
                                strategy = select(curDeadlineNanos);
                            }
                        } finally {
                            nextWakeupNanos.lazySet(AWAKE);
                        }
                    default:
                    }
                } catch (IOException e) {
                    rebuildSelector0();
                    selectCnt = 0;
                    handleLoopException(e);
                    continue;
                }

                selectCnt++;
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                boolean ranTasks;
                if (ioRatio == 100) {
                    try {
                        if (strategy > 0) {
                            processSelectedKeys();
                        }
                    } finally {
                        ranTasks = runAllTasks();
                    }
                } else if (strategy > 0) {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        final long ioTime = System.nanoTime() - ioStartTime;
                        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                } else {
                    ranTasks = runAllTasks(0); 
                }

                if (ranTasks || strategy > 0) {
                    if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS && logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                                selectCnt - 1, selector);
                    }
                    selectCnt = 0;
                } else if (unexpectedSelectorWakeup(selectCnt)) { // Unexpected wakeup (unusual case)
                    selectCnt = 0;
                }
            } catch (CancelledKeyException e) {
                if (logger.isDebugEnabled()) {
                    logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                            selector, e);
                }
            } catch (Error e) {
                throw (Error) e;
            } catch (Throwable t) {
                handleLoopException(t);
            } finally {
                try {
                    if (isShuttingDown()) {
                        closeAll();
                        if (confirmShutdown()) {
                            return;
                        }
                    }
                } catch (Error e) {
                    throw (Error) e;
                } catch (Throwable t) {
                    handleLoopException(t);
                }
            }
        }
}
```
这段代码是Netty处理任务的核心逻辑，它主要处理IO事件(即处理客户端的连接和读写操作)，也有非Io事件(用户提交的Runnable)的调度和执行,这里先分析非IO事件的逻辑。主要调用的是
runAllTasks方法，该方法的源码如下:
```java
protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue();
    Runnable task = pollTask();
    if (task == null) {
        afterRunningAllTasks();
        return false;
    }

    final long deadline = timeoutNanos > 0 ? ScheduledFutureTask.nanoTime() + timeoutNanos : 0;
    long runTasks = 0;
    long lastExecutionTime;
    for (;;) {
        safeExecute(task);

        runTasks ++;
        if ((runTasks & 0x3F) == 0) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            if (lastExecutionTime >= deadline) {
                break;
            }
        }
        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }
    afterRunningAllTasks();
    this.lastExecutionTime = lastExecutionTime;
    return true;
}
```
先把把延时任务队列里的任务，移动到普通队列里(如果时间到了),然后从普通任务队列中拿出一个Runnable,调用safeExecute(task)执行任务，使用try/catch包裹起来执行
`runTasks & 0x3F == 0`表示每执行64个任务检查一次时间是否超过阈值，防止耗时太久阻塞IO,lastExecutionTime表示记录任务队列最后一次运行的时间（用于延时任务调度）。

## 3. Netty线程处理IO事件任务源码
### 3.1 连接接收(Accept)

上面的SingleThreadEventExecutor的run方法开始部分就是先处理IO事件的，我们来分析一下具体都做了什么事情。

先执行这段代码`selectStrategy.calculateStrategy`来选择一段执行策略，它主要是根据队列中是否有任务来判断的，如果有就返回0表示要先处理任务,如果没有返回-1
说明当前没有任务，尝试阻塞等待IO事件(如连接、读写)，返回-1的执行代码逻辑是:
```java
case SelectStrategy.SELECT:
    long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
    if (curDeadlineNanos == -1L) {
        curDeadlineNanos = NONE;
    }
    nextWakeupNanos.set(curDeadlineNanos);
    try {
        if (!hasTasks()) {
            strategy = select(curDeadlineNanos);
        }
    } finally {
        nextWakeupNanos.lazySet(AWAKE);
    }
```
首先计算定时任务的截止时间,用来确保Netty即使阻塞，也不会错过定时任务（比如用户设置了10秒后发送心跳包）,如果当前任务队列为空，才进行阻塞的select()，否则直接继续处理任务。

```java
catch (IOException e) {
    rebuildSelector0(); // 修复 Selector 空轮询 Bug
    selectCnt = 0;
    handleLoopException(e);
    continue;
}
```
如果 Selector 崩溃，比如触发了JDK的Bug，Netty 会自动重建一个 Selector。rebuildSelector0() 是 Netty 的“自我修复能力”的体现，生产环境非常重要。

接下来定义一些变量，
```java
selectCnt++;                   // select 调用计数器
cancelledKeys = 0;             // 重置取消 key 的标志
needsToSelectAgain = false;    // 这次不需要再 select
final int ioRatio = this.ioRatio;
```
- selectCnt：统计select次数，用于后面判断selector是否空轮询。

- ioRatio：这是用户可以配置的比例(默认是 50 或 100)，表示 I/O 时间和任务时间的比例，比如：

  - 100表示只处理 I/O，然后处理所有任务。
  - 50 表示 处理 I/O 的时间 ≈ 处理普通任务的时间。

**分三种策略:**

- **情况一：ioRatio == 100，全力冲 I/O，再任务**

```java
if (ioRatio == 100) {
    try {
        if (strategy > 0) {
            processSelectedKeys();  // 处理 IO 事件：连接、读写
        }
    } finally {
        ranTasks = runAllTasks();   // 处理所有任务
    }
}
```
先处理IO事件，strategy > 0表示有连接或者读写事件过来，否则就调用runAllTasks方法处理所有任务。

- **情况二：ioRatio != 100，比如 50**

```java
else if (strategy > 0) {
    final long ioStartTime = System.nanoTime();
    try {
        processSelectedKeys(); 
    } finally {
        final long ioTime = System.nanoTime() - ioStartTime;
        ranTasks = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
    }
}
```
用一个nanoTime()记录 I/O 执行的耗时,然后计算出要花在runAllTasks上的时间`ioTime * (100 - ioRatio) / ioRatio`,如果ioRatio是50%,I/O花了 1ms → task 最大执行时间也会限制在 1ms左右。

- **情况三：没IO，就尽量跑任务**

```java
else {
    ranTasks = runAllTasks(0);
}
```
下面分析一下有连接请求时候的源码。

有连接请求到来的时候会走到Selector发现有事件以后得到strategy值大于0，会调用processSelectedKeys方法，该方法源码如下:
```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```
此时selectedKeys不等于null,调用processSelectedKeysOptimized方法，该方法源码如下:
```java
private void processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        selectedKeys.keys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            selectedKeys.reset(i + 1);
            selectAgain();
            i = -1;
        }
    }
}
```
调用SelectionKey的attachment方法，此时得到的是NioServerSocketChannel对象，这个在上篇文章说过了是在注册时候添加进去的，此时判断NioServerSocketChannel是否是
instanceof AbstractNioChannel，这里很显然是那么就调用processSelectedKey方法，该方法源码如下:
```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    if (!k.isValid()) {
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            return;
        }
        if (eventLoop == this) {
            unsafe.close(unsafe.voidPromise());
        }
        return;
    }

    try {
        int readyOps = k.readyOps();
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
            unsafe.finishConnect();
        }
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            ch.unsafe().forceFlush();
        }
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
```
获取NioServerSocketChannel的NioUnsafe对象用于处理底层的IO操作，获取readyOps连接值，判断是否有对应的事件，如果有调用
unsafe.read方法，该方法源码如下:
```java
public void read() {
        assert eventLoop().inEventLoop();
        final ChannelConfig config = config();
        final ChannelPipeline pipeline = pipeline();
        final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
        allocHandle.reset(config);
        boolean closed = false;
        Throwable exception = null;
        try {
            try {
                do {
                    int localRead = doReadMessages(readBuf);
                    if (localRead == 0) {
                        break;
                    }
                    if (localRead < 0) {
                        closed = true;
                        break;
                    }

                    allocHandle.incMessagesRead(localRead);
                } while (continueReading(allocHandle));
            } catch (Throwable t) {
                exception = t;
            }
            int size = readBuf.size();
            for (int i = 0; i < size; i ++) {
                readPending = false;
                pipeline.fireChannelRead(readBuf.get(i));
            }
            readBuf.clear();
            allocHandle.readComplete();
            pipeline.fireChannelReadComplete();
            if (exception != null) {
                closed = closeOnReadError(exception);

                pipeline.fireExceptionCaught(exception);
            }
            if (closed) {
                inputShutdown = true;
                if (isOpen()) {
                    close(voidPromise());
                }
            }
        } finally {
            if (!readPending && !config.isAutoRead()) {
                removeReadOp();
            }
        }
    }
}
```
获取NioServerSocketChannel的ChannelPipeline对象，获取配置ChannelConfig对象，allocHandle是用于动态控制读取大小（adaptive read）的对象。 读取循环：调用doReadMessages()方法
doReadMessages()是子类实现(比如NioServerSocketChannel是readBuf.add(accept())),它读取socket内容（或接收连接）并写入到readBuf中，循环条件是continueReading()，它控制自适应读取(避免过多次syscall),
doReadMessages源码如下：
```java
protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel());

        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
            logger.warn("Failed to create a new channel from an accepted socket.", t);

            try {
                ch.close();
            } catch (Throwable t2) {
                logger.warn("Failed to close a socket.", t2);
            }
        }

        return 0;
}
```
从源码可以看出NioServerSocketChannel实现就是获取原生SocketChannel对象，然后包装成NioSocketChannel对象，代表一个客户端的连接抽象，然后返回1，NioSocketChannel构造函数和NioServerSocketChannel差不多都是一顿
调用父类的构造函数,主要设置成员变量readInterestOp为可读、NIO原生的SelectableChannel、DefaultChannelPipeline对象、Unsafe对象，返回1以后接着执行read方法中的这一段逻辑:
```java
int size = readBuf.size();
for (int i = 0; i < size; i ++) {
    readPending = false;
    pipeline.fireChannelRead(readBuf.get(i));
}
```
调用NioServerSocketChannel的ChannelPipeline对象的fireChannelRead方法把NioSocketChannel对象传入，最后回调用到ServerBootstrapAcceptor的channelRead方法，至于为啥会调用到这里上文我已经说过了感兴趣的可以看一下，该方法源码
如下:
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg;

    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);
    setAttributes(child, childAttrs);

    try {
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```
child.pipeline()是获取NioSocketChannel的ChannelPipeline对象然后添加childHandler对象，childHandler对象是上文分析过的匿名内部类，
```java
new ChannelInitializer<Channel>() {

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ch.pipeline()
                .addLast(new MyHandlerA())
                .addLast(new DelimiterBasedFrameDecoder(4096, Delimiters.lineDelimiter()))
                .addLast(new LineBasedFrameDecoder(1024))
                .addLast(new StringDecoder(CharsetUtil.UTF_8))
                .addLast(new StringEncoder(CharsetUtil.UTF_8))
                .addLast(new ChatServerHandler());
    }
}
```
接着调用childGroup(worker工作线程)的register方法注册，经过一些列执行最终把MyHandlerA、DelimiterBasedFrameDecoder等ChannelHandler注册到NioSocketChannel的
ChannelPipeline上等待处理读操作。

### 3.2 数据读取(Read)
当有读取事件时候还是和上面处理逻辑一样只不过是unsafe.read方法执行逻辑不太一样，这里的unsafe实现类是NioSocketChannelUnsafe对象，它的read方法源码如下:
```java
public final void read() {
        final ChannelConfig config = config();
        if (shouldBreakReadReady(config)) {
            clearReadPending();
            return;
        }
        final ChannelPipeline pipeline = pipeline();
        final ByteBufAllocator allocator = config.getAllocator();
        final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
        allocHandle.reset(config);

        ByteBuf byteBuf = null;
        boolean close = false;
        try {
            do {
                byteBuf = allocHandle.allocate(allocator);
                allocHandle.lastBytesRead(doReadBytes(byteBuf));
                if (allocHandle.lastBytesRead() <= 0) {
                    byteBuf.release();
                    byteBuf = null;
                    close = allocHandle.lastBytesRead() < 0;
                    if (close) {
                        readPending = false;
                    }
                    break;
                }
                allocHandle.incMessagesRead(1);
                readPending = false;
                pipeline.fireChannelRead(byteBuf);
                byteBuf = null;
            } while (allocHandle.continueReading());
            allocHandle.readComplete();
            pipeline.fireChannelReadComplete();
            if (close) {
                closeOnRead(pipeline);
            }
        } catch (Throwable t) {
            handleReadException(pipeline, byteBuf, t, close, allocHandle);
        } finally {
            if (!readPending && !config.isAutoRead()) {
                removeReadOp();
            }
        }
    }
}
```
读取前先检查是否需要提前退出读逻辑，比如channel已经关闭,clearReadPending() 清除是否还有 pending 的读取任务。

获取到NioSocketChannel的ChannelPipeline对象、ByteBufAllocator：用于分配ByteBuf内存、allocHandle：自适应读取控制器(会根据上次读取数据量决定下次是否继续读)。

下面开始循环读取数据通过do/while方式读取,首先分配ByteBuf,调用方法doReadBytes从Java NIO的底层读取Socket数据到ByteBuf中,lastBytesRead()：记录本次读了多少字节,
lastBytesRead()：记录本次读了多少字节,continueReading()：控制是否继续读取（一般基于消息数量、总字节数、是否有剩余等）。

```java
allocHandle.readComplete();
pipeline.fireChannelReadComplete();
```
- 通知ByteBufAllocator本次读取结束(用于自适应调整 buffer 大小)
- 向pipeline触发channelReadComplete()，供handler做最终处理，本例中就会依次调用到LineBasedFrameDecoder、StringDecoder等，用于把读取数据用\r\n分隔，最终转换为string类型数据。

如果对方关闭连接，则关闭Channel。

如果发生异常调用handleReadException方法处理异常，释放 ByteBuf，通知pipeline。

## 4.补充：Selector空轮询Bug与修复

JDK的Selector.select() 在高并发下会偶发进入“空轮询”状态，CPU 100%。

Netty检测到这种情况后会 rebuild selector：

```java
if (selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
    rebuildSelector();
    selectCnt = 0;
}
```
rebuildSelector下面调用得是rebuildSelector0方法，该方法源码如下:
```java
private void rebuildSelector0() {
        final Selector oldSelector = selector;
        final SelectorTuple newSelectorTuple;
        if (oldSelector == null) {
            return;
        }
        try {
            newSelectorTuple = openSelector();
        } catch (Exception e) {
            return;
        }
        int nChannels = 0;
        for (SelectionKey key: oldSelector.keys()) {
            Object a = key.attachment();
            try {
                if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
                    continue;
                }

                int interestOps = key.interestOps();
                key.cancel();
                SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
                if (a instanceof AbstractNioChannel) {
                    ((AbstractNioChannel) a).selectionKey = newKey;
                }
                nChannels ++;
            } catch (Exception e) {
                logger.warn("Failed to re-register a Channel to the new Selector.", e);
                if (a instanceof AbstractNioChannel) {
                    AbstractNioChannel ch = (AbstractNioChannel) a;
                    ch.unsafe().close(ch.unsafe().voidPromise());
                } else {
                    NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                    invokeChannelUnregistered(task, key, e);
                }
            }
        }
        selector = newSelectorTuple.selector;
        unwrappedSelector = newSelectorTuple.unwrappedSelector;

        try {
            oldSelector.close();
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close the old Selector.", t);
            }
        }

        if (logger.isInfoEnabled()) {
            logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
        }
}
```
由源码可以看出创建一个干净的新Selector,拿到旧的channel,重新注册到新Selector上，抛弃旧Selector,最后关闭掉旧的Selector。

## 5. 总结

Netty的高性能不是单靠NIO，而是：

- 自研事件循环模型（EventLoopGroup）
- 非阻塞 IO + 任务队列并行执行（ioRatio）
- 拆分任务类型（定时、尾部）优化调度
- 可重建 Selector 规避空轮询 bug
- pipeline 解耦、责任链清晰

这使得Netty在海量连接、高并发场景下依然保持稳定和灵活。