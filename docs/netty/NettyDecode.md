# Netty 解码器深入分析：调用时机与继承关系
## 一、引言

- 简要介绍 Netty 作为高性能网络框架的应用场景。

- 解码器在 Netty 中的作用：从 字节流 解码为 消息对象，并解决粘包/拆包问题。

- 本文将重点分析解码器的调用时机、继承关系和LengthFieldBasedFrameDecoder、LineBasedFrameDecoder的实现源码。

## 二、Netty解码器概述
### 2.1 什么是Netty解码器?

- 定义：解码器是 Netty 中用来从 ByteBuf 中提取消息内容的组件。

- 作用：将字节流转化为 Java 对象，以供业务层处理。

- 常见解码器：LengthFieldBasedFrameDecoder、LineBasedFrameDecoder、DelimiterBasedFrameDecoder等。

### 2.2 解码器的作用

- 解析协议：比如基于长度、分隔符、固定长度等协议格式。

- 解决粘包/拆包问题：保证从字节流中解析出完整的一帧数据。

## 三、Netty解码器的继承结构
### 3.1 解码器的父类：ByteToMessageDecoder

- ByteToMessageDecoder是所有Netty解码器的基类，继承自ChannelInboundHandlerAdapter。

- 它主要实现了decode()方法，负责从ByteBuf中读取数据，并填充解码后的消息。

- decode() 的默认实现是将数据存入out。

### 3.2 类的继承结构图：

![img_10.png](..%2Fimg%2Fimg_10.png)

### 3.3 解码器的调用顺序

- ChannelPipeline的工作流程：

  - 数据通过 ChannelPipeline 传递，每个 ChannelHandler 会处理接收到的数据。

  - 解码器作为 ChannelInboundHandler 的一部分，被放置在 ChannelPipeline 中来处理入站数据。

  - 顺序：数据从 ChannelPipeline 中的每个 handler 顺序传递，解码器的执行会在数据流到达时被调用。

## 四、解码器的调用时机
### 4.1 ChannelPipeline的调用流程

- 数据进入ChannelPipeline：每个 ChannelHandler（包括解码器）依次处理入站数据。

- 解码器的作用时机：

  - 在数据传输到达某个 handler 时，解码器会被触发。

  - 一旦解码器读取到足够的数据，它会触发 channelRead()，将解码后的数据传递到下一个handler。
### 4.2 decode()方法的执行时机

- decode() 方法在解码器中被自动调用，Netty 会根据当前的 ByteBuf 数据调用 decode() 处理。

- decode() 会读取传入的 ByteBuf 并解码，只处理可读取的部分，并将解码后的对象添加到 out中。

### 4.3什么时候调用 decode()？

- 解码器会在channelRead() 被触发时调用 decode() 方法。

- 当接收到的ByteBuf数据足够解析时，decode()会被自动执行。

- 解码器也会在cumulation（内部缓存区）数据积累完毕后调用 decode()。

## 五、LineBasedFrameDecoder源码分析
### 5.1 类定义与构造方法
- 类定义：LineBasedFrameDecoder是一个基于行分隔符的帧解码器，通常用于处理类似HTTP、IRC 等协议。
- 构造方法：
  - maxLength：设置消息的最大长度，用于防止无限制读取数据。
  - delimiter：行分隔符(通常为 \n 或 \r\n)。

### 5.2 成员属性
1.**maxLength**:代表一个消息的最大允许的长度。

2.**failFast**:布尔类型的属性,如果设置为 true，当解码器发现一个消息超过了最大允许的长度(例如，消息太长，无法在缓冲区中存储)，它会立刻抛出异常，并放弃该消息的解码处理。如果设置为 false，解码器会等到处理完整个消息或者收到足够的数据后再进行判断，不会立即抛出异常。

3.**stripDelimiter**: 是一个布尔类型的属性，通常在基于分隔符的解码器中使用，用来决定是否从解码出的消息中移除分隔符(例如，\n或 \r\n)。它的作用是:如果stripDelimiter = true，解码器会去掉消息中的分隔符，只返回有效的数据部分。例如，LineBasedFrameDecoder会去掉每行的换行符(\n 或 \r\n),
如果stripDelimiter = false，解码器会保留分隔符，解码出的消息将包括分隔符。

4.**discarding**: 是一个布尔值属性，通常在处理缓冲区时使用，表示是否正在丢弃数据。如果一个缓冲区正在丢弃数据(比如数据已经超出了处理范围，或者暂时不需要处理)，discarding 的值会被设置为true。

5.**discardedBytes**: discardedBytes是一个整数属性，表示已经丢弃的字节数。它用来记录自上次丢弃数据以来，已经丢弃了多少字节。

6.**offset**: offset是一个整数属性，表示数据在缓冲区中的偏移量。它通常指示从数据的起始位置到某个指定位置的距离。

### 5.3 源码分析
当我们在一个ChannelPipeline中添加LineBasedFrameDecoder时候会调用其channelRead方法，这个方法是其父类ByteToMessageDecoder的方法,该方法源码如下：
```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof ByteBuf) {
            CodecOutputList out = CodecOutputList.newInstance();
            try {
                first = cumulation == null;
                cumulation = cumulator.cumulate(ctx.alloc(),
                        first ? Unpooled.EMPTY_BUFFER : cumulation, (ByteBuf) msg);
                callDecode(ctx, cumulation, out);
            } catch (DecoderException e) {
                throw e;
            } catch (Exception e) {
                throw new DecoderException(e);
            } finally {
                try {
                    if (cumulation != null && !cumulation.isReadable()) {
                        numReads = 0;
                        cumulation.release();
                        cumulation = null;
                    } else if (++numReads >= discardAfterReads) {
                        numReads = 0;
                        discardSomeReadBytes();
                    }
                    int size = out.size();
                    firedChannelRead |= out.insertSinceRecycled();
                    fireChannelRead(ctx, out, size);
                } finally {
                    out.recycle();
                }
            }
        } else {
            ctx.fireChannelRead(msg);
        }
}
```
从对象池中借出一个CodecOutputList实例，如果没有可用的旧对象就新建一个,调用cumulator.cumulate方法,cumulator是一个匿名类，它的源码是这样的:
```java
public static final Cumulator MERGE_CUMULATOR = new Cumulator() {
    @Override
    public ByteBuf cumulate(ByteBufAllocator alloc, ByteBuf cumulation, ByteBuf in) {
        if (!cumulation.isReadable() && in.isContiguous()) {
            cumulation.release();
            return in;
        }
        try {
            final int required = in.readableBytes();
            if (required > cumulation.maxWritableBytes() ||
                    (required > cumulation.maxFastWritableBytes() && cumulation.refCnt() > 1) ||
                    cumulation.isReadOnly()) {
                return expandCumulation(alloc, cumulation, in);
            }
            cumulation.writeBytes(in, in.readerIndex(), required);
            in.readerIndex(in.writerIndex());
            return cumulation;
        } finally {
            in.release();
        }
    }
}
```
MERGE_CUMULATOR是字节缓冲累加器逻辑(Cumulator),Netty使用它在处理半包、拆包(如 TCP 粘包)时，把多次接收到的ByteBuf数据累积到一个连续的缓冲区中。为后续的解码器提供一个连续的数据区域。

`if (!cumulation.isReadable() && in.isContiguous()) `

- cumulation.isReadable()：判断旧缓冲区是否还有数据，如果没有说明可以丢掉。

- in.isContiguous()：判断in是否是连续内存（如非 CompositeBuffer）。

如果旧缓冲区是空的并且新的缓冲区是连续内存，使用新数据的in作为cumulation，cumulation.isReadable()为false说明上次的数据已经全部消费掉了，没有剩余数据,也就是当前解码起点是新的包头，不需要拼接历史数据了。

```java
final int required = in.readableBytes();
if (required > cumulation.maxWritableBytes()
    || (required > cumulation.maxFastWritableBytes() && cumulation.refCnt() > 1)
    || cumulation.isReadOnly()) {
    return expandCumulation(alloc, cumulation, in);
}
```
这段代码说明了三种情况说明旧缓冲区无法继续写入新数据，需要替换或扩容:

- maxWritableBytes：写空间不足；

- refCnt() > 1：被多个地方引用，不安全，不能原地扩容；

- isReadOnly()：是只读缓冲区(如 Unpooled.wrappedBuffer(byte[])；

`cumulation.writeBytes(in, in.readerIndex(), required);` 将 in的数据写入cumulation，此时cumulation 成为数据总汇。

设置 `in.readerIndex = writerIndex：`表示in的数据已经全部读完了，下次不能再读,最后释放in,调用的是in.release()。

接着调用callDecode方法，传入ChannelHandlerContext ctx、上面计算得出的ByteBuf cumulation、Object msg,该方法源码如下:
```java
protected void callDecode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        try {
            while (in.isReadable()) {
                final int outSize = out.size();
                if (outSize > 0) {
                    fireChannelRead(ctx, out, outSize);
                    out.clear();
                    if (ctx.isRemoved()) {
                        break;
                    }
                }

                int oldInputLength = in.readableBytes();
                decodeRemovalReentryProtection(ctx, in, out);
                if (ctx.isRemoved()) {
                    break;
                }
                if (out.isEmpty()) {
                    if (oldInputLength == in.readableBytes()) {
                        break;
                    } else {
                        continue;
                    }
                }
                if (oldInputLength == in.readableBytes()) {
                    throw new DecoderException(
                            StringUtil.simpleClassName(getClass()) +
                                    ".decode() did not read anything but decoded a message.");
                }
                if (isSingleDecode()) {
                    break;
                }
            }
        } catch (DecoderException e) {
            throw e;
        } catch (Exception cause) {
            throw new DecoderException(cause);
        }
    }
```
调用in.readableBytes()读取出字节数，然后调用decodeRemovalReentryProtection方法，该方法源码如下:
```java
final void decodeRemovalReentryProtection(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
            throws Exception {
        decodeState = STATE_CALLING_CHILD_DECODE;
        try {
            decode(ctx, in, out);
        } finally {
            boolean removePending = decodeState == STATE_HANDLER_REMOVED_PENDING;
            decodeState = STATE_INIT;
            if (removePending) {
                fireChannelRead(ctx, out, out.size());
                out.clear();
                handlerRemoved(ctx);
            }
        }
}
```
这个方法调用了decode方法，该方法源码如下:
```java
@Override
protected final void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
    Object decoded = decode(ctx, in);
    if (decoded != null) {
        out.add(decoded);
    }
}
```
又调用重载方法decode方法，该方法源码如下:
```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
    final int eol = findEndOfLine(buffer);
    if (!discarding) {
        if (eol >= 0) {
            final ByteBuf frame;
            final int length = eol - buffer.readerIndex();
            final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1;

            if (length > maxLength) {
                buffer.readerIndex(eol + delimLength);
                fail(ctx, length);
                return null;
            }

            if (stripDelimiter) {
                frame = buffer.readRetainedSlice(length);
                buffer.skipBytes(delimLength);
            } else {
                frame = buffer.readRetainedSlice(length + delimLength);
            }

            return frame;
        } else {
            final int length = buffer.readableBytes();
            if (length > maxLength) {
                discardedBytes = length;
                buffer.readerIndex(buffer.writerIndex());
                discarding = true;
                offset = 0;
                if (failFast) {
                    fail(ctx, "over " + discardedBytes);
                }
            }
            return null;
        }
    } else {
        if (eol >= 0) {
            final int length = discardedBytes + eol - buffer.readerIndex();
            final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1;
            buffer.readerIndex(eol + delimLength);
            discardedBytes = 0;
            discarding = false;
            if (!failFast) {
                fail(ctx, length);
            }
        } else {
            discardedBytes += buffer.readableBytes();
            buffer.readerIndex(buffer.writerIndex());
            offset = 0;
        }
        return null;
    }
}
```
这里面又调用了findEndOfLine方法，该方法主要是找是否有以\n结尾或者\r结尾的字符，该方法的源码如下:
```java
private int findEndOfLine(final ByteBuf buffer) {
    int totalLength = buffer.readableBytes();
    int i = buffer.forEachByte(buffer.readerIndex() + offset, totalLength - offset, ByteProcessor.FIND_LF);
    if (i >= 0) {
        offset = 0;
        if (i > 0 && buffer.getByte(i - 1) == '\r') {
            i--;
        }
    } else {
        offset = totalLength;
    }
    return i;
}
```
如果找到了i就会是大于或者等于0，把offset=0,如果没有找到i是小于0的,offset等于读取到的字符长度，最终返回i。

- 如果不是丢弃状态并且 i < 0的情况会执行下面的逻辑：
```java
 final int length = buffer.readableBytes();
    if (length > maxLength) {
        discardedBytes = length;
        buffer.readerIndex(buffer.writerIndex());
        discarding = true;
        offset = 0;
        if (failFast) {
            fail(ctx, "over " + discardedBytes);
        }
    }
    return null;
```
   - 读取字节如果大于maxLength,记录被丢弃的字节数(discardedBytes = length),将读指针跳到写指针，跳过这段数据(readerIndex = writerIndex),进入丢弃状态discarding = true，表示接下来数据都不要了,如果 failFast == true，则立刻触发异常(调用 fail()),最后返回 null，表示这一帧数据不继续往下传。

   - 如果小于maxLength直接返回null。

- 如果不是丢弃状态并且 i > 0的情况会执行下面的逻辑：
```java
 if (eol >= 0) {
      final ByteBuf frame;
      final int length = eol - buffer.readerIndex();
      final int delimLength = buffer.getByte(eol) == '\r'? 2 : 1;

      if (length > maxLength) {
          buffer.readerIndex(eol + delimLength);
          fail(ctx, length);
          return null;
      }

      if (stripDelimiter) {
          frame = buffer.readRetainedSlice(length);
          buffer.skipBytes(delimLength);
      } else {
          frame = buffer.readRetainedSlice(length + delimLength);
      }

      return frame;
} 
```
计算length的值，它是当前读取位置到换行符之间的字节长度(即这一行的正文内容长度),判断换行符的长度是 1 字节(`\n`)还是 2 字节(`\r\n`),如果是 `\r\n`，总长度是2,即delimLength=2,否则即delimLength=1,判断这行数据是否超过最大长度，如果是
跳过整行(包括分隔符),调用 fail(ctx, length) 抛出异常或处理,返回 null 表示这帧数据无效。

判断stripDelimiter如果是true，表示不要分隔符数据，只读取正文，如果是false则表示连同换行符也作为数据的一部分读出,返回成功解析的一帧数据并添加到List<Object> out,此时返回到callDecode方法，继续执行while方法判断是否还有可以读取的字节，如果有先判断
out是否不为空,如果不为空就调用fireChannelRead方法往下游传递字符数据,这种情况常见于一次性手动多个完整的帧数据的情况，每次读完一个完整帧数据就往下游传递,供下游 handler处理。

## 六、LengthFieldBasedFrameDecoder源码分析
### 6.1 类定义与构造方法
- 类定义：LengthFieldBasedFrameDecoder是Netty提供的一个非常常用的分包/粘包处理解码器。它通过读取报文中某个**长度字段**来决定每一帧(frame)的边界，解决了TCP粘包/拆包问题,它会读取设置的某个长度字段，根据这个字段的值，截取出一帧完整的消息，然后交给后续的处理器。

- 构造方法：

```java
public LengthFieldBasedFrameDecoder(
    int maxFrameLength,
    int lengthFieldOffset,
    int lengthFieldLength,
    int lengthAdjustment,
    int initialBytesToStrip
)
```
### 6.2 成员属性

| 变量名                                         |             说明              |
  |:--------------------------------------------|:---------------------------:|
| maxFrameLength                              |        最大允许帧长度，防止攻击         | 
| lengthFieldOffset                           |          长度字段的起始偏移          | 
| lengthFieldLength	                          |          长度字段的字节长度          |
| lengthAdjustment	                           |     从长度字段值到帧末尾需要调整的偏移量      |
| initialBytesToStrip	                        |    解码后要丢弃的字节数量（一般用于跳过头部）    |
| failFast	                                   | 是否在超长帧读到长度字段时就立即抛异常（否则等读完整） |
| discardingTooLongFrame	                     |       是否正在丢弃超长帧(状态标记)       |

**1. maxFrameLength：最大帧长度（防止攻击或异常）**

- 类型：int

- 作用：控制每一帧最大允许的长度。超出就认为数据异常（如恶意客户端粘包攻击），会抛 TooLongFrameException。

- 示例：你设为 1024，表示最多每帧 1024 字节，超出丢弃。

**2.lengthFieldOffset：长度字段的起始偏移**

- 类型：int

- 作用：告诉 Netty 从第几个字节开始是长度字段。

- 示例：如果前 2 字节是“魔数”或“标志位”，长度字段从第 3 字节开始，那就设置为 2。

**3. lengthFieldLength：长度字段的长度**

- 类型：int

- 作用：告诉 Netty，长度字段占几个字节（1/2/4/8）。

- 常见设置：

  - 1：byte

  - 2：short

  - 4：int(最常见)

  - 8：long
  
**4. lengthAdjustment：长度补偿（纠偏）**

- 类型：int

- 作用：长度字段是否包含其他字段（如头部）？

  - 如果长度字段的值只表示数据体长度（不含头），那这个参数设置为 0；

  - 如果长度字段的值包含了头部，比如“总长度 = 头部 + 数据”，那需要减去头部长度（比如 -6）。

示例：
协议结构：

| 魔数(2) | 长度(4) | 数据(N bytes) |

如果长度字段（4）表示的数据长度 不包含前面的6个字节，那 adjustment 就是 0。

如果长度字段包含整个包（比如长度字段 = N+6），那你就要设置 adjustment = -6。

**5. initialBytesToStrip：解码后跳过的字节数**

- 类型：int

- 作用：解码成功后，是否跳过前面的某些字节？比如魔数、版本号之类你不想处理，可以 strip 掉。

- 示例：

 - 如果只关心业务体，头部不需要，就设置为头部长度，比如跳过前 6 字节。

 - 如果想完整保留头部，就设为 0。

最常见的协议是这样的：

| 长度字段(4) | 实际数据(N bytes) |

表示长度字段表示的是 “数据体的长度”，不包含自己。

```java
new LengthFieldBasedFrameDecoder(
    1024,     // 最大帧长度
    0,        // 长度字段从第0字节开始
    4,        // 长度字段占4字节
    0,        // 长度字段就代表数据体，不需要补偿
    4         // 跳过长度字段，只保留数据体
);
```
### 6.3 decode源码分析

它的源码是这样的:

```java
protected Object decode(ChannelHandlerContext ctx, ByteBuf in) throws Exception {
        if (discardingTooLongFrame) {
            discardingTooLongFrame(in);
        }
        if (in.readableBytes() < lengthFieldEndOffset) {
            return null;
        }
        int actualLengthFieldOffset = in.readerIndex() + lengthFieldOffset;
        long frameLength = getUnadjustedFrameLength(in, actualLengthFieldOffset, lengthFieldLength, byteOrder);
        if (frameLength < 0) {
            failOnNegativeLengthField(in, frameLength, lengthFieldEndOffset);
        }
        frameLength += lengthAdjustment + lengthFieldEndOffset;
        if (frameLength < lengthFieldEndOffset) {
            failOnFrameLengthLessThanLengthFieldEndOffset(in, frameLength, lengthFieldEndOffset);
        }
        if (frameLength > maxFrameLength) {
            exceededFrameLength(in, frameLength);
            return null;
        }
        int frameLengthInt = (int) frameLength;
        if (in.readableBytes() < frameLengthInt) {
            return null;
        }
        if (initialBytesToStrip > frameLengthInt) {
            failOnFrameLengthLessThanInitialBytesToStrip(in, frameLength, initialBytesToStrip);
        }
        in.skipBytes(initialBytesToStrip);
        int readerIndex = in.readerIndex();
        int actualFrameLength = frameLengthInt - initialBytesToStrip;
        ByteBuf frame = extractFrame(ctx, in, readerIndex, actualFrameLength);
        in.readerIndex(readerIndex + actualFrameLength);
        return frame;
}
```
读取ByteBuf中数据，判断是否小于长度字段，如果小于说明一条完整消息还没来，先返回 null，等待下次调用,从指定偏移读取“未调整的帧长度”，也就是字节流中原始写入的那个长度字段的值,
长度不能是负数，非法协议，抛异常或关闭连接,计算完整帧的总长度frameLength,如果调整后的总长度都比长度字段末尾还小，明显出错，协议格式错误，超过最大允许的帧长，丢弃或报错处理。

```java
if (in.readableBytes() < frameLengthInt) {
    return null;
}
```
这段代码代表读取的数据如果小于当前数据还不够整帧，就返回null等下一次数据到达再处理。

```java
int readerIndex = in.readerIndex();
int actualFrameLength = frameLengthInt - initialBytesToStrip;
ByteBuf frame = extractFrame(ctx, in, readerIndex, actualFrameLength);
in.readerIndex(readerIndex + actualFrameLength);
return frame;
```
- ByteBuf中提取一段完整的帧(消息体)并返回。

- Netty会把返回的这个frame继续往下传给下游handler。

之后的逻辑就又回到ByteToMessageDecoder中了和上面的逻辑一样。

## 七、总结

- 解码器的调用时机：解码器在ChannelPipeline的数据流中起到重要作用，解码的调用时机是由 decode() 方法自动触发的。

- 继承关系：Netty解码器通过继承ByteToMessageDecoder类实现，解码器通过多态处理不同类型的数据流。