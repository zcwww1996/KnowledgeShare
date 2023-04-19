[TOC]
# 1.选择结构
# 2.循环结构

辨析：**break和continue关键字的作用**

1. break：<font color="red">**直接跳出**</font>当前循环体（while、for、do while）或程序块（switch）。其中switch case执行时，一定会先进行匹配，匹配成功返回当前case的值，再根据是否有break，判断是否继续输出，或是跳出判断。

2. continue：<font color="red">不再执行</font>循环体中<font color="red">continue语句之后的代码，直接进行下一次循环。</font>

代码展示：

```java
public class Test {
    public static void main(String[] args) {
        System.out.println("-------continue测试----------");
        for (int i = 0; i < 5; i++) {
            if (i == 2) {
                System.out.println("跳过下面输出语句，返回for循环");
                continue;
            }
             
            System.out.println(i);
        }
 
 
        System.out.println("----------break测试----------");
        for (int i = 0; i < 5; i++) {
            if (i == 2) {
                System.out.println("跳过下面输出语句，终止for循环");
                break;
            }
            System.out.println(i);
        }
 
    }
}
```

[![break和continue关键字的作用](http://p1.so.qhimgs1.com/t018315c4863d44d434.jpg "break和continue区别")](https://i.niupic.com/images/2020/04/16/7qYo.png)

可以看到测试 continue时，当 i==3，直接跳过了continue之后的输出语句，进入下一次循环。

在break测试中，当 i==2，直接跳出了 for 循环，不再执行之后的循环