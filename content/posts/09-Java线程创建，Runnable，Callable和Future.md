---
title: "Java线程创建，Runnable，Callable和Future"
date: 2023-07-24T18:21:00+08:00
draft: false
ShowToc: true
TocOpen: false
---

## 线程创建的几种方式

**创建线程实际上就是创建一个Thread类对象。**

#### 方式1 继承Thread类并重写run方法

从Thread派生一个自定义类，然后覆写run()方法：

```java
public class Main {
    public static void main(String[] args) {
        Thread t = new MyThread();
        t.start(); // 启动新线程
    }
}

class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("start new thread!");
    }
}
```


#### 方式2 实现Runnable接口，即实现run方法，而后以实现了runnable接口的实例为target传递到Thread中创建Thread对象。

java通过传入Runnable函数来创建线程实例
```java
public class Main {
    public static void main(String[] args) {
        Thread t = new Thread(new MyRunnable());
        t.start(); // 启动新线程
    }
}

class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("start new thread!");
    }
}
```



#### 方式3 实现Callable接口，实现call方法，通过FutureTask构造方法把callable传进去，通过FutureTask创建Thread对象

```java
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

public class ThreadNew {
    public static void main(String[] args){
        //3、创建callable接口实现类的对象
        NewThread newThread= new NewThread();
        //4、将此callable接口实现类的对象作为参数传递到FutureTask类的构造器中创建出一个FutureTask的实现类
        FutureTask futureTask = new FutureTask(newThread);
        //5、将FutureTask类的对象作为参数传递给Thread类的构造器创建一个Thread对象然后调用start()方法启动该线程
        new Thread(futureTask).start();
        try {
            //6、可以通过futureTask.get()方法获取call()方法中的返回值
            System.out.println(futureTask.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }

}
//1、创建一个类实现callable接口
class NewThread implements Callable<Integer> {
    //2、实现该接口中的call()方法
    @Override
    public Integer call() {
        int sum = 0;
        for (int i = 1; i <= 100; i++) {
            if (i % 2 == 0) {
                sum += i;
            }
        }
        return sum;
    }
}
```


#### 方式 4 通过lambda匿名函数创建线程

java通过lambda匿名函数以Thread匿名内部类方式创建线程
```java
public class Main {
    public static void main(String[] args) {
        Thread t = new Thread(() -> {
            System.out.println("start new thread!");
        });
        t.start(); // 启动新线程
    }
}
```
[参考](https://www.cnblogs.com/remyuu/p/16258322.html)

## Runnable、Callable和Future

### Runnable和Callable的区别

Callable中唯一规定的方法是call()，Runnable中唯一的方法是run()方法（废话）
Callable接口的call方法是一个泛型接口，能指定类型和返回值
而Runnable接口的run方法没有返回值，因此需要通过共享变量或者线程通信的方式来获取结果。
call方法可以抛出异常，run方法不可以。 
运行Callable任务可以拿到一个Future对象，表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过Future对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果。

Future接口是对runnable和callable任务进行管理的接口，有着获取执行结果、查看任务是否完成、取消任务，查看任务是否取消的功能。
FutureTask是实现了Future和runnable的接口，实现了runnable和callable的线程管理的功能。

即使是Callable和FutureTask实现的线程也是要通过轮询的方式来查询执行结果，虽然轮询的方式也是异步，但效率不如回调函数的形式高。
（**异步不一定非要靠回调函数实现，FutureTask的异步就是通过轮询实现的。**）


### Future和FutureTask的使用

Java本身在JDK5中引入的的Future的原理是将任务封装成一个FutureTask，将任务放投入线程池的时候实际上执行的是该任务的run方法。主线程和FutureTask创建的任务线程相当于通过FutureTask对象建立了通信通道，主线程需要等待业务线程执行结束才能获得执行结果。

Futrue 的使用方式是：投递一个任务到 Future 中执行，操作完之后调用 Future#get() 或者 Future#isDone() 方法判断是否执行完毕。从这个逻辑上看， Future 提供的功能是：用户线程需要主动轮询 Future 线程是否完成当前任务，如果不通过轮询是否完成而是同步等待获取则会阻塞直到执行完毕为止。所以从这里看，Future并不是真正的异步，因为它少了一个回调，充其量只能算是一个同步非阻塞模式。


Future是异步，是通过isDone轮询的方式实现的异步，但不如回调方式来得高效。
对于异步编程，我们想要的实现是：提交一个任务，在任务执行期间提交者可以做别
的事情，这个任务是在异步执行的，当任务执行完毕通知提交者任务完成获取结果。

而Netty的Future是通过监听器的方式获得任务执行的结果或者异常情况。

对Netty中Future/Promise模式的分析拟参考 [对JDK和Netty中Future机制的分析](https://www.cnblogs.com/rickiyang/p/12742091.html) 之后再写一篇关于Future和Netty的笔记



## 线程使用简单举例

Runnable是一个接口，Thread是实现了Runnable的一个类，Java中创建线程时，继承Runnable和继承Thread都可以。Runnable整个接口，仅有三行，仅定义了一个run方法，无其他，十分简洁；而thread定义了很多东西，总共两千多行。在创建线程时可以继承runnable也可以继承Thread，继承Runnable更灵活简洁一些，即要自己实所有方法。继承Thread方便一些，很多接口都已提供，但可能不如runnable简洁省内存。
```java
Thread t=new Thread(new RunnableClass());
t.start();
t.sleep(3000);
while (t.isAlive()){
    t.join(1000)
    if(somehappend){
        t.interrupt();

    }
}

```
interrupt()不会中断一个正在运行的线程。这一方法实际上完成的是，在线程受到阻塞时抛出一个中断信号，这样线程就得以退出阻塞的状态。更确切的说，如果线程被Object.wait, Thread.join和Thread.sleep三种方法之一阻塞，那么，它将接收到一个中断异常（InterruptedException），从而提早地终结被阻塞状态。 如果线程没有被阻塞，这时调用interrupt()将不起作用；
