# Tomcat源码分析一(Tomcat目录和配置文件说明)



## 一、Tomcat目录说明



| 文件夹名称 |                       作用                       |
| :--------: | :----------------------------------------------: |
|    bin     |          存放启动和关闭Tomcat的脚本文件          |
|    conf    |             存放Tomcat的各种配置文件             |
|    lib     |              存放Tomcat的依赖jar包               |
|    logs    |               存放Tomcat的日志文件               |
|    temp    |          存放Tomcat运行中产生的临时文件          |
|  webapps   | web应用所在目录，即供外界访问的web资源的存放目录 |
|    work    |                 Tomcat的工作目录                 |

### 1.1 各个配置文件说明

####  **1.1.1 catalina.properties**

 Tomcat的catalina.properties文件位于%CATALINA_HOME%/conf/目录下面，该文件主要配置tomcat的安全设置、类加载设置、不需要扫描的类设置、字符缓存设置四大块。

- 类加载设置

  tomcat的类加载顺序为：

  `Bootstrap -> System -> Common -> WebAppClassLoader(/WEB-INF/classes + /WEB-INF/lib/*.jar)`

  **Common的配置是通过catalina.properties的commons.loader设置的**

  `common.loader="${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar"`

  CATALINA_HOME是Tomcat的安装目录，CATALINA_BASE是Tomcat的工作目录，一个Tomcat可以通过配置CATALINA_BASE来增加多个工作目录，也就是增加多个实例。多个实例各自可以有自己的conf，logs，temp，webapps。

**Web 应用先委派给父类加载器加载**（common + system），再加载自己的 `/WEB-INF/classes` 和 `/WEB-INF/lib`,因为`Common`（common.loader）是 **Web 应用的父类加载器**

**WebAppClassLoader 默认启用双亲委派**

- 先找 common.loader（父类）
- 再找自己 `/WEB-INF/classes`
- 再找自己 `/WEB-INF/lib/*.jar`

`server.loader和shared.loader`

在common.loader加载完毕后，tomcat启动程序会检查catalina.properties文件中配置的server.loader和shared.loader是否设置。如果设置，读取tomcat下对应的server和shared这两个目录的类库。server和shared是对应tomcat目录下的两个目录，在Tomcat中默认是没有，catalina.properties中默认也是没有设置其值。设置方法如下：

```
server.loader=${catalina.base}/server/classes,${catalina.base}/server/lib/*.jar
shared.loader=${catalina.base}/shared/classes,${catalina.base}/shared/lib/*.jar
```

**server.loader的作用是只给 Tomcat 自己用，Web 应用完全看不到。**

**shared.loader多个 Web 应用可共享，但 Tomcat 本身不依赖**。

这里感觉Tomcat 的 `common.loader` 和 `shared.loader`作用相似，下面详细说明一下它们的区别

|       名称        |   对应配置    |                        加载路径                         |         谁可见         |                作用                |
| :---------------: | :-----------: | :-----------------------------------------------------: | :--------------------: | :--------------------------------: |
| CommonClassLoader |    common     |    `${catalina.base}/lib`、`${catalina.home}/lib` 等    | Tomcat + 所有 Web 应用 |   Tomcat 核心类 + 容器级共享 jar   |
| SharedClassLoader | shared.loader | ${catalina.base}/shared/lib`${catalina.home}/shared/lib |      仅 Web 应用       | Web 应用共享 jar，不是 Tomcat 核心 |

**核心区别**

1. **谁能看到**

- **CommonClassLoader**：所有类都能看到，包括 Tomcat 自身和所有 Web 应用
- **SharedClassLoader**：仅供 Web 应用使用，Tomcat 核心自己**看不到**

1. **用途不同**

- **common.loader**：
  - 加载 Tomcat 核心依赖，比如 `catalina.jar`、`servlet-api.jar`
  - 加载容器级工具类，比如 JDBC driver、日志工具
- **shared.loader**：
  - 加载多个 Web 应用之间共享的 jar
  - 避免每个 WebApp 自己放同一个 jar，节省内存

**1.1.2 context.xml**

`context.xml` 用来配置“某一个 Web 应用（Context）”的运行环境，而不是整个 Tomcat。

它主要用来配置：

- 数据源（JNDI）
- 会话管理（Session）
- 资源（JNDI Resource）
- 类加载、缓存、监控等 **WebApp 级别**能力

配置在 `$CATALINA_BASE/conf/context.xml` 里的内容，会作为「所有 Web 应用的默认 Context 配置」，对每一个 Web 应用都生效。

**1.1.3 server.xml**

tomcat的主要配置文件，解析器用这个文件在启动时根据规范创建容器。

默认的server.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>
  <Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    <Engine name="Catalina" defaultHost="localhost">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
    </Engine>
  </Service>
</Server>
```

Server是顶级元素，代表一个Tomcat实例。可以包含一个或多个Service，每个Service都有自己的Engines和Connectors。

> **Server元素**
>
> - port
>
>    server在这个端口上监听一个shutdown命令。设置为-1表示禁用shutdown命令。
>
> - shutdown
>
>    连接到指定端口的TCP/IP收到这个命令字符后，将会关闭Tomcat。
>
> - Listeners元素
>
>   Server可以包含多个监听器。一个监听器监听指定事件，并对其作出响应。
>
> - **Services元素**
>
>   一个Service可以连接一个或多个Connectors到一个引擎。默认配置定义了一个名为“Catalina”的Service，连接了一个Connectors：HTTP引擎。
>
>   - name
>
>     Service的显示名称，如果采用了标准的Catalina组件，将会包含日志信息。每个Service与某个特定的Server关联的名称必须是唯一的。
>
>   - **Connectors元素**
>
>     一个Connector关联一个TCP端口，负责处理Service与客户端之间的交互。默认配置定义了一个Connectors。
>
>     - HTTP/1.1
>
>     ​        处理HTTP请求，使得Tomcat成为一个HTTP服务器。客户端可以通过Connector向服务器发送HTTP请求，接收服务器端的HTTP响应信息。
>
>             ```xml
>                <Connector port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
>             ```
>
>   - **Engine引擎元素**
>
>     引擎是容器中最高级别的部分。可以包含一个或多个Host。Tomcat服务器可以配置为运行在多个主机名上，包括虚拟主机。
>
>     ```xml
>     <Engine name="Catalina" defaultHost="localhost">
>     ```
>
>     Catalina引擎从HTTP connector接收HTTP请求，并根据请求头部信息中主机名或IP地址重定向到正确的主机上。
>
>     - defaultHost
>
>       默认主机名，定义了处理指向该服务器的请求所在主机的名称，但名称不是在这个文件中配置。
>
>     - name
>
>       Engine的逻辑名称，用在日志和错误信息中。当在相同的Server中使用多个Service元素时，每个Engine必须制定一个唯一的名称。
>
>     - **Host**
>
>       一个Host定义了在Engine下的一个虚拟机，反过来其又支持多个Context（web应用）。
>
>       ```xml
>       <Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true">
>       ```
>
>       - name
>
>         虚拟主机名（域名）,用来匹配请求里的 `Host` 请求头,浏览器访问时：`http://localhost:8080`命中这个host
>
>       - appBase
>
>         完整路径是$CATALINA_BASE/webapps，这个目录下:webapps/
>          ├─ ROOT/
>          ├─ docs/
>          ├─ manager/
>          └─ examples/
>
>          **每一个目录 / war = 一个 Context（Web 应用）**
>
>       - unpackWARs
>
>          是否自动解压 WAR 包:`true`（默认）xxx.war` → 解压成 `xxx/,解压的好处：启动快、方便热更新 JSP / 静态资源
>
>       - autoDeploy
>
>          是否自动部署 Web 应用，Tomcat 会监控 `appBase` 目录，自动部署新放进去的 war / 目录，自动卸载被删除的应用。
>
>       - **Value**
>
>         Value（阀门）作为请求的前置处理程序，可以在请求发送到应用之前拦截HTTP请求。可以定义在任何容器中，比如Engine、Host、Context和Cluster。默认配置中，AccessLogValue会拦截HTTP请求，并在日志文件中创建一个切入点，Valve ≈ Tomcat 内部拦截器，
>
>         Filter ≈ Servlet 规范的拦截器，本例配置是放在Host下面，意味着Value作用范围是Host 下的所有 Web 应用
>
>         ```xml
>         <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
>                        prefix="localhost_access_log" suffix=".txt"
>                        pattern="%h %l %u %t &quot;%r&quot; %s %b" />
>         ```
>
>         这个配置是Tomcat 内置访问日志 Valve，记录 **每一次 HTTP 请求**，directory="logs"，表示日志存放目录（相对于 `$CATALINA_BASE`），prefix / suffix，表示日志文件名规则，例如:
>
>         ```text
>         localhost_access_log2025-12-16.txt
>         ```
>
>         pattern代表了日志格式，日志格式通常是这样的:
>
>         ```
>         127.0.0.1 - - [16/Dec/2025:10:20:01 +0800] "GET /order/api/list HTTP/1.1" 200 512
>         ```

**1.1.4 web.xml**

`web.xml` 配的是「Web 应用里的 Servlet 组件」,例如:Servlet、Filter、Filter、URL 映射、Session 配置、Welcome file、错误页等。

```xml
<web-app>
    <servlet>
        <servlet-name>hello</servlet-name>
        <servlet-class>com.demo.HelloServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>hello</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>
</web-app>
```

下面介绍一下这里面的主要标签的作用:

- **<context-param>**

  <context-param>元素含有一对参数名和参数值，用作应用的servlet上下文初始化参数。参数名在整个web应用中必须是唯一的,例如:

  ```xml
  <context-param>
  	<param-name>name</param-name>
    	<param-value>haha</param-value>
  </context-param>
  ```

- **filter**

  filter元素用于指定web容器中的过滤器,在请求和响应对象被servlet处理之前或之后,里面有`<filter-mapping>`,它是是 Servlet 规范中定义 **Filter 生效范围和触发条件** 的关键元素， 用来告诉容器：Filter 对哪些请求 URL 或 Dispatcher 类型生效。

  ```
  <filter>
      <filter-name>authFilter</filter-name>
      <filter-class>com.demo.AuthFilter</filter-class>
  </filter>
  
  <filter-mapping>
      <filter-name>authFilter</filter-name>
      <url-pattern>/*</url-pattern>
  </filter-mapping>
  ```

  其中`<filter-name>`必须和 `<filter>` 中的 `<filter-name>` 对应,容器通过名字找到 Filter 类。

  `<url-pattern>`定义 **URL 匹配规则**,常用形式：

  - `/*` → 匹配所有请求
  - /api/*  → 匹配 `/api/xxx`
  - *.jsp → 匹配 `/api/xxx`

- **servlet**

  **`<servlet>` 元素在 `web.xml` 中定义一个 Servlet 类及其初始化参数和映射关系，<servlet-mapping>决定 **URL 到 Servlet 的映射**。

```
<servlet>
    <servlet-name>HelloServlet</servlet-name>
    <servlet-class>com.demo.HelloServlet</servlet-class>
    <init-param>
        <param-name>greeting</param-name>
        <param-value>Hello</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
```

下面我画一个表格来说明一下servlet中比较重要的标签:

|     子元素      |                    说明                     |            示例            |
| :-------------: | :-----------------------------------------: | :------------------------: |
|  servlet-name   | Servlet 名字，用于 `<servlet-mapping>` 匹配 |        HelloServlet        |
|  servlet-class  |             Servlet 类全限定名              |   com.demo.HelloServlet    |
|    jsp-file     |          指定 JSP 文件路径（可选）          |         /hello.jsp         |
|   init-param    |          初始化参数（Servlet 级）           |                            |
| load-on-startup |   控制 Servlet 是否随 Web 应用启动初始化    | 0 或正整数，越小越先初始化 |

- listener

  listener元素用来注册一个监听器类，可以在web应用中包含该类。使用listener元素，可以收到事件什么时候发生以及用什么作为响应的通知。

  listener元素用来定义Listener接口，它的主要子元素为<listener-class>

  ```
  <listener>
  	<listener-class>com.gnd.web.listener.TestListener</listener-class>
  </listener>
  ```

  