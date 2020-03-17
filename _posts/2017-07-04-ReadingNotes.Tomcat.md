---
layout: post
title: Tomcat8
category: SearchingNotes
description: Tomcat8 source SearchingNotes.
---

# Tomcat8

# 1. ClassLoader

`Bootstrap`创建一些列class loads，并创建和启动Catalina实例。
> tomcat为什么要自定义classLoader？
> 1. 实现servlet规范中对类加载的要求；
> 2. 实现不同webapp的类隔离）

`org.apache.catalina.startup.Bootstrap`启动类，main方法启动tomcat，首先调用init方法处理
化类加载器。这里先启动类加载器`initClassLoaders`。先创建`common classLoader`，再创建
`server classLoader`和`shared classLoader`。`commonLoader`为`serverLoader`和`sharedLoader`的父类。通过`ClassLoaderFactory.createClassLoader`
创建classLoader，`UrlClassLoader`。（Tomcat为什么需要使用`AccessController.doPrivileged`
来创建`UrlClassLoader`？）

类加载器创建完成后，将`serverLoader`设置为当前线程的的上下文类加载器。
`serverLoader`加载器通过反射方式创建`org.apache.catalina.startup.Catalina`启动类。再通过反射的方式将`Catalina`启动类的父加载容器设置为`java.lang.ClassLoader`。
（所以在启动的时候，做了什么？ClassLoader之间的关系是怎么样的？

类加载器关系图
classloader->commonLoader->serverLoader
					     ->sharedLoader
		   ->catalinaLoader

）

初始化完成后，`Bootstrap`调用`Catalina`启动类的`start`方法完成启动工作。到这里
`Bootstrap.main`方法到此结束。之后的工作都交给了`Catalina.start`方法。

`Catalina`定义了一个内部类`CatalinaShutdownHook`监听关闭事件，当服务被kill时，将调用
`Catalina.this.stop()`方法安全停止服务。现在来看`Catalina.start`方法做了什么？

首先创建Digester，启动`org.apache.catalina.core.StandardServer`服务，加载`conf/server.xml`配置文件，解析配置。

StandardServer 类图

![](/assets/images/how-tomcat-works/StandardServer.png)

Server->Services
      ->Services
	

## 1.1 Catalina

Catalina的服务默认是StandardServer。在createStartDigester方法解析conf/server.xml 配置文件，并配置了默认的服务实现类`org.apache.catalina.core.StandardServer`

启动服务

## 1.2 Server

StandardServer可以管理多个Service，使用数组来存储Service。

addService方法里面每次有新的service，创建新的Service数组，然后使用System.arraycopy将旧的Service数组复制到新的数组中，并将新添加的Service添加到新的数组。添加成功后启动该新的Service。

removeService方法将service从数组中删除，并停止该Service。

initInternal方法处理化所有的Service

startInternal方法启动所有的Service

stopInternal方法停止所有的Service

destroyInternal方法销毁所有的Service


# 2. Connector

## 2.1 Connector

Connector 类图

![](/assets/images/how-tomcat-works/Connector.png)

（一句话概括Connector的功能--2018/03/02）

在构造方法中设置连接协议，（tomcat8配置文件server.xml默认协议是HTTP/1.1）会判断是否启用了Apr协议。如果启用了Apr协议，则使用"Http11AprProtocol"协议；否则使用"Http11NioProtocl"协议（默认协议）。
初始化的时候创建adapter和handler。
启动调用endpoint的start方法。

Connector创建一个Adapter（CoyoteAdapter）和ProtocolHandler（Http11NioProtocol）。Http11NioProtocol创建NioEndpoint（NioEndpoint是Connector中处理客户端连接的核心类，负责创建服务器套接字，并绑定到监听端口；同时还创建accepter线程来接收客户端的连接以及poller线程来处理连接中的读写请求）。

Protocol 类图

![](/assets/images/how-tomcat-works/Protocol.png)

Http11NioProtocol构造了一个NioEndpoint和Http11ConnectionHandler类。（这里可以不用一步步的嵌套说，直接说Protocol做了什么就可以）同时将handler也保存到endpoint。endpoint接收连接请求，并将请求传给handler处理。
Http11AprProtocol构造了一个AprEndpoint和Http11ConnectionHandler类。

`NioEndpoint<NioChannel>`和`AprEndpoint<Long>`都继承`AbstractEndpoint<S>`。

NioEndpoint使用ServerSocketChannel进行监听连接；AprEndpoint使用本地方法启动Apr来监听连接。
NioEndpoint启动时创建Acceptor线程循环监听新的连接请求，当接收到新的连接后，将新连接封装成NioChannel，然后将NioChannel注册到Poller队列上，由Poller来处理请求，Poller类实现了Runnable接口。NioEndpoint维护着一个Poller循环队列，每次将NioChannel顺序注册到Poller上。Poller的register注册方法，将NioChannel封装成PollerEvent事件，PollerEvent事件也实现了Runnable接口。
PollerEvent将事件注册到Socket上，Poller循环
监听Socket上的事件，一旦有事件注册到Socket上，就处理事件。

NioChannel 类图

![](/assets/images/how-tomcat-works/NioChannel.png)

Http11Processor 类图

![](/assets/images/how-tomcat-works/Http11Processor.png)

endpoint如何接收请求？

Endpoint 类图

![](/assets/images/how-tomcat-works/Endpoint.png)

初始化线程池，创建Acceptor，Acceptor负责监听TCP/IP的连接请求。将请求注册到Poller上。
AbstractEndpoint:
createExecutor 创建线程池，'-exec'
startAcceptorThreads 创建 Acceptor，'-Acceptor-'。后台线程用于监听TCP/IP的连接请求。

Http11ConnectionHandler处理请求。首先创建一个Processor，然后调用Processor的process方法处理请求。
Processor一行行的解析。解析完成后将生成的Request和Response交给Adapter处理。Adapter
的service方法，选择容器，调用容器的invoke方法。

InputBuffer 类图

![](/assets/images/how-tomcat-works/InputBuffer.png)

## 2.1 Service

Service关联着一组Connector。

addConnector方法采用跟addService方法类似，使用数组管理Connector，将旧的Connector数组复制到新的数组，并将新添加的Connector添加至数组中。然后启动Connector。（其实Server和Service都只是一个封装，并没有做任何事情。主要是Connector和Container两个组件，）

# 3. Container

Container 类图

![](/assets/images/how-tomcat-works/Container.png)

ContainerBase基类实现了Container接口，并持有一个管道Pipeline。管道连接着多个容器。Engine、Host、Context、Wrapper都是容器的实现类。管道会顺序的调用所有容器，管道内的容器需要实现invoke方法。

Value 类图

![](/assets/images/how-tomcat-works/Value.png)

每个容器都管理着对应的Value，容器实例化的时候创建一个新的Value添加到管道中。管道调用Value的invoke方法。

StandardWrapperValue的invoke方法，获取StandardWrapper容器，然后通过getParent方法获取Wrapper的父容器Context。检查Context和Wrapper是否可用，如果可用，Wrapper将通过allocate方法实例化一个Servlet来处理请求。

StandardWrapper的loadServlet方法完成servlet实例化工作，该方法是一个synchronized方法

{% highlight java %}
public synchronized Servlet loadServlet() throws ServletException {
	
	...

	InstanceManager instanceManager = ((StandardContext)getParent()).getInstanceManager();

	servlet = (Servlet)instanceManager.newInstace(servletClass);

	...

	return servlet;
}
{% endhighlight %}

Servlet实例化完成后，Wrapper为请求创建一个过滤链ApplicationFilterChain。过滤链会依次调用链上的所有Filter。当链上所有的Filter都处理完成后，最后会调用servlet的service方法。（Servlet是从哪里来？）

ApplicationFilterChain 类图

![](/assets/images/how-tomcat-works/ApplicationFilterChain.png)

# 4 jasper
`org.apache.jasper.servlet.JspServlet` The JSP engine.