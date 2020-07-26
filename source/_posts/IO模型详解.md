## NIO是什么?

定义：NIO(Non-blocking I/O,New I/O)即非阻塞I/O,属于IO模型的一种

分类：同步/异步  阻塞/非阻塞 I/O 



首先什么是I/O操作?I/O(input/output),是机器获取和交换信息的主要渠道。

在操作系统执行I/O操作的时候,通常都会涉及到用户态(User Mode)和内核态(Kernel Mode)的转换。

对于一个input操作来说，通常有两个步骤。1、等待数据就绪 2、将数据从kernel缓冲复制到user缓冲

Unix(-like)系统下的五种基本I/O模型

- blocking I/O

  一种常见的I/O模型，进程在调用recvfrom之后就一直处于阻塞状态，直到内核进程返回执行结果。

  

- nonblocking I/O

  进程不断的调用recvfrom，内核在数据包就绪之前一直返回EWOULDBLOCK标识。数据包就绪之后再开始复制数据，完成后返回OK。

  

- I/O multiplexing (`select` and `poll`)

  在直接调用recvfrom之前，先调用select来确定数据报是否就绪，如果就绪了内核会返回一个readable标识。收到readable标识后再去调用recvform来通知内核执行复制操作。

  

  总结：这个和第一个blocking I/O来比没啥太大优点，还多了一次系统调用。好处就是可以等待多个数据报就绪的结果了，然后哪个就绪了就处理哪个。

  ##### 关于select,poll和epoll

  他们都是用来监听文件描述符fd的系统调用方法。select和poll可以认为是一样的，都是使用轮询的方式来从很多fd中检查哪一个就绪了。由于时间复杂度为O(n)，不适合执行大量密集fd的监控,后来出现了时间复杂度为O(1)的epoll替代他们。它是一种类似订阅的实现, `epoll` 包括(`epoll_create`, `epoll_ctl`, `epoll_wait`) ,首先epoll_create建立一个epoll，`epoll_ctl`来设定要监控的fd类型和事件类型，`epoll_wait`来获取结果

- signal driven I/O (`SIGIO`)

  

  首先调用sigaction建立一个信号句柄，内核会立即返回，这个过程非阻塞。数据报就绪后，内核会给进程生成一个SIGIO，这个时候可以再去调用recvfrom

- asynchronous I/O (the POSIX `aio_` functions)

  

通过调用aio_read告知内核再整个input操作完成后(包括等待数据、复制到user缓冲)再通知，整个过程中进程是没有阻塞的，是真正的异步操作。

**POSIX**如下规定：

一个同步IO操作让申请进程在I/O操作结束之前一直被阻塞，反之异步操作就是没有被阻塞。

所以说前面四种都是同步I/O，只有最后一个asynchronous I/O才是真正的异步I/O



#### Java中的BIO，NIO，AIO

bio: 基于java.lang.io

nio: java1.4后引入 java.lang.nio

aio:Java 7引入



#### 其他

文件描述符(****file descriptor**,FD**,**fildes**)是一个获取文件资源(或其他IO资源)的句柄([handle](https://en.wikipedia.org/wiki/Handle_(computing)))，通常是一个非负整数

The **Portable Operating System Interface** (**POSIX**) :旨在维持操作系统兼容性的标准

参考文献：

https://jvns.ca/blog/2017/06/03/async-io-on-linux--select--poll--and-epoll/

https://www.ibm.com/developerworks/cn/java/j-lo-javaio/index.html

http://tutorials.jenkov.com/

