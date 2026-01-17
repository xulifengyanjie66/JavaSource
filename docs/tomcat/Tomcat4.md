# SpringBoot嵌入Tomcat注册Servlet、Filter流程

## 一、前言
在前面的几篇文章中，我们已经从源码层面分析了 Tomcat 的启动流程和请求处理机制，包括 init / start 生命周期以及一次 HTTP 请求在容器内部的完整流转过程。

但在实际项目中，越来越多的应用并不是以“独立 Tomcat + WAR 包”的形式部署，而是通过 Spring Boot 以嵌入式 Tomcat 的方式启动。这也带来了一个常见的疑问：
当没有 web.xml、没有手动部署应用时，Servlet 和 Filter 是在什么时候、又是如何被加载到 Tomcat 中的？

本文将在前文 Tomcat 源码分析的基础上，结合 Spring Boot 启动流程，重点分析 嵌入式 Tomcat 场景下 Servlet、Filter 的注册与生效机制，帮助同志们建立从容器到框架的完整认知。

## 二、源码分析

Springboot启动时候会调用到类对象`AnnotationConfigServletWebServerApplicationContext`的createWebServer方法用于创建启动Tomcat的工厂类，该方法源码如下:

```java
private void createWebServer() {
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			StartupStep createWebServer = getApplicationStartup().start("spring.boot.webserver.create");
			ServletWebServerFactory factory = getWebServerFactory();
			createWebServer.tag("factory", factory.getClass().toString());
			this.webServer = factory.getWebServer(getSelfInitializer());
		
		}
	
}
```
重要的方法是调用`factory.getWebServer(getSelfInitializer())`,factory的实现类是`TomcatServletWebServerFactory`,传入的参数是getSelfInitializer方法，它是一个Lambda表达式，源码是如下:

```java
private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
		return this::selfInitialize;
}
```
```java

private void selfInitialize(ServletContext servletContext) throws ServletException {
    prepareWebApplicationContext(servletContext);
    for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
        beans.onStartup(servletContext);
    }
}

```
调用getWebServer方法主要是实例化嵌入式Tomcat对象，然后设置`Connector`、`Service`、`Engine`、`Host`、`Context`对象等，我们只看重点的代码就是配置Context的代码，源码如下:

```java
protected void configureContext(Context context, ServletContextInitializer[] initializers) {
		TomcatStarter starter = new TomcatStarter(initializers);
		context.addServletContainerInitializer(starter, NO_CLASSES);
}
```
我们看到实例化了一个TomcatStarter对象，它是什么那，`TomcatStarter implements ServletContainerInitializer `,它实现了ServletContainerInitializer接口在分析Tomcat源码时候我们知道会调用实现类的onStartup方法，
所以需要重点看一下TomcatStarter的onStartup方法如下:

```java
@Override
	public void onStartup(Set<Class<?>> classes, ServletContext servletContext) throws ServletException {
		try {
			for (ServletContextInitializer initializer : this.initializers) {
				initializer.onStartup(servletContext);
			}
		}
		catch (Exception ex) {
			this.startUpException = ex;
			if (logger.isErrorEnabled()) {
				logger.error("Error starting Tomcat context. Exception: " + ex.getClass().getName() + ". Message: "
						+ ex.getMessage());
			}
		}
}
```
遍历ServletContextInitializer的onStartup方法此时会执行到Lambda表达式的selfInitialize方法，因为这Lambda就是ServletContextInitializer类型，上面我已经给出源码了，首先调用的是prepareWebApplicationContext方法传入ServletContext对象，该方法源码如下:

```java
protected void prepareWebApplicationContext(ServletContext servletContext) {
		Object rootContext = servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
		if (rootContext != null) {
			if (rootContext == this) {
				throw new IllegalStateException(
						"Cannot initialize context because there is already a root application context present - "
								+ "check whether you have multiple ServletContextInitializers!");
			}
			return;
		}
		servletContext.log("Initializing Spring embedded WebApplicationContext");
		try {
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this);
			if (logger.isDebugEnabled()) {
				logger.debug("Published root WebApplicationContext as ServletContext attribute with name ["
						+ WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
			}
			setServletContext(servletContext);
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - getStartupDate();
				logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
			}
		}
		catch (RuntimeException | Error ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
}
```
从ServletContext上下中获取属性名称为org.springframework.web.context.WebApplicationContext.ROOT的对象，首次获取指定为空，然后把当前对象设置到servletContext上，Value值是`AnnotationConfigServletWebServerApplicationContext`对象，然后把ServletContext对象设置到`AnnotationConfigServletWebServerApplicationContext`对象成员属性上。

接着调用`getServletContextInitializerBeans`，它的方法源码如下:

```java
protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
    return new ServletContextInitializerBeans(getBeanFactory());
}
```

返回一个`ServletContextInitializerBeans`对象，它的类实现了`AbstractCollection`接口是个集合类，里面泛型类是`ServletContextInitializer`,它的构造方法如下:

```java
public ServletContextInitializerBeans(ListableBeanFactory beanFactory,
                                      Class<? extends ServletContextInitializer>... initializerTypes) {
    this.initializers = new LinkedMultiValueMap<>();
    this.initializerTypes = (initializerTypes.length != 0) ? Arrays.asList(initializerTypes)
        : Collections.singletonList(ServletContextInitializer.class);
    addServletContextInitializerBeans(beanFactory);
    addAdaptableBeans(beanFactory);
    this.sortedList = this.initializers.values()
        .stream()
        .flatMap((value) -> value.stream().sorted(AnnotationAwareOrderComparator.INSTANCE))
        .toList();
    logMappings(this.initializers);
}
```
其中调用`addServletContextInitializerBeans`方法，该方法源码如下:

```java
private void addServletContextInitializerBeans(ListableBeanFactory beanFactory) {
		for (Class<? extends ServletContextInitializer> initializerType : this.initializerTypes) {
			for (Entry<String, ? extends ServletContextInitializer> initializerBean : getOrderedBeansOfType(beanFactory,
					initializerType)) {
				addServletContextInitializerBean(initializerBean.getKey(), initializerBean.getValue(), beanFactory);
			}
		}
}
```
调用getOrderedBeansOfType方法从Spring容器中获取List<Entry<String,ServletContextInitializer>>对象，这些对象都是什么那，我们看一下`DispatcherServletAutoConfiguration`自动配置类里面创建了一个Bean对象，它是这么创建的:

```java
@Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
		@ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
		public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet,
				WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig) {
			DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet,
					webMvcProperties.getServlet().getPath());
			registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
			registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
			multipartConfig.ifAvailable(registration::setMultipartConfig);
			return registration;
		}
```
创建一个DispatcherServletRegistrationBean对象，它的条件要求是存在DispatcherServlet对象，在`DispatcherServletAutoConfiguration`里面也创建了一个

`DispatcherServlet`对象所以条件成立，`DispatcherServletRegistrationBean`类的继承结构图如下:

![8b5ba9c6-d966-4c01-bf65-5af1a10a8257.png](..%2Fimg%2F8b5ba9c6-d966-4c01-bf65-5af1a10a8257.png)

从结构图可以看出该类实现了`ServletContextInitializer`接口，那么上面addServletContextInitializerBeans方法的`initializerBean.getValue()`的值就是该对象，此时回到selfInitialize方法执行这段代码:

```java
	for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
			beans.onStartup(servletContext);
	}
```
执行DispatcherServletRegistrationBean对象的`onStartup`方法，该方法源码如下:

```java
public final void onStartup(ServletContext servletContext) throws ServletException {
		String description = getDescription();
		if (!isEnabled()) {
			logger.info(StringUtils.capitalize(description) + " was not registered (disabled)");
			return;
		}
		register(description, servletContext);
	}
```

又调用了register方法，最后又调用到addRegistration方法

```java
	protected ServletRegistration.Dynamic addRegistration(String description, ServletContext servletContext) {
		String name = getServletName();
		return servletContext.addServlet(name, this.servlet);
	}
```

调用`servletContext.addServlet(name, this.servlet)`方法往Context上下文里面添加一个Wrapper,接着设置url的请求路径，Servlet的loadOnStartup参数，这就是Tomcat的核心源码了，前面已经分析过了。

到这里位置可以看出DispatcherServlet已经被添加到Servlet容器中了，等后面的代码执行Tomcat启动就可以调用DispatcherServlet的init方法了。

## 三、结束语

通过对 Spring Boot 嵌入式 Tomcat 启动流程的分析可以看到，Servlet 和 Filter 并不是“凭空生效”的，而是在 Tomcat 容器初始化阶段，由 Spring Boot 主动完成注册并交由容器管理。

从原生 Tomcat 的 init / start 生命周期，到 Spring Boot 通过 ServletContextInitializer 介入容器启动，本质上都是围绕 Servlet 容器模型的同一套规范展开。

当理解了这条从容器到框架的完整链路之后，再回过头来看 DispatcherServlet、Filter、Listener 的加载与执行顺序，就不再是零散的知识点，而是一条清晰的启动时序。