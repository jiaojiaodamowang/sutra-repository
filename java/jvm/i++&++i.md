# 从JVM层面理解i++与++i

## 情景1

> 源代码
```java
public class demo {
    
    public static void main(Strin[] args) {
        iPlusPlus();
        plusPlusI();
    }
    
    public static void iPlusPlus() {
        int i = 1;
        i++;
    }
    
    public static void plusPlusI() {
        int i = 1;
        ++i;
    }
}
```
> 字节码（iPlusPlus()）
```text
iconst_1
istore_1
iinc 1 1
return
```
> 字节码(plusPlusI())
```text
iconst_1
istore_1
iinc 1 1
return
```
在没有再对i进行赋值的情况下，两种写法编译后的字节码完全一样。

## 字节码定义

> iinc index const

- iinc: 操作码，局部变量表`index`位置的值`+ const`
- index: 变量索引
- const: 操作数

上述例子`iinc 1 1`即表示：局部变量表`1`位置的值执行`+1`操作

## 情景2
> 源代码
```java
public class demo {
    
    public static void main(Strin[] args) {
        iPlusPlus();
        plusPlusI();
    }
    
    public static void iPlusPlus() {
        int i = 1;
        i = i++;
    }
    
    public static void plusPlusI() {
        int i = 1;
        i = ++i;
    }
}
```
> 字节码(iPlusPlus())
```text
iconst_1
istore_1
iload_1 # 注意这里两行
iinc 1 1 # 注意这里两行
istore_1
return
```
> 字节码(plusPlusI())
```text
iconst_1
istore_1
iinc 1 1 # 注意这里两行
iload_1 # 注意这里两行
istore_1
return
```
字节码的指令相同，但是`执行顺序不同`。
- i++: 先执行`iload_1`再执行`iinc 1 1`
- ++i: 先执行`iinc 1 1`再执行`iload_1`

## JVM栈视角
### i++
![image0](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/java/jvm/resource/jvm_stack_i++_0.webp)
<br />
Step1:
![image1](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/java/jvm/resource/jvm_stack_i++_1.webp)
<br />
Step2:
![image2](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/java/jvm/resource/jvm_stack_i++_2.webp)
<br />
Step3:
![image3](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/java/jvm/resource/jvm_stack_i++_3.webp)
<br />
Step4:
![image4](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/java/jvm/resource/jvm_stack_i++_4.webp)
<br />
Step5:
![image5](https://github.com/jiaojiaodamowang/sutra-repository/blob/main/java/jvm/resource/jvm_stack_i++_5.webp)
<br />
> 完成变量`i`赋值操作后，先执行`iload_1`压入操作数栈,接着执行`iinc 1 1`将局部变量`+1`，`istore_1`会将操作数栈顶值弹出并`覆盖`局部变量表`index=1`的值。

### ++i
> 这里先执行`iinc 1 1`将局部变量`+1`，再执行`iload_1`压入操作数栈，最后再次执行`istore_1`弹出栈顶值覆盖局部变量的值。

## Reference
[JVM指令集](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.iinc)
