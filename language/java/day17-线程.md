[TOC]

线程是程序中的执行线程。Java 虚拟机允许应用程序并发地运行多个执行线程


---
补充开始
# 1. 线程、进程及其联系与区别
## 1.1 进程
### 1.1.1 进程的概念
  进程是操作系统实现并发执行的重要手段，也是操作系统为程序提供的重要运行环境抽象。
  进程最根本的属性是动态性和并发性。以下是从不同角度对进程的解释：
1. 进程是程序的一次执行
2. 进程是可以与其他计算并发执行的计算
3. 进程是一个程序程序及其数据在处理器上顺序执行时发生的活动
4. 进程是程序在一个数据集合上的运行过程，是系统进行资源分配和调度的一个独立单位
5. 进程是进程实体的一次活动
6. 进程是支持程序运行的机制

### 1.1.2 进程的定义
   进程是具有一定功能的程序在一个数据结合上的运行过程，它是系统进行资源分配和调度管理的一个可并发执行的基本单位。
### 1.1.3 进程的基本特性
1. **动态性**：进程的实质是程序的一次执行过程，它由系统创建而产生，能够被调度而执行，因申请的共享资源被其他进程占用而暂停，完成任务后被撤销。动态性是进程最重要的特性。
2. **独立性**：系统内多个进程可以并发执行，引入进程的目的也是为了使系统某个程序能够和其他进程并发执行。
3. **异步性**：进程由于共享资源和协同合作，因此产生了相互制约的关系，进程实体通过进程管理以异步的方式使用处理器和其他资源，系统必须统一调度，依据一定的算法来保证各个进程能够协同运行并共享处理器和其他资源。
4. **结构特性**：系统中运行的进程实体通常由程序、数据和一个PCB（进程控制块）组成。

### 1.1.4 进程的基本状态和转换
[![进程的基本状态和转换](https://images2015.cnblogs.com/blog/1143814/201706/1143814-20170608150555262-2027458817.png "进程的基本状态和转换")](https://images2015.cnblogs.com/blog/1143814/201706/1143814-20170608150555262-2027458817.png "进程的基本状态和转换")
上图中的运行——>阻塞中的事件请求即等待资源、事件
阻塞——>就绪中的事件发生即事件发生，资源释放。
## 1.2 线程
### 1.2.1 线程的概念
线程是进程中实施调度和分派的基本单位。
 
操作系统提供现成的目的就是为了方便高效地实现并发处理（进一步提高并发度）。
### 1.2.2 线程分类
 线程一般分为用户级线程和核心级线程。
### 1.2.3 线程池
设计思想：在创建一个进程是，相应地创建若干个线程，将它们放在一个缓冲池中，这些线程在等待工作。当服务器接收到一个请求时，系统就唤醒其中的一个线程，并将请求传给它，由这个线程进行服务。当完成任务后，线程重新被放入线程池中，等待下面新的请求和服务。如果线程池中没有可用的线程，服务器就要等待，直到有一个线程被释放。
## 1.3 线程和进程的关系
1. 一个进程可以有多个线程，但至少有一个线程；而一个线程只能在一个进程的地址空间内活动。
2. 资源分配给进程，同一个进程的所有线程共享该进程所有资源。
3. CPU分配给线程，即真正在处理器运行的是线程。
4. 线程在执行过程中需要协作同步，不同进程的线程间要利用消息通信的办法实现同步。

**注：进程是最基本的资源拥有单位和调度单位。**
进程间通信方式：（1）消息传递（2）共享存储（3）管道通信

---
补充结束

## 1.4 问题补充
- (1)为什么要重写run()方法

因为线程启动之后，会默认执行run方法；可以将我们想在子线程运行代码放在run方法里面
- (2)启动线程使用的是哪个方法

start
- (3)线程~~能~~**不能**多次启动

<font style="background-color: Yellow
;">**不能**</font>
- (4)run()和start()方法的区别

t.run();//这个不是在子线程中执行的，仅仅相当于使用对象调用方法，还是运行在主线程中

t.start();//这个方法内部会先启动一个子线程，然后在子线程中调用run方法，run方法运行在子线程中
- (5)改变子线程名称

1. 使用public final void setName(String name)改变线程名称，使之与参数 name 相同
2. 使用构造方法。public Thread(String name)分配新的 Thread 对象。
- (6)获取主线程的对象

public static Thread currentThread()    返回对当前正在执行的线程对象的引用

# 2. 创建线程的方法
Java使用Thread类代表线程，所有的线程对象都必须是Thread类或其子类的实例。Java可以用三种方式来创建线程，如下所示：

1) 继承Thread类创建线程
2) 实现Runnable接口创建线程
3) 使用Callable和Future创建线程
4) 使用线程池创建线程(项目最常用)

下面让我们分别来看看这三种创建线程的方法。
## 2.1 将类声明为 Thread 的子类
通过继承Thread类来创建并启动多线程的一般步骤如下

1】定义Thread类的子类，并重写该类的**run()** 方法，该方法的方法体就是线程需要完成的任务，run()方法也称为线程执行体。

2】创建Thread子类的实例，也就是创建了线程对象

3】启动线程，即调用线程的**start()** 方法
```java
public class ThreadDemo {
    public static void main(String[] args) {
        MyThread t = new MyThread("子线程");
        //t.run();//这个不是在子线程中执行的，仅仅相当于使用对象调用方法，还是运行在主线程中
        t.start();//这个方法内部会先启动一个子线程，然后在子线程中调用run方法，run方法运行在子线程中
        
        //public final void setName(String name)改变线程名称，使之与参数 name 相同
        //t.setName("子线程");
        
        //public final String getName()返回该线程的名称
        System.out.println(t.getName());//Thread-0
        
        //主线程的名称，先获取主线程的对象
        //public static Thread currentThread()返回对当前正在执行的线程对象的引用
        Thread mainThread = Thread.currentThread();
        System.out.println(mainThread.getName());//main
        
        System.out.println("over");
    }
}
// 一种方法是将类声明为 Thread 的子类。该子类应重写 Thread 类的 run 方法。接下来可以分配并启动该子类的实例
class MyThread extends Thread {
    public MyThread(String threadName) {
        //调用父类的有参构造方法
        super(threadName);
    }
    @Override
    public void run() {
        // 这里面就是我们自己要在子线程做的事情
        for (int i = 0; i < 10; i++) {
            System.out.println(i);
        }
    }
}
```

**继承了Thread的类无法多次start()**，会出现`java.lang.IllegalThreadStateException`错误

###  2.1.1 线程阻塞
1. join()这个方法正被哪个线程调用，哪个线程暂停，等待调用这个线程实例执行完毕之后再执行。
代码第7行， t.join()被主线程调用，主线程暂停等待，t线程插队执行完，主线程再继续执行
2. sleep()让当前正在执行的线程休眠。

```java
public class ThreadDemo2 {
    public static void main(String[] args) {
        MyThread2 t = new MyThread2();
        t.start();
        // public final void join()这个方法被哪个线程实例调用，那么执行该方法的线程就会等到这个线程实例执行完毕之后再执行
        /*try {
            t.join();//当前的主线程就会等到t线程执行完毕之后再执行
        } catch (InterruptedException e) {
            e.printStackTrace();
        }*/
        
        // public static void yield()暂停当前正在执行的线程对象，并执行其他线程
        //Thread mainThread = Thread.currentThread();
        //mainThread.yield();
        System.out.println("over");
    }
}
class MyThread2 extends Thread {
    @Override
    public void run() {
        /*try {
            // public static void sleep(long millis)在指定的毫秒数内让当前正在执行的线程休眠
            sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }*/
        for (int i = 0; i < 10; i++) {
            System.out.println(i);
        }
    }
}
```
### 2.1.2 中断线程
// public final void stop()----stop()已过期
```java
public class ThreadDemo3 {
    public static void main(String[] args) {
        MyThread3 t = new MyThread3();
        t.start();
        System.out.println("over");
    }
}
class MyThread3 extends Thread {
    private boolean flag = true;
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            // 当前i等于3的时候，结束子线程
            // public final void stop()
            if (i == 3) {
                flag = false;
            }
            if (flag) {
                System.out.println(i);
            } else {
                break;
            }
        }
    }
}
```
### 2.1.3 清除阻塞
```java
public class ThreadDemo4 {
    public static void main(String[] args) {
        //把主线程对象传递给子线程
        MyThread4 t = new MyThread4(Thread.currentThread());
        t.start();
        
        try {
            t.join();//主线程就进入阻塞状态
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        /*try {
            //等待t线程一定是阻塞状态
            Thread.sleep(200);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // public void interrupt()清除当前线程的等待状态并且线程还会收到一个InterruptedException
        t.interrupt();
        */
        System.out.println("over");
    }
}
class MyThread4 extends Thread {
    private Thread mainThread;
    public MyThread4(Thread mainThread) {
        this.mainThread = mainThread;
    }
    @Override
    public void run() {
        try {
            sleep(200);//为了主线程一定是阻塞状态
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }
        
        mainThread.interrupt();
        
        try {
            sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        for (int i = 0; i < 10; i++) {
            System.out.println(i);
        }
    }
}
```
## 2.2 声明实现Runnable 接口的类（推荐）
通过实现Runnable接口创建并启动线程一般步骤如下：

1】定义Runnable接口的实现类，一样要重写run()方法，这个run（）方法和Thread中的run()方法一样是线程的执行体

2】创建Runnable实现类的实例，并用这个实例作为Thread的target来创建Thread对象，这个Thread对象才是真正的线程对象

3】第三部依然是通过调用线程对象的start()方法来启动线程
```java
/*
public Thread( Runnable target)  分配新的 Thread 对象。
参数：target - 其 run 方法被调用的对象
*/
public class ThreadDemo5 {
    public static void main(String[] args) {
        // 实现了数据和线程的有效分离
        MyRunnable task = new MyRunnable();
        
        Thread t = new Thread(task);
        t.start();
        Thread t2 = new Thread(task);
        t2.start();
    }
}
// 创建线程的另一种方法是声明实现 Runnable 接口的类。该类然后实现 run 方法。然后可以分配该类的实例，在创建 Thread
// 时作为一个参数来传递并启动
class MyRunnable implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            System.out.println(i);
        }
    }
}
```
## 2.3 使用Callable和Future创建线程
和Runnable接口不一样，**Callable接口提供了一个call()方法作为线程执行体，call()方法比run()方法功能要强大**。

》**call()方法可以有返回值**

》**call()方法可以声明抛出异常**



介绍了相关的概念之后，创建并启动有返回值的线程的步骤如下：

1】创建Callable接口的实现类，并实现call()方法，然后创建该实现类的实例（从java8开始可以直接使用Lambda表达式创建Callable对象）。

2】使用FutureTask类来包装Callable对象，该FutureTask对象封装了Callable对象的call()方法的返回值

3】使用FutureTask对象作为Thread对象的target创建并启动线程（因为FutureTask实现了Runnable接口）

4】调用FutureTask对象的get()方法来获得子线程执行结束后的返回值

代码实例：

```java
public class MyThread3 implements Callable {//实现Callable接口
　　  @Override
      public String call() throws Exception {
          System.out.println("new Thread 3");//输出:new Thread 3
          return "111";
      }

public class Main {
　　public static void main(String[] args){
　　　MyThread3 th=new MyThread3();
     FutureTask<Integer> future=new FutureTask<Integer>(th);
//　　　//使用Lambda表达式创建Callable对象
//　　   //使用FutureTask类来包装Callable对象
//    
//　　　FutureTask<Integer> future=new FutureTask<Integer>(
//　　　　(Callable<Integer>)()->{
//　　　　　　return 111;
//　　　　}
//　　  );
　　　new Thread(future,"有返回值的线程").start();//实质上还是以Callable对象来创建并启动线程
　　  try{
　　　　System.out.println("子线程的返回值："+future.get());//get()方法会阻塞，直到子线程执行结束才返回
 　　 }catch(Exception e){
　　　　ex.printStackTrace();
　　　}
　　}
}
```
## 2.4 三种创建线程方法对比

实现Runnable和实现Callable接口的方式基本相同，不过是后者执行call()方法有返回值，后者线程执行体run()方法无返回值，因此可以把这两种方式归为一种。这种方式与继承Thread类的方法之间的差别如下：

1. 线程只是实现Runnable或实现Callable接口，还可以继承其他类。

2. 这种方式下，多个线程可以共享一个target对象，非常适合多线程处理同一份资源的情形。

3. 但是编程稍微复杂，如果需要访问当前线程，必须调用Thread.currentThread()方法。

4. 继承Thread类的线程类不能再继承其他父类（Java单继承决定）。

注：<font color="red">一般推荐采用实现接口的方式来创建多线程</font>
## 2.5 线程池
```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolDemo {
    public static void main(String[] args) {
        // public static ExecutorService
        // newCachedThreadPool()创建返回一个可存活60s的线程的线程池
        ExecutorService threadPool = Executors.newCachedThreadPool();
        
        //public static ExecutorService newFixedThreadPool(int nThreads)创建一个可重用固定线程数的线程池
        threadPool = Executors.newFixedThreadPool(1);
        
        //public static ExecutorService newSingleThreadExecutor()创建一个使用单个 worker 线程的 Executor
        threadPool = Executors.newSingleThreadExecutor();
        
        // void execute(Runnable command)在未来某个时间执行给定的命令
        MyTask task = new MyTask();
        //threadPool.execute(task);
        ThreadPoolUtils.execute(task);
        //new Thread(task).start();
        
        //public void shutdown()按过去执行已提交任务的顺序发起一个有序的关闭，但是不接受新任务
        //threadPool.shutdown();
        ThreadPoolUtils.shutdown();
    }
}

class MyTask implements Runnable {
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            System.out.println(i);
        }
    }
}
```
### 2.4.1 线程池封装
```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolUtils {
    // 创建了一个线程池
    private static ExecutorService threadPool = Executors.newCachedThreadPool();

    /**
    * 将Runnable任务交给线程池执行
    * 
    * @param command
    */
    public static void execute(Runnable command) {
        threadPool.execute(command);
    }

    /**
    * 当程序退出的时候，调用此方法，关闭线程池
    */
    public static void shutdown() {
        // 先判断一下线程池是否已经被关闭
        if (!threadPool.isShutdown()) {
            threadPool.shutdown();
        }
    }
}
```
# 3. 线程案例 
## 3.1 模拟火车站售票
- 要求：春节某火车站正在售票，假设还剩100张票，而它有3个售票窗口售票，请设计一个程序模拟该火车站售票。

- 判断是否存存在线程安全问题依据？
    + 是否是多线程环境
    + 是否存在共享数据
    + 是否存在多条语句操作共享数据

- 多线程同步安全问题怎么解决呢？

(多线程同步安全问题:三个窗口同时显示出售100张票、出售97张票、出售94张票)

那就让程序没有多线程安全问题；让同一时刻只能有一个线程访问共享数据的代码

注意事项：锁对象：**多个线程使用的必须是同一个锁对象或者让锁对象保持唯一**

### 3.1.1 多线程同步安全问题
**问题原因**：

当多条语句在操作同一个线程共享数据时，一个线程对多条语句只执行了一部分，还没执行完，另一个线程参与进来执行，导致共享数据的错误。

**解决思想**：

对多条操作共享数据的语句，只能让一个线程都执行完，在执行过程中，其他线程不执行。

**解决办法**：

<font color="red">**锁对象必须保持唯一且一致性；**</font>

使用同步代码块；
#### 3.1.1.1 同步代码
    
    synchronized(锁对象){
    存在线程安全问题的代码
    }
    
    同步代码块锁对象：多个线程必须使用的是同一个锁对象或者说锁对象保持唯一性

#### 3.1.1.2 同步方法(默认的锁是this)

public synchronized void run(){....}
- 同步方法锁对象：this
- 静态同步方法锁对象：当前类的字节码文件对象

什么时候使用同步方法，什么时候使用同步代码块？
+ 当**整个方法的代码**都存在线程安全问题时**且可以使用this**作为锁对象时，使用**同步方法**
+ 当有**一部分代码**存在线程安全问题时，推荐使用**同步代码块**

#### 3.1.1.3 使用JDK1.5新特性
ReentrantLock对象的lock和unLock方法
    
    // 创建一把锁
    private Lock lock = new ReentrantLock()
     ……
     lock.lock();
    锁对象
    lock.unlock();
    
代码
```java
public class SaleTicketDemo {
    public static void main(String[] args) {
        //第一种方式：继承Thread的方式
        //创建三个窗口，也就是三个线程
        /*new SaleTicketThread("窗口1").start();
        new SaleTicketThread("窗口2").start();
        new SaleTicketThread("窗口3").start();*/
        
        //第二种方式：继承Runnable接口
        TicketTask ticketTask = new TicketTask();
        new Thread(ticketTask, "窗口1").start();
        new Thread(ticketTask, "窗口2").start();
        new Thread(ticketTask, "窗口3").start();
    }
}
```
第一种方法：继承Thread的方式
```java
public class SaleTicketThread extends Thread {
    //所有对象共享100张票
    private static int ticket = 100;
    public SaleTicketThread(String threadName) {
        super(threadName);
    }
    @Override
    public void run() {
        while(true){
            synchronized (Object.class) {
                if(ticket>0){
                    System.out.println(getName()+"正在出售第"+ticket+"张票");
                    ticket--;
                }else{
                    break;
                }
            }
            
            //现实生活中，卖票肯定是有一点延迟的
            try {
                sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
第二种方法：继承Runnable接口
```
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class TicketTask implements Runnable {
    private int ticket = 100;
    // 创建一把锁
    private Lock lock = new ReentrantLock();
    @Override
    public void run() {
        while (true) {
            // synchronized (Object.class) {
            // public void lock()获取锁
            lock.lock();
            if (ticket > 0) {
                System.out.println(Thread.currentThread().getName() + "正在出售第" + ticket + "张票");
                ticket--;
            } else {
                break;
            }
            // }
            // public void unlock()试图释放此锁
            lock.unlock();
            // 现实生活中，卖票肯定是有一点延迟的
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
#### 3.1.1.4 利用线程池完成任务
启动线程可以使用Executors创建线程池然后使用线程池完成任务；
## 3.2 乘客进站案例

在坐火车时，在火车没有到站或者时间没有到之前，所有人都是在排队的；工作人员一般都会在站内等着，到了进站的时间的时候，工作人员就会将乘客放行

有的地方是一个人一个人的检票放行，有的地方是一下会将所有的人都放行然后再查票

对应到java程序中：

通过上述描述，我们可以将进站设置条件为一个布尔类型的标记isOpen，isOpen初始状态为false，所以刚开始其他的人都等着排队(排队不是理想状态)

但是工作人员是有钥匙的所以可以随便改变这个标记可以随便进出

每一个线程就是一个乘客，当调用notify时，会随机选取一个等待的乘客进行放行，当调用notifyAll时，会将所有的等待乘客进行放行
    
**注意事项（synchronized(),wait,notify() 对象一致性[见WaitTest代码]）：wait，notify，notifyAll方法必须在同步代码中通过同一个锁对象进行调用**
    
- 为什么wait，notify，notifyAll是定义在Object中？
        锁对象可以是任意的对象，wait，notify，notifyAll只能通过锁对象调用；所以可以被任意对象调用的方法必须定义在Object类中
        
- 锁对象什么时候释放？
    + 当锁对象调用wait方法时
    + 当正常执行完同步代码时

代码
```
public class EnterTrainDemo {
    public static void main(String[] args) {
        new EnterTrainThread("张三").start();
        new EnterTrainThread("李四").start();
        new EnterTrainThread("王五").start();
        new EnterTrainThread("赵六").start();
        // 睡眠200毫秒，确保所有的乘客都进入了等待状态
        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 工作人员进站
        EnterTrainThread t = new EnterTrainThread("工作人员");
        t.setIsOpen(true);
        t.start();
    }
}
class EnterTrainThread extends Thread {
    private static boolean isOpen;
    public static void setIsOpen(boolean flag) {
        isOpen = flag;
    }
    public EnterTrainThread(String name) {
        super(name);
    }
    @Override
    public void run() {
        synchronized (Object.class) {
            if (isOpen) {
                isOpen = false;
                System.out.println(getName() + "进站了，3秒钟之后放行乘客~");
                try {
                    sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 唤醒单个线程：public final void notify()唤醒的线程具有随机性;唤醒在此对象监视器上等待的单个线程
                // Object.class.notify();
                // 唤醒所有线程：public final void notifyAll()唤醒的线程具有随机性;唤醒在此对象监视器上等待的所有线程
                Object.class.notifyAll();
            } else {
                // 乘客等待
                System.out.println(getName() + "正在等待进站~");
                try {
                    // 线程等待：public final void wait()
                    Object.class.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(getName() + "进站了~");
            }
        }
    }
}
```
synchronized(),wait,notify() 对象一致性
```
class ThreadA extends Thread{  
    public ThreadA(String name) {  
        super(name);  
    }  
    public void run() {  
        synchronized (this) {  
            try {                         
                Thread.sleep(1000); //  使当前线阻塞 1 s，确保主程序的 t1.wait(); 执行之后再执行 notify()  
            } catch (Exception e) {  
                e.printStackTrace();  
            }             
            System.out.println(Thread.currentThread().getName()+" call notify()");  
            // 唤醒当前的wait线程  
            this.notify();  
        }  
    }  
}  
public class WaitTest {  
    public static void main(String[] args) {  
        ThreadA t1 = new ThreadA("t1");  
        synchronized(t1) {  
            try {  
                // 启动“线程t1”  
                System.out.println(Thread.currentThread().getName()+" start t1");  
                t1.start();  
                // 主线程等待t1通过notify()唤醒。  
                System.out.println(Thread.currentThread().getName()+" wait()");  
                t1.wait();  //  不是使t1线程等待，而是当前执行wait的线程等待  
                System.out.println(Thread.currentThread().getName()+" continue");  
            } catch (InterruptedException e) {  
                e.printStackTrace();  
            }  
        }  
    }  
}  
```