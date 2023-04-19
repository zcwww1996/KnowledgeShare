[TOC]
# 什么是多态?

多态是同一个行为具有多个不同表现形式或形态的能力。

1. 面向对象的三大特性：封装、继承、多态。从一定角度来看，封装和继承几乎都是为多态而准备的。这是我们最后一个概念，也是最重要的知识点。
 
2. 多态的定义：指允许不同类的对象对同一消息做出响应。即同一消息可以根据发送对象的不同而采用多种不同的行为方式。（发送消息就是函数调用）
  
3. 实现多态的技术称为：动态绑定（dynamic binding），是指在执行期间判断所引用对象的实际类型，根据其实际的类型调用其相应的方法。
  
4. 多态的作用：消除类型之间的耦合关系。

## 多态存在的三个必要条件
一、继承；

二、重写；

三、父类引用指向子类对象。

## 多态的成员访问特点：
    访问成员变量：
            编译看左边，运行看左边
    访问成员方法：
            编译看左边，运行看右边
    访问静态成员方法：
            编译看左边，运行看左边，所以我说它没有方法重写的特性

## 多态的优点
1. 消除类型之间的耦合关系
2. 可替换性
3. 可扩充性
4. 接口性
5. 灵活性
6. 简化性

## 总结
对于多态，可以总结以下几点：

一、使用父类类型的引用指向子类的对象；

二、该引用只能调用父类中定义的方法和变量；

三、如果子类中重写了父类中的一个方法，那么在调用这个方法的时候，将会调用子类中的这个方法；（动态连接、动态调用）;

四、变量不能被重写（覆盖），"重写"的概念只针对方法，如果在子类中"重写"了父类中的变量，那么在编译时会报错。

一个父类类型的引用指向一个子类的对象<br>
**<font color="red">自己用自己的，儿子有父亲一样的，就用儿子的，能省则省。儿子独有的，父亲不会，所以不能用</font>**

### 例子1：
```Java
public class Test {
    public static void main(String[] args) {
      show(new Cat());  // 以 Cat 对象调用 show 方法
      show(new Dog());  // 以 Dog 对象调用 show 方法
                
      Animal a = new Cat();  // 向上转型  
      a.eat();               // 调用的是 Cat 的 eat
      Cat c = (Cat)a;        // 向下转型  
      c.work();        // 调用的是 Cat 的 work
  }  
            
    public static void show(Animal a)  {
      a.eat();  
        // 类型判断
        if (a instanceof Cat)  {  // 猫做的事情 
            Cat c = (Cat)a;  
            c.work();  
        } else if (a instanceof Dog) { // 狗做的事情 
            Dog c = (Dog)a;  
            c.work();  
        }  
    }  
}
 
abstract class Animal {  
    abstract void eat();  
}  
  
class Cat extends Animal {  
    public void eat() {  
        System.out.println("吃鱼");  
    }  
    public void work() {  
        System.out.println("抓老鼠");  
    }  
}  
  
class Dog extends Animal {  
    public void eat() {  
        System.out.println("吃骨头");  
    }  
    public void work() {  
        System.out.println("看家");  
    }  
}
```

执行以上程序，输出结果为：
```Java
吃鱼
抓老鼠
吃骨头
看家
吃鱼
抓老鼠
```


### 例子2：

定义一个父类类型的引用指向一个子类的对象既可以使用子类强大的功能，又可以抽取父类的共性。

所以，父类类型的引用可以调用父类中定义的所有属性和方法，而对于子类中定义而父类中没有的方法，它是无可奈何的；

对于父类中定义的方法，如果子类中重写了该方法，那么父类类型的引用将会调用子类中的这个方法，这就是动态连接。

```java
class Father {
	public void func1(){
		func2();
	} 
	public void func2(){
		System.out.println("AAA");
	}
}
 
class Child extends Father{
	
	public void func1(int i){
		System.out.println("BBB");
		} 
	
	public void func2(){
		System.out.println("CCC");
		}
	
}
 
 
public class Test {
	public static void main(String[] args) {
		
		Father child = new Child();
		child.func1();//打印结果将会是什么？
  }
}
```

上面的程序是个很典型的多态的例子。子类Child继承了父类Father，并重载了父类的func1()方法，重写了父类的func2()方法。重载后的func1(int i)和func1()不再是同一个方法，由于父类中没有func1(int i)，那么，父类类型的引用child就不能调用func1(int i)方法。而子类重写了func2()方法，那么父类类型的引用child在调用该方法时将会调用子类中重写的func2()。

**经过上面的分析我们可以知道打印的结果是什么呢? 很显然,应该是"CCC"**