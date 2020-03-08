---
layout: post
title: MySQL-InnoDb
category: ReadingNotes,MySQL,Database
description: MySQL技术内幕InnoDB存储引擎 读书笔记
---

# MySQL技术内幕InnoDB存储引擎

# 1. MySQL体系结构和存储引擎
MySQL是一个单进程多线程结构的数据库（SQL Server也是，Oracle在Windows上是）。MySQL数据库
实例在系统上的表现就是一个进程。MySQL实例启动时，会去读取配置文件，如果没有配置文件，会按照
编译时的默认参数设置启动实例（Oracle启动时读取不到配置文件会报错，MySQL通过命令 
mysql --help | grep my.cnf 查找配置文件）。如果有多个配置文件存储，MySQL会以最后一个
配置文件中的参数为准。

## MSQL体系结构

MySQL 体系结构图
![](/assets/images/mysql/MySQL-architech.png)

MySQL由以下几部分组成：
* 连接池组件
* 管理服务和工具组件
* SQL接口组件
* 查询分析器组件
* 优化器组件
* 缓冲（Cache）组件
* 插件式存储引擎
* 物理文件

MySQL是基于插件式的表存储引擎。存储引擎是基于表的，而不是数据库。

## MySQL存储引擎
MySQL数据库的核心在于存储引擎；用户可以根据MySQL预定义的存储引擎接口编写自己的存储引擎。

### InnoDB存储引擎
支持事务，设计目标主要面向在线事务处理（OLTP）的应用。其特点是行锁设计、支持外键，并支持类似于Oracle的非锁定读，即默认读取操作不会产生锁。MySQL 5.5.8 之后 InnoDB为默认的存储引擎。

InnoDB存储引擎将数据放在一个逻辑的表空间中；SQL标准的REPEATABLE隔离级别；还提供了插入缓冲
（insert buffer）、二次写（double write）、自适应哈希索引（adaptive hashindex）、预读（read ahead）
等功能。

InnoDB存储引擎采用了聚集的方式存储表中的数据，每张表的存储都是按主键的顺序进行存放。如果没有
显示地指定主键，默认会为每一行生成一个6字节的ROWID，并以此作为主键。

### MyISAM存储引擎
不支持事务、表锁设计、支持全文索引，主要面向一些OLAP数据库应用。MySQL 5.5.8 之前 MyISAM为默认的存储引擎。
MyISAM存储引擎的缓冲池只缓存（cache）索引文件，而不缓冲数据文件。

MyISAM存储引擎表由MYD和MYI组成，MYD用来存放数据文件，MYI用来存放索引文件。可以使用myisampack工具
压缩数据文件，该工具使用Huffman编码静态算法来压缩数据，因此压缩的表是只读的。

MySQL各存储引擎对比图
![](/assets/images/mysql/MySQL-StorageEngine.png)

## 连接MySQL
连接MySQL操作是一个连接进程和MySQL数据库实例进行通信。本质上是进程通信。常用的进程通信方式有
管道、命名管道、命名字、TCP/IP套接字、UNIX域套接字。

TCP/IP `mysql -h127.0.0.1 -u root -p`
 
# 2. InnoDB存储引擎

InnoDB存储引擎体系架构图
![](/assets/images/mysql/InnoDB-Architect.png)
	
InnoDB存储引擎有多个内存块，组成一个大的内存池：
* 维护所有进程/线程需要访问的多个内部数据结构；
* 缓存磁盘上的数据，方便快速地读取，同时在对磁盘文件的数据修改前在这里缓存；
* 重做日志（redo log）缓冲；

后台线程主要负责刷新内存池中的数据，保证内存中缓存的数据是最近的，以及将已修改的数据文件
刷新到磁盘文件，保证在数据库发生异常的情况下InnoDB能恢复到正常运行状态。

## 后台线程
InnoDB存储引擎是多线程模型，后台有多个不同的线程负责处理不同的任务。
1. Master Thread
负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插入缓冲（INSERT BUFFER）、UNDO页
的回收等。
2. IO Thread
在InnoDB存储引擎中大量使用了AIO（Async IO）来处理写IO请求。IO Thread的工作主要是负责这些IO请求的回调（call back）
处理。有4个IO Thread，分别是write、read、insert buffer和log IO Thread。
3. Purge Thread
事务被提交后，undolog可能不再需要，因此需要Purge Thread来回收已经使用并分配的undo页。

## 内存
1. 缓冲池
InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。在数据库中进行读取页的操作，首先将从磁盘读到的
页存放在缓冲池中，这个过程称为页“FIX”在缓冲池中。下一次再读相同的页时，首先判断该页是否在缓冲池中。若在则称该页被命中，
直接读取该页。否则，读取磁盘上的页。对于页的修改，首先修改缓冲池中的页，然后再以一定的频率（Checkpoint机制）刷新到磁盘上。

缓冲池中缓冲的数据页类型有：索引页、数据页、undo页、插入缓冲（insert buffer）、自适应哈希索引（adaptive hash index）
、InnoDB存储的锁信息（lock info）、数据字典信息（data dictionary）等。

InnoDB内存数据对象图
![](/assets/images/mysql/BufferPool-Architect.png)

允许有多个缓冲池实例。每个页根据哈希值平均分配到不同的缓存池。

2. LRU List、Free List和Flush List
数据库中的缓冲池是通过LRU算法来进行管理的。即最频繁使用的页在LRU列表的前端，
而最少使用的页在LRU列表的尾端。当缓冲池不能存放新读取到的页时，将首先释放LRU列表中
尾端的页。

在InnoDB存储引擎中，缓冲池中页的大小默认为16KB，并对LRU算法做了一些优化，在LRU列表中加入了midpoint位置。新
读取到的页，并不直接放入LRU列表的首部，而是放入到LRU列表的midpoint位置。把midpoint之后的列表称为old列表，之
前的列表称为new列表。new列表中的页都是最为活跃的热点数据。

3.重做日志缓冲
redo log buffer
* Master Thread每一秒将重做日志缓冲刷新到重做日志文件
* 每个事务提交时会重做日志缓冲刷新到重做日志文件
* 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件。

4.额外的内存池

## Checkpoint技术
缓冲池的设计目的是为了协调CPU速度与磁盘速度的鸿沟。因此页的操作首先都是在缓冲中完成的。如果一条DML语句改变了页中的记录，该页称为脏页，即缓冲池中的页的版本要比磁盘的新。在缓冲池将页的新版本刷新到磁盘时如果发送宕机，那么数据就不能恢复了。因此当前事务数据库系统都采用了Write Ahread Log策略，即当事务提交时，先写重做日志，再修改页。当发生宕机而导致数据丢失时，通过重做日志来完成数据恢复。事务ACID中D（Durability持久性）的要求。

Checkpoint（检查点）技术的目的是解决以下几个问题：
* 缩短数据库的恢复时间；
* 缓冲池不够用时，将脏页刷新到磁盘；
* 重做日志不可用时，刷新脏页。

在InnoDB存储引擎内部，有两种Checkpoint，分别为：
1）Sharp Checkpoint：发生在数据库关闭时将所有脏页都刷新回磁盘；
2）Fuzzy Checkpoint：在数据库运行时进行页的刷新，每次只刷新一部分脏页。
Fuzzy Checkpoint 发生情况：
* Master Thread Checkpoint：异步操作，用户查询线程不会阻塞。
* FLUSH_LRU_LIST Checkpoint：检查操作放在一个单独的Page Cleaner线程中进行。
* Async/Sync Flush Checkpoint：重做日志文件不可用情况，强制将页刷新回磁盘；该操作同样放到了一个单独的Page Cleaner Thread中，故不会阻塞用户查询线程。
* Dirty Page too much Checkpoint：脏页数量太多，强制进行Checkpoint。

## Master Thread 工作方式
Master Thread具有最高的线程优先级别。其内部由多个循环（loop）组成：主循环（loop）、后台循环（backgroup loop）、刷新循环（flush loop）、暂停循环（suspend loop）。Master Thread会根据数据库运行状态在这些循环中进行切换。

Loop主循环，有两大部分操作：每秒钟的操作和每10秒的操作。
loop循环通过thread sleep来实现，在负载很大的情况下可能会有延迟（delay）。

每秒一次的操作包括：
* 日志缓冲刷新到磁盘，即使这个事务还没有提交（总是）；
* 合并插入缓冲（可能）；
* 至多刷新100个InnoDB的缓冲池中的脏页到磁盘（可能）；
* 如果当前没有用户活动，则切换到background loop（可能）；

每10秒的操作包括：
* 刷新100个脏页到磁盘（可能的情况下）；
* 合并至多5个插入缓冲（总是）；
* 删除无用的Undo页（总是）；
* 刷新100个或者10个脏页到磁盘（总是）。

若当前没有用户活动（数据库空闲时）或者数据库关闭（shutdown），就会切换到backgroup loop。background loop会执行以下操作：
* 删除无用的Undo页（总是）；
* 合并20个插入缓冲（总是）；
* 跳回到主循环（总是）；
* 不断刷新100个页直到符合条件（可能，跳转到flush loop中完成）。

若flush loop中也没有什么事情可以做，就会切换到suspend_loop，将Master Thread挂起，等待事件的发生。

## InnoDB关键特性

### 插入缓冲（Insert buffer）
1.Insert Buffer
在InnoDB存储引擎中，主键是行唯一的标识符。通常应用程序中行记录的插入顺序是按照主键递增的顺序进行插入的。因此，插入聚集索引（Primary Key）一般是顺序的，不需要磁盘的随机读取。

对于一张表上有多个非聚集的辅助索引（secondary index），在进行插入操作时，非聚集索引叶子节点的插入不再是顺序的了，这时就需要离散地访问非聚集索引页，由于随机读取的存在而导致了插入操作性能下降。

InnoDB存储引擎，对于非聚集索引的插入或更新操作，不是每一次直接插入到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入；若不在，则先放入到一个Insert Buffer对象中。（这里看不是很懂，以后再来看一遍！）

2.Change Buffer
Insert Buffer、Delete Buffer、Purge Buffer。Change Buffer适用的对象依然是非唯一的辅助索引。对一条记录的UPDATE操作可能分为两个过程：
* 将记录标记为已删除
* 真正将记录删除。

3.Insert Buffer的内部实现
Insert Buffer的使用场景，即非唯一辅助索引的插入操作。其内部数据结构是一颗B+树。全局只有一颗Insert Buffer B+树，负责对所有的表的辅助索引进行Insert Buffer。而这颗B+树存放在共享空间中，默认也就是ibdata1中。

B+树非页节点存放的是查询的search key（键值 space|marker|offset）。space表示待插入记录所在表的表空间id，offset表示页所在的偏移量。当一个辅助索引要插入到页（space, offset）时，如果这个页不在缓冲池中，InnoDB首先构造一个search key，接下来查询Insert Buffer这颗B+树，然后再将这条记录插入到树的叶子节点中。

4.Merge Insert Buffer

### 两次写（Double Write）

InnoDB存储引擎doublewrite架构
![](/assets/images/mysql/doublewrite.png)

doublewrite由两部分组成，一部分是内存中的doublewrite buffer，大小为2MB，另一部分是物理磁盘上共享表空间中连续的128个页，即2个区（extent），大小同样为2MB。在对缓冲池的脏页进行刷新时，并不直接写磁盘，而是全通过memcpy函数将脏页先复制到内存中的doublewrite buffer，之后通过doublewrite buffer再分两次，每次1MB顺序地写入共享表空间的物理磁盘上，然后马上调用fsync函数，同步磁盘，避免缓冲写带来的问题。在这个过程中，因为doublewrite页是连续的，因此这个过程是顺序写的，开销并不是很大。在完成doublewrite页的写入后，再将doublewrite buffer中的页写入各个表空间文件中，此时的写入则是离散的。

如果操作系统在将页写入磁盘的过程中发送了崩溃，在恢复过程中，InnoDB存储引擎可以从共享表空间中的doublewrite中找到该页的一个副本，将其复制到表空间文件，再应用重做日志。

### 自适应哈希索引（Adaptive Hash Index）
哈希是一种非常快的查找方法，一般情况下查找时间复杂度为O(1)，而B+树的查找次数，取决于B+树的高度，在生成环境中，B+树的高度一般为3~4层，故需要3~4次的查询。

InnoDB存储引擎会监控对表上各索引页的查询，可以建立索引提高速度就会创建哈希索引，称为自适应哈希索引（Adaptive Hash Index，AHI）。AHI是通过缓冲池的B+树页构造而来，因此建立的速度很快，而且不需要对整张表构建哈希索引。InnoDB存储引擎会自动根据访问的频率和模式来自动地为某些热点页建立哈希索引。

### 异步IO（Async IO）


### 刷新邻接页（Flush Neighbor Page）
当刷新一个脏页时，InnoDB存储引擎会检测该页所在区（extent）的所有页，如果是脏页，那么一起进行刷新。通过AIO可以将多个IO写入操作合并为一个IO操作。
对于传统机械硬盘建议开启该特性，而对于固态硬盘有着超高IOPS性能的磁盘，则建议关闭此特性。

# 3. 文件
参数文件
日志文件
socket文件
pid文件
MySQL表结构文件
存储引擎文件

# 4. 表

## 4.1 索引组织表
在InnoDB存储引擎中，表都是根据主键顺序组织存放的，这种存储方式的表称为索引组织表（index organized table）。
如果在创建表时没有显式地定义主键，则InnoDB存储引擎会按2种方式选择或创建主键：
1）首先判断表中是否有非空的唯一索引（Unique NOT NULL），如果有，则该列即为主键（如有多个，则根据定义索引的顺序选择）；
2）如果不符合上述条件，InnoDB存引擎自动创建一个6字节大小的指针。（6字节有多大范围？2^6？）
_rowid只能用于查看单个列为主键的情况，多列组成主键无法查看。

## 4.2 InnoDB逻辑存结构
所有数据都被逻辑地存放在一个空间中，称为表空间（tablespace）。表空间又由段（segment）、区（extent）、页（page）/块（block）组成。

### 4.2.1 表空间
表空间，InnoDB存储引擎逻辑结构最高层，所有数据都存放在表空间。
InnoDB存储引起有一个共享表空间ibdata1，即所有数据都存放在这个表空间内。如果用户启用了参数innodb_file_per_table，则每张表内的数据
可以单独放到一个表空间内（但是每张表的表空间内存放的只是数据、索引和插入缓冲Bitmap页，其他类的数据，如回滚（undo）信息，插入缓冲
索引页、系统事务信息、二次写缓冲（Double write buffer）等还是存放在原来的共享表空间内。）。

这同时说明，即使在启动了参数innodb_file_per_table之后，共享表空间还是会不断地增加其大小。

### 4.2.2 段
表空间由各个段组成，常见的段有数据段、索引段、回滚段等。

InnoDB存储引擎表是索引结构组织的（index organized），因此数据即索引，索引即数据。那么数据
段即为B+树的叶子节点（Leaf node segment），索引段即为B+树的非索引节点（Non-leaf node segment）。

### 4.2.3 区
区是由连续页组成的空间，在任何情况下每个区的大小都为1MB。为了保证区中页的连续性，InnoDB存储引擎
一次从磁盘中申请4~5个区。在默认情况下，InnoDB存储引擎页的大小为16KB，即一个区中一共有64个连续的页。

InnoDB 1.0.x 版本引入压缩页，通过参数KEY_BLOCK_SIZE设置为2K、4K、8K，因此每个区对应页的数量为
512、256、128。
InnoDB 1.2.x 版本新增参数innodb_page_size，来将默认页的大小设置为4K、8K。

碎片页（fragment page）：在用户启用了参数innodb_file_per_table后，创建的表默认大小是96KB。
但是区中是64个连续的页，创建的表的大小至少是1MB才对？其实因为在每个段开始时，先用32个页大小
的碎片页（fragment page）来存放数据，在使用完这些页之后才是64个连续页的申请。这样做的目的是，
对于一些小表，或者是undo这类的段，可以在开始时申请较少的空间，节省磁盘容量的开销。

### 4.2.4 页
页（块）是InnoDB磁盘管理的最小单位。在InnoDB存储引擎中，默认每个页的大小为16KB。

在InnoDB存储引擎中，常见的页类型有：
* 数据页（B-tree Node）
* undo 页（undo Log Page）
* 系统页（System Page）
* 事务数据页（Transaction system Page）
* 插入缓冲位图页（Insert Buffer Bitmap）
* 插入缓冲空闲列表页（Insert Buffer Free List）
* 未压缩的二进制大对象页（Uncompressed BLOB Page）
* 压缩的二进制大对象页（compressed BLOB Page）

### 4.2.5 行
InnoDB存储引擎是面向行的（row-oriented），也就是说数据是按行进行存放的。每个页存放的行记录也是
有硬性定义的，最多运行存放16KB/2-200行的记录，即7992行记录。

## 4.3 InnoDB行记录格式
InnoDB存储引擎记录是以行的形式存储的。这意味着页中保存着表中一行行的数据。在InnoDB 1.0.x 版本之前，
InnoDB存储引擎提供了Compact和Redundant两种格式来存放行记录数据。Redundant格式是为兼容之前版本而保留的。在MySQL 5.1版本中，默认设置为Compact行格式。

数据库实例的作用之一就是读取页中存放的行记录。如果用户自己知道页中行记录的组织规则，也可以自行通过
编写工具的方式来读取其中的记录，如之前介绍的py_innodb_page_info工具。

### 4.3.1 Compact 行记录格式
Compact行记录是在MySQL 5.0引入的，其设计目标是高效地存储数据。简单来说，一个页中存放的行数据越多，
其性能就越高。

Compact行记录格式的首部是一个非NULL变长字段长度列表（如何定位？），并且其是按照列的顺序逆序放置的，其长度为：
* 若列的长度小于255字节，用1字节表示；
* 若大于255个字节，用2字节表示。

变长字段长度列表是逆序存放，InnoDB每行有隐藏列TransactionID和Roll Pointer。固定长度CHAR字段在未能
完全占用其长度空间时，会用0x20来进行填充。InnoDB存储引擎在页内部是通过一种链表的结构来串连各个行记录的。

NULL标志位转换成二进制00000110，为1的值代表第2列和第3列的数据为NULL。在其后存储列数据的部分，用户会发现没有存储NULL列，而只存储了第1列和第4列非NULL的值。因此这例子说明，不管是CHAR类型还是VARCHAR类型，在compact格式下NULL值都不占用任何存储空间。

### 4.3.2 Redundant 行记录格式
Redundant是MySQL 5.0 版本之前InnoDB的行记录存储方式，MySQL 5.0 支持Redundant是为了兼容之前版本的
页格式。

Redundant行记录格式的首部是一个字段长度偏移列表，同样是按照列的顺序逆序放置的。第二个部分为记录头信息（record header），占用6字节（48位）。n_fields值代表一行中列的数量，占用10位。所以MySQL数据库一行
支持最多的列为1023。1byte_offs_flag值定义了偏移列表占用1字节还是2字节。

对于VARCHAR类型的NULL值，Redundant行记录格式同样不占用任何存储空间，而CHAR类型的NULL值需要
占用空间。

### 4.3.3 行溢出数据
MySQL官方手册中定义的65535长度是指所有VARCHAR列的长度总和，如果列的长度总和超出这个长度，依然无法创建。InnoDB存储引擎的页为16KB，即16384字节，怎么能存放65532字节？因为在一般情况下，InnoDB存储引擎的数据都是存放在页类型为B-tree node中。但是发生行溢出时，数据存放在页类型为Uncompress BLOB页中。

InnoDB存储引擎表是索引组织的，即B+Tree的结构，这样每个页中至少应该有两条记录（否则失去了B+Tree的意义，变成链表了）。因此，如果页中只能存放下一条记录，那么InnoDB存储引擎会自动将行数据存放到溢出页中
。如果可以在一个页中至少放入两行数据，那VARCHAR类型的行数据就不会存放到BLOB页中去。对于TEXT或
BLOB的数据类型，是放在数据页中还是BLOB页中，也是至少保证一个页能存放两条记录。

### 4.3.4 Compressed和Dynamic行记录格式
InnoDB 1.0.x 版本开始引入了新的文件格式（file format，新的页格式），以前支持的Compact和Redundant
格式称为Antelope文件格式，新的文件格式称为Barracuda文件格式。Barracuda文件格式下拥有两种新的行记录格式：Compressed和Dynamic。

新的两种记录格式对于存放在BLOB中的数据采用了完全的行溢出的方式，在数据页中只存放20个字节的指针，
实际的数据都存放在Off Page中，而之前的Compact和Redundant两种格式会存放768个前缀字节。

Compressed行记录格式的另一个功能就是，存储在其中的行数据会以zlib的算法进行压缩，因此对于BLOB、TEXT、VARCHAR这类大长度类型的数据能够进行非常有效的存储。

### 4.3.5 CHAR的行结构存储
行结构的内部存储，每行的变长字段长度的列表都没有存储CHAR类型的长度。从MySQL 4.1 版本开始，CHAR（N）
中的N指的是字符的长度，而不是之前版本的字节长度。（字符长度？）也就是说在不同的字符集下，CHAR类型
列内部存储的可能不是定长的数据。因此，对于多字节字符编码的CHAR数据类型的存储，InnoDB存储引擎在
内部将其视为变长字符类型。这也就意味着在变长长度列表中会记录CHAR数据类型的长度。（前面又说没有？）
因此可以认为在多字节字符集的情况下，CHAR和VARCHAR的实际行存储基本是没有区别的。

## 4.4 InnoDB数据页结构
页类型为B-tree Node的页存放的即是表中行的实际数据。
InnoDB数据页由一下7个部分组成：
* File Header（文件头）
* Page Header（页头）
* Infimun和Supermum Records
* User Records（用户记录，即行记录）
* Free Space（空闲空间）
* Page Directory（页目录）
* File Trailer（文件结尾信息）
其中File Header、Page Header、File Trailer的大小是固定的，分别为38、56、8字节，这些空间用来
标记该页的一些信息，如Checksum，数据页所在B+树索引的层数等。User Records、Free Space、Page Directory这些部分为实际的行记录存储空间，因此大小是动态的。

### 4.4.3 Infimum和Supremum Record
在InnoDB存储引擎中，每个数据页中有两个虚拟的行记录，用来限定记录的边界。Infimum记录是比该页
任何主键值都要小的值，Supremum指比任何可能大的值还要大的值。

### 4.4.4 User Record和Free Space
User Record就是之前讨论过的部分，即实际存行记录的内容。再次强调，InnoDB存储引擎表总是B+树索引
组织的。
Free Space就是空闲空间，同样也是个链表数据结构。在一条记录被删除后，该空间会被加入到空闲链表中。

### 4.4.5 Page Directory
（这里还是要结合下外面资料一起看才好理解）
Page Directory（页目录）中存放了记录的相对位置（页相对位置），这些记录指针称为Slots（槽）或
目录槽（Directory Slots）。在InnoDB中并不是每个记录拥有一个槽，InnoDB存储引擎的槽是一个稀疏目录
（sparse directory），即一个槽中可能包含多个记录。当记录被插入或删除时需要对槽进行分裂或平衡的
维护操作。

在Slots中记录按照索引键值顺序存放，这样可以利用二叉查找迅速找到记录的指针。由于在InnoDB存储引擎中
Page Directory是稀疏目录，二叉查找的结果只是一个粗略的结果，因此InnoDB存储引擎必须通过recorder header中的next_record来继续查找相关记录。

B+树索引本身并不能找到具体的一条记录，能找到只是该记录所在的页。数据库把页载入到内存，然后通过
Page Directory再进行二叉查找。

### 4.4.6 File Trailer
为了检测页是否已经完整地写入磁盘，InnoDB存储引擎的页中设置了File Trailer部分。

File Trailer只有一个FIL_PAGE_END_LSN部分，占用8字节。前4字节代表该页的checksum值，最后4字节和File Header中的FIL_PAGE_LSN相同。将这两个值与File Header中的值进行比较，看是否一致，以此来保证页
的完整性（not corrupted）。（这里要学习下它的设计思路，为什么这样来设计？）

在默认配置下，InnoDB存储引擎每次从磁盘读取一个页就会检测该页的完整性，即页是否发生Corrupt，这
就是通过File Trailer部分进行检测，而该部分的检测会有一定的开销。用户可以通过参数innodb_checksums
来开启或关闭对这个页完整性的检查。

### 4.4.7 InnoDB数据页结构实例分析
（这一节要花时间来认真看下，才能吸收前面所讲的。弄明白看这些是为了什么？学习什么？想
从中获取什么？存储方式、设计思路）

# 5. 索引与算法

## 5.1 InnoDB 存储引擎索引概述
InnoDB存储引擎支持以下几种常见的索引：
* B+树索引
* 全文索引
* 哈希索引

B+树索引并不能找到一个给定键值的具体行。B+树索引能找到的只是被查找数据行所在的页。然后数据库
通过页读入到内存，再在内存中进行查找，最后得到要查找的数据。

二分查找：每页Page Directory中的槽是按照主键的顺序存放的，对于某一条具体的查询是通过对Page Directory
进行二分查找得到的。

平衡二叉树：查询速度的确很快，但是维护一颗平衡二叉树的代价是非常大的。

B+树：为了保持平衡对于新插入的键值可能需要做大量的拆分页（split）操作。因为B+树结构主要用于
磁盘，页的拆分意味着磁盘的操作，所以应该在可能的情况下尽量减少页的拆分操作。

## 5.4 B+树索引
B+树索引的本质就是B+树在数据库中的实现。数据库中的B+树索引可以分为聚集索引（clustered index）
和辅助索引（secondary index），两者内部都是B+树，即高度平衡的，叶子节点存放着所有的数据。聚集索引
和辅助索引不同是，叶子节点存放的是否是一整行的信息。

辅助索引查找到记录后，根据这个记录到聚集索引查找对应的页。