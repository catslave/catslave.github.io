---
layout: post
title: Redis In Action
category: Redis
description: Redis实战相关内容，深入Redis配置文件、主从复制、高可用、持久化、键过期策略、内存淘汰策略。
---

## 配置文件

redis.conf常见参数，以下是基于Redis5的配置说明

./redis-server /path/to/redis.conf 按照指定的配置文件启动
include /path/to/other.conf 包含其它的redis配置文件

### NETWORK

网络配置

- bind 127.0.0.1
- protected-mode yes 默认开启安全模式，必须要bind ip或则设计了password才能访问
- port 6379 默认6379，如果设置为0会怎么样？
- tcp-backlog 511 表示tcp连接中已完成3次握手的队列长度，tcp连接中有两个队列，一个等待队列，一个完成队列。系统并发连接量大时，可以调大该值，避免客户端连接缓慢。注意，该值应小于Linux系统的/proc/sys/net/core/somaxconn的值（默认128）。
- timeout 0 客户端心跳超时时间，超时后断开连接，0表示不启用
- tcp-keepalive 300 心跳检测包，定时向client发送tcp_ack包探测client是否存活

### GENERAL

基本配置

- daemonize no 是否以守护进程方式启动
- supervised no 系统启动方式
- loglevel notice 日志等级
- databases 16 设置数据库数量，启动时创建16个数据库，默认选择第0个数据库DB0，使用`select <dbid>`可以切换数据库。数据库间的数据是相互隔离的，客户端可以选择不同的数据库。

### SNAPSHOTTING

RDB快照方式，Redis的默认持久化方式。
> 适合小规模数据或丢失一部分数据也不会造成问题的应用

持久化频率

- save 900 1
- save 300 10
- save 60 10000 指定在多长时间内，多少次更新操作，就将数据同步到数据库文件。 例如60秒内有10000次修改操作，就将数据同步到数据库。
- stop-writes-on-bgsave-error yes yes Redis 会创建一个新的后台进程dump_rdb，同步数据。但是如果后台进程出现问题，主进程也会拒绝写入。 如果设置为no，即bgsave进程报错，也不影响主进程运行。
- rdbcompression yes 启用压缩，对字符串进行压缩
- rdbchecksum yes 启用CRC64校验码

### SECURITY

- requirepass foobared 客户端连接需要密码认证

### CLIENTS

- maxclients 10000 客户端最大连接数

### MEMORY MANAGEMENT

当内存达到上限时，Redis可以配置六种方来释放内存
> volatile-lfu -> Evict using approximated LFU among the keys with an expire set.
> allkeys-lfu -> Evict any key using approximated LFU.
> volatile-random -> Remove a random key among the ones with an expire set.
> allkeys-random -> Remove a random key, any key.
> volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
> noeviction -> Don't evict anything, just return an error on write operations. 

- maxmemory-policy noeviction 默认不释放内存，直接返回错误

### REPLICATION

主从相关的配置

Redis的主从复制采用异步方式进行。
    
    #   +------------------+      +---------------+
    #   |      Master      | ---> |    Replica    |
    #   | (receive writes) |      |  (exact copy) |
    #   +------------------+      +---------------+

- replica-serve-stale-data yes Redis Master主节点可以继续接受客户端的请求
- replica-read-only yes Redis Replica从节点只读
- repl-diskless-sync no Redis支持两种拷贝策略：文件或Socket；文件方式：master将数据写入磁盘文件，然后replica读取该文件。socket方式：master将数据直接通过socket发送给replica。
> 基于socket方法，master一次只能与一个replica进行数据备份，其他replica只能等待。
- repl-diskless-sync-delay 5
- repl-disable-tcp-nodelay no 是否启用TCP_NODELAY，no master数据会立即传输到replica。
- replica-priority

### LAZY FREEING

Redis有两种方式来删除key
1. DEL 阻塞方式：使用DEL，Redis将会停止接收客户端的请求，开始对过期的key进行删除。
2. UNLINK 非阻塞方式：Redis使用另一个线程来删除key，不会阻塞主线程。

- lazyfree-lazy-eviction no 使用DEL同步方式释放内存
- lazyfree-lazy-expire no 使用DEL同步方式删除过期key
- lazyfree-lazy-server-del no
- replica-lazy-flush no

### APPEND ONLY MODE

数据持久化相关配置

Redis默认采用异步方式持久化数据文件的。

Redis支持AOF和RDB两种持久化方法，两种方式可以同时使用。

AOF模式的配置参数

- appendonly no 是否启用AOF模式，no 默认不启用
- appendfsync everysec AOF持久化频率，有3种选项：always 有新数据写入就刷盘；everysec 每秒刷一次；no 使用操作系统自身的刷盘策略
- no-appendfsync-on-rewrite no 在AOF重写期间，是否允许执行AOF文件同步。 no 允许，在AOF重写期间，允许AOF文件继续执行写入，新的命令会写入AOF文件；yes 不允许 将会阻塞AOF文件写入，命令暂存在aof缓存中。对于延迟要求高的应用，可以设置为yes。
> no 重写期间，AOF文件也可以执行写入，这样会存在文件句柄的竞争，造成时延。但安全性高，新的命令会写入AOF文件，即使宕机也可以全部恢复。
> yes 重写期间，会阻塞appendfsync方法执行，即阻塞AOF缓存写入文件和同步，新命令全部暂存在aof_buf缓存区，由操作系统策略执行写文件（Linux默认30s）。所以如果意外宕机，这期间的数据会丢失。
- auto-aof-rewrite-percentage 100 AOF文件大小与上一次的大小比率超过100%，触发重写
- auto-aof-rewrite-min-size 64mb AOF文件超过64M，触发重写
- aof-load-truncated yes
- aof-use-rdb-preamble yes

### REDIS CLUSTER

Redis集群相关的配置

Redis单个实例必须要以cluster集群身份启动才能作为集群节点。

- cluster-enabled yes 作为集群节点启动
- cluster-config-file nodes-6379.conf 集群节点配置文件，由Redis自动创建和维护
- cluster-node-timeout 15000 集群节点超时时间

## 常用命令

**set**

**get**

**select**

**del**

**hset**

**add**

**push**

**设置key的过期时间**

expire/pexpire

setex

## 键过期策略

支持三种键过期处理策略
- 定时
- 惰性
- 定期

Redis使用惰性和定期两种策略

在持久化AOF和RDB中，和主从复制中这两者是如何处理过期的键？

RDB
> 持久化：过期的键不持久化到RDB文件中
> 载入：

AOF：
> 持久化：过期的键Master会写入DEL删除命令
> 载入：检查键是否过期

复制：
> 同步：Master发送DEL命令，Replica执行删除过期键

## 内存淘汰策略

提供六种内存键淘汰策略
- 默认不淘汰
- 所有键中，LRU最近最少使用淘汰
- 所有键中，Radom随机淘汰
- 被设置了过期的键中，LRU最近最少使用淘汰
- 被设置了过期的键中，Radom随机淘汰
- 被设置了过期的键中，TTL存活时间最短的淘汰
