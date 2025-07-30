# Netty的Http解码器源码分析



## 一、HTTP协议简介



HTTP（HyperText Transfer Protocol，超文本传输协议）是一种基于 **请求-响应模型** 的无状态应用层协议，广泛用于客户端（如浏览器）和服务器之间的数据通信。其主要特点包括：

- **基于 TCP 连接**（默认端口 80 / HTTPS 默认端口 443）；
- **无状态**：每次请求都是独立的，服务器不会保留会话状态；
- **文本格式**：数据以文本形式组织，便于调试和阅读；
- **支持多种方法**：常见如 GET、POST、PUT、DELETE 等；



##  二、HTTP 请求报文结构

一个标准的 HTTP 请求报文由三部分组成：

1. **请求行**（Request Line）
    格式：`方法 URI 协议版本`
    示例：`GET /index.html HTTP/1.1`

2. **请求头**（Request Headers）
    每一行一个键值对，用于传递客户端信息、内容格式等。
    示例：

   ```
   Host: www.example.com
   User-Agent: Mozilla/5.0
   Content-Length: 123
   ```

3. **请求体**（Request Body）
    一般用于 POST/PUT 请求，承载请求参数或文件内容等。

------


## 三、HTTP 响应报文结构

响应报文也由三部分组成：

1. **状态行**
    格式：`协议版本 状态码 状态说明`
    示例：`HTTP/1.1 200 OK`

2. **响应头部**
   提供内容类型、长度、服务器信息等
   示例：

   ```
   
   Content-Type: application/json
   Content-Length: 456
   ```

3. **响应体**
    包含服务器返回的数据，如 HTML 页面、JSON 字符串等。

    

## 四、Netty 中如何处理 HTTP

Netty 是一个高性能的网络编程框架，其对 HTTP 协议的支持主要由以下几个解码器/编码器实现：



|         类名         |                           作用                           |
| :------------------: | :------------------------------------------------------: |
|  HttpRequestDecoder  |     将字节流解码为 HTTP 请求对象（如 `HttpRequest`）     |
| HttpResponseDecoder  |                         解码响应                         |
|  HttpObjectDecoder   |             抽象基础类，负责协议通用部分解析             |
|   HttpServerCodec    | `HttpRequestDecoder` 和 `HttpResponseEncoder` 的组合封装 |
| HttpObjectAggregator |    将多个 HTTP 片段聚合成完整消息（常用于 POST/PUT）     |

## 五、HttpRequestDecoder源码分析

我这篇文章主要分析一下HttpRequestDecoder源码，主要是分析如何把TCP转过来的字节流转换为Netty自己实现的HttpRequest对象，其它源码暂且不做过多分析，有兴趣的同志可以自己阅读一下。

这里我写了一个处理Http请求的例子，它的类是HttpServer,代码我放在下边了，分析源码主要以这个例子进行分析。

```java
public class HttpServer {

    private final int port;

    public HttpServer(int port) {
        this.port = port;
    }

    public void start() throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new HttpServerInitializer())
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            Channel ch = b.bind(port).sync().channel();

            System.out.println("HTTP Server started on port: " + port);
            ch.closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new HttpServer(8080).start();
    }
}
```
```java
public class HttpServerInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();

        // 解码器：将 HTTP 请求字节流解析为 HttpRequest
        pipeline.addLast(new HttpRequestDecoder());

        // 编码器：将 HttpResponse 对象转换为字节流
        pipeline.addLast(new HttpResponseEncoder());

        // 其他业务逻辑处理
        pipeline.addLast(new HttpServerHandler());
    }
}

```
HttpRequestDecoder继承HttpObjectDecoder,在实例化时候会调用父类的HttpObjectDecoder的无参构造函数，源码如下：

```java
protected HttpObjectDecoder() {
this(DEFAULT_MAX_INITIAL_LINE_LENGTH, DEFAULT_MAX_HEADER_SIZE, DEFAULT_MAX_CHUNK_SIZE,
DEFAULT_CHUNKED_SUPPORTED);
}
```

这里又调用重载构造函数,DEFAULT_MAX_INITIAL_LINE_LENGTH值是4096,DEFAULT_MAX_HEADER_SIZE值是8192,DEFAULT_MAX_CHUNK_SIZE值是8192,DEFAULT_CHUNKED_SUPPORTED值是true,
最终调用到重载函数，该函数源码如下:

```java
protected HttpObjectDecoder(
int maxInitialLineLength, int maxHeaderSize, int maxChunkSize,
boolean chunkedSupported, boolean validateHeaders, int initialBufferSize,
boolean allowDuplicateContentLengths) {
checkPositive(maxInitialLineLength, "maxInitialLineLength");
checkPositive(maxHeaderSize, "maxHeaderSize");
checkPositive(maxChunkSize, "maxChunkSize");

    AppendableCharSequence seq = new AppendableCharSequence(initialBufferSize);
    lineParser = new LineParser(seq, maxInitialLineLength);
    headerParser = new HeaderParser(seq, maxHeaderSize);
    this.maxChunkSize = maxChunkSize;
    this.chunkedSupported = chunkedSupported;
    this.validateHeaders = validateHeaders;
    this.allowDuplicateContentLengths = allowDuplicateContentLengths;
}
```
创建一个AppendableCharSequence对象,成员参数是128,表示char数组长度,初始化一个LineParser对象，它的成员属性分别是AppendableCharSequence、maxInitialLineLength,初始化一个HeaderParser对象，它的成员属性分别是AppendableCharSequence 、maxHeaderSize，然后分别初始化成员属性maxChunkSize=8192、chunkedSupported=true、validateHeaders=true、allowDuplicateContentLengths=false。

参数详细解释：

**1.int maxInitialLineLength**

- 作用：限制请求行的最大长度（比如：GET /index.html HTTP/1.1）。

- 场景：防止恶意客户端发送超长请求行攻击（DoS）。

- 默认 4096，超过则抛 TooLongFrameException。

**2.int maxHeaderSize**

- 作用：限制所有请求头部字段总大小（包括所有 header 的 key 和 value 加起来）。

- 场景：防止请求头爆破，保护内存。

- 默认 8192，超过抛异常。

**3.int maxChunkSize**

- 作用：当使用 Transfer-Encoding: chunked 时，每个 chunk 的最大大小。

- 场景：控制分块传输时每次读取的数据量，防止内存溢出。

**4.boolean chunkedSupported**

- 作用：是否支持 Transfer-Encoding: chunked 编码的请求或响应。

- 说明：HTTP/1.1 中支持 chunked 编码，可以不使用 Content-Length。

- 设为 false：不解析 chunk 数据，适合某些特殊或简化场景。

**5.boolean validateHeaders**

- 作用：是否校验 HTTP 请求头是否合法（如重复头名、非法字符等）。

- 说明：默认 true 会严格验证，如果性能要求高或确定格式规范可关闭。

**6.int initialBufferSize**

- 作用：构造 AppendableCharSequence 的初始容量，优化内存分配。

- 说明：用于读取请求行和头时拼接字符串的缓存空间。

- 默认值建议：一般设置成 128~512，避免频繁扩容。

**7.boolean allowDuplicateContentLengths**

- 作用：是否允许出现多个 Content-Length 头字段。

- 说明：默认 HTTP/1.1 不应该重复，设为 true 则放宽限制。

- 用途：某些非标准客户端或代理可能出现此问题。

 ```java
AppendableCharSequence seq = new AppendableCharSequence(initialBufferSize);
lineParser = new LineParser(seq, maxInitialLineLength);
headerParser = new HeaderParser(seq, maxHeaderSize);
 ```

这三句是设置了解析请求行和请求头的工具类（状态机）：

- seq: 字符串拼接缓存区。
- lineParser: 用于解析请求行（第一行）。
- headerParser: 用于解析 header 部分。

构造函数和成员变量介绍完成以后，下面我们主要看它的decode源码，它是解码的关键代码,它的代码如下:

```java
protected void decode(ChannelHandlerContext ctx, ByteBuf buffer, List<Object> out) throws Exception {
        if (resetRequested) {
            resetNow();
        }

        switch (currentState) {
        case SKIP_CONTROL_CHARS:
            // Fall-through
        case READ_INITIAL: try {
            AppendableCharSequence line = lineParser.parse(buffer);
            if (line == null) {
                return;
            }
            String[] initialLine = splitInitialLine(line);
            if (initialLine.length < 3) {

                currentState = State.SKIP_CONTROL_CHARS;
                return;
            }

            message = createMessage(initialLine);
            currentState = State.READ_HEADER;
            // fall-through
        } catch (Exception e) {
            out.add(invalidMessage(buffer, e));
            return;
        }
        case READ_HEADER: try {
            State nextState = readHeaders(buffer);
            if (nextState == null) {
                return;
            }
            currentState = nextState;
            switch (nextState) {
            case SKIP_CONTROL_CHARS:

                out.add(message);
                out.add(LastHttpContent.EMPTY_LAST_CONTENT);
                resetNow();
                return;
            case READ_CHUNK_SIZE:
                if (!chunkedSupported) {
                    throw new IllegalArgumentException("Chunked messages not supported");
                }

                out.add(message);
                return;
            default:

                long contentLength = contentLength();
                if (contentLength == 0 || contentLength == -1 && isDecodingRequest()) {
                    out.add(message);
                    out.add(LastHttpContent.EMPTY_LAST_CONTENT);
                    resetNow();
                    return;
                }

                assert nextState == State.READ_FIXED_LENGTH_CONTENT ||
                        nextState == State.READ_VARIABLE_LENGTH_CONTENT;

                out.add(message);

                if (nextState == State.READ_FIXED_LENGTH_CONTENT) {

                    chunkSize = contentLength;
                }

                return;
            }
        } catch (Exception e) {
            out.add(invalidMessage(buffer, e));
            return;
        }
        case READ_VARIABLE_LENGTH_CONTENT: {

            int toRead = Math.min(buffer.readableBytes(), maxChunkSize);
            if (toRead > 0) {
                ByteBuf content = buffer.readRetainedSlice(toRead);
                out.add(new DefaultHttpContent(content));
            }
            return;
        }
        case READ_FIXED_LENGTH_CONTENT: {
            int readLimit = buffer.readableBytes();

            if (readLimit == 0) {
                return;
            }

            int toRead = Math.min(readLimit, maxChunkSize);
            if (toRead > chunkSize) {
                toRead = (int) chunkSize;
            }
            ByteBuf content = buffer.readRetainedSlice(toRead);
            chunkSize -= toRead;

            if (chunkSize == 0) {
                // Read all content.
                out.add(new DefaultLastHttpContent(content, validateHeaders));
                resetNow();
            } else {
                out.add(new DefaultHttpContent(content));
            }
            return;
        }
        ..............................
    }
```



这里我主要分析处理请求行、请求头、请求体的源码，其它不做过多的分析。

先调用的是`lineParser.parse(buffer)`方法,该方法的源码如下:

```java
  public AppendableCharSequence parse(ByteBuf buffer) {
            final int oldSize = size;
            seq.reset();
            int i = buffer.forEachByte(this);
            if (i == -1) {
                size = oldSize;
                return null;
            }
            buffer.readerIndex(i + 1);
            return seq;
  }
```



这里又调用了`buffer.forEachByte(this)`方法,它的源码如下:



```java
public boolean process(byte value) throws Exception {
            if (currentState == State.SKIP_CONTROL_CHARS) {
                char c = (char) (value & 0xFF);
                if (Character.isISOControl(c) || Character.isWhitespace(c)) {
                    increaseCount();
                    return true;
                }
                currentState = State.READ_INITIAL;
            }
            return super.process(value);
}
```




调用buffer.forEachByte(this)一个一个字符进行读取，char c = (char) (value & 0xFF)这段代码把value和 0xFF进行与运算是为了让它变成无符号位，因为byte数据类型范围是-128-127，如果出现负数转换为字符char可能会出现乱码。

```java
if (Character.isISOControl(c) || Character.isWhitespace(c)) {
    increaseCount();
    return true;
}
```


判断果当前读取的字符是控制字符或空白字符（比如空格、Tab），就跳过它。这是解析 HTTP 请求前跳过非法前缀的标准做法。把读取到的字符添加到AppendableCharSequence对象中，执行完成以后执行代码buffer.readerIndex(i + 1)，把readerIndex值加上i+1,为了下次读取数据时候从i+1处开始读取。

调用splitInitialLine方法，把请求行分为三部分，分别是请求类型POST/GET,请求路径、Http协议版本放到String数组中。

调用createMessage方法，传入上面的String数组，该方法源码如下:
```java
protected HttpMessage createMessage(String[] initialLine) throws Exception {
    return new DefaultHttpRequest(
            HttpVersion.valueOf(initialLine[2]),
            HttpMethod.valueOf(initialLine[0]), initialLine[1], validateHeaders);
}
```
可以看出最外层实例化一个DefaultHttpRequest对象，它的成员属性有HttpVersion、HttpMethod,创建HttpVersion对象时候会根据客户端传递的HTTP协议版本不同创建HttpVersion(1.0)或者HttpVersion(1.1)对象，它两不同点主要是HTTP 1.1 Keep-Alive默认是true，表示连接不断开可以复用，HTTP 1.1支持动态生成内容分块传输。
创建HttpMethod对象，接着调用DefaultHttpRequest的构造函数，它的构造函数源码如下:

```java
public DefaultHttpRequest(HttpVersion httpVersion, HttpMethod method, String uri, boolean validateHeaders) {
    super(httpVersion, validateHeaders, false);
    this.method = checkNotNull(method, "method");
    this.uri = checkNotNull(uri, "uri");
}
```
调用了父类的构造函数，它的源码如下:
```java
protected DefaultHttpMessage(final HttpVersion version, boolean validateHeaders, boolean singleFieldHeaders) {
    this(version,
            singleFieldHeaders ? new CombinedHttpHeaders(validateHeaders)
                               : new DefaultHttpHeaders(validateHeaders));
}
```
根据singleFieldHeaders值来判断创建是CombinedHttpHeaders还是DefaultHttpHeaders，此处创建的是DefaultHttpHeaders对象，这样DefaultHttpRequest就被赋值上了成员属性HttpVersion、DefaultHttpHeaders、HttpMethod。

此时回到decode方法继续往下执行到`currentState = State.READ_HEADER`,到达`case READ_HEADER`分支调用readHeaders方法开始读取Http请求头部信息，该方法源码如下:
```java
private State readHeaders(ByteBuf buffer) {
        final HttpMessage message = this.message;
        final HttpHeaders headers = message.headers();
        AppendableCharSequence line = headerParser.parse(buffer);
        if (line == null) {
            return null;
        }
        if (line.length() > 0) {
            do {
                char firstChar = line.charAtUnsafe(0);
                if (name != null && (firstChar == ' ' || firstChar == '\t')) {
                    String trimmedLine = line.toString().trim();
                    String valueStr = String.valueOf(value);
                    value = valueStr + ' ' + trimmedLine;
                } else {
                    if (name != null) {
                        headers.add(name, value);
                    }
                    splitHeader(line);
                }
                line = headerParser.parse(buffer);
                if (line == null) {
                    return null;
                }
            } while (line.length() > 0);
        }
        
        if (name != null) {
            headers.add(name, value);
        }
        name = null;
        value = null;
        HttpMessageDecoderResult decoderResult = new HttpMessageDecoderResult(lineParser.size, headerParser.size);
        message.setDecoderResult(decoderResult);

        List<String> contentLengthFields = headers.getAll(HttpHeaderNames.CONTENT_LENGTH);
        if (!contentLengthFields.isEmpty()) {
            HttpVersion version = message.protocolVersion();
            boolean isHttp10OrEarlier = version.majorVersion() < 1 || (version.majorVersion() == 1
                    && version.minorVersion() == 0);
            contentLength = HttpUtil.normalizeAndGetContentLength(contentLengthFields,
                    isHttp10OrEarlier, allowDuplicateContentLengths);
            if (contentLength != -1) {
                headers.set(HttpHeaderNames.CONTENT_LENGTH, contentLength);
            }
        }
        if (isContentAlwaysEmpty(message)) {
            HttpUtil.setTransferEncodingChunked(message, false);
            return State.SKIP_CONTROL_CHARS;
        } else if (HttpUtil.isTransferEncodingChunked(message)) {
            if (!contentLengthFields.isEmpty() && message.protocolVersion() == HttpVersion.HTTP_1_1) {
                handleTransferEncodingChunkedWithContentLength(message);
            }
            return State.READ_CHUNK_SIZE;
        } else if (contentLength() >= 0) {
            return State.READ_FIXED_LENGTH_CONTENT;
        } else {
            return State.READ_VARIABLE_LENGTH_CONTENT;
        }
}
```
调用headerParser.parse方法去解析出来一个头部信息赋值给AppendableCharSequence line,之后有一个do while循环不断处理头部信息把头拆分成name和value两部分，拆分过程中是以":"分隔的，然后拆分完成调用headers.add(name, value)方法添加name、value最终保存到Map.Entry的数组中。

此时继续执行readHeaders方法中的`HttpMessageDecoderResult decoderResult = new HttpMessageDecoderResult(lineParser.size, headerParser.size);`代码片段，创建一个HttpMessageDecoderResult对象代表解析请求头和请求行成功，然后把请求头长度和请求行长度赋值给它的成员变量,并把HttpMessageDecoderResult设置到对象的HttpMessage成员属性DecoderResult中,得到content-length的长度赋值给List<String> contentLengthFields,如果有 Content-Length,就设定chunkSize，接下来的 decode() 中会按照chunkSize长度读 body部分,如果大于-1就返回状态机State.READ_FIXED_LENGTH_CONTENT状态,表示下次在decode中处理请求体的内容，为啥要这样做那，因为TCP是流式处理可能请求体内容还没有到避免进行阻塞所以选择了下次处理。

下次执行会到这块的代码逻辑:

```java
case READ_FIXED_LENGTH_CONTENT: {
    int readLimit = buffer.readableBytes();
    
    if (readLimit == 0) {
        return;
    }
    int toRead = Math.min(readLimit, maxChunkSize);
    if (toRead > chunkSize) {
        toRead = (int) chunkSize;
    }
    ByteBuf content = buffer.readRetainedSlice(toRead);
    chunkSize -= toRead;

    if (chunkSize == 0) {
        out.add(new DefaultLastHttpContent(content, validateHeaders));
        resetNow();
    } else {
        out.add(new DefaultHttpContent(content));
    }
    return;
}
```
得到可以读取字节长度赋值给readLimit,如果等于0直接返回，readLimit和maxChunkSize取最小值赋值给toRead变量，如果它大于chunkSize就把chunkSize值赋值给toRead,因为chunkSize就是Content-Length的长度，要按照这个长度读取字节否则可能读取到无关请求体的数据，得到ByteBuf content,chunkSize=chunkSize-toRead,如果chunkSize等于0说明完成把它添加List<Object> out中,然后调用resetNow重置状态机状态为State.SKIP_CONTROL_CHARS并且清除其他信息比如第一次保存的数据为了处理第二次来的HTTP请求。如果chunkSize不等于0说明没有读取完也添加到List<Object> out中等待下一次读取。

如果是GET请求是没有请求体的，那么Content-Length值就是-1,到这里就分析完成了HttpRequest和HttpContent对象的封装，下游的Handler就可以拿到这些对象进行相应的业务处理。



## 六、总结

本文围绕 `HttpRequestDecoder` 的源码展开，从字节流解析、请求行与请求头提取，请求体提取到组装为 Java 层级的 `HttpRequest` 对象、HttpContent对象，完整梳理了其在 Netty 中处理 HTTP 请求的全流程。这不仅展示了 Netty 在高性能网络通信中的优秀架构设计，也有助于开发者深入理解 HTTP 协议在实际工程中的解码实现细节，提升网络编程能力。