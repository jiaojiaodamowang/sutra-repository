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

示例：
1. `事务A`执行`SELECT * FROM 'table_name' WHERE name = 'xx' FOR UPDATE;`，那么`name=xx`这条记录被锁定，其他事务无法插入、修改、删除`name=xx`的记录。
2. 此时`事务A`还未提交，另一个`事务B`执行`INSERT INTO 'table_name' (name) VALUES ('xx')`，被阻塞。
3. 接下来再有`事务C`执行`INSERT INTO 'table_name' (name) VALUES ('yy')。

> `事务C`是否会被阻塞，取决于`name`列有没有索引。因为表锁实际是所用到索引上，如果列没有索引，只能回到聚簇索引上找，而聚簇索引上只有主键，故只能锁全表，`事务C`也被阻塞。如果有索引，`事务C`就不会阻塞。
>
> 因此没有索引的列不要轻易使用锁。 

> 间隙锁（Gap Locks）：要给还不存在的记录加锁，预防`幻读`的出现，需要使用间隙锁。

示例：
![lock_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/lock_intro_1.jpg)

1. 现在有1、3、5、10四条记录，数据页中还有两条虚拟的记录`Infimum`、`Supremum`
2. 间隙锁锁的就是记录之间的间隙
3. 如果执行`SELECT * from 'table_name' WHERE id in (3,5) FOR UPDATE;`，便锁住`记录3`、`记录5`的间隙，此时插入`id=4`，就被阻塞了。这样就避免了`幻读`

> Q：间隙锁之前是否冲突？
> 
> A：不会。间隙锁的目的是为了防止其他事物插入数据，即使两个间隙锁锁住相同的区间，因为目的是一致的，所以不会产生冲突。
> 
> 
> Q：如何禁用间隙锁？
> 
> A：间隙锁在事务隔离级别为`READ REPEATABLE`时生效，如果调整为`READ COMMITTED`。此时间隙锁对于搜索和索引扫描是禁用的，仅用于外键约束检查和唯一键检查。

> Next-Key Locks：记录锁 + 间隙锁

示例：
间隙锁锁定(3, 5)这个区间，`Next-Key Locks`可以锁定(3, 5]，这样就能防止id=5的幻读。

> 插入意向锁（Insert Intention Locks）：它也是一类间隙锁，但目的不是锁定区间，而是等待某个区间。

示例：
1. 执行`SELECT * from 'table_name' WHERE id in (3,5) FOR UPDATE;`锁住(3, 5)区间。
2. 插入`id=4`，被阻塞。此时该事务生成一个插入意向锁，等待区间释放。（插入意向锁之间不会阻塞）

> 锁其实就是在内存里的一个结构。每个事务为记录或者区间上锁，即创建一个锁对象。如果某个事务没有抢到资源，也会生成一个锁对象，只是状态是等待中。等拥有资源的事务释放锁，就会寻找正在等待当前资源的锁对象，选择一个唤醒，获得资源的事务继续执行。

![lock_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/lock_intro_2.jpg)

![lock_3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/lock_intro_3.jpg)
