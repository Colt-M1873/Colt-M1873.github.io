---
title: "对Java BIO，NIO，AIO的理解"
date: 2023-07-24T18:35:21+08:00
draft: false
ShowToc: true
TocOpen: false
---

**Taxonomy**
BIO Blocking I/O
NIO Non-Blocking I/O
AIO Asyncronous Non-blocking I/O

JAVA支持BIO NIO AIO

关于同步、异步、阻塞、非阻塞参考 [对同步、异步、阻塞、非阻塞的理解](http://3ms.huawei.com/km/blogs/details/14498633?l=zh-cn)

## BIO Blocking I/O
Java BIO：同步并阻塞（传统阻塞型），服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不作任何事情会造成不必要的线程开销。BIO是面向流的，并且流是阻塞式的



## NIO Non-Blocking I/O

Java NIO：同步非阻塞，服务器实现模式为一个线程处理多个请求(连接)，即客户端发送的连接请求会被注册到多路复用器上，多路复用器轮询到有 I/O 请求就会进行处理。NIO是面向块的，并且有缓冲区

NIO服务器实现模式为一个请求一个线程，客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理。NIO基于Reactor模式，其select底层调用操作系统的select/poll模型(Linux2.6采用epoll)不断的轮询，直到有数据可操作为止，这其实就是一种同步IO，当发现有数据操作之后进行的IO操作将不会被阻塞，即是非阻塞IO。另外在这种模型下，从通知应用程序有数据可读，到read调用把外部数据加载至内核空间，然后再从内核空间拷贝至用户空间需要一定时间间隔，这也会降低整体数据吞吐量。

## AIO Asyncronous Non-blocking I/O

Java AIO：异步非阻塞，AIO 引入了异步通道的概念，采用了 Proactor 模式，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用。

AIO服务器实现模式为一个有效请求一个线程，客户端的I/O请求都是由OS先完成了再通知服务器应用去启动线程进行处理。与NIO不同，AIO基于Proactor模式当进行读写操作时，只须直接调用API的read或write方法即可。这两种方法均为异步的，对于读操作而言，当有流可读取时，操作系统会将可读的流传入read方法的缓冲区，并通知应用程序；对于写操作而言，当操作系统将write方法传递的流写入完毕时，操作系统主动通知应用程序。在这种情况下，操作系统会负责将可读取的数据从内核空间转移至用户空间之后，才主动通知应用程序，应用程序不但不需要不停的轮询，并且数据的读取过程也更加快捷。

在Linux 2.6以后，java NIO的实现，Linux是通过epoll来实现的，Windows是通过较低效的select/poll实现的，这点可以通过jdk的源代码发现。而对于AIO，在windows上是通过IOCP实现的，在Linux上还是通过epoll来实现的。

关于select/poll和epoll，可参考[这篇select，poll和epoll的总结文章，内含链接指向三篇独立的解释文章](https://www.cnblogs.com/sky-heaven/p/7011684.html)
