[TOC]
# 1. 数组
数组是java语言内置的数据类型，他是一个线性的序列，所有可以快速访问其他的元素，数组和其他语言不同，当你创建了一个数组时，他的容量是不变的，而且在生命周期也是不能改变的，还有JAVA数组会做边界检查，如果发现有越界现象，会报RuntimeException异常错误，当然检查边界会以效率为代价。
# 2. 集合
JAVA还提供其他集合，**list，map，set**，他们处理对象的时候就好像这些对象没有自己的类型一样，而是直接归根于Object，这样只需要创建一个集合，把对象放进去，取出时转换成自己的类型就行了。
# 3. 数组和集合的区别

1. <font color="red">数组声明了它容纳的元素的类型，而集合不声明。</font>
2. <font color="red">数组是静态的，一个数组实例具有固定的大小，一旦创建了就无法改变容量了。而集合是可以动态扩展容量，可以根据需要动态改变大小，集合提供更多的成员方法，能满足更多的需求。</font>
3. <font color="red">数组的存放的类型只能是一种（基本类型/引用类型）,集合存放的类型可以不是一种(不加泛型时添加的类型是Object)。</font>
4. <font style="background-color: yellow;color: red;">**数组是java语言中内置的数据类型,是线性排列的,执行效率或者类型检查都是最快的**。</font>

> [Java数组转ArrayList的注意事项](https://blog.csdn.net/makersy/article/details/107645451)
> 
> 将一个int[]数组转化成List<Integer>类型，好像是一个挺常见的场景。于是立刻写下：
> 
> 
> ```
> ArrayList<Integer> list = new ArrayList<>(Arrays.asList(array));
> 
> 报错：
> Line 22: error: incompatible types: Integer[] cannot be converted to int[]
>             int[] temp = new Integer[size];
>                          ^
> Line 35: error: incompatible types: inference variable T has incompatible bounds
>             List<Integer> list = Arrays.asList(temp);
>                                               ^
>     equality constraints: Integer
>     lower bounds: int[]
>   where T is a type-variable:
>     T extends Object declared in method <T>asList(T...)
> 2 errors
> 
> ```
> 
> 为什么报格式不匹配呢？
> 
> 因为`Arrays.asList()`是泛型方法，传入的对象必须是对象数组。如果传入的是基本类型的数组，那么此时得到的list只有一个元素，那就是这个数组对象`int[]`本身。
> 
> 解决方法：<br>
> 将基本类型数组转换成包装类数组，这里将`int[]`换成`Integer[]`即

# 4. 集合体系结构

**Collection**
- List (有序集合，允许相同元素和null)
  - LinkedList (非同步，允许相同元素和null，遍历效率低插入和删除效率高)
  - ArrayList (非同步，允许相同元素和null，实现了动态大小的数组，遍历效率高，用的多)
  - Vector (同步，允许相同元素和null，效率低)
    - Stack (继承自Vector，实现一个后进先出的堆栈)
- Set (非同步，无序集合，不允许相同元素【底层基于Map，key不允许重复】，最多有一个null元素)
  - HashSet(非同步，无序集合，不允许相同元素，最多有一个null元素)
  - TreeSet(非同步，有序，唯一，底层是一个TreeMap)
  - linkedHashSet(非同步，底层LinkedHashMap)

```
graph TD
Collection((Collection))
Collection---List
Collection---Set
List---LinkedList
List---ArrayList
List---Vector
Vector---Stack
Set---HashSet
Set---SortedSet
HashSet---LinkedHashSet
SortedSet---NavigableSet
NavigableSet---TreeSet
```

**Map**
(没有实现collection接口，key不能重复，value可以重复，一个key映射一个value)

- Hashtable (实现Map接口，同步，不允许null作为key和value，用自定义的类当作key的话要复写hashCode和eques方法)
- HashMap (实现Map接口，非同步，允许null作为key和value，用的多)
- WeakHashMap (实现Map接口)

```
graph TD
Map((Map))
Map---Hashtable
Map---HashMap
Map---WeakHashMap
```

## 4.1 Collection接口

[Java线程安全的集合详解](https://blog.csdn.net/lixiaobuaa/article/details/79689338)

> 早期的线程安全的集合，它们是Vector和HashTable
> 
> **一、Collections包装方法**<br>
> Vector和HashTable被弃用后，它们被ArrayList和HashMap代替，但它们不是线程安全的，所以Collections工具类中提供了相应的包装方法把它们包装成线程安全的集合
> 
> ```java
> List<E> synArrayList = Collections.synchronizedList(new ArrayList<E>());
> 
> Set<E> synHashSet = Collections.synchronizedSet(new HashSet<E>());
> 
> Map<K,V> synHashMap = Collections.synchronizedMap(new HashMap<K,V>());
> ```
> Collections针对每种集合都声明了一个线程安全的包装类，在原集合的基础上添加了锁对象，集合中的每个方法都通过这个锁对象实现同步
> 
> **二、java.util.concurrent包中的集合**<br>
> 1. ConcurrentHashMap<br>
> ConcurrentHashMap和HashTable都是线程安全的集合，它们的不同主要是加锁粒度上的不同。HashTable的加锁方法是给每个方法加上synchronized关键字，这样锁住的是整个Table对象。而ConcurrentHashMap是更细粒度的加锁
> 在JDK1.8之前，ConcurrentHashMap加的是分段锁，也就是Segment锁，每个Segment含有整个table的一部分，这样不同分段之间的并发操作就互不影响
> JDK1.8对此做了进一步的改进，它取消了Segment字段，直接在table元素上加锁，实现对每一行进行加锁，进一步减小了并发冲突的概率
> 
> 2. CopyOnWriteArrayList和CopyOnWriteArraySet
> 它们是加了写锁的ArrayList和ArraySet，锁住的是整个对象，但读操作可以并发执行
> 
> 3. 除此之外还有ConcurrentSkipListMap、ConcurrentSkipListSet、ConcurrentLinkedQueue、ConcurrentLinkedDeque等，至于为什么没有ConcurrentArrayList，原因是无法设计一个通用的而且可以规避ArrayList的并发瓶颈的线程安全的集合类，只能锁住整个list，这用Collections里的包装类就能办到

Collection是最基本的集合接口，一个Collection代表一组Object，即Collection的元素（Elements）。一些Collection允许相同的元素而另一些不行。一些能排序而另一些不行。Java SDK不提供直接继承自Collection的类，Java SDK提供的类都是继承自Collection的"子接口"，如List和Set。

所有实现Collection接口的类都必须提供两个标准的构造函数：无参数的构造函数用于创建一个空的Collection，有一个Collection参数的构造函数用于创建一个新的Collection，这个新的Collection与传入的Collection有相同的元素。后一个构造函数允许用户复制一个Collection。

如何遍历Collection中的每一个元素？不论Collection的实际类型如何，它都支持一个iterator()的方法，该方法返回一个迭代子，使用该迭代子即可逐一访问Collection中每一个元素。典型的用法如下：

```java
Iterator it = collection.iterator(); // 获得一个迭代子
　　　　while(it.hasNext()) {
　　　　　　Object obj = it.next(); // 得到下一个元素
　　　　}
```

**<font color="red">由Collection接口派生的两个接口是List和Set。</font>**
### 4.1.1 List接口
<font color="red">**List是有序的Collection，使用此接口能够精确的控制每个元素插入的位置。用户能够使用索引（元素在List中的位置，类似于数组下标）来访问List中的元素，这类似于Java的数组。**</font>

和下面要提到的Set不同，**List允许有相同的元素**。

除了具有Collection接口必备的iterator()方法外，List还提供一个listIterator()方法，返回一个ListIterator接口，和标准的Iterator接口相比，ListIterator多了一些add()之类的方法，允许添加，删除，设定元素，还能向前或向后遍历。

实现List接口的常用类有LinkedList，ArrayList，Vector和Stack。

#### 4.1.1.1 LinkedList类
<font color="red">LinkedList实现了List接口，允许null元素。LinkenList底层采用了双向链表来存储数据，每个节点都存储着上一个节点和下一个节点的地址以及本节点的数据。</font>此外LinkedList提供额外的get，remove，insert方法在LinkedList的首部或尾部。这些操作使LinkedList可被用作堆栈（stack），队列（queue）或双向队列（deque）。

注意：**LinkedList没有同步方法**。如果多个线程同时访问一个List，则必须自己实现访问同步。一种解决方法是在创建List时构造一个同步的List：
　　
```java
List list = Collections.synchronizedList(new LinkedList(...));
```

#### 4.1.1.2 ArrayList类
<font color="red">ArrayList实现了可变大小的数组。它允许所有元素，包括null。ArrayList底层采用动态数组的存储方式，便利效率非常高，ArrayList是线程不安全的。</font>

**size，isEmpty，get，set方法运行时间为常数**。但是add方法开销为分摊的常数，添加n个元素需要O(n)的时间。其他的方法运行时间为线性。

　　每个ArrayList实例都有一个容量（Capacity），即用于存储元素的数组的大小。这个容量可随着不断添加新元素而自动增加，但是增长算法并没有定义。当需要插入大量元素时，在插入前可以调用ensureCapacity方法来增加ArrayList的容量以提高插入效率。

**<font color="red">ArrayList和LinkedList的区别</font>**：
1. ArrayList是实现了基于动态数组的数据结构，LinkedList基于链表的数据结构。 
2. 对于**随机访问get和set，ArrayList优于**LinkedList，因为LinkedList要移动指针。 
3. 对于新增和删除操作**add和remove，LinkedList比较占优势**，因为ArrayList要移动数据。

#### 4.1.1.3 Vector类
<font color="red">Vector非常类似ArrayList，但是**Vector是同步的**。</font>由Vector创建的Iterator，虽然和ArrayList创建的Iterator是同一接口，但是，因为Vector是同步的，当一个Iterator被创建而且正在被使用，另一个线程改变了Vector的状态（例如，添加或删除了一些元素），这时调用Iterator的方法时将抛出ConcurrentModificationException，因此必须捕获该异常。
#### 4.1.1.4 Stack 类
Stack继承自Vector，实现一个后进先出的堆栈。Stack提供5个额外的方法使得Vector得以被当作堆栈使用。基本的push和pop方法，还有peek方法得到栈顶的元素，empty方法测试堆栈是否为空，search方法检测一个元素在堆栈中的位置。Stack刚创建后是空栈。

### 4.1.2 Set接口
**<font color="red">Set是一种无序的并且不包含重复的元素的Collection</font>**，即任意的两个元素e1和e2都有e1.equals(e2)=false，**<font color="red">Set最多有一个null元素</font>**。

很明显，Set的构造函数有一个约束条件，传入的Collection参数不能包含重复的元素。
#### 4.1.2.1 HashSet类
是通过哈希表实现的,**HashSet中的数据是无序的**，可以放入null，但**只能放入一个null**，两者中的值都不能重复，因为HashSet的底层实现是HashMap，但是HashSet只使用了HashMap的key来存取数据所以HashSet存的数据不能重复。 

<font color="red">HashSet要求放入的对象必须实现HashCode()方法，放入的对象，是以hashcode码作为标识的，而具有相同内容的 String对象，hashcode是一样，所以放入的内容不能重复。但是同一个类的对象可以放入不同的实例。</font>

## 4.2 Map接口
**<font color="red">Map没有继承Collection接口</font>**，Map提供key到value的映射。**<font color="red">一个Map中不能包含相同的key，每个key只能映射一个value</font>**。Map接口提供3种集合的视图，Map的内容可以被当作一组key集合，一组value集合，或者一组key-value映射。

### 4.2.1 Hashtable类
Hashtable继承Map接口，实现一个key-value映射的哈希表。**<font color="red">任何非空（non-null）的对象都可作为key或者value。</font>**

添加数据使用put(key, value)，取出数据使用get(key)，这两个基本操作的时间开销为常数。

使用Hashtable的简单示例如下，将1，2，3放到Hashtable中，他们的key分别是”one”，”two”，”three”：

```java
Hashtable numbers = new Hashtable();
numbers.put(“one”, new Integer(1));
numbers.put(“two”, new Integer(2));
numbers.put(“three”, new Integer(3));
```

要取出一个数，比如2，用相应的key：

```java
Integer n = (Integer)numbers.get(“two”);
System.out.println(“two = ” + n);
```

由于作为key的对象将通过计算其散列函数来确定与之对应的value的位置，因此任何作为key的对象都必须实现hashCode和equals方法。hashCode和equals方法继承自根类Object，如果你用自定义的类当作key的话，要相当小心，按照散列函数的定义，如果两个对象相同，即obj1.equals(obj2)=true，则它们的hashCode必须相同，但如果两个对象不同，则它们的hashCode不一定不同，如果两个不同对象的hashCode相同，这种现象称为冲突，冲突会导致操作哈希表的时间开销增大，所以尽量定义好的hashCode()方法，能加快哈希表的操作。

如果相同的对象有不同的hashCode，对哈希表的操作会出现意想不到的结果（期待的get方法返回null），要避免这种问题，只需要牢记一条：**<font color="red">要同时复写equals方法和hashCode方法，而不要只写其中一个。</font>**

**<font style="background-color: yellow;color: red;">Hashtable是同步的。</font>**

### 4.2.2 HashMap类
<font color="red">HashMap和Hashtable类似，不同之处在于HashMap是非同步的，并且允许null，即null value和null key</font>，但是将HashMap视为Collection时（values()方法可返回Collection）。HashMap实际上是一个“链表散列”的数据结构，即<font style="background-color: yellow;color: red;">**数组+链表**</font>的结合体，HashMap无序，LinkedHashMap 有序（<font color="blue">由哈希表结构保证数据唯一性，由链表保证存入和取出的顺序一致性</font>）。

HashMap由数组+链表组成的，**数组是HashMap的主体，链表则是主要为了解决哈希冲突而存在的**，如果定位到的数组位置不含链表（当前entry的next指向null）,那么对于查找，添加等操作很快，仅需一次寻址即可；如果定位到的数组包含链表，对于添加操作，其时间复杂度依然为O(1)，因为最新的Entry会插入链表头部，急需要简单改变引用链即可，而对于查找操作来讲，此时就需要遍历链表，然后通过key对象的equals方法逐一比对查找。所以，性能考虑，HashMap中的链表出现越少，性能才会越好。
### 4.2.3 WeakHashMap类
WeakHashMap是一种改进的HashMap，它对key实行“弱引用”，如果一个key不再被外部所引用，那么该key可以被GC回收。

总结

1. <font style="color:red;">如果涉及到堆栈，队列等操作，应该考虑用List，对于需要快速插入，删除元素，应该使用LinkedList，如果需要快速随机访问元素，应该使用ArrayList。</font>
2. <font style="color:red;">如果程序在单线程环境中，或者访问仅仅在一个线程中进行，考虑非同步的类，其效率较高，如果多个线程可能同时操作一个类，应该使用同步的类。</font>
3.  <font style="color:red;">要特别注意对哈希表的操作，作为key的对象要正确复写equals和hashCode方法。</font>
4.  <font style="color:red;">尽量返回接口而非实际的类型，如返回List而非ArrayList，这样如果以后需要将ArrayList换成LinkedList时，客户端代码不用改变。这就是针对抽象编程。</font>