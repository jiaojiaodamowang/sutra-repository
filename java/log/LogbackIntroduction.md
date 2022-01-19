# Logback Introduction

## Dependency
> 通过`pom.xml`文件引入依赖。
```xml
<dependencies>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-core</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

## Configuration
> 通过`logback.xml`文件进行配置。
### tag:\<configuration>

|属性|说明|默认值|
|-|-|-|
|scan|当此属性设置为true时，如果配置文件发生改变，将会被重新加载|true|
|scanPeriod|设置监测配置文件是否有修改的时间间隔，仅当`scan`=`true`生效|60 seconds|
|debug|当此属性设置为true时，打印出logback内部日志信息，实时查看logback运行状态|false

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false">
    <!-- 其他配置省略-->  
</configuration>
```

#### tag:\<contextName>
> 每个logger都关联到logger上下文，默认上下文名称为`default`。使用`<contextName>`设置,用于区分不同应用程序的记录。一旦设置,不能修改。
```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false">
    <contextName>demo</contextName>
    <!-- 其他配置省略-->  
</configuration>
```

#### tag:\<property>
> 用来定义变量值,通过`<property`定义的值会被插入到logger上下文中。定义变量后,可以使`${}`来使用变量。

|属性|说明|默认值|
|-|-|-|
|name|属性名称||
|value|属性值||

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false">
    <property name="context_name" value="demo" />
    <contextName>${context_name}</contextName>
    <!-- 其他配置省略-->  
</configuration>
```

#### tag:\<timestamp>

|属性|说明|默认值|
|-|-|-|
|key|标识此`<timestamp>`的名称||
|datePattern|设置将当前时间（解析配置文件的时间）转换为字符串的模式，遵循java.txt.SimpleDateFormat的格式||

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false">
    <timestamp key="default" datePattern="yyyy-MM-dd HH:mm:ss"/>
    <!-- 其他配置省略-->  
</configuration>
```

#### tag:\<logger> & \<root>
> logger

|属性|说明|默认值|
|-|-|-|
|name|指定受此logger约束的包或者全限定类名||
|level|日志打印级别|`TRACE`，`DEBUG`，`INFO`，`WARN`，`ERROR`，`ALL`，`OFF`，还有一个特殊值`INHERITED`或者同义词`NULL`，代表强制执行上级的级别。如果未设置此属性，那么当前logger继承上级logger的日志级别|
|additivity|是否向上级logger传递打印信息（即logger本身打印一次，root接到后又打印一次）|true|

> root：特殊的`<logger>`

|属性|说明|默认值|
|-|-|-|
|level|日志打印级别|同`<logger>`|

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false">
    <contextName>demo</contextName>
    <!-- 其他配置省略-->
    
    <logger name="com.example.demo" level="INFO" additivity="false">
        <appender-ref ref="STDOUT"/>    
    </logger>
    
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

#### tag:\<appender>

> 负责写日志的组件

|属性|说明|默认值|
|-|-|-|
|name|标识此appender||
|class|appender全限定类名||

> ConsoleAppender：向控制台输出日志
> FileAppender：向文件输出日志
> RollingFileAppender：向文件滚动输出日志

```xml
<configuration>
    <!-- Property -->
    <property name="pattern" value="%d{yyyy-MM-dd:HH:mm:ss.SSS} [%t] %p [%c] : %msg%n"/>
    
    <!-- ConsoleAppender -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <target>System.out</target>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>

    <!-- FileAppender -->
    <appender name="TEST" class="ch.qos.logback.core.FileAppender">
        <file>test.log</file>
        <encoder>
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>

    <!-- RollingFileAppender -->
    <appender name="INFO" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>info.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>info.%d{yyyy-MM-dd}.%d.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${pattern}</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    <appender name="ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>error.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>error.%d{yyyy-MM-dd}.%d.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>${pattern}</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    
    <!-- root -->
    <root level="DEBUG">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="INFO" />
        <appender-ref ref="ERROR" />
    </root>
</configuration>
```

#### tag:\<encoder> & \<pattern>

> 配置日志输出格式

<table>
    <tr>
        <th>转换符</th>
        <th>说明</th>
    </tr>
    <tr>
        <td>c{length}<br/>lo{length}<br/>logger{length}</td>
        <td>输出logger名称<br/>
            <table>
                <tr><th>示例</th><th>logger名称</th><th>输出内容</th></tr>
                <tr><td>%c</td><td>mainPackage.sub.sample.Bar</td><td>mainPackage.sub.sample.Bar</td></tr>
                <tr><td>%c{0}</td><td>mainPackage.sub.sample.Bar</td><td>Bar</td></tr>
                <tr><td>%c{0}</td><td>mainPackage.sub.sample.Bar</td><td>Bar</td></tr>
            </table>
        </td>
    </tr>
    <tr>
        <td>C{length}<br/>class</td>
        <td>输出调用记录日志方法的全限定类名，影响性能，慎用</td>
    </tr>
    <tr>
        <td>contextName<br/>cn</td>
        <td>输出上下文名称</td>
    </tr>
    <tr>
        <td>d{pattern}<br/>date{pattern}</td>
        <td>
            <table>
                <tr><th>示例</th><th>pattern</th><th>输出内容</th></tr>
                <tr><td>%d</td><td></td><td>2006-10-20 14:06:49,812</td></tr>
                <tr><td>%d</td><td>yyyy-MM-dd:HH:mm:ss.SSS</td><td>2006-10-20 14:06:49.812</td></tr>
            </table>
        </td>
    </tr>
    <tr>
        <td>F / file</td>
        <td>输出调用记录日志方法的源文件名，影响性能，慎用</td>
    </tr>
    <tr>
        <td>L / line</td>
        <td>输出调用记录日志方法的行号，影响性能，慎用</td>
    </tr>
    <tr>
        <td>m / msg / message</td>
        <td>输出调用记录日志方法时传入的信息</td>
    </tr>
    <tr>
        <td>M / method</td>
        <td>输出调用记录日志方法的方法名</td>
    </tr>
    <tr>
        <td>p / le / level</td>
        <td>输出日志级别</td>
    </tr>
        <tr>
        <td>t / thread</td>
        <td>输出调用记录日志方法的线程</td>
    </tr>
</table>

<table>
    <tr>
        <th>格式修饰符</th>
        <th>说明</th>
    </tr>
    <tr>
        <td>%20logger</td>
        <td>右对齐，logger名称长度不足20字符，空格填充</td>
    </tr>
    <tr>
        <td>%-20logger</td>
        <td>左对齐，logger名称长度不足20字符，空格填充</td>
    </tr>
    <tr>
        <td>%.30logger</td>
        <td>logger名称长度超30字符，从头开始截取</td>
    </tr>
    <tr>
        <td>%.-30logger</td>
        <td>logger名称长度超30字符，从尾开始截取</td>
    </tr>
    <tr>
        <td>%20.30logger</td>
        <td>右对齐，logger名称长度不足20字符，空格填充；logger名称长度超30字符，从头开始截取</td>
    </tr>
    <tr>
        <td>%-20.-30logger</td>
        <td>左对齐，logger名称长度不足20字符，空格填充；logger名称长度超30字符，从尾开始截取</td>
    </tr>
</table>

```xml
<encoder>
    <pattern>%d{yyyy-MM-dd:HH:mm:ss.SSS} [%t] %p [%c] : %msg%n</pattern>
</encoder>
```

### Asynchronous Mode
> 异步输出日志只需要添加一个appender，并指向上例中的appender即可。
```xml
<appender name="ASYNC-INFO" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 丢弃日志，如果队列的80%已满，则会丢弃TRACE，DEBUG，INFO级别的日志-->
    <!--discardingThreshold>0</discardingThreshold-->
    <!-- 队列大小 -->
    <!--queueSize>256</queueSize-->
    <!-- 指定附加的appender，唯一 -->
    <appender-ref ref="INFO" />
</appender>
```

## Reference
- [Logback Home](https://logback.qos.ch/)