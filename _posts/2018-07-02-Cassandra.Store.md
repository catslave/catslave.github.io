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

Writer首先获取文件的当前位置,dataFile.position()。

`BigTableWriter.append`


# Memtable

## PartitionPosition
' private final ConcurrentNavigableMap<PartitionPosition, AtomicBTreePartition> partitions = new ConcurrentSkipListMap<>();'通过PartitionPosition来索引memtable，可以实现范围查询。

## MemtablePool

## Keyspace.apply
该方法将数据添加到日志文件，然后更新memtable和index。调用ColumnFamilyStore的apply方法。ColumnFamilyStore最终调用Memtable的put方法。

put方法的流程：根据数据的key获取该key在memtable中的partitionPosition，如果该key在memtable中未找到，则新建一个partitionPosition。put方法还是写内存。

## AtomicBTreePartition
'AtomicBTreePartition.addAllWithSizeDelta'将数据写入memtable。使用自定义BTree来存储数据。该类继承至抽象类'AbstractBTreePartition'，AbstractBTreePartition类持有一个Holder内部类，Hodler类定义了一个Object[]，tree数组。所以memtable内部是使用一个btree数组保存数据的。这是它的数据结构。一个BTree保存的内容是什么？（OceanBase的存储结构是类似的，底层也是用btree实现的，可以参考它的实现。btree不支持删除，支持插入和更新，节点可以分裂不能合并。写操作基于读写锁+Copy On Write，读操作不加锁。并发写Btree原理剖析：https://www.cnblogs.com/foxmailed/p/2914625.html .LevelDB的memtable底层是用SkipList实现的。）

如果是用数组来保存数据，那如何检索数据？通过key可以找到PartitionPosition，PartitionPosition关联到对应的AtomicBTreeParition，一个AtomicBTreePartition对应一组btree。
