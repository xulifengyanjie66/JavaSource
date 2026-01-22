# Spring Security 是如何注入 Tomcat Filter 链的 —— 启动与请求源码分析

## 一、前言
在前面的文章中，我们已经从源码层面分析了 Tomcat 的启动流程、请求处理模型，以及 Spring Boot 是如何在嵌入式 Tomcat 场景下完成 Servlet 和 Filter 注册的。

在此基础上，一个自然的问题是：
像 Spring Security 这样的安全框架，是如何介入 Tomcat 的 Filter 链，并在请求到达业务代码之前完成认证和鉴权的？

本文将以 Tomcat 的 Filter 机制为起点，结合 Spring Security 的启动流程，深入分析 Spring Security 的 Filter 是如何被创建、注册到 Tomcat 中的，以及一次请求是如何在 Security FilterChain 中流转的。

## 二、DelegatingFilterProxy注册到Tomcat和Spring Security启动源码

在Springboot自动配置类中org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration，注入了这样一个
bean对象,代码如下:

```java
	@Bean
	@ConditionalOnBean(name = DEFAULT_FILTER_NAME)
	public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(
			SecurityProperties securityProperties) {
		DelegatingFilterProxyRegistrationBean registration = new DelegatingFilterProxyRegistrationBean(
				DEFAULT_FILTER_NAME);
		registration.setOrder(securityProperties.getFilter().getOrder());
		registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
		return registration;
	}
```

可以看出它注入到容器的条件是需要存在一个名称为`springSecurityFilterChain`的Bean,那么先来看这个bean是在哪里注入的那,它是在配置类`org.springframework.security.config.annotation.web.configuration.WebSecurityConfiguration`中通@Bean方式注入的，源码如下:

```java
@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasFilterChain = !this.securityFilterChains.isEmpty();
		if (!hasFilterChain) {
			this.webSecurity.addSecurityFilterChainBuilder(() -> {
				this.httpSecurity.authorizeHttpRequests((authorize) -> authorize.anyRequest().authenticated());
				this.httpSecurity.formLogin(Customizer.withDefaults());
				this.httpSecurity.httpBasic(Customizer.withDefaults());
				return this.httpSecurity.build();
			});
		}
		for (SecurityFilterChain securityFilterChain : this.securityFilterChains) {
			this.webSecurity.addSecurityFilterChainBuilder(() -> securityFilterChain);
		}
		for (WebSecurityCustomizer customizer : this.webSecurityCustomizers) {
			customizer.customize(this.webSecurity);
		}
		return this.webSecurity.build();
	}
```
指定bean的名称是`springSecurityFilterChain`,其中`securityFilterChains`是通过依赖注入进来的，代码如下:

```java
	@Autowired(required = false)
	void setFilterChains(List<SecurityFilterChain> securityFilterChains) {
		this.securityFilterChains = securityFilterChains;
	}
```
最终返回的是一个`FilterChainProxy`对象，它实现了`Filter`对象，虽然它实现了`Filter`对象，但是它没有加入到Tomcat的容器中，真正加入Tomcat的Filter是`DelegatingFilterProxyRegistrationBean`代表的Filter，原理就是它实现了`ServletContextInitializer`接口，上文我分析了SpringBoot嵌入Tomcat原理并加入Servlet和Filter逻辑可知会调用`DelegatingFilterProxyRegistrationBean`的`getFilter`方法把`DelegatingFilterProxy`类加入到Tomcat的容器中，
它同样实现了`Filter`接口，它的源码如下:

```java
	public DelegatingFilterProxy getFilter() {
		return new DelegatingFilterProxy(this.targetBeanName, getWebApplicationContext()) {

			@Override
			protected void initFilterBean() throws ServletException {
				// Don't initialize filter bean on init()
			}

		};
	}
```
而DelegatingFilterProxy持有`Spring`容器中的name名称是`springSecurityFilterChain`所代表的FilterChainProxy对象，这样就把Tomcat容器和Spring容器关联起来了，如果有请求先是被Tomcat注册的Filter类DelegatingFilterProxy处理，然后可以把请求转发给FilterChainProxy处理，这个主要大前提理解了就能捋顺后面的处理逻辑了。

上面说到`FilterChainProxy`会自动调用`setFilterChains`方法注入一个集合对象`List<SecurityFilterChain>`，这个可以在自己项目中自定义。

我的项目配置SecurityFilterChain是如下这样的：

```java
    @Bean
    protected SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
                httpSecurity
                                .cors(Customizer.withDefaults())
                                .csrf(AbstractHttpConfigurer::disable)
                                .sessionManagement(c -> c.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .headers(c -> c.frameOptions(HeadersConfigurer.FrameOptionsConfig::disable))

                .exceptionHandling(c -> c.authenticationEntryPoint(authenticationEntryPoint)
                        .accessDeniedHandler(accessDeniedHandler));

        Multimap<HttpMethod, String> permitAllUrls = getPermitAllUrlsFromAnnotations();

        httpSecurity

                .authorizeHttpRequests(c -> c

                    .requestMatchers(HttpMethod.GET, "/*.html", "/*.html", "/*.css", "/*.js").permitAll()

                    .requestMatchers(HttpMethod.GET, permitAllUrls.get(HttpMethod.GET).toArray(new String[0])).permitAll()
                    .requestMatchers(HttpMethod.POST, permitAllUrls.get(HttpMethod.POST).toArray(new String[0])).permitAll()
                    .requestMatchers(HttpMethod.PUT, permitAllUrls.get(HttpMethod.PUT).toArray(new String[0])).permitAll()
                    .requestMatchers(HttpMethod.DELETE, permitAllUrls.get(HttpMethod.DELETE).toArray(new String[0])).permitAll()
                    .requestMatchers(HttpMethod.HEAD, permitAllUrls.get(HttpMethod.HEAD).toArray(new String[0])).permitAll()
                    .requestMatchers(HttpMethod.PATCH, permitAllUrls.get(HttpMethod.PATCH).toArray(new String[0])).permitAll()

                    .requestMatchers(securityProperties.getPermitAllUrls().toArray(new String[0])).permitAll()

                    .requestMatchers(buildAppApi("/**")).permitAll()
                        //老系统请求得接口放行
                        .requestMatchers("/admin-api/medica/unified-entrance/getPageContractList").permitAll()
                )

                .authorizeHttpRequests(c -> authorizeRequestsCustomizers.forEach(customizer -> customizer.customize(c)))

                .authorizeHttpRequests(c -> c.anyRequest().authenticated());


        httpSecurity.addFilterBefore(authenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
        return httpSecurity.build();
    }
```
其中HttpSecurity对象也是通过自动配置类配置好的，这里我就不详细讲解这个配置了。

我挑一些重点得说执行这段代码`exceptionHandling(c -> c.authenticationEntryPoint(authenticationEntryPoint)`,`exceptionHandling`的方法签名如下:

```java
public HttpSecurity exceptionHandling(
			Customizer<ExceptionHandlingConfigurer<HttpSecurity>> exceptionHandlingCustomizer) throws Exception {
		exceptionHandlingCustomizer.customize(getOrApply(new ExceptionHandlingConfigurer<>()));
		return HttpSecurity.this;
}
```

`Customizer`里面的customize方法接收一个对象进行处理，它的方法签名是这样的,`void customize(T t)`,最终调用的是`ExceptionHandlingConfigurer`的authenticationEntryPoint方法，该方法源码如下:

```java
	public ExceptionHandlingConfigurer<H> authenticationEntryPoint(AuthenticationEntryPoint authenticationEntryPoint) {
		this.authenticationEntryPoint = authenticationEntryPoint;
		return this;
	}
```
最终把AuthenticationEntryPoint对象设置到ExceptionHandlingConfigurer的成员属性上，`AuthenticationEntryPoint`主要是处理用户没登录时候返回给前端的处理逻辑，例如我这个项目就是在用户没登录时候返回给用户404提示。

```java
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException e) {
        log.debug("[commence][访问 URL({}) 时，没有z登录]", request.getRequestURI(), e);
                ServletUtils.writeJSON(response, CommonResult.error(UNAUTHORIZED));
    }
```
调用`authorizeHttpRequests`方法往`HttpSecurity`对象添加一个`AuthorizeHttpRequestsConfigurer`对象，在实例化`AuthorizationManagerRequestMatcherRegistry`对象过程中初始化一个`AuthorizationManagerRequestMatcherRegistry`成员变量，传入到Lambda的对象就是`AuthorizationManagerRequestMatcherRegistry`对象，然后调用该对象的requestMatchers方法

调用`Multimap<HttpMethod, String> permitAllUrls = getPermitAllUrlsFromAnnotations();`获取所有标有`PermitAll`注解的方法,以key为请求方法，value
是所有标有`PermitAll`注解的方法url路径，接着调用`.requestMatchers(HttpMethod.GET, permitAllUrls.get(HttpMethod.GET).toArray(new String[0])).permitAll()`。
先看requestMatchers方法源码如下:
```java
public C requestMatchers(HttpMethod method, String... patterns) {
		if (!mvcPresent) {
			return requestMatchers(RequestMatchers.antMatchersAsArray(method, patterns));
		}
		if (!(this.context instanceof WebApplicationContext)) {
			return requestMatchers(RequestMatchers.antMatchersAsArray(method, patterns));
		}
		WebApplicationContext context = (WebApplicationContext) this.context;
		ServletContext servletContext = context.getServletContext();
		if (servletContext == null) {
			return requestMatchers(RequestMatchers.antMatchersAsArray(method, patterns));
		}
		boolean isProgrammaticApiAvailable = isProgrammaticApiAvailable(servletContext);
		List<RequestMatcher> matchers = new ArrayList<>();
		for (String pattern : patterns) {
			AntPathRequestMatcher ant = new AntPathRequestMatcher(pattern, (method != null) ? method.name() : null);
			MvcRequestMatcher mvc = createMvcMatchers(method, pattern).get(0);
			if (isProgrammaticApiAvailable) {
				matchers.add(resolve(ant, mvc, servletContext));
			}
			else {
				matchers.add(new DeferredRequestMatcher(() -> resolve(ant, mvc, servletContext), mvc, ant));
			}
		}
		return requestMatchers(matchers.toArray(new RequestMatcher[0]));
}
```
如果mvcPresent是false说明是非Web环境直接用Ant风格匹配url,不是WebApplicationContext也用Ant风格匹配url,servletContext如果为空说明
Tomcat 还没 fully ready，不能立刻用 MVC，isProgrammaticApiAvailable判断当前Spring MVC 的 HandlerMappingIntrospector 是否已经可安全使用，如果是调用
```java
if (isProgrammaticApiAvailable) {
    matchers.add(resolve(ant, mvc, servletContext));
}
```
调用resolve方法看当前环境能否用mvc,如果能优先使用否则使用Ant,如果isProgrammaticApiAvailable是false执行以下代码:
```java
matchers.add(new DeferredRequestMatcher(() -> resolve(ant, mvc, servletContext), mvc, ant));
```
创建一个DeferredRequestMatcher对象，意思是现在不决定用 Ant 还是 MVC，等“第一次请求来时”再决定
最后调用`requestMatchers(matchers.toArray(new RequestMatcher[0]))`最后返回`new AuthorizedUrl(requestMatchers)`,AuthorizedUrl包装了RequestMatcher对象
requestMatchers方法执行返回以后会调用permitAll方法，该方法源码如下:
```java
	public AuthorizationManagerRequestMatcherRegistry permitAll() {
			return access(permitAllAuthorizationManager);
    }
```
其中permitAllAuthorizationManager是一个Lambda表达式，它的声明是这样的：`static final AuthorizationManager<RequestAuthorizationContext> permitAllAuthorizationManager = (a,
o) -> new AuthorizationDecision(true);`可以看出它返回一个true对象的AuthorizationDecision代表不拦截，这个在分析请求时候就可以看到它的用处。

再来看看access方法源码:
```java
public AuthorizationManagerRequestMatcherRegistry access(
        AuthorizationManager<RequestAuthorizationContext> manager) {
    Assert.notNull(manager, "manager cannot be null");
    return AuthorizeHttpRequestsConfigurer.this.addMapping(this.matchers, manager);
}
```
可以看出是调用它的外部类对象AuthorizeHttpRequestsConfigurer的addMapping方法，该方法源码如下:
```java
	private AuthorizationManagerRequestMatcherRegistry addMapping(List<? extends RequestMatcher> matchers,
			AuthorizationManager<RequestAuthorizationContext> manager) {
		for (RequestMatcher matcher : matchers) {
			this.registry.addMapping(matcher, manager);
		}
		return this.registry;
	}
```
又调用它的成员变量AuthorizationManagerRequestMatcherRegistry的addMapping方法，它的方法源码如下:
```java
		private void addMapping(RequestMatcher matcher, AuthorizationManager<RequestAuthorizationContext> manager) {
			this.unmappedMatchers = null;
			this.managerBuilder.add(matcher, manager);
			this.mappingCount++;
		}
```
在这里又调用了成员变量RequestMatcherDelegatingAuthorizationManager.Builder的add方法，该方法源码如下:
```java
public Builder add(RequestMatcher matcher, AuthorizationManager<RequestAuthorizationContext> manager) {
    Assert.state(!this.anyRequestConfigured, "Can't add mappings after anyRequest");
    Assert.notNull(matcher, "matcher cannot be null");
    Assert.notNull(manager, "manager cannot be null");
    this.mappings.add(new RequestMatcherEntry<>(matcher, manager));
    return this;
}
```
最终把matcher对象和manager对象封装为RequestMatcherEntry对象添加到List<RequestMatcherEntry<AuthorizationManager<RequestAuthorizationContext>>>成员属性上。

接着调用authorizeHttpRequests(c -> c.anyRequest().authenticated())方法，anyRequest方法的源码如下:
```java
	public C anyRequest() {
		Assert.state(!this.anyRequestConfigured, "Can't configure anyRequest after itself");
		C configurer = requestMatchers(ANY_REQUEST);
		this.anyRequestConfigured = true;
		return configurer;
	}
```
其中ANY_REQUEST的实现是AnyRequestMatcher，它代表的是匹配任意路径，然后调用authenticated方法，它的方法源码如下:
```java
	public AuthorizationManagerRequestMatcherRegistry authenticated() {
			return access(AuthenticatedAuthorizationManager.authenticated());
   }
```
AuthenticatedAuthorizationManager.authenticated()对象的实现是AuthenticatedAuthorizationManager，它与permitAll代表的Manager不同，它是需要校验是否有认证信息的
这个在请求时候在做具体分析，最终还是调用AuthorizationManagerRequestMatcherRegistry.access方法把AnyRequestMatcher、AuthenticatedAuthorizationManager放到List<RequestMatcherEntry<AuthorizationManager<RequestAuthorizationContext>>>成员属性上
最后调用httpSecurity.build()方法，该方法源码如下:
```java
protected final O doBuild() throws Exception {
    synchronized (this.configurers) {
        this.buildState = BuildState.INITIALIZING;
        beforeInit();
        init();
        this.buildState = BuildState.CONFIGURING;
        beforeConfigure();
        configure();
        this.buildState = BuildState.BUILDING;
        O result = performBuild();
        this.buildState = BuildState.BUILT;
        return result;
    }
}
```
通过遍历SecurityConfigurer的config方法往httpSecurity添加一些Filter,这里就分析几个主要的SecurityConfigurer，先来看一下
ExceptionHandlingConfigurer的configure方法，该方法源码如下:
```java
	public void configure(H http) {
		AuthenticationEntryPoint entryPoint = getAuthenticationEntryPoint(http);
		ExceptionTranslationFilter exceptionTranslationFilter = new ExceptionTranslationFilter(entryPoint,
				getRequestCache(http));
		AccessDeniedHandler deniedHandler = getAccessDeniedHandler(http);
		exceptionTranslationFilter.setAccessDeniedHandler(deniedHandler);
		exceptionTranslationFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());
		exceptionTranslationFilter = postProcess(exceptionTranslationFilter);
		http.addFilter(exceptionTranslationFilter);
	}
```
可以看出手动创建一个ExceptionTranslationFilter对象并设置一些成员属性比如AuthenticationEntryPoint，然后添加到httpSecurity上,
ExceptionTranslationFilter 是 Spring Security 中“安全异常的统一翻译器”,专门捕获 AuthenticationException 和 AccessDeniedException异常并将其转换为登录跳转、401 或 403 响应

再来看一下AuthorizeHttpRequestsConfigurer的configure方法源码:
```java
public void configure(H http) {
    AuthorizationManager<HttpServletRequest> authorizationManager = this.registry.createAuthorizationManager();
    AuthorizationFilter authorizationFilter = new AuthorizationFilter(authorizationManager);
    authorizationFilter.setAuthorizationEventPublisher(this.publisher);
    authorizationFilter.setShouldFilterAllDispatcherTypes(this.registry.shouldFilterAllDispatcherTypes);
    authorizationFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());
    http.addFilter(postProcess(authorizationFilter));
}
```
先调用AuthorizationManagerRequestMatcherRegistry对象的createAuthorizationManager方法，该方法主要是创建一个RequestMatcherDelegatingAuthorizationManager对象并把List<RequestMatcherEntry<AuthorizationManager<RequestAuthorizationContext>>> mappings对象
传递给它的成员变量。

最终再创建一个ObservationAuthorizationManager对象里面包装RequestMatcherDelegatingAuthorizationManager对象，这两个类都是实现了AuthorizationManager类，它们是组合关系，真正实现的逻辑在于RequestMatcherDelegatingAuthorizationManager对象。

接着创建AuthorizationFilter对象，它是一个Filter对象，把ObservationAuthorizationManager传递给它的成员变量，调用getSecurityContextHolderStrategy方法获取SecurityContextHolderStrategy对象，它的实现是TransmittableThreadLocalSecurityContextHolderStrategy，这个类是我项目自定义的类，它里面的ThreadLocal实现是阿里巴巴的
TransmittableThreadLocal，这个类可以不同线程池中持有某个资源，比如认证信息，把TransmittableThreadLocalSecurityContextHolderStrategy也设置到Filter对象成员属性上,最后把AuthorizationFilter
添加到HttpSecurity上。

再来看一下AnonymousConfigurer的方法源码，它的源码处理逻辑是在init和configure里面都有，下面分别贴出源码:
```java
public void init(H http) {
		if (this.authenticationProvider == null) {
			this.authenticationProvider = new AnonymousAuthenticationProvider(getKey());
		}
		if (this.authenticationFilter == null) {
			this.authenticationFilter = new AnonymousAuthenticationFilter(getKey(), this.principal, this.authorities);
			this.authenticationFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());
		}
		this.authenticationFilter.setSecurityContextHolderStrategy(getSecurityContextHolderStrategy());
		this.authenticationProvider = postProcess(this.authenticationProvider);
		http.authenticationProvider(this.authenticationProvider);
}
```
```java
	public void configure(H http) {
		this.authenticationFilter.afterPropertiesSet();
		http.addFilter(this.authenticationFilter);
	}
```
主要是实例化一个AnonymousAuthenticationFilter对象，设置成员属性principal为anonymousUser、List<GrantedAuthority> authorities为ROLE_ANONYMOUS、设置SecurityContextHolderStrategy然后把这个Filter添加到
HttpSecurity中，这个Filter主要作用是即使没登陆也放个Authentication认证对象，后面请求我会具体分析。

好到这里开始执行doBuild方法里面的performBuild，该方法源码如下:
```java
protected DefaultSecurityFilterChain performBuild() {
    this.filters.sort(OrderComparator.INSTANCE);
    List<Filter> sortedFilters = new ArrayList<>(this.filters.size());
    for (Filter filter : this.filters) {
        sortedFilters.add(((OrderedFilter) filter).filter);
    }
    return new DefaultSecurityFilterChain(this.requestMatcher, sortedFilters);
}
```
主要是给Filter进行排序，order小的Filter在最前面，最后创建一个DefaultSecurityFilterChain对象传入AnyRequestMatcher、排好序的Filter集合返回DefaultSecurityFilterChain。
我的项目中Filter集合主要有这些，其中TokenAuthenticationFilter是我自定义的Filter用于添加认证信息的。

![Filter.png](..%2Fimg%2FFilter.png)

## 三、Spring Security请求源码分析

首先我这个项目请求先经过我自定义的TokenAuthenticationFilter，我是通过校验请求传递的Token去查询Redis里面是否有有效token如果有构造一个LoginUser对象，然后创建一个
UsernamePasswordAuthenticationToken对象，代码如下:
```java
    private static Authentication buildAuthentication(LoginUser loginUser, HttpServletRequest request) {
        UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(
        loginUser, null, Collections.emptyList());
        authenticationToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
        return authenticationToken;
   }
```
然后调用SecurityContextHolder.getContext().setAuthentication(authentication)方法把UsernamePasswordAuthenticationToken放到ThreadLocal中，前面说过这个ThreadLocal的实现是
阿里巴巴的TransmittableThreadLocal，可以在多个线程之间共享资源，这是用户正确登录的情况下，如果用户没有登录就不设置这个，那么ThreadLocal里面就没有认证信息，下面
看AnonymousAuthenticationFilter是如何处理这两种不同情况的,它的doFilter源码如下:
```java
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        Supplier<SecurityContext> deferredContext = this.securityContextHolderStrategy.getDeferredContext();
        this.securityContextHolderStrategy
            .setDeferredContext(defaultWithAnonymous((HttpServletRequest) req, deferredContext));
        chain.doFilter(req, res);
    }
```
声明一个Supplier表达式，它的实现是如下:
  ```java
       public SecurityContext getContext() {
        SecurityContext ctx = CONTEXT_HOLDER.get();
        if (ctx == null) {
            ctx = createEmptyContext();
            CONTEXT_HOLDER.set(ctx);
        }
        return ctx;
    }
  ```
从ThreadLocal中取出SecurityContext,如果当前用户已经登录那ctx指定不为空直接返回，如果没登录就调用createEmptyContext创建一个SecurityContextImpl对象然后设置到 ThreadLocal中。

接着调用defaultWithAnonymous方法该方法源码如下:
```java
private SecurityContext defaultWithAnonymous(HttpServletRequest request, SecurityContext currentContext) {
		Authentication currentAuthentication = currentContext.getAuthentication();
		if (currentAuthentication == null) {
			Authentication anonymous = createAuthentication(request);
			if (this.logger.isTraceEnabled()) {
				this.logger.trace(LogMessage.of(() -> "Set SecurityContextHolder to " + anonymous));
			}
			else {
				this.logger.debug("Set SecurityContextHolder to anonymous SecurityContext");
			}
			SecurityContext anonymousContext = this.securityContextHolderStrategy.createEmptyContext();
			anonymousContext.setAuthentication(anonymous);
			return anonymousContext;
		}
		else {
			if (this.logger.isTraceEnabled()) {
				this.logger.trace(LogMessage.of(() -> "Did not set SecurityContextHolder since already authenticated "
						+ currentAuthentication));
			}
		}
		return currentContext;
}
```
判断SecurityContext中是否有Authentication对象，如果有的话就是前面的Filter已经设置了我上面说的登录场景，对象是UsernamePasswordAuthenticationToken，那么就直接返回，
如果没有登录调用createAuthentication方法创建一个AnonymousAuthenticationToken对象，源码如下:
```java
	protected Authentication createAuthentication(HttpServletRequest request) {
		AnonymousAuthenticationToken token = new AnonymousAuthenticationToken(this.key, this.principal,
				this.authorities);
		token.setDetails(this.authenticationDetailsSource.buildDetails(request));
		return token;
	}
```
然后把创建好的AnonymousAuthenticationToken对象设置到SecurityContext中相当于设置到ThreadLocal中。
继续执行到AuthorizationFilter的doFilter方法，该方法源码如下:
```java
public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain)
			throws ServletException, IOException {

		HttpServletRequest request = (HttpServletRequest) servletRequest;
		HttpServletResponse response = (HttpServletResponse) servletResponse;

		if (this.observeOncePerRequest && isApplied(request)) {
			chain.doFilter(request, response);
			return;
		}

		if (skipDispatch(request)) {
			chain.doFilter(request, response);
			return;
		}

		String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
		request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
		try {
			AuthorizationDecision decision = this.authorizationManager.check(this::getAuthentication, request);
			this.eventPublisher.publishAuthorizationEvent(this::getAuthentication, request, decision);
			if (decision != null && !decision.isGranted()) {
				throw new AccessDeniedException("Access Denied");
			}
			chain.doFilter(request, response);
		}
		finally {
			request.removeAttribute(alreadyFilteredAttributeName);
		}
	}
```
这里主要调用了AuthorizationManager的check方法，它的实现类上面我说过是ObservationAuthorizationManager对象，它的源码如下:
```java
public AuthorizationDecision check(Supplier<Authentication> authentication, T object) {
		AuthorizationObservationContext<T> context = new AuthorizationObservationContext<>(object);
		Supplier<Authentication> wrapped = () -> {
			context.setAuthentication(authentication.get());
			return context.getAuthentication();
		};
		Observation observation = Observation.createNotStarted(this.convention, () -> context, this.registry).start();
		try (Observation.Scope scope = observation.openScope()) {
			AuthorizationDecision decision = this.delegate.check(wrapped, object);
			context.setDecision(decision);
			if (decision != null && !decision.isGranted()) {
				observation.error(new AccessDeniedException(
						this.messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access Denied")));
			}
			return decision;
		}
		catch (Throwable ex) {
			observation.error(ex);
			throw ex;
		}
		finally {
			observation.stop();
		}
	}
```
这里面又调用了delegate.check方法，这个delegate的实现是前面说的RequestMatcherDelegatingAuthorizationManager对象，Supplier<Authentication> authentication这是存储在
ThreadLocal中的Authentication对象，RequestMatcherDelegatingAuthorizationManager对象的check方法源码如下:
```java
	public AuthorizationDecision check(Supplier<Authentication> authentication, HttpServletRequest request) {
		if (this.logger.isTraceEnabled()) {
			this.logger.trace(LogMessage.format("Authorizing %s", request));
		}
		for (RequestMatcherEntry<AuthorizationManager<RequestAuthorizationContext>> mapping : this.mappings) {

			RequestMatcher matcher = mapping.getRequestMatcher();
			MatchResult matchResult = matcher.matcher(request);
			if (matchResult.isMatch()) {
				AuthorizationManager<RequestAuthorizationContext> manager = mapping.getEntry();
				if (this.logger.isTraceEnabled()) {
					this.logger.trace(LogMessage.format("Checking authorization on %s using %s", request, manager));
				}
				return manager.check(authentication,
						new RequestAuthorizationContext(request, matchResult.getVariables()));
			}
		}
		if (this.logger.isTraceEnabled()) {
			this.logger.trace(LogMessage.of(() -> "Denying request since did not find matching RequestMatcher"));
		}
		return DENY;
	}
```
根据请求的url路径匹配AuthorizationManager，前面分析过对于PermitAll修饰的注册的是这样一个Lambda表达式 `AuthorizationManager<RequestAuthorizationContext> permitAllAuthorizationManager = (a, o) -> new AuthorizationDecision(true)`，对于需要认证的是注册的是AuthenticatedAuthorizationManager,现在分别分析它两的不同之处
先看AuthenticatedAuthorizationManager,它的check源码如下:
```java
public AuthorizationDecision check(Supplier<Authentication> authentication, T object) {
    boolean granted = this.authorizationStrategy.isGranted(authentication.get());
    return new AuthorizationDecision(granted);
}
```
调用AuthenticatedAuthorizationStrategy对象isGranted方法，该方法源码如下:
```java
	public boolean isAnonymous(Authentication authentication) {
		if ((this.anonymousClass == null) || (authentication == null)) {
			return false;
		}
		return this.anonymousClass.isAssignableFrom(authentication.getClass());
	}
```
判断当前认证信息的Token是不是AnonymousAuthenticationToken如果是说明需要认证的url是没有登录的，返回new AuthorizationDecision(granted)对象，其中granted值是false,如果登录了granted值是true,
根据这个值判断是否抛出认证失败异常，如果认证失败抛出AccessDeniedException异常，ExceptionTranslationFilter会捕获异常信息然后调用上面注册的AuthenticationEntryPoint对象的commence方法返回给前端一个登录异常提示。

## 四、结束语
通过对 Spring Security 启动和请求流程的分析可以看到，安全框架并不是绕过 Tomcat 实现权限控制，而是严格建立在 Servlet Filter 机制 之上。

Spring Security 在容器启动阶段完成 FilterChain 的构建，并通过标准扩展点将其注册到 Tomcat 的 Filter 链中；在请求阶段，则通过一层层 Security Filter 对请求进行拦截和处理。

当从容器视角重新审视 Spring Security 时，那些看似复杂的安全流程，其实只是一次次标准的 Filter 调用而已。
