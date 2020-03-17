---
layout: post
title: Tomcat In Action
category: Tomcat
description: Tomcat6
---

## Tomcat总体结构

Tomcat的总体结构

![](/assets/images/how-tomcat-works/tomcat-arch.gif)

Tomcat的核心组件有两个：Connector和Engine（即Servlet容器）。

## Connector是如何工作的

Connector 组件是 Tomcat 中两个核心组件之一，它的主要任务是负责接收浏览器的发过来的 tcp 连接请求，创建一个 Request 和 Response 对象分别用于和请求端交换数据，然后会产生一个线程来处理这个请求并把产生的 Request 和 Response 对象传给处理这个请求的线程，处理这个请求的线程就是 Container 组件要做的事了。

Connector是多线程设计。

## Container是如何工作的

Container 是容器的父接口，所有子容器都必须实现这个接口，Container 容器的设计用的是典型的责任链的设计模式，它有四个子容器组件构成，分别是：Engine、Host、Context、Wrapper，这四个组件不是平行的，而是父子关系，Engine 包含 Host,Host 包含 Context，Context 包含 Wrapper。通常一个 Servlet class 对应一个 Wrapper。

当 Connector 接受到一个连接请求时，将请求交给 Container。每个容器都通过Pipeline连接，Container会依次执行Pipeline上的每一个节点Value。

Context通过Mapper来找到对应的Wrapper。

Wrapper 代表一个 Servlet，它负责管理一个 Servlet，包括的 Servlet 的装载、初始化、执行以及资源回收。Wrapper 是最底层的容器，它没有子容器了，所以调用它的 addChild 将会报错。

Wrapper 的实现类是 StandardWrapper，StandardWrapper 还实现了拥有一个 Servlet 初始化信息的 ServletConfig，由此看出 StandardWrapper 将直接和 Servlet 的各种信息打交道。

## Tomcat如何处理请求？

## 如何实现ServletContainer的？

## Tomcat中应用的设计模式

### 门面模式

在 Request 和 Response 对象封装中、Standard Wrapper 到 ServletConfig 封装中、ApplicationContext 到 ServletContext 封装中等都用到了这种设计模式。

### 观察者模式

Lifecycle接口

### 命令模式

Connector和Container组件之间的交互

### 责任链模式

Pipeline