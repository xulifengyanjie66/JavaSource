Tomcat的启动类是位于`org.apache.catalina.startup.Bootstrap`包下的类，会调用其main方法，该方法的源码如下:

```java
public static void main(String[] args) {

        synchronized (daemonLock) {
            if (daemon == null) {
                Bootstrap bootstrap = new Bootstrap();
                try {
                    bootstrap.init();
                } catch (Throwable t) {
                    handleThrowable(t);
                    log.error("Init exception", t);
                    return;
                }
                daemon = bootstrap;
            } else {
                Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
            }
        }

        try {
            String command = "start";
            if (args.length > 0) {
                command = args[args.length - 1];
            }

            switch (command) {
                case "startd":
                    args[args.length - 1] = "start";
                    daemon.load(args);
                    daemon.start();
                    break;
                case "stopd":
                    args[args.length - 1] = "stop";
                    daemon.stop();
                    break;
                case "start":
                    daemon.setAwait(true);
                    daemon.load(args);
                    daemon.start();
                    if (null == daemon.getServer()) {
                        System.exit(1);
                    }
                    break;
                case "stop":
                    daemon.stopServer(args);
                    break;
                case "configtest":
                    daemon.load(args);
                    if (null == daemon.getServer()) {
                        System.exit(1);
                    }
                    System.exit(0);
                    break;
                default:
                    break;
            }
        } catch (Throwable t) {
            Throwable throwable = t;
            if (throwable instanceof InvocationTargetException && throwable.getCause() != null) {
                throwable = throwable.getCause();
            }
            handleThrowable(throwable);
            System.exit(1);
        }
    }
```

我们看到主要创建了Bootstrap类，调用其init方法，该方法的源码如下:

```java
public void init() throws Exception {

        initClassLoaders();

        Thread.currentThread().setContextClassLoader(catalinaLoader);

        SecurityClassLoad.securityClassLoad(catalinaLoader);

        // Load our startup class and call its process() method
        if (log.isTraceEnabled()) {
            log.trace("Loading startup class");
        }
        Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
        Object startupInstance = startupClass.getConstructor().newInstance();

        // Set the shared extensions class loader
        if (log.isTraceEnabled()) {
            log.trace("Setting startup class properties");
        }
        String methodName = "setParentClassLoader";
        Class<?>[] paramTypes = new Class[1];
        paramTypes[0] = Class.forName("java.lang.ClassLoader");
        Object[] paramValues = new Object[1];
        paramValues[0] = sharedLoader;
        Method method = startupInstance.getClass().getMethod(methodName, paramTypes);
        method.invoke(startupInstance, paramValues);

        catalinaDaemon = startupInstance;
}
```

我们可以看到又调用了initClassLoaders方法，那么我们就先分析它的这个执行路径Bootstrap.main->Bootstrap.init()->Bootstrap.initClassLoaders(),initClassLoaders方法的源码如下:

```java
   private void initClassLoaders() {
        try {
            commonLoader = createClassLoader("common", null);
            if (commonLoader == null) {
                // no config file, default to this loader - we might be in a 'single' env.
                commonLoader = this.getClass().getClassLoader();
            }
            catalinaLoader = createClassLoader("server", commonLoader);
            sharedLoader = createClassLoader("shared", commonLoader);
        } catch (Throwable t) {
            handleThrowable(t);
            log.error("Class loader creation threw exception", t);
            System.exit(1);
        }
    }
```

可以明显看出这个方法创建了三个类加载器，分别是commonLoader、catalinaLoader、sharedLoader，其中catalinaLoader、sharedLoader的父类加载器是commonLoader这正是我上篇文章说的那样的类加载器关系，那现在重点来看createClassLoader方法，它的源码如下:

```java
 private ClassLoader createClassLoader(String name, ClassLoader parent) throws Exception {

        String value = CatalinaProperties.getProperty(name + ".loader");
        if ((value == null) || (value.isEmpty())) {
            return parent;
        }

        value = replace(value);

        List<Repository> repositories = new ArrayList<>();

        String[] repositoryPaths = getPaths(value);

        for (String repository : repositoryPaths) {
            // Check for a JAR URL repository
            try {
                URI uri = new URI(repository);
                @SuppressWarnings("unused")
                URL url = uri.toURL();
                repositories.add(new Repository(repository, RepositoryType.URL));
                continue;
            } catch (IllegalArgumentException | MalformedURLException | URISyntaxException e) {
                // Ignore
            }

            // Local repository
            if (repository.endsWith("*.jar")) {
                repository = repository.substring(0, repository.length() - "*.jar".length());
                repositories.add(new Repository(repository, RepositoryType.GLOB));
            } else if (repository.endsWith(".jar")) {
                repositories.add(new Repository(repository, RepositoryType.JAR));
            } else {
                repositories.add(new Repository(repository, RepositoryType.DIR));
            }
        }

        return ClassLoaderFactory.createClassLoader(repositories, parent);
    }
```

读取`catalina.properties`配置文件中的`common.loader="${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar"`

解析出对应的项目目录封装为repositoryPaths字符串数组，遍历该数组封装为`List<Repository> repositories`,最后调用`ClassLoaderFactory.createClassLoader(repositories, parent)`方法创建一个URLClassLoader类加载器，加载路径就是从这些路径上的jar包下加载类，同理server、shared都是类似加载规则，但是默认情况下catalina.properties都没配置这两个路径，由源码可知如果没有配置默认返回父类加载器，所以

catalinaLoader=sharedLoader=commonLoader。

好，我们继续回到Bootstrap的init方法，执行这段代码`Thread.currentThread().setContextClassLoader(catalinaLoader);`设置当前线程类加载器是catalinaLoader，反射调用创建org.apache.catalina.startup.Catalina对象并设置Catalina的成员属性的ClassLoader为sharedLoader，最后把BootStrap对象的成员属性catalinaDaemon设置为反射创建的org.apache.catalina.startup.Catalina对象。

执行到这里返回到Bootstrap的load方法，该方法里面又反射调用Catalina的load方法，该方法的源码如下:

```java
public void load() {

        if (loaded) {
            return;
        }
        // Parse main server.xml
        parseServerXml(true);
        Server s = getServer();
        if (s == null) {
            return;
        }

        getServer().setCatalina(this);
        getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
        getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

        // Stream redirection
        initStreams();

        // Start the new server
        try {
            getServer().init();
        } catch (LifecycleException e) {
            if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
                throw new Error(e);
            } else {
                log.error(sm.getString("catalina.initError"), e);
            }
        }

        if (log.isInfoEnabled()) {
            log.info(sm.getString("catalina.init",
                    Long.toString(TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - t1))));
        }
    }
```

先调用parseServerXml方法解析conf/server.xml配置文件，把它封装为Server对象，Server对象里面又有多个Service对象，就行server.xml那样嵌套的结构一样。

设置Server对象的Catalina、catalinaHome、catalinaBase属性分为Catalina对象、catalinaHome文件路径、catalinaBase文件路径，调用Server的init方法，Server的实现类是StandardServer，它的类继承结构是这样的，`public final class StandardServer extends LifecycleMBeanBase implements Server`,LifecycleMBeanBase是一个抽象类，继承了LifecycleBase抽象类实现Lifecycle接口，LifecycleBase抽象类实现了接口Lifecycle的`init`、`start`、`destroy`等方法、init、start方法里面分别又调用了`initInternal`、`startInternal`方法，把这些实现都留给了子类去实现，这是典型的模板方法设计模式。

好，那我们现在来看看StandardServer的init方法源码，实际上是它的父类LifecycleBase的init方法，源码如下:

```java
    public final synchronized void init() throws LifecycleException {
        if (!state.equals(LifecycleState.NEW)) {
            invalidTransition(BEFORE_INIT_EVENT);
        }

        try {
            setStateInternal(LifecycleState.INITIALIZING, null, false);
            initInternal();
            setStateInternal(LifecycleState.INITIALIZED, null, false);
        } catch (Throwable t) {
            handleSubClassException(t, "lifecycleBase.initFail", toString());
        }
    }
```

又调用了setStateInternal方法，该方法源码如下:

```java
 private synchronized void setStateInternal(LifecycleState state, Object data, boolean check)
            throws LifecycleException {

        this.state = state;
        String lifecycleEvent = state.getLifecycleEvent();
        if (lifecycleEvent != null) {
            fireLifecycleEvent(lifecycleEvent, data);
        }
    }
```

主要做的事情主要有:

1.修改内部状态为**INITIALIZING**

2.触发 `BEFORE_INIT_EVENT`

3.通知所有 `LifecycleListener`

子类真正实现init逻辑主要是调用initInternal方法，它的实现类是StandardServer，该方法源码如下:

```java
protected void initInternal() throws LifecycleException {

        if (getCatalina() != null) {
            ClassLoader cl = getCatalina().getParentClassLoader();
            while (cl != null && cl != ClassLoader.getSystemClassLoader()) {
                if (cl instanceof URLClassLoader) {
                    URL[] urls = ((URLClassLoader) cl).getURLs();
                    for (URL url : urls) {
                        if (url.getProtocol().equals("file")) {
                            try {
                                File f = new File(url.toURI());
                                if (f.isFile() && f.getName().endsWith(".jar")) {
                                    ExtensionValidator.addSystemResource(f);
                                }
                            } catch (URISyntaxException | IOException e) {
                                // Ignore
                            }
                        }
                    }
                }
                cl = cl.getParent();
            }
        }
        // Initialize our defined Services
        for (Service service : findServices()) {
            service.init();
        }
 }
```

此处先留下if (getCatalina() != null) 逻辑，后续看明白再来补充。

接着变量Service调用其init方法，只需要看它的initInternal方法，该方法源码如下:

```java
 protected void initInternal() throws LifecycleException {

        if (engine != null) {
            engine.init();
        }

        // Initialize any Executors
        for (Executor executor : findExecutors()) {
            if (executor instanceof JmxEnabled) {
                ((JmxEnabled) executor).setDomain(getDomain());
            }
            executor.init();
        }

        // Initialize mapper listener
        mapperListener.init();

        // Initialize our defined Connectors
        for (Connector connector : findConnectors()) {
            connector.init();
        }
  }
```

这里我们重点分析一下connector.init()的逻辑，主要是初始化 Tomcat 的所有网络监听端口和底层通信组件，但并不开始接收请求,Connector是Tomcat的网络入口，它的作用如下:

|      职责      |     举例     |
| :------------: | :----------: |
|    监听端口    | 8080 / 8443  |
|    协议解析    | HTTP / HTTPS |
|    线程模型    |  NIO / APR   |
| 把请求交给容器 |    Engine    |

它的initInternal源码如下:

```
protected void initInternal() throws LifecycleException {

        super.initInternal();

        if (protocolHandler == null) {
            throw new LifecycleException(sm.getString("coyoteConnector.protocolHandlerInstantiationFailed"));
        }
        adapter = new CoyoteAdapter(this);
        protocolHandler.setAdapter(adapter);

        try {
            protocolHandler.init();
        } catch (Exception e) {
            throw new LifecycleException(sm.getString("coyoteConnector.protocolHandlerInitializationFailed"), e);
        }
    }
```

首先判断protocolHandler是否等于null，如果是null抛出异常，一般情况下它不会是null,它是在解析Server.xml时候赋值的，默认是Http11NioProtocol，它主要解析Http协议的类，接着创建一个CoyoteAdapter对象，它主要 负责把解析后的请求，交给 Tomcat 容器（Engine / Host / Context / Servlet）。接着调用Http11NioProtocol的init方法，该方法里面实例化一个HttpParser对象，它负责解析http协议，这里面它又调用了父类AbstractProtocol的init方法，这里主要调用了NioEndpoint的init方法，NioEndpoint主要负责网络监听的类，作用主要包括bind 端口、启动启动 Acceptor 线程等。

各个组件的init执行完成后都会执行其setStateInternal方法，该方法主要是

1.修改内部状态为**INITIALIZED**

2.触发 `AFTER_INIT_EVENT`

3.通知所有 `LifecycleListener`

至此Server的init方法执行完成了，`Server.init()` 的作用：初始化整个 Tomcat 实例的“运行骨架”，创建并初始化所有 Service（包括 Engine 和 Connector），但此时还不对外提供服务。

此时会调用到Catalina.start方法，该方法源码如下:

```java
public void start() {

        if (getServer() == null) {
            load();
        }

        if (getServer() == null) {
            log.fatal(sm.getString("catalina.noServer"));
            return;
        }

        long t1 = System.nanoTime();

        try {
            getServer().start();
        } catch (LifecycleException e) {
            log.fatal(sm.getString("catalina.serverStartFail"), e);
            try {
                getServer().destroy();
            } catch (LifecycleException e1) {
                log.debug(sm.getString("catalina.destroyFail"), e1);
            }
            return;
        }

        if (log.isInfoEnabled()) {
            log.info(sm.getString("catalina.startup",
                    Long.toString(TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - t1))));
        }

        if (generateCode) {
            generateLoader();
        }

        if (useShutdownHook) {
            if (shutdownHook == null) {
                shutdownHook = new CatalinaShutdownHook();
            }
            Runtime.getRuntime().addShutdownHook(shutdownHook);
            LogManager logManager = LogManager.getLogManager();
            if (logManager instanceof ClassLoaderLogManager) {
                ((ClassLoaderLogManager) logManager).setUseShutdownHook(false);
            }
        }

        if (await) {
            await();
            stop();
        }
 }
```

首先调用的是Server的start方法，继而调用所有子节点的start方法，我们来看一下Engine的start方法，因为Server的start方法没什么逻辑，首先需要说明一下Engine不是像Server和Service直接继承Lifecycle，而是直接继承的Container接口，那么Container是什么那？为啥有的实现了 `Container`，有的实现了 `Lifecycle`，**`Container`** 解决的是请求如何被路由和处理的问题；**`Lifecycle` 解决的是「组件如何被启动、停止和管理」的问题。**只有Engine、Host、

Context、Wrapper直接继承了该接口，因为它们都涉及到请求的处理,StandardEngine的startInternal方法调用了父类ContainerBase的startInternal方法，该方法源码如下:

```java
 protected void startInternal() throws LifecycleException {

        reconfigureStartStopExecutor(getStartStopThreads());

        Container[] children = findChildren();
        List<Future<Void>> results = new ArrayList<>(children.length);
        for (Container child : children) {
            results.add(startStopExecutor.submit(new StartChild(child)));
        }

        MultiThrowable multiThrowable = null;

        for (Future<Void> result : results) {
            try {
                result.get();
            } catch (Throwable t) {
                log.error(sm.getString("containerBase.threadedStartFailed"), t);
                if (multiThrowable == null) {
                    multiThrowable = new MultiThrowable();
                }
                multiThrowable.add(t);
            }

        }
        if (multiThrowable != null) {
            throw new LifecycleException(sm.getString("containerBase.threadedStartFailed"),
                    multiThrowable.getThrowable());
        }

        // Start the Valves in our pipeline (including the basic), if any
        if (pipeline instanceof Lifecycle) {
            ((Lifecycle) pipeline).start();
        }

        setState(LifecycleState.STARTING);

        if (backgroundProcessorDelay > 0) {
            monitorFuture = Container.getService(ContainerBase.this).getServer().getUtilityExecutor()
                    .scheduleWithFixedDelay(new ContainerBackgroundProcessorMonitor(), 0, 60, TimeUnit.SECONDS);
        }
    }
```

先是调用了reconfigureStartStopExecutor方法实例化一个startStopExecutor变量是InlineExecutorService，它实现了ExecutorService线程池对象，获取子Container对象数组，即是StandardHost,在异步线程池中提交调用StandardHost的start方法并调用result.get()方法阻塞获取结果，那这里就先需要看一下StandardHost的startInternal方法执行逻辑，该方法源码如下:

```java
protected void startInternal() throws LifecycleException {

        String errorValve = getErrorReportValveClass();
        if ((errorValve != null) && (!errorValve.isEmpty())) {
            try {
                boolean found = false;
                Valve[] valves = getPipeline().getValves();
                for (Valve valve : valves) {
                    if (errorValve.equals(valve.getClass().getName())) {
                        found = true;
                        break;
                    }
                }
                if (!found) {
                    Valve valve = ErrorReportValve.class.getName().equals(errorValve) ? new ErrorReportValve() :
                            (Valve) Class.forName(errorValve).getConstructor().newInstance();
                    getPipeline().addValve(valve);
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                log.error(sm.getString("standardHost.invalidErrorReportValveClass", errorValve), t);
            }
        }
        super.startInternal();
    }
```

获取Valve集合，上篇文章说过，配置Valve是在所有执行Context之前执行用于做一些事情，比如Server.xml中的这段配置:

```xml
  <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
```

用户记录日志操作,这里又在Pipeline中添加一个ErrorReportValve一个个执行。

接着调用父类的startInternal方法，此时去获取StandardHost的子Container集合，此处获取的是空的，但是它发布了一个STARTING实际，HostConfig监听了

STARTING，它处理逻辑主要是添加Context，即webapps下面的所有配置，它的deployApps方法如下:

```java
 protected void deployApps() {
        File appBase = host.getAppBaseFile();
        File configBase = host.getConfigBaseFile();
        String[] filteredAppPaths = filterAppPaths(appBase.list());
        // Deploy XML descriptors from configBase
        deployDescriptors(configBase, configBase.list());
        // Deploy WARs
        deployWARs(appBase, filteredAppPaths);
        // Deploy expanded folders
        deployDirectories(appBase, filteredAppPaths);
    }
```

首先调用解析deployDescriptors方法解析conf\Catalina\localhost下的配置文件，主要调用的是HostConfig的deployDescriptor方法,生成一个StandardContext对象，在生成StandardContext对象过程中实例化它的成员属性ApplicationContext context，它代表的是一个Web上下文对象，反射创建一个ContextConfig对象并把它设置为StandardContext的监听对象，当调用StandardContext的start方法时候会调用ContextConfig的webConfig解析web.xml，然后调用`host.addChild(context)`把Context加入到Host对象中,下面重点看一下StandardContext的startInternal方法，该方法源码简化如下:

```java
protected void startInternal() throws LifecycleException {
    setResources(new StandardRoot(this));
    resourcesStart();
    if (getLoader() == null) {
        WebappLoader webappLoader = new WebappLoader();
        webappLoader.setDelegate(getDelegate());
        setLoader(webappLoader);
    }
    ClassLoader oldCCL = bindThread();
    Loader loader = getLoader();
    if (loader instanceof Lifecycle) {
        ((Lifecycle) loader).start();
    }
}
```

先说明一下StandardRoot，它是 **Tomcat 中 Web 应用的“资源视图层”，负责统一访问 Web 应用的所有资源**包括：

- WEB-INF/web.xml
- WEB-INF/classes
- WEB-INF/lib/*.jar
- 静态资源（html / js / css / 图片）
- 外部挂载目录（docBase 之外的资源）

首先创建一个StandardRoot对象，它内部维护多个 **WebResourceSet**，每个代表一类资源，用一个表格说明里面的多个WebResourceSet都代表什么：

| ResourceSet |       来源        |
| :---------: | :---------------: |
|    Main     |   webapp 根目录   |
|    Class    |  WEB-INF/classes  |
|     Jar     | WEB-INF/lib/*.jar |
| Pre / Post  |   外部挂载资源    |

然后调用该对象的start方法，该方法源码如下:

```java
protected void startInternal() throws LifecycleException {
        mainResources.clear();

        main = createMainResourceSet();

        mainResources.add(main);

        for (List<WebResourceSet> list : allResources) {
            if (list != classResources) {
                for (WebResourceSet webResourceSet : list) {
                    webResourceSet.start();
                }
            }
        }
        processWebInfLib();
        for (WebResourceSet classResource : classResources) {
            classResource.start();
        }

        cache.enforceObjectMaxSizeLimit();

        setState(LifecycleState.STARTING);
}
```

调用createMainResourceSet方法创建一个代表conf\Catalina\localhost配置文件下的docBase指定的文件夹对象，实现类是DirResourceSet，这块暂时没看明白先空下。

接着继续执行StandardContext的startInternal方法，创建一个WebappLoader，它是为 Web 应用准备“自己的类加载器（WebappClassLoader),它主要是扫描**WEB-INF/classes**、**WEB-INF/lib/*.jar**、**Servlet / Filter / Listener**等。

接着调用bindThread方法绑定当前线程的ClassLoader为WebappClassLoader，第一次oldCCL是null,调用Loader的startInternal方法进行绑定，我们重点来看一下它的方法源码是如何实现的:

```java
protected void startInternal() throws LifecycleException {

        try {

            classLoader = createClassLoader();
            classLoader.setResources(context.getResources());
            classLoader.setDelegate(this.delegate);
            setClassPath();
        } catch (Throwable t) {
            Throwable throwable = ExceptionUtils.unwrapInvocationTargetException(t);
            ExceptionUtils.handleThrowable(throwable);
            throw new LifecycleException(sm.getString("webappLoader.startError"), throwable);
        }

        setState(LifecycleState.STARTING);
}
```

调动`createClassLoader`方法创建一个ParallelWebappClassLoader对象并设置它的父的Classloader是URLClassLoader，**从这以后就用这个ClassLoader去加载每个应用WEB-INF、WEB-CLASSES下面的Listener、Filter、Servlet**，知道这一点很重要，接着设置WebResourceRoot路径，设置delegate属性默认是false,代表是子优先加载（即是Web应用），如果是true表示让父去加载（比如JDK/Tomcat容器），一般情况我们都设置为false,表示要让ParallelWebappClassLoader去加载，调用setClassPath方法把自己的路径和父Classloader能加载的路径赋值到成员属性classpath上，最终复制到ServletContext的key是`org.apache.catalina.jsp_classpath`,value值是classpath。

此时又返回到StandardContext的startInternal方法接着执行`bindThread`方法把创建出来的ParallelWebappClassLoader绑定到当前线程。

接着调用`fireLifecycleEvent(CONFIGURE_START_EVENT, null)`发布Context已经启动的监听事件，此时会调用到`ContextConfig`的configureStart方法，又调用webConfig，这个方法就比较重要了，它是解析web.xml主要方法，Listener、Filter、Servlet等解析都在这里。这里我跳一些主要的分析。

首先是调用processServletContainerInitializers方法，该方法源码如下:

```java
protected void processServletContainerInitializers() {

        List<ServletContainerInitializer> detectedScis;
        try {
            WebappServiceLoader<ServletContainerInitializer> loader = new WebappServiceLoader<>(context);
            detectedScis = loader.load(ServletContainerInitializer.class);
        } catch (IOException ioe) {
            log.error(sm.getString("contextConfig.servletContainerInitializerFail", context.getName()), ioe);
            ok = false;
            return;
        }

        for (ServletContainerInitializer sci : detectedScis) {
            initializerClassMap.put(sci, new HashSet<>());

            HandlesTypes ht;
            try {
                ht = sci.getClass().getAnnotation(HandlesTypes.class);
            } catch (Exception e) {
                if (log.isDebugEnabled()) {
                    log.debug(sm.getString("contextConfig.sci.debug", sci.getClass().getName()), e);
                } else {
                    log.info(sm.getString("contextConfig.sci.info", sci.getClass().getName()));
                }
                continue;
            }
            if (ht == null) {
                continue;
            }
            Class<?>[] types = ht.value();
            if (types == null) {
                continue;
            }

            for (Class<?> type : types) {
                if (type.isAnnotation()) {
                    handlesTypesAnnotations = true;
                } else {
                    handlesTypesNonAnnotations = true;
                }
                typeInitializerMap.computeIfAbsent(type, k -> new HashSet<>()).add(sci);
            }
        }
    }
```

可以看到这块有个主要类**`ServletContainerInitializer`**,它是Servlet3.0 引入的“容器启动扩展点”，让框架可以在 Web 应用启动时，由容器主动回调，而不是靠 web.xml，主要通过SPI机制去应用的`META-INF/services/javax.servlet.ServletContainerInitializer`文件去找实现ServletContainerInitializer接口实现类，然后可以自定义添加自定义的Servlet,先看WebappServiceLoader的load方法，该方法源码如下:

```java
 public List<T> load(Class<T> serviceType) throws IOException {
        String configFile = SERVICES + serviceType.getName();

        ClassLoader loader = context.getParentClassLoader();
        Enumeration<URL> containerResources;
        if (loader == null) {
            containerResources = ClassLoader.getSystemResources(configFile);
        } else {
            containerResources = loader.getResources(configFile);
        }
        LinkedHashSet<String> containerServiceClassNames = new LinkedHashSet<>();
        Set<URL> containerServiceConfigFiles = new HashSet<>();
        while (containerResources.hasMoreElements()) {
            URL containerServiceConfigFile = containerResources.nextElement();
            containerServiceConfigFiles.add(containerServiceConfigFile);
            parseConfigFile(containerServiceClassNames, containerServiceConfigFile);
        }

        if (containerSciFilterPattern != null) {
            containerServiceClassNames.removeIf(s -> containerSciFilterPattern.matcher(s).find());
        }
        LinkedHashSet<String> applicationServiceClassNames = new LinkedHashSet<>();

        List<String> orderedLibs = (List<String>) servletContext.getAttribute(ServletContext.ORDERED_LIBS);

        if (orderedLibs == null) {
            Enumeration<URL> allResources = servletContext.getClassLoader().getResources(configFile);
            while (allResources.hasMoreElements()) {
                URL serviceConfigFile = allResources.nextElement();
                if (!containerServiceConfigFiles.contains(serviceConfigFile)) {
                    parseConfigFile(applicationServiceClassNames, serviceConfigFile);
                }
            }
        } else {
            URL unpacked = servletContext.getResource(CLASSES + configFile);
            if (unpacked != null) {
                parseConfigFile(applicationServiceClassNames, unpacked);
            }

            for (String lib : orderedLibs) {
                URL jarUrl = servletContext.getResource(LIB + lib);
                if (jarUrl == null) {
                    continue;
                }

                String base = jarUrl.toExternalForm();
                URL url;
                if (base.endsWith("/")) {
                    URI uri;
                    try {
                        uri = new URI(base + configFile);
                    } catch (URISyntaxException e) {
                        throw new IOException(e);
                    }
                    url = uri.toURL();
                } else {
                    url = JarFactory.getJarEntryURL(jarUrl, configFile);
                }
                try {
                    parseConfigFile(applicationServiceClassNames, url);
                } catch (FileNotFoundException e) {
                }
            }
        }
        containerServiceClassNames.addAll(applicationServiceClassNames);

        if (containerServiceClassNames.isEmpty()) {
            return Collections.emptyList();
        }
        return loadServices(serviceType, containerServiceClassNames);
 }
```

先获取应用的父加载器的ClassLoader去$CATALINA_HOME/lib寻找`META-INF/services/javax.servlet.ServletContainerInitializer`,如果有加入到有顺序的LinkedHashSet中这样可以保证父ClassLoader加载的在前面，接着在执行自己的ClassLoader去寻找同样加入到LinkedHashSet中，然后调用loadServices方法反射创建`META-INF/services/javax.servlet.ServletContainerInitializer`指定的类并把它实例化为对象。

接着执行返回到processServletContainerInitializers方法得到一个`List<ServletContainerInitializer> detectedScis`集合，遍历该集合把它放入到Map<ServletContainerInitializer,Set<Class<?>>>数据结构中，其中key是得到的ServletContainerInitializer对象，value是一个Set集合，接着判断ServletContainerInitializer上是否有HandlesTypes注解类，`@HandlesTypes` 是 **Servlet 3.0+ 中 ServletContainerInitializer的一个关键注解**，它的作用主要是 **告诉容器在启动阶段哪些类对这个 SCI 可能感兴趣**，方便容器提前扫描和传递给 `onStartup` 方法。
它的主要查找逻辑是找到所有HandlesTypes注解修饰的类所有实现类最终加入到initializerClassMap的Value中。
接着会执行ContextConfig的processClass方法,该方法主要是查找是否有@WebServlet、WebFilter、WebListener注解把它们扫描到对应的定义类中去，我们来看看这个方法源码:
```java
protected void processClass(WebXml fragment, JavaClass clazz) {
        AnnotationEntry[] annotationsEntries = clazz.getAnnotationEntries();
        if (annotationsEntries != null) {
            String className = clazz.getClassName();
            for (AnnotationEntry ae : annotationsEntries) {
                String type = ae.getAnnotationType();
                if ("Ljavax/servlet/annotation/WebServlet;".equals(type)) {
                    processAnnotationWebServlet(className, ae, fragment);
                } else if ("Ljavax/servlet/annotation/WebFilter;".equals(type)) {
                    processAnnotationWebFilter(className, ae, fragment);
                } else if ("Ljavax/servlet/annotation/WebListener;".equals(type)) {
                    fragment.addListener(className);
                } else {
                    // Unknown annotation - ignore
                }
            }
        }
}
```
首先获取类名称，然后获取它的类上的注解，如果注解是WebServlet,调用processAnnotationWebServlet方法，该方法源码如下:
```java
protected void processAnnotationWebServlet(String className, AnnotationEntry ae, WebXml fragment) {
        String servletName = null;
        List<ElementValuePair> evps = ae.getElementValuePairs();
        for (ElementValuePair evp : evps) {
           String name = evp.getNameString();
           if ("name".equals(name)) {
            servletName = evp.getValue().stringifyValue();
            break;
           }
        }
        ServletDef servletDef = new ServletDef();
        servletDef.setServletName(servletName);
        servletDef.setServletClass(className);
}
```
先获取WebFilter注解的name名称然后赋值给变量servletName，变量evps判断是否有urlPatterns属性，代表的是Servlet的请求路径，如果有解析出来赋值给变量urlPatterns，它是一个String数组
判断注解是否有loadOnStartup属性如果有设置到ServletDef上,判断是否有initParams属性






