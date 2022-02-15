# Redis Introduction

## 1. 数据类型

### 1.1 基本类型
- String
- Hash
- List
- Set
- Zset
### 1.2 其他类型
- bitmap
- GeoHash
- HyperLogLog
- Streams

### 1.3 版本差异
![iamge_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_1.jpg)
> 在`Redis3.0`版本中，List的底层数据结构由`双向列表`或`压缩列表`实现，但是在`Redis3.2`版本之后则由`quicklist`实现。
> 
> 在最新的版本中，`压缩列表`已经废弃，转由`listpack`数据结构来实现。

### 1.4 KV数据库的实现
```text
> SET title "test title"
OK

> HSET student name "test" age 18
0

> RPUSH students "John" "Mike"
(integer) 4
```
- 第一条命令：title是`字符串键`，因为键的值是`字符串对象`
- 第二条命令：student是`哈希表键`，因为键的值是`包含两个键值对的哈希表对象`
- 第三条命令：students是`列表键`，因为键的值是`包含两个元素的列表对象`
> Redis中，通过一个`哈希表`来保存所有键值对，时间复杂度复杂度是O(1)。哈希表就是一个数组，数组里的元素叫`哈希桶`。
> 
> `哈希桶`存放指向键值对的指针。键值对的数据结构中并不直接存放数据，通过`void* key`和`void* value`指向实际的键对象以及值对象。
![iamge_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_2.jpg)
- redisDb：描述Redis数据库的结构
- dict：存放两个哈希表，正常情况只使用`dict1`，`dict2`在`rehash`时使用
- dictht：表示哈希表的结构，结构里存放了哈希表数组，数组中每个元素都是哈希键值对（dictEntry）
- dictEntry：键值对结构，key为`String`对象，value为`基本类型`对象

![iamge_3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_3.jpg)

## 2 数据结构

`RedisObject`
> RedisObject是Redis对内部存储数据的抽象定义
 
|类型|属性|描述|
|-|-|-|
|unsigned|type|表示对象的基本类型，如：String，Hash，List...|
|unsigned|encoding|编码格式，即存储数据使用的数据结构。同一个类型的数据，Redis会根据数据量、占用内存等情况使用不同的编码，最大限度地节省内存|
|unsigned|lru|对应内存淘汰策略，24位，LRU时间戳或LFU计数|
|int|refcount|引用计数，为了节省内存，Redis会在多处引用同一个redisObject|
|void|*ptr|指向底层实际的数据结构的指针，如：SDS|

完整的结构如下：
![iamge_4](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_4.jpg)

|数据类型|说明|编码|底层数据结构|
|-|-|-|-|
|String|字符串|OBJ_ENCODING_INT|long long/long|
|String|字符串|OBJ_ENCODING_EMBSTR|SDS|
|String|字符串|OBJ_ENCODING_RAW|SDS|
|-|-|-|
|list|列表|OBJ_ENCODING_QUICKLIST|quicklist
|hash|散列|OBJ_ENCODING_HT|dict|
|hash|散列|OBJ_ENCODING_LISTPACK|listpack|
|set|集合|OBJ_ENCODING_HT|dict|
|set|集合|OBJ_ENCODING_INTSET|intset|
|zset|有序集合|OBJ_ENCODING_LISTPACK|listpack|
|zset|有序集合|OBJ_ENCODING_SKIPLIST|skiplist|

- OBJ_ENCODING_EMBSTR：长度小于或等于OBJ_ENCODING_EMBSTR_SIZE_LIMIT（44字节）的字符串。是Redis针对短字符串的优化
    1. 内存申请和释放都只需要调用一次内存操作函数。
    2. redisObject、sdshdr结构保存在一块连续的内存中，减少了内存碎片。
- OBJ_ENCODING_RAW：长度大于OBJ_ENCODING_EMBSTR_SIZE_LIMIT（44字节）的字符。在该编码中，redisObject、SDS结构存放在两个不连续的内存块中
- OBJ_ENCODING_INT：将数值型字符串转换为整型，可以大幅降低数据占用的内存空间，如字符串“123456789012”需要占用12字节，在Redis中，会将它转化为long long类型，只占用8字节

### 2.1 SDS
> Simple Dynamic String，Redis自行封装的一个结构。C语言字符串其实就是字符数组，其通过`\0`来填充数组末尾，表示字符串的结束。因此，除了结尾位置，字符串中不能含有`\0`，使得C语言的字符串只能保存文本数据，不能保存图片、音频、视频等二进制数据。并且在C语言中获取字符串长度，需要遍历字符数组，找到`\0`以后才能返回统计到字符个数。

SDS结构如下：
![sds_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_sds_1.jpg)
- lens：记录字符串长度。这样获取字符串长度的时间复杂度就做到了O(1)
- alloc：分配给字符串的总空间长度，（alloc-lens）即为空闲空间。当修改字符串时，计算剩余空间是否够用。不够用自动将空间扩展至执行修改所需的大小，再执行修改操作
- flags：sds类型。低3位代表sdshdr的类型，高5位只在sdshdr5中使用，表示字符串的长度，所以sdshdr5中没有len属性。另外，由于Redis对sdshdr5的定义是常量字符串，不支持扩容，所以不存在alloc属性
- buf[]：字符数组。保存实际的数据，不仅可以保存文本，也支持二进制数据
> sds结构中的`flags`对应5种类型：sdshhdr5，sdshdr8，sdshdr16，sdshdr32以及sdshdr64，主要区别是数据结构中的len和alloc的数据类型不同

示例：
```text
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len;
    uint16_t alloc;
    unsigned char flags;
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len;
    uint32_t alloc;
    unsigned char flags;
    char buf[];
};
```
- sdshdr16的len和alloc的数据类型是`uint16_t`，表示字符数字的长度和分配空间大小不能超过2的16次方
- sdshdr32的len和alloc的数据类型是`uint32_t`，表示字符数字的长度和分配空间大小不能超过2的32次方

> 这样设计的目的，是为了灵活保存不同大小的字符串，从而节省空间。
> 
> 此外，Redis还使用了专门的编译优化来节省内存空间，在struct中声明了`__attribute__ ((__packed__))`，告诉编译器取消结构体在编译过程中的优化对齐，按照实际占用字节数进行对齐。

示例：
> 默认情况下，编译器使用`字节对齐`方式分配内存。虽然`char`占用`1`字节，但是`int`占用`4`字节，所以为`char`的变量分配内存时分配了`4`字节，多余的`3`字节即为对齐而分配，存在空间浪费。
```
#include <stdio.h>

struct test {
    char a;
    int b;
} test;

int main() {
    printf("%lu\n", sizeof(test));
    return 0;
}
```
![c_struct_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_c_struct_1.jpg)
> struct声明时加上`__attribute__ ((__packed__))`
```
#include <stdio.h>

struct __attribute__ ((__packed__)) test {
    char a;
    int b;
} test;

int main() {
    printf("%lu\n", sizeof(test));
    return 0;
}
```
![c_struct_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_c_struct_2.jpg)

### 2.2 linked list
> 底层双向链表结构
```
typedef struct listNode {
  // prev node
  struct listNode *preve;
  // next node
  struct listNode *next;
  // value
  void *value
} listNode;
```

![listnode_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_listnode_1.jpg)

> redis在`listNode`基础上又封装了一层`list`结构，包含了头尾节点，节点数量等属性以及dup，free，match函数。
```
typedef struct list {
    // head node
    listNode *head;
    // tail node
    listNode *tail;
    // dup function
    void *(*dup)(void *ptr);
    // free function
    void (*free)(void *ptr);
    // match function
    int (*match)(void *ptr, void *key);
    // node count
    unsigned long len;
} list;
```
![listnode_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_listnode_2.jpg)

> 链表的优点
- listNode结构包含`prev`，`next`指针指向当前节点的前后节点，即获取某个节点的前后节点的时间复杂度`O(1)`
- list结构包含`head`，`tail`指针指向列表的首尾节点，即获取整个列表的首尾节点的时间复杂度`O(1)`
- list结构包含`len`属性，记录列表元素数量，时间复杂度`O(1)`
- listNode使用指针`value`保存不同类型的值

> 链表的缺点
- 链表每个节点之间并不连续，无法很好的利用CPU缓存（不同于数组方式，数组方式是在内存上连续分配的一段空间）
- 保存一个链表节点的值，都需要一个链表结构头的分配，内存开销更大

> `Redis3.0`的List，当包含元素比较少时，采用`压缩列表`作为底层数据结构的实现，其优点是更节省内存空间。不过`压缩列表`也有其他性能问题。在`Redis3.2`后，List采用了新设计的`quicklist`作为底层数据结构的实现。
> `Redis5.0`设计了新的`listpack`数据结构，沿用了紧凑型的内存布局，替代`压缩列表`。
### 2.3 zip list
> `压缩列表`被设计成一种内存紧凑型的数据结构，占用一块连续的内存空间，不仅可以利用 CPU 缓存，而且会针对不同长度的数据，进行相应编码，这种方法可以有效地节省内存开销。

> 压缩列表的缺点
- 不能保存过多的元素，否则查询效率就会降低
- 新增或修改某个元素时，压缩列表占用的内存空间需要重新分配，甚至可能引发连锁更新的问题

压缩列表的结构：
![ziplist_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_ziplist_1.jpg)
- zlbytes：记录整个列表的占用的内存字节数
- zltail：记录列表尾的自起始地址的偏移量
- zllen：记录列表节点数量
- zlend：标记压缩列表的结束点，固定值`0xFF`（十进制：255）

> 在压缩列表中，获取首尾节点的时间复杂度O(1)，查找其他元素需要遍历，时间复杂度O(N)

压缩列表entry的结构：
![ziplist_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_ziplist_2.jpg)
- prevlen：前一个节点的长度
- encoding：当前节点的实际数据类型和长度
- data：当前节点的实际数据

> 连锁更新问题
> 
> 压缩列表新增某个元素或修改某个元素时，如果空间不不够，压缩列表占用的内存空间就需要重新分配。而当新插入的元素较大时，可能会导致后续元素的`prevlen`占用空间都发生变化，从而引起`连锁更新`问题，导致每个元素的空间都要重新分配，造成访问压缩列表性能的下降。

压缩列表节点的 prevlen 属性会根据前一个节点的长度进行不同的空间大小分配：
- 如果前一个节点的长度`<254`字节，那么`prevlen`属性需要用`1`字节的空间来保存这个长度值；
- 如果前一个节点的长度`>=254`字节，那么`prevlen`属性需要用`5`字节的空间来保存这个长度值；

现在假设一个压缩列表中有多个连续的、长度在 250～253 之间的节点，如下图：
![ziplist_3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_ziplist_3.jpg)
因为这些节点长度值`<254`字节，所以`prevlen`属性需要用`1`字节的空间来保存这个长度值。
这时，如果将一个长度`>=254`字节的新节点加入到压缩列表的表头节点，即新节点将成为`e1`的前置节点，如下图：
![ziplist_4](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_ziplist_4.jpg)
因为`e1`节点的`prevlen`属性只有`1`个字节大小，无法保存新节点的长度，此时就需要对压缩列表的空间重分配操作，并将 e1 节点的`prevlen`属性从原来的`1`字节大小扩展为`5`字节大小。
![ziplist_5](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_ziplist_5.jpg)
这种在特殊情况下产生的连续多次空间扩展操作就叫做`连锁更新`，就像多米诺牌的效应一样，第一张牌倒下了，推动了第二张牌倒下；第二张牌倒下，又推动了第三张牌倒下
### 2.4 hash table
> 哈希表是一种保存键值对（key-value）的数据结构。
> 
> 哈希表中每一个key都是唯一的，程序根据key访问、更新或者删除value。
> 
> Redis的Hash对象底层实现之一是`压缩列表（ziplist）`（在后续版本被替换成`listpack`），另一种底层实现即`哈希表`。
> 
> Redis采用了`拉链法`解决`哈希冲突`的问题。

哈希表的结构
```
typedef struct dictht {
  // 哈希表数组
  dictEntry **table;
  // 哈希表大小
  unsigned long size;
  // 哈希表大小掩码，用于计算索引值
  unsigned long sizemask;
  // 哈希表已有节点数量
  unsigned long used;
} dictht;
```
哈希表节点的结构
```
typedef struct dictEntry {
  // 键
  void *key;
  // 值
  union {
    void *val;
    uint64_t u64;
    int64_t s64;
    double d;
  } v;
  // 下一个节点
  struct dictEntry *next;
} ditEntry;
```
![hashtable_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_hashtable_1.jpg)

- dictht：table指针指向一个数组，即哈希桶。每一个元素指向一个节点dictEntry
- dictEntry：不仅包含指向键和值的指针，还有一个next指针指向下一个节点dictEntry，即拉链法解决哈希冲突
- union：dictEntry的值可以是一个指向实际值的指针val，也可以是一个无符号64位整数、有符号64位整数或者双精度浮点数。这样做也是为了节省空间，如果是满足条件的数值，可以直接内嵌在dictEntry结构里，无需再用一个额外指针指向实际的值

> 哈希冲突

当key通过hash函数计算后得到哈希值，再将哈希值按哈希表大小取模填充到哈希桶的过程可能发生不同的key映射到哈希桶相同的位置上

> 拉链法解决哈希冲突

dictEntry通过next指针指向下一个节点
![hashtable_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_hashtable_2.jpg)
随着链表长度的增加，查询的时间复杂度会趋向于O(N)

> rehash

在1.4部分提到了Redis定义了两张哈希表，在进行rehash时，就需要用到第二张哈希表
![hashtable_3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_hashtable_3.jpg)

- 正常服务请求阶段，插入的数据都写`哈希表1`，此时`哈希表2`没有分配空间
- 随着数据量的增加，触发rehash
 1. 为`哈希表2`分配空间，一般比`哈希表1`大两倍
 2. 将`哈希表1`的数据迁移到`哈希表2`中
 3. 迁移完成，释放`哈希表1`占用的空间，将`哈希表2`设置成`哈希表1`，然后新建一个哈希表作为新的`哈希表2`，为下次rehash作准备
- 为了避免迁移过程中拷贝数据的开销对Redis服务性能的损耗，采用了`渐进式rehash`，即分次迁移
 1. 为`哈希表2`分配空间
 2. 在rehash期间，每次执行新增、删除或者更新操作，Redis还会将`哈希表1`上的键值对迁移到`哈希表2`
 3. 随着处理客户端操作请求的增加，最终将`哈希表1`上所有键值对迁移到`哈希表2`上

以上过程中，所有操作都会在两个哈希表上进行。例如，查找某个key，先在`哈希表1`上找，如果没找到，继续在`哈希表2`上找。
另外，在渐进式rehash期间，新增一个key-value，会直接保存到`哈希表2`中，`哈希表1`则不再进行添加操作。这样保证了`哈希表1`的key-value数量只会减少。随着rehash操作完成，最终`哈希表1`变成空表。

> rehash触发条件：负载因子=哈希表已保存节点数量/哈希表大小
- 当负载因子>=1，而且Redis没有执行`bgsave`命令或者`bgrewriteaof`命令（即没有执行RDB快照或者AOF重写），就会进行rehash
- 当负载因子>=5，此时哈希冲突非常严重，强制执行rehash

### 2.5 int set
> 整数集合是Set类型底层数据结构实现之一。当一个Set对象只包含`整数值`元素，并且元素`数量`不多时。就会使用整数集合来实现。

整数集合的结构
```
typedef struct intset {
  // encoding
  uint32_t encoding;
  // length
  uint32_t length;
  // element array
  int8_t contents[];
} intset;
```
保存元素的容器是一个`contents`数组，虽然声明为`int8_t`类型，但是元素的真正类型取决于`encoding`。
- encoding=`INTSET_ENC_INT16`：contents是`int16_t`类型的数组
- encoding=`INTSET_ENC_INT32`：contents是`int32_t`类型的数组
- encoding=`INTSET_ENC_INT64`：contents是`int64_t`类型的数组
不同类型的contents数组，占用的空间大小也不同。
> 整数集合的升级
- 当向一个`int16_t`类型的整数集合，加入一个`int32_t`类型的新元素时，需要扩展contents数组的空间大小后再添加新元素。升级过程中也要维持整数集合的有序性
- 整数集合升级不会重新分配一个新类型的数组，而是在原数组上扩展空间，然后再将每个元素按间隔类型大小分隔。如encoding=`INTSET_ENC_INT16`，每个元素间隔即为16位

例如：
1. 一个包含3个类型为`int16_t`元素的整数集合

![intset_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_intset_1.jpg)

2. 现在，往整数集合添加一个新元素`65535`，这个元素需要`int32_t`类型来保存，故需要先将contents数组扩容。在原来空间的基础上，再扩容80位（4 * 32 - 3 * 16 = 80），这样可以保存下4个`int32_t`类型的元素。

![intset_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_intset_2.jpg)

3. contents扩容后，将之前的3个元素转换为`int32_t`类型，并且维持元素的顺序不变。最后添加新元素

![intset_3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_intset_3.jpg)
> 通过这种动态`扩容`的机制，避免了内存空间的浪费。
> 
> 整数集合只支持升级，不支持降级。即删除`65535`这个元素，整数集合的数组还是`int32_t`类型。
### 2.6 skip list
> Zset类型的底层数据结构同时包含两种实现，`哈希表`+`跳表`，平均时间复杂度为O(logN)。利用`跳表`支持高效的范围查询（ZRANGEBYSCORE），通过`哈希表`索引，能够在常数时间复杂度获取到元素节点的SCORE。

Zset的结构：
```
typedef struct zset {
  dict *dict;
  zskiplist *zsl;
}} zset;
```
> 跳表的设计
> 
> 跳表是在链表（时间复杂度O(N)）的基础上改进得到的，是一种`多层`链表

![skiplist_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_skiplist_1.jpg)

- `L0`层共`5`个节点，分别是1，2，3，4，5
- `L1`层共`2`个节点，分别是2，3，5
- `L2`层共`1`个节点，即3

如果是以链表结构存储，先找到4节点需要从从头开始遍历，共查找4次。而使用跳表以后，可以由`l2`层跳至`3`节点，然后找到`4`。
这个过程就是在多个层级上跳跃，最后找到目标元素。当数据量很大的时候，查询的时间复杂度为O(logN)。

跳表结构：
```
typedef struct zskiplist {
  // head & tail
  struct ZskiplistNode *head, *tail;
  // length
  unsigned long length;
  // level
  int level;
} zskiplist;
```
- head&tail：指向跳表的头尾节点，可以在O(1)时间复杂度下访问
- length：跳表的长度，可以在O(1)时间复杂度下获取跳表包含的元素数量
- level：最大层数

- 跳表节点的结构：
```text
typedef struct zskiplistNode {
    // 元素值
    sds ele;
    // 权重
    double score;
    // 前一个元素节点
    struct zskiplistNode *backward;
    // 节点level数组，保存每层的下一个节点和跨度
    struct zskiplistLevel {
        // 下一个节点
        struct zskiplistNode *forward;
        // 跨度
        unsigned long span;
    } level[];
} zskiplistNode;
```
- 跳表中的节点通过`ele`变量保存元素实际数据，`score`保存元素权重
- `backward`指向前一个节点，这样做是为了方便从尾节点开始倒序访问
- 跳表包含了层级结构，每一个层级包含多个节点以指针连接，`zskiplistLevel`
- level数组中每`一个元素`代表跳表的`一层`，level[0]表示第一层，level[1]表示第二层

![skiplist_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_skiplist_2.jpg)

> 跳表的查询过程

- 从跳表头节点最高层开始，逐一遍历每一层
  1. 如果当前节点`score<目标权重`，访问`该层下一节点`
  2. 如果当前节点`score=目标权重 && ele<目标数据`，访问`该层下一节点`
- 如果以上条件不满足，或者下一节点为`null`，则访问当前节点`level`数组里的下一层节点指针

- ![skiplist_3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_skiplist_3.jpg)

例如：查找目标元素（ele：abcd，score：4）

- 从头节点最高层开始，`l2`指向元素节点（ele：abc，score：3）。节点`score`比目标score小，则访问`l2`层下一节点`null`
- 由于下一节点为`null`，则访问当前节点level数组下一层`level[1]`的节点指针
- 元素节点`（ele：abc，score：3）`的`level[1]`指向`（ele：abcde，score：4）`，此时`score`与目标score一致，当`ele`大于目标ele
- 继续跳到`（ele：abc，score：3）`的`level[0]`指向`（ele：abcd，score：4）`，该节点正是目标节点，查询结束

> 跳表的层数设置
> 
> 相邻两层的节点数量比例，直接影响整个跳表的性能

反例：

![skiplist_4](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_skiplist_4.jpg)

`l1`层节点只有1个，`l0`层有6个。想要找到节点6，基本就是在`l0`层遍历链表

正例：

![skiplist_5](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_skiplist_5.jpg)

> 跳表相邻两层节点数比例为2：1，访问时间复杂度可以降低到O(logN)
> 
> 如果每次新增、删除节点时调整相邻层的节点数比例，开销较大。Redis采用了更为巧妙的方法，创建节点时，生成一个`[0，1]`范围上的随机数，如果随机数`小于0.25`（25%的概率），则层数`+1`，然后继续生成下一个随机数，重复以上过程直至结果`>0.25`结束。层数越高概率越低，层高最大限制`64`。

### 2.7 quick list
> `Redis3.0`之前，List类型的底层数据结构又`双向链表`和`压缩列表`实现。`Redis3.2`后，改为`quicklist`实现。
> 
> `quicklist`就是`双向链表`+`压缩列表`的组合，因为`quicklist`本身是一个`链表`，其内部每一个元素又是一个`压缩列表`。
> 
> `压缩列表`通过紧凑型的布局节省了内存开销，但是随着元素数量增多或者元素变大，可能引发`连锁更新`风险，造成性能下降。故`quicklist`通过控制每个元素节点中`压缩列表`的大小或者数量，来避免`连锁更新`。

quicklist的结构：
```
typedef struct quicklist {
  // head
  struct quicklistNode *head;
  // tail
  struct quicklistNode *tail;
  // length
  unsigned long len;
  ...
} quicklist;
```
quicklistNode的结构：
```
typedef struct quicklistNode {
  // prev node
  struct quicklistNode *prev;
  // next node
  struct quicklistNode *next;
  // point to zip list
  unsigned char *zl;
  // zip list byte size
  unsigned int sz;
  // zip list count
  unsigned int count;
}
```
> 可以看到`quicklistNode`结构体里包含了`prev`和`next`指针，形成了`双向链表`。每个元素节点不再单纯保存元素值，而是指向`zp`（压缩列表）。
![quicklist_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_quicklist_1.jpg)
- 每次添加一个元素，不会像普通链表那样直接新增节点。而是`先检查目标位置的压缩列表是否能容纳新元素，优先保存到压缩列表中。如果不行再新增一个quicklistNode节点加入链表

### 2.8 list pack