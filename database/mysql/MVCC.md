# MVCC Introduction

## 定义
> MVCC（Multi-Version Concurrency Control），是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问。具体是指一条记录存在多个版本，每次修改同一条数据，需要记录修改之前的版本，多个版本之间串联起来形成版本链
> 
> 不同时刻开启的事务可以`无锁`获取不同版本的数据，此时读写操作不会阻塞，提高了数据库的并发性能。

![mvcc_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/mvcc_intro_1.jpg)

## 事务的隔离级别
1. read-uncommitted
2. read-committed
3. repeatable-read
4. serializable

## MVCC的具体实现
### 版本链
> InnoDB并不真的存储多个版本的数据，而是借助`undolog`记录每次操作的反向操作。因此索引上对应的记录只会有一个版本，即最新版本。只不过可以通过`undolog`中的记录反向操作从而得到数据的历史版本，所以看起来是多个版本。
![mvcc_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/mvcc_intro_2.jpg)

> 以insert(1, "xx")语句为例，成功执行后，产生一条记录。除了`id`、`name`字段，还包括`trx_id`、`roll_pointer`隐藏字段
- trx_id：当前事务ID
- roll_pointer：指向`undolog`的指针
![mvcc_3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/mvcc_intro_3.jpg)
得知此时插入数据的事务ID是`1`，`roll_pointer`指向一条undolog。这条undolog的类型是`TRX_UNDO_INSERT_REC`，代表是insert操作生成的，其中记录了主键长度和值。

InnoDB可以根据`undolog`中主键的值，找到对应记录，然后执行删除操作来实现回滚。

> 事务1提交后，另外一个`事务ID=5`的事务开启，执行`update NO where id = 1`的语句，此时这条记录发生如下变化
![mvcc_4](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/mvcc_intro_4.jpg)
之前`insert`生成的`undolog`没有了。`update`操作产生的`undolog`类型为`TRX_UNDO_UPD_EXIST_REC`，记录了上一个版本的事务ID、回滚指针以及旧值。

> 事务5提交，另外再有一个`事务ID=11`的事务开启，执行`update YES where id = 1`的语句，此这条记录继续变化
![mvcc_5](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/mvcc_intro_5.jpg)
`update`操作生产的`undolog`不会马上删除，可能会有其他事务需要访问之前的版本。此时版本链就产生了。记录本身加上两条`undolog`共记录了3个版本。

### 版本可见性 