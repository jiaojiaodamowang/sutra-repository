# Lock Introduction

## 颗粒度
> 锁的颗粒度越细，并发性能越好。

||InnoDB|MyISAM|
|-|-|-|
|颗粒度|行级锁|表级锁|

*默认情况下，InnoDB使用`行锁`。即使执行`DDL`修改表结构也是通过`MDL(Metadata Locks)`来锁定整个表。

- LOCK TABLES `table name` READ：对表加`S锁`
- LOCK TABLES `table name` WRITE：对表加`X锁`
- UNLOCK TABLES：解除所有表锁

## 锁分类

- S：即`共享锁`，事务读取记录（SELECT XXX FROM TT LOCK IN SHARE）时，允许多个事务同时获取S锁，互相不冲突
- X：即`排他锁`，事务修改记录（SELECT XXX FROM TT FOR UPDATE）时获取X锁，只允许一个事务获取X锁，其他事务会被阻塞

> 表锁与行锁之间是否冲突？如果要加表锁，是否需要遍历记录行判断是否存在行锁？

意向锁（Intention Locks）：
- IS（Intention Shared Lock）：共享意向锁
- IX（Intention Exclusive Lock）：独占意向锁

> 但需要对表中某行记录加`S锁`（或者`X锁`），先在表上加个`IS锁`（或者`IX锁`）。这样操作有以后，如果需要加表锁，只需要判断表上是否有存在意向锁。

|冲突|S|X|IS|IX|
|-|-|-|-|-|
|S|不冲突|冲突|不冲突|不冲突|
|X|冲突|冲突|冲突|冲突|
|IS|不冲突|冲突|不冲突|不冲突|
|IX|冲突|冲突|不冲突|不冲突|

### InnoDB行锁

> 记录锁（Record Locks）：锁住当前行，作用在索引上，`锁记录即是锁索引`。

