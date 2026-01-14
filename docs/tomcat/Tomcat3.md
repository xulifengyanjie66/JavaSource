# Tomcat源码分析三(Tomcat请求源码分析)
## 一、前言
在调试 Spring MVC 或编写自定义 Filter、Valve 时，我们经常会遇到一些问题：
- Request 是在什么时候创建的？
- org.apache.catalina.connector.Request 和 HttpServletRequest 之间是什么关系？
- 请求是如何一步步交给应用层 Servlet 的？

带着这些问题，本文将以 Tomcat 源码为线索，从 Connector 到 Container，梳理一次 HTTP 请求在 Tomcat 内部的完整执行路径,本文直接是从原始的请求Request开始往下分析,不会具体分析
请求是如果变成Request对象的，本文基于Tomcat9。

## 二、请求源码分析
上文说到Tomcat在启动时候创建两个线程，一个线程名称是Poller线程、一个是Acceptor线程，Acceptor线程负责接收请求，然后Poller线程负责处理请求中的事件，比如读写事件然后解析Http请求封装为原始的Request对象，这都是网络io的知识，这篇文章我会从封装为Request对象的源码开始分析是如何一步步调用到Servlet的，这块开始的源码是位于`CoyoteAdapter`类，调用的方法是`service`,该方法源码如下:

```java
public void service(org.apache.coyote.Request req, org.apache.coyote.Response res) throws Exception {
    Request request = connector.createRequest();
    request.setCoyoteRequest(req);
    Response response = connector.createResponse();
    response.setCoyoteResponse(res);
    request.setResponse(response);
    response.setRequest(request);
}
```
通过调用`connector.createRequest()`方法创建一个Request,它实现了`HttpServletRequest`对象，实现了Servlet规范，同时把原始的Request对象设置到成员属性org.apache.coyote.Request coyoteRequest上，然后调用`connector.createResponse()`方法创建一个Response对象，它实现了`HttpServletResponse`对象，同样是实现了Servlet规范，把原始的Response对象设置到成员属性`org.apache.coyote.Response coyoteResponse`上，然后分别设置HttpServletRequest的成员属性是HttpServletResponse,HttpServletResponse的成员属性是HttpServletRequest。

然后调用`org.apache.catalina.mapper.Mapper的map`方法根据请求路径匹配Context，这块的代码就不详细分析了，感兴趣的同志可以自己去看看源码。

接着调用`connector.getService().getContainer().getPipeline().getFirst().invoke(request, response)`方法，经过调用来到`StandardHostValve`的invoke方法，该方法源码如下:

```java
public void invoke(Request request, Response response) throws IOException, ServletException {

        Context context = request.getContext();
        try {
            context.bind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);
            try {
                if (!response.isErrorReportRequired()) {
                    context.getPipeline().getFirst().invoke(request, response);
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                container.getLogger().error(sm.getString("standardHostValve.exception", request.getRequestURI()), t);
            }
            response.setSuspended(false);

            Throwable t = (Throwable) request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);
            
            if (!context.getState().isAvailable()) {
                return;
            }
            if (response.isErrorReportRequired()) {
                AtomicBoolean result = new AtomicBoolean(false);
                response.getCoyoteResponse().action(ActionCode.IS_IO_ALLOWED, result);
                if (result.get()) {
                    if (t != null) {
                        throwable(request, response, t);
                    } else {
                        status(request, response);
                    }
                }
            }

            if (!request.isAsync() && !asyncAtStart) {
                context.fireRequestDestroyEvent(request.getRequest());
            }
        } finally {
            if (ACCESS_SESSION) {
                request.getSession(false);
            }

            context.unbind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER);
        }
    }
```
该方法主要是获取到请求路径解析出来的`StandardContext`对象，调用bind方法，这个方法我已经在启动时候分析过了它主要是切换当前的ClassLoader为`ParallelWebappClassLoader`,这样做的目的是为了加载指定目录下的Filter、Servlet等。
接着调用`context.getPipeline().getFirst().invoke(request, response)`方法，该方法会最后执行到`StandardContextValve`类的invoke方法，该方法源码如下:

```java
public void invoke(Request request, Response response) throws IOException, ServletException {
    Wrapper wrapper = request.getWrapper();
    if (wrapper == null || wrapper.isUnavailable()) {
        response.sendError(HttpServletResponse.SC_NOT_FOUND);
        return;
    }
    wrapper.getPipeline().getFirst().invoke(request, response);
}
```
还是从请求解析获取出Wrapper对象，它里面包含Servlet对象，判断如果wrapper为空返回错误，接着调用`wrapper.getPipeline().getFirst().invoke(request, response)`,这时候请求来到类`StandardWrapperValve`的invoke方法，这里是真正执行Filter、Servlet实现逻辑的地方，下面我们详细看一下它的源码实现:

```java
public void invoke(Request request, Response response) throws IOException, ServletException {
        boolean unavailable = false;
        Throwable throwable = null;
        long t1 = System.currentTimeMillis();
        requestCount.incrementAndGet();
        StandardWrapper wrapper = (StandardWrapper) getContainer();
        Servlet servlet = null;
        Context context = (Context) wrapper.getParent();
        try {
            if (!unavailable) {
                servlet = wrapper.allocate();
            }
        } catch (UnavailableException e) {
            container.getLogger().error(sm.getString("standardWrapper.allocateException", wrapper.getName()), e);
            checkWrapperAvailable(response, wrapper);
        } catch (ServletException e) {
            container.getLogger().error(sm.getString("standardWrapper.allocateException", wrapper.getName()),
                    StandardWrapper.getRootCause(e));
            throwable = e;
            exception(request, response, e);
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            container.getLogger().error(sm.getString("standardWrapper.allocateException", wrapper.getName()), t);
            throwable = t;
            exception(request, response, t);
        }
        ApplicationFilterChain filterChain = ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
        Container container = this.container;
        try {
            if ((servlet != null) && (filterChain != null)) {
                if (context.getSwallowOutput()) {
                    try {
                        SystemLogHandler.startCapture();
                        if (request.isAsyncDispatching()) {
                            request.getAsyncContextInternal().doInternalDispatch();
                        } else {
                            filterChain.doFilter(request.getRequest(), response.getResponse());
                        }
                    } finally {
                        String log = SystemLogHandler.stopCapture();
                        if (log != null && !log.isEmpty()) {
                            context.getLogger().info(log);
                        }
                    }
                } else {
                    if (request.isAsyncDispatching()) {
                        request.getAsyncContextInternal().doInternalDispatch();
                    } else {
                        filterChain.doFilter(request.getRequest(), response.getResponse());
                    }
                }

            }
        } catch (BadRequestException e) {
            if (container.getLogger().isDebugEnabled()) {
                container.getLogger().debug(
                        sm.getString("standardWrapper.serviceException", wrapper.getName(), context.getName()), e);
            }
            throwable = e;
            exception(request, response, e, HttpServletResponse.SC_BAD_REQUEST);
        } catch (CloseNowException e) {
            if (container.getLogger().isDebugEnabled()) {
                container.getLogger().debug(
                        sm.getString("standardWrapper.serviceException", wrapper.getName(), context.getName()), e);
            }
            throwable = e;
            exception(request, response, e);
        } catch (IOException ioe) {
            container.getLogger()
                    .error(sm.getString("standardWrapper.serviceException", wrapper.getName(), context.getName()), ioe);
            throwable = ioe;
            exception(request, response, ioe);
        } catch (UnavailableException e) {
            container.getLogger()
                    .error(sm.getString("standardWrapper.serviceException", wrapper.getName(), context.getName()), e);
            wrapper.unavailable(e);
            checkWrapperAvailable(response, wrapper);
        } catch (ServletException e) {
            Throwable rootCause = StandardWrapper.getRootCause(e);
            if (!(rootCause instanceof BadRequestException)) {
                container.getLogger().error(sm.getString("standardWrapper.serviceExceptionRoot", wrapper.getName(),
                        context.getName(), e.getMessage()), rootCause);
            }
            throwable = e;
            exception(request, response, e);
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            container.getLogger()
                    .error(sm.getString("standardWrapper.serviceException", wrapper.getName(), context.getName()), t);
            throwable = t;
            exception(request, response, t);
        } finally {
            if (filterChain != null) {
                filterChain.release();
            }
            try {
                if (servlet != null) {
                    wrapper.deallocate(servlet);
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                container.getLogger().error(sm.getString("standardWrapper.deallocateException", wrapper.getName()), t);
                if (throwable == null) {
                    throwable = t;
                    exception(request, response, t);
                }
            }
            try {
                if ((servlet != null) && (wrapper.getAvailable() == Long.MAX_VALUE)) {
                    wrapper.unload();
                }
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                container.getLogger().error(sm.getString("standardWrapper.unloadException", wrapper.getName()), t);
                if (throwable == null) {
                    exception(request, response, t);
                }
            }
        }
}
```
获取Wrapper调用它的allocate方法，这个方法逻辑主要是判断它里面的成员变量Servlet是否为空，为空说明还没有实例化该Servlet对象，会用ParallelWebappClassLoader类加载器实例化对象并调用它的init方法。

接着执行这段代码`ApplicationFilterChain filterChain = ApplicationFilterFactory.createFilterChain(request, wrapper, servlet)`,它的方法如下:

```java
public static ApplicationFilterChain createFilterChain(ServletRequest request, Wrapper wrapper, Servlet servlet) {

        if (servlet == null) {
            return null;
        }
        ApplicationFilterChain filterChain;
        if (request instanceof Request) {
            Request req = (Request) request;
            if (Globals.IS_SECURITY_ENABLED) {
                filterChain = new ApplicationFilterChain();
            } else {
                filterChain = (ApplicationFilterChain) req.getFilterChain();
                if (filterChain == null) {
                    filterChain = new ApplicationFilterChain();
                    req.setFilterChain(filterChain);
                }
            }
        } else {
            filterChain = new ApplicationFilterChain();
        }

        filterChain.setServlet(servlet);
        filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());
        StandardContext context = (StandardContext) wrapper.getParent();
        FilterMap filterMaps[] = context.findFilterMaps();
        if (filterMaps == null || filterMaps.length == 0) {
            return filterChain;
        }

        DispatcherType dispatcher = (DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);

        String requestPath = FilterUtil.getRequestPath(request);

        String servletName = wrapper.getName();

        for (FilterMap filterMap : filterMaps) {
            if (!matchDispatcher(filterMap, dispatcher)) {
                continue;
            }
            if (!FilterUtil.matchFiltersURL(filterMap, requestPath)) {
                continue;
            }
            ApplicationFilterConfig filterConfig =
                    (ApplicationFilterConfig) context.findFilterConfig(filterMap.getFilterName());
            if (filterConfig == null) {
                log.warn(sm.getString("applicationFilterFactory.noFilterConfig", filterMap.getFilterName()));
                continue;
            }
            filterChain.addFilter(filterConfig);
        }
        for (FilterMap filterMap : filterMaps) {
            if (!matchDispatcher(filterMap, dispatcher)) {
                continue;
            }
            if (!matchFiltersServlet(filterMap, servletName)) {
                continue;
            }
            ApplicationFilterConfig filterConfig =
                    (ApplicationFilterConfig) context.findFilterConfig(filterMap.getFilterName());
            if (filterConfig == null) {
                log.warn(sm.getString("applicationFilterFactory.noFilterConfig", filterMap.getFilterName()));
                continue;
            }
            filterChain.addFilter(filterConfig);
        }
        return filterChain;
 }
```
创建一个`ApplicationFilterChain`，它实现了Servlet的规范接口`FilterChain`,它主要是包含了Servlet和一些Filter的执行数组，并把创建好的对象设置到HttpServletRequest对象的成员变量FilterChain filterChain上，设置`ApplicationFilterChain`成员属性为匹配的Servelt,然后获取StandardContext中保存的FilterMap数组，它里面包含所有的Filter对象和url路径,如果FilterMap数组为空直接返回ApplicationFilterChain对象，否则根据当前的请求路径和FilterMap里面的每个url路径做匹配，如果匹配成功得到ApplicationFilterConfig对象，调用`filterChain.addFilter(filterConfig)`,把ApplicationFilterConfig对象放在`ApplicationFilterChain`成员属性ApplicationFilterConfig[]上。

接着又来遍历一次FilterMap找出和servletName名称匹配的Filter，如果是匹配所有返回true，如果不是就根据Filter里面配置的Servlet名称和当前Servlet名称做匹配如果有同样加入到`ApplicationFilterChain`成员属性ApplicationFilterConfig[]上。

得到了ApplicationFilterChain对象以后就调用了它的doFilter方法，传入`HttpServletRequest`对象和`HttpServletResponse`对象，内部又调用了internalDoFilter方法，该方法源码如下:

```java
private void internalDoFilter(ServletRequest request, ServletResponse response)
            throws IOException, ServletException {

        if (pos < n) {
            ApplicationFilterConfig filterConfig = filters[pos++];
            try {
                Filter filter = filterConfig.getFilter();

                if (request.isAsyncSupported() && !(filterConfig.getFilterDef().getAsyncSupportedBoolean())) {
                    request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR, Boolean.FALSE);
                }
                if (Globals.IS_SECURITY_ENABLED) {
                    final ServletRequest req = request;
                    final ServletResponse res = response;
                    Principal principal = ((HttpServletRequest) req).getUserPrincipal();

                    Object[] args = new Object[] { req, res, this };
                    SecurityUtil.doAsPrivilege("doFilter", filter, classType, args, principal);
                } else {
                    filter.doFilter(request, response, this);
                }
            } catch (IOException | ServletException | RuntimeException e) {
                throw e;
            } catch (Throwable t) {
                t = ExceptionUtils.unwrapInvocationTargetException(t);
                ExceptionUtils.handleThrowable(t);
                throw new ServletException(sm.getString("filterChain.filter"), t);
            }
            return;
        }
 }       
```
获取ApplicationFilterConfig对象中的Filter对象然后调用它的doFilter方法，参数除了包含`HttpServletRequest`对象和`HttpServletResponse`对象，还包含了当前调用方法的对象即ApplicationFilterChain，以便下一个Filter可以继续调用其它的Filter对象。

Filter都执行完成以后调用`servlet.service(request, response);`进而执行Servlet的doGet或者doPost方法,真正执行我们的业务逻辑。

执行完以后按照调用链路一路返回到`StandardHostValve`类的invoke方法的finally代码块执行`context.unbind(Globals.IS_SECURITY_ENABLED, MY_CLASSLOADER)`切换为Tomcat所使用的Classloader

## 三、总结
通过以上源码分析，我们可以清晰地看到Tomcat处理HTTP请求的完整路径：

请求流转路径：
Connector → CoyoteAdapter → StandardHostValve → StandardContextValve → StandardWrapperValve → ApplicationFilterChain → Servlet

关键设计要点：

- 分层架构：Tomcat采用典型的责任链模式，通过Pipeline-Valve机制实现请求的逐层传递

- 请求封装：将底层org.apache.coyote.Request封装为Servlet规范的HttpServletRequest

- 类加载隔离：通过context.bind()/context.unbind()实现Web应用类加载器的切换

- Filter链构建：运行时动态构建匹配请求的Filter链，支持URL模式和Servlet名称两种匹配方式

- Servlet生命周期：Wrapper负责Servlet的延迟初始化、缓存和销毁
