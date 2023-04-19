[TOC]
# 反射定义
反射：就是对于任意一个类都能够知道它的成员（成员变量和成员方法），对于任意一个对象都能够调用它的属性和方法

相当解剖一个类，先要获取字节码文件对象，获取字节码文件对象一共常用的有三种方式
## 获取字节码文件对象的三种方式
```java
public class ReflectDemo {
    public static void main(String[] args) throws Exception {
        //public final Class<?> getClass()返回此 Object 的运行时类
        Class clazz = new Person().getClass();
        //直接通过类名.class属性
        Class clazz2 = Person.class;
        //public static Class<?> forName(String className)返回与带有给定字符串名的类或接口相关联的 Class 对象
        Class clazz3 = Class.forName("cn.edu360.Person");
        
        System.out.println(clazz==clazz2);//true
        System.out.println(clazz==clazz3);//true
    }
}
```
```
public class Person {
    public String name;
    protected int age;
    char sex;
    private String address;
    public Person(){
        
    }
    protected Person(String name){
        this.name = name;
    }
    Person(String name,int age){
        this.name = name;
        this.age = age;
    }
    private Person(String name,int age,String address){
        this.name = name;
        this.age = age;
        this.address = address;
    }
    private Person(String name,int age,char sex,String address){
        this.name = name;
        this.age = age;
        this.address = address;
        this.sex = sex;
    }
    
    public void show(){
        System.out.println("show");
    }
    protected void test(String msg){
        System.out.println(msg);
    }
    int add(int a,int b){
        return a+b;
    }
    private void method(){
        System.out.println("method");
    }
    @Override
    public String toString() {
        return "Person [name=" + name + ", age=" + age + ", sex=" + sex + ", address=" + address + "]";
    }
}
```
## 解剖构造方法
构造方法是创建对象
```
public class ReflectConstructor {
    public static void main(String[] args) throws Exception {
        //需要获取字节码文件对象
        Class clazz = Class.forName("cn.edu360.Person");
        
        //获取所有的构造方法
        //public Constructor<?>[] getConstructors()返回一个包含某些 Constructor 对象的数组，这些对象反映此 Class 对象所表示的类的所有公共构造方法
        //Constructor[] constructors = clazz.getConstructors();
        //public Constructor<?>[] getDeclaredConstructors()返回 Constructor 对象的一个数组，这些对象反映此 Class 对象表示的类声明的所有构造方法
        //constructors = clazz.getDeclaredConstructors();
        //for (Constructor c : constructors) {
            //System.out.println(c);
        //}
        
        //获取指定的构造方法对象创建当前类的对象
        
        //获取公共的构造方法并创建对象
        //public Constructor<T> getConstructor(Class<?>... parameterTypes)返回一个 Constructor 对象，它反映此 Class 对象所表示的类的指定公共构造方法
        Constructor c = clazz.getConstructor();
        //public T newInstance(Object... initargs)使用此 Constructor 对象表示的构造方法来创建该构造方法的声明类的新实例，并用指定的初始化参数初始化该实例
        Object obj = c.newInstance();
        //当类中有空参的构造方法时，可以直接使用字节码文件对象的newInstance方法创建一个对象
        //public T newInstance()创建此 Class 对象所表示的类的一个新实例
        obj = clazz.newInstance();
        System.out.println(obj);
        
        //获取受保护的构造方法并创建对象
        c = clazz.getDeclaredConstructor(String.class);
        obj = c.newInstance("张三");
        System.out.println(obj);
        
        //获取默认修饰符修饰的构造方法并创建对象
        c = clazz.getDeclaredConstructor(String.class,int.class);
        obj = c.newInstance("张三",18);
        System.out.println(obj);
        
        //获取私有的构造方法并创建对象
        c = clazz.getDeclaredConstructor(String.class,int.class,String.class);
        //取消java语法检查
        c.setAccessible(true);//暴力反射
        obj = c.newInstance("张三",18,"北京");
        System.out.println(obj);
        
        c = clazz.getDeclaredConstructor(String.class,int.class,char.class,String.class);
        //取消java语法检查
        c.setAccessible(true);//暴力反射
        obj = c.newInstance("张三",18,'男',"北京");
        System.out.println(obj);
    }
}
```
## 解剖成员变量
```
public class ReflectField {
    public static void main(String[] args) throws Exception {
        //获取字节码文件对象
        Class clazz = Class.forName("cn.edu360.Person");
        
        //获取所有的成员变量对象
        //public Field[] getFields()返回一个包含某些 Field 对象的数组，这些对象反映此 Class 对象所表示的类或接口的所有可访问公共字段
        //Field[] fields = clazz.getFields();
        //public Field[] getDeclaredFields()返回 Field 对象的一个数组，这些对象反映此 Class 对象所表示的类或接口所声明的所有字段。包括公共、保护、默认（包）访问和私有字段，但不包括继承的字段
        //fields = clazz.getDeclaredFields();
        //for (Field f : fields) {
            //System.out.println(f);
        //}
        
        //获取指定的成员变量对象并赋值
        Object obj = clazz.newInstance();
        System.out.println(obj);
        
        //获取公共的成员变量并赋值
        //public Field getField(String name)返回一个 Field 对象，它反映此 Class 对象所表示的类或接口的指定公共成员字段
        Field field = clazz.getField("name");
        //public void set(Object obj,Object value)将指定对象变量上此 Field 对象表示的字段设置为指定的新值
        field.set(obj, "李四");//把obj上面的name字段赋值为"李四"
        System.out.println(obj);
        
        //获取受保护的成员变量并赋值
        field = clazz.getDeclaredField("age");
        field.set(obj, 18);
        System.out.println(obj);
        
        //获取默认修饰符的成员变量并赋值
        field = clazz.getDeclaredField("sex");
        field.set(obj, '女');
        System.out.println(obj);
        
        //获取私有成员变量并赋值
        field = clazz.getDeclaredField("address");
        field.setAccessible(true);//暴力反射，取消java语法检查
        field.set(obj, "天津");
        System.out.println(obj);
    }
}
```
## 解剖成员方法
```
public class ReflectMethod {
    public static void main(String[] args) throws Exception {
        //获取字节码文件对象
        Class clazz = Class.forName("cn.edu360.Person");
        
        //获取所有的成员方法
        //public Method[] getMethods()返回一个包含某些 Method 对象的数组，这些对象反映此 Class 对象所表示的类或接口
        //Method[] methods = clazz.getMethods();
        //public Method[] getDeclaredMethods()返回 Method 对象的一个数组，这些对象反映此 Class 对象表示的类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法
        //methods = clazz.getDeclaredMethods();
        //for (Method m : methods) {
            //System.out.println(m);
        //}
        
        //获取指定的成员方法并使用
        Object obj = clazz.newInstance();
        //获取公共的方法并使用
        /*
        * public Method getMethod(String name,Class<?>... parameterTypes)返回一个 Method 对象，它反映此 Class 对象所表示的类或接口的指定公共成员方法
        *        name - 方法名
                parameterTypes - 参数列表
        */
        Method method = clazz.getMethod("show", null);
        //public Object invoke(Object obj,Object... args)对带有指定参数的指定对象调用由此 Method 对象表示的底层方法
        method.invoke(obj, null);
        
        //获取受保护的成员方法并使用
        method = clazz.getDeclaredMethod("test", String.class);
        method.invoke(obj, "今天29号");
        
        //获取默认修饰符修饰的成员方法并使用
        method  = clazz.getDeclaredMethod("add", int.class,int.class);
        Object value = method.invoke(obj, 1,1);
        System.out.println(value);
        
        //获取私有的成员方法并使用
        method  = clazz.getDeclaredMethod("method", null);
        method.setAccessible(true);//暴力反射，取消java语法检查
        method.invoke(obj, null);
    }
}
```
