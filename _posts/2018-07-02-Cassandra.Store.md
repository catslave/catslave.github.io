---
layout: post
title: Cassandra In Action - Store
category: Cassandra
description: 梳理Cassandra的写过程。特别是Memtable的flush过程。
---

# Cassandra.Store

## ColumnFamilyStore
ColumnFamilyStore switchMemtable方法将刷盘动作提交flush到flushExecutor线程池处理。flushExecutor线程池的数量等于memtable_flush_writers的数量。flushExecutor将flush事件提交到perDiskflushExecutors进行处理。flush操作会阻塞知道所有刷盘动作执行完成。

'createSSTableMultiWriter'方法用于创建SSTableWriter。SSTableWriter内部实现了刷盘操作。

`flush`方法是由谁、什么时间、什么情况下调用的目前还不清楚，这里后面可以再分析，现在主要分析刷盘时候做了什么事情，如何刷盘的。现在知道memtable是直接插入内存，这个速度会非常的快，那刷盘呢？刷盘是怎么刷的，如何保持高效？

`forceFlush`将当前未刷盘的memtable刷盘：1.获取当前memtable；2.调用`waitForFlushes`方法，创建一个`ListenableFutureTask<CommitLogPosition>`任务，提交到`postFlushExecutor`线程池执行。

`Flush`线程的`flushMemtable`方法执行刷盘操作：调用memtable.flushRunnables方法创建FlushRunnable任务，`flushRunnables`方法调用memtable.createFlushRunnables方法创建FlushRunnable任务。FlushRunnable实例化SSTableWriter。当任务创建完成后，将任务提交到perDiskflushExecutors线程池中执行。`Flush`线程首先加锁，然后将memtable标记为正在刷盘，再调用flushMemtable方法执行刷盘。`Memtable.Flush`线程调用writeSortedContents方法，遍历所有的PartitionPosition将unfilteredIterator添加到writer。

`IndexWriter`负责写入索引文件（index文件，同时还负责写入BloomFilter、Summary文件），`SequentialWriter`负责写入数据文件（data文件）

`BigTableWriter.append()`方法实际写入一行记录：当memtable刷盘时，会把内存中有序的数据追加到BigTableWriter。`append()`方法首先获取文件的当前位置`startPosition = dataFile.position()`，然后调用`columnIndex.buildRowIndex()`方法写入数据文件，最后调用`indexWriter.append()`写入索引文件。`columnIndex.buildRowIndex()`方法首先`writePartitionHeader()`写头部信息，然后`add()`写入数据文件，最后`finish()`写入结束符。`columnIndex.add()`方法首先获取文件的当前位置`currentPosition()`，然后调用`UnfilteredSerializer.serializer.serialize()`方法，最终由该方法调用`sequentialWriter`写入数据。（Cassandra源码分析-存储引擎：http://zqhxuyuan.github.io/2016/10/19/Cassandra-Code-StorageEngine/#BigTableWriter）

`RowIndexEntry`索引文件

## ColumnFamilyStore.Flush
该任务类用于交换memtable，将已满的memtable置为非活跃只读状态同时创建一个新的活跃memtable用于数据写入。


# Memtable
数据先写入memtable再写入文件，现在已经分析了数据如何写入memtable和数据如何写入文件，现在要分析数据何时从memtable写入文件，这是一个关键的节点也许很简单也许要再找找。好像是有`Tracker`Memtable的生命周期管理类来进行标识是否可刷盘。`ColumnFamilyStore.Flush`构造函数创建新的memtable并调用`Tracker.switchMemtable`。明天研究下文件写入的格式！

## PartitionPosition
' private final ConcurrentNavigableMap<PartitionPosition, AtomicBTreePartition> partitions = new ConcurrentSkipListMap<>();'通过PartitionPosition来索引memtable，可以实现范围查询。

## MemtablePool

## Keyspace.apply
该方法将数据添加到日志文件，然后更新memtable和index。调用ColumnFamilyStore的apply方法。ColumnFamilyStore最终调用Memtable的put方法。

put方法的流程：根据数据的key获取该key在memtable中的partitionPosition，如果该key在memtable中未找到，则新建一个partitionPosition。put方法还是写内存。

## AtomicBTreePartition
'AtomicBTreePartition.addAllWithSizeDelta'将数据写入memtable。使用自定义BTree来存储数据。该类继承至抽象类'AbstractBTreePartition'，AbstractBTreePartition类持有一个Holder内部类，Hodler类定义了一个Object[]，tree数组。所以memtable内部是使用一个btree数组保存数据的。这是它的数据结构。一个BTree保存的内容是什么？（OceanBase的存储结构是类似的，底层也是用btree实现的，可以参考它的实现。btree不支持删除，支持插入和更新，节点可以分裂不能合并。写操作基于读写锁+Copy On Write，读操作不加锁。并发写Btree原理剖析：https://www.cnblogs.com/foxmailed/p/2914625.html .LevelDB的memtable底层是用SkipList实现的。）

如果是用数组来保存数据，那如何检索数据？通过key可以找到PartitionPosition，PartitionPosition关联到对应的AtomicBTreeParition，一个AtomicBTreePartition对应一组btree。

## Tracker
Memtable的生命周期管理类来进行标识是否可刷盘，方法switchMemtable调用view.switchMemtable方法将活跃的memtable标识为只读，同时创建新的活跃memtable。

## View
`lifecycle\View`将活跃的memtable标识为immutable，同时创建新的活跃memtable。



# SSTable
SSTable在磁盘的数据存储是有序的，分为数据文件和索引文件。SSTable分成SSTableWriter和SSTableReader，具体操作实现类为：BigTableWriter和BigTableReader。BigTableWriter针对索引文件和数据文件的写入分别是：IndexWriter和SequentialWriter。

## Unfiltered
`Unfiltered->Row->AbstractRow->BTreeWow`表示一行数据。

## SequentialWriter
`DataOutput->DataOutputPlus->DataOutputStreamPlus->BufferedDataOutputStreamPlus->SequentialWriter`，DataOutput接口定义了将Java数据类型转换为字节写入到二进制流方法。接口DataOutputPlus继承了DataOutput，同时扩展了将ByteBuffer和Memory转换为字节写入流方法。抽象类DataOutputStreamPlus实现了DataOutputPlus接口同时继承了OutputStream抽象类，即提供了ByteBuffer和Memory的数据写入同时也提供了数据流的输出。实现类BufferedDataOutputStreamPlus继承DataOutputStreamPlus实现了部分接口，比如write(byte[] b)方法：将字节写入ByteBuffer。该类是线程不安全的。SequentialWriter类继承了BufferedDataOutputStreamPlus同时实现了Transactional事务管理接口。

`SequentialWriter`在实例化时调用父类构造函数初始化ByteBuffer（`ByteBuffer.allocate()`默认64K：64 * 1024，堆内内存，默认容量大小10M，超过10M就刷盘，这些数据定义在SequentialWriterOption里面），定义了openChannel打开文件方法，sync刷盘方法将byteBuffer写入文件并强制force刷盘。目前有三处会调用SequentialWriter.sync方法，BufferedDataOutputStreamPlus的所有写入方法在写完数据后都会刷盘！ BigTableWriter的openFinalEarly方法：indexFile.sync()和dataFile.sync()，打开文件之前会确认byteBuffer的数据都已刷盘。OnDiskIndexBuilder的finish方法：out.sync()。