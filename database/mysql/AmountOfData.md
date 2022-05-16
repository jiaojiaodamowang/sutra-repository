# MySQL单表最多存2KW条数据？

## 前言

> 网络上时常看到帖子说数据库`单表`建议最大存2KW条数据的说法。如果超过这个数据量，性能会下降。那么这个指标是如何计算出来的？

## 实验

建表
```text
CREATE TABLE `t_user` (
    `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT `主键`,
    `name` varchar(100) NOT NULL DEFAULT `` COMMENT `名字`,
    `age` int(11) NOT NULL DEFAULT 0 COMMENT `年龄`,
    PRIMARY KEY ('id'),
    KET 'index_age' ('age')
) ENGINE=InnoDB DEFAULT CHARSETS='utf8mb4'
```
其中`id`作为主键可以限制表中数据行的上限：
- 比如声明为`int`，即32位，最大可以支持2^32-1，约`21亿`
- 如果是`bigint`，那就是2^64-1，是个极大的数字
- 如果声明为`tinyint`，一个字节，最大2^8-1,也就是`255`

此时插入一条`id=256`的数据，就会报错
```text
mysql> INSERT INTO `t_user` (`id`, `name`, `age`) VALUES (256, 'Hello', 20);
ERROR 1264 (22003): Out of range value for column `id` at row 1 
```

## 索引的结构

### 数据页

InnoDB内部的索引采用B+树的结构，此处略过。假设现在有一张`user`表如下：

|id|name|age|
|1|Sunday|1|
|2|Monday|2|
|3|Tuesday|3|
|4|Wednesday|4|
|5|Thursday|5|
|6|Friday|6|
|7|Saturday|7|

> 上面的数据表，在硬盘上，放在`user.ibd`文件中，意为`user`表的`innodbdata`文件。专业称为`表空间`。
> 虽然在数据表里看起来这些数据行是挨在一起的，实际上在`user.ibd`中它们被划分很多小的`数据页`

![page_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/innodb_page_1.jpg)

- 每个`数据页`的默认大小为`16K`，肯定放不下所有数据行。因此数据行们分成好多份存放进数据页。为了知道具体是哪一页，就需要标识`页号`（其实是一个表空间的地址偏移量）。同时为了将这些数据页关联起来，引入了`指针`。这些信息都存在`页头`里。
- 数据页需要被读写，为了保证数据页的正确性，还引入了`校验码`，这部分信息存在`页尾`。
- 最后剩下的空间，才用来存放数据行。为了提高查询效率，为这些数据生成了一个`页目录`，得以`二分查找`的方式进行查找

![page_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/innodb_page_2.jpg)

### 从数据页到索引

> 如果要查一条数据记录，可以把表空间里的每一页捞出来，再把里面的数据行逐行捞出来进行判断。显然这样的方式效率低下。
> 
> 为了加速搜索，可以将每个数据页里选出`主键id`最小的数据行。将它的`主键id`和所在页的`页号组成新的数据行放到数据页中，这个新的数据页与之前的页结构没有区别，大小还是16K。
> 
> 为了与之前的数据页进行区分，这种索引数据页加入了`页层级（page level）`的信息，从0开始往上算。于是页与页之间有了`上下层级`的概念。

![page_3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/innodb_page_3.jpg)

此时，页与页之间组成了`B+`树索引。

最下面一层数据页，`page level = 0`，即所谓的`叶子节点`，其余都是`非叶子节点`。

上图展示的是`两层`的树，随着数据变多，通过相同的方式，可以构建出`三层`树。

![page_4](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/innodb_page_4.jpg)

> 比如现在要查找数据记录id=5。先从顶层页的数据行入手。最小id=1，页号=6，最小id=7，页号=20。于是顺着找到`数据页6`。再判断id=5>4，找到`数据页105`。在该数据页中找到id=5的数据行。

![page_5](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/innodb_page_5.jpg)

另外需要注意的是，上面的页号并不是连续的，它们在磁盘中也不一定是挨在一起的。

这个过程中查询了3个页，如果3个页都在磁盘上（没有被加载到内存），那么需要经历`3次磁盘IO`，才会被加载到内存。

### B+树索引数据量

> 从上面的B+树索引结构可以看到，`叶子节点`存放了数据，`非叶子节点`用来存放查询数据的索引。

- `非叶子节点`指向其他页的指针数量为`X`
- `叶子节点`能容纳的数据量为`Y`
- B+树高度为`Z`

> 可得B+树的数据总量 `total = ( X ^ (Z-1) ) * Y`。
![page_6](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/innodb_page_6.jpg)

#### 估算X(非叶子节点)

![page_7](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/innodb_page_7.jpg)

- 假设现在主键`bigint(8 Bytes)`，页号在源码里定义为`FIL_PAGE_OFFSET(4 Bytes)`。那么一条非叶子节点的数据共占`12 Bytes`。
- 整个数据页大小为`16K Bytes`，页头页尾大概占用`128 Bytes`，页目录大概占用`1K Bytes`。即剩下`15K Bytes`除以`12 Bytes`，`X≈1280`，也就是可以指向约1280页。
- 对M叉树来说，一个节点指向M个新节点，称为`扇出（Fanout）`。上述的B+树节点可以指向1280个新节点，可以说扇出非常高。

#### 估算Y(叶子节点)

- 假设一行真正的数据占`1K Bytes`，可得一页存放15条数据，即`Y=15`。

#### 估算总数据量

> 回到`total=(X ^ (Z - 1)) * Y`这个公式。已知`X=1280`,`Y=15`。
- 对于一棵高度为2的B+树，`Z=2`，total：(1280 ^ (2 - 1)) * 15 = 19,200
- 对于一棵高度为3的B+树，`Z=3`，total：(1280 ^ (3 - 1)) * 15 = 24,567,000

> 这就是`单表2500万条数据量`的由来。再加一层，数据量就大得多。

### B树的数据量

> `B树`同`B+树`很像，数据页的结构也是一样的。区别在于，`B+树`的数据都存在叶子节点，而`B树`不作区分。

![page_8](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/database/mysql/resource/innodb_page_8.jpg)

> `B树`的数据页大小还是`16K`，扣除页头页尾还剩`15K`，同样一行数据占`1K`。即使不考虑各种指针，也只能存放`15`条数据。数据页扇出远远小于`B+树`。
> 计算总数据量的公式变成一个`等比数列`：15 + 15^2 + 15^3 + ... + 15^Z。

- 假设要存放2000万左右的数据，需要`Z >= 6`。最坏情况下需要`6次磁盘IO`才能查到数据。

> 同样的数据量下，`B+树`的层数更低，自然磁盘IO的次数也更低。