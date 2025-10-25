# RocketMQ源码详解(消费端启动流程)

## 一、前言

在分布式消息系统中，**消费端的启动流程**往往决定了系统能否稳定、高效地处理消息。消费者不仅仅是简单地“连上 Broker 然后收消息”，而是涉及到 **客户端初始化、负载均衡、队列分配、消息拉取线程、消费服务线程** 等一系列环节。

理解消费端的启动流程，有助于我们掌握 RocketMQ **如何实现高可用的消费机制**，以及在遇到消息堆积、消费延迟、负载不均等问题时，能快速定位到具体环节。

在这篇文章中，我们就从源码角度梳理 **RocketMQ 消费端启动流程**，看看一个消费者从 `start()` 方法开始，到真正能够消费消息，中间都经历了哪些关键步骤。

## 二、代码示例

一个消费者启动往往都是通过下面这样的代码实现的：

```java
public class MyConsumer {

    public static void main(String[] args) throws Exception {
        // 1. 创建消费者，指定消费者组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("my_consumer_group");

        // 2. 设置 NameServer 地址
        consumer.setNamesrvAddr("127.0.0.1:9876");

        // 3. 订阅 topic
        consumer.subscribe("eooi2156", "*");  // * 表示订阅所有 tag

        // 4. 注册消息监听器(并发消费消息)
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    System.out.printf("收到消息：msgId=%s, topic=%s, body=%s%n",
                            msg.getMsgId(),
                            msg.getTopic(),
                            new String(msg.getBody()));
                }
                // 5. 标记消息已成功消费
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        //  注册消息监听器(顺序消费消息)
        consumer.registerMessageListener(new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                return null;
            }
        });

        // 6. 启动消费者
        consumer.start();
        System.out.println("消费者已启动！");
    }
}
```

这里消费端消费方式主要有两种：

- **并发消费**

     消费者端启动 **多个线程**，并发地处理不同队列中的消息，**同一个队列内**的消息可以被**多线程并行消费**（不保证顺序）。主要特点是性能高、速度快。

- **顺序消费**

     RocketMQ 保证**一个队列内的消息**会被**一个线程顺序消费**。只有前一条消息消费成功，才会取下一条。主要特点是保证同一业务的消息按发送顺序被消费。


以一张图片显示并发消费和顺序消费的流程，帮助大家加深印象:

![deepseek_mermaid_20251025_8313aa.png](..%2Fimg%2Fdeepseek_mermaid_20251025_8313aa.png)

本文就以这两个不同消费逻辑为主线一步步分析消费者启动流程以及里面的一些组件的处理逻辑源码。

## 三、启动流程源码分析

初始化一个DefaultMQPushConsumer对象，传入消费者组名称，这里面调用一个重载的构造函数代码如下:

```java
 public DefaultMQPushConsumer(final String consumerGroup) {
        this(null, consumerGroup, null, new AllocateMessageQueueAveragely());
 }
```

- 主要是初始化成员变量AllocateMessageQueueStrategy allocateMessageQueueStrategy，它的实现默认是AllocateMessageQueueAveragely,它的主要作用是在消费者 **Rebalance（重新分配队列）** 时，把一个 Topic 下的多个队列 **尽量均匀地分配** 给同一个 Consumer Group 里的不同消费者实例。
- DefaultMQPushConsumerImpl defaultMQPushConsumerImpl，它主要负责消息拉取和消息消费，消费者启动等逻辑。
- String consumerGroup,它是消费者组名称。

接着设置NameSrv的地址、订阅的Topic主题以及是否订阅所有Tag，注册消息监听器，主要类型有是并发消费还是顺序消费，最终调用DefaultMQPushConsumer的start方法启动消费者端。我们来看看start方法的签名。

```java
public synchronized void start() throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:

                if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
                    this.defaultMQPushConsumer.changeInstanceNameToPID();
                }

                this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);

                this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
                this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
                this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
                this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

                this.pullAPIWrapper = new PullAPIWrapper(
                    mQClientFactory,
                    this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
                this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

                if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                    this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
                } else {
                    switch (this.defaultMQPushConsumer.getMessageModel()) {
                        case BROADCASTING:
                            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                            break;
                        case CLUSTERING:
                            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                            break;
                        default:
                            break;
                    }
                    this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
                }
                this.offsetStore.load();

                if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                    this.consumeOrderly = true;
                    this.consumeMessageService =
                        new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
                } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                    this.consumeOrderly = false;
                    this.consumeMessageService =
                        new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
                }

                this.consumeMessageService.start();

                boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;
                    this.consumeMessageService.shutdown(defaultMQPushConsumer.getAwaitTerminationMillisWhenShutdown());
                    throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
                }
                mQClientFactory.start();
                log.info("the consumer [{}] start OK.", this.defaultMQPushConsumer.getConsumerGroup());
                this.serviceState = ServiceState.RUNNING;
                break;
            default:
                break;
        }
        this.updateTopicSubscribeInfoWhenSubscriptionChanged();
        this.mQClientFactory.checkClientInBroker();
        this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
        this.mQClientFactory.rebalanceImmediately();
 }
```

初始化MQClientInstance对象，设置RebalancePushImpl成员属性，主要包括消费者组名称、客户端负责均衡策略、创建PullAPIWrapper对象，它主要用于Broker拉取消息，判断RocketMQ 消费者消息模式MessageModel ，它是一个枚举类，包含**CLUSTERING**、**BROADCASTING**两种模式。

- **CLUSTERING（集群模式，默认）**，表示同一个 ConsumerGroup 下的多个消费者会 **均摊**（负载均衡）消费消息,通常用来做并行处理，比如日志处理、订单处理等，保证消息只被 **一个消费者实例** 消费一次。
- **BROADCASTING（广播模式）**,表示同一个 ConsumerGroup 下的 **每个消费者** 都会收到并消费一份完整的消息，比如配置刷新、通知广播、缓存更新。

根据MessageModel不同创建不同的OffsetStore对象，默认的MessageModel是CLUSTERING那么创建的是**RemoteBrokerOffsetStore**对象,它的作用是把消费进度存储到Broker上，而不是本地磁盘。优点：多个消费者进程可以共享消费进度（例如重启或迁移）缺点：每次读写需要网络访问，延迟略高。

然后把它设置到对象DefaultMQPushConsumer成员变量上。接着判断监听器如果是MessageListenerOrderly代表是顺序消费，那么新创建一个ConsumeMessageOrderlyService对象，用于处理顺序消费逻辑，如果是MessageListenerConcurrently代表并发消费，那么创建一个ConsumeMessageConcurrentlyService对象，用于处理并发消费逻辑。

调用ConsumeMessageService的start方法，然后调用了MQClientInstance的start方法，这个方法其实在上篇文章发送者已经分析过了，现在主要重点分析这个方法里面`PullMessageService`的start方法和`RebalanceService`的start方法。

## 四、RebalanceService的start方法源码分析

`RebalanceService`间接实现了Runnable接口，会调用其run方法，该方法源码如下:

```java
public void run() {
        log.info(this.getServiceName() + " service started");

        while (!this.isStopped()) {
            this.waitForRunning(waitInterval);
            this.mqClientFactory.doRebalance();
        }

        log.info(this.getServiceName() + " service end");
}
```

是个死循环每隔20秒执行一下负载均衡算法，把Broker端队列均匀分配给消费者客户端，调用的是`MQClientInstance`的doRebalance方法，该方法经过一步步调用会调用到对象`RebalancePushImpl`的rebalanceByTopic方法，该方法源码如下:

```java
 private void rebalanceByTopic(final String topic, final boolean isOrder) {
        switch (messageModel) {
            case BROADCASTING: {
                Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
                if (mqSet != null) {
                    boolean changed = this.updateProcessQueueTableInRebalance(topic, mqSet, isOrder);
                    if (changed) {
                        this.messageQueueChanged(topic, mqSet, mqSet);
                        log.info("messageQueueChanged {} {} {} {}",
                            consumerGroup,
                            topic,
                            mqSet,
                            mqSet);
                    }
                } else {
                    log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
                }
                break;
            }
            case CLUSTERING: {
                Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
                List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);
                if (null == mqSet) {
                    if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                        log.warn("doRebalance, {}, but the topic[{}] not exist.", consumerGroup, topic);
                    }
                }

                if (null == cidAll) {
                    log.warn("doRebalance, {} {}, get consumer id list failed", consumerGroup, topic);
                }

                if (mqSet != null && cidAll != null) {
                    List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
                    mqAll.addAll(mqSet);

                    Collections.sort(mqAll);
                    Collections.sort(cidAll);

                    AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;

                    List<MessageQueue> allocateResult = null;
                    try {
                        allocateResult = strategy.allocate(
                            this.consumerGroup,
                            this.mQClientFactory.getClientId(),
                            mqAll,
                            cidAll);
                    } catch (Throwable e) {
                        log.error("AllocateMessageQueueStrategy.allocate Exception. allocateMessageQueueStrategyName={}", strategy.getName(),
                            e);
                        return;
                    }
                    Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
                    if (allocateResult != null) {
                        allocateResultSet.addAll(allocateResult);
                    }
                    boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
                    if (changed) {
                        log.info(
                            "rebalanced result changed. allocateMessageQueueStrategyName={}, group={}, topic={}, clientId={}, mqAllSize={}, cidAllSize={}, rebalanceResultSize={}, rebalanceResultSet={}",
                            strategy.getName(), consumerGroup, topic, this.mQClientFactory.getClientId(), mqSet.size(), cidAll.size(),
                            allocateResultSet.size(), allocateResultSet);
                        this.messageQueueChanged(topic, mqSet, allocateResultSet);
                    }
                }
                break;
            }
            default:
                break;
        }
  }
```

我们默认的messageModel模式是**CLUSTERING**所以我们来重点看一下这个分支的逻辑。

获取主题的所有队列集合，执行方法`this.mQClientFactory.findConsumerIdList(topic, consumerGroup);`获取所有消费者端id集合cidAll，调用AllocateMessageQueueStrategy对象allocate计算出当前客户端应该分配当前主题的队列集合，关于这个算法我会专门在以后的文章详细分析一下。

接着调用updateProcessQueueTableInRebalance方法，该方法源码如下：

```java
  private boolean updateProcessQueueTableInRebalance(final String topic, final Set<MessageQueue> mqSet,
        final boolean isOrder) {
        boolean changed = false;

        Iterator<Entry<MessageQueue, ProcessQueue>> it = this.processQueueTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<MessageQueue, ProcessQueue> next = it.next();
            MessageQueue mq = next.getKey();
            ProcessQueue pq = next.getValue();

            if (mq.getTopic().equals(topic)) {
                if (!mqSet.contains(mq)) {
                    pq.setDropped(true);
                    if (this.removeUnnecessaryMessageQueue(mq, pq)) {
                        it.remove();
                        changed = true;
                        log.info("doRebalance, {}, remove unnecessary mq, {}", consumerGroup, mq);
                    }
                } else if (pq.isPullExpired()) {
                    switch (this.consumeType()) {
                        case CONSUME_ACTIVELY:
                            break;
                        case CONSUME_PASSIVELY:
                            pq.setDropped(true);
                            if (this.removeUnnecessaryMessageQueue(mq, pq)) {
                                it.remove();
                                changed = true;
                            }
                            break;
                        default:
                            break;
                    }
                }
            }
        }
        List<PullRequest> pullRequestList = new ArrayList<PullRequest>();
        for (MessageQueue mq : mqSet) {
            if (!this.processQueueTable.containsKey(mq)) {
                if (isOrder && !this.lock(mq)) {
                    log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
                    continue;
                }
                this.removeDirtyOffset(mq);
                ProcessQueue pq = new ProcessQueue();
                pq.setLocked(true);
                long nextOffset = -1L;
                try {
                    nextOffset = this.computePullFromWhereWithException(mq);
                } catch (Exception e) {
                    log.info("doRebalance, {}, compute offset failed, {}", consumerGroup, mq);
                    continue;
                }

                if (nextOffset >= 0) {
                    ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
                    if (pre != null) {
                        log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
                    } else {
                        log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
                        PullRequest pullRequest = new PullRequest();
                        pullRequest.setConsumerGroup(consumerGroup);
                        pullRequest.setNextOffset(nextOffset);
                        pullRequest.setMessageQueue(mq);
                        pullRequest.setProcessQueue(pq);
                        pullRequestList.add(pullRequest);
                        changed = true;
                    }
                } else {
                    log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
                }
            }
        }

        this.dispatchPullRequest(pullRequestList);

        return changed;
    }
```

遍历processQueueTable本地Map判断队列是否在重新分配的Set<MessageQueue>集合中，如果不在说明客户端负载均衡可能重新分配了，可能新增了客户端这个队列分配给其他客户端了把ProcessQueue的dropped属性设置为true表示是不能在消费消息后续会判断这个标识如果为true就不会消费，然后把它冲Map中移除设置changed = true，表示发生了变化，这是其中之一的情况。

还有一种情况判断`pq.isPullExpired()`它的处理逻辑是判断当前时间减去上次拉取时间如果大于120秒说明这个队列长时间没进行拉取了也移除掉。

当processQueueTable本地Map什么都没有时候会遍历mqSet这些都是新的队列，如果是顺序队列需要去Broker申请锁，清除历史的offset,重新计算offset值，创建ProcessQueue对象，把当前遍历的MessageQueue对象，以MessageQueue为key、ProcessQueue为value放入到本地Map中，构造PullRequest对象，设置主要属性有消费者组名称、消费偏移量、要消费的队列、本地队列然后添加到集合中，集合是`List<PullRequest> pullRequestList`调用dispatchPullRequest方法，该方法源码如下:

```java
 public void dispatchPullRequest(List<PullRequest> pullRequestList) {
        for (PullRequest pullRequest : pullRequestList) {
            this.defaultMQPushConsumerImpl.executePullRequestImmediately(pullRequest);
            log.info("doRebalance, {}, add a new pull request {}", consumerGroup, pullRequest);
        }
 }
```

又调用了DefaultMQPushConsumerImpl的executePullRequestImmediately方法，最终调用到`PullMessageService`的executePullRequestImmediately方法，把`PullRequest`放到`LinkedBlockingQueue<PullRequest> pullRequestQueue`成员变量上，它是一个阻塞的有界队列，后续线程会从这里取进行消费。

`PullMessageService`是消费者，它从LinkedBlockingQueue队列中取出`PullRequest`对象然后进行业务处理去Broker获取数据，它也是一个线程，它的run方法如下:

```java
public void run() {
    log.info(this.getServiceName() + " service started");

    while (!this.isStopped()) {
        try {
            PullRequest pullRequest = this.pullRequestQueue.take();
            this.pullMessage(pullRequest);
        } catch (InterruptedException ignored) {
        } catch (Exception e) {
            log.error("Pull Message Service Run Method exception", e);
        }
    }

    log.info(this.getServiceName() + " service end");
}
```

调用pullMessage方法，该方法源码如下:

```java
private void pullMessage(final PullRequest pullRequest) {
    final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
    if (consumer != null) {
        DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
        impl.pullMessage(pullRequest);
    } else {
        log.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
    }
}
```

调用`DefaultMQPushConsumerImpl`对象的pullMessage方法，该方法源码如下:

```java
public void pullMessage(final PullRequest pullRequest) {
        final ProcessQueue processQueue = pullRequest.getProcessQueue();
        if (processQueue.isDropped()) {
            log.info("the pull request[{}] is dropped.", pullRequest.toString());
            return;
        }
        pullRequest.getProcessQueue().setLastPullTimestamp(System.currentTimeMillis());
       
        long cachedMessageCount = processQueue.getMsgCount().get();
        long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);

        if (cachedMessageCount > this.defaultMQPushConsumer.getPullThresholdForQueue()) {
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL);
            if ((queueFlowControlTimes++ % 1000) == 0) {
                log.warn(
                    "the cached message count exceeds the threshold {}, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                    this.defaultMQPushConsumer.getPullThresholdForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
            }
            return;
        }

        if (cachedMessageSizeInMiB > this.defaultMQPushConsumer.getPullThresholdSizeForQueue()) {
            this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL);
            if ((queueFlowControlTimes++ % 1000) == 0) {
                log.warn(
                    "the cached message size exceeds the threshold {} MiB, so do flow control, minOffset={}, maxOffset={}, count={}, size={} MiB, pullRequest={}, flowControlTimes={}",
                    this.defaultMQPushConsumer.getPullThresholdSizeForQueue(), processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), cachedMessageCount, cachedMessageSizeInMiB, pullRequest, queueFlowControlTimes);
            }
            return;
        }

        if (!this.consumeOrderly) {
            if (processQueue.getMaxSpan() > this.defaultMQPushConsumer.getConsumeConcurrentlyMaxSpan()) {
                this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL);
                if ((queueMaxSpanFlowControlTimes++ % 1000) == 0) {
                    log.warn(
                        "the queue's messages, span too long, so do flow control, minOffset={}, maxOffset={}, maxSpan={}, pullRequest={}, flowControlTimes={}",
                        processQueue.getMsgTreeMap().firstKey(), processQueue.getMsgTreeMap().lastKey(), processQueue.getMaxSpan(),
                        pullRequest, queueMaxSpanFlowControlTimes);
                }
                return;
            }
        } else {
            if (processQueue.isLocked()) {
                if (!pullRequest.isPreviouslyLocked()) {
                    long offset = -1L;
                    try {
                        offset = this.rebalanceImpl.computePullFromWhereWithException(pullRequest.getMessageQueue());
                    } catch (Exception e) {
                        this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
                        log.error("Failed to compute pull offset, pullResult: {}", pullRequest, e);
                        return;
                    }
                    boolean brokerBusy = offset < pullRequest.getNextOffset();
                    log.info("the first time to pull message, so fix offset from broker. pullRequest: {} NewOffset: {} brokerBusy: {}",
                        pullRequest, offset, brokerBusy);
                    if (brokerBusy) {
                        log.info("[NOTIFYME]the first time to pull message, but pull request offset larger than broker consume offset. pullRequest: {} NewOffset: {}",
                            pullRequest, offset);
                    }

                    pullRequest.setPreviouslyLocked(true);
                    pullRequest.setNextOffset(offset);
                }
            } else {
                this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
                log.info("pull message later because not locked in broker, {}", pullRequest);
                return;
            }
        }

        final SubscriptionData subscriptionData = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
        if (null == subscriptionData) {
            this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
            log.warn("find the consumer's subscription failed, {}", pullRequest);
            return;
        }

        final long beginTimestamp = System.currentTimeMillis();

        PullCallback pullCallback = new PullCallback() {
            @Override
            public void onSuccess(PullResult pullResult) {
                if (pullResult != null) {
                    pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                        subscriptionData);

                    switch (pullResult.getPullStatus()) {
                        case FOUND:
                            long prevRequestOffset = pullRequest.getNextOffset();
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                            long pullRT = System.currentTimeMillis() - beginTimestamp;
                            DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                                pullRequest.getMessageQueue().getTopic(), pullRT);

                            long firstMsgOffset = Long.MAX_VALUE;
                            if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                                DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            } else {
                                firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();

                                DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                                    pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());

                                boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                                DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                                    pullResult.getMsgFoundList(),
                                    processQueue,
                                    pullRequest.getMessageQueue(),
                                    dispatchToConsume);

                                if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                                    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                        DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                                } else {
                                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                                }
                            }

                            if (pullResult.getNextBeginOffset() < prevRequestOffset
                                || firstMsgOffset < prevRequestOffset) {
                                log.warn(
                                    "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
                                    pullResult.getNextBeginOffset(),
                                    firstMsgOffset,
                                    prevRequestOffset);
                            }

                            break;
                        case NO_NEW_MSG:
                        case NO_MATCHED_MSG:
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                            DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            break;
                        case OFFSET_ILLEGAL:
                            log.warn("the pull request offset illegal, {} {}",
                                pullRequest.toString(), pullResult.toString());
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                            pullRequest.getProcessQueue().setDropped(true);
                            DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {

                                @Override
                                public void run() {
                                    try {
                                        DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
                                            pullRequest.getNextOffset(), false);

                                        DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());

                                        DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());

                                        log.warn("fix the pull request offset, {}", pullRequest);
                                    } catch (Throwable e) {
                                        log.error("executeTaskLater Exception", e);
                                    }
                                }
                            }, 10000);
                            break;
                        default:
                            break;
                    }
                }
            }

            @Override
            public void onException(Throwable e) {
                if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    log.warn("execute the pull request exception", e);
                }

                if (e instanceof MQBrokerException && ((MQBrokerException) e).getResponseCode() == ResponseCode.FLOW_CONTROL) {
                    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_BROKER_FLOW_CONTROL);
                } else {
                    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
                }
            }
        };

        boolean commitOffsetEnable = false;
        long commitOffsetValue = 0L;
        if (MessageModel.CLUSTERING == this.defaultMQPushConsumer.getMessageModel()) {
            commitOffsetValue = this.offsetStore.readOffset(pullRequest.getMessageQueue(), ReadOffsetType.READ_FROM_MEMORY);
            if (commitOffsetValue > 0) {
                commitOffsetEnable = true;
            }
        }

        String subExpression = null;
        boolean classFilter = false;
        SubscriptionData sd = this.rebalanceImpl.getSubscriptionInner().get(pullRequest.getMessageQueue().getTopic());
        if (sd != null) {
            if (this.defaultMQPushConsumer.isPostSubscriptionWhenPull() && !sd.isClassFilterMode()) {
                subExpression = sd.getSubString();
            }

            classFilter = sd.isClassFilterMode();
        }

        int sysFlag = PullSysFlag.buildSysFlag(
            commitOffsetEnable, // commitOffset
            true, // suspend
            subExpression != null, // subscription
            classFilter // class filter
        );
        try {
            this.pullAPIWrapper.pullKernelImpl(
                pullRequest.getMessageQueue(),
                subExpression,
                subscriptionData.getExpressionType(),
                subscriptionData.getSubVersion(),
                pullRequest.getNextOffset(),
                this.defaultMQPushConsumer.getPullBatchSize(),
                sysFlag,
                commitOffsetValue,
                BROKER_SUSPEND_MAX_TIME_MILLIS,
                CONSUMER_TIMEOUT_MILLIS_WHEN_SUSPEND,
                CommunicationMode.ASYNC,
                pullCallback
            );
        } catch (Exception e) {
            log.error("pullKernelImpl exception", e);
            this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
        }
    }
```

判断本地`ProcessQueue`对象dropped如果为true，表示该队列不属于这个消费者直接return,设置`ProcessQueue`的lastPullTimestamp为当前时间。

```java
long cachedMessageCount = processQueue.getMsgCount().get();
long cachedMessageSizeInMiB = processQueue.getMsgSize().get() / (1024 * 1024);
```

获取ProcessQueue当前缓存量，主要用于**控制消息拉取速度，避免 ProcessQueue 缓存过大导致 OOM 或消费压力过大**。

- cachedMessageCount：ProcessQueue 中已经缓存的消息条数。
- cachedMessageSizeInMiB:缓存消息总大小（MB）。

判断cachedMessageCount 如果大于`this.defaultMQPushConsumer.getPullThresholdForQueue()`即1000的时候，不立即拉取新的消息,调用 `executePullRequestLater` 将 `PullRequest` 延迟一定时间再执行（这里是 `PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL`=50）,每 1000 次触发时，打印一次警告日志，提示该队列发生了流控。

判断cachedMessageSizeInMiB 如果大于`this.defaultMQPushConsumer.getPullThresholdSizeForQueue()`即100M的时候，不立即拉取新的消息，调用executePullRequestLater将 `PullRequest` 延迟一定时间再执行（这里是 `PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL`=50）,每 1000 次触发时，打印一次警告日志，提示该队列发生了流控,逻辑和消息数阈值类似，只是从内存大小角度控制。

判断`ProcessQueue` 中 **最大消息 offset - 最小消息 offset** 的跨度如果大于消费者配置的 **队列最大跨度阈值(2000)**,说明队列积压严重，消费可能跟不上生产,不立即拉取新的消息，调用executePullRequestLater将 `PullRequest` 延迟一定时间再执行（这里是 `PULL_TIME_DELAY_MILLS_WHEN_CACHE_FLOW_CONTROL`=50）,每 1000 次触发时，打印一次警告日志，提示该队列发生了流控,逻辑和消息数阈值类似。

如果是顺序消费要求每个队列在集群中只允许一个消费者实例消费，`processQueue.isLocked()` 表示本消费端对这个队列是否获取到 **分布式锁**,**如果没锁**，则不拉消息，延迟执行 `executePullRequestLater`防止多个消费者同时消费同一队列导致顺序混乱。

接着调用`this.pullAPIWrapper.pullKernelImpl`方法从Broker拉取消息，该方法源码如下:

```java
public PullResult pullKernelImpl(
        final MessageQueue mq,
        final String subExpression,
        final String expressionType,
        final long subVersion,
        final long offset,
        final int maxNums,
        final int sysFlag,
        final long commitOffset,
        final long brokerSuspendMaxTimeMillis,
        final long timeoutMillis,
        final CommunicationMode communicationMode,
        final PullCallback pullCallback
    ) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
            .................................................
 
            PullMessageRequestHeader requestHeader = new PullMessageRequestHeader();
            requestHeader.setConsumerGroup(this.consumerGroup);
            requestHeader.setTopic(mq.getTopic());
            requestHeader.setQueueId(mq.getQueueId());
            requestHeader.setQueueOffset(offset);
            requestHeader.setMaxMsgNums(maxNums);
            requestHeader.setSysFlag(sysFlagInner);
            requestHeader.setCommitOffset(commitOffset);
            requestHeader.setSuspendTimeoutMillis(brokerSuspendMaxTimeMillis);
            requestHeader.setSubscription(subExpression);
            requestHeader.setSubVersion(subVersion);
            requestHeader.setExpressionType(expressionType);
            requestHeader.setBname(mq.getBrokerName());

            String brokerAddr = findBrokerResult.getBrokerAddr();
            if (PullSysFlag.hasClassFilterFlag(sysFlagInner)) {
                brokerAddr = computePullFromWhichFilterServer(mq.getTopic(), brokerAddr);
            }

            PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(
                brokerAddr,
                requestHeader,
                timeoutMillis,
                communicationMode,
                pullCallback);

            return pullResult;
        }

        throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
 }
```

这块异步去Broker中取得拉取数据获取PullResult对象，然后回调`PullCallback`对象的onSuccess方法，这个方法在上面贴出过，我再贴出来一下:

```java
PullCallback pullCallback = new PullCallback() {
            @Override
            public void onSuccess(PullResult pullResult) {
                if (pullResult != null) {
                    pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult,
                        subscriptionData);

                    switch (pullResult.getPullStatus()) {
                        case FOUND:
                            long prevRequestOffset = pullRequest.getNextOffset();
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                            long pullRT = System.currentTimeMillis() - beginTimestamp;
                            DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                                pullRequest.getMessageQueue().getTopic(), pullRT);

                            long firstMsgOffset = Long.MAX_VALUE;
                            if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                                DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            } else {
                                firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();

                                DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                                    pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());

                                boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                                DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                                    pullResult.getMsgFoundList(),
                                    processQueue,
                                    pullRequest.getMessageQueue(),
                                    dispatchToConsume);

                                if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                                    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                        DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                                } else {
                                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                                }
                            }

                            if (pullResult.getNextBeginOffset() < prevRequestOffset
                                || firstMsgOffset < prevRequestOffset) {
                                log.warn(
                                    "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
                                    pullResult.getNextBeginOffset(),
                                    firstMsgOffset,
                                    prevRequestOffset);
                            }

                            break;
                        case NO_NEW_MSG:
                        case NO_MATCHED_MSG:
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                            DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            break;
                        case OFFSET_ILLEGAL:
                            log.warn("the pull request offset illegal, {} {}",
                                pullRequest.toString(), pullResult.toString());
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                            pullRequest.getProcessQueue().setDropped(true);
                            DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {

                                @Override
                                public void run() {
                                    try {
                                        DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(),
                                            pullRequest.getNextOffset(), false);

                                        DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());

                                        DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());

                                        log.warn("fix the pull request offset, {}", pullRequest);
                                    } catch (Throwable e) {
                                        log.error("executeTaskLater Exception", e);
                                    }
                                }
                            }, 10000);
                            break;
                        default:
                            break;
                    }
                }
            }

            @Override
            public void onException(Throwable e) {
                if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    log.warn("execute the pull request exception", e);
                }

                if (e instanceof MQBrokerException && ((MQBrokerException) e).getResponseCode() == ResponseCode.FLOW_CONTROL) {
                    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_BROKER_FLOW_CONTROL);
                } else {
                    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, pullTimeDelayMillsWhenException);
                }
            }
        };
```

保存上一次请求的offset(prevRequestOffset),将 `pullRequest.nextOffset` 更新为 `pullResult.nextBeginOffset`（broker 返回的下一次起始拉取位置）。

**消息为空情况**

- 如果 `pullResult.getMsgFoundList()` 是空的 说明 broker 返回 FOUND 但没消息
- 直接 **立即重新发起拉取请求**（`executePullRequestImmediately`）

**消息非空情况**

- 取第一条消息的队列偏移量（`firstMsgOffset`）
- 将消息放入 **ProcessQueue**（`processQueue.putMessage(...)`）,主要设置最新偏移量和最大偏移量可以计算出跨度、条数、以及内存数量以方便做流控管理
- 调用 **ConsumeMessageService** 提交消费任务（`submitConsumeRequest`）
- 判断是否需要延迟下一次拉取（`pullInterval` 配置），有则延迟，没有则立即拉取下一批

**数据一致性校验**

- 如果 broker 返回的 `nextBeginOffset` 比上次请求 offset 小
- 或者返回的消息 offset 比上次请求 offset 小
- 说明可能有 bug（数据错乱），打印警告日志

如果是NO_NEW_MSG / NO_MATCHED_MSG（没有新消息 / 没有匹配消息），则更新下次拉取 offset，立即再发起下一次拉取。

**onException 逻辑**

如果拉取过程发生异常：

- 如果不是重试队列（`RETRY_GROUP_TOPIC`），打印警告日志
- 如果是 broker 流控（`ResponseCode.FLOW_CONTROL`），延迟一段时间后再发起拉取请求
- 其他异常，使用默认的异常延迟时间重试



## 4.1 并发消费逻辑

下面我们就来详细分析一下消费者线程处理逻辑，对于并发消费回调用`ConsumeMessageConcurrentlyService`的submitConsumeRequest方法，该方法源码如下:

```java
public void submitConsumeRequest(
        final List<MessageExt> msgs,
        final ProcessQueue processQueue,
        final MessageQueue messageQueue,
        final boolean dispatchToConsume) {
        final int consumeBatchSize = this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
        if (msgs.size() <= consumeBatchSize) {
            ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
            try {
                this.consumeExecutor.submit(consumeRequest);
            } catch (RejectedExecutionException e) {
                this.submitConsumeRequestLater(consumeRequest);
            }
        } else {
            for (int total = 0; total < msgs.size(); ) {
                List<MessageExt> msgThis = new ArrayList<MessageExt>(consumeBatchSize);
                for (int i = 0; i < consumeBatchSize; i++, total++) {
                    if (total < msgs.size()) {
                        msgThis.add(msgs.get(total));
                    } else {
                        break;
                    }
                }

                ConsumeRequest consumeRequest = new ConsumeRequest(msgThis, processQueue, messageQueue);
                try {
                    this.consumeExecutor.submit(consumeRequest);
                } catch (RejectedExecutionException e) {
                    for (; total < msgs.size(); total++) {
                        msgThis.add(msgs.get(total));
                    }

                    this.submitConsumeRequestLater(consumeRequest);
                }
            }
        }
    }
```

主要是封装为ConsumeRequest对象传入已经查询到的消费信息集合、本地队列ProcessQueue、Broker队列MessageQueue,然后提交到线程池中去执行，ConsumeRequest实现Runnable接口，主要看它的run方法：

```java
public void run() {
            if (this.processQueue.isDropped()) {
                log.info("the message queue not be able to consume, because it's dropped. group={} {}", ConsumeMessageConcurrentlyService.this.consumerGroup, this.messageQueue);
                return;
            }

            MessageListenerConcurrently listener = ConsumeMessageConcurrentlyService.this.messageListener;
            ConsumeConcurrentlyContext context = new ConsumeConcurrentlyContext(messageQueue);
            ConsumeConcurrentlyStatus status = null;
            defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());

            ConsumeMessageContext consumeMessageContext = null;
            if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                consumeMessageContext = new ConsumeMessageContext();
                consumeMessageContext.setNamespace(defaultMQPushConsumer.getNamespace());
                consumeMessageContext.setConsumerGroup(defaultMQPushConsumer.getConsumerGroup());
                consumeMessageContext.setProps(new HashMap<String, String>());
                consumeMessageContext.setMq(messageQueue);
                consumeMessageContext.setMsgList(msgs);
                consumeMessageContext.setSuccess(false);
                ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
            }

            long beginTimestamp = System.currentTimeMillis();
            boolean hasException = false;
            ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
            try {
                if (msgs != null && !msgs.isEmpty()) {
                    for (MessageExt msg : msgs) {
                        MessageAccessor.setConsumeStartTimeStamp(msg, String.valueOf(System.currentTimeMillis()));
                    }
                }
                status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);
            } catch (Throwable e) {
                log.warn(String.format("consumeMessage exception: %s Group: %s Msgs: %s MQ: %s",
                    RemotingHelper.exceptionSimpleDesc(e),
                    ConsumeMessageConcurrentlyService.this.consumerGroup,
                    msgs,
                    messageQueue), e);
                hasException = true;
            }
            long consumeRT = System.currentTimeMillis() - beginTimestamp;
            if (null == status) {
                if (hasException) {
                    returnType = ConsumeReturnType.EXCEPTION;
                } else {
                    returnType = ConsumeReturnType.RETURNNULL;
                }
            } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
                returnType = ConsumeReturnType.TIME_OUT;
            } else if (ConsumeConcurrentlyStatus.RECONSUME_LATER == status) {
                returnType = ConsumeReturnType.FAILED;
            } else if (ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status) {
                returnType = ConsumeReturnType.SUCCESS;
            }

            if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
            }

            if (null == status) {
                log.warn("consumeMessage return null, Group: {} Msgs: {} MQ: {}",
                    ConsumeMessageConcurrentlyService.this.consumerGroup,
                    msgs,
                    messageQueue);
                status = ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }

            if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                consumeMessageContext.setStatus(status.toString());
                consumeMessageContext.setSuccess(ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status);
                consumeMessageContext.setAccessChannel(defaultMQPushConsumer.getAccessChannel());
                ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
            }

            ConsumeMessageConcurrentlyService.this.getConsumerStatsManager()
                .incConsumeRT(ConsumeMessageConcurrentlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);

            if (!processQueue.isDropped()) {
                ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
            } else {
                log.warn("processQueue is dropped without process consume result. messageQueue={}, msgs={}", messageQueue, msgs);
            }
        }
```

如果ProcessQueue的dropped是true表示队列已经被分给其他消费者了，直接return,接着获取用户设置的监听器,封装ConsumeConcurrentlyContext上下文对象例如 `messageQueue`，用于传给监听器, ConsumeConcurrentlyStatus status是消费结果，用户返回的值。

```java
long beginTimestamp = System.currentTimeMillis();
boolean hasException = false;
ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;

try {
    if (msgs != null && !msgs.isEmpty()) {
        for (MessageExt msg : msgs) {
            MessageAccessor.setConsumeStartTimeStamp(msg, String.valueOf(System.currentTimeMillis()));
        }
    }
    status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);
} catch (Throwable e) {
    log.warn(String.format("consumeMessage exception: %s Group: %s Msgs: %s MQ: %s",
        RemotingHelper.exceptionSimpleDesc(e),
        ConsumeMessageConcurrentlyService.this.consumerGroup,
        msgs,
        messageQueue), e);
    hasException = true;
}

```

这段代码的逻辑是设置每条消息的消费开始时间,调用用户实现的 `consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context)`。如果抛出异常，catch 住，并记录 `hasException=true`。

如果 `processQueue` 没丢弃，调用 `processConsumeResult`它的源码如下:

```java
public void processConsumeResult(
        final ConsumeConcurrentlyStatus status,
        final ConsumeConcurrentlyContext context,
        final ConsumeRequest consumeRequest
    ) {
        int ackIndex = context.getAckIndex();

        if (consumeRequest.getMsgs().isEmpty())
            return;

        switch (status) {
            case CONSUME_SUCCESS:
                if (ackIndex >= consumeRequest.getMsgs().size()) {
                    ackIndex = consumeRequest.getMsgs().size() - 1;
                }
                int ok = ackIndex + 1;
                int failed = consumeRequest.getMsgs().size() - ok;
                this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), ok);
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), failed);
                break;
            case RECONSUME_LATER:
                ackIndex = -1;
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(),
                    consumeRequest.getMsgs().size());
                break;
            default:
                break;
        }

        switch (this.defaultMQPushConsumer.getMessageModel()) {
            case BROADCASTING:
                for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                    MessageExt msg = consumeRequest.getMsgs().get(i);
                    log.warn("BROADCASTING, the message consume failed, drop it, {}", msg.toString());
                }
                break;
            case CLUSTERING:
                List<MessageExt> msgBackFailed = new ArrayList<MessageExt>(consumeRequest.getMsgs().size());
                for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                    MessageExt msg = consumeRequest.getMsgs().get(i);
                    boolean result = this.sendMessageBack(msg, context);
                    if (!result) {
                        msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                        msgBackFailed.add(msg);
                    }
                }

                if (!msgBackFailed.isEmpty()) {
                    consumeRequest.getMsgs().removeAll(msgBackFailed);

                    this.submitConsumeRequestLater(msgBackFailed, consumeRequest.getProcessQueue(), consumeRequest.getMessageQueue());
                }
                break;
            default:
                break;
        }

        long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
        if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
            this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
        }
    }

```

如果业务返回**`CONSUME_SUCCESS`**：

- 计算成功消息数 `ok`，失败消息数 `failed`。
- 更新消费成功/失败的 TPS 监控。

如果业务返回**`RECONSUME_LATER`**:

- 设置 `ackIndex = -1`，表示整批失败。
- 失败计数器增加。

广播模式下消息失败不会重试，直接 **丢弃**。

集群模式下，失败消息会：

- 通过 `sendMessageBack` 发回 Broker → 投递到 **重试队列**（%RETRY%topic）。
- 如果 `sendMessageBack` 失败（比如网络问题），就把消息保留下来，重新提交到 `ConsumeRequest` 里，稍后重试。

调用`consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs())`方法从 `ProcessQueue` 中移除已消费消息，返回最新的消费位点,`updateOffset`：更新 offset 到本地缓存（`OffsetStore`）。最终 offset 会定期 **提交到 Broker**（默认 5s）。



## 4.2 顺序消费逻辑

顺序消费是调用的`ConsumeMessageOrderlyService`的内部类`ConsumeRequest`的run方法，该方法源码如下:

```java
public void run() {
            if (this.processQueue.isDropped()) {
                log.warn("run, the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                return;
            }

            final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
            synchronized (objLock) {
                if (MessageModel.BROADCASTING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                    || (this.processQueue.isLocked() && !this.processQueue.isLockExpired())) {
                    final long beginTime = System.currentTimeMillis();
                    for (boolean continueConsume = true; continueConsume; ) {
                        if (this.processQueue.isDropped()) {
                            log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                            break;
                        }

                        if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                            && !this.processQueue.isLocked()) {
                            log.warn("the message queue not locked, so consume later, {}", this.messageQueue);
                            ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                            break;
                        }

                        if (MessageModel.CLUSTERING.equals(ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.messageModel())
                            && this.processQueue.isLockExpired()) {
                            log.warn("the message queue lock expired, so consume later, {}", this.messageQueue);
                            ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 10);
                            break;
                        }

                        long interval = System.currentTimeMillis() - beginTime;
                        if (interval > MAX_TIME_CONSUME_CONTINUOUSLY) {
                            ConsumeMessageOrderlyService.this.submitConsumeRequestLater(processQueue, messageQueue, 10);
                            break;
                        }

                        final int consumeBatchSize =
                            ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();

                        List<MessageExt> msgs = this.processQueue.takeMessages(consumeBatchSize);
                        defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());
                        if (!msgs.isEmpty()) {
                            final ConsumeOrderlyContext context = new ConsumeOrderlyContext(this.messageQueue);

                            ConsumeOrderlyStatus status = null;

                            ConsumeMessageContext consumeMessageContext = null;
                            if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                                consumeMessageContext = new ConsumeMessageContext();
                                consumeMessageContext
                                    .setConsumerGroup(ConsumeMessageOrderlyService.this.defaultMQPushConsumer.getConsumerGroup());
                                consumeMessageContext.setNamespace(defaultMQPushConsumer.getNamespace());
                                consumeMessageContext.setMq(messageQueue);
                                consumeMessageContext.setMsgList(msgs);
                                consumeMessageContext.setSuccess(false);
                                // init the consume context type
                                consumeMessageContext.setProps(new HashMap<String, String>());
                                ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookBefore(consumeMessageContext);
                            }

                            long beginTimestamp = System.currentTimeMillis();
                            ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
                            boolean hasException = false;
                            try {
                                this.processQueue.getConsumeLock().lock();
                                if (this.processQueue.isDropped()) {
                                    log.warn("consumeMessage, the message queue not be able to consume, because it's dropped. {}",
                                        this.messageQueue);
                                    break;
                                }

                                status = messageListener.consumeMessage(Collections.unmodifiableList(msgs), context);
                            } catch (Throwable e) {
                                log.warn(String.format("consumeMessage exception: %s Group: %s Msgs: %s MQ: %s",
                                    RemotingHelper.exceptionSimpleDesc(e),
                                    ConsumeMessageOrderlyService.this.consumerGroup,
                                    msgs,
                                    messageQueue), e);
                                hasException = true;
                            } finally {
                                this.processQueue.getConsumeLock().unlock();
                            }

                            if (null == status
                                || ConsumeOrderlyStatus.ROLLBACK == status
                                || ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                                log.warn("consumeMessage Orderly return not OK, Group: {} Msgs: {} MQ: {}",
                                    ConsumeMessageOrderlyService.this.consumerGroup,
                                    msgs,
                                    messageQueue);
                            }

                            long consumeRT = System.currentTimeMillis() - beginTimestamp;
                            if (null == status) {
                                if (hasException) {
                                    returnType = ConsumeReturnType.EXCEPTION;
                                } else {
                                    returnType = ConsumeReturnType.RETURNNULL;
                                }
                            } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
                                returnType = ConsumeReturnType.TIME_OUT;
                            } else if (ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT == status) {
                                returnType = ConsumeReturnType.FAILED;
                            } else if (ConsumeOrderlyStatus.SUCCESS == status) {
                                returnType = ConsumeReturnType.SUCCESS;
                            }

                            if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                                consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
                            }

                            if (null == status) {
                                status = ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT;
                            }

                            if (ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                                consumeMessageContext.setStatus(status.toString());
                                consumeMessageContext
                                    .setSuccess(ConsumeOrderlyStatus.SUCCESS == status || ConsumeOrderlyStatus.COMMIT == status);
                                consumeMessageContext.setAccessChannel(defaultMQPushConsumer.getAccessChannel());
                                ConsumeMessageOrderlyService.this.defaultMQPushConsumerImpl.executeHookAfter(consumeMessageContext);
                            }

                            ConsumeMessageOrderlyService.this.getConsumerStatsManager()
                                .incConsumeRT(ConsumeMessageOrderlyService.this.consumerGroup, messageQueue.getTopic(), consumeRT);

                            continueConsume = ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);
                        } else {
                            continueConsume = false;
                        }
                    }
                } else {
                    if (this.processQueue.isDropped()) {
                        log.warn("the message queue not be able to consume, because it's dropped. {}", this.messageQueue);
                        return;
                    }

                    ConsumeMessageOrderlyService.this.tryLockLaterAndReconsume(this.messageQueue, this.processQueue, 100);
                }
            }
        }
```

同样判断`ProcessQueue`的dropped属性如果为true，如果当前 `ProcessQueue` 已经被丢弃（比如 Rebalance 迁移时），就直接退出，不再消费。

获取**MessageQueue 级别锁**，保证同一个 `MessageQueue` 在同一个 Consumer 实例内只会有一个线程在消费,RocketMQ 在顺序消费模式下需要严格保证消息顺序，因此这里加 `synchronized`。

校验队列分布式锁（Clustering 模式）某个 Queue 可能分配给多个消费者实例，所以需要 **Broker 端分布式锁** 来保证 **同一时刻只有一个 Consumer 能消费这个队列**，这句话需要理解一下，正常情况下一个队列只能分配给一个客户端，无论是并发消费模式还是顺序模式，但是Reblance情况可能因为网络抖动导致这样一个现象，在 Consumer A 还没释放 `Queue` 的时候，Consumer B 可能已经拿到了 `Queue` 的分配权。短时间内就会出现 **同一个 Queue 看似被两个 Consumer 拿到** 的情况。

如果没锁住，或者锁过期，就 100ms 后重试 `tryLockLaterAndReconsume`。

接着写一个for死循环进行分批消费，为什么要分批消费那，因为顺序消费是需要加锁的，如果一次性全部给消费者可能造成阻塞，每次从 `ProcessQueue` 拉一批消息（`consumeBatchSize`，默认 1），调用用户实现的 `MessageListenerOrderly.consumeMessage()` 执行业务逻辑。根据消费结果（`status`）决定是否继续消费，还是暂停/回退。

最后还是调用` ConsumeMessageOrderlyService.this.processConsumeResult(msgs, status, context, this);`处理消费结果。

- 根据 `status` 不同，有不同处理：
  - `SUCCESS`：提交 offset，继续消费。
  - `SUSPEND_CURRENT_QUEUE_A_MOMENT`：挂起一会儿，再继续消费。
  - `ROLLBACK` / `COMMIT`：控制事务性场景。
  
这段逻辑处理很复杂，这里我画了一张流程图，概括上面源码每步处理的流程:

![deepseek_mermaid_20251025_548f58.png](..%2Fimg%2Fdeepseek_mermaid_20251025_548f58.png)

## 五、总结

这篇文章系统地分析了 **RocketMQ 消费端的启动与消息处理流程**，从消费者初始化、负载均衡、消息拉取，到并发与顺序两种消费模式的处理机制，都做了深入的源码级解读。

### 核心流程总结：

1. **消费者启动**
   - 初始化 `DefaultMQPushConsumer`，设置消费者组、NameServer、Topic 订阅和消息监听器。
   - 调用 `start()` 方法后，启动 `RebalanceService`、`PullMessageService` 等核心服务。
2. **负载均衡（Rebalance）**
   - `RebalanceService` 每隔 20 秒执行一次队列分配。
   - 使用 `AllocateMessageQueueStrategy`（默认平均分配）为消费者分配消息队列。
   - 更新本地 `ProcessQueueTable`，移除无效队列，创建新的 `PullRequest`。
3. **消息拉取（PullMessageService）**
   - 从 `PullRequestQueue` 中取出拉取任务，调用 `pullMessage()` 向 Broker 拉取消息。
   - 支持流控机制：消息数量、内存大小、队列跨度等阈值控制拉取频率。
4. **消息消费处理**
   - **并发消费**：使用 `ConsumeMessageConcurrentlyService`，提交 `ConsumeRequest` 到线程池并发处理。
   - **顺序消费**：使用 `ConsumeMessageOrderlyService`，对每个 `MessageQueue` 加锁，保证同一队列内消息顺序处理。
5. **消费结果处理**
   - **成功**：更新消费进度（OffsetStore），提交到 Broker。
   - **失败**：根据模式（广播/集群）决定是否重试（发送到重试队列）。

RocketMQ 消费端的设计体现了 **高可用、可扩展、灵活可控** 的架构理念。通过 Rebalance、PullMessage、ConsumeMessage 三大核心服务的协作，实现了消息的可靠投递与高效处理。理解这一流程，有助于在实际使用中更好地进行性能调优与问题排查。
