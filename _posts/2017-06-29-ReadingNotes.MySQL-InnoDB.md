---
layout: post
title: MySQL-InnoDb
category: ReadingNotes
description: MySQL技术内幕InnoDB存储引擎 读书笔记
---

# MySQL技术内幕InnoDB存储引擎

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