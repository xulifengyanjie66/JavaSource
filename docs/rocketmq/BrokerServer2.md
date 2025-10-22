# RocketMQ源码详解(Broker保存消息源码)

## 一、前言

在分布式消息队列系统中，消息存储是其核心组成部分之一，它直接决定了消息的可靠性、系统的扩展性与性能。RocketMQ 作为一款广泛使用的高性能、高可靠性的消息队列中间件，凭借其灵活的架构和强大的功能，已成为大规模分布式系统中不可或缺的一部分。

在本文中，我将重点解析 RocketMQ Broker 端的消息存储机制。我们将从 **消息的持久化过程**、**存储结构设计**、**性能优化** 以及 **存储与消费的高效协作** 等方面进行详细剖析，探讨 RocketMQ 如何确保消息在高并发、高吞吐量的环境下依然能够可靠地存储并快速地消费,本文基于RocketMQ4.9.8。

## 二、消息存储源码

Broker端处理消息存储的类是SendMessageProcessor,处理方法是asyncProcessRequest,它的方法源码如下:

```java
    public void asyncProcessRequest(ChannelHandlerContext ctx, RemotingCommand request, RemotingResponseCallback responseCallback) throws Exception {
        asyncProcessRequest(ctx, request).thenAcceptAsync(responseCallback::callback, this.brokerController.getPutMessageFutureExecutor());
    }
```

看到这里调用了一个重载方法asyncProcessRequest，其中有两个参数一个是netty的ChannelHandlerContext对象，一个是客户端请求对象RemotingCommand，重载方法源码如下:

```java
    public CompletableFuture<RemotingCommand> asyncProcessRequest(ChannelHandlerContext ctx,
                                                                  RemotingCommand request) throws RemotingCommandException {
        
                //省略不太重要的源码
                final SendMessageContext mqtraceContext;

                SendMessageRequestHeader requestHeader = parseRequestHeader(request);
 
                mqtraceContext = buildMsgContext(ctx, requestHeader);
                this.executeSendMessageHookBefore(ctx, request, mqtraceContext);

                return this.asyncSendMessage(ctx, request, mqtraceContext, requestHeader);
        }
    }
```

可以看出该重载方法返回的是一个CompletableFuture对象，说明这是一个异步的执行方法，在逻辑执行完成以后会执行thenAcceptAsync里面得业务逻辑，这里我主要先分析asyncProcessRequest源码逻辑。

1. 首先获取客户端请求头部封装为SendMessageRequestHeader对象。
2. 根据SendMessageRequestHeader对象构造发送消息上下文对象SendMessageContext。

调用asyncSendMessage方法处理存储消息逻辑,该方法的源码如下:

```java
private CompletableFuture<RemotingCommand> asyncSendMessage(ChannelHandlerContext ctx, RemotingCommand request,
                                                                SendMessageContext mqtraceContext,
                                                                SendMessageRequestHeader requestHeader) {
        final RemotingCommand response = preSend(ctx, request, requestHeader);
      
        final byte[] body = request.getBody();

        int queueIdInt = requestHeader.getQueueId();
        TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());

        if (queueIdInt < 0) {
            queueIdInt = randomQueueId(topicConfig.getWriteQueueNums());
        }

        MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
        msgInner.setTopic(requestHeader.getTopic());
        msgInner.setQueueId(queueIdInt);
        
        //省略设置其他属性......................
       
        putMessageResult = this.brokerController.getMessageStore().asyncPutMessage(msgInner);

        return handlePutMessageResultFuture(putMessageResult, response, request, msgInner, responseHeader, mqtraceContext, ctx, queueIdInt);
    }
```

1. 构造响应对象RemotingCommand。
2. 获取客户端请求数据的字节数组body。
3. 获取客户端请求的要存储消息的队列id，如果是没有传递随机选取一个消息队列id。
4. 创建MessageExtBrokerInner对象，封装存储消息属性，这里主要包含主题、消息队列id、消息请求内容、集群名称等，接着调用核心处理方法DefaultMessageStore对象的asyncPutMessage方法。

DefaultMessageStore的asyncPutMessage方法源码如下:

```
 public CompletableFuture<PutMessageResult> asyncPutMessage(MessageExtBrokerInner msg) {
         
        //省略检查代码。。。。。。。。。。
        
        long beginTime = this.getSystemClock().now();
        CompletableFuture<PutMessageResult> putResultFuture = this.commitLog.asyncPutMessage(msg);

        putResultFuture.thenAccept(result -> {
            long elapsedTime = this.getSystemClock().now() - beginTime;
            if (elapsedTime > 500) {
                log.warn("putMessage not in lock elapsed time(ms)={}, bodyLength={}", elapsedTime, msg.getBody().length);
            }
            this.storeStatsService.setPutMessageEntireTimeMax(elapsedTime);

            if (null == result || !result.isOk()) {
                this.storeStatsService.getPutMessageFailedTimes().add(1);
            }
        });

        return putResultFuture;
    }
```

可以看出该方法还是返回CompletableFuture对象说明也是一个异步操作，它的主要流程是调用了CommitLog的asyncPutMessage方法,它的方法源码如下：

```java
 public CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {
        

          AppendMessageResult result = null;
        
          PutMessageThreadLocal putMessageThreadLocal = this.putMessageThreadLocal.get();
          updateMaxMessageSize(putMessageThreadLocal);
          if (!multiDispatch.isMultiDispatchMsg(msg)) {
          PutMessageResult encodeResult = putMessageThreadLocal.getEncoder().encode(msg);
          if (encodeResult != null) {
          return CompletableFuture.completedFuture(encodeResult);
          }
          msg.setEncodedBuff(putMessageThreadLocal.getEncoder().getEncoderBuffer());
          }
      
        PutMessageContext putMessageContext = new PutMessageContext(generateKey(putMessageThreadLocal.getKeyBuilder(), msg));

        long elapsedTimeInLock = 0;
        MappedFile unlockMappedFile = null;

        putMessageLock.lock(); 
        try {
            MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
            long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
            this.beginTimeInLock = beginLockTimestamp;

            msg.setStoreTimestamp(beginLockTimestamp);

            if (null == mappedFile || mappedFile.isFull()) {
                mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
            }
            if (null == mappedFile) {
                log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null));
            }

            result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
            switch (result.getStatus()) {
                case PUT_OK:
                    break;
                case END_OF_FILE:
                    unlockMappedFile = mappedFile;
                    mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                    if (null == mappedFile) {
                        log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                        return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result));
                    }
                    result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
                    break;
                case MESSAGE_SIZE_EXCEEDED:
                case PROPERTIES_SIZE_EXCEEDED:
                    return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result));
                case UNKNOWN_ERROR:
                    return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result));
                default:
                    return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result));
            }

            elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
        } finally {
            beginTimeInLock = 0;
            putMessageLock.unlock();
        }

        if (elapsedTimeInLock > 500) {
            log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", elapsedTimeInLock, msg.getBody().length, result);
        }

        if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
            this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
        }

        PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

        storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).add(1);
        storeStatsService.getSinglePutMessageTopicSizeTotal(topic).add(result.getWroteBytes());

        CompletableFuture<PutMessageStatus> flushResultFuture = submitFlushRequest(result, msg);
        CompletableFuture<PutMessageStatus> replicaResultFuture = submitReplicaRequest(result, msg);
        return flushResultFuture.thenCombine(replicaResultFuture, (flushStatus, replicaStatus) -> {
            if (flushStatus != PutMessageStatus.PUT_OK) {
                putMessageResult.setPutMessageStatus(flushStatus);
            }
            if (replicaStatus != PutMessageStatus.PUT_OK) {
                putMessageResult.setPutMessageStatus(replicaStatus);
            }
            return putMessageResult;
        });
    }
```

先从ThreadLocal<PutMessageThreadLocal>中获取MessageExtEncoder对象，然后调用其encode方法，在分析之前先来看看ThreadLocal<PutMessageThreadLocal>是怎么声明的，在CommitLog构造时候初始化了一个ThreadLocal<PutMessageThreadLocal>对象，代码如下:

```java
putMessageThreadLocal = new ThreadLocal<PutMessageThreadLocal>() {
    @Override
    protected PutMessageThreadLocal initialValue() {
        return new PutMessageThreadLocal(defaultMessageStore.getMessageStoreConfig().getMaxMessageSize());
    }
};
```

可以看出每个线程在执行时候会执行到initialValue方法创建一个PutMessageThreadLocal对象，那么不同线程之间的对象就是隔离的，互不影响，这在高并发下很重要不用加锁了提高Broker的处理性能，每个线程的PutMessageThreadLocal对象的成员数据也是隔离的不会相互影响写数据。知道了这个问题我们来看一下

PutMessageThreadLocal的构造方法，代码如下：

```java
PutMessageThreadLocal(int maxMessageBodySize) {
    encoder = new MessageExtEncoder(maxMessageBodySize);
    keyBuilder = new StringBuilder();
}
```

传入参数maxMessageBodySize代表消息体的最大大小默认是4M，又实例化一个成员属性MessageExtEncoder对象把最大消息长度传入,源码是如下这样的:

```java
MessageExtEncoder(final int maxMessageBodySize) {
    ByteBufAllocator alloc = UnpooledByteBufAllocator.DEFAULT;
    //Reserve 64kb for encoding buffer outside body
    int maxMessageSize = Integer.MAX_VALUE - maxMessageBodySize >= 64 * 1024 ?
        maxMessageBodySize + 64 * 1024 : Integer.MAX_VALUE;
    byteBuf = alloc.directBuffer(maxMessageSize);
    this.maxMessageBodySize = maxMessageBodySize;
    this.maxMessageSize = maxMessageSize;
}
```

创建Netty 的 `ByteBufAllocator` 分配直接内存缓冲区（DirectBuffer），用于高效写入消息到 CommitLog，它能避免 JVM 堆复制，提高 IO 性能。

消息编码除了 body 外，还需要存储一些消息元数据（头部信息、属性、系统标志等）。

RocketMQ 预留 **64KB** 空间给编码时的头部信息和 buffer 额外开销。

如果 `Integer.MAX_VALUE - maxMessageBodySize` 足够大，就用 `maxMessageBodySize + 64KB`。

否则直接用 `Integer.MAX_VALUE`（理论上防止整型溢出）。

知道上面信息后，既可以分析MessageExtEncoder对象的encode方法了，该方法源码如下:

```java
protected PutMessageResult encode(MessageExtBrokerInner msgInner) {
            this.byteBuf.clear();
           
            final byte[] propertiesData =
                    msgInner.getPropertiesString() == null ? null : msgInner.getPropertiesString().getBytes(MessageDecoder.CHARSET_UTF8);

            final int propertiesLength = propertiesData == null ? 0 : propertiesData.length;

            if (propertiesLength > Short.MAX_VALUE) {
                log.warn("putMessage message properties length too long. length={}", propertiesData.length);
                return new PutMessageResult(PutMessageStatus.PROPERTIES_SIZE_EXCEEDED, null);
            }

            final byte[] topicData = msgInner.getTopic().getBytes(MessageDecoder.CHARSET_UTF8);
            final int topicLength = topicData.length;

            final int bodyLength = msgInner.getBody() == null ? 0 : msgInner.getBody().length;

            final int msgLen = calMsgLength(msgInner.getSysFlag(), bodyLength, topicLength, propertiesLength);

            // Exceeds the maximum message body
            if (bodyLength > this.maxMessageBodySize) {
                CommitLog.log.warn("message body size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLength
                    + ", maxMessageSize: " + this.maxMessageBodySize);
                return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);
            }

            // Exceeds the maximum message
            if (msgLen > this.maxMessageSize) {
                CommitLog.log.warn("message size exceeded, msg total size: " + msgLen + ", msg body size: " + bodyLength
                        + ", maxMessageSize: " + this.maxMessageSize);
                return new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, null);
            }

            // 1 TOTALSIZE
            this.byteBuf.writeInt(msgLen);
            // 2 MAGICCODE
            this.byteBuf.writeInt(CommitLog.MESSAGE_MAGIC_CODE);
            // 3 BODYCRC
            this.byteBuf.writeInt(msgInner.getBodyCRC());
            // 4 QUEUEID
            this.byteBuf.writeInt(msgInner.getQueueId());
            // 5 FLAG
            this.byteBuf.writeInt(msgInner.getFlag());
            // 6 QUEUEOFFSET, need update later
            this.byteBuf.writeLong(0);
            // 7 PHYSICALOFFSET, need update later
            this.byteBuf.writeLong(0);
            // 8 SYSFLAG
            this.byteBuf.writeInt(msgInner.getSysFlag());
            // 9 BORNTIMESTAMP
            this.byteBuf.writeLong(msgInner.getBornTimestamp());

            // 10 BORNHOST
            ByteBuffer bornHostBytes = msgInner.getBornHostBytes();
            this.byteBuf.writeBytes(bornHostBytes.array());

            // 11 STORETIMESTAMP
            this.byteBuf.writeLong(msgInner.getStoreTimestamp());

            // 12 STOREHOSTADDRESS
            ByteBuffer storeHostBytes = msgInner.getStoreHostBytes();
            this.byteBuf.writeBytes(storeHostBytes.array());

            // 13 RECONSUMETIMES
            this.byteBuf.writeInt(msgInner.getReconsumeTimes());
            // 14 Prepared Transaction Offset
            this.byteBuf.writeLong(msgInner.getPreparedTransactionOffset());
            // 15 BODY
            this.byteBuf.writeInt(bodyLength);
            if (bodyLength > 0)
                this.byteBuf.writeBytes(msgInner.getBody());
            // 16 TOPIC
            this.byteBuf.writeByte((byte) topicLength);
            this.byteBuf.writeBytes(topicData);
            // 17 PROPERTIES
            this.byteBuf.writeShort((short) propertiesLength);
            if (propertiesLength > 0)
                this.byteBuf.writeBytes(propertiesData);

            return null;
        }
```

整体看来是把消息对象编码成二进制格式，写入byteBuf，包含 **消息头、消息体、Topic、Properties**。同时需要做长度校验防止消息过大。

首先先将消息属性字符串序列化为UTF-8字节数组，propertiesLength计算消息属性长度，属性长度超过 2 字节能表示的范围就报错。

接着把topic序列化为字节数组，得到topic长度，接着获取消息体长度，调用calMsgLength计算消息总长度。

校验消息体大小不能超过最大允许的长度，总长度同样需要校验不能超过最大允许的长度。

接着就开始往直接内存中写入数据的过程，为了直观我这里画一个表格说明一下各个字段的含义：

| 序号 |            字段             |  类型  |   长度（字节）   |            备注             |
| :--: | :-------------------------: | :----: | :--------------: | :-------------------------: |
|  1   |          TOTALSIZE          |  int   |        4         |         消息总长度          |
|  2   |          MAGICCODE          |  int   |        4         |       校验码，固定值        |
|  3   |           BODYCRC           |  int   |        4         |        消息体 CRC32         |
|  4   |           QUEUEID           |  int   |        4         |           队列号            |
|  5   |            FLAG             |  int   |        4         |       用户自定义标记        |
|  6   |         QUEUEOFFSET         |  long  |        8         | 消费队列偏移，写 0 后续更新 |
|  7   |       PHYSICALOFFSET        |  long  |        8         | 消息物理偏移，写 0 后续更新 |
|  8   |           SYSFLAG           |  int   |        4         |          系统标记           |
|  9   |        BORNTIMESTAMP        |  long  |        8         |       消息生产时间戳        |
|  10  |          BORNHOST           | byte[] |       6~18       |      生产者 IP + 端口       |
|  11  |       STORETIMESTAMP        |  long  |        8         |       消息存储时间戳        |
|  12  |      STOREHOSTADDRESS       | byte[] |       6~18       |      Broker IP + 端口       |
|      |       RECONSUMETIMES        |  int   |        4         |        消费重试次数         |
|      | PREPARED_TRANSACTION_OFFSET |  long  |        8         |    事务消息的预提交偏移     |
|      |            BODY             |  int   |        4         |         消息体长度          |
|      |        BODY CONTENT         | byte[] |    bodyLength    |           消息体            |
|      |            TOPIC            |  byte  |        1         |          主题长度           |
|      |        TOPIC CONTENT        | byte[] |   topicLength    |          主题内容           |
|      |         PROPERTIES          | short  |        2         |        消息属性长度         |
|      |     PROPERTIES CONTENT      | byte[] | propertiesLength |        消息属性内容         |

```java
PutMessageContext putMessageContext = new PutMessageContext(generateKey(putMessageThreadLocal.getKeyBuilder(), msg));
```

该段代码调用了generateKey方法拼接逻辑队列和id直接的关系，形式大概是这样的topic-topicId,后面用于查询这个逻辑偏移量。

存储完成以后设置MessageExtBrokerInner对象的ByteBuffer encodedBuff属性为上面写入的ByteBuffer,即MessageExtEncoder成员变量的ByteBuf。

然后从MappedFileQueue队列获取一个MappedFile对象，在前面文章说过它代表一个CommitLog的物理文件是用mmap映射的，效率极高,接着调用MappedFile的appendMessage方法，该方法源码如下:

```
 public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb,
            PutMessageContext putMessageContext) {
        assert messageExt != null;
        assert cb != null;

        int currentPos = this.wrotePosition.get();

        if (currentPos < this.fileSize) {
            ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
            byteBuffer.position(currentPos);
            AppendMessageResult result;
            if (messageExt instanceof MessageExtBrokerInner) {
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos,
                        (MessageExtBrokerInner) messageExt, putMessageContext);
            } else if (messageExt instanceof MessageExtBatch) {
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos,
                        (MessageExtBatch) messageExt, putMessageContext);
            } else {
                return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
            }
            this.wrotePosition.addAndGet(result.getWroteBytes());
            this.storeTimestamp = result.getStoreTimestamp();
            return result;
        }
        log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
        return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
    }
```

获取当前文件的写索引值，如果判断是否小于文件总大小，如果大于返回一个错误对象AppendMessageResult，如果是小于总大小，调用对象MappedByteBuffer的slice方法切出来一个mmap映射对象MappedByteBuffer并且设置为从当前的currentPos开始写入数据，调用doAppend方法开始真正写入数据，该方法源码如下:

```java
public AppendMessageResult doAppend(final long fileFromOffset, final ByteBuffer byteBuffer, final int maxBlank,
            final MessageExtBrokerInner msgInner, PutMessageContext putMessageContext) {

            long wroteOffset = fileFromOffset + byteBuffer.position();

            Supplier<String> msgIdSupplier = () -> {
                int sysflag = msgInner.getSysFlag();
                int msgIdLen = (sysflag & MessageSysFlag.STOREHOSTADDRESS_V6_FLAG) == 0 ? 4 + 4 + 8 : 16 + 4 + 8;
                ByteBuffer msgIdBuffer = ByteBuffer.allocate(msgIdLen);
                MessageExt.socketAddress2ByteBuffer(msgInner.getStoreHost(), msgIdBuffer);
                msgIdBuffer.clear();//because socketAddress2ByteBuffer flip the buffer
                msgIdBuffer.putLong(msgIdLen - 8, wroteOffset);
                return UtilAll.bytes2string(msgIdBuffer.array());
            };

            // Record ConsumeQueue information
            String key = putMessageContext.getTopicQueueTableKey();
            Long queueOffset = CommitLog.this.topicQueueTable.get(key);
            if (null == queueOffset) {
                queueOffset = 0L;
                CommitLog.this.topicQueueTable.put(key, queueOffset);
            }

            boolean multiDispatchWrapResult = CommitLog.this.multiDispatch.wrapMultiDispatch(msgInner);
            if (!multiDispatchWrapResult) {
                return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
            }

            // Transaction messages that require special handling
            final int tranType = MessageSysFlag.getTransactionValue(msgInner.getSysFlag());
            switch (tranType) {
                // Prepared and Rollback message is not consumed, will not enter the
                // consumer queue
                case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
                case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                    queueOffset = 0L;
                    break;
                case MessageSysFlag.TRANSACTION_NOT_TYPE:
                case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                default:
                    break;
            }

            ByteBuffer preEncodeBuffer = msgInner.getEncodedBuff();
            final int msgLen = preEncodeBuffer.getInt(0);

            // Determines whether there is sufficient free space
            if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
                this.msgStoreItemMemory.clear();
                // 1 TOTALSIZE
                this.msgStoreItemMemory.putInt(maxBlank);
                // 2 MAGICCODE
                this.msgStoreItemMemory.putInt(CommitLog.BLANK_MAGIC_CODE);
                // 3 The remaining space may be any value
                // Here the length of the specially set maxBlank
                final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
                byteBuffer.put(this.msgStoreItemMemory.array(), 0, 8);
                return new AppendMessageResult(AppendMessageStatus.END_OF_FILE, wroteOffset,
                        maxBlank, /* only wrote 8 bytes, but declare wrote maxBlank for compute write position */
                        msgIdSupplier, msgInner.getStoreTimestamp(),
                        queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);
            }

            int pos = 4 + 4 + 4 + 4 + 4;
            // 6 QUEUEOFFSET
            preEncodeBuffer.putLong(pos, queueOffset);
            pos += 8;
            // 7 PHYSICALOFFSET
            preEncodeBuffer.putLong(pos, fileFromOffset + byteBuffer.position());
            int ipLen = (msgInner.getSysFlag() & MessageSysFlag.BORNHOST_V6_FLAG) == 0 ? 4 + 4 : 16 + 4;
            // 8 SYSFLAG, 9 BORNTIMESTAMP, 10 BORNHOST, 11 STORETIMESTAMP
            pos += 8 + 4 + 8 + ipLen;
            // refresh store time stamp in lock
            preEncodeBuffer.putLong(pos, msgInner.getStoreTimestamp());


            final long beginTimeMills = CommitLog.this.defaultMessageStore.now();
            // Write messages to the queue buffer
            byteBuffer.put(preEncodeBuffer);
            msgInner.setEncodedBuff(null);
            AppendMessageResult result = new AppendMessageResult(AppendMessageStatus.PUT_OK, wroteOffset, msgLen, msgIdSupplier,
                msgInner.getStoreTimestamp(), queueOffset, CommitLog.this.defaultMessageStore.now() - beginTimeMills);

            switch (tranType) {
                case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
                case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                    break;
                case MessageSysFlag.TRANSACTION_NOT_TYPE:
                case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                    // The next update ConsumeQueue information
                    CommitLog.this.topicQueueTable.put(key, ++queueOffset);
                    CommitLog.this.multiDispatch.updateMultiQueueOffset(msgInner);
                    break;
                default:
                    break;
            }
            return result;
   }
```

此处从PutMessageContext获取逻辑便宜的key,从内存中拿到当前队列的逻辑偏移,获取之间写入数据的直接内存ByteBuffer，获取最前面四个字节的数据即是数据的总大小msgLen。

```java
if ((msgLen + END_FILE_MIN_BLANK_LENGTH) > maxBlank) {
    // 写入文件末尾特殊空白标记，标记该文件写满
    // 返回END_OF_FILE状态，提示换下一个文件
}
```

上面这段代码判断如果消息长度加上文件末尾的空白长度超过当前文件剩余空间，则写入特殊的“空白消息”表示文件末尾。

返回状态 `END_OF_FILE`。

```java
int pos = 4 + 4 + 4 + 4 + 4;
// 6 QUEUEOFFSET
preEncodeBuffer.putLong(pos, queueOffset);
```

这段代码是跳过TOTALSIZE, MAGICCODE, BODYCRC, QUEUEID, FLAG 五个int字段，然后写入写入逻辑队列偏移，接着写入物理偏移，然后写入时间戳信息。

```
byteBuffer.put(preEncodeBuffer);msgInner.setEncodedBuff(null);
```

调用此方法把ByteBuffer数据写入到CommitLog对应的MappedFile中，注意此时还只是在内存中还没有同步到Broker服务器的磁盘中,然后释放消息中的编码缓存。

```java
AppendMessageResult result = new AppendMessageResult(
    AppendMessageStatus.PUT_OK,
    wroteOffset,
    msgLen,
    msgIdSupplier,
    msgInner.getStoreTimestamp(),
    queueOffset,
    CommitLog.this.defaultMessageStore.now() - beginTimeMills);
```

构造返回结果AppendMessageResult，包含写入状态、物理偏移、消息长度、消息ID、存储时间等。

最后更新 `topicQueueTable` 中队列偏移，通知多线程分发更新队列逻辑偏移即写入到ConsumeQueue对应的文件中。

到此为止asyncPutMessage方法的写入消息逻辑以及分析完成后面的执行逻辑主要是:

```java
 CompletableFuture<PutMessageStatus> flushResultFuture = submitFlushRequest(result, msg);
        CompletableFuture<PutMessageStatus> replicaResultFuture = submitReplicaRequest(result, msg);
        return flushResultFuture.thenCombine(replicaResultFuture, (flushStatus, replicaStatus) -> {
            if (flushStatus != PutMessageStatus.PUT_OK) {
                putMessageResult.setPutMessageStatus(flushStatus);
            }
            if (replicaStatus != PutMessageStatus.PUT_OK) {
                putMessageResult.setPutMessageStatus(replicaStatus);
            }
            return putMessageResult;
        });
```

这两段逻辑很重要，上面说过只是把数据先写入到内存中，确切的说是**内存页缓存（page cache）** 中，还没有真正安全地“落盘”或“同步到副本”，接着要**刷盘**和

**主从复制**（Replication,如果是集群部署，还要把消息从 Master 同步到 Slave 节点)。

**刷盘工作机制：**

1.    **异步刷盘 (ASYNC_FLUSH)**

   - 直接把任务放入队列（`FlushRealTimeService` 或 `GroupCommitService`）；
   - 后台线程异步执行 `fileChannel.force()`；
   - 阻塞当前写入线程，性能高但可靠性稍低。

2.   **同步刷盘 (SYNC_FLUSH)**

   - 当前线程会等待刷盘线程返回成功信号；

   - 也就是写完消息后必须确认磁盘落盘完成；
   - 可靠性高但性能略低。

都执行完成以后返回`CompletableFuture<PutMessageStatus>`， 表示“刷盘完成结果”的异步回调（成功或失败），`PutMessageStatus` 可能是：

- `PUT_OK`（写入成功）
- `FLUSH_DISK_TIMEOUT`（刷盘超时）
- `UNKNOWN_ERROR` 等。

**主从复制工作机制:**

- 在 `SYNC_MASTER` 模式下，消息写入后必须等待 **至少一个 Slave 同步成功**；
- 在 `ASYNC_MASTER` 模式下，Master 立即返回，不等待 Slave；
- 复制逻辑由 `HAService`（High Availability Service）管理。

同样是一个 `CompletableFuture<PutMessageStatus>`，表示主从复制结果的异步通知：

- `PUT_OK`：同步成功；

- `SLAVE_NOT_AVAILABLE`：找不到从节点；

- `FLUSH_SLAVE_TIMEOUT`：等待从节点确认超时。

最后调用CompletableFuture的thenCombine方法它会等待两个CompletableFuture都执行完成，然后执行一个回调函数 `(flushStatus, replicaStatus)`。

1. 如果 **刷盘失败**（比如超时、磁盘IO异常）
    → 把最终结果设为 `flushStatus`（例如 `FLUSH_DISK_TIMEOUT`）；
2. 如果 **主从同步失败**（比如从节点不可用）
    → 覆盖前面的结果，设为 `replicaStatus`；
3. 如果都成功
    → `putMessageResult` 的状态保持为默认的 `PUT_OK`。

最终返回一个统一的 `CompletableFuture<PutMessageResult>` 对象。

此时返回返回到最开头的调用者对象SendMessageProcessor的方法asyncSendMessage的这段代码

```java
return handlePutMessageResultFuture(putMessageResult, response, request, msgInner, responseHeader, mqtraceContext, ctx, queueIdInt)
```

该方法就是设置返回给客户端的一些响应头对象我这里就不分析了，感兴趣的朋友可以自己看一下这块的逻辑。

下面我分析一些同步和异步刷盘的逻辑，它的方法源码如下:

```java
 public CompletableFuture<PutMessageStatus> submitFlushRequest(AppendMessageResult result, MessageExt messageExt) {
        // Synchronization flush
        if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
            if (messageExt.isWaitStoreMsgOK()) {
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes(),
                        this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                flushDiskWatcher.add(request);
                service.putRequest(request);
                return request.future();
            } else {
                service.wakeup();
                return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
            }
        }
        // Asynchronous flush
        else {
            if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                flushCommitLogService.wakeup();
            } else  {
                commitLogService.wakeup();
            }
            return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
        }
   }
```

可以看出这个方法也是返回的CompletableFuture，如果是异步操作直接就会返回结果`CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);`调用者不需要等待,但是同步的没有立即返回结果需要等待刷盘以后才会调用类似的completedFutrue方法通知调用者，那我们先看同步刷屏的逻辑，即if语句块里面的逻辑。

**同步刷盘逻辑:**

 首先创建一个GroupCommitRequest对象，wroteOffset + wroteBytes代表消息写入后的位置，也就是刷盘目标点，然后将请求放入到 `GroupCommitService` 的队列中，`GroupCommitService` 是一个**后台线程**，会周期性地从队列中取出请求，执行刷盘（`mappedFileQueue.flush`）。现在重点看后台线程是怎么刷盘的，代码如下:

```java
private void doCommit() {
    if (!this.requestsRead.isEmpty()) {
        for (GroupCommitRequest req : this.requestsRead) {
            // There may be a message in the next file, so a maximum of
            // two times the flush
            boolean flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();
            for (int i = 0; i < 2 && !flushOK; i++) {
                CommitLog.this.mappedFileQueue.flush(0);
                flushOK = CommitLog.this.mappedFileQueue.getFlushedWhere() >= req.getNextOffset();
            }

            req.wakeupCustomer(flushOK ? PutMessageStatus.PUT_OK : PutMessageStatus.FLUSH_DISK_TIMEOUT);
        }

        long storeTimestamp = CommitLog.this.mappedFileQueue.getStoreTimestamp();
        if (storeTimestamp > 0) {
            CommitLog.this.defaultMessageStore.getStoreCheckpoint().setPhysicMsgTimestamp(storeTimestamp);
        }

        this.requestsRead = new LinkedList<>();
    } else {
        // Because of individual messages is set to not sync flush, it
        // will come to this process
        CommitLog.this.mappedFileQueue.flush(0);
    }
}
```

核心逻辑:

**检查是否已经刷到目标 offset**：

- `mappedFileQueue.getFlushedWhere()` 返回当前已经刷盘到的位置。
- 如果 `>= req.getNextOffset()`，说明消息已经落盘，无需再刷。

**刷盘两次**：

- 为了防止消息刚好跨文件（RocketMQ 的 commit log 是分文件存储的），最多刷两次。
- 调用 `mappedFileQueue.flush(0)` 实际触发 `MappedFile` 的写入磁盘操作。

刷盘成功后调用`req.wakeupCustomer(flushOK ? PutMessageStatus.PUT_OK : PutMessageStatus.FLUSH_DISK_TIMEOUT);`唤醒等待线程。

`wakeupCustomer` 会触发 **CompletableFuture 完成**，通知发送线程：

- `PUT_OK` → 消息刷盘成功
- `FLUSH_DISK_TIMEOUT` → 消息刷盘超时

这就是同步刷盘模式下，发送线程被阻塞等待刷盘结果的机制核心。



**异步刷盘逻辑:**

异步刷盘后台处理线程是`CommitRealTimeService`,该方法源码如下:

```java
public void run() {
            CommitLog.log.info(this.getServiceName() + " service started");
            while (!this.isStopped()) {
                int interval = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitIntervalCommitLog();

                int commitDataLeastPages = CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogLeastPages();

                int commitDataThoroughInterval =
                    CommitLog.this.defaultMessageStore.getMessageStoreConfig().getCommitCommitLogThoroughInterval();

                long begin = System.currentTimeMillis();
                if (begin >= (this.lastCommitTimestamp + commitDataThoroughInterval)) {
                    this.lastCommitTimestamp = begin;
                    commitDataLeastPages = 0;
                }

                try {
                    boolean result = CommitLog.this.mappedFileQueue.commit(commitDataLeastPages);
                    long end = System.currentTimeMillis();
                    if (!result) {
                        this.lastCommitTimestamp = end; // result = false means some data committed.
                        //now wake up flush thread.
                        flushCommitLogService.wakeup();
                    }

                    if (end - begin > 500) {
                        log.info("Commit data to file costs {} ms", end - begin);
                    }
                    this.waitForRunning(interval);
                } catch (Throwable e) {
                    CommitLog.log.error(this.getServiceName() + " service has exception. ", e);
                }
            }

            boolean result = false;
            for (int i = 0; i < RETRY_TIMES_OVER && !result; i++) {
                result = CommitLog.this.mappedFileQueue.commit(0);
                CommitLog.log.info(this.getServiceName() + " service shutdown, retry " + (i + 1) + " times " + (result ? "OK" : "Not OK"));
            }
            CommitLog.log.info(this.getServiceName() + " service end");
        }
```

- 每次刷盘循环等待的时间（默认 200ms 左右）

- **commitDataLeastPages**：最少刷的页数（默认 4 页，每页 4KB 或 8KB，根据系统配置）。

- **commitDataThoroughInterval**：彻底刷盘间隔（比如 10 次 commit 或 1 秒），即使数据不足页数也强制 commit。

  **lastCommitTimestamp**：上一次彻底刷盘时间，用于计算是否强制刷盘。

### 三、总结

RocketMQ 的消息存储机制是其高可靠、高性能的核心保障。通过 **内存映射、异步处理、线程隔离、刷盘(同步刷盘、异步刷盘)与复制策略** 等多种技术手段，实现了在高并发场景下消息的快速、可靠存储。理解其源码实现，不仅有助于优化使用姿势，也为自研分布式存储系统提供了宝贵参考。