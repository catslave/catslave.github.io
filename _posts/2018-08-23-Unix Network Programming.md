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

 4种不同的IPC形式
（1）消息传递（管道、FIFO、消息队列）
（2）同步（互斥锁、条件变量、读写锁、信号量）
（3）共享内存区（匿名共享内存区、有名共享内存区）
（4）过程调用（Solaris门、Sun RPC）

# Prosix IPC

* Posix消息队列
* Posix信号量
* Posix共享内存区

## 进程、线程与信息共享
Unix进程间的信息共享方式
1）文件系统共享，但是每个进程都得穿越内核。
2）内核信息共享，管道是这种共享类型的一个例子。每次访问共享信息都需要进行一次内核调用。
3）共享内存区，不需要调用内存操作。进程内的所有线程都共享同样的全局变量（也就是说共享内存区的概念对这种模型是内在的）

管道和FIFO的限制

* OPEN_MAX 一个进程在任意时刻打开的最大描述符数
* PIPE_BUF 可原子地写往一个管道或FIFO的最大数据量

读写锁（read-write lock）

只要没有线程在修改某个给定的数据，那么任意数目的线程都可以拥有该数据的读访问权。仅当没有其他线程在读或修改某个给定给的数据时，当前线程才可以修改它。某些应用中读数据比修改数据频繁，这些应用可以从改用读写锁代替互斥锁中获益。

# 共享内存区

共享内存区是可用IPC形式中最快的。一旦这样的内存区映射到共享它的进程的地址空间，这些进程间数据的传递就不再涉及内核。

## 管道
所有式样的Unix都提供管道。它由pipe函数创建，提供一个单路（单向）数据流。（也有全双工管道sockepair函数。）
{% highlight java %}
#include <unistd.h>

int pipe(int fd[2]);
{% endhighlight %}
该函数返回两个文件描述符：fd[0]和fd[1]。前者打开来读，后者打开来写。

管道的典型用途是为两个不同进程（父子进程）提供进程间的通信手段。父进程创建一个管道然后fork一个自身副本。接着父进程关闭管道的读出端，子进程关闭管道的写入端。这就在父子进程间提供了一个单向数据流。
Unix shell中输入命令：who | sort | lp 该shell执行将创建三个进程和两个管道。
前面所示的管道都是半双工的即单向的，只提供一个方向的数据流。当需要一个双向数据流时，我们必须创建两个管道，每个方向一个。

父子进程间通信，子进程在往管道写入最终数据后调用exit首先终止。它随后变成了一个僵尸进程（zombie）：自身已终止，但其父进程仍在运行且尚未等待该子进程的进程。当子进程终止时，内核还给父进程产生一个SIGCHLD信号，不过父进程没有捕获这个信号，而该信号的默认行为就是忽略。此后不久，父进程在从管道读入最终数据后返回。父进程随后调用waitpid取得已终止子进程的终止状态。要是父进程没有调用waitpid，而是直接终止，那么子进程将成为托孤给init进程的孤儿进程，内核将为此向init进程发送另外一个SIGCHLD信号，init进程随后将取得僵尸进程的终止状态。

### 全双工管道socketpair
全双工管道：整个管道只有一个缓冲区，写入管道的任何数据都添加到该缓冲区的末尾，从管道读出的都是取自该缓冲区的开头的数据。该实现的问题是虽然是双向通信，但是无法同时进行读和写。一个进程往该全双工管道写入数据，过后再对该管道调用read时，有可能读回刚写入的数据。我们需要的是两个独立的数据流，每个方向一个。

所以真正的全双工管道是由两个半双工管道构成的。写入fd[1]的数据只能从fd[0]读出，写入fd[0]的数据只能从fd[1]读出。

### FIFO
由mkfifo函数创建。
{% highlight java %}
#include <sys/teyps.h>
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);
{% endhighlight %}
pathname 为FIFO的名字，FIFO也是半双工的。对管道和FIFO的write总是往末尾添加数据，对他们的read则总是从开头返回数据。如果对管道或FIFO调用lseek，那就返回ESPIPE错误。

如果当前尚没有任何进程打开某个FIFO来写，那么打开该FIFO来读的进程将阻塞。父子进程都打开同一个FIFO来读，然而此时并没有任何进程打开该文件来写，于是父子进程都阻塞，出现死锁（deadlock）。

# JNI

native源码路径openjdk/jdk/src/share/native/java/net/net_util.h

native方法
JNIEXPORT void JNICALL Java_java_net_Inet4Address_init(JNIEnv *env, jclass cls);

对应Java的native方法
java/net/Inet4Address.java

private static native void init();

Java_表示前缀
java_net_Inet4Addrss_表示类名 -> java/net/Inet4Address
init表示方法 -> Inet4Address.init()
void -> 表示返回值 Inet4Address.init():void



