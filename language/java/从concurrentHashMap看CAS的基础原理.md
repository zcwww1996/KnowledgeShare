[TOC]

来源：https://blog.csdn.net/weixin_42636552/article/details/82383272

今天在看concurrentHashMap的时候，知道在jdk1.8之后，这个集合的并发控制使用Synchronized和CAS来操作，所以就想找博客看看CAS。  
看到这篇文章，感觉还不错，加上自己的些许理解，转载在这里。  
[https://blog.csdn.net/mmoren/article/details/79185862](https://blog.csdn.net/mmoren/article/details/79185862)

本篇的思路是先阐明无锁执行者CAS的核心算法原理然后分析Java执行CAS的实践者Unsafe类，该类中的方法都是native修饰的，因此我们会以说明方法作用为主介绍Unsafe类，最后再介绍并发包中的Atomic系统使用CAS原理实现的并发类。

# 无锁的概念


在谈论无锁概念时，总会关联起乐观派与悲观派，对于乐观派而言，他们认为事情总会往好的方向发展，总是认为坏的情况发生的概率特别小，可以无所顾忌地做事，但对于悲观派而已，他们总会认为发展事态如果不及时控制，以后就无法挽回了，即使无法挽回的局面几乎不可能发生。  
这两种派系映射到并发编程中就如同加锁与无锁的策略，即加锁是一种悲观策略，无锁是一种乐观策略，因为对于加锁的并发程序来说，它们总是认为每次访问共享资源时总会发生冲突，因此必须对每一次数据操作实施加锁策略。  
而无锁则总是假设对共享资源的访问没有冲突，线程可以不停执行，无需加锁，无需等待，一旦发现冲突，无锁策略则采用一种称为CAS的技术来保证线程执行的安全性，这项CAS技术就是无锁策略实现的关键，下面我们进一步了解CAS技术的奇妙之处。

# 无锁的执行者-CAS

## 一、介绍CAS

CAS的全称是Compare And Swap 即比较交换，其算法核心思想如下

```
执行函数：CAS(V,E,N)

```

其包含3个参数

```
V表示要更新的变量

E表示预期值

N表示新值

```

_如果V值等于E值，则将V的值设为N。若V值和E值不同，则说明已经有其他线程做了更新，则当前线程什么都不做。_  
通俗的理解就是CAS操作需要我们提供一个期望值，当期望值与当前线程的变量值相同时，说明还没线程修改该值，当前线程可以进行修改，也就是执行CAS操作，但如果期望值与当前线程不符，则说明该值已被其他线程修改，此时不执行更新操作，但可以选择重新读取该变量再尝试再次修改该变量，也可以放弃操作。  
原理图如下：  
![](https://img-blog.csdn.net/20180904131304221)
  
由于CAS操作属于乐观派，它总认为自己可以成功完成操作，当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作，这点从图中也可以看出来。基于这样的原理，CAS操作即使没有锁，同样知道其他线程对共享资源操作影响，并执行相应的处理措施。_**同时从这点也可以看出，由于无锁操作中没有锁的存在，因此不可能出现死锁的情况，也就是说无锁操作天生免疫死锁。**_

## 二、CPU指令对CAS的支持

或许我们可能会有这样的疑问，假设存在多个线程执行CAS操作并且CAS的步骤很多，有没有可能在判断V和E相同后，正要赋值时，切换了线程，更改了值。造成了数据不一致呢？答案是否定的，因为CAS是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，并且原语的执行必须是连续的，在执行过程中不允许被中断，也就是说,**CAS是一条CPU的原子指令，不会造成所谓的数据不一致问题。**

## 三、鲜为人知的指针: Unsafe类

Unsafe类存在于sun.misc包中，其内部方法操作可以像C的指针一样直接操作内存，单从名称看来就可以知道该类是非安全的，毕竟Unsafe拥有着类似于C的指针操作，因此总是不应该首先使用Unsafe类，Java官方也不建议直接使用的Unsafe类，但我们还是很有必要了解该类，因为Java中CAS操作的执行依赖于Unsafe类的方法，注意Unsafe类中的所有方法都是native修饰的，也就是说Unsafe类中的方法都直接调用操作系统底层资源执行相应任务，关于Unsafe类的主要功能点如下：

内存管理，Unsafe类中存在直接操作内存的方法；

```java
//分配内存指定大小的内存
public native long allocateMemory(long bytes);

//根据给定的内存地址address设置重新分配指定大小的内存
public native long reallocateMemory(long address, long bytes);

//用于释放allocateMemory和reallocateMemory申请的内存
public native void freeMemory(long address);

//将指定对象的给定offset偏移量内存块中的所有字节设置为固定值
public native void setMemory(Object o, long offset, long bytes, byte value);

//设置给定内存地址的值
public native void putAddress(long address, long x);

//获取指定内存地址的值
public native long getAddress(long address);

//设置给定内存地址的long值
public native void putLong(long address, long x);

//获取指定内存地址的long值
public native long getLong(long address);

//设置或获取指定内存的byte值

//其他基本数据类型(long,char,float,double,short等)的操作与putByte及getByte相同

public native byte getByte(long address);

public native void putByte(long address, byte x);

//操作系统的内存页大小
public native int pageSize();

提供实例对象新途径：
//传入一个对象的class并创建该实例对象，但不会调用构造方法

public native Object allocateInstance(Class cls) throws InstantiationException;

/**类和实例对象以及变量的操作，就不贴出来了
传入Field f，获取字段f在实例对象中的偏移量
获得给定对象偏移量上的int值，所谓的偏移量可以简单理解为指针指向该变量的内存地址，
通过偏移量便可得到该对象的变量
通过偏移量可以设置给定对象上偏移量的int值
获得给定对象偏移量上的引用类型的值
通过偏移量可以设置给定对象偏移量上的引用类型的值*/

```

虽然在Unsafe类中存在getUnsafe()方法，但该方法只提供给高级的Bootstrap类加载器使用，普通用户调用将抛出异常，所以我们在Demo中使用了反射技术获取了Unsafe实例对象并进行相关操作。

## 四、Unsafe里的CAS 操作相关

CAS是一些CPU直接支持的指令，也就是我们前面分析的无锁操作，在Java中无锁操作CAS基于以下3个方法实现，在稍后讲解Atomic系列内部方法是基于下述方法的实现的。

```java
//第一个参数o为给定对象，offset为对象内存的偏移量，通过这个偏移量迅速定位字段并设置或获取该字段的值，
//expected表示期望值，x表示要设置的值，下面3个方法都通过CAS原子指令执行操作。
public final native boolean compareAndSwapObject(Object o, long offset,Object expected, Object x);                                                                                                  
 
public final native boolean compareAndSwapInt(Object o, long offset,int expected,int x);
 
public final native boolean compareAndSwapLong(Object o, long offset,long expected,long x);

```

### 1、挂起与恢复  
将一个线程进行挂起是通过park方法实现的，调用 park后，线程将一直阻塞直到超时或者中断等条件出现。unpark可以终止一个挂起的线程，使其恢复正常。

Java对线程的挂起操作被封装在 LockSupport类中，LockSupport类中有各种版本pack方法，其底层实现最终还是使用Unsafe.park()方法和Unsafe.unpark()方法

```
//线程调用该方法，线程将一直阻塞直到超时，或者是中断条件出现。  
public native void park(boolean isAbsolute, long time);  
 
//终止挂起的线程，恢复正常.java.util.concurrent包中挂起操作都是在LockSupport类实现的，其底层正是使用这两个方法，  
public native void unpark(Object thread); 

```

### 2、并发包中的原子操作类(Atomic系列)

通过前面的分析我们已基本理解了无锁CAS的原理并对Java中的指针类Unsafe类有了比较全面的认识，下面进一步分析CAS在Java中的应用，即并发包中的原子操作类(Atomic系列)，从JDK 1.5开始提供了java.util.concurrent.atomic包，在该包中提供了许多基于CAS实现的原子操作类，用法方便，性能高效，主要分以下4种类型。

### 3、原子更新基本类型

原子更新基本类型主要包括3个类：

```
AtomicBoolean：原子更新布尔类型
AtomicInteger：原子更新整型
AtomicLong：原子更新长整型

```

这3个类的实现原理和使用方式几乎是一样的，这里我们以AtomicInteger为例进行分析，AtomicInteger主要是针对int类型的数据执行原子操作，它提供了原子自增方法、原子自减方法以及原子赋值方法等，鉴于AtomicInteger的源码不多，我们直接看源码

```java
 public class AtomicInteger extends Number implements java.io.Serializable {
        private static final long serialVersionUID = 6214790243416807050L;
     
        // 获取指针类Unsafe
        private static final Unsafe unsafe = Unsafe.getUnsafe();
     
        //下述变量value在AtomicInteger实例对象内的内存偏移量
        private static final long valueOffset;
     
        static {
            try {
               //通过unsafe类的objectFieldOffset()方法，获取value变量在对象内存中的偏移
               //通过该偏移量valueOffset，unsafe类的内部方法可以获取到变量value对其进行取值或赋值操作
                valueOffset = unsafe.objectFieldOffset
                    (AtomicInteger.class.getDeclaredField("value"));
            } catch (Exception ex) { throw new Error(ex); }
        }
       //当前AtomicInteger封装的int变量value
        private volatile int value;
     
        public AtomicInteger(int initialValue) {
            value = initialValue;
        }
        public AtomicInteger() {
        }
       //获取当前最新值，
        public final int get() {
            return value;
        }
        //设置当前值，具备volatile效果，方法用final修饰是为了更进一步的保证线程安全。
        public final void set(int newValue) {
            value = newValue;
        }
        //最终会设置成newValue，使用该方法后可能导致其他线程在之后的一小段时间内可以获取到旧值，有点类似于延迟加载
        public final void lazySet(int newValue) {
            unsafe.putOrderedInt(this, valueOffset, newValue);
        }
       //设置新值并获取旧值，底层调用的是CAS操作即unsafe.compareAndSwapInt()方法
        public final int getAndSet(int newValue) {
            return unsafe.getAndSetInt(this, valueOffset, newValue);
        }
       //如果当前值为expect，则设置为update(当前值指的是value变量)
        public final boolean compareAndSet(int expect, int update) {
            return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
        }
        //当前值加1返回旧值，底层CAS操作
        public final int getAndIncrement() {
            return unsafe.getAndAddInt(this, valueOffset, 1);
        }
        //当前值减1，返回旧值，底层CAS操作
        public final int getAndDecrement() {
            return unsafe.getAndAddInt(this, valueOffset, -1);
        }
       //当前值增加delta，返回旧值，底层CAS操作
        public final int getAndAdd(int delta) {
            return unsafe.getAndAddInt(this, valueOffset, delta);
        }
        //当前值加1，返回新值，底层CAS操作
        public final int incrementAndGet() {
            return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
        }
        //当前值减1，返回新值，底层CAS操作
        public final int decrementAndGet() {
            return unsafe.getAndAddInt(this, valueOffset, -1) - 1;
        }
       //当前值增加delta，返回新值，底层CAS操作
        public final int addAndGet(int delta) {
            return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
        }
       //省略一些不常用的方法....
    }

```

通过上述的分析，可以发现AtomicInteger原子类的内部几乎是基于前面分析过Unsafe类中的CAS相关操作的方法实现的，这也同时证明AtomicInteger是基于无锁实现的，这里重点分析自增操作实现过程，其他方法自增实现原理一样。

我们发现AtomicInteger类中所有自增或自减的方法都间接调用Unsafe类中的getAndAddInt()方法实现了CAS操作，从而保证了线程安全，关于getAndAddInt其实前面已分析过，它是Unsafe类中1.8新增的方法，源码如下

```java
    //Unsafe类中的getAndAddInt方法
    public final int getAndAddInt(Object o, long offset, int delta) {
            int v;
            do {
                v = getIntVolatile(o, offset);
            } while (!compareAndSwapInt(o, offset, v, v + delta));
            return v;
        }

```

可看出getAndAddInt通过一个while循环不断的重试更新要设置的值，直到成功为止，调用的是Unsafe类中的compareAndSwapInt方法，是一个CAS操作方法。这里需要注意的是，上述源码分析是基于JDK1.8的，如果是1.8之前的方法，AtomicInteger源码实现有所不同，是基于for死循环的，如下

```java
  //JDK 1.7的源码，由for的死循环实现，并且直接在AtomicInteger实现该方法，
    //JDK1.8后，该方法实现已移动到Unsafe类中，直接调用getAndAddInt方法即可
    public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
    }

```

## CAS的ABA问题及其解决方案

假设这样一种场景，当第一个线程执行CAS(V,E,U)操作，在获取到当前变量V，准备修改为新值U前，另外两个线程已连续修改了两次变量V的值，使得该值又恢复为旧值，这样的话，我们就无法正确判断这个变量是否已被修改过，如下图  
![](https://img-blog.csdn.net/20180904143127511)
  
这就是典型的CAS的ABA问题，一般情况这种情况发现的概率比较小，可能发生了也不会造成什么问题，比如说我们对某个做加减法，不关心数字的过程，那么发生ABA问题也没啥关系。但是在某些情况下还是需要防止的，那么该如何解决呢？在Java中解决ABA问题，我们可以使用以下两个原子类

### AtomicStampedReference类

AtomicStampedReference原子类是一个带有时间戳的对象引用，在每次修改后，AtomicStampedReference不仅会设置新值而且还会记录更改的时间。当AtomicStampedReference设置对象值时，对象值以及时间戳都必须满足期望值才能写入成功，这也就解决了反复读写时，无法预知值是否已被修改的窘境

底层实现为： 通过Pair私有内部类存储数据和时间戳, 并构造volatile修饰的私有实例

接着看AtomicStampedReference类的compareAndSet（）方法的实现：

同时对当前数据和当前时间进行比较，只有两者都相等是才会执行casPair()方法，

单从该方法的名称就可知是一个CAS方法，最终调用的还是Unsafe类中的compareAndSwapObject方法

到这我们就很清晰AtomicStampedReference的内部实现思想了，

通过一个键值对Pair存储数据和时间戳，在更新时对数据和时间戳进行比较，

只有两者都符合预期才会调用Unsafe的compareAndSwapObject方法执行数值和时间戳替换，也就避免了ABA的问题。

### AtomicMarkableReference类

AtomicMarkableReference与AtomicStampedReference不同的是，

AtomicMarkableReference维护的是一个boolean值的标识，也就是说至于true和false两种切换状态，

经过博主测试，这种方式并不能完全防止ABA问题的发生，只能减少ABA问题发生的概率。

AtomicMarkableReference的实现原理与AtomicStampedReference类似，这里不再介绍。到此，我们也明白了如果要完全杜绝ABA问题的发生，我们应该使用AtomicStampedReference原子类更新对象，而对于AtomicMarkableReference来说只能减少ABA问题的发生概率，并不能杜绝。

## 再谈自旋锁

自旋锁是一种假设在不久将来，当前的线程可以获得锁，因此虚拟机会让当前想要获取锁的线程做几个空循环(这也是称为自旋的原因)，在经过若干次循环后，如果得到锁，

就顺利进入临界区。如果还不能获得锁，那就会将线程在操作系统层面挂起，这种方式确实也是可以提升效率的。但问题是当线程越来越多竞争很激烈时，

占用CPU的时间变长会导致性能急剧下降，因此Java虚拟机内部一般对于自旋锁有一定的次数限制，可能是50或者100次循环后就放弃，直接挂起线程，让出CPU资源。

如下通过AtomicReference可实现简单的自旋锁。

```java
 public class SpinLock {
      private AtomicReference<Thread> sign =new AtomicReference<>();
     
      public void lock(){
        Thread current = Thread.currentThread();
        while(!sign .compareAndSet(null, current)){
        }
      }
     
      public void unlock (){
        Thread current = Thread.currentThread();
        sign .compareAndSet(current, null);
      }
    }

```

使用CAS原子操作作为底层实现，lock()方法将要更新的值设置为当前线程，并将预期值设置为null。unlock()函数将要更新的值设置为null，并预期值设置为当前线程。然后我们通过lock()和unlock来控制自旋锁的开启与关闭，注意这是一种非公平锁。  
事实上AtomicInteger(或者AtomicLong)原子类内部的CAS操作也是通过不断的自循环(while循环)实现，不过这种循环的结束条件是线程成功更新对于的值，但也是自旋锁的一种。
