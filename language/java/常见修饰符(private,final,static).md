[TOC]
# 1. final的概念

继承的出现提高了代码的复用性，并方便开发。但随之也有问题，有些类在描述完之后，不想被继承，或者有些类中的部分方法功能是固定的，不想让子类重写。可是当子类继承了这些特殊类之后，就可以对其中的方法进行重写，那怎么解决呢？

要解决上述的这些问题，需要使用到一个关键字final，final的意思为最终，不可变。final是个修饰符，它可以用来修饰类，类的成员，以及局部变量。
## 1.1 final的特点

- final修饰类不可以被继承，但是可以继承其他类。


```java
class Yy {}

final class Fu extends Yy{} //可以继承Yy类

class Zi extends Fu{} //不能继承Fu类
```


- final修饰的方法不可以被覆盖,但父类中没有被final修饰方法，子类覆盖/重写后可以加final。

    class Fu {


```
// final修饰的方法，不可以被覆盖，但可以继承使用

        public final void method1(){}

        public void method2(){}

    }

    class Zi extends Fu {

    //重写method2方法

    public final void method2(){}

    }
```


- final修饰的变量称为常量，这些变量只能赋值一次。


```java
final int i = 20;

i = 30; //赋值报错，final修饰的变量只能赋值一次
```


- 引用类型的变量值为对象地址值，地址值不能更改，但是地址内的对象属性值可以修改。
- 
```java
final Person p = new Person();

Person p2 = new Person();

p = p2; //final修饰的变量p，所记录的地址值不能改变

p.name = "小明";//可以更改p对象中name属性值

p不能为别的对象，而p对象中的name或age属性值可更改。
```

- 修饰成员变量，需要在创建对象前赋值，否则报错。(当没有显式赋值时，多个构造方法的均需要为其赋值。)


```java
class Demo {

  //直接赋值

  final int m = 100;

  //final修饰的成员变量，需要在创建对象前赋值，否则报错。

  final int n;

  public Demo () {

    //可以在创建对象时所调用的构造方法中，为变量n赋值

    n = 2016;

  }

}
```
# 2. static概念

当在定义类的时候，类中都会有相应的属性和方法。而属性和方法都是通过创建本类对象调用的。当在调用对象的某个方法时，这个方法没有访问到对象的特有数据时，方法创建这个对象有些多余。可是不创建对象，方法又调用不了，这时就会想，那么我们能不能不创建对象，就可以调用方法呢？

可以的，我们可以通过static关键字来实现。static它是静态修饰符，一般用来修饰类中的成员。

## 2.1 static特点

- 被static修饰的成员变量属于类，不属于这个类的某个对象。（ <font style="color: red;">也就是说，多个对象在访问或修改static修饰的成员变量时，其中一个对象将static成员变量值进行了修改，其他对象中的static成员变量值跟着改变，即多个对象共享同一个static成员变量</font>）

代码演示：


```java
class Demo {

public static int num = 100;

}

 

class Test {

  public static void main(String[] args) {

    Demo d1 = new Demo();

    Demo d2 = new Demo();

    d1.num = 200;

    System.out.println(d1.num); //结果为200

    System.out.println(d2.num); //结果为200

  }

}
```


- 被static修饰的成员可以并且建议通过类名直接访问。

**访问静态成员的格式：**

 <font style="color: red;">类名.静态成员变量名</font>

 <font style="color: red;">类名.静态成员方法名(参数)</font>

对象名.静态成员变量名      ------不建议使用该方式，会出现警告

对象名.静态成员方法名(参数) ------不建议使用该方式，会出现警告

## 2.2 static注意事项

- 静态内容是优先于对象存在，只能访问静态，不能使用this/super。静态修饰的内容存于静态区。
- 同一个类中，静态成员只能访问静态成员


```java
class Demo {

  //成员变量

  public int num = 100;

  //静态成员变量

  public static int count = 200;

  //静态方法

  public static void method () {

    //System.out.println(num); 静态方法中，只能访问静态成员变量或静态成员方法

    System.out.println(count);

  }

}
```

2.3 定义静态常量

开发中，我们想在类中定义一个静态常量，通常使用<font style="color: red;">public static final修饰的变量来完成定义</font>。此时变量名用全部大写，多个单词使用下划线连接。

定义格式：

<font style="color: red;">**public static final 数据类型 变量名 = 值**;</font>

如下演示：


```java
class Hello {

  public static final String SAY_HELLO = "你好Java";

  public static void method(){

    System.out.println("一个静态方法");

  }

}
```

当我们想使用类的静态成员时，不需要创建对象，直接使用类名来访问即可。


``java
System.out.println(Hello.SAY_HELLO); //打印

    Hello.method(); // 调用一个静态方法
```


     

- 注意：


   1. <font style="color: red;">接口中的每个成员变量都默认使用public static final修饰。

   2. 所有接口中的成员变量已是静态常量，由于接口没有构造方法，所以必须显示赋值。可以直接用接口名访问。</font>

```java
interface Inter {

  public static final int COUNT = 100;

 }
```


访问接口中的静态变量

    Inter.COUNT

# 3. 关于private的描述
     

了解到封装在生活的体现之后，又要回到Java中，细说封装的在Java代码中的体现，先从描述Person说起。

描述人。Person

属性：年龄。

行为：说话：说出自己的年龄。


```java
class Person {

  int age;

  String name;


  public void show() {

    System.out.println("age=" + age + ",name" + name);

  }

}


public class PersonDemo {

  public static void main(String[] args) {

    // 创建Person对象

    Person p = new Person();

    p.age = -20; // 给Person对象赋值

    p.name = "人妖";

    p.show(); // 调用Person的show方法

  }

}
```


通过上述代码发现，虽然我们用Java代码把Person描述清楚了，但有个严重的问题，就是Person中的属性的行为可以任意访问和使用。这明显不符合实际需求。

可是怎么才能不让访问呢？需要使用一个Java中的关键字也是一个修饰符 private(私有，权限修饰符)。只要将Person的属性和行为私有起来，这样就无法直接访问。



```
class Person {

  private int age;

  private String name;


  public void show() {

    System.out.println("age=" + age + ",name" + name);

  }

}
```


年龄已被私有，错误的值无法赋值，可是正确的值也赋值不了，这样还是不行，那肿么办呢？按照之前所学习的封装的原理，隐藏后，还需要提供访问方式。只要对外提供可以访问的方法，让其他程序访问这些方法。同时在方法中可以对数据进行验证。

一般对成员属性的访问动作：赋值(设置 set)，取值(获取 get)，因此对私有的变量访问的方式可以提供对应的 setXxx或者getXxx的方法。


```java
class Person {

  // 私有成员变量

  private int age;

  private String name;

  // 对外提供设置成员变量的方法
  public void setAge(int a) {

    // 由于是设置成员变量的值，这里可以加入数据的验证

    if (a < 0 || a > 130) {

      System.out.println(a + "不符合年龄的数据范围");

      return;

    }

    age = a;

  }

  // 对外提供访问成员变量的方法
  public void getAge() {

    return age;

  }

}
```



- 总结：

  1. 类中不需要对外提供的内容都私有化，包括属性和方法。
  2. 以后再描述事物，属性都私有化，并提供setXxx getXxx方法对其进行访问。

- 注意：私有仅仅是封装的体现形式而已。