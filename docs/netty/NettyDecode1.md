# Netty解码器在RocketMQ的应用

## 一、引言
本文将介绍 Netty 解码器在 RocketMQ 中的实际应用，以 Broker 注册到 NameServer 的流程 为例，说明自定义编解码逻辑在客户端与服务端之间是如何协作完成消息解码的。

文中将涉及部分 RocketMQ 的核心源码，但不对其进行深入讲解，相关源码机制将在后续的 RocketMQ 源码分析专题中详细展开。

## 二、Broker端Netty源码

Broker端作为RocketMQ客户端，它向NameServer服务端发送注册请求同时可以处理服务端返回的数据，它的源码如下:
```java
Bootstrap handler = this.bootstrap.group(this.eventLoopGroupWorker).channel(NioSocketChannel.class)
    .option(ChannelOption.TCP_NODELAY, true)
    .option(ChannelOption.SO_KEEPALIVE, false)
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, nettyClientConfig.getConnectTimeoutMillis())
    .handler(new ChannelInitializer<SocketChannel>() {
        @Override
        public void initChannel(SocketChannel ch) throws Exception {
            ChannelPipeline pipeline = ch.pipeline();
            if (nettyClientConfig.isUseTLS()) {
                if (null != sslContext) {
                    pipeline.addFirst(defaultEventExecutorGroup, "sslHandler", sslContext.newHandler(ch.alloc()));
                    log.info("Prepend SSL handler");
                } else {
                    log.warn("Connections are insecure as SSLContext is null!");
                }
            }
            pipeline.addLast(
                defaultEventExecutorGroup,
                new NettyEncoder(),
                new NettyDecoder(),
                new IdleStateHandler(0, 0, nettyClientConfig.getClientChannelMaxIdleTimeSeconds()),
                new NettyConnectManageHandler(),
                new NettyClientHandler());
        }
    });
}
```
它要向服务器写数据必先经过编码阶段才能把数据写出去，那这个关键类是NettyDecoder,它的源码如下：
```java
public class NettyEncoder extends MessageToByteEncoder<RemotingCommand> {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(RemotingHelper.ROCKETMQ_REMOTING);

    @Override
    public void encode(ChannelHandlerContext ctx, RemotingCommand remotingCommand, ByteBuf out)
        throws Exception {
        try {
            remotingCommand.fastEncodeHeader(out);
            byte[] body = remotingCommand.getBody();
            if (body != null) {
                out.writeBytes(body);
            }
        } catch (Exception e) {
            log.error("encode exception, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()), e);
            if (remotingCommand != null) {
                log.error(remotingCommand.toString());
            }
            RemotingUtil.closeChannel(ctx.channel());
        }
    }
}
```
主要是调用了encode方法，把注册请求对象RemotingCommand写入到NameServer端，先是调用了`remotingCommand.fastEncodeHeader`,该方法源码如下:
```java
 public void fastEncodeHeader(ByteBuf out) {
        int bodySize = this.body != null ? this.body.length : 0;
        int beginIndex = out.writerIndex();
        // skip 8 bytes
        out.writeLong(0);
        int headerSize;
        if (SerializeType.ROCKETMQ == serializeTypeCurrentRPC) {
            if (customHeader != null && !(customHeader instanceof FastCodesHeader)) {
                this.makeCustomHeaderToNet();
            }
            headerSize = RocketMQSerializable.rocketMQProtocolEncode(this, out);
        } else {
            this.makeCustomHeaderToNet();
            byte[] header = RemotingSerializable.encode(this);
            headerSize = header.length;
            out.writeBytes(header);
        }
        out.setInt(beginIndex, 4 + headerSize + bodySize);
        out.setInt(beginIndex + 4, markProtocolType(headerSize, serializeTypeCurrentRPC));
}
```
把body长度赋值给变量bodySize,先获取ByteBuf的writerIndex值赋值给变量beginIndex,此时它的值是0,`out.writeLong(0)`写入一个Long类型的0，占用了八个字节
此时writerIndex值变为8，接着写入头部数据writerIndex值增加为byte[]数组长度,`out.setInt(beginIndex, 4 + headerSize + bodySize)`设置从偏移量为0
到3(占用4个字节)的位置一个int数值(代表的是数据总长度),接着调用`out.setInt(beginIndex + 4, markProtocolType(headerSize, serializeTypeCurrentRPC));`,从偏移量4的
位置写入一个int数值(代表的是头部长度),fastEncodeHeader执行完成以后回到encode方法设置请求体的内容，同样writerIndex增加body体字节数组的长度,执行完这些代码以后内存结构图是下面这样的:

| Index | 长度 | 内容                  | 示例值（假设）        |
|-------|------|-----------------------|------------------------|
| 0     | 4B   | totalLength           | 0x0000012C（300）      |
| 4     | 4B   | headerMeta            | 0x01000064（type=1, len=100） |
| 8     | 100B | headerData            | 编码后的 RemotingCommand |
|108    | 192B | bodyData（如果有）    | 业务字节内容            |

它代表了一个完整RocketMQ的帧结构,Netty会根据这些判断如果不符合就继续等待接入更多字节流。

## 三、NameServer端Netty源码

NameServer端作为RocketMQ服务器端，它接收来自Broker的注册请求，它的源码如下：
```java
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
        )});
    try {
        ChannelFuture sync = this.serverBootstrap.bind().sync();
        InetSocketAddress addr = (InetSocketAddress) sync.channel().localAddress();
        this.port = addr.getPort();
    } catch (InterruptedException e1) {
      throw new RuntimeException("this.serverBootstrap.bind().sync() InterruptedException", e1);
    }
```
那么按照上面的执行逻辑来分析一下服务器端是如何进行解码把ByteBuf对象转换为业务对象RemotingCommand的,它的关键处理类是NettyDecoder,从名字看出它是
一个解码器,它的源码如下:
```java
public class NettyDecoder extends LengthFieldBasedFrameDecoder {

    @Override
    public Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        ByteBuf frame = null;
        try {
            frame = (ByteBuf) super.decode(ctx, in);
            if (null == frame) {
                return null;
            }
            return RemotingCommand.decode(frame);
        } catch (Exception e) {
            log.error("decode exception, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()), e);
            RemotingUtil.closeChannel(ctx.channel());
        } finally {
            if (null != frame) {
                frame.release();
            }
        }

        return null;
    }
}
```
可以看出它继承了LengthFieldBasedFrameDecoder类，这个类上篇文章已经分析过了，NettyDecoder的decode方法先调用父类的decode方法去除掉头部字段得到一个
新的帧数据ByteBuf frame,它是包含了头部长度、头部数据、body数据的字节流，然后调用`RemotingCommand.decode`方法，该方法源码如下:
```java
public static RemotingCommand decode(final ByteBuf byteBuffer) throws RemotingCommandException {
    int length = byteBuffer.readableBytes();
    int oriHeaderLen = byteBuffer.readInt();
    int headerLength = getHeaderLength(oriHeaderLen);
    if (headerLength > length - 4) {
        throw new RemotingCommandException("decode error, bad header length: " + headerLength);
    }

    RemotingCommand cmd = headerDecode(byteBuffer, headerLength, getProtocolType(oriHeaderLen));

    int bodyLength = length - 4 - headerLength;
    byte[] bodyData = null;
    if (bodyLength > 0) {
        bodyData = new byte[bodyLength];
        byteBuffer.readBytes(bodyData);
    }
    cmd.body = bodyData;

    return cmd;
}
```
获取总的长度length,获取头部长度oriHeaderLen,buteBuffer.readInt()读取完长度以后readIndex增加4,此时在去读取就是头部的字节数组,调用
headerDecode方法,该方法源码如下:
```java
private static RemotingCommand headerDecode(ByteBuf byteBuffer, int len, SerializeType type) throws RemotingCommandException {
        switch (type) {
            case JSON:
                byte[] headerData = new byte[len];
                byteBuffer.readBytes(headerData);
                RemotingCommand resultJson = RemotingSerializable.decode(headerData, RemotingCommand.class);
                resultJson.setSerializeTypeCurrentRPC(type);
                return resultJson;
            case ROCKETMQ:
                RemotingCommand resultRMQ = RocketMQSerializable.rocketMQProtocolDecode(byteBuffer, len);
                resultRMQ.setSerializeTypeCurrentRPC(type);
                return resultRMQ;
            default:
                break;
        }
        return null;
}
```
len是头部长度，构造headerData的字符长度是len,返回反序列化到RemotingCommand对象中,接着回到decode方法,`int bodyLength = length - 4 - headerLength;`是
获取body的字符长度，如果大于0即存在body读取出来设置到RemotingCommand对象中，这样就完成了解码的逻辑。

## 四、Netty解码器在RocketMQ中的应用总结

本文详细分析了Netty解码器在RocketMQ中的实际应用，重点围绕Broker注册到NameServer的流程展开，展示了自定义编解码逻辑在客户端与服务端之间的协作机制。以下是核心要点总结：

**1.Broker端编码流程**

  使用NettyEncoder将RemotingCommand对象编码为字节流，通过fastEncodeHeader方法构建完整的帧结构（包括总长度、头部元数据、头部数据和可选的Body数据）。

  帧结构的设计清晰，通过ByteBuf的索引操作确保数据的正确性和完整性。

**2.NameServer端解码流程**

  NettyDecoder通过继承LengthFieldBasedFrameDecoder的NettyDecoder类，实现基于长度字段的帧解码逻辑。

  调用RemotingCommand.decode方法将字节流反序列化为业务对象，解析过程分为头部解码和Body数据填充两部分。
  
最后用一张图表示Broker和NameServer直接的编解码关系:

![deepseek_mermaid_20250727_84677a.png](..%2Fimg%2Fdeepseek_mermaid_20250727_84677a.png)