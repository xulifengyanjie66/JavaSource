

# RocketMQ源码详解(发送消息源码)

## 一、前言

在分布式消息系统中，**发送端（Producer）** 扮演着至关重要的角色。它不仅负责将消息高效、可靠地投递到 Broker，还需要在网络波动、Broker负载变化等复杂场景下保证消息的一致性和可达性。
 Apache RocketMQ 作为阿里巴巴开源并捐赠给 Apache 基金会的高性能消息中间件，其发送端实现有着极高的工程价值和设计考量。

很多人使用 RocketMQ 时，只需调用几行 API 就能发送消息，却很少去关注背后到底发生了什么：

- 消息是如何从 Producer 组装、序列化并发往 Broker 的？
- 发送过程中的重试、超时、路由更新是怎么实现的？
- 同步发送、异步发送、单向发送底层的区别在哪里？代码是如何实现的？

本文将基于 **RocketMQ 4.9.8** 版本，从源码角度深入解析发送端的核心流程，帮助你透彻理解 Producer 发出一条消息背后的完整链路。

##  二、测试代码

本文中基于下面的代码进行测试演示Producer发送消息的流程,我的代码如下:

```java
DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
producer.setNamesrvAddr("127.0.0.1:9876");
producer.start();

String topic = "eooi2156";

Message msg = new Message(topic, "TagA", ("Hello RocketMQ").getBytes());
//同步发送消息
producer.send(msg);
//异步发送消息
producer.send(msg, new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {

    }
    @Override
    public void onException(Throwable e) {

    }
});
//单向发送消息
producer.sendOneway(msg);
```

发送普通消息主要分为三种一种是同步发送消息、一种是异步发送消息、一种是单向发送消息，它们的区别我列举一个表格说明一下:

| 发送方式 | 是否阻塞等待 | 是否关注发送结果 | 是否关注发送结果 |         适用场景         |
| :------: | :----------: | :--------------: | :--------------: | :----------------------: |
| 同步发送 |      是      |        是        |        是        |  关键业务，可靠性要求高  |
| 异步发送 |      否      |  是（异步回调）  |        是        | 高吞吐、低延迟，关注结果 |
| 单向发送 |      否      |        否        |        否        |  性能优先，丢消息可接受  |

下面对这三种情况分别做源码分析。

## 三、同步发送源码分析

首先创建一个DefaultMQProducer对象，它继承了ClientConfig类，实现了MQProducer接口，其中MQProducer接口中定义了关键方法，比如start、shutdown、send、sendOneway、sendMessageInTransaction(事务方法)，它的继承结构如下：



![image-20250809141538949](C:\Users\Administrator\Desktop\image-20250809141538949.png)



它的构造函数初始化一个成员变量DefaultMQProducerImpl，DefaultMQProducerImpl又初始化了成员变量`BlockingQueue<Runnable> asyncSenderThreadPoolQueue`和`ExecutorService defaultAsyncSenderExecutor`,这块的源码如下:

```java
public DefaultMQProducerImpl(final DefaultMQProducer defaultMQProducer, RPCHook rpcHook) {
        this.defaultMQProducer = defaultMQProducer;
        this.rpcHook = rpcHook;

        this.asyncSenderThreadPoolQueue = new LinkedBlockingQueue<Runnable>(50000);
        this.defaultAsyncSenderExecutor = new ThreadPoolExecutor(
            Runtime.getRuntime().availableProcessors(),
            Runtime.getRuntime().availableProcessors(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.asyncSenderThreadPoolQueue,
            new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, "AsyncSenderExecutor_" + this.threadIndex.incrementAndGet());
                }
            });
 }
```

ExecutorService defaultAsyncSenderExecutor主要是发送异步消息时候，消息发送逻辑在此线程池执行防止阻塞主线程。



设置DefaultMQProducer的NameSrv的注册中心地址，开始调用DefaultMQProducer的start方法,该方法源码如下:

```java
public void start() throws MQClientException {
    this.setProducerGroup(withNamespace(this.producerGroup));
    this.defaultMQProducerImpl.start();
    if (null != traceDispatcher) {
        try {
            traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
        } catch (MQClientException e) {
            log.warn("trace dispatcher start failed ", e);
        }
    }
}
```

设置ProducerGroup的名称，调用DefaultMQProducerImpl的start方法，该方法源码如下:

```java
public void start(final boolean startFactory) throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;

                this.checkConfig();

                if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                    this.defaultMQProducer.changeInstanceNameToPID();
                }

                this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);

                boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;
                    throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
                }

                this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

                if (startFactory) {
                    mQClientFactory.start();
                }

                log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                    this.defaultMQProducer.isSendMessageWithVIPChannel());
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The producer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
            default:
                break;
        }

        this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();

        RequestFutureHolder.getInstance().startScheduledTask(this);

}
```

这里通过`MQClientManager.getInstance()`获取一个MQClientManager对象，它是一个静态成员变量因此不管调用几次获取的都是同一个对象，调用它的方法getOrCreateMQClientInstance获取是一个MQClientInstance对象，它是一个客户端运行时的大管家，它负责管理一个 JVM 进程里所有 Producer 和 Consumer 的公共资源与后台任务，它的主要作用是:

### 1. **连接管理中心**

- 一个客户端进程（可能有多个 Producer 和 Consumer）**只会有一个 `MQClientInstance`** 对应一个唯一的 `clientId`（通常是 *ip@instanceName*）。
- 它会维护到 **NameServer** 和 **Broker** 的网络连接（基于 `NettyRemotingClient`）。
- 负责跟 NameServer 定时拉取最新的 Topic 路由信息，并缓存到本地。

### 2. **路由信息管理**

- 内部有 `MQClientAPIImpl` 和 `MQAdminImpl` 负责与 NameServer 交互。
- 会定时调用 `updateTopicRouteInfoFromNameServer` 拉取最新的 Topic -> Broker 映射关系。
- Producer 发送消息前，都是通过 `MQClientInstance` 查询路由信息来找到目标 Broker。

### 3. **心跳与注册**

- 定时向 Broker 发送心跳包，注册 Producer Group / Consumer Group 信息。

- 如果 Broker 检测到客户端心跳超时，就会认为该 Producer 或 Consumer 已经下线。

我们先来看看getOrCreateMQClientInstance方法签名:

```java
public MQClientInstance getOrCreateMQClientInstance(final ClientConfig clientConfig, RPCHook rpcHook) {
    String clientId = clientConfig.buildMQClientId();
    MQClientInstance instance = this.factoryTable.get(clientId);
    if (null == instance) {
        instance =
            new MQClientInstance(clientConfig.cloneClientConfig(),
                                 this.factoryIndexGenerator.getAndIncrement(), clientId, rpcHook);
        MQClientInstance prev = this.factoryTable.putIfAbsent(clientId, instance);
        if (prev != null) {
            instance = prev;
            log.warn("Returned Previous MQClientInstance for clientId:[{}]", clientId);
        } else {
            log.info("Created new MQClientInstance for clientId:[{}]", clientId);
        }
    }

    return instance;
}
```

获取clientId,去ConcurrentMap中是否已经有对应的MQClientInstance对象，如果没有则创建MQClientInstance对象，该对象构造函数如下：

```java
public MQClientInstance(ClientConfig clientConfig, int instanceIndex, String clientId, RPCHook rpcHook) {
        this.clientConfig = clientConfig;
        this.instanceIndex = instanceIndex;
        this.nettyClientConfig = new NettyClientConfig();
        this.nettyClientConfig.setClientCallbackExecutorThreads(clientConfig.getClientCallbackExecutorThreads());
        this.nettyClientConfig.setUseTLS(clientConfig.isUseTLS());
        this.clientRemotingProcessor = new ClientRemotingProcessor(this);
        this.mQClientAPIImpl = new MQClientAPIImpl(this.nettyClientConfig, this.clientRemotingProcessor, rpcHook, clientConfig);

        if (this.clientConfig.getNamesrvAddr() != null) {
            this.mQClientAPIImpl.updateNameServerAddressList(this.clientConfig.getNamesrvAddr());
            log.info("user specified name server address: {}", this.clientConfig.getNamesrvAddr());
        }

        this.clientId = clientId;

        this.mQAdminImpl = new MQAdminImpl(this);

        this.pullMessageService = new PullMessageService(this);

        this.rebalanceService = new RebalanceService(this);

        this.defaultMQProducer = new DefaultMQProducer(MixAll.CLIENT_INNER_PRODUCER_GROUP);
        this.defaultMQProducer.resetClientConfig(clientConfig);

        this.consumerStatsManager = new ConsumerStatsManager(this.scheduledExecutorService);

        log.info("Created a new client Instance, InstanceIndex:{}, ClientID:{}, ClientConfig:{}, ClientVersion:{}, SerializerType:{}",
            this.instanceIndex,
            this.clientId,
            this.clientConfig,
            MQVersion.getVersionDesc(MQVersion.CURRENT_VERSION), RemotingCommand.getSerializeTypeConfigInThisServer());
}
```



它有几个重要的成员变量，下面我以表格的方式列举一下:

|               成员变量                |                            作用                             |
| :-----------------------------------: | :---------------------------------------------------------: |
|    MQClientAPIImpl mQClientAPIImpl    | 底层网络通信组件，封装了与 NameServer 和 Broker 的 RPC 调用 |
| PullMessageService pullMessageService |            专门处理 Consumer 拉取消息的服务线程             |
|   RebalanceService rebalanceService   |             消费负载均衡服务，定时调整队列分配              |
|                                       |                                                             |

调用`this.mQClientAPIImpl.start()`方法启动Netty客户端服务，调用`this.pullMessageService.start()`启动拉取消息服务，调用`this.rebalanceService.start()`启动消费复杂均衡服务。

以一张形象图描述MQClientInstance客户端实例:

![7550e6b3-fc93-4262-ab7d-4092ff52eb90.png](..%2Fimg%2F7550e6b3-fc93-4262-ab7d-4092ff52eb90.png)

这些都执行完成回到测试代码开始执行DefaultMQProducer的send方法，传入要发送的Message消息，它主要包含主题名称、标签、以及消息体内容，send方法一步步调用最终回调用到sendDefaultImpl方法，该方法源码如下：

```java
private SendResult sendDefaultImpl(
        Message msg,
        final CommunicationMode communicationMode,
        final SendCallback sendCallback,
        final long timeout
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        this.makeSureStateOK();
        Validators.checkMessage(msg, this.defaultMQProducer);
        final long invokeID = random.nextLong();
        long beginTimestampFirst = System.currentTimeMillis();
        long beginTimestampPrev = beginTimestampFirst;
        long endTimestamp = beginTimestampFirst;
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            boolean callTimeout = false;
            MessageQueue mq = null;
            Exception exception = null;
            SendResult sendResult = null;
            int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
            int times = 0;
            String[] brokersSent = new String[timesTotal];
            for (; times < timesTotal; times++) {
                String lastBrokerName = null == mq ? null : mq.getBrokerName();
                MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                if (mqSelected != null) {
                    mq = mqSelected;
                    brokersSent[times] = mq.getBrokerName();
                    try {
                        beginTimestampPrev = System.currentTimeMillis();
                        if (times > 0) {
                            //Reset topic with namespace during resend.
                            msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
                        }
                        long costTime = beginTimestampPrev - beginTimestampFirst;
                        if (timeout < costTime) {
                            callTimeout = true;
                            break;
                        }

                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                        switch (communicationMode) {
                            case ASYNC:
                                return null;
                            case ONEWAY:
                                return null;
                            case SYNC:
                                if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                    if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                        continue;
                                    }
                                }

                                return sendResult;
                            default:
                                break;
                        }
                    } catch (RemotingException e) {
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        continue;
                    } catch (MQClientException e) {
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        continue;
                    } catch (MQBrokerException e) {
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        exception = e;
                        if (this.defaultMQProducer.getRetryResponseCodes().contains(e.getResponseCode())) {
                            continue;
                        } else {
                            if (sendResult != null) {
                                return sendResult;
                            }

                            throw e;
                        }
                    } catch (InterruptedException e) {
                        endTimestamp = System.currentTimeMillis();
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                        log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mq), e);
                        log.warn(msg.toString());
                        throw e;
                    }
                } else {
                    break;
                }
            }

            if (sendResult != null) {
                return sendResult;
            }

            String info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s",
                times,
                System.currentTimeMillis() - beginTimestampFirst,
                msg.getTopic(),
                Arrays.toString(brokersSent));

            info += FAQUrl.suggestTodo(FAQUrl.SEND_MSG_FAILED);

            MQClientException mqClientException = new MQClientException(info, exception);
            if (callTimeout) {
                throw new RemotingTooMuchRequestException("sendDefaultImpl call timeout");
            }

            if (exception instanceof MQBrokerException) {
                mqClientException.setResponseCode(((MQBrokerException) exception).getResponseCode());
            } else if (exception instanceof RemotingConnectException) {
                mqClientException.setResponseCode(ClientErrorCode.CONNECT_BROKER_EXCEPTION);
            } else if (exception instanceof RemotingTimeoutException) {
                mqClientException.setResponseCode(ClientErrorCode.ACCESS_BROKER_TIMEOUT);
            } else if (exception instanceof MQClientException) {
                mqClientException.setResponseCode(ClientErrorCode.BROKER_NOT_EXIST_EXCEPTION);
            }

            throw mqClientException;
        }

        validateNameServerSetting();

        throw new MQClientException("No route info of this topic: " + msg.getTopic() + FAQUrl.suggestTodo(FAQUrl.NO_TOPIC_ROUTE_INFO),
            null).setResponseCode(ClientErrorCode.NOT_FOUND_TOPIC_EXCEPTION);
    }
```

先是调用`tryToFindTopicPublishInfo`方法从远处NameSrv获取对应主题路由信息`TopicPublishInfo`,判断如果是同步模式，发送消息时如果失败，可以重试，重试次数是1 + this.defaultMQProducer.getRetryTimesWhenSendFailed(),配置this.defaultMQProducer.getRetryTimesWhenSendFailed()默认是2，总共是3次，选出一个MessageQueue对象用于发送消息到这个队列，接着调用重载方法sendKernelImpl发送消息,发送消息分为同步发送、异步发送、单向发送，其中同步发送消息得逻辑主要是封装SendMessageRequestHeader对象，设置消费者组名称、主题名称、Broker名称然后调用`RemotingClient`对象的`invokeSync`方法最后调用await阻塞等待获取结果，这块逻辑已经在Broker注册心跳那块已经分析过了就不再重复分析了。



## 四、单向发送消息源码：

单向发送消息核心源码如下：

```java
public void invokeOnewayImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis)
        throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
        request.markOnewayRPC();
        boolean acquired = this.semaphoreOneway.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
        if (acquired) {
            final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreOneway);
            try {
                channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture f) throws Exception {
                        once.release();
                        if (!f.isSuccess()) {
                            log.warn("send a request command to channel <" + channel.remoteAddress() + "> failed.");
                        }
                    }
                });
            } catch (Exception e) {
                once.release();
                log.warn("write send a request command to channel <" + channel.remoteAddress() + "> failed.");
                throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
            }
        } else {
            if (timeoutMillis <= 0) {
                throw new RemotingTooMuchRequestException("invokeOnewayImpl invoke too fast");
            } else {
                String info = String.format(
                    "invokeOnewayImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreOnewayValue: %d",
                    timeoutMillis,
                    this.semaphoreOneway.getQueueLength(),
                    this.semaphoreOneway.availablePermits()
                );
                log.warn(info);
                throw new RemotingTimeoutException(info);
            }
        }
}
```

可以看出这段代码只是调用了Netty的Channel对象的writeAndFlush方法写给Broker消息并没有处理返回的消息所以说它是消息不可靠的，但是速度最快的一种方式适合不关心是否发送成功的场景，比如日志。

## 五、异步发送消息源码 

前面测试代码说过发送异步消息的源码的API是这样写的：

```java
producer.send(msg, new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {

    }

    @Override
    public void onException(Throwable e) {

    }
});
```

其中SendCallback参数是异步回调时候要调用的对象，它主要有两个方法，一个是onSuccess表示成功后处理的逻辑，一个是onException表示失败后处理的逻辑，它的send函数主要处理逻辑源码如下:

```java
public void send(final Message msg, final SendCallback sendCallback, final long timeout)
        throws MQClientException, RemotingException, InterruptedException {
        final long beginStartTime = System.currentTimeMillis();
        ExecutorService executor = this.getAsyncSenderExecutor();
        try {
            executor.submit(new Runnable() {
                @Override
                public void run() {
                    long costTime = System.currentTimeMillis() - beginStartTime;
                    if (timeout > costTime) {
                        try {
                            sendDefaultImpl(msg, CommunicationMode.ASYNC, sendCallback, timeout - costTime);
                        } catch (Exception e) {
                            sendCallback.onException(e);
                        }
                    } else {
                        sendCallback.onException(
                            new RemotingTooMuchRequestException("DEFAULT ASYNC send call timeout"));
                    }
                }

            });
        } catch (RejectedExecutionException e) {
            throw new MQClientException("executor rejected ", e);
        }

}
```

可以看出它是在线程池中执行的sendDefaultImpl方法，这是不阻塞主线程的关键之处，接着调用sendDefaultImpl方法，该方法的逻辑和上面同步方法逻辑一样，只不过此处传入的第二个参数是`CommunicationMode.ASYNC`代表异步操作，最终会调用到`sendMessageAsync`方法，该方法源码如下:

```java
 private void sendMessageAsync(
        final String addr,
        final String brokerName,
        final Message msg,
        final long timeoutMillis,
        final RemotingCommand request,
        final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
        final MQClientInstance instance,
        final int retryTimesWhenSendFailed,
        final AtomicInteger times,
        final SendMessageContext context,
        final DefaultMQProducerImpl producer
    ) {
        final long beginStartTime = System.currentTimeMillis();
        try {
            this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
                @Override
                public void operationComplete(ResponseFuture responseFuture) {
                    long cost = System.currentTimeMillis() - beginStartTime;
                    RemotingCommand response = responseFuture.getResponseCommand();
                    if (null == sendCallback && response != null) {

                        try {
                            SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response, addr);
                            if (context != null && sendResult != null) {
                                context.setSendResult(sendResult);
                                context.getProducer().executeSendMessageHookAfter(context);
                            }
                        } catch (Throwable e) {
                        }

                        producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                        return;
                    }

                    if (response != null) {
                        try {
                            SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response, addr);
                            assert sendResult != null;
                            if (context != null) {
                                context.setSendResult(sendResult);
                                context.getProducer().executeSendMessageHookAfter(context);
                            }

                            try {
                                sendCallback.onSuccess(sendResult);
                            } catch (Throwable e) {
                            }

                            producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                        } catch (Exception e) {
                            producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                            onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                retryTimesWhenSendFailed, times, e, context, false, producer);
                        }
                    } else {
                        producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                        if (!responseFuture.isSendRequestOK()) {
                            MQClientException ex = new MQClientException("send request failed", responseFuture.getCause());
                            onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                retryTimesWhenSendFailed, times, ex, context, true, producer);
                        } else if (responseFuture.isTimeout()) {
                            MQClientException ex = new MQClientException("wait response timeout " + responseFuture.getTimeoutMillis() + "ms",
                                responseFuture.getCause());
                            onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                retryTimesWhenSendFailed, times, ex, context, true, producer);
                        } else {
                            MQClientException ex = new MQClientException("unknow reseaon", responseFuture.getCause());
                            onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                retryTimesWhenSendFailed, times, ex, context, true, producer);
                        }
                    }
                }
            });
        } catch (Exception ex) {
            long cost = System.currentTimeMillis() - beginStartTime;
            producer.updateFaultItem(brokerName, cost, true);
            onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                    retryTimesWhenSendFailed, times, ex, context, true, producer);
        }
}
```

该方法会调用`this.remotingClient.invokeAsync`传入一个回调对象InvokeCallback, 最终会调用到`invokeAsyncImpl`方法，该方法源码如下:

```java
public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,
        final InvokeCallback invokeCallback)
        throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
        long beginStartTime = System.currentTimeMillis();
        final int opaque = request.getOpaque();
        boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
        if (acquired) {
            final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                once.release();
                throw new RemotingTimeoutException("invokeAsyncImpl call timeout");
            }

            final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis - costTime, invokeCallback, once);
            this.responseTable.put(opaque, responseFuture);
            try {
                channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture f) throws Exception {
                        if (f.isSuccess()) {
                            responseFuture.setSendRequestOK(true);
                            return;
                        }
                        requestFail(opaque);
                    }
                });
            } catch (Exception e) {
                responseFuture.release();
                throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
            }
        } else {
            if (timeoutMillis <= 0) {
                throw new RemotingTooMuchRequestException("invokeAsyncImpl invoke too fast");
            } else {
                String info =
                    String.format("invokeAsyncImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreAsyncValue: %d",
                        timeoutMillis,
                        this.semaphoreAsync.getQueueLength(),
                        this.semaphoreAsync.availablePermits()
                    );
                log.warn(info);
                throw new RemotingTimeoutException(info);
            }
        }
    }
```

创建Netty的Channel向Broker发起创建消息请求，这块跟心跳注册代码差不多还是封装成一个ResponseFuture对象,服务器返回成功以后客户端可以处理这个对象。

那下面就重点看一下客户端Producer在接收到Broker成功返回的情况下的处理逻辑，它会执行Netty的NettyClientHandler的channelRead0方法执行处理逻辑，最终会执行到这段代码，这段代码我在分析心跳时候也分析过。

```java
   public void processResponseCommand(ChannelHandlerContext ctx, RemotingCommand cmd) {
        final int opaque = cmd.getOpaque();
        final ResponseFuture responseFuture = responseTable.get(opaque);
        if (responseFuture != null) {
            responseFuture.setResponseCommand(cmd);

            responseTable.remove(opaque);

            if (responseFuture.getInvokeCallback() != null) {
                executeInvokeCallback(responseFuture);
            } else {
                responseFuture.putResponse(cmd);
                responseFuture.release();
            }
        } else {
            log.warn("receive response, but not matched any request, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
            log.warn(cmd.toString());
        }
    }
```

取出ResponseFuture对象，判断如果回调InvokeCallback对象不为空执行executeInvokeCallback方法，主要分析一下这个方法做了什么：

```java
private void executeInvokeCallback(final ResponseFuture responseFuture) {
        boolean runInThisThread = false;
        ExecutorService executor = this.getCallbackExecutor();
        if (executor != null) {
            try {
                executor.submit(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            responseFuture.executeInvokeCallback();
                        } catch (Throwable e) {
                        } finally {
                            responseFuture.release();
                        }
                    }
                });
            } catch (Exception e) {
                runInThisThread = true;
            }
        } else {
            runInThisThread = true;
        }

        if (runInThisThread) {
            try {
                responseFuture.executeInvokeCallback();
            } catch (Throwable e) {
            } finally {
                responseFuture.release();
            }
        }
 }
```

主要是在线程池中执行`responseFuture.executeInvokeCallback()`,回调的方法是在异步发送中注册的。

```java
 new InvokeCallback() {
                @Override
                public void operationComplete(ResponseFuture responseFuture) {
                    long cost = System.currentTimeMillis() - beginStartTime;
                    RemotingCommand response = responseFuture.getResponseCommand();
                    if (null == sendCallback && response != null) {

                        try {
                            SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response, addr);
                            if (context != null && sendResult != null) {
                                context.setSendResult(sendResult);
                                context.getProducer().executeSendMessageHookAfter(context);
                            }
                        } catch (Throwable e) {
                        }

                        producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                        return;
                    }

                    if (response != null) {
                        try {
                            SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response, addr);
                            assert sendResult != null;
                            if (context != null) {
                                context.setSendResult(sendResult);
                                context.getProducer().executeSendMessageHookAfter(context);
                            }

                            try {
                                sendCallback.onSuccess(sendResult);
                            } catch (Throwable e) {
                            }

                            producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                        } catch (Exception e) {
                            producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                            onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                retryTimesWhenSendFailed, times, e, context, false, producer);
                        }
                    } else {
                        producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), true);
                        if (!responseFuture.isSendRequestOK()) {
                            MQClientException ex = new MQClientException("send request failed", responseFuture.getCause());
                            onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                retryTimesWhenSendFailed, times, ex, context, true, producer);
                        } else if (responseFuture.isTimeout()) {
                            MQClientException ex = new MQClientException("wait response timeout " + responseFuture.getTimeoutMillis() + "ms",
                                responseFuture.getCause());
                            onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                retryTimesWhenSendFailed, times, ex, context, true, producer);
                        } else {
                            MQClientException ex = new MQClientException("unknow reseaon", responseFuture.getCause());
                            onExceptionImpl(brokerName, msg, timeoutMillis - cost, request, sendCallback, topicPublishInfo, instance,
                                retryTimesWhenSendFailed, times, ex, context, true, producer);
                        }
                    }
                }
});
```

接着调用MQClientAPIImpl的processSendResponse方法，该方法源码如下:

```java
  private SendResult processSendResponse(
        final String brokerName,
        final Message msg,
        final RemotingCommand response,
        final String addr
    ) throws MQBrokerException, RemotingCommandException {
        SendStatus sendStatus;
        switch (response.getCode()) {
            case ResponseCode.FLUSH_DISK_TIMEOUT: {
                sendStatus = SendStatus.FLUSH_DISK_TIMEOUT;
                break;
            }
            case ResponseCode.FLUSH_SLAVE_TIMEOUT: {
                sendStatus = SendStatus.FLUSH_SLAVE_TIMEOUT;
                break;
            }
            case ResponseCode.SLAVE_NOT_AVAILABLE: {
                sendStatus = SendStatus.SLAVE_NOT_AVAILABLE;
                break;
            }
            case ResponseCode.SUCCESS: {
                sendStatus = SendStatus.SEND_OK;
                break;
            }
            default: {
                throw new MQBrokerException(response.getCode(), response.getRemark(), addr);
            }
        }

        SendMessageResponseHeader responseHeader =
            (SendMessageResponseHeader) response.decodeCommandCustomHeader(SendMessageResponseHeader.class);

        //If namespace not null , reset Topic without namespace.
        String topic = msg.getTopic();
        if (StringUtils.isNotEmpty(this.clientConfig.getNamespace())) {
            topic = NamespaceUtil.withoutNamespace(topic, this.clientConfig.getNamespace());
        }

        MessageQueue messageQueue = new MessageQueue(topic, brokerName, responseHeader.getQueueId());

        String uniqMsgId = MessageClientIDSetter.getUniqID(msg);
        if (msg instanceof MessageBatch) {
            StringBuilder sb = new StringBuilder();
            for (Message message : (MessageBatch) msg) {
                sb.append(sb.length() == 0 ? "" : ",").append(MessageClientIDSetter.getUniqID(message));
            }
            uniqMsgId = sb.toString();
        }
        SendResult sendResult = new SendResult(sendStatus,
            uniqMsgId,
            responseHeader.getMsgId(), messageQueue, responseHeader.getQueueOffset());
        sendResult.setTransactionId(responseHeader.getTransactionId());
        String regionId = response.getExtFields().get(MessageConst.PROPERTY_MSG_REGION);
        String traceOn = response.getExtFields().get(MessageConst.PROPERTY_TRACE_SWITCH);
        if (regionId == null || regionId.isEmpty()) {
            regionId = MixAll.DEFAULT_TRACE_REGION_ID;
        }
        if (traceOn != null && traceOn.equals("false")) {
            sendResult.setTraceOn(false);
        } else {
            sendResult.setTraceOn(true);
        }
        sendResult.setRegionId(regionId);
        return sendResult;
 }
```

用Switch/Case语句匹配返回的code值，如果成功返回SendStatus.SEND_OK，其他情况可能返回不同的状态码，从响应中提取出消息存储的queueId封装为MessageQueue对象，最终返回SendResult对象，然后调用sendCallback.onSuccess传入SendResult对象就回调到我们代码的处理逻辑了。

如果返回的响应为null会走到onExceptionImpl方法，该方法主要是重试发送，重试次数是timesTotal默认最大是2次，超过2次仍然没有成功会走到sendCallback.onException(e)走到我们的处理逻辑，该方法源码如下:

```java
private void onExceptionImpl(final String brokerName,
        final Message msg,
        final long timeoutMillis,
        final RemotingCommand request,
        final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
        final MQClientInstance instance,
        final int timesTotal,
        final AtomicInteger curTimes,
        final Exception e,
        final SendMessageContext context,
        final boolean needRetry,
        final DefaultMQProducerImpl producer
    ) {
        int tmp = curTimes.incrementAndGet();
        if (needRetry && tmp <= timesTotal) {
            String retryBrokerName = brokerName;
            if (topicPublishInfo != null) {
                MessageQueue mqChosen = producer.selectOneMessageQueue(topicPublishInfo, brokerName);
                retryBrokerName = mqChosen.getBrokerName();
            }
            String addr = instance.findBrokerAddressInPublish(retryBrokerName);
            request.setOpaque(RemotingCommand.createNewRequestId());
            sendMessageAsync(addr, retryBrokerName, msg, timeoutMillis, request, sendCallback, topicPublishInfo, instance,
                    timesTotal, curTimes, context, producer);
        } else {
            if (context != null) {
                context.setException(e);
                context.getProducer().executeSendMessageHookAfter(context);
            }
            try {
                sendCallback.onException(e);
            } catch (Exception ignored) {
            }
        }
    }
```



## 六、总结

这篇RocketMQ发送端源码分析从工程实践角度深入剖析了消息发送的完整链路，具有很高的技术价值。文章通过对**同步发送、异步发送、单向发送**三种模式的对比分析，揭示了RocketMQ在高性能、高可靠消息投递背后的设计哲学。

1. **架构设计精妙**：`MQClientInstance`作为客户端运行时管家，采用单例模式统一管理网络连接、路由信息、心跳检测等公共资源，体现了"资源共享、职责分离"的设计思想。
2. **线程模型优化**：异步发送通过`defaultAsyncSenderExecutor`线程池实现非阻塞操作，既保证了性能又不失可靠性；信号量控制(`semaphoreAsync`)有效防止了过度并发导致的系统过载。
3. **容错机制完善**：同步发送的自动重试策略机制，共同构建了生产级的高可用保障体系。
4. **网络通信高效**：基于Netty的`RemotingClient`封装了不同通信模式，`invokeSync`阻塞等待、`invokeAsync`回调通知、`invokeOneway`直接发送，满足了不同场景下的性能要求。
