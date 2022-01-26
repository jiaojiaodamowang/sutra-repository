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
![iamge1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_1.jpg)
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
![iamge2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_2.jpg)
- redisDb：描述Redis数据库的结构
- dict：存放两个哈希表，正常情况只使用`dict1`，`dict2`在`rehash`时使用
- dictht：表示哈希表的结构，结构里存放了哈希表数组，数组中每个元素都是哈希键值对（dictEntry）
- dictEntry：键值对结构，key为`String`对象，value为`基本类型`对象

![iamge3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_3.jpg)
> 注意：实际的数据结构封装在RedisObject结构里
- type：表示对象的基本类型，如：String，Hash，List...
- encoding：表示对象的底层数据结构
- ptr：指向底层数据结构的指针

完整的结构如下：
![iamge4](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_intro_4.jpg)

## 2 数据结构

### 2.1 SDS
> Simple Dynamic String，Redis自行封装的一个结构。C语言字符串其实就是字符数组，其通过`\0`来填充数组末尾，表示字符串的结束。因此，除了结尾位置，字符串中不能含有`\0`，使得C语言的字符串只能保存文本数据，不能保存图片、音频、视频等二进制数据。并且在C语言中获取字符串长度，需要遍历字符数组，找到`\0`以后才能返回统计到字符个数。

SDS结构如下：
![sds1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/sds_1.jpg)
- lens：记录字符串长度。这样获取字符串长度的时间复杂度就做到了O(1)
- alloc：分配给字符串的空间长度。当修改字符串时，计算剩余空间是否够用。不够用自动将空间扩展至执行修改所需的大小，在执行修改操作
- flags：sds类型
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
![c_struct_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/c_struct_1.jpg)
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
![c_struct_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/c_struct_2.jpg)

### 2.2 linked list
### 2.3 zip list
### 2.4 hash table
### 2.5 int set
### 2.6 skip list
### 2.7 quick list
### 2.8 list pack