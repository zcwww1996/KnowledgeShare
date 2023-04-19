[TOC]

来源：https://www.jianshu.com/p/193f590ea42b?utm_source=oschina-app

参考：<br>
https://blog.csdn.net/suifeng629/article/details/82179996

https://mp.weixin.qq.com/s/sWXxJeEyrPQVSN0lkviWug

HashMap是基于哈希表的Map接口的非同步实现。此实现提供所有可选的映射操作，并允许使用null值和null键。此类不保证映射的顺序，特别是它不保证该顺序恒久不变。

# HashMap的数据结构
在 JDK 1.7 中 HashMap 是以 “**数组加链表**” 的形式组成的，JDK 1.8 之后新增了 “**红黑树**” 的组成结构，“**当链表长度大于 8 并且 hash 桶的容量大于 64 时，链表结构会转换成红黑树结构**”。所以，它的组成结构如下图所示：

JDK1.8红黑树实现的哈希表<br>
[![HashMap底层数据结构](https://hbimg.huabanimg.com/fd66660db126c5a18f20947ac94e9f3a54ed58c114bc2-KshWo7 "JDK1.8红黑树实现的哈希表")](https://www.z4a.net/images/2021/04/26/HashMap.png)

> - JDK 1.8 之所以添加红黑树，是因为一旦链表过长，会严重影响 HashMap 的性能，而红黑树具有**快速增删改查**的特点，这样就可以有效的解决链表过长时操作比较慢的问题。
jdk1.8的最大区别就在于底层的红黑树策略。
> - 红黑树的时间复杂度为O(logn)。如果是链表,时间复杂度一定是O(n)。

HashMap 中数组的每一个元素又称为哈希桶，也就是 key-value 这样的实例。在 Java7 中叫 Entry，Java8 中叫 Node。

因为它本身所有的位置都为 null，在 put 插入的时候会根据 key 的 hash 去计算一个 index 值。比如，我 put （"狗哥"，666），在 HashMap 中插入 "狗哥" 这个元素，通过 hash 算法算出 index 位置是 3。这时的结构如下所示，还是个数组。

[![初次put](https://z3.ax1x.com/2021/04/26/gSIWNV.png)](https://www.z4a.net/images/2021/04/26/18d3dbd0be8784454.png)

以上没 hash 冲突时，若发生 hash 冲突就得引入链表啦。假设我再次 put （"阿狗"，666），在 HashMap 中插入 "阿狗" 这个元素，通过 hash 算法算出 index 位置也是 3。这时的结构如下所示：形成了链表。

[![再次put](https://z3.ax1x.com/2021/04/26/gSIRA0.png)](https://www.z4a.net/images/2021/04/26/2246e05a9479c4d6f.png)

HashMap重要概念<br>
![HashMap重要概念](https://ftp.bmp.ovh/imgs/2019/10/13ac5884d3f060ca.png 'HashMap重要概念')


在Java编程语言中，最基本的结构就是两种，一个是数组，另外一个是模拟指针（引用），所有的数据结构都可以用这两个基本结构来构造的，HashMap也不例外。<font color="red">HashMap实际上是一个“链表散列”的数据结构，即数组和链表的结合体。</font>

Node的源码如下所示：“可以看出每个哈希桶中包含了四个字段：**hash、key、value、next，其中 next 表示链表的下一个节点**”。
```java
static class Node < K, V > implements Map.Entry < K, V > {
    final int hash;
    final K key;
    V value;
    Node < K,
    V > next;
    
    ...
}
```
可以看出，Node就是数组中的元素，每个 Map.Entry 其实就是一个key-value对，它持有一个指向下一个元素的引用，这就构成了链表。

# HashMap的存取实现
## 存储put

```java
public V put(K key, V value) {
    // 对 key 进行哈希操作
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
    boolean evict) {
    Node < K, V > [] tab;
    Node < K, V > p;
    int n, i;
    // 哈希表为空则创建表
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 根据 key 的哈希值计算出要插入的数组索引 i
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 如果 table[i] 等于 null，则直接插入
        tab[i] = newNode(hash, key, value, null);
    else {
        Node < K, V > e;
        K k;
        // 如果 key 已经存在了，直接覆盖 value
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果 key 不存在，判断是否为红黑树
        else if (p instanceof TreeNode)
            // 红黑树直接插入键值对
            e = ((TreeNode < K, V > ) p).putTreeVal(this, tab, hash, key, value);
        else {
            // 为链表结构，循环准备插入
            for (int binCount = 0;; ++binCount) {
                // 下一个元素为空时
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 转换为红黑树进行处理
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //  key 已经存在直接覆盖 value
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // existing mapping for key
        if (e != null) { 
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 超过最大容量，扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

**put 流程**

[![hashmap_put.jpg](https://i.loli.net/2021/04/27/XWGvSCyrtzEsQ1u.jpg "put 流程")](https://c-ssl.duitang.com/uploads/blog/202104/27/20210427101944_fc3cf.jpeg)


从上面的源代码中可以看出：当我们往HashMap中put元素的时候，先根据key的hashCode重新计算hash值，根据hash值得到这个元素在数组中的位置（即下标），  <font color="red">如果数组该位置上已经存放有其他元素了，那么在这个位置上的元素将以链表的形式存放，新加入的放在链头，最先加入的放在链尾。如果数组该位置上没有元素，就直接将该元素放到此数组中的该位置上。</font>



## 读取get

```java
public V get(Object key) {
    Node < K, V > e;
    // 对 key 进行哈希操作
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node < K, V > getNode(int hash, Object key) {
    Node < K, V > [] tab;
    Node < K, V > first, e;
    int n;
    K k;
    // 非空判断
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 判断第一个元素是否是要查询的元素
        // always check first node
        if (first.hash == hash && 
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 下一个节点非空判断
        if ((e = first.next) != null) {
            // 如果第一节点是树结构，则使用 getTreeNode 直接获取相应的数据
            if (first instanceof TreeNode)
                return ((TreeNode < K, V > ) first).getTreeNode(hash, key);
            do { // 非树结构，循环节点判断
                // hash 相等并且 key 相同，则返回此节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

有了上面存储时的hash算法作为基础，理解起来这段代码就很容易了。从上面的源代码中可以看出：从HashMap中get元素时，首先计算key的hashCode，找到数组中对应位置的某一元素，然后通过key的equals方法在对应位置的链表/红黑树中找到需要的元素。

### 归纳
  <font color="red">**简单地说，HashMap 在底层将 key-value 当成一个整体进行处理，这个整体就是一个 Node 对象。HashMap 底层采用一个 Node[] 数组来保存所有的 key-value 对，当需要存储一个 Node 对象时，会根据hash算法来决定其在数组中的存储位置，在根据equals方法决定其在该数组位置上的链表中的存储位置；当需要取出一个Entry时，
也会根据hash算法找到其在数组中的存储位置，再根据equals方法从该位置上的链表中取出该Entry**。</font>

## 扩容 resize
当HashMap中的元素越来越多的时候，hash冲突的几率也就越来越高，因为数组的长度是固定的。所以为了提高查询的效率，就要对HashMap的数组进行扩容，数组扩容这个操作也会出现在ArrayList中，这是一个常用的操作

```java
final Node < K, V > [] resize() {
    // 扩容前的数组
    Node < K, V > [] oldTab = table;
    // 扩容前的数组的大小和阈值
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    // 预定义新数组的大小和阈值
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 超过最大值就不再扩容了
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩大容量为当前容量的两倍，但不能超过 MAXIMUM_CAPACITY
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
            oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 当前数组没有数据，使用初始化的值
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else { // zero initial threshold signifies using defaults
        // 如果初始化的值为 0，则使用默认的初始化容量
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果新的容量等于 0
    if (newThr == 0) {
        float ft = (float) newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float) MAXIMUM_CAPACITY ?
            (int) ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({
        "rawtypes",
        "unchecked"
    })
    Node < K, V > [] newTab = (Node < K, V > []) new Node[newCap];
    // 开始扩容，将新的容量赋值给 table
    table = newTab;
    // 原数据不为空，将原数据复制到新 table 中
    if (oldTab != null) {
        // 根据容量循环数组，复制非空元素到新 table
        for (int j = 0; j < oldCap; ++j) {
            Node < K, V > e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果链表只有一个，则进行直接赋值
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 红黑树相关的操作
                    ((TreeNode < K, V > ) e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 链表复制，JDK 1.8 扩容优化部分
                    // 如果节点不为空，且为单链表，则将原数组中单链表元素进行拆分
                    Node < K, V > loHead = null, loTail = null;//保存在原有索引的链表
                    Node < K, V > hiHead = null, hiTail = null;//保存在新索引的链表
                    Node < K, V > next;
                    do {
                        next = e.next;
                        // 哈希值和原数组长度进行 & 操作，为 0 则在原数组的索引位置
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引 + oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 将原索引放到哈希桶中
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 将原索引 + oldCap 放到哈希桶中
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

从以上源码可以看出，扩容主要分两步：
- **扩容**：创建一个新的 Entry 空数组，长度是原数组的 2 倍。
- **位运算**：原来的元素哈希值和原数组长度进行 & 运算。

JDK 1.8 在扩容时并没有像 JDK 1.7 那样，重新计算每个元素的哈希值，而是通过高位运算`(e.hash & oldCap)`来确定元素是否需要移动，它有两种结果，一个是0，一个是oldCap。

假设 key1 的信息如下：
- key1.hash = 10；二进制：0000 1010
- oldCap = 16；二进制：0001 0000

> **补充：Java 位运算(移位、位与、或、异或、非）**
> 
> ```java
> 位与(&)：第一个操作数的的第n位于第二个操作数的第n位如果都是1，那么结果的第n为也为1
>
> 位或(|)：第一个操作数的的第n位于第二个操作数的第n位 只要有一个是1，那么结果的第n为也为1，否则为0
>
> 位异或(^)：第一个操作数的的第n位于第二个操作数的第n位 相反，那么结果的第n为也为1，否则为0
>
> 位非(~)：操作数的第n位为1，那么结果的第n位为0，反之。
> ```


**“使用 e.hash & oldCap 得到的结果，高一位为 0，当结果为 0 时表示元素在扩容时位置不会发生任何变化”**，而假设 key 2 信息如下：
- key2.hash = 17；二进制：0001 0001
- oldCap = 16；二进制：0001 0000

**“这时候得到的结果，高一位为 1，当结果为 1 时，表示元素在扩容时位置发生了变化，新的下标位置等于原下标位置 + 原数组长度”**，如下图所示：key2、kry4 虚线为移动的位置。

[![扩容.png](https://i.loli.net/2021/04/27/QuMk4eTmaZBFsr2.png "扩容")](https://c-ssl.duitang.com/uploads/blog/202104/27/20210427110345_c1737.png)



# HashMap的性能参数

## 属性

```Java
// HashMap 初始化长度
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// HashMap 最大长度
static final int MAXIMUM_CAPACITY = 1 << 30; // 1073741824

// 默认的加载因子 (扩容因子)
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 当链表长度大于此值且数组长度大于 64 时，会从链表转成红黑树
static final int TREEIFY_THRESHOLD = 8;

// 转换链表的临界值，当元素小于此值时，会将红黑树结构转换成链表结构
static final int UNTREEIFY_THRESHOLD = 6;

// 最小树容量
static final int MIN_TREEIFY_CAPACITY = 64;
```

## 构造器
HashMap 包含如下几个构造器：

1. HashMap()：构建一个初始容量为 16，负载因子为 0.75 的 HashMap。
2. HashMap(int initialCapacity)：构建一个初始容量为 initialCapacity，负载因子为 0.75 的 HashMap。
3. HashMap(int initialCapacity, float loadFactor)：以指定初始容量、指定的负载因子创建一个 HashMap。

HashMap的基础构造器HashMap(int initialCapacity, float loadFactor)带有两个参数，它们是初始容量initialCapacity和负载因子loadFactor。

负载因子loadFactor衡量的是一个散列表的空间的使用程度，负载因子越大表示散列表的装填程度越高，反之愈小。对于使用链表法的散列表来说，查找一个元素的平均时间是O(1+a)，因此如果负载因子越大，对空间的利用更充分，然而后果是查找效率的降低；如果负载因子太小，那么散列表的数据将过于稀疏，对空间造成严重浪费。

HashMap的实现中，通过threshold字段来判断HashMap的最大容量：

`threshold = (int)(capacity * loadFactor);`

结合负载因子的定义公式可知，threshold就是在此loadFactor和capacity对应下允许的最大元素数目，超过这个数目就重新resize，以降低实际的负载因子。默认的的负载因子0.75是对空间和时间效率的一个平衡选择。当容量超出此最大容量时， resize后的HashMap容量是容量的两倍：

# HashMap常见疑惑
  <font color="red">我们知道java.util.HashMap不是线程安全的，因此如果在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓fail-fast策略。</font>


## 1、为什么 HashMap 的初始化长度是 16 ？

前面说过，从 Key 映射到 HashMap 数组的对应位置，会用到一个 Hash 函数，比如：index = Hash ("狗哥")

注意到 HashMap 初始化长度用的是 `1<<4`，而不是直接写 16。这是为啥呢？其实这样是为了位运算的方便，**“位与运算比算数计算的效率高太多了，之所以选择 16，是为了服务将 Key 映射到 index 的算法”**。

那如何实现一个尽量均匀分布的 Hash 函数呢？从而减少 HashMap 碰撞呢？没错，就是通过 Key 的 HashCode 值来做位运算。

有公式（Length 是 HashMap 的长度）：**“HashCode（Key） & （Length- 1）”**

我举个例子，key 为 "book" 的十进制为 3029737 那二进制就是 101110001110101110 1001 HashMap 长度是默认的 16，length - 1 的结果。十进制 : 15；二进制 : 1111

把以上两个结果做与运算：101110001110101110 1001 & 1111 = 1001；1001 的十进制 = 9, 所以 index=9。

也就是说：**“hash 算法最终得到的 index 结果，取决于 hashcode 值的最后几位”**

> 你可以试试把长度指定为 10 以及其他非 2 次幂的数字，做位运算。发现 index 出现相同的概率大大升高。而长度 16 或者其他 2 的幂，length - 1 的值是所有二进制位全为 1, 这种情况下，index 的结果等同于 hashcode 后几位的值，只要输入的 hashcode 本身分布均匀，hash 算法的结果就是均匀的


**“所以，HashMap 的默认长度为 16，是为了降低 hash 碰撞的几率”**。

## 2、为什么树化是 8，退树化是 6？

红黑树平均查找长度为 log (n)，长度为 8 时，查找长度为 3，而链表平均查找长度为 n/2；也就是 8 除以 2；查找长度链表大于树，转化为树，效率更高。

当为 6 时，树：2.6；链表：3。链表 > 树。这时理应也还是树化，但是树化需要时间，为了这点效率牺牲时间是不划算的。

## 3、什么是加载因子？加载因子为什么是 0.75 ？

前面说了扩容机制。那什么时候扩容呢？这就取决于原数组长度和加载因子两个因素了。

 
> 加载因子也叫扩容因子或负载因子，用来判断什么时候进行扩容的，假如加载因子是 0.5，HashMap 的初始化容量是 16，那么当 HashMap 中有 16\*0.5=8 个元素时，HashMap 就会进行扩容。


那加载因子为什么是 0.75 而不是 0.5 或者 1.0 呢？这其实是出于容量和性能之间平衡的结果：

- 上面说到，为了提升扩容效率，HashMap 的容量（capacity）有一个固定的要求，那就是一定是 2 的幂。所以，如果负载因子是 3/4 的话，那么和 capacity 的乘积结果就可以是一个整数
- 当加载因子设置较大时，扩容门槛提高，扩容发生频率低，占用的空间小，但此时发生 Hash 冲突的几率就会提升，因此需要更复杂的数据结构来存储元素，这样对元素的操作时间就会增加，运行效率也会降低；
- 当加载因子设置较小时，扩容门槛降低，会占用更多的空间，此时元素的存储就比较稀疏，发生哈希冲突的可能性就比较小，因此操作性能会比较高。
    

**“所以综合了以上情况，就取了一个 0.5 到 1.0 的平均数 0.75，作为加载因子”**。

## 3、HashMap 是线程安全的么？

不是，因为 get 和 put 方法都没有上锁。**“多线程操作无法保证：此刻 put 的值，片刻后 get 还是相同的值，会造成线程安全问题”**。

还有个 HashTable 是线程安全的，但是加锁的粒度太大。并发度很低，最多同时允许一个线程访问，性能不高。一般我们使用 java.util.concurrent包下面的ConcurrentHashMap

> - ConcurrentHashMap在 jdk1.8 其中抛弃了 jdk1.7 原有的 Segment 分段锁，而采用了 CAS + synchronized 来保证并发安全性
> - ConcurrentHashMap 的 get 方法是非常高效的，因为整个过程都不需要加锁


> CAS（Compare and swap）比较和替换是设计并发算法时用到的一种技术。简单来说，比较和替换是使用一个期望值和一个变量的当前值进行比较，如果当前变量的值与我们期望的值相等，就使用一个新值替换当前变量的值。
## 4、为什么重写 equals 方法的时，需要重写 hashCode 方法呢？

Java 中，所有的对象都是继承于 Object 类。Ojbect 类中有两个方法 equals、hashCode，这两个方法都是用来比较两个对象是否相等的。

先来看看 equals 方法：

```java
public boolean equals(Object obj) {  
    return (this == obj);  
}  
```

在未重写 equals 方法，他其实就是 == 。有以下两个特点：

- 对于值对象，== 比较的是两个对象的值

- 对于引用对象，== 比较的是两个对象的地址

看回 put 方法的源码：**“HashMap 是通过 key 的 hashcode 去寻找地址 index 的。如果 index 一样就会形成链表”**，也即是 "狗哥" 和 "阿狗" 是有可能在同一个位置上。

前面的 get 方法说过：**“当哈希冲突时我们不仅需要判断 hash 值，还需要通过判断 key 值是否相等，才能确认此元素是不是我们想要的元素”**。我们去 get 首先是找到 hash 值一样的，那怎么知道你想要的是那个对象呢？**“没错，就是利用 equals 方法”**，如果仅重写 hashCode 方法，不写 equals 方法，当发生 hash 冲突，hashcode 一样时，就不知道取哪个对象了。

## 5、HashMap 死循环分析

以下代码基于 JDK1.7 分析。这个问题，主要是 JDK1.7 的链表尾插法造成的。假设 HashMap 默认大小为 2，原本 HashMap 中没有一个元素。使用三个线程：t1、t2、t3 添加元素 key1，key2，key3。我在扩容之前打了个断点，让三个线程都停在这里。源码如下：


```Java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry < K, V > e: table) {
        while (null != e) {
            Entry < K, V > next = e.next; // 此处加断点
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```


假设 3 个元素 hash 冲突，放到同一个链表上。其中 key1→key2→key3 这样的顺序。没毛病，一切很正常。

尾插法<br>
[![尾插法](https://mmbiz.qpic.cn/mmbiz_png/D7p5icluLfVtOV0SadPYxnQPK7C25qTI5Y80ha1WNiadmX46BUy68rrBqO8ibqibriaWzfiariaWG2metMEF6I2w4y8rg/640?wx_fmt=png "尾插法")](https://c-ssl.duitang.com/uploads/blog/202104/27/20210427114236_894b5.png)


此时放开断点，HashMap 扩容。就有可能变成这样：原来是 key1→key2→key3。很不幸扩容之后，key1 和 key2 还是在同一个位置，这时形成链表，如果 key2 比 key1 后面插入，根据头插法。此时就变成 key2→key1

链表反转<br>
[![链表反转](https://mmbiz.qpic.cn/mmbiz_png/D7p5icluLfVtOV0SadPYxnQPK7C25qTI5pkImh8AIg9Qlkf32UtylKibP5SAGGFatWPlnMicbKT0MnK6XKVhbrz7g/640?wx_fmt=png "链表反转")](https://c-ssl.duitang.com/uploads/blog/202104/27/20210427114236_6f63f.png)


最终 3 个线程都调整完毕，就会出现下图所示的死循环：这个时候 get (key1) 就会出现 Infinite Loop 异常。

循环引用<br>
[![循环引用](https://mmbiz.qpic.cn/mmbiz_png/D7p5icluLfVtOV0SadPYxnQPK7C25qTI5c7uiaHhdRA25ljfWABNibSxZ0EyM0GPMGusedRncE6Eb1llM1ar0KyQg/640?wx_fmt=png "循环引用")](https://c-ssl.duitang.com/uploads/blog/202104/27/20210427114236_6f63f.png)



当然发生死循环的原因是 JDK 1.7 链表插入方式为首部倒序插入，这种方式在扩容时会改变链表节点之间的顺序。**“这个问题在 JDK 1.8 得到了改善，变成了尾部正序插入”**，在扩容时会保持链表元素原本的顺序，就不会出现链表成环的问题。

# HashMap的两种遍历方式
## 第一种（推荐）

```java
Map map = new HashMap();
　　Iterator iter = map.entrySet().iterator();
　　while (iter.hasNext()) {
　　Map.Entry entry = (Map.Entry) iter.next();
　　Object key = entry.getKey();
　　Object val = entry.getValue();
　　}
```

效率高,以后一定要使用此种方式！

## 第二种

```java
Map map = new HashMap();
　　Iterator iter = map.keySet().iterator();
　　while (iter.hasNext()) {
　　Object key = iter.next();
　　Object val = map.get(key);
　　}
```


效率低,以后尽量少使用！