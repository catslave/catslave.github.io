---
layout: post
title: MySQL In Action
category: MySQL
description: MySQL实战 MySQL索引、查询优化、锁等相关概念
---

## EXPLAIN

EXPLAIN访问类型，返回SQL分析的结果，对这些结果信息进行分析。

![](/assets/images/mysql/explain.png)

### type 访问类型

- all 全表扫描，速度最慢，扫描行数最多
- index 基于索引全表扫描，按照索引顺序执行全表扫描，需要根据索引然后回表取数据（二次查询），执行效率与all差不多
- range 范围扫描，有范围的索引扫描，也是基于索引，性能优于index。常见between、and、in、or语句都是范围扫描。
- ref 索引扫描，有ref和ref_eq两种类型。
> ref 基于索引扫描，但是索引不是主键或则唯一索引，索引值存在重复。所以在目标值附近要进行小范围扫描。性能优于全表扫描。
> ref_eq 基于唯一/主键索引扫描，索引值唯一，性能优。
- const 常量扫描，如果是基于唯一/主键索引扫描，MySQL优化器会将其转为一个常量进行查询优化，性能最优。


### select_type 查询类型

- simple 简单SQL查询语句，不包含子查询或UNION

### rows 预计扫描行数

### filtered

role_id没加索引时是10，加索引变为100？

### Extra 查询详细信息

- Using where
- Using join buffer