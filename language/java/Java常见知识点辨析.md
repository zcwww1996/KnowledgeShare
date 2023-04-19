[TOC]

# 1. Java中==号与equals()方法的区别
来源：https://blog.csdn.net/striverli/article/details/52997927
## 1.1 `==`号
首先，<font color="red">`==`号</font>在比较<font color="red">基本数据类型</font>时比较的是<font color="red">值</font>，而用<font color="red">`==`号</font>比较两个<font color="red">对象</font>时比较的是两个对的<font color="red">地址值</font>：

```java
int x = 10;
int y = 10;
String str1 = new String("abc");
String str2 = new String("abc");
System.out.println(x == y); // 输出true
System.out.println(str1 == str2); // 输出false
```

## 1.2 equals()方法
那equals()方法呢？我们可以通过查看源码知道，equals()方法存在于Object类中，因为Object类是所有类的直接或间接父类，也就是说所有的类中的<font color="blue">equals()</font>方法都继承自Object类，而通过源码我们发现，Object类中equals()方法底层<font color="blue">依赖的是`==`号</font>，那么，在所有<font color="blue">没有重写equals()方法</font>的类中，调用equals()方法其实和使用==号的效果一样，也是<font color="blue">比较的地址值</font>，然而，Java提供的所有类中，<font color="blue">绝大多数类都重写了equals()方法</font>，重写后的equals()方法一般都是<font color="blue">比较两个对象的值</font>。

# 2. String，StringBuilder，StringBuffer三者的区别

这三个类之间的区别主要是在两个方面，即运行速度和线程安全这两方面。
## 2.1 运行速度
首先说运行速度，或者说是执行速度，在这方面运行速度快慢为：<font color="red">**StringBuilder > StringBuffer > String**</font>

String最慢的原因：

**String为字符串常量，而StringBuilder和StringBuffer均为字符串变量，即String对象一旦创建之后该对象是不可更改的，但后两者的对象是变量，是可以更改的。**

以下面一段代码为例：

```java
String str="abc";
System.out.println(str);
str=str+"de";
System.out.println(str);
```

如果运行这段代码会发现先输出“abc”，然后又输出“abcde”，好像是str这个对象被更改了，其实，这只是一种假象罢了，JVM对于这几行代码是这样处理的，首先创建一个String对象str，并把“abc”赋值给str，然后在第三行中，其实JVM又创建了一个新的对象也名为str，然后再把原来的str的值和“de”加起来再赋值给新的str，而原来的str就会被JVM的垃圾回收机制（GC）给回收掉了，所以，str实际上并没有被更改，也就是前面说的String对象一旦创建之后就不可更改了。所以，Java中对String对象进行的操作实际上是一个不断创建新的对象并且将旧的对象回收的一个过程，所以执行速度很慢。

而StringBuilder和StringBuffer的对象是变量，对变量进行操作就是直接对该对象进行更改，而不进行创建和回收的操作，所以速度要比String快很多。

另外，有时候我们会这样对字符串进行赋值

```java
String str="abc"+"de";
StringBuilder stringBuilder=new StringBuilder().append("abc").append("de");
System.out.println(str); 
System.out.println(stringBuilder.toString());
```

这样输出结果也是“abcde”和“abcde”，但是String的速度却比StringBuilder的反应速度要快很多，这是因为第1行中的操作和`String str="abcde";`是完全一样的，所以会很快

而如果写成下面这种形式

```java
String str1="abc";
String str2="de";
String str=str1+str2;
```

那么JVM就会像上面说的那样，不断的创建、回收对象来进行这个操作了。速度就会很慢。

## 2.2 线程安全

<font color="red">**在线程安全上，StringBuilder是线程不安全的，而StringBuffer是线程安全的**</font>

如果一个StringBuffer对象在字符串缓冲区被多个线程使用时，StringBuffer中很多方法可以带有synchronized关键字，所以可以保证线程是安全的，但StringBuilder的方法则没有该关键字，所以不能保证线程安全，有可能会出现一些错误的操作。所以如果要进行的操作是多线程的，那么就要使用StringBuffer，但是在单线程的情况下，还是建议使用速度比较快的StringBuilder。

## 2.3 总结
- <font color="red">**String：适用于少量的字符串操作的情况**</font>

- <font color="red">**StringBuilder：适用于单线程下在字符缓冲区进行大量操作的情况**</font>

- <font color="red">**StringBuffer：适用多线程下在字符缓冲区进行大量操作的情况**</font>

# 3. Comparable与Comparator的区别
## 3.1 概述
Comparable和Comparator都是用来实现集合中元素的比较、排序的。

Comparable是在**集合内部定义**的方法实现的排序，位于java.lang下。

Comparator是在**集合外部实现**的排序，位于java.util下。

Comparable是一个对象本身就已经支持自比较所需要实现的接口，如String、Integer自己就实现了Comparable接口，可完成比较大小操作。自定义类要在加入list容器中后能够排序，也可以实现Comparable接口，在用Collections类的sort方法排序时若不指定Comparator，那就以自然顺序排序。所谓自然顺序就是实现Comparable接口设定的排序方式。

Comparator是一个专用的比较器，当这个对象不支持自比较或者自比较函数不能满足要求时，可写一个比较器来完成两个对象之间大小的比较。Comparator体现了一种策略模式(strategy design pattern)，就是不改变对象自身，而用一个策略对象(strategy object)来改变它的行为。

总而言之Comparable是自已完成比较，Comparator是外部程序实现比较。

## 3.2 例子
### 3.2.1 Comparable

Comparable是排序接口。若一个类实现了Comparable接口，就意味着该类支持排序。实现了Comparable接口的类的对象的列表或数组可以通过Collections.sort或Arrays.sort进行自动排序。

此外，实现此接口的对象可以用作有序映射中的键或有序集合中的集合，无需指定比较器。该接口定义如下：

```java
package java.lang;
import java.util.*;
public interface Comparable<T> 
{
    public int compareTo(T o);
}
```

T表示可以与此对象进行比较的那些对象的类型。

此接口只有一个方法compare，比较此对象与指定对象的顺序，如果该对象小于、等于或大于指定对象，则分别返回负整数、零或正整数。

现在有两个Person类的对象，我们如何来比较二者的大小呢？我们可以通过让Person实现Comparable接口：

```java
public class Person implements Comparable<Person>
{
    String name;
    int age;
    public Person(String name, int age)
    {
        super();
        this.name = name;
        this.age = age;
    }
    public String getName()
    {
        return name;
    }
    public int getAge()
    {
        return age;
    }
    @Override
    public int compareTo(Person p)
    {
        return this.age-p.getAge();
    }
    public static void main(String[] args)
    {
        Person[] people=new Person[]{new Person("xujian", 20),new Person("xiewei", 10)};
        System.out.println("排序前");
        for (Person person : people)
        {
            System.out.print(person.getName()+":"+person.getAge());
        }
        Arrays.sort(people);
        System.out.println("\n排序后");
        for (Person person : people)
        {
            System.out.print(person.getName()+":"+person.getAge());
        }
    }
}
```

程序执行结果为：

![Comparable结果](https://i.niupic.com/images/2020/01/02/6d8k.png)
### 3.2.2 Comparator

Comparator是比较接口，我们如果需要控制某个类的次序，而该类本身不支持排序(即没有实现Comparable接口)，那么我们就可以建立一个“该类的比较器”来进行排序，这个“比较器”只需要实现Comparator接口即可。也就是说，我们可以通过实现Comparator来新建一个比较器，然后通过这个比较器对类进行排序。该接口定义如下：

```java
package java.util;
public interface Comparator<T>
 {
    int compare(T o1, T o2);
    boolean equals(Object obj);
 }
```

注意：
1) 若一个类要实现Comparator接口：它一定要实现compareTo(T o1, T o2) 函数，但可以不实现 equals(Object obj) 函数。
2) int compare(T o1, T o2) 是“比较o1和o2的大小”。返回“负数”，意味着“o1比o2小”；返回“零”，意味着“o1等于o2”；返回“正数”，意味着“o1大于o2”。

现在假如上面的Person类没有实现Comparable接口，该如何比较大小呢？我们可以新建一个类，让其实现Comparator接口，从而构造一个“比较器"。

```java
public class PersonCompartor implements Comparator<Person>
{
    @Override
    public int compare(Person o1, Person o2)
    {
        return o1.getAge()-o2.getAge();
    }
}
```

现在我们就可以利用这个比较器来对其进行排序：

```java
public class Person
{
    String name;
    int age;
    public Person(String name, int age)
    {
        super();
        this.name = name;
        this.age = age;
    }
    public String getName()
    {
        return name;
    }
    public int getAge()
    {
        return age;
    }
    public static void main(String[] args)
    {
        Person[] people=new Person[]{new Person("xujian", 20),new Person("xiewei", 10)};
        System.out.println("排序前");
        for (Person person : people)
        {
            System.out.print(person.getName()+":"+person.getAge());
        }
        Arrays.sort(people,new PersonCompartor());
        System.out.println("\n排序后");
        for (Person person : people)
        {
            System.out.print(person.getName()+":"+person.getAge());
        }
    }
}
```

程序运行结果为：

![Comparator结果](https://i.ibb.co/4pQsk3t/Comparator.png)
## 3.3 总结
Comparable和Comparator区别比较


Comparable是排序接口，若一个类实现了Comparable接口，就意味着“该类支持排序”。而Comparator是比较器，我们若需要控制某个类的次序，可以建立一个“该类的比较器”来进行排序。

Comparable相当于“内部比较器”，而Comparator相当于“外部比较器”。

两种方法各有优劣， 用Comparable 简单， 只要实现Comparable 接口的对象直接就成为一个可以比较的对象，但是需要修改源代码。 用Comparator 的好处是不需要修改源代码， 而是另外实现一个比较器， 当某个自定义的对象需要作比较的时候，把比较器和对象一起传递过去就可以比大小了， 并且在Comparator 里面用户可以自己实现复杂的可以通用的逻辑，使其可以匹配一些比较简单的对象，那样就可以节省很多重复劳动了。