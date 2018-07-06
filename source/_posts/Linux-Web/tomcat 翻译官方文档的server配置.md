---
title: tomcat 翻译官方文档
date: 2018-06-24 11:59:58
tags:  tomcat,web 
categories: [Linux-Web]
comments: true
copyright: true
---

## tomcat 翻译官方文档的server配置



官方文档:[tomcat9](http://tomcat.apache.org/tomcat-9.0-doc/index.html)

> note:此文档翻译自官方的tomcat9版本的文档,可能和其他版本部分配置有区别

<!--more-->

### 各组件官方解释:

- **Server** :

​     <Server> is the root element of the entire configuration file

​            **Server**组件是整个配置文件的根元素.

- **Service**:

​         <Service> represents a group of Connectors that is associated with an Engine.

​           **Service**组件代表关联一个Engine的一组Connectors(调度器,接口)

- **Connectors** :

​           **Connectors** - Represent the interface between external clients sending requests to (and receiving responses from) a particular Service.

​           **Connectors**代表外部客户端发送请求至某个服务(或者从某个服务接收响应)的接口

- **Containers**:

​           Containers - Represent components whose function is to process incoming requests, and create the corresponding responses. An Engine handles all requests for a Service, a Host handles all requests for a particular virtual host, and a Context handles all requests for a specific web application.

​           **Containers**----代表处理入站请求,创建相关响应等这些功能的组件.一个Engine为Service处理所有请求,一个Host为某个虚拟主机处理所有请求,一个Context为某个WEB应用处理所有请求      

 所以,一个Containers容器应该包含Engine,Host,Context组件.而这些组件都是用来处理和响应请求的.只是处理和响应的对象范围不同Nested Components:

-  **Nested** **Components** :

​           Nested Components --- Represent elements that can be nested inside the element for a Container. Some elements can be nested inside any Container, while others can only be nested inside a Context.

​            **Nested** **Components**-代表一些可以nest(嵌套)进Containers下各组件的元素.有些元素可以嵌套进所有Container下的组件,而有些元素只能嵌套进Context组件.

**Nested** **Components**组件包含有:Valve,Cluster,Realm,Manager,Resources,Loader,Listener,等.下面再详细解释

 

------

### 详细组件介绍

#### Server:

A Server element represents the entire Catalina servlet container. Therefore, it must be the single outermost element in the conf/server.xml configuration file

 **Server**元素代表整个Catalina servlet容器,只能有一个Server,且必须定义在server.xml配置文件的最外层

**Server公共属性**:

- **className**: Java使用的类名,该类必须实现 org.apache.catalina.Server 接口,如果类名没有指定,默认使用标准工具实现(the standard implementation will be used.)
- **address**:  服务等待shutdown命令的TCP/IP地址,如果没有指定,默认使用Localhost
- **port**:          服务等待shutdown命令的TCP/IP端口,
- **shutdown**: 关闭Tomcat

- **Service**:

A Service element represents the combination of one or more Connector components that share a single [Engine](http://tomcat.apache.org/tomcat-9.0-doc/config/engine.html) component for processing incoming requests. One or more Service elements may be nested inside a [Server](http://tomcat.apache.org/tomcat-9.0-doc/config/server.html) element.

可能一个或者多个Service元素嵌套在Server元素中

**Serveice公共属性:**

- **ClassName**:Java使用的类名,该类必须实现 org.apache.catalina.Server 接口,如果类名没有指定,默认使用标准工具实现(the standard implementation will be used.)
- **name**:          Service显示名,如果使用标准Catalina组件,这个名字会显示在日志消息中.每个Service名字必须唯一

- **Executor**:

The Executor represents a thread pool that can be shared between components in Tomcat. Historically there has been a thread pool per connector created but this allows you to share a thread pool, between (primarily) connector but also other components when those get configured to support executors

**Executor** 代表Tomcat各组件共享的线程池,虽然每个Connector都创造了一个线程池,但是允许在各Connector之间共享一个线程池,另外其他组件也支持Executors

**Executor公共属性:**

- **ClassName**: The default value for the className is org.apache.catalina.core.StandardThreadExecutor
- **Name**: The name used to reference this pool in other places in server.xml. The name is required and must be unique. Name的值必须要指定,且唯一

**部分重要属性值:**

- **daemon**: (布尔值),是否为后台线程.默认为True
- **namePrefix**: 线程名前缀.(字符串),每个线程名字的前缀,用来标记每个线程名字.这样的话每个线程的名字就为:线程前缀(namePrefix)+线程号(threadNumber).比如:catalina-exec-1
- **maxThreads**:最大线程数 (整数值),线程池中最大活跃线程数.默认为200.
- **minSpareThreads**: 最小空闲线程.(整数值)最少永远保持活跃的线程数量,默认为25.----无论是否有用户请求都保持活跃
- **maxIdleTime**: 最大空闲时间.(整数值),一个空闲线程关闭之前的延时时间.除非当前活跃线程小于或者等于最小空闲线程,单位是毫秒.默认指为600000(1分钟)
- **prestartminSpareThreads**:最小空闲线程预启动.(布尔值).当启动Executor时,最小空闲线程(minsparethread)是否一起启动.默认为fals

- **Connector**

The HTTP Connector element represents a Connector component that supports the HTTP/1.1 protocol. It enables Catalina to function as a stand-alone web server, in addition to its ability to execute servlets and JSP pages. A particular instance of this component listens for connections on a specific TCP port number on the server. One or more such Connectors can be configured as part of a single [Service](http://tomcat.apache.org/tomcat-9.0-doc/config/service.html), each forwarding to the associated [Engine](http://tomcat.apache.org/tomcat-9.0-doc/config/engine.html) to perform request processing and create the response.

**HTTP Connector**元素表示Connector组件支持HTTP/1.1协议,它使得Catalina像一个独立web服务器那样运行.另外还可以运行servlets和JSP页面..这个组件的一个特定的实例侦听服务器的一个特定端口.一个或者多个Connectors可以配置为一个单个Service的一部分.每次转发到相关引擎都由Connector执行请求处理和创建响应.

Each incoming request requires a thread for the duration of that request. If more simultaneous requests are received than can be handled by the currently available request processing threads, additional threads will be created up to the configured maximum (the value of the maxThreads attribute). If still more simultaneous requests are received, they are stacked up inside the server socket created by the Connector, up to the configured maximum (the value of the acceptCount attribute). Any further simultaneous requests will receive "connection refused" errors, until resources are available to process them.

如果接收到多个同时并发的入站请求,且可以被当前活跃的线程处理的话,那么每个入站请求都需要一个线程去处理这个持续的请求.另外,线程会被持续创建直到到达最大线程数(maxThreads属性).如果还有更多并发请求接收到的话,他们会堆积在Connector创建的server socket内,直到达到最大队列数(acceptCount属性).此时更多的后续同时并发请求会接收到"connection refused"错误,直到有资源去处理他们.

**常用公共属性:**

- **allowTrace**:   (布尔值)是否开启TRACE HTTP方法,如果没有指定,默认为false
- **enableLookups**:   (布尔值)如果为true则调用request.getRemoteHost()方法去执行DNS lookups,返回远程用户的真实主机名,如果设置为false,则跳过DNS查询,直接以字符串格式返回IP地址,(因此可以提高性能)
- **maxHeaderCount** (数值),请求的最大首部数,如果请求的首部数超过指定的值,则拒绝.如果设置为负值,则表示没有限制,如果没有指定,则默认是100
- **port**:    (数值)Connectory用来创建socket,且侦听入站连接请求的TCP端口
- **protocol**: 协议. 处理入站流量的协议.默认是http/1.1. http/1.1使用一种自动切换机制去选择java NIO connector或者APR/native connector.如果你想使用一个明确的协议来代替前面提到的自动切换机制的话.可以使用下面协议:

 org.apache.coyote.http11.Http11NioProtocol - non blocking Java NIO connector  #非阻塞JAVA NIO connector

org.apache.coyote.http11.Http11Nio2Protocol - non blocking Java NIO2 connector #非阻塞JAVA NIO2 connector

org.apache.coyote.http11.Http11AprProtocol - the APR/native connector.   #APR connector

 其他自定义的协议实现方法或许也可以使用.

下图显示了这几种connectors的区别

{% qnimg web/tomcat.png %}

- **proxyName**: 代理名,如果这个connector配置为代理,配置服务名用来返回调用request.getServerName()
- **proxyPort**:    如果connector被配置为代理,则指定一个端口返回调用request.getServerName()
- **redirectPort**: 如果这个connector支持non-ssl请求,并且接收到一个匹配<security-constraint>要求SSL传输的请求,Catalina会自动重定向这个SSL的请求到这个定义的端口
- **scheme**:  设置这个属性为你想要返回调用request.isSecure()的协议名称.例如:如果你使用一个SSL connector.则设置此属性为"https".默认值为"http"
- **secure**:   如果你希望调用request.isSecure()来为connector接收到的请求返回true的话,那么设置这个属性为true,默认为false
- **URIEncoding**: 这个值定义了用来解码URI的字符编码.如果没有指定的话,默认会使用UTF-8.除非org.apache.catalina.STRICT_SERVLET_COMPLIANCE  [system property](http://tomcat.apache.org/tomcat-9.0-doc/config/systemprops.html) 设置为true,在这种情况下会使用ISO-8859-1
- **useIPVHosts**: 如果设置为true,那么tomcat会使用接收到请求的IP地址来确定使用让这个IP的主机发送请求.默认值为false...(这个属性我有点搞不懂)

**标准属性:**

标准HTTP connectors(包括NIO,NIO2以及ARP/native)都支持除了上面提到的普通属性wait,还支持以下标准属性.

- **acceptCount**: 当所有可用的处理请求线程都在使用时,所允许的最大入站请求队列长度.当该队列已满时,任何接收到的请求将被拒绝.默认值是100.(优化性能参数)
- **acceptorThreadCount**: 用来接收连接的线程数量,虽然你从未真的想过需要大于2个线程,但是在有多个物理CPU的服务器上增加这个属性值.另外,对于很多非持久连接,你可能也想增加这个值,默认值是1
- **address**: 对于拥有多个IP地址的服务器来说,这个属性指定了哪个地址被用来侦听某个tomcat端口,默认情况下,connector会侦听所有本地地址,如果配置为0.0.0.0或者::,那么会侦听IPV4和IPV6地址.
- **allowHostHeaderMismatch**:默认情况下,Tomcat会拒绝那些请求行中指定的主机和请求首部指定的主机不匹配的请求.如果设置为false,则不检查是否匹配,默认为true
- **bindOnInit**: 控制何时绑定一个connector和一个socket.默认情况下,当connector初始化的时候绑定socket,当connector被销毁时解绑这个socket.如果设置为false.则当connector启动的时候才绑定一个socket.而关闭的时候解绑这个socket
- **clientCertProvider**:如果一个客户端证书信息不是以java.security.cert.x509Certficate实例的形式提供,那么在使用前需要进行转义.这个属性定义了哪个JSSE提供者被用来执行证书转义,如果没有指定,则使用默认提供者
- **compressibleMimeType**: 这个值是一个以逗号分隔的MIME类型的列表,用来定义可以被HTTP压缩的类型.默认值是text/html,text/xml,text/plain,text/css,text/javascript,application/javascript,application/json,application/xml
- **compression**: Connector可以使用HTTP/1.1的GZIP压缩试图节约服务器的带宽.有如下值可以使用.如果这个属性没有指定,默认是off:

**off**:关闭压缩功能

**on**:开启压缩功能,会压缩上面提到的MIME类型

**force**:任何情况下都强制压缩

整数值:相当于on功能,但是在压缩前指定了一个最小数据字节值,如果内容长度未知,或者压缩功能被设置为on或者更激进,那么输出也会同样被压缩,

> Note:如果Connector支持sendfile特性(比如NIO connector),那么在使用压缩功能(节约带宽)和使用sendfile特性(节约CPU时钟周期)这两种功能之间需要权衡考虑.如果两个功能同时使用的话,sendfile会比compression优先级高,这样一来大于48kb的静态文件将不会被压缩.你可以在useSendfile属性中关掉sendfile功能,或者在conf/web.xml,或网站应用配置文件web.xml等配置文件的DefaultServlet配置中修改sendfile的使用门槛

- **compressionMinSize**:如果compression功能被设置为on.那么这个属性可以指定可被压缩的最小数据字节,如果没有指定,默认值是2048
- **connectionLinger**: 当connector被关闭时,sockets被关闭的延迟时间.单位为秒.默认值为-1,表示关闭socket延迟时间
- **connectionTimeout**: 当接收到一个请求URI行,在接收到连接后,Connector会等待的时间数,单位为毫秒.如果设置为-1,则表示永不超时.默认值为60000(60秒),但是注意:Tomcat的标准server.xml设置为20000(20秒).除非disableUploadTimeout属性值被设置为false,否则当读取用户的请求主体时,也会同样使用这个属性值设置的超时时间.
- **connectionUploadTimeout**:当一个数据上传中时,指定一个超时时间,单位毫秒.这个属性只有当disableUploadTimeout被设置为false时才会生效
- **disableUploadTimeout**:这个属性允许servlet容器在数据上传期间使用不同的,通常是特别长的连接超时时间.如果没有指定,默认为true,也就是关闭超时时间
- **executor**: 参考之前提到的Executor元素解释.如果这个属性被设置,而且指定了一个已经存在的executor名称,那么这个connector会使用executor中的线程池,而且其他connector中指定的有关线程的属性值会被忽略.

> Note:如果connector没有指定一个共享的executor名字,那么这个connector会使用一个私有的,内部executor去提供线程池

- **executorTerminationTimeoutMillis**:在Connector关闭的过程中,内部私有的executor线程池会等待多久时间后终止处理请求的线程.单位为毫秒,如果没有指定,则默认是5000(5秒)
- **KeepAliveTimeout**:在关闭一个连接前,Connector会等待多久去接收另外一个HTTP请求,单位为毫秒,默认情况下会使用connectionTimeout属性中设置的值.如果设置为-1,则表示永不超时
- **maxConnections**:服务器将会接收的最大连接数.当当前请求连接达到这个数指时,服务器会接收,但是不会处理连接.而对于后续其他连接,会被阻塞直到连接数被处理完毕,低于maxConnections的值,届时,服务器会再次开始接收和处理新的连接.

> Note:一旦这个临界值被触发,由于acceptCount的设置操作系统可能会仍然继续接收新的连接(这里,我理解是acceptCount的值大于maxConnections的值)

对于不同的connector类型,maxConenctions的默认值也不一样.比如对于NIO和NIO2,默认值是10000.而对于APR/native,默认值是8192

如果这个属性值设置为-1,那么会关闭maxConnections特性,而且对于连接将不会计数

- **maxCookieCount**: 一次请求可被允许的最大cookies数量.此值小于0则表示没有限制.如果没有指定,则默认值为200
- **maxHttpHeaderSize**: 最大请求HTTP首部和响应HTTP首部字节长度.如果没有指定,则默认值为8192(单位是bytes).也就是8KB
- **maxKeepAliveRequests**:可被流水线式处理的最多数量的HTTP请求直到服务器关闭这个连接.设置为1会关闭HTTP/1.0和HTTP/1.1的keep-alive.设置为-1会允许无限制的流水线式的,或者长连接的HTTP请求数.如果没有指定,这个属性值为100
- **maxSwallowSize:Tomcat**会因上传终止而忍受的最大请求主体字节数(不包括传输过程中编码开销).当Tomcat发现请求主体将被忽略时,而客户端仍然继续发送的话,上传就会终止.如果Tomcat不吞下请求主体的话,客户端很可能看不到tomcat响应.如果没有指定,默认是2097152(2M).如果该值小于0,则表示无限制
- **maxThreads:Connector**创建的最大处理请求的线程数,这个值确定了可以最大并发处理的请求数.如果没有指定.默认值为200.如果connector指定了一个executor,那么会使用excutor线程池,这个属性会被忽略,(优化性能)
- **minSpareThreads**:始终处于运行状态的最小线程数,默认是10.如果connector指定了一个executor,那么会使用excutor线程池,这个属性会被忽略,(优化性能)
- **processorCache**:协议处理器缓存处理器对象提高性能.这个设置指定了有多少对象可以被缓存.-1表示无限制.默认是200.如果没有使用Servlet3.0异步处理那么建议使用和maxThreads一样的属性值.否则建议设置比maxThreads更大的值.或者最大的可接受的并发请求值(同步或者异步)(优化性能)
- **rejectIllegalHeaderName**:如果接收到一个包含非法首部名的HTTP请求(比如,首部名不是一个token),如果设置为true,则会发送一个400响应去拒绝这次请求,如果设置为false则会忽略非法首部.默认值是true.
- **server**:重写HTTP响应的服务器首部,如果设置了此属性,这个属性的值会重写web程序设置的服务器首部.如果没有定义,将会使用web程序定义的服务器首部.如果web程序没有指定一个服务器首部,那么就不设置任何服务器首部
- **serverRemoveAppProvidedValues**:如果为true,删除web程序设置的所有服务器首部信息.如果上面的server属性已经设置过了.这个属性会被忽略.如果上面的server属性没有定义,那么默认为false
- **SSLEnabled**:使用这个属性开启SSL流量.设置为true会在Connector上开启SSL握手/加密/解密.默认为false.

> note:如果设置为true,你可能还要设置scheme属性和secure属性传递正确的request.getScheme()和request.isSecure()值到servlets.相信信息可以参考[SSL Support](http://tomcat.apache.org/tomcat-9.0-doc/config/http.html#SSL_Support)文档

**tcpNoDelay**:如果设置为true,TCP_NO_DELAY选项会被设置到服务器的socket套接字,在大多数情况下会提高性能.默认为true.

**threadPriority:JVM**内部的处理用户请求的线程优先级.默认是5.如果connector指定了一个executor,那么会使用excutor线程池,这个属性会被忽略,(See the JavaDoc for the java.lang.Thread class for more details on what this priority means)

**throwOnFailure**:如果这个Connector在生命周期内遇到异常,这个异常是应该再次抛出,还是被记录?如果没有设置,默认为false

------

### Context

Context元素表示一个虚拟主机的web程序.每个WEB程序都是基于Web Application Archive(WAR)文件.或者包含相关未打包内容的相应目录.web程序用来处理Catalina挑选的和基于每个Context定义的context path匹配的最长请求URI后缀的HTTP请求.(英文有点绕口.就是处理匹配context中定义的context path匹配的URI后缀的HTTP请求).Context会根据web程序调度定义的servlet mappings选择一个合适的servlet去处理入站请求.

你可以多个Context元素,在一个虚拟主机里每个Context元素必须有一个唯一的context名字,但是如果你使用并行部署(parallel deployment)时,context path可以不用唯一.(具体参考官网相关信息-------->[parallel deployment](http://tomcat.apache.org/tomcat-9.0-doc/config/context.html))

另外,一个Context必须存在一个等于零长度字符串的context path.这个Context会成为该虚拟主机的默认web程序,并且被用来处理那些不匹配其他Context的context path的所有请求.

**如何定义Context:**

不建议吧<Context>元素直接放在server.xml文件内.这是因为如果修改context配置文件会变的有风险,因为server.xml配置文件不重启Tomcat服务的情况下不会加载生效.

每个Context文件可能在如下文件中被定义

1. 在web应用程序下的/META-INF/context.xml中定义.或者复制context.xml配置文件到conf/Catalina/[hostname]/目录下重命名为[web应用程序名字].xml
2. conf/Catalina/[hostname]/目录下.context path和版本会从不带xml后缀的文件名中派生出来.该文件会优先于任何web程序下META-INF目录下打包的context.xml文件
3. 在server.xml配置内的Host元素内

默认的Context元素可能被定义到多个web程序.对单个web程序的配置会重写这些默认配置.

- conf/context.xml文件内定义的Context元素会被所有web程序加载
- conf/Catalina/[hostname]/context.xml文件内定义的Context元素会被这台主机上的所有web程序加载

>  除了server.xml配置文件以外,只能定义一个Context元素

**部分重要公共属性:**

- **crossContext**:如果想在这个web程序内调用ServletContext.getContext()去为运行在这个虚拟主机的其他web程序成功的返回调度请求,那么设置为true.在安全场景下默认为false.使getContext()永远返回Null
- **docBase**: 此web程序的根目录(document base 或者 context root).或者该web程序的WAR包文件路径.你可以指定一个绝对路径或者相对于此context所属Host的appBase目录的相对路径

除非在server.xml文件中定义了Context元素或者docBase没有位于Host的appBase下.否则不能设置这个属性

- **override**:设置为true会忽略任何全局或者Host的默认context.默认情况下,会使用default context.但是可以被此Context明确定义的属性设置重写
- **path**:此web程序的context path.catalina将每个URL的起始和context path进行比较，选择合适的web应用处理该请求。特定Host下的context path必须是惟一的。如果context path为空字符串（""），这个context是所属Host的缺省web应用,用来处理不能匹配任何context path的请求。
- **reloadable**:如果你希望Catalina监控/WEB-INF/classes/和/WEB-INF/lib/下的所有类的变化,并且如果检测到发生了变化就自动重载web程序,那么设置为true.在web程序部署的时候,这个特性非常有用.但是它会带来非常大的运行开销,所以不建议用在已经部署好的生产环境.默认值是false.如果需要的话,你可以使用Manager web application去触发web程序重载.(设置为false,提升性能)

**Context的嵌套组件**:

**Cookie Processor**----------配置HTTP cookie首部的解析和生成

**Manager**---------------------配置session管理.可用来创建,销毁,持久化HTTP sessions.通常session管理的默认配置就已经足够了

**Realm**------------------------配置此特定web程序锁允许的用户数据库和相关规则,如果没有指定,此web程序会使用所属Host或者Engine所关联的Realm

**Resources**-------------------配置此特定web程序用来访问静态资源的资源管理器.通常默认配置就已经足够了

------

### Engine容器:

The Engine element represents the entire request processing machinery associated with a particular Catalina [Service](http://tomcat.apache.org/tomcat-9.0-doc/config/service.html). It receives and processes all requests from one or more Connectors, and returns the completed response to the Connector for ultimate transmission back to the client.

Exactly one Engine element MUST be nested inside a [Service](http://tomcat.apache.org/tomcat-9.0-doc/config/service.html) element, following all of the corresponding Connector elements associated with this Service.

**Engine**元素表示此特定的Catalina Service关联的请求处理机制.它接收和处理所有从一个或者多个Connoctor进来的请求.且返回完整响应给Connector,最终传输回到用户.

一个具体的Engine元素必须嵌套进Service元素.且携带所有相关Connector元素关联到这个Service

部分重要公共属性:

- **defaultHost**:默认主机名,标识用来处理所有指向此服务器主机名的请求的主机,但是此主机又没有在此配置文件中配置.这个主机名必须匹配其中一个前套内的HOST元素的name属性值
- **name**:此Engine的逻辑名称,被用于log日志和错误日志.当在同一个Server中存在多个Service元素时,每个Engine名称必须指定唯一名称
- **jvmRoute**:被用来在load balancing场景中启用session亲和力(粘性)特性时指定的标识符.在一个Tomcat集群内的所有Tomcat服务器内这个属性值必须唯一.此标识符会被追加到生成的session标识符.因此允许前端代理服务器永远转发某个session到同一个Tomcat实例

------

### Host容器:

**Host**元素表示一个虚拟主机,它关联到一个tomcat运行的服务器的网络域名(比如:www.abc.com).如果为了用户能够使用网络域名连接Tomcat服务器,那么这个域名必须在DNS服务中已注册.

很多时候系统管理员想关联多个网络域名到同一个虚拟主机上.使用Host Name Aliases(虚拟主机别名)特性可以实现

一个或多个Host元素被嵌套进Engine元素中.在Host元素内,可以嵌套Context元素关联一个web程序到此虚拟主机.

Host元素的name属性值必须和它关联的Engine元素的defaultHost属性值相同

用户端通常使用主机名去标识一个他们连接的服务器,这个主机名也被包含进HTTP请求首部.Tomcat从HTTP首部提取主机名且寻找想匹配的主机名.如果没有找到匹配的主机名,则defaulthost默认主机提供请求服务.默认主机名不需要是一个DNS域名(虽然可以是)

Host部分重要公共属性:

- **appBase**: 该虚拟主机的web程序基目录.部署到该虚拟主机的web程序的目录路径.你可以指定一个绝对路径,或者$CATALINA_BASE的相对路径.
- **xmlBase**:虚拟主机的XML基名.部署到该虚拟主机的context XML描述符的目录路径.你可以指定一个绝对路径,或者$CATALINA_BASE的相对路径.如果没有指定,默认为 conf/<engine_name>/<host_name>
- **createDirs**:如果为true,则Tomcat会在启动阶段,自动创建上面2个属性的目录名.默认为true.如果设置为true,且目录创建失败.会出现一个错误消息,但是不会阻止tomcat启动
- **autoDeploy**:这个属性的值指示当Tomcat运行时是否应该定期的去检查是否有新部署的web程序,或者web程序是否有更新.如果设置为true.Tomcat会定期检查appBase和xmlBase目录,如果发现新的web程序和context XML描述符,Tomcat会自动部署.更新一个web程序或者context XML描述符会自动触发web程序重载.默认为true.(设置为false,提升性能)
- **deployIgnore**:这是一个正则表达式,当autoDeploy和deployOnStartup属性值被设置时,此正则表达式定义哪些哪些路径被忽略.这可以让你的配置文件在一个版本控制系统内,比如不部署appBase目录内的.svn或者cvs文件夹.此正则表达式相对于appBase目录
- **deployOnStartup**:此属性指示当tomcat启动时,web程序是否自动部署.默认为true
- **name**:通常是该虚拟主机的DNS域名.不分大小写.此值必须和Engine元素的defaultHost值相同.
- **unpackWARs**:如果你想将appBase目录下的web程序的WAR包文件解压成相应的磁盘目录结构,那么设置为true.如果为false则web程序直接运行WAR包.(设置为true,提升性能)

------

### Realm嵌套元素

**Realm**元素表示一个用户名,密码的"数据库",以及指定到那些用户的roles(有点像linux的groups概念).Realm的不同实现允许Catalina集成到已经创建,维护的认证信息环境.且利用这些信息实现容器安全管理

一个Catalina容器(Engine,Host,Context)都可能嵌套多个Realm元素(甚至Realm自身也可能包含多个Realm嵌套元素).Engine或者Host内嵌套的Realm元素会被其他级别更低的容器组件继承.除非其他低级别容器下明确指定了嵌套的Realm元素,如果Engine没有配置任何Realm元素,那么一个Null Realm实例会被Engine自动创建

公共属性:

**className**: java实现的类名.此类必须实现org.apache.catalina.Realm接口

JDBC Database Realm - org.apache.catalina.realm.JDBCRealm

JDBC Database Realm通过一个JDBC驱动连接Tomcat到一个真实的数据库.用于执行用户名,密码以及关联角色的查找,因为每次需要时都会进行查找,因此任何对数据库的修改都会立即反映在验证用户新登录信息中

有丰富的属性可以让你配置和数据库的连接,以及检索信息的表和列名:

- **connectionName**: JDBC连接的数据库名
- **connectionPassword**:JDBC连接的数据库密码
- **connectionURL**:建立一个数据库连接的连接URL
- **driverName**:用来来接到身份验证数据库的JDBC驱动完全合格java类名
- **roleNameCol**:"user roles"表的列名."user roles"表包含分配给相应用户的角色名
- **userCredCol**:"Users"表的列名,"Users"表包含用户的证书信息(例如密码).如果CredentialHandler属性有定义的话,这个密码会以指定的算法进行编码.否则,密码会是明文
- **userNameCol**:"Users"和"user roles"表的列名.此表包含用户名
- **userRoleTable**:"user roles"表名.必须包含userNameCol和roleNameCol属性定义的列名.对于大多数配置来说这个属性是必须的
- **userTable**:"users"表的名字,必须包含userNameCol和userCredCol属性

DataSource Database Realm - org.apache.catalina.realm.DataSourceRealm ---------------此组件和上面提到的JDBC相似,属性也几乎相同.不再赘述

UserDatabase Realm - org.apache.catalina.realm.UserDatabaseRealm

UserDatabase Realm是基于UserDatabase资源的一种Realm实现方法.UserDatabase是此Tomcat实例上配置的全局JDNI资源提供的.

UserDatabase Realm实现支持下列额外属性:

resoureceName:此realm使用的全局UserDatabase资源名称.

其他reaml机制不再一一赘述,详情查阅官方文档

------

### Resources组件

Resources元素表示此web程序的所有可用资源.resources资源包含类名,JAR文件,HTML,JSPs和此web程序的其他文件.提供目录,JAR文件,WAR作为这些resources的资源.并且resources实现还可以扩展提供文件存储的支持,例如文件存储进数据库或者版本库.默认情况下Resources会被缓存

Resources元素可能被嵌套在Context组件内,如果没有,则一个默认的基于文件系统的Resources会被自动创建.默认的Resources已经可以满足绝大多数需求