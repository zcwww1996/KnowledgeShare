[TOC]
# 1.关键字
概述：是被java语言赋予特殊含义的单词

组成规则：全部小写的英文

特点:

goto和const是作为保留字存在的，在JDK后续的版本升级中可能提升为关键字

在类似于Notepad++这样的高级记事本中会变色
　　
# 2.标识符

概述：就是给类，接口，变量，方法等等起名字的字符序列

    组成规则:

         所有的英文大小写字母
        
         数字0-9

         $和_

    注意事项：

         不能以数字开头
         
         不能以java关键字命名
         
         区分大小写
    
命名规则：起名字要做到见名之意

- 包：

其实就是硬盘上面的文件夹，一般包的名字都是以公司的域名反转之后的前两位

    www.edu360.cn->cn.edu360.www->cn.edu360
    wwww.baidu.com->com.baidu.www->com.baidu

    一个单词：全部小写

    多个单词：全部小写，每个单词之间用"."隔开
  
- 类和接口：

    一个单词：首字母大写 Person

    多个单词：每个单词的首字母大写 InputStream OuputStream
        
- 方法和变量：

    一个单词：全部小写read write
    
    多个单词：从第二个单词开始首字母大写 readLine
    
- 常量：

    一个单词：全部大写 KEY
    
    多个单词：全部大写，每个单词之间用"_"隔开 ROUND_HALFUP

# 3.常量
概述：在java程序执行的过程中，其值不能发生改变的量
字面值常量：

- 字符串常量

 就是用双引号括起来的内容    "小牛学堂"
 
- 字符常量

就是用单引号括起来的内容    '中'

- 整数常量

包括所有的整数，正整数和负整数    12 、-12

- 小数常量

包括所有的小数，正小数和负小数    12.12 、   -12.12

- 布尔常量

只有两个值，要么true要么false

- 空常量

null，在数组的时候讲解

- 自定义常量

面向对象的时候会说

# 4.进制

进制 |开头|组成
---|---|---
二进制|以0b开头|由0-1组成
八进制|以0开头|由0-7组成
十进制|默认就是十进制|由0-9组成
十六进制|以0x开头|由0-f组成
    
## 4.1 其他进制如何转换成十进制：
位值：对应位上面的数值

进制数：x进制的进制数就是x

次方：从右向左，从0开始编号，对应位上面的编号就是该位的次方

**将每一个进制上面的位值`*`进制数的次方之和就是十进制**
  
## 4.2 十进制如何转换成其他进制

**除以要转换的进制数，直至商为0，余数反转**

```java
        System.out.println("把2,8,16的数字的字符串形式，转化为10进制：");
        System.out.println(Integer.parseInt("10", 10));
        System.out.println(Integer.parseInt("10", 2));
        System.out.println(Integer.parseInt("10", 8));
        System.out.println(Integer.parseInt("10", 16));
        System.out.println();

        System.out.println("把10进制，转化为2,8,16进制：");
        System.out.println(Integer.toString(10));
        System.out.println(Integer.toBinaryString(10));
        System.out.println(Integer.toOctalString(10));
        System.out.println(Integer.toHexString(10));
        System.out.println();

```

```java
   把2,8,16的数字的字符串形式，转化为10进制：
   10
   2
   8
   16
 
   把10进制，转化为2,8,16进制：
   10
   1010
   12
   a
```

**有符号数据表示法** 

所有数据的运算都是采用补码进行的 

**原码**：就是二进制定点表示法，即最高位为符号位，“0”表示正，“1”表示负，其余位表示数值的大小

**反码**：正数的反码与其原码相同；负数的反码是对其原码逐位取反，但符号位除外。

**补码**：正数的补码与其原码相同；负数的补码是在其反码的末位加1。

 <font color="red">正数原码、反码、补码完全一致</font>

# 5.变量
概述：在java程序执行的过程中其值可以在一定范围之内发生改变的量

变量的定义格式：

**数据类型 变量名 = 初始化值；**

变量定义在哪一级大括号里面，那么这个大括号就是这个变量的作用域；在同一个作用域里面不能有同名的变量
变量不初始化不能使用

在一行上面可以同时定义多个同一种数据类型变量

# 6.数据类型
因为java语言是强类型语言，针对每一种数据都定义了对应的数据类型，针对每一种数据类型都分配了不同的内存空间
    
## 6.1 基本数据类型：
组成的单词都是小写，而且都是关键字

四类八种

四类：整数型，小数型，字符型，布尔型
  
|类型|字节|位数|默认值|
| :------------: | :------------: | :------------: | :------------: |
|  byte: | 1 | ８  |  0  |
| short: | 2 | 16  |  0  |
|  int： | 4 | 32  |  0   |
| long： | 8 | 64  |  0   |
| float：| 4 | 32  | 0.0  |
| double:| 8 | 64  | 0.0  |
|  char: | 1 | 16  |  ``  |
|boolean:| 2 |  8  |false |

**java八种基本数据类型及包装类**<br>
[![java八种基本数据类型及包装类](https://s.pc.qq.com/tousu/img/20211025/2662618_1635146396.jpg)](https://z3.ax1x.com/2021/10/25/54pSDP.jpg)

## 6.2 引用数据类型：
除了基本数据类型之外的类型都是引用数据类型

类，接口，数组

默认的整数都是int类型，如果想要声明成long类型的值，需要在数值的后面<font color="red">加l或者L</font>
  
默认的小数都是double类型，如果想要声明成float类型的值，需要在数值的后面<font color="red">加f或者F</font>

**数据类型之间的运算规则**：
1. boolean类型的数值无法和其他数据类型之间做运算
2. char,byte,short之间不能直接相互运算，需要转换成int类型数据之后再相互运算

**数据类型之间的运算精度顺序**：
 char,byte,short->int->long->float->double
 
**数据类型的强转**：
当左边的变量可以接收右边的值时，但是又由于数据类型的不匹配，可以使用数据类型的强转

目标数据类型 变量名 = (目标数据类型)(要被强转的值);
   
**字符串拼接运算**：
1. 任何数据和字符串拼接的结果都是字符串
2. 表达式的运算顺序是从左向右的，如果有大括号先算大括号里面的

## 6.3 List, Integer[], int[]的相互转换
有时候list<Integer>和数组int[]转换很麻烦。

List<String>和String[]也同理。利用Java8的特性lambda表达式，可以轻松转换

```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;
 
public class Main {
    public static void main(String[] args) {
        int[] data = {4, 5, 3, 6, 2, 5, 1};
 
        // int[] 转 List<Integer>
        List<Integer> list1 = Arrays.stream(data).boxed().collect(Collectors.toList());
        // Arrays.stream(arr) 可以替换成IntStream.of(arr)。
        // 1.使用Arrays.stream将int[]转换成IntStream。
        // 2.使用IntStream中的boxed()装箱。将IntStream转换成Stream<Integer>。
        // 3.使用Stream的collect()，将Stream<T>转换成List<T>，因此正是List<Integer>。
 
        // int[] 转 Integer[]
        Integer[] integers1 = Arrays.stream(data).boxed().toArray(Integer[]::new);
        // 前两步同上，此时是Stream<Integer>。
        // 然后使用Stream的toArray，传入IntFunction<A[]> generator。
        // 这样就可以返回Integer数组。
        // 不然默认是Object[]。
 
        // List<Integer> 转 Integer[]
        Integer[] integers2 = list1.toArray(new Integer[0]);
        //  调用toArray。传入参数T[] a。这种用法是目前推荐的。
        // List<String>转String[]也同理。
 
        // List<Integer> 转 int[]
        int[] arr1 = list1.stream().mapToInt(Integer::valueOf).toArray();
        // 想要转换成int[]类型，就得先转成IntStream。
        // 这里就通过mapToInt()把Stream<Integer>调用Integer::valueOf来转成IntStream
        // 而IntStream中默认toArray()转成int[]。
 
        // Integer[] 转 int[]
        int[] arr2 = Arrays.stream(integers1).mapToInt(Integer::valueOf).toArray();
        // 思路同上。先将Integer[]转成Stream<Integer>，再转成IntStream。
 
        // Integer[] 转 List<Integer>
        List<Integer> list2 = Arrays.asList(integers1);
        // 最简单的方式。String[]转List<String>也同理。
 
        // 同理
        String[] strings1 = {"a", "b", "c"};
        // String[] 转 List<String>
        List<String> list3 = Arrays.asList(strings1);
        // List<String> 转 String[]
        String[] strings2 = list3.toArray(new String[0]);
 
    }
}
```

打印数组
```java
    Object[] paramValues;
    
     for (int i = 0; i < paramValues.length; i++) {
            System.out.print(paramValues[i] + ", ");
        }
        
    for(Object n: paramValues)
        System.out.println(n+", ");
        
        System.out.println( Arrays.toString(paramValues) );
        System.out.println(Arrays.asList(paramValues));
        Arrays.asList(arr).stream().forEach(s -> System.out.println(s));//java8
```


# 7.算术运算符
+：
1. 正号
2. 加法
3. 字符串拼接
  
%：取的余数，a%b的取值范围：0-(b-1)

/：如果做除法运算的两个数都是整数，结果<font color="red">想要有小数时</font>，需要在除数或者被除数乘以1.0f_