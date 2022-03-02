# 基于Redis实现MQ的研究

## 基于List实现
> List数据类型的底层结构是链表，在头尾操作元素，时间复杂度O(1)。利用List可以实现最简单的MQ场景

生产者使用LPUSH发送消息：
```text
127.0.0.1:6379> LPUSH queue msg1
(integer) 1
127.0.0.1:6379> LPUSH queue msg2
(integer) 2
```
消费者使用RPOP接收消息：
```text
127.0.0.1:6379> RPOP queue
“msg1"
127.0.0.1:6379> RPOP queue
"msg2"
```
业务模型：
![redis_mq_1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_mq_1.jpg)

但是这里存在一个问题，如果消息队列中没有消息，消费者执行`RPOP`直接返回`nil`
```text
127.0.0.1:6379> RPOP queue
(nil)
```
通常消费者的业务逻辑代码会写在一个死循环中，不断从消息队列拉取消息。此时消息队列为空，消费者不断尝试，造成`CPU空转`浪费资源。
```
// 伪代码示例
while(true) {
    msg = redis.rpop("queue");
    if (msg == null) {
        continue;    
    }
    process(msg);
}
```
想到通过sleep()让出CPU资源.
```
// 伪代码示例
while(true) {
    msg = redis.rpop("queue");
    if (msg == null) {
        sleep(2);
        continue;  
    }
    process(msg);
}
```
此时如果没有拉到消息，进行`休眠`。但是引发了另一个问题，休眠期间有生产者发送消息，无法及时处理，即消息延迟。
`休眠`时间越短，造成CPU空转的可能性越大。所幸Redis提供了`阻塞式`拉取消息的命令。如果消息队列为空，消费者就阻塞，一旦有新消息送达，再进行处理。

![redis_mq_2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_mq_2.jpg)

```
// 伪代码示例
while(true) {
    // 阻塞方式拉取，第二个参数指定超时时间，0为不设置超时
    msg = redis.brpop("queue", 0);
    if (msg == null) {
        continue;  
    }
    process(msg);
}
```
> *如果设置的超时时间过长，这个连接处于不活跃状态，可能会被Redis Server判定为无效连接强制剔除下线。故这种方法还要考虑重连接的机制。

> 到目前为止，以上业务模型解决了消息处理不及时的问题，仍有以下缺点：

- 消息的重复消费：消费者一旦从List拉取消息，就从List里删除了。其他消费者无法消费
- 消息丢失：消费者拉取消息后发生异常，这条消息就丢失了

## Pub/Sub模式
> Pub/Sub模式可以解决基于List实现方案的第一个缺点：支持重复消费。
>
> 如多对多的场景：

![redis_mq_3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_mq_3.jpg)

启动两个消费者客户端订阅消息队列（消费者会被阻塞）
```text
127.0.0.1:6379> SUBSCRIBE queue
Reading messages... (Press Ctrl-C to quit)
1) "subscribe"
2) "queue"
3) (integer) 1
```
启动生产者客户端发送消息
```text
127.0.0.1:6379> PUBLISH queue msg1
(integer) 1
```
消费者被唤醒
```text
127.0.0.1:6379> SUBSCRIBE queue
1) "message"
2) "queue"
3) "msg1"
```
> 可见通过Pub/Sub模式，既支持阻塞式拉取消息，又支持了多消费者消费同一批数据的业务场景。
> 
> 另外，Redis还提供了`匹配订阅`的模式，允许消费者`按一定规则`订阅`多个`消息队列
```text
127.0.0.1:6379> PSUBSCRIBE queue.*
Reading messages... (Press Ctrl-C to quit)
1) "subscribe"
2) "queue.*"
3) (integer) 1
```
这里，消费者订阅了`queue.*`的消息队列。由生产者分别向`queue.p1`、`queue.p2`发布消息。
![redis_mq_4](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_mq_4.jpg)

```text
127.0.0.1:6379> PUBLISH queue.p1 msg1
(integer) 1

127.0.0.1:6379> PUBLISH queue.p2 msg2
(integer) 1
```
回到消费者这边，接收到两个队列的消息。
```text
...
// 来自queue.p1的消息
1) "pmessage"
2) "queue.*"
3) "queue.p1"
4) "msg1"

// 来自queue.p2的消息
1) "pmessage"
2) "queue.*"
3) "queue.p2"
4) "msg2"
```
> Q：Pub/Sub模式还存在哪些问题？
> 
> A：在一些场景下存在消息丢失

1. 消费者下线
2. Redis宕机
3. 消息堆积

> Pub/Sub的实现非常简单，不做任何的数据存储，仅仅作为`通道`连接生产者、消费者。
>
> 1. 消费者订阅指定队列，Redis记录一个映射关系：队列->消费者
> 2. 生产者向指定队列发送消息，Redis从映射关系中找到消费者，将消息转发给它

![redis_mq_5](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_mq_5.jpg)

- 假如一个消费者发生异常下线，重新上线后再次进行订阅。只能接收到之后的新消息。下线期间的消息就丢失了。如果所有消费者下线，生产者发布的消息因为找不到任何一个消费者，全部丢弃
- 注意该模式下，一定要先让消费者订阅消息队列，生产者才能发布消息

> 当消费者消费消息的速度跟不上生产者发布消息的速度时，会出现`消息堆积`。
> 
> 如果是基于List的实现方式，相当于是往链表不断追加新元素节点，最终Redis占用的内存会不断增加。
> 
> Pub/Sub模式下，当消费者每订阅一个消息队列，Redis为消费者分配一个`缓冲区`（内存中的一块区域）。生产者发布消息后，Redis把消息写入对应消费者的缓冲区。消费者从缓冲区读取消息进行处理。

![redis_mq_6](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_mq_6.jpg)

> 这里的`缓冲区`的大小，是有上限可配置的。一旦写入超过上限，Redis会将消费者强制踢出下线。这样数据就丢失了。
>
> Redis配置文件中：client-output-buffer-limit pubsub 32mb 8mb 60
> 
> 32mb：缓冲区达到32MB，就把消费者踢下线
> 8mb 60：缓冲区达到8MB并且支持60秒，就把消费者踢下线

## Stream
> Redis作者在开发Redis期间，另外开源了一个项目`disque`。该项目定位是一个基于内存的分布式消息队列中间件。
> 
> 至`Redis5.0`版本，作者将`disque`移植到了Redis，并给它重新定义了一种新的数据类型`Stream`

生产者发布两条消息：
```text
// *表示让Redis生成唯一消息ID（格式：时间戳-自增序列号）
127.0.0.1:6379> XADD queue * name zhangsan
"1618469123380-0"
127.0.0.1:6379> XADD queue * name lisi
"1618469127777-0"
```
消费者拉取消息：
```text
// 从头开始拉取5条消息，0-0表示从头开始
127.0.0.1:6379> XREAD COUNT 5 STREAMS queue 0-0
1) 1) "queue"
   2) 1) 1) "1618469123380-0"
         2) 1) "name"
            2) "zhangsan"
      2) 1) "1618469127777-0"
         2) 1) "name"
            2) "lisi"
```
如果想继续拉取，需要传入上一条消息ID
```text
127.0.0.1:6379> XREAD COUNT 5 STREAMS queue 1618469127777-0
```
Stream业务模型：
![redis_mq_7](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_mq_7.jpg)

> Q：Stream如何支持`阻塞式`拉取消息？
>
> A：读取时候增加BLOCK参数
```text
127.0.0.1:6379> XREAD COUNT 5 BLOCK 0 STREAMS queue 0-0
```

> Q：Stream如何支持Pub/Sub？
>
> A：`XGROUP CREATE`：创建消费者组，`XREADGROUP GROUP`：在指定消费组下拉取消息

生产者发布2条消息：
```text
127.0.0.1:6379> XADD queue * name zhangsan
"1618469123380-0"
127.0.0.1:6379> XADD queue * name lisi
"1618469127777-0"
```
开启2个消费组订阅同一个消息队列：
```text
// 创建消费组1，订阅quque队列，从0-0开始拉取消息
127.0.0.1:6379> XGROUP CREATE queue group1 0-0

// 创建消费组2，订阅quque队列，从0-0开始拉取消息
127.0.0.1:6379> XGROUP CREATE queue group2 0-0
```
消费组下添加消费者：
```text
// group1的消费者开始消费，>表示拉取最新数据
127.0.0.1:6379> XREADGROUP GROUP group1 consumer COUNT 5 STREAMS queue >
1) 1) "queue"
   2) 1) 1) "1618469123380-0"
         2) 1) "name"
            2) "zhangsan"
      2) 1) "1618469127777-0"
         2) 1) "name"
            2) "lisi"

// group2的消费者开始消费，>表示拉取最新数据
127.0.0.1:6379> XREADGROUP GROUP group2 consumer COUNT 5 STREAMS queue >
1) 1) "queue"
   2) 1) 1) "1618469123380-0"
         2) 1) "name"
            2) "zhangsan"
      2) 1) "1618469127777-0"
         2) 1) "name"
            2) "lisi"
```
业务模型：
![redis_mq_8](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/middleware/redis/resource/redis_mq_8.jpg)

## 与MQ中间件比较