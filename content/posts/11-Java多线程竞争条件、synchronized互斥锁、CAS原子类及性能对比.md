---
title: "Java多线程竞争条件、synchronized互斥锁、CAS原子类及性能对比"
date: 2023-07-26T14:03:58+08:00
draft: false
ShowToc: true
TocOpen: false
---

在[Oracle官网的Java并发教程](https://docs.oracle.com/javase/tutorial/essential/concurrency/interfere.html)中，线程同步问题的第一个案例就是Thread Interference，译为线程干涉、线程冲突或者线程竞争。

## 线程冲突Thread Interference / 竞争条件Race Condition

线程冲突Thread Interference 和竞争条件Race Condition描述的基本是同一个现象：
就是**多个线程对同一个资源操作时，操作结果不可控**的现象。

以下是一个最简单的线程冲突举例：

```java
public class a_ThreadInterference {
    private static class Counter {
        private int c = 0;
        public void increment() {
            c++;
        }
        public void decrement() {
            c--;
        }
        public int value() {
            return c;
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Counter ct=new Counter();
        Thread t1=new Thread(() ->{
            for(int i=0;i<100000000;i++){
                ct.increment();
            }
        });
        Thread t2=new Thread(() ->{
            for(int i=0;i<100000000;i++){
                ct.decrement();
            }
        });
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(ct.value()); // 结果是不确定的，不等于0，特别是在加减次数大于一万次之后，波动很大
    }
}
// 运行时间 55ms
```
在这段代码中两个线程一个负责加一个负责减，并且执行加减的次数是相同的，理想情况下输出结果应当是0，但实际上输出结果不可确定并且波动非常大。

**其中每个线程对ct的操作都分为三个步骤：读取，增减，写入，如果每次读取增减写入的这整个过程都是完整独立不可被打扰的，结果就将是正确的。**

而实际上两个线程在运行时，很可能同时对ct进行了读取，读到了同一个数，各自进行了增减，并各自写入了结果，而最终记录的结果只有一个，这就使得其中很多增减操作的效果作废了。

此处读取增减写入发生混乱的流程图可参考 [竞争条件维基中的示例一节](https://zh.wikipedia.org/zh-hans/%E7%AB%B6%E7%88%AD%E5%8D%B1%E5%AE%B3)



## 竞争条件的解决：互斥与原子性

在以上案例中，**每个线程对ct的操作都都分为三个步骤：读取，增减，写入，如果每次读取增减写入的这整个过程都是完整独立不可被打扰的，结果就将是正确的。**

其中某个操作完整独立不可被打扰的性质，可以被描述为 **原子性 Atomicity **
而各个线程在执行某段程序时不可相互干扰，同一时刻只有一个线程能执行，表示这多个线程之间存在着**互斥 Mutual Exclusive**的关系

所以根据以上两点，为了解决线程竞争，实现完整独立不可被打扰，就要从**操作的原子性**，和**线程之间的互斥关系**来入手。

**互斥 Mutual Exclusive** 可以通过锁，信号量等手段来实现。 互斥锁multex lock中mutex就是Mutual Exclusive的缩写。

**原子性 Atomicity** 可以通过JUC包提供的各种原子类来实现

首先讲互斥

多个线程对共享资源之间的访问必须是各自互斥的，
互斥 即每次只有一个线程能够对资源进行操作，也就是上文所述的**整个过程完整独立不可被打扰**。

实现互斥的方法之一是加锁，常用的有synchronized和ReentrantLock

### synchronized锁

使用jdk自带的synchronized关键字

synchronized的作用：**当一个线程访问某个类中被synchronized修饰的代码块时，其他试图访问此对象的线程都将被阻塞**

```java
public class a_ThreadInterference_Solution_Synchronized {
    private static class Counter {
        private int c = 0;
        public synchronized void increment() {
            c++;
        }
        public synchronized void decrement() {
            c--;
        }
        public synchronized int value() {
            return c;
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Counter ct=new Counter();
        Thread t1=new Thread(() ->{
            for(int i=0;i<100000000;i++){
                ct.increment();
            }
        });
        Thread t2=new Thread(() ->{
            for(int i=0;i<100000000;i++){
                ct.decrement();
            }
        });
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(ct.value()); 
    }
}
// 运行时间 4.5s
```

synchronized作用[Oriacle官方解释](https://docs.oracle.com/javase/tutorial/essential/concurrency/syncmeth.html)
> First, it is not possible for two invocations of synchronized methods on the same object to interleave. When one thread is executing a synchronized method for an object, all other threads that invoke synchronized methods for the same object block (suspend execution) until the first thread is done with the object.
> Second, when a synchronized method exits, it automatically establishes a happens-before relationship with any subsequent invocation of a synchronized method for the same object. This guarantees that changes to the state of the object are visible to all threads.

简言之synchronized作用就是 一保持了互斥，仅有一个线程能访问；二是与之后的所有操作建立了happens-before关系，其操作结果对其他所有线程可见(这同时也确保了[Memory Consistency](https://docs.oracle.com/javase/tutorial/essential/concurrency/memconsist.html))

[synchronized的底层通过monitorenter和monitorexit两个指令来实现同步](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.monitorenter)

而monitorenter和monitorexit，是[java内部锁Intrinsic Lock(又称monitor，中文语境下常以监视器代称内部锁)的一部分](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)

关于Intrinsic Lock以及synchronized实现，还有ReentrantLock和synchronized之间的异同，能力有限留待之后讲解。


### ReentrantLock可重入锁

首先是ReentrantLock这个名称，Reentrant为可重入的，[Oriacle对可重入的解释如下](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)：
一个线程不能获取已被另一个线程占有的锁，但其能获取其已自己已经获取的锁，一个线程获取自己已经获取的锁便称为重入


synchronized和ReentrantLock的原理是相似的，都是锁，只不过synchronized是java内部关键字，不需要手动catch exception，而ReentrantLock是JUC包，java.util.concurrent.locks包实现的。

synchronized是非常简洁便利的，由编译器自动保证去加锁或者释放锁，进入synchronized代码块就加锁，离开就释放锁；而ReentrantLock需要手动释放，为避免遗忘一般在finally中写入ReentrantLock的释放过程。


```java
import java.util.concurrent.locks.ReentrantLock;

public class a_ThreadInterference_Solution_ReentrantLock {
    private static class Counter {
        private int c = 0;
        public void increment() {
            c++;
        }
        public void decrement() {
            c--;
        }
        public int value() {
            return c;
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Counter ct=new Counter();
        ReentrantLock l1=new ReentrantLock();
        Thread t1=new Thread(() ->{
            for(int i=0;i<100000000;i++){
                l1.lock();
                try {
                    ct.increment();
                }finally {
                    l1.unlock();
                }
            }
        });
        Thread t2=new Thread(() ->{
            for(int i=0;i<100000000;i++){
                l1.lock();
                try {
                    ct.decrement();
                }finally {
                    l1.unlock();
                }
            }
        });
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(ct.value());
    }
}
// 运行时间 5.5s
```

### AtomicInteger 线程安全的原子类

关于原子性或者[原子访问](https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html) ，在Oriacle官方文档中定义如下：

> In programming, an atomic action is one that effectively happens all at once. An atomic action cannot stop in the middle: it either happens completely, or it doesn't happen at all. No side effects of an atomic action are visible until the action is complete.

即原子操作被看作一个整体发生，不可打断，要么完全发生要么完全不发生，并且操作结果只有在结束后才能被外界看到。

在Java中，有些读写操作是具有原子性的。包括：
* 对引用类型的读写和对大多数原始类型的读写(除long和double之外)是有原子性的
* 所有声明为volatile的变量的读写是具有原子性的

在本文涉及的案例中，`a++`或者`a--`这样的代码，是不具有原子性的，因为`a++`实际上设计了读取、加一和写入三段操作。

为了让`a++`或者`a--`这样的操作具有原子性，JUC有AtomicInteger类，利用CAS机器指令提供了原子性自增和自减的方法`incrementAndGet()`和`decrementAndGet()`。

```java
import java.util.concurrent.atomic.AtomicInteger;

public class a_ThreadInterference_Solution_AtomicInteger {
    private static class Counter {
        private AtomicInteger c = new AtomicInteger(0);
        public void increment() {
            c.incrementAndGet();
        }
        public void decrement() {
            c.decrementAndGet();;
        }
        public int value() {
            return c.get();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        Counter ct=new Counter();
        Thread t1=new Thread(() ->{
            for(int i=0;i<100000000;i++){
                ct.increment();
            }
        });
        Thread t2=new Thread(() ->{
            for(int i=0;i<100000000;i++){
                ct.decrement();
            }
        });
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(ct.value());
    }
}
// 运行时间 2.6s
```



## 性能对比：Mutual exclusion Vs Atomic variable

在以上的四段代码末尾都标注了用IDEA自带的Prifile跑出的时间，
其中不加锁的时间最短，只要0.05s
加synchronized锁后耗时4.5s
加ReentrantLock锁后耗时5.5s
使用AtomicInteger耗时2.6s

可以看到在简单的自增自减操作本身上消耗的时间是非常少的，只有50ms。大量的时间放在了锁的获取与管理上，从火焰图中也能看到，耗时越长的，火焰图堆叠层数越多越复杂，并且主要开销放在了加锁解锁上。

在三种解决线程竞争的方法中，耗时由多到少分别为ReentrantLock，synchronized，AtomicInteger

其中synchronized耗时比ReentrantLock耗时少很好理解，毕竟synchronized是JDK级别的，是java自带的关键字，其锁释放由jvm自动进行管理；而ReentrantLock是JUC包的一个类，第三方库的性能略低于原生指令是很好理解的。

但同样是来自JUC包的AtomicInteger原子类确是耗时是最少的，这引起了笔者的不解。

参考Stackoverflow此回答：[Mutual exclusion Vs Atomic variable](https://stackoverflow.com/a/47608179)
以及 CSDN[互斥锁与CAS的开销对比](https://blog.csdn.net/weixin_48460141/article/details/123944815)

二者的性能差距在于synchronized是阻塞同步的形式；而AtomicInteger使用的是MISC中的CAS Compare and Swap指令，是CPU级别的非阻塞同步的指令。

### AtomicInteger实现

这个AtomicInteger类，观察其实现代码可看到是调用了`sun.misc.unsafe`来实现的，由misc的包名可以看出是CPU指令集级别的操作，是偏操作系统底层的。
（注：MISC Minimal Insturction Set Computer，相关有RISC，CISC）

```java
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
```
其中的`incrementAndGet()`方法，最终是依赖于`sun.misc.unsafe`包中的`compareAndSwapInt()`函数
```java
    /**
     * Atomically update Java variable to <tt>x</tt> if it is currently
     * holding <tt>expected</tt>.
     * @return <tt>true</tt> if successful
     */
    public final native boolean compareAndSwapInt(Object o, long offset,int expected,int x);
```
这个`compareAndSwapInt()`函数，是java native方法，也就是java调用其他语言例如汇编语言实现的方法，是高速且底层的机器指令，就像普通读写操作一样，具有原子性，并且在Java级别不涉及线程切换和锁管理。

### 阻塞同步和非阻塞同步

其中synchronized使用的是阻塞同步方法，即多个线程争夺互斥资源，持有锁的线程继续运行，其他线程进入阻塞状态。

阻塞同步的开销是较大的，因为主流的JVM采用1：1模型，其阻塞的是内核线程，需要JVM向操作系统申请进行用户态到内核态的切换以实现线程之间的运行阻塞等操作。并且现成的切换需要对线程运行环境进行保存恢复等操作，带来很大的开销。

而AtomicInterger原子类，使用unsafe包中CAS函数，对应的是实现CAS操作的机器指令，CAS的原理实际上就是非阻塞同步，是乐观锁，该指令包含三个操作数，即保存在线程工作内存的预期值A，要更新的新值B，以及主内存当中的实际值C；执行更新操作时将A与C进行比对，若A与C一致，则将B刷到主存，更新成功；否则，则更新失败，可通过循环的方式继续尝试更新。由于比较并更新的操作是使用原子性的机器指令完成的，这个过程不会被打断，因此是线程安全的。

CAS的开销是比较小的，因为一直处于用户态，线程状态不变，不需要向操作系统申请阻塞线程，不涉及系统调用，不进入内核态。


在线程数少的情况下，自旋cas所带来的资源消耗与阻塞同步相比是微不足道的，因此可以使用cas+自旋的乐观锁模式来进行同步 

在线程数多的情况下，自旋cas所带来的cpu资源消耗可能是巨额的，多条线程不停地自旋会使系统压力骤增，此时使用互斥同步可能会合算些


这个StackOverflow回答[Unsafe compareAndSwapInt vs synchronize](https://stackoverflow.com/questions/34944212/unsafe-compareandswapint-vs-synchronize)中也阐述了synchronized和CAS的适用情况
>Using synchronised is more efficient if you expect to be waiting a long time (e.g. milli-seconds) as the thread can fall asleep and release the CPU to do other work.
> Using compareAndSwap is more efficient if you expect the operation to happen quite quickly. This is because it is a simple machine code instruction and take as little as 10 ns. However if a resources is heavily contented this instruction must busy wait and if it cannot obtain the value it needs, it can consume the CPU busily until it does.

即synchronized适用于线程切换不频繁，执行任务时间较长，并且线程数量多的时候；
CAS适用于执行任务简单，时间短，线程数少的情况，如果大量线程中同时使用CAS，自旋的开销是巨大的。

回到三种解决竞争条件的方法对比上来：
耗时由多到少分别为ReentrantLock，synchronized，AtomicInteger，也分别对应着外部JUC包实现、JDK自带实现和直接调用CPU指令实现之间的速度关系，对于简单的任务，越接近底层，速度越快。



## 总结与延申

在以上的问题中，线程干涉/竞争/冲突 Thread Interference 和 竞争条件 Race Condition 实际上描述的是同一个问题：即多个线程同时访问同一块资源，导致执行结果不可控。
解决这个问题的方法是：实现互斥、保证操作原子性
实现方法是：锁机制
即使用锁机制来实现互斥和原子性，并以此来避免竞争条件，保障内存一致性。

多线程编程的核心问题：互斥与同步

在Oracle的阐述中，Java中一切的同步问题都是基于内部锁Intrinsic Lock或称Monitor Lock实现的，而[内部锁的功能被描述为](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)：
> Intrinsic locks play a role in both aspects of synchronization: enforcing exclusive access to an object's state and establishing happens-before relationships that are essential to visibility.

其中的核心就是 exclusive access和 happens-before relationship

对临界资源的exclusive access即线程间的互斥，保证了不会相互干涉
而各个操作之间的happens-before relationship正确建立，即可保证多线程程序的结果是稳定可控的。

其中将happens-before relationship描述为essential to visibility，各操作的结果清晰可见，即保证各线程在同一时刻到的资源是一致的，也就实现了Memory Consistency，此处详见[内存一致性和happens before关系](https://docs.oracle.com/javase/tutorial/essential/concurrency/memconsist.html)

内存一致性模型也是有趣的问题，详见[内存一致性模型笔记 (Memory Consistency Model)](https://blog.csdn.net/qq_29328443/article/details/107616795)


### 延申：由多线程联想到ACID原则和分布式系统CAP theorem

在多线程编程中，对内存访问的原子性Atomicity、和内存内容对各线程之间的一致性Memory Consistency，

很容易让人联想到数据库设计的ACID原则，
即：Atomicity Consisitency Isolation Durability

以及分布式系统设计中的CAP难题，
即 Consistency Availability PartitionTolerance 三选二


多线程编程中的原子操作Atomicity和数据库中的Atomicity一样，操作是原子化的，只有成功和失败，没有中间态。

多线程编程中的Memory Consistency和分布式系统设计中的Consistency是相近的，而与数据库设计中的Consistency不同。
多线程编程和分布式系统中的Consistency都指的是同一份数据在某一时刻对各个线程(各个节点)看到的内容是相同的。
而数据库的Consistency的限定范围更广，包括了事务执行前后逻辑上的一致性，即在数据库中指数据库应当处于一个合理的状态，即前后增减有序，事务执行前后数据库的状态具有一致性，700块被分给几个人并经过无数次转账后，总额仍然得是700块。

数据库中的Isolation指的是并发执行的各个事务之间不能互相干扰，即一个事务内部的操作及使用的数据，对并发的其他事务是隔离的。此属性确保并发执行一系列事务的效果等同于以某种顺序串行地执行它们，也就是要达到这么一种效果：对于任意两个并发的事务T1和T2，在事务T1看来，T2要么在T1开始之前就已经结束，要么在T1结束之后才开始，这样每个事务都感觉不到有其他事务在并发地执行。此性质与多线程编程中的happens-before关系是相似的。

关于CAS原理,内存一致性模型，悲观锁乐观锁与自旋锁,synchronized的monitor底层实现，以及锁之间的对比，JVM对操作系统发出系统调用的过程等等，仍有无限可写，留待以后。

## 参考

[竞争条件及其过程示例图](https://zh.wikipedia.org/zh-hans/%E7%AB%B6%E7%88%AD%E5%8D%B1%E5%AE%B3)

Oracle官网Java并发编程教程：
[线程干涉 Thread Interference](https://docs.oracle.com/javase/tutorial/essential/concurrency/interfere.html)
[内存一致性 Memory Consistency](https://docs.oracle.com/javase/tutorial/essential/concurrency/memconsist.html)
[内部锁，同步与可重入Intrinsic Locks and Synchronization](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)
[原子访问Atomic Access](https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html) 

synchronized底层：
[monitorenter和monitorexit](https://docs.oracle.com/javase/specs/jvms/se11/html/jvms-6.html#jvms-6.5.monitorenter)

synchronized与CAS对比：
[Mutual exclusion Vs Atomic variable](https://stackoverflow.com/a/47608179)
[互斥锁与CAS的开销对比](https://blog.csdn.net/weixin_48460141/article/details/123944815)
[Unsafe compareAndSwapInt vs synchronize](https://stackoverflow.com/questions/34944212/unsafe-compareandswapint-vs-synchronize)

[内存一致性模型笔记 (Memory Consistency Model)](https://blog.csdn.net/qq_29328443/article/details/107616795)
