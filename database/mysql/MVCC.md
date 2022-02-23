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
- creator_trx_id：当前事务ID
- m_ids：生成`readview`时还活跃的事务ID（已开启未提交）
- min_trx_id：活跃事务ID中最小值
- max_trx_id：生成`readview`时InnoDB将分配给下一个新事务的ID

> 可见性规则
 
1. 如果当前数据版本`trx_id = creator_trx_id`，说明修改这条数据的事务就是当前事务，数据可见。
2. 如果当前数据版本`trx_id < min_trx_id`，说明修改这条数据的事务在当前事务生成`readview`前已提交，数据可见。
3. 如果当前数据版本`trx_id in min_trx_id`，说明修改这条数据的事务在当前事务生成`readview`时未提交，数据不可见。
4. 如果当前数据版本`trx_id > max_trx_id`，说明修改这条数据的事务在当前事务生成`readview`时未启动，数据不可见。

### 示例
> 当前事务隔离级别：read-committed
>
> `事务1已经提交，事务5已执行未提交`，此时开启`新事务6`执行`update YY where id = 2，未提交`。再开启一个事务执行查询操作`select name where id = 1`

- creator_trx_id：0【事务有修改操作时才分配事务ID】
- m_ids：[5, 6]【两个事务都未提交，处于活跃状态】
- min_trx_id：5【m_ids最小值】
- max_trx_id：7【当前已经分配到6，待分配的下一个事务ID】

> 现在查询`id=1`的记录：

![mvcc_6](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/mvcc_intro_6.jpg)
>`trx_id=5`，处于`m_ids`中，是由未提交的事务修改，不可见。通过`roll_pointer`找到上一版本。

> 通过指针在`undolog`上找到`trx_id=1`的版本，符合`trx_id < min_ids`。即当前事务生成readview时，该版本数据已经提交，故可见。此时select语句返回结果name=xx。

> 然后事务5提交，再次查询`select name where id = 1`，这次生成`新的readview`。

- creator_trx_id：0【事务还是没有修改操作】
- m_ids：[6]【事务5已提交，事务6处于活跃状态】
- min_trx_id：6【m_ids最小值】
- max_trx_id：7【当前已经分配到6，待分配的下一个事务ID】

> 还是查询`id=1`的记录：

![mvcc_6](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/mvcc_intro_6.jpg)
> 此时最新版本的记录上的`trx_id=5`不变，而`trx_id < min_trx_id`，说明是已提交事务，数据可见。因此这一次select语句返回的name=NO。

> 此时的事务隔离级别：`read-committed`，同一个事务中两次查询同一条记录，得到了不同的结果，就是`不可重复读`。

> 为了解决`不可重复读`问题，需要调整事务隔离级别为`repeatable-read`

> MVCC的可见性规则不变，唯一不同的地方在于，`read-committed`在每次select时生成一个`readview`,而`repeatable-read`只在`第一次`select时生成`readview`，保证了两次查询读到相同的结果。

