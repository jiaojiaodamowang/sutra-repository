# GC日志输出配置

## 打印基本信息

> 开启GC日志打印的基本参数
```text
-XX:+PrintGCDetails
-XX:+PrintGCDataStamps
```

> 打印分布对象
```text
-XX:+PrintTenuringDistribution
```
输出内容示例：
```text
Desired survivor size 59244544 bytes, new threshold 15 (max 15)
- age   1:     963176 bytes,     963176 total
- age   2:     791264 bytes,    1754440 total
- age   3:     210960 bytes,    1965400 total
- age   4:     167672 bytes,    2133072 total
- age   5:     172496 bytes,    2305568 total
- age   6:     107960 bytes,    2413528 total
- age   7:     205440 bytes,    2618968 total
- age   8:     185144 bytes,    2804112 total
- age   9:     195240 bytes,    2999352 total
- age  10:     169080 bytes,    3168432 total
- age  11:     114664 bytes,    3283096 total
- age  12:     168880 bytes,    3451976 total
- age  13:     167272 bytes,    3619248 total
- age  14:     387808 bytes,    4007056 total
- age  15:     168992 bytes,    4176048 total
```

> GC后打印堆数据
```text
-XX:+PrintHeapAtGC
```
输出内容示例：
```text
{Heap before GC invocations=0 (full 0):
 garbage-first heap   total 1024000K, used 324609K [0x0000000781800000, 0x0000000781901f40, 0x00000007c0000000)
  region size 1024K, 6 young (6144K), 0 survivors (0K)
 Metaspace       used 3420K, capacity 4500K, committed 4864K, reserved 1056768K
  class space    used 371K, capacity 388K, committed 512K, reserved 1048576K
Heap after GC invocations=1 (full 1):
 garbage-first heap   total 1024000K, used 21755K [0x0000000781800000, 0x0000000781901f40, 0x00000007c0000000)
  region size 1024K, 0 young (0K), 0 survivors (0K)
 Metaspace       used 3420K, capacity 4500K, committed 4864K, reserved 1056768K
  class space    used 371K, capacity 388K, committed 512K, reserved 1048576K
}
```

> 打印`STW`时间
```text
-XX:+PrintGCApplicationStoppedTime
```
输出内容示例：
```text
Total time for which application threads were stopped: 0.0254260 seconds, Stopping threads took: 0.0000218 seconds
```

> 打印`safe point`信息

进入STW阶段之前，需要要找到一个合适的`safe point`，这个指标一样很重要（非必选，出现 GC 问题时最好加上此参数调试）
```text
-XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1
```
输出内容示例：
```text
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
0.371: ParallelGCFailedAllocation       [      10          0              0    ]      [     0     0     0     0     7    ]  0   
Execute full gc...dataList has been promoted to cms old space
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
0.379: ParallelGCSystemGC               [      10          0              0    ]      [     0     0     0     0    16    ]  0   
         vmop                    [threads: total initially_running wait_to_block]    [time: spin block sync cleanup vmop] page_trap_count
0.396: no vm operation                  [       9          1              1    ]      [     0     0     0     0   341    ]  0   
```

> 打印`Reference`处理信息
```text
-XX:+PrintReferenceGC
```
输出内容示例：
```text
2021-02-19T12:41:30.462+0800: 5072726.605: [SoftReference, 0 refs, 0.0000521 secs]
2021-02-19T12:41:30.462+0800: 5072726.605: [WeakReference, 0 refs, 0.0000069 secs]
2021-02-19T12:41:30.462+0800: 5072726.605: [FinalReference, 0 refs, 0.0000056 secs]
2021-02-19T12:41:30.462+0800: 5072726.605: [PhantomReference, 0 refs, 0 refs, 0.0000059 secs]
2021-02-19T12:41:30.462+0800: 5072726.605: [JNI Weak Reference, 0.0000131 secs], 0.4635293 secs]
```

## 输出方式
上面只是定义了打印的内容，默认情况下，这些日志会输出到控制台（标准输出）。如果应用程序的日志也输出到控制台，日志内容就会很乱，分析起来很麻烦。如果你是追加的方式（比如`tomcat`的`catalina.out`就是追加），这个文件会越来越大。
> JVM日志分割
```text
# GC日志输出的文件路径
-Xloggc:/path/to/gc.log
# 开启日志文件分割
-XX:+UseGCLogFileRotation 
# 最多分割几个文件，超过之后从头开始写
-XX:NumberOfGCLogFiles=10
# 每个文件上限大小，超过就触发分割
-XX:GCLogFileSize=100M
```
按照这个参数，每个GC日志只要超过100M就会进行分割，最多分割10个文件，文件名依次是gc.log.0,gc.log.1,gc.log.2,gc.log.3,gc.log.4, .....
- `-Xloggc:`方式指定的日志文件，是覆盖写的方式，每次启动都会覆盖，历史日志会丢失
- `-XX:NumberOfGCLogFiles:`并不能设置为无限大，一旦超过最大分割数后，会从第0个文件开始覆盖写入

> 使用时间戳命名文件
 
于是有另一种解决方案。不使用 JVM 提供的日志分割功能，而是每次启动用时间戳命名日志文件，这样可以每次启动都使用不同的文件，就不会出现覆盖的问题了。
```text
# 使用-%t作为日志文件名
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/path/to/gc-%t.log
```
生成的文件名是这种：
- gc-2021-03-29_20-41-47.log
虽然没有覆盖的问题，但由于没有日志分割的功能，每次启动后只有一个GC日志文件，单个日志文件可能会非常巨大。过大的日志文件分析起来是很麻烦的，必须得分割。

> 二者结合
```text
# GC日志输出的文件路径
-Xloggc:/path/to/gc-%t.log
# 开启日志文件分割
-XX:+UseGCLogFileRotation 
# 最多分割几个文件，超过之后从头开始写
-XX:NumberOfGCLogFiles=10
# 每个文件上限大小，超过就触发分割
-XX:GCLogFileSize=100M
```
最终得到的日志文件名会像这个样子：
- gc-2021-03-29_20-41-47.log.0
- gc-2021-03-29_20-41-47.log.1
- gc-2021-03-29_20-41-47.log.2
- gc-2021-03-29_20-41-47.log.4
- gc-2021-03-29_20-41-47.log.5

## 最佳实践
```text
# required
-XX:+PrintGCDetails 
-XX:+PrintGCDateStamps 
-XX:+PrintTenuringDistribution 
-XX:+PrintHeapAtGC 
-XX:+PrintReferenceGC 
-XX:+PrintGCApplicationStoppedTime

# optional
-XX:+PrintSafepointStatistics 
-XX:PrintSafepointStatisticsCount=1

# GC日志文件路径
-Xloggc:/path/to/gc-%t.log
# 开启日志文件分割
-XX:+UseGCLogFileRotation 
# 最多分割几个文件，超过之后从头文件开始写
-XX:NumberOfGCLogFiles=10
# 每个文件上限大小，超过就触发分割
-XX:GCLogFileSize=100M
```