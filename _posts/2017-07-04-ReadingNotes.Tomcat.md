---
layout: post
title: How Tomcat Works
category: ReadingNotes
description: How Tomcat Works A Guide to Developing Your Own Java Servlet Container 读书笔记
---

# How Tomcat Works

# 4. Connector

## 4.1 Connector

Connector 类图
![](/assets/images/how-tomcat-works/Connector.png)

在构造方法中设置连接协议，会判断是否启用了Apr协议。如果启用了Apr协议，则使用"Http11AprProtocol"协议；否则使用"Http11NioProtocl"协议。
初始化的时候创建adapter和handler。
启动调用endpoint的start方法。

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

## 4.2 Container