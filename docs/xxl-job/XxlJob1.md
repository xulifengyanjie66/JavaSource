# XXL-JOB执行器启动与注册源码分析

## 一、前言

在分布式任务调度架构中，XXL-JOB 以其轻量级、易集成的特性被广泛应用于定时任务的管理与执行。本文将从源码层面深入分析其执行器（Executor）的启动机制，特别是通过 XxlJobSpringExecutor 这一 Spring Bean 的配置与初始化过程。我们将重点探讨其配置参数的实际作用、执行器的注册流程，以及底层 Netty 服务与调度中心（Admin）的通信机制，帮助开发者更好地理解其内部工作原理。

## 二、启动注册源码分析

项目中一般配置xxl-job都是写一个这样的自动配置Bean,代码如下:
```java
@Bean
public XxlJobSpringExecutor xxlJobExecutor() {
    logger.info(">>>>>>>>>>> xxl-job config init.");
    XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
    xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
    xxlJobSpringExecutor.setAppname(appname);
    xxlJobSpringExecutor.setAddress(address);
    xxlJobSpringExecutor.setIp(ip);
    xxlJobSpringExecutor.setPort(port);
    xxlJobSpringExecutor.setAccessToken(accessToken);
    xxlJobSpringExecutor.setLogPath(logPath);
    xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);

    return xxlJobSpringExecutor;
}
```
创建一个XxlJobSpringExecutor对象，然后设置它的一些成员属性，这些属性都有什么用那，我画了一个表格

|        配置项        |   谁用   |         核心作用         |
| :------------------: | :------: | :----------------------: |
|  **adminAddresses**  | Executor | 告诉执行器：调度中心在哪 |
|     **address**      | Executor | 执行器对外的完整访问地址 |
|        **ip**        | Executor | 执行器暴露给 Admin 的 IP |
|       **port**       | Executor | 执行器接收调度请求的端口 |
|     **logPath**      | Executor |     Job 执行日志存哪     |
| **logRetentionDays** | Executor |  日志保留多久，自动清理  |

**一、adminAddresses**

 执行器启动时候需要把自己注册到注册中心，就是根据这个指定的url来注册的，然后执行器定时发送心跳,可以配置高可用，一个可能配置例子如下:

```xml
http://a:8080/xxl-job-admin,http://b:8080/xxl-job-admin
```

**二、address / ip / port**

1.address优先级最高,这是一个“最终态配置”,Admin 直接用它,ip / port 会被忽略,这种情况用法是Docker、K8s、反向代理、NAT / 多网卡。



2.ip主要告诉Admin回调的时候访问这个ip地址，如果不配置自动取InetAddress.getLocalHost()，多网卡 / 容器里**极容易取错**。



3.port指定 执行器 **Netty 服务监听端口**,Admin调度时候会访问这个指定的ip+端口号，后面源码会分析。

它们如果同时指定记住这个规则就可以了，**address > ip + port > 自动探测**



**三、logPath**

它通常是这样配置的:

```xml
logPath: /data/applogs/xxl-job/jobhandler
```

包括：

- 执行开始 / 结束
- 执行耗时
- 异常堆栈
- `XxlJobHelper.log()` 输出

**四、logRetentionDays**

定时清理过期 Job 日志，它的一个可能配置如下:

```xml
logRetentionDays: 30
```

了解它的基本属性以后，我们来看看这个类的继承类和接口的情况，它的类继承结构图如下:

![xxl-job.png](..%2Fimg%2Fxxl-job.png)

可以看出它继承了XxlJobExecutor类，实现了ApplicationContextAware、SmartInitializingSingleton、DisposableBean接口，
这由Spring原理可以知道再初始化XxlJobSpringExecutor时候可以注入ApplicationContext的对象，再完成所有Bean实例化初始化后会调用对象的
afterSingletonsInstantiated方法，现在我们就重点分析一下这个方法，它是执行注册的关键逻辑,它的源码如下:
```java
 public void afterSingletonsInstantiated() {
    
        initJobHandlerMethodRepository(applicationContext);

        try {
            super.start();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
}
```
1.initJobHandlerMethodRepository方法的逻辑：
它的源码如下:
  ```java
private void initJobHandlerMethodRepository(ApplicationContext applicationContext) {
        if (applicationContext == null) {
            return;
        }
        String[] beanDefinitionNames = applicationContext.getBeanNamesForType(Object.class, false, true);
        for (String beanDefinitionName : beanDefinitionNames) {
            Object bean = applicationContext.getBean(beanDefinitionName);

            Map<Method, XxlJob> annotatedMethods = null;
            try {
                annotatedMethods = MethodIntrospector.selectMethods(bean.getClass(),
                        new MethodIntrospector.MetadataLookup<XxlJob>() {
                            @Override
                            public XxlJob inspect(Method method) {
                                return AnnotatedElementUtils.findMergedAnnotation(method, XxlJob.class);
                            }
                        });
            } catch (Throwable ex) {
                logger.error("xxl-job method-jobhandler resolve error for bean[" + beanDefinitionName + "].", ex);
            }
            if (annotatedMethods==null || annotatedMethods.isEmpty()) {
                continue;
            }
            for (Map.Entry<Method, XxlJob> methodXxlJobEntry : annotatedMethods.entrySet()) {
                Method executeMethod = methodXxlJobEntry.getKey();
                XxlJob xxlJob = methodXxlJobEntry.getValue();
                if (xxlJob == null) {
                    continue;
                }

                String name = xxlJob.value();
                if (name.trim().length() == 0) {
                    throw new RuntimeException("xxl-job method-jobhandler name invalid, for[" + bean.getClass() + "#" + executeMethod.getName() + "] .");
                }
                if (loadJobHandler(name) != null) {
                    throw new RuntimeException("xxl-job jobhandler[" + name + "] naming conflicts.");
                }
                executeMethod.setAccessible(true);

                Method initMethod = null;
                Method destroyMethod = null;

                if (xxlJob.init().trim().length() > 0) {
                    try {
                        initMethod = bean.getClass().getDeclaredMethod(xxlJob.init());
                        initMethod.setAccessible(true);
                    } catch (NoSuchMethodException e) {
                        throw new RuntimeException("xxl-job method-jobhandler initMethod invalid, for[" + bean.getClass() + "#" + executeMethod.getName() + "] .");
                    }
                }
                if (xxlJob.destroy().trim().length() > 0) {
                    try {
                        destroyMethod = bean.getClass().getDeclaredMethod(xxlJob.destroy());
                        destroyMethod.setAccessible(true);
                    } catch (NoSuchMethodException e) {
                        throw new RuntimeException("xxl-job method-jobhandler destroyMethod invalid, for[" + bean.getClass() + "#" + executeMethod.getName() + "] .");
                    }
                }
                registJobHandler(name, new MethodJobHandler(bean, executeMethod, initMethod, destroyMethod));
            }
        }
}
```
方法逻辑比较简单获取所有的Bean,判断Bean的方法上是否由XxlJob注解,如果有的话就封装为Map<Method, XxlJob>对象，然后遍历此Map,取出
XxlJob的注解名称判断如果为空就抛出异常,校验注解名称是否相同如果相同同样抛出错误异常，取出方法Method对象设置accessible属性为true
判断注解是否指定了init和destroy方法，如果有同样获取它的相应的方法设置accessible属性为true,接着调用registJobHandler方法，调用之前
先把bean, executeMethod, initMethod, destroyMethod封装为MethodJobHandler方法，然后把它们放在类XxlJobExecutor的静态成员变量
jobHandlerRepository上，它是一个ConcurrentMap结构，`private static ConcurrentMap<String, IJobHandler> jobHandlerRepository = new ConcurrentHashMap<String, IJobHandler>()`

2.super.start()方法的逻辑:
它的源码如下:
```java
   public void start() throws Exception {

        // init logpath
        XxlJobFileAppender.initLogPath(logPath);

        // init invoker, admin-client
        initAdminBizList(adminAddresses, accessToken);


        // init JobLogFileCleanThread
        JobLogFileCleanThread.getInstance().start(logRetentionDays);
        
        // init executor-server
        initEmbedServer(address, ip, port, appname, accessToken);
    }
```
这里面又调用了一些方法,首先是调用`XxlJobFileAppender.initLogPath(logPath)`初始化日志路径，方法比较简单根据logPath值判断文件路径是否存在，如果不存在就创建。

接着调用`initAdminBizList(adminAddresses, accessToken)`方法该方法源码如下:
```java
private void initAdminBizList(String adminAddresses, String accessToken) throws Exception {
        if (adminAddresses!=null && adminAddresses.trim().length()>0) {
            for (String address: adminAddresses.trim().split(",")) {
                if (address!=null && address.trim().length()>0) {

                    AdminBiz adminBiz = new AdminBizClient(address.trim(), accessToken);

                    if (adminBizList == null) {
                        adminBizList = new ArrayList<AdminBiz>();
                    }
                    adminBizList.add(adminBiz);
                }
            }
        }
    }
```
循环adminAddresses的地址以逗号分隔把遍历的地址和accessToken封装为AdminBizClient对象添加到静态集合List<AdminBiz>中。

接着调用`JobLogFileCleanThread.getInstance().start(logRetentionDays)`方法启动一个定时线程删除过期的日志这里就不详细分析了。

接着调用`initEmbedServer`方法，该方法源码如下:
```java
 private void initEmbedServer(String address, String ip, int port, String appname, String accessToken) throws Exception {

        port = port>0?port: NetUtil.findAvailablePort(9999);
        ip = (ip!=null&&ip.trim().length()>0)?ip: IpUtil.getIp();

        if (address==null || address.trim().length()==0) {
            String ip_port_address = IpUtil.getIpPort(ip, port);   // registry-address：default use address to registry , otherwise use ip:port if address is null
            address = "http://{ip_port}/".replace("{ip_port}", ip_port_address);
        }

        if (accessToken==null || accessToken.trim().length()==0) {
            logger.warn(">>>>>>>>>>> xxl-job accessToken is empty. To ensure system security, please set the accessToken.");
        }
        embedServer = new EmbedServer();
        embedServer.start(address, port, appname, accessToken);
    }
```
可以看出如果指定了address就直接用它，如果没有指定就拼接ip+port作为Admin回调地址。

接着创建一个EmbedServer对象并调用它的start方法，该方法源码如下:
```java
public void start(final String address, final int port, final String appname, final String accessToken) {
        executorBiz = new ExecutorBizImpl();
        thread = new Thread(new Runnable() {

            @Override
            public void run() {
                EventLoopGroup bossGroup = new NioEventLoopGroup();
                EventLoopGroup workerGroup = new NioEventLoopGroup();
                ThreadPoolExecutor bizThreadPool = new ThreadPoolExecutor(
                        0,
                        200,
                        60L,
                        TimeUnit.SECONDS,
                        new LinkedBlockingQueue<Runnable>(2000),
                        new ThreadFactory() {
                            @Override
                            public Thread newThread(Runnable r) {
                                return new Thread(r, "xxl-rpc, EmbedServer bizThreadPool-" + r.hashCode());
                            }
                        },
                        new RejectedExecutionHandler() {
                            @Override
                            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                                throw new RuntimeException("xxl-job, EmbedServer bizThreadPool is EXHAUSTED!");
                            }
                        });


                try {
                    ServerBootstrap bootstrap = new ServerBootstrap();
                    bootstrap.group(bossGroup, workerGroup)
                            .channel(NioServerSocketChannel.class)
                            .childHandler(new ChannelInitializer<SocketChannel>() {
                                @Override
                                public void initChannel(SocketChannel channel) throws Exception {
                                    channel.pipeline()
                                            .addLast(new IdleStateHandler(0, 0, 30 * 3, TimeUnit.SECONDS))  // beat 3N, close if idle
                                            .addLast(new HttpServerCodec())
                                            .addLast(new HttpObjectAggregator(5 * 1024 * 1024))  // merge request & reponse to FULL
                                            .addLast(new EmbedHttpServerHandler(executorBiz, accessToken, bizThreadPool));
                                }
                            })
                            .childOption(ChannelOption.SO_KEEPALIVE, true);

                    ChannelFuture future = bootstrap.bind(port).sync();

                    logger.info(">>>>>>>>>>> xxl-job remoting server start success, nettype = {}, port = {}", EmbedServer.class, port);

                    startRegistry(appname, address);

                    future.channel().closeFuture().sync();

                } catch (InterruptedException e) {
                    if (e instanceof InterruptedException) {
                        logger.info(">>>>>>>>>>> xxl-job remoting server stop.");
                    } else {
                        logger.error(">>>>>>>>>>> xxl-job remoting server error.", e);
                    }
                } finally {
                    try {
                        workerGroup.shutdownGracefully();
                        bossGroup.shutdownGracefully();
                    } catch (Exception e) {
                        logger.error(e.getMessage(), e);
                    }
                }

            }

        });
        thread.setDaemon(true);	
        thread.start();
    }
```
创建一个ExecutorBizImpl对象，它是执行业务的关键类，接着启动一个线程，在run方法里面启动Netty服务并且创建一个业务线程池ThreadPoolExecutor用于执行后续业务操作,这里的Netty是启动的Http服务器，监听Http请求的。
当有http请求时候会由HttpServerCodec、HttpObjectAggregator解析解码为Java对象然后传给EmbedHttpServerHandler的Handler做业务处理。

启动完成以后会调用startRegistry方法向Admin发起注册请求，该方法源码如下:
```java
public void start(final String appname, final String address){

        if (appname==null || appname.trim().length()==0) {
            logger.warn(">>>>>>>>>>> xxl-job, executor registry config fail, appname is null.");
            return;
        }
        if (XxlJobExecutor.getAdminBizList() == null) {
            logger.warn(">>>>>>>>>>> xxl-job, executor registry config fail, adminAddresses is null.");
            return;
        }
        registryThread = new Thread(new Runnable() {
            @Override
            public void run() {
                while (!toStop) {
                    try {
                        RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appname, address);
                        for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
                            try {
                                ReturnT<String> registryResult = adminBiz.registry(registryParam);
                                if (registryResult!=null && ReturnT.SUCCESS_CODE == registryResult.getCode()) {
                                    registryResult = ReturnT.SUCCESS;
                                    logger.debug(">>>>>>>>>>> xxl-job registry success, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                                    break;
                                } else {
                                    logger.info(">>>>>>>>>>> xxl-job registry fail, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                                }
                            } catch (Exception e) {
                                logger.info(">>>>>>>>>>> xxl-job registry error, registryParam:{}", registryParam, e);
                            }

                        }
                    } catch (Exception e) {
                        if (!toStop) {
                            logger.error(e.getMessage(), e);
                        }

                    }

                    try {
                        if (!toStop) {
                            TimeUnit.SECONDS.sleep(RegistryConfig.BEAT_TIMEOUT);
                        }
                    } catch (InterruptedException e) {
                        if (!toStop) {
                            logger.warn(">>>>>>>>>>> xxl-job, executor registry thread interrupted, error msg:{}", e.getMessage());
                        }
                    }
                }
            }
        });
        registryThread.setDaemon(true);
        registryThread.setName("xxl-job, executor ExecutorRegistryThread");
        registryThread.start();
}
```
可以看出如果没有停止的话toStop是false是一个while的死循环目的是定时发送心跳到Admin,定时心跳时间是30秒,发送的参数是封装为RegistryParam对象，该对象
属性主要是registryGroup=executor、registryKey=appname、registryValue=address,然后循环高可用Admin列表循环注册。

### 三、结束语

希望通过本文的讲解，你能对 XXL-JOB 的执行器启动、注册流程和关键配置有更清晰的理解。掌握这些底层机制，不仅能帮助我们更好地定位问题，还能在复杂网络环境或容器化部署中做出合适的配置调整。如果你有相关疑问或补充，欢迎一起交流讨论！