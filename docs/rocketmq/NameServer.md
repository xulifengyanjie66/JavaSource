# RocketMQ源码详解(NameServer启动流程)


## 一、NameServer简介
 消息中间件的设计思路一般是基于主题订阅发布的机制，消息生产者(Producer)发送某一个主题到消息服务器，消息服务器负责将消息持久化存储，消息消费者(Consumer)订阅该兴趣的主题，消息服务器根据订阅信息(路由信息)将消息推送到消费者(Push模式)或者消费者主动向消息服务器拉去(Pull模式)，从而实现消息生产者与消息消费者解耦。
 为了避免消息服务器的单点故障导致的整个系统瘫痪，通常会部署多台消息服务器共同承担消息的存储。那消息生产者如何知道消息要发送到哪台消息服务器呢？如果某一台消息服务器宕机了，那么消息生产者如何在不重启服务情况下感知呢？NameServer就是为了解决以上问题设计的，下面这张图片展示了
NameServer和Broker和Producer(生产者)和Consumer(消费者)之间关系。

![ea58df94615952bfeffba7170bea4061.jpeg](..%2Fimg%2Fea58df94615952bfeffba7170bea4061.jpeg)

- Broker消息服务器在启动的时向所有NameServer注册。
- 消息生产者(Producer)在发送消息时之前先从NameServer获取Broker服务器地址列表，然后根据负载均衡算法从列表中选择一台服务器进行发送。
- NameServer与每台Broker保持长连接，并间隔30S检测Broker是否存活，如果检测到Broker宕机，则从路由注册表中删除。但是路由变化不会马上通知消息生产者。这样设计的目的是为了降低NameServer实现的复杂度，在消息发送端提供容错机制保证消息发送的可用性。
- 本篇源码基于RocketMQ 4.9.8版本。

## 二、创建NamesrvController过程
Namesrv启动的入口地址在org.apache.rocketmq.namesrv.NamesrvStartup的main方法中，该方法源码如下:

```java
public static void main(String[] args) {
    main0(args);
}
```

又调用了main0方法,该方法源码如下:

```java
public static NamesrvController main0(String[] args) {

        try {
            NamesrvController controller = createNamesrvController(args);
            start(controller);
            String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
            log.info(tip);
            System.out.printf("%s%n", tip);
            return controller;
        } catch (Throwable e) {
            e.printStackTrace();
            System.exit(-1);
        }

        return null;
}
```

又调用了createNamesrvController方法，该方法源码如下:

```java
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
         //解析命令行参数
        // ... (省略具体的解析代码)
        final NamesrvConfig namesrvConfig = new NamesrvConfig();
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        nettyServerConfig.setListenPort(9876);
        if (commandLine.hasOption('c')) {
           // ... (省略加载properties文件和属性映射的代码)
        }
        // 配置日志框架
        // ... (省略Logback配置代码)
        final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

        // remember all configs to prevent discard
        controller.getConfiguration().registerConfig(properties);

        return controller;
   }
```

首先创建NamesrvConfig对象,RocketMQ的NamesrvConfig 类是用于配置 NameServer 的相关参数，它是 RocketMQ 中非常核心的组件之一，负责管理和调度消息的元数据。NamesrvConfig 配置了 NameServer 的一些基本行为，确保消息能够正确地路由、调度，并提供集群中各节点的元数据。

创建Netty服务端对象NettyServerConfig并设置端口号是9876,下面有`commandLine.hasOption`代码是从命令行中读取参数并且设置到NamesrvConfig,NettyServerConfig。

创建NamesrvController对象并且传入构造函数NamesrvConfig、NettyServerConfig对象，它的构造函数如下:

```java
public NamesrvController(NamesrvConfig namesrvConfig, NettyServerConfig nettyServerConfig) {
    this.namesrvConfig = namesrvConfig;
    this.nettyServerConfig = nettyServerConfig;
    this.kvConfigManager = new KVConfigManager(this);
    this.routeInfoManager = new RouteInfoManager();
    this.brokerHousekeepingService = new BrokerHousekeepingService(this);
    this.configuration = new Configuration(
        log,
        this.namesrvConfig, this.nettyServerConfig
    );
    this.configuration.setStorePathFromConfig(this.namesrvConfig, "configStorePath");
}
```



- 初始化成员变量RouteInfoManager，它主要负责维护 Broker、Topic 的路由注册、变更、查询等信息。
- 初始化成员变量BrokerHousekeepingService,它主要负责监控与 Broker 的连接状态，处理断连等异常情况。

创建完NamesrvController对象返回到main0方法，调用start(controller),该方法源码如下:

```java
public static NamesrvController start(final NamesrvController controller) throws Exception {

        if (null == controller) {
            throw new IllegalArgumentException("NamesrvController is null");
        }

        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }

        Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, (Callable<Void>) () -> {
            controller.shutdown();
            return null;
        }));

        controller.start();

        return controller;
}
```

先调用controller的initialize方法，该方法源码如下:

```java
public boolean initialize() {

        this.kvConfigManager.load();

        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

        this.registerProcessor();

        this.scheduledExecutorService.scheduleAtFixedRate(NamesrvController.this.routeInfoManager::scanNotActiveBroker, 5, 10, TimeUnit.SECONDS);

        this.scheduledExecutorService.scheduleAtFixedRate(NamesrvController.this.kvConfigManager::printAllPeriodically, 1, 10, TimeUnit.MINUTES);

        if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
            // Register a listener to reload SslContext
            try {
                fileWatchService = new FileWatchService(
                    new String[] {
                        TlsSystemConfig.tlsServerCertPath,
                        TlsSystemConfig.tlsServerKeyPath,
                        TlsSystemConfig.tlsServerTrustCertPath
                    },
                    new FileWatchService.Listener() {
                        boolean certChanged, keyChanged = false;
                        @Override
                        public void onChanged(String path) {
                            if (path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                                log.info("The trust certificate changed, reload the ssl context");
                                reloadServerSslContext();
                            }
                            if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                                certChanged = true;
                            }
                            if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                                keyChanged = true;
                            }
                            if (certChanged && keyChanged) {
                                log.info("The certificate and private key changed, reload the ssl context");
                                certChanged = keyChanged = false;
                                reloadServerSslContext();
                            }
                        }
                        private void reloadServerSslContext() {
                            ((NettyRemotingServer) remotingServer).loadSslContext();
                        }
                    });
            } catch (Exception e) {
                log.warn("FileWatchService created error, can't load the certificate dynamically");
            }
        }

        return true;
 }
```



创建一个NettyRemotingServer对象，它主要是与Netty网络服务有关的对象，它的构造函数如下：
```java
public NettyRemotingServer(final NettyServerConfig nettyServerConfig,
        final ChannelEventListener channelEventListener) {
        super(nettyServerConfig.getServerOnewaySemaphoreValue(), nettyServerConfig.getServerAsyncSemaphoreValue());
        this.serverBootstrap = new ServerBootstrap();
        this.nettyServerConfig = nettyServerConfig;
        this.channelEventListener = channelEventListener;

        int publicThreadNums = nettyServerConfig.getServerCallbackExecutorThreads();
        if (publicThreadNums <= 0) {
            publicThreadNums = 4;
        }

        this.publicExecutor = Executors.newFixedThreadPool(publicThreadNums, new ThreadFactory() {
            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "NettyServerPublicExecutor_" + this.threadIndex.incrementAndGet());
            }
        });

        if (useEpoll()) {
            this.eventLoopGroupBoss = new EpollEventLoopGroup(1, new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyEPOLLBoss_%d", this.threadIndex.incrementAndGet()));
                }
            });

            this.eventLoopGroupSelector = new EpollEventLoopGroup(nettyServerConfig.getServerSelectorThreads(), new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);
                private int threadTotal = nettyServerConfig.getServerSelectorThreads();

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyServerEPOLLSelector_%d_%d", threadTotal, this.threadIndex.incrementAndGet()));
                }
            });
        } else {
            this.eventLoopGroupBoss = new NioEventLoopGroup(1, new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyNIOBoss_%d", this.threadIndex.incrementAndGet()));
                }
            });

            this.eventLoopGroupSelector = new NioEventLoopGroup(nettyServerConfig.getServerSelectorThreads(), new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);
                private int threadTotal = nettyServerConfig.getServerSelectorThreads();

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, String.format("NettyServerNIOSelector_%d_%d", threadTotal, this.threadIndex.incrementAndGet()));
                }
            });
        }

        loadSslContext();
    }
```
创建一个固定线程池对象ExecutorService publicExecutor，它的线程池线程对象是4。判断当前操作系统的情况:

- 如果是Linux系统,创建一个EpollEventLoopGroup的boss线程池，线程池线程数量是1，线程工厂是自定义工厂,创建一个EpollEventLoopGroup的
  worker线程池，线程池线程数量是nettyServerConfig.getServerSelectorThreads()(默认配置是3),线程工厂同样是是自定义工厂。

- 如果不是Linux系统，创建一个NioEventLoopGroup的boss线程池，线程池线程数量是1，线程工厂是自定义工厂,创建一个NioEventLoopGroup的
  worker线程池，线程池线程数量是nettyServerConfig.getServerSelectorThreads()(默认配置是3),线程工厂同样是是自定义工厂。

这段代码`Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));`

创建一个固定的线程池，线程池数量是取得nettyServerConfig.getServerWorkerThreads()的数量，默认是8，它的主要作用业务处理。

调用方法`this.registerProcessor();`注册处理器，将具体的 **请求处理器** 注册到 `remotingServer` 上，比如处理注册 Broker 的请求、处理客户端路由查询请求,不同的请求 code 会对应不同的 `NettyRequestProcessor` 实现类。

`this.scheduledExecutorService.scheduleAtFixedRate(NamesrvController.this.routeInfoManager::scanNotActiveBroker, 5, 10, TimeUnit.SECONDS);`

这段代码主要定时处理扫描不活跃的Broker,每隔 10 秒执行一次，首次延迟 5 秒。检查 Broker 是否还活着（最后心跳时间超过 120 秒则判定失联）,将失联的 Broker 从路由表中移除，保持数据一致性。下面以一个表格说明一下这些参数得作用：

|           参数名            | 默认值 |              说明              |
| :-------------------------: | :----: | :----------------------------: |
|     serverWorkerThreads     |   8    |        Netty 业务线程数        |
|    serverSelectorThreads    |   3    |        Netty I/O 线程数        |
| brokerChannelMaxIdleSeconds |  120   |      Broker 心跳超时时间       |
| scanNotActiveBrokerInterval |   10   | 扫描不活跃 Broker 的间隔（秒） |

 接着调用NamesrvController的start方法，该方法源码如下:
```java
 public void start() throws Exception {
        this.remotingServer.start();

        if (this.fileWatchService != null) {
            this.fileWatchService.start();
        }
}
```
可以看出调用remotingServer的start方法为了启动Netty服务器端,该方法源码如下:
```java
 public void start() {
    this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
        nettyServerConfig.getServerWorkerThreads(),
        new ThreadFactory() {

            private AtomicInteger threadIndex = new AtomicInteger(0);

            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "NettyServerCodecThread_" + this.threadIndex.incrementAndGet());
            }
        });

    prepareSharableHandlers();

    ServerBootstrap childHandler =
        this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
            .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, nettyServerConfig.getServerSocketBacklog())
            .option(ChannelOption.SO_REUSEADDR, true)
            .option(ChannelOption.SO_KEEPALIVE, false)
            .childOption(ChannelOption.TCP_NODELAY, true)
            .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline()
                        .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)
                        .addLast(defaultEventExecutorGroup,
                            encoder,
                            new NettyDecoder(),
                            new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                            connectionManageHandler,
                            serverHandler
                        );
                }
            });
    if (nettyServerConfig.getServerSocketSndBufSize() > 0) {
        log.info("server set SO_SNDBUF to {}", nettyServerConfig.getServerSocketSndBufSize());
        childHandler.childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize());
    }
    if (nettyServerConfig.getServerSocketRcvBufSize() > 0) {
        log.info("server set SO_RCVBUF to {}", nettyServerConfig.getServerSocketRcvBufSize());
        childHandler.childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize());
    }
    if (nettyServerConfig.getWriteBufferLowWaterMark() > 0 && nettyServerConfig.getWriteBufferHighWaterMark() > 0) {
        log.info("server set netty WRITE_BUFFER_WATER_MARK to {},{}",
                nettyServerConfig.getWriteBufferLowWaterMark(), nettyServerConfig.getWriteBufferHighWaterMark());
        childHandler.childOption(ChannelOption.WRITE_BUFFER_WATER_MARK, new WriteBufferWaterMark(
                nettyServerConfig.getWriteBufferLowWaterMark(), nettyServerConfig.getWriteBufferHighWaterMark()));
    }

    if (nettyServerConfig.isServerPooledByteBufAllocatorEnable()) {
        childHandler.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
    }

    try {
        ChannelFuture sync = this.serverBootstrap.bind().sync();
        InetSocketAddress addr = (InetSocketAddress) sync.channel().localAddress();
        this.port = addr.getPort();
    } catch (InterruptedException e1) {
        throw new RuntimeException("this.serverBootstrap.bind().sync() InterruptedException", e1);
    }

    if (this.channelEventListener != null) {
        this.nettyEventExecutor.start();
    }

    this.timer.scheduleAtFixedRate(new TimerTask() {

        @Override
        public void run() {
            try {
                NettyRemotingServer.this.scanResponseTable();
            } catch (Throwable e) {
                log.error("scanResponseTable exception", e);
            }
        }
    }, 1000 * 3, 1000);
}
```
创建DefaultEventExecutorGroup对象,调用prepareSharableHandlers方法，该方法源码如下：
```java
private void prepareSharableHandlers() {
    handshakeHandler = new HandshakeHandler(TlsSystemConfig.tlsMode);
    encoder = new NettyEncoder();
    connectionManageHandler = new NettyConnectManageHandler();
    serverHandler = new NettyServerHandler();
}
```
分别初始化HandshakeHandler handshakeHandler、NettyEncoder encoder(Netty编码器对象)、NettyConnectManageHandler connectionManageHandler、NettyServerHandler serverHandler(用来处理入站数据),接下来就是
调用serverBootstrap的group方法把上面组件注册上，然后调用bind方法启动Netty服务。

## 三、总结

NameServer 作为 RocketMQ 的元数据中枢和轻量级路由发现服务，其启动流程充分体现了**简洁高效、职责清晰**的设计哲学。整个过程并非简单地启动一个网络服务，而是有条不紊地完成了三件核心大事：

- **解析与装配**：读取命令行与环境配置，初始化核心参数，为后续组件提供运行依据。

- **初始化核心组件:** 构建 `RouteInfoManager`（路由表）、`KVConfigManager`（配置管理）、`BrokerHousekeepingService`（连接监护）等核心管理组件，各司其职。
- **启动服务与定时任务**：初始化 Netty 服务器以接收网络请求，并启动两个关键的后台线程：
  - **心跳检测**：定期扫描并清理失效的 Broker 节点，确保路由信息的实时性与准确性。
  - **配置持久化**：定期将运行时配置落盘，保证数据的可恢复性。

