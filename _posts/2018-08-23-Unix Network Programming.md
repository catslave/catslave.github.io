---
layout: post
title: Unix Network Programming
category: Unix
description: Unix Network Programming 1.
---

# IPC进程间通信（interprocess communication）

消息传递发展阶段：
* 管道
* 消息队列
* 远程过程调用

管道是没有名字的只能用于同个祖先的进程；FIFO又叫有名管道（named pipe），可以通过名字来找到管道，所以能够用于不同祖先的进程。

（1）消息传递（管道、FIFO、消息队列）
（2）同步（互斥锁、条件变量、读写锁、信号量）
（3）共享内存区（匿名共享内存区、有名共享内存区）
（4）过程调用（Solaris门、Sun RPC）

# Prosix IPC

* Posix消息队列
* Posix信号量
* Posix共享内存区

管道和FIFO的限制

* OPEN_MAX 一个进程在任意时刻打开的最大描述符数
* PIPE_BUF 可原子地写往一个管道或FIFO的最大数据量

读写锁（read-write lock）

只要没有线程在修改某个给定的数据，那么任意数目的线程都可以拥有该数据的读访问权。仅当没有其他线程在读或修改某个给定给的数据时，当前线程才可以修改它。某些应用中读数据比修改数据频繁，这些应用可以从改用读写锁代替互斥锁中获益。

# 共享内存区

共享内存区是可用IPC形式中最快的。一旦这样的内存区映射到共享它的进程的地址空间，这些进程间数据的传递就不再涉及内核。