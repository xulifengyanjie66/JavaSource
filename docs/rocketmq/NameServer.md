# RocketMQ源码详解(NameServer启动流程)
- [1. NameServer 简介](#1-nameserver-简介)
- [2. 启动流程](#2-启动流程)
    - [2.1 Main 方法入口](#21-main-方法入口)
    - [2.2 初始化配置](#22-初始化配置)
    - [2.3 创建 Controller](#23-创建-controller)
    - [2.4 启动核心组件](#24-启动核心组件)
- [3. 核心组件解析](#3-核心组件解析)
    - [3.1 RouteInfoManager](#31-routeinfomanager)
    - [3.2 KVConfigManager](#32-kvconfigmanager)
    - [3.3 RemotingServer（Netty）](#33-remotingservernetty)
- [4. Broker 注册与心跳](#4-broker-注册与心跳)
    - [4.1 Broker 注册流程](#41-broker-注册流程)
    - [4.2 心跳检查机制](#42-心跳检查机制)
- [5. 常见请求处理流程](#5-常见请求处理流程)
    - [5.1 GET_ROUTEINFO_BY_TOPIC](#51-get_routeinfo_by_topic)
    - [5.2 PUT_KV_CONFIG / GET_KV_CONFIG](#52-put_kv_config--get_kv_config)
- [6. 总结](#6-总结)

## 一、NameServer简介
 消息中间件的设计思路一般是基于主题订阅发布的机制，消息生产者(Producer)发送某一个主题到消息服务器，消息服务器负责将消息持久化存储，消息消费者(Consumer)订阅该兴趣的主题，消息服务器根据订阅信息(路由信息)将消息推送到消费者(Push模式)或者消费者主动向消息服务器拉去(Pull模式)，从而实现消息生产者与消息消费者解耦。
 为了避免消息服务器的单点故障导致的整个系统瘫痪，通常会部署多台消息服务器共同承担消息的存储。那消息生产者如何知道消息要发送到哪台消息服务器呢？如果某一台消息服务器宕机了，那么消息生产者如何在不重启服务情况下感知呢？NameServer就是为了解决以上问题设计的，下面这张图片展示了
NameServer和Broker和Producer(生产者)和Consumer(消费者)之间关系。

![ea58df94615952bfeffba7170bea4061.jpeg](..%2Fimg%2Fea58df94615952bfeffba7170bea4061.jpeg)

- Broker消息服务器在启动的时向所有NameServer注册。
- 消息生产者(Producer)在发送消息时之前先从NameServer获取Broker服务器地址列表，然后根据负载均衡算法从列表中选择一台服务器进行发送。
- NameServer与每台Broker保持长连接，并间隔30S检测Broker是否存活，如果检测到Broker宕机，则从路由注册表中删除。但是路由变化不会马上通知消息生产者。这样设计的目的是为了降低NameServer实现的复杂度，在消息发送端提供容错机制保证消息发送的可用性。

## 二、NamesrvConfig介绍
RocketMQ的NamesrvConfig 类是用于配置 NameServer 的相关参数，它是 RocketMQ 中非常核心的组件之一，负责管理和调度消息的元数据。NamesrvConfig 配置了 NameServer 的一些基本行为，确保消息能够正确地路由、调度，并提供集群中各节点的元数据。

NamesrvController的initialize方法，该方法的源码如下:
```java
public boolean initialize() {

        ..................

        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

        this.registerProcessor();

        this.scheduledExecutorService.scheduleAtFixedRate(NamesrvController.this.routeInfoManager::scanNotActiveBroker, 5, 10, TimeUnit.SECONDS);

        this.scheduledExecutorService.scheduleAtFixedRate(NamesrvController.this.kvConfigManager::printAllPeriodically, 1, 10, TimeUnit.MINUTES);

        .................................
```
创建一个NettyRemotingServer对象，它主要是与netty网络服务有关的对象，它的构造函数如下：
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
创建一个固定线程池对象ExecutorService publicExecutor，它的线程池线程对象是4。

判断是否是Linux系统，如果是Linux系统，创建一个EpollEventLoopGroup的boss线程池，线程池线程数量是1，线程工厂是自定义工厂,创建一个EpollEventLoopGroup的
worker线程池，线程池线程数量是nettyServerConfig.getServerSelectorThreads()(如额外配置默认是3),线程工厂同样是是自定义工厂。

判断是否是Linux系统，如果不是Linux系统，创建一个NioEventLoopGroup的boss线程池，线程池线程数量是1，线程工厂是自定义工厂,创建一个NioEventLoopGroup的
worker线程池，线程池线程数量是nettyServerConfig.getServerSelectorThreads()(如额外配置默认是3),线程工厂同样是是自定义工厂。

调用NamesrvController的start方法，该方法源码如下:
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