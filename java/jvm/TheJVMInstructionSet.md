# The Java Virtual Machine Instruction Set Introduction

## 字节码指令的组成
- 操作码`Opcode`：一个字节长度（0-255，即指令集的操作码总数不超过256条），代表特定意义的此操作
- 操作数`Operand`：零个或多个，紧跟在操作码之后，表示具体操作需要的参数

> 由于JVM是基于`栈`而不是`寄存器`的结构，有的指令只需一个`操作码`,例如：`aload_0`（将局部变量表索引为0的数据压入操作数栈）。而`invokespecial #1`（调用成员方法或构造方法，并传递常量池索引为1的常量）则是由操作码和操作数组成。

### 加载与存储指令
> 加载（load）与存储（store）相关的指令是使用最频繁的指令，用于将数据从`栈帧`的`局部变量表`和`操作数栈`之间来回传递。

#### 1.将局部变量表的变量压入操作数栈

- <font color="red">x</font>load_<font color="red">\<n\></font>：`x`为操作码助记符，`n`表示局部变量表索引，范围[0,3]
- <font color="red">x</font>load <font color="red">&nbsp;\<n\></font>：通过指定参数的形式，将局部变量表对应索引的数据压入操作数栈

|操作码助记符|数据类型|
|-|-|
|i|int|
|l|long|
|s|short|
|b|byte|
|c|char|
|f|float|
|d|double|
|a|引用数据类型|

> 大部分指令中，`byte`、`char`被编译器带符号扩展（Sign-Extend）为`int`，`short`零位扩展（Zero-Extend）为`int`。
> 所有`boolean`被零位扩展（Zero-Extend）为`int`。
> 有些指令不包含操作码助记符，如：arraylength。

示例：
```text
iload_1：将局部变量表索引为1的int变量压入操作数栈。
aload_2：将局部变量表索引为2的int应用数据类型变量压入操作数栈。
iload 5：将局部变量表索引为5的int变量（实际可能是boolean）压入操作数栈。
```

#### 2.将常量池中的常量压入操作数栈 
- <font color=red>x</font>const_<font color=red>\<n\></font>：用特殊的常量入栈

|指令|说明|
|-|-|
|iconst_<n>|n：[-1, 5]，其中`-1`用`m1`表示|
|lconst_<n>|n：[0, 1]|
|fconst_<n>|n：[0, 2]|
|dconst_<n>|n：[0, 1]|
|aconst_null|将null入栈|

- bipush <font color=red>\<n\></font>：将`[-128, 127]`压入栈
- sipush <font color=red>\<n\></font>：将`[-32768, 32767]`压入栈
- ldc <font color=red>\<n\></font>：将`[-2147483648, 2147483647]`压入栈

示例：
```text
iconst_m1：将-1压入栈
bipsush 127：将127压入栈
sipush 32767：将32767压入栈
ldc #6 <32768>：将常量池下标为6的常量（32768）压入栈
aconst_null：将null压入栈
ldc #8 <test>：将常量池下标为8的常量（“test”）压入栈
```

#### 3.将栈顶数据弹出写入局部变量表
> 对应常见的赋值语句

- <font color=red>x</font>store_<font color=red>\<n\></font>：从操作数栈弹出栈顶数据（x类型）赋值给局部变量表索引为`n`的变量，范围[0,3]
- <font color=red>x</font>store <font color=red>&nbsp;\<n\></font>：从操作数栈弹出栈顶数据（x类型）赋值给局部变量表索引为`n`的变量

|操作码助记符|数据类型|
|-|-|
|i|int|
|l|long|
|s|short|
|b|byte|
|c|char|
|f|float|
|d|double|
|a|引用数据类型|

> 由于局部变量表中前几个位置通常是最常用的，通过`xstore_<n>`的形式，一条指令仅包含操作码（占用`1`字节），起到了节省空间的目的。

示例：
```text
istore_3：将操作数栈顶数据(整数)弹出并赋值给局部变量表索引为3的变量
astore 4：将操作数栈顶数据（引用数据类型）弹出并赋值给局部变量表索引为4的变量
```

### 算数指令
> 算数指令用于对操作数栈上的数据进行某种特定运算，并将结果重新压入栈。可以分为`整型数据`和`浮点型数据`的运算指令。
> 
> 数据运算可能导致溢出（如两个很大的正整数相加），JVM规范并未对这种情况给出具体结果，故编写程序时不会有显示报错。
> 
> 当发生溢出时，使用有符号的无穷大Infinity来表示；如果某个操作结果没有明确的数学定义，将使用NaN来表示。所有使用NaN作为操作数的算数运算，结果也返回NaN。

JVM提供两种运算模式：
- 向最接近数舍入：在进行浮点数运算时，所有的结果必须舍入到一个适当的精度。如果有两种可表示的形式与该值接近，优先选择最低有效位为0的
- 向零舍入：将浮点数转为整数时，将在目标数值类型中选择一个最接近但是不大于原值的数字作为舍入结果

示例：
```text
+：iadd，ladd，fadd，dadd
-：isub，lsub，fsub，dsub
*：imul，lmul，fmul，dmul
/：idiv，ldiv，fdiv，ddiv
%：irem，lrem，frem，drem
negate（-）：ineg，lneg，fneg，dneg，
&：iand，land
|：ior，lor
^：ixor，lxor
自增：iinc
```

### 类型转换指令

- 由小转大：比如int->long
- 由大转小：比如long->int

示例：
```text
int->long：i2l
int->float：i2f #可能发生精度丢失
int->double：i2d
long->float：l2f #可能发生精度丢失
long->double：l2d #可能发生精度丢失
float->double：f2d

int->byte：i2b
int->char：i2c
int->short：i2s
long->int：l2i
float->int：f2i
float->long：f2l
double->int：d2i
double->long：d2l
double->float：d2f
# 以上由大转小都可能发生精度丢失
```

### 对象创建和访问指令

#### 1.创建指令
- newarray `aType`：创建基本数据类型（见下表）的数组
- anewarray `indextype1`，`indextype2`：创建引用数据类型（使用由indexbyte1和indexbyte2两个8位操作数合起来表示常量池中无符号16位长度的索引，找到相应的类）的数组
- multianewarray `indextype1`，`indextype2`，`dimensions`：创建多维数组
- new：创建对象

|数据类型|aType|
|-|-|
|T_BOOLEAN|4|
|T_CHAR|5|
|T_FLOAT|6|
|T_DOUBLE|7|
|T_BYTE|8|
|T_SHORT|9|
|T_INT|10|
|T_LONG|11|

示例：
```text
new #13 <java/lang/String>：创建一个字符串对象
new #15 <java/io/File>：创建一个File对象
newarray 10 (int)：创建一个int类型的数组
```

#### 2.字段访问指令

- getstatic：读静态变量
- putstatic：写静态变量
- getfield：读成员变量
- putfield：写成员变量

示例：
```text
getstatic #12 <com/demo/test.aaa>：读静态变量aaa
getfield #18 <com/demo/test.bbb>：读成员变量bbb
```

### 方法调用和返回指令

- invokevirtual：调用对象的成员方法，根据对象的实际类型进行分派，支持多态。
- invokeinterface：调用接口方法，在运行时搜索由特定对象实现的接口方法。
- invokespecial：调用一些需要特殊处理的方法，如：`构造方法`、`私有方法`、`父类方法`。
- invokestatic：调用静态方法
- invokedynamic：调用在运行时动态解析出调用点限定符所引用的方法。

示例：
```java
public class demo {
    
    public static void main(String[] args) {
        print();
        Demo demo = new demo();
        demo.run();
    }
    
    private static void print() {
        System.out.println("invoke static method");
    }
    
    private void run() {
        List<String> list = new ArrayList<>();
        list.add("aaa");
        
        ArrayList<String> list2 = new ArrayList<>();
        list2.add("bbb");
    }
}
```
```text
compiled from "Demo.java"
public class com.demo.Demo {
    public com.demo.Demo();
      code:
         0: aload_0
         1: invokespecial         #1          // Method java/lang/Object."<init>":()V
         4: return
    private void run();
      code:
         0: new                   #2          // class java/util/ArrayList
         3: dup
         4: invokespecial         #3          // Method java/util/ArrayList."<init>":()V
         7: astore_1
         8: aload_1
         9: ldc                   #4          // String aaa
        11: invokerinterface      #5,   2     // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
        16: pop
        17: new                   #2          // class java/util/ArrayList
        20: dup
        21: invokespecial         #3          // Method java/util/ArrayList."<init>":()V
        24: astore_2
        25: aload_2
        26: ldc                   #6          // String bbb
        28: invokevirtual         #7          // Method java/utl/ArrayList.add:(Ljava/lang/Object;)Z
        31: pop
        32: return
    public static void print();
      code:
         0: getstatic             #8          // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc                   #9          // String invokestatatic
         5: invokevirtual         #10         // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
    public static void main(java.lang.String[]);
      code:
         0: invokestatic          #11         // Method print:()V
         3: new                   #12         // class com/demo/Demo
         6: dup
         7: invokespecial         #13         // Method "<init>"()V
        10: astore_1
        11: aload_1
        12: invokevirtual         #14         // Method run:()V
        15: return
}
```
> Demo类共4个方法（包含隐式的无参构造方法），构造方法内部调用超类Object的构造方法。

> 成员方法`run()`中，`list`变量声明为`List`，所以`add()`方法执行了`invokeinterface`指令，在运行时确定`List`接口的实现类`ArrayList`的`add()`方法。
> 
> `list2`变量声明为`ArrayList`，所以`add()`方法执行`invokevirtual`指令。

> `main()`方法中，`print()`方法是静态方法，执行`invokestatic`指令。

|方法返回指令|类型|
|-|-|
|return|void|
|ireturn|int(boolean、byte、char、short)|
|lreturn|long|
|freturn|float|
|dreturn|double|
|areturn|引用类型|

### 操作数栈管理指令

- pop：将操作数栈栈顶的一个元素（非long、double）弹出
- pop2：将操作数栈栈顶的一个元素（long、double）或两个其他类型的元素弹出
- dup：复制操作数栈栈顶元素并重新入栈
- dup_x1：复制操作数栈栈顶元素并插入栈顶第二个元素之下
- swap：将操作数栈栈顶两个元素交换

示例：
```java
public class Demo {
    int number;
    public int inc() {
        return ++number;
    }
}
```
```text
aload_0                                            // this入栈：[this]
dup                                                // 复制栈顶`this`，此时栈上有两个this：[this,this]
getfield #2           #2 <com.demo.Demo.number>    // 将常量池索引为2的常量入栈，同时将一个`this`出栈：[this, 0]
iconst_1                                           // 常量`1`入栈：[this, 0, 1]
iadd                                               // 栈顶两个元素弹出并相加后重新入栈：[this, 1]
dup_x1                                             // 复制栈顶元素，插入栈顶第二个元素下：[1, this, 1]
putfield              #2 <com.demo.Demo.number>    // 将栈顶两个元素出栈，赋值给number：[1]
ireturn                                            // 将栈顶元素弹出返回：[]
```

### 控制转移指令

|指令|说明|
|ifeq|如果栈顶int类型变量`==0`，跳转|
|ifne|如果栈顶int类型变量`!=0`，跳转|
|iflt|如果栈顶int类型变量`<0`，跳转|
|ifle|如果栈顶int类型变量`<=0`，跳转|
|ifgt|如果栈顶int类型变量`>0`，跳转|
|ifge|如果栈顶int类型变量`>=0`，跳转|
|if_icmpeq|比较栈顶两个int类型变量`a==b`，跳转|
|if_icmpne|比较栈顶两个int类型变量`a!=b`，跳转|
|if_icmplt|比较栈顶两个int类型变量`a<b`，跳转|
|if_icmple|比较栈顶两个int类型变量`a<=b`，跳转|
|if_icmpgt|比较栈顶两个int类型变量`a>b`，跳转|
|if_icmpge|比较栈顶两个int类型变量`a>=b`，跳转|
|if_acmpeq|比较栈顶两个引用类型变量`a==b`，跳转|
|if_acmpne|比较栈顶两个引用类型变量`a!=b`，跳转|
|goto|无条件跳转|
|tableswitch|switch语句，case值连续时使用，内部只存放起始值和终止值以及若干跳转偏移量，通过给定的操作数index，立即定位到跳转偏移量位置，效率较高|
|lookupswitch|switch语句，case值离散时使用，内部存放各个离散的case-offset对，每次执行都要搜索全部case-offset对，找到匹配的case值，根据offset计算跳转地址，效率较低|

示例：
```java
public class Demo {
    
    public int isPositive(int n) {
        if (n > 0) {
            return 1;
        } else {
            return 0;
        }
    }
}
```
```text
code:
0: iload_0
1: ifeq          6      // 如果栈顶元素等于0，跳到6行处
4: iconst_1
5: ireturn
6: iconst_0
7: ireturn
```