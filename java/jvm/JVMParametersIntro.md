# JVM Parameters Introduction

## 1. 标准参数
- -verison：获取JDK版本
- -help：获取帮助信息
- -showversion：获取JDK版本吧和帮助

## 2. X参数
- -Xint：解释执行
- -Xcomp：编译执行
- -XMixed：混合模式

## 3. XX参数
- Boolean型
> -XX:+属性 或者 -XX:-属性，`+`表示开启，`-`表示关闭。

示例：
> -XX:+PrintGCDetails，表示`开启`GC详情输出
- Key-Value型
> -XX:key=value

示例：
> -XX:metaspace=100000，表示设置metaspace大小

### 3.1 查看参数
查看某个参数：
> jinfo -flag PrintGCDetails 518（JVM进程）

![image1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/java/jvm/resource/jvm_params_intro_1.png?raw=true)

查看已配置的所有参数：
> jinfo -flags 518（JVM进程）

![image2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/java/jvm/resource/jvm_params_intro_2.png?raw=true)

Tips：
> 常见的-Xms,-Xmx其实是别名，本质上还是XX参数
> 
> -Xms -> -XX:InitialHeapSize
> 
> -Xmx -> -XX:MaxHeapSize
 
示例：
> -XX:InitialHeapSize=1024m 等价于 -Xms1024m