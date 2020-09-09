## 并发/多线程

#### 1、两个线程对可以同一个ArrayList进行add操作吗？会出现什么结果？

```java
import java.util.ArrayList;
import java.util.List;

public class A {
     static List<Integer> list = new ArrayList<>();
     static class BB implements Runnable {
        @Override
        public void run() {
            for (int j = 0; j < 100; j++) {
                list.add(j);
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        BB b = new BB();
        Thread t1 = new Thread(b);
        Thread t2 = new Thread(b);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(list.size());
    }
}


```

比如上面的例子，打印的结果不一定是200.

因为ArrayList不是线程安全的，问题出在add方法

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

上面的程序，可能有三种情况发生：

- **数组下标越界**。首先要检查容量，必要时进行扩容。每当在数组边界处，如果A线程和B线程同时进入并检查容量，也就是它们都执行完ensureCapacityInternal方法，因为还有一个空间，所以不进行扩容，此时如果A暂停下来，B成功自增；然后接着A从`elementData[size++] = e`开始执行，由于A之前已经检查过没有扩容，而B成功自增使得现在没有空余空间了，此时A就会发生数组下标越界。
- **小于200**。size++可以看成是`size = size + 1`，这一行代码包括三个步骤，先读取size，然后将size加1，最后将这个新值写回到size。此时若A和B线程同时读取到size假设为10，B先自增成功size变11，然后回来A因为它读到的size也是10，所以自增后写入size被更新成11，也就是说两次自增，实际上size只增大了1。因此最后的size会小于200。
- 200。运气很好，没有发生以上的情况。

#### 2、volatile和synchronized讲一下？

synchronized保证了当有多个线程同时操作共享数据时，任何时刻只有一个线程能进入临界区操作共享数据，其他线程必须等待。因此它可以保证操作的原子性。synchronized通过同步锁保证线程安全，进入临界区前必须获得对象的锁，其他没有获得锁的线程不可进入。当临界区中的线程操作完毕后，它会释放锁，此时其他线程可以竞争锁，得到锁的那个线程便可以进入临界区。

synchronized还可以保证可见性。因为对一个变量的unlock操作之前，必须先把次变量同步回主内存中。它还可以保证有序性，因为一个变量在任何时刻只能有一个线程对其进行lock操作（也就是任何时刻只有一个线程可以获得该锁对象），这决定了持有同一把锁的两个同步块只能串行进入。

volatile是一个关键字，用于修饰变量。被其修饰的变量具有可见性和有序性。

可见性，当一条线程修改了这个变量的值，新值能被其他线程立刻观察到。具体来说，volatile的作用是：在本CPU对变量的修改直接写入主内存中，同时这个写操作使得其他CPU中对应变量的缓存行无效，这样其他线程在读取这个变量时候必须从主内存中读取，所以读取到的是最新的，这就是上面说得能被立即“看到”。

有序性，volatile可以禁止指令重排。volatile在其汇编代码中有一个lock操作，这个操作相当于一个内存屏障，指令重排不能越过内存屏障。具体来说在执行到volatile变量时，内存屏障之前的语句一定被执行过了且结果对后面是已知的，而内存屏障后面的语句一定还没执行到；在volatile变量之前的语句不能被重排后其之后，相反其后的语句也不能被重排到之前。

#### 3、synchronized和重入锁的区别？

synchronized是JVM的内置锁，而重入锁是Java代码实现的。重入锁是synchronized的扩展，可以完全代替后者。重入锁可以重入，允许同一个线程连续多次获得同一把锁。其次，重入锁独有的功能有：

- 可以相应中断，synchronized要么获得锁执行，要么保持等待。而重入锁可以响应中断，使得线程在迟迟得不到锁的情况下，可以不再等待。主要由`lockInterruptibly()`实现，这是一个可以对中断进行响应的锁申请动作，锁中断可以避免死锁。
- 锁的申请可以有等待时限，用`tryLock()`可以实现限时等待，如果超时还未获得锁会返回false，也防止了线程迟迟得不到锁时一直等待，可避免死锁。
- 公平锁，即锁的获得按照线程先来后到的顺序依次获得，不会产生饥饿现象。synchronized的锁默认是不公平的，重入锁可通过传入构造方法的参数实现公平锁。
- 重入锁可以绑定多个Condition条件，这些condition通过调用await/singal实现线程间通信。

#### 4、synchronized作了哪些优化？

synchronized对内置锁引入了偏向锁、轻量级锁、自旋锁、锁消除等优化。使得性能和重入锁差不多了。

- 偏向锁：偏向锁会偏向第一个获得它的线程，如果在接下来的执行过程中，该锁没有被其他线程获取，则持有偏向锁的线程永远也不需要再进行同步。偏向锁是在无竞争的情况下把整个同步都消除掉，CAS操作也没有了。适合于同一个线程请求同一个锁，不适用于不同线程请求同一个锁，此时会造成偏向锁失效。
- 轻量级锁：如果偏向锁失效，虚拟机不会立即挂起线程，会使用一种称为轻量级锁的优化手段，轻量级锁的加锁和解锁都是通过CAS操作完成的。如果线程获得轻量级锁成功，则可以顺利进入临界区。如果轻量级锁加锁失败，表示其他线程抢先得到了锁，轻量级锁将膨胀为重量级锁。
- 自旋锁：锁膨胀后，虚拟机为了避免线程真实地在操作系统层面挂起，虚拟机还会做最后的努力--自旋锁。如果共享数据的锁定状态只有很短的一段时间，为了这段时间去挂起和恢复线程（都需要转入内核态）并不值得，所以此时让后面请求锁的那个线程稍微等待以下，但不放弃处理器的执行时间。这里的等待其实就是执行了一个忙循环，这就是所谓的自旋。虚拟机会让当前线程做几个循环，若干次循环后如果得到了锁，就顺利进入临界区；如果还是没得到，这才将线程在操作系统层面挂起。
- 锁消除：虚拟机即时编译时，对一些代码上要求同步，但被检测到不可能存在共享数据竞争的锁进行消除。锁消除的依据来源于“逃逸分析”技术。堆上的所有数据都不会逃逸出去被其他线程访问到，就可以把它们当栈上的数据对待，认为它们是线程私有的，同步加锁就是没有必要的。

#### 5、Java中线程的创建方式有哪些？

- 继承Thread并重写run方法
- 实现Runnable并重写run方法，然后作为参数传入Thread
- 实现Callable，并重写call()，call方法有返回值。使用FutureTask包装Callable实现类，其中FutureTask实现了Runnable和Future接口，最后将FutureTask作为参数传入Thread中
- 由线程池创建并管理线程。

#### 6、Java中线程池怎么实现的，核心参数讲一讲？

Executors是线程池的工厂类，通过调用它的静态方法如

```java
Executors.newCachedThreadPool();
Executors.newFixedThreadPool(n);
```

可返回一个线程池。这些静态方法统一返回一个`ThreadPoolExecutor`，只是参数不同而已。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {}
```

包括以上几个参数，其中：

- corePoolSize：指定了线程池中线程的数量；
- maximumPoolSize：线程池中的最大线程数量；
- keepAliveTime：当线程池中线程数量超过corePoolSize时，多余的空闲线程的存活时间；
- unit：上一个参数keepAliveTime的单位
- 任务队列，被提交但还未被执行额任务
- threadFactory：线程工厂，用于创建线程，一般用默认工厂即可。
- handler：拒绝策略。当任务太多来不及处理的时候，采用什么方法拒绝任务。

最重要的是任务队列和拒绝策略。

任务队列主要有ArrayBlockingQueue有界队列、LinkedBlockingQueue无界队列、SynchronousQueue直接提交队列。

使用ArrayBlockingQueue，当线程池中实际线程数小于核心线程数时，直接创建线程执行任务；当大于核心线程数而小于最大线程数时，提交到任务队列中；因为这个队列是有界的，当队列满时，在不大于最大线程的前提下，创建线程执行任务；若大于最大线程数，执行拒绝策略。

使用LinkedBlockingQueue时，当线程池中实际线程数小于核心线程数时，直接创建线程执行任务；当大于核心线程数而小于最大线程数时，提交到任务队列中；因为这个队列是有无界的，所以之后提交的任务都会进入任务队列中。newFixedThreadPool就采用了无界队列，同时指定核心线程和最大线程数一样。

使用SynchronousQueue时，该队列没有容量，对提交任务的不做保存，直接增加新线程来执行任务。newCachedThreadPool使用的是直接提交队列，核心线程数是0，最大线程数是整型的最大值，keepAliveTime是60s，因此当新任务提交时，若没有空闲线程都是新增线程来执行任务，不过由于核心线程数是0，当60s就会回收空闲线程。

当线程池中的线程达到最大线程数时，就要开始执行拒绝策略了。有如下几种

- 直接抛出异常
- 在调用者的线程中，运行当前任务
- 丢弃最老的一个请求，也就是将队列头的任务poll出去
- 默默丢弃无法处理的任务，不做任何处理

#### 7、BIO、NIO、AIO的区别？

首先要搞明白在I/O中的同步、异步、阻塞、非阻塞是什么意思。

- 同步I/O。由用户进程自己处理I/O的读写，处理过程中不能做其他事。需要主动去询问I/O状态。

- 异步I/O。由系统内核完成I/O操作，完成后系统会通知用户进程。

- 阻塞。I/O请求操作需要的条件不满足，请求操作一直等待，直到条件满足。

- 非阻塞。 I/O请求操作需要的条件不满足，会立即返回一个标志，而不会一直等待。

现在来看BIO、NIO、AIO的区别。

**BIO**：同步并阻塞。用户进程在发起一个I/O请求后，必须等待I/O准备就绪，I/O操作也由自己来处理，在IO操作未完成之前，用户进程必须等待。 

**NIO**：同步非阻塞。用户进程发起一个I/O请求后可立即返回去做其他任务，当I/O准备就绪时它会收到通知。接着由这个线程自行进行I/O操作，I/O操作本身还是同步的。

**AIO**：异步非阻塞。用户进程发起一个I/O操作以后可立即返回去做其他任务，真正的I/O操作由内核完成后通知用户进程。

NIO和AIO的不同：NIO是操作系统通知用户进程I/O已经准备就绪，由用户进程自行完成I/O操作；AIO是操作系统完成I/O后通知用户进程。 

BIO是为每一个客户端连接开启一个线程，简单说就是一个连接一个线程。

NIO主要组件有Seletor、Channel、Buffer，数据需要通过BUffer包装后才能使用Channel进行读取和写入。一个Selector可以由一个线程管理，每一个Channel可看作一个客户端连接。一个Selector可以监听多个Channel，即使用一个或极少数的线程来管理大量的客户端连接。当与客户端连接的数据没有准备好时，Selector处于等待状态，一旦某个Channel的准备好了数据，Selector就能立即得到通知。

#### 8、两个线程交替打印奇数和偶数？

先使用synchronized实现。PrintOdd用于打印奇数；PrintEven用于打印偶数。核心就是判断当前count如果是奇数，就让PrintEven阻塞，PrintOdd打印后唤醒在lock对象上等待的PrintEven并且释放锁。此时PrintEven获得锁打印偶数再唤醒PrintOdd，两个线程如此交替唤醒对方就实现了交替打印奇偶数。

```java
public class PrintOddEven {
    private static final Object lock = new Object();
    private static int count = 1;

    static class PrintOdd implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                synchronized (lock) {
                    try {
                        while ((count & 1) != 1) {
                            lock.wait();
                        }
                        System.out.println(Thread.currentThread().getName() + " " +count);
                        count++;
                        lock.notify();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }

    static class PrintEven implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                synchronized (lock) {
                    try {
                        while ((count & 1) != 0) {
                            lock.wait();
                        }
                        System.out.println(Thread.currentThread().getName() + " " +count);
                        count++;
                        lock.notify();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }

    public static void main(String[] args) {
        new Thread(new PrintOdd()).start();
        new Thread(new PrintEven()).start();
    }
}
```

如果要实现3个线程交替打印ABC呢？这次打算使用重入锁，和上面没差多少，但是由于现在有三个线程了，在打印完后需要唤醒其他线程，注意不可使用`sigal()`，因为唤醒的线程是随机的，不能保证打印顺序不说，还会造成死循环。一定要使用`sigalAll()`唤醒所有线程。

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ThreeThreadPrintABC {
    private static ReentrantLock lock = new ReentrantLock();
    private static Condition wait = lock.newCondition();
    // 用来控制该打印的线程
    private static int count = 0;

    public static void main(String[] args) {
        Thread printA = new Thread(new PrintA());
        Thread printB = new Thread(new PrintB());
        Thread printC = new Thread(new PrintC());
        printA.start();
        printB.start();
        printC.start();

    }

    static class PrintA implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 0) {
                        wait.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " A");
                    count++;
                    wait.signalAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    static class PrintB implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 1) {
                        wait.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " B");
                    count++;
                    wait.signalAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    static class PrintC implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 2) {
                        wait.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " C");
                    count++;
                    wait.signalAll();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }
}

```

如果觉得不好理解，重入锁是可以绑定多个条件的。创建3个Condition分别让三个打印线程在上面等待。A打印完了，唤醒等待在waitB对象上的PrintB；B打印完了唤醒在waitC对象上的PrintC；C打印完了，唤醒在waitA对象上等待的PrintA，如此循环地唤醒对方即可。

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ThreeThreadPrintABC {
    private static ReentrantLock lock = new ReentrantLock();
    private static Condition waitA = lock.newCondition();
    private static Condition waitB = lock.newCondition();
    private static Condition waitC = lock.newCondition();
    // 用来控制该打印的线程
    private static int count = 0;

    public static void main(String[] args) {
        Thread printA = new Thread(new PrintA());
        Thread printB = new Thread(new PrintB());
        Thread printC = new Thread(new PrintC());
        printA.start();
        printB.start();
        printC.start();

    }

    static class PrintA implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 0) {
                        waitA.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " A");
                    count++;
                    waitB.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    static class PrintB implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 1) {
                        waitB.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " B");
                    count++;
                    waitC.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }

    static class PrintC implements Runnable {
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                lock.lock();
                try {
                    while ((count % 3) != 2) {
                        waitC.await();
                    }
                    System.out.println(Thread.currentThread().getName() + " C");
                    count++;
                    waitA.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }
    }
}


```

#### 9、进程间通信的方式？线程间通信的方式？

进程间通信的方式[推荐阅读这篇博客](https://www.cnblogs.com/liugh-wait/p/8533003.html)

- 管道。分为几种管道。普通管道PIPE：单工，单向传输，只能在父子或者兄弟进程间使用；流管道，半双工，可双向传输，只能在父子或兄弟进程间使用；命名管道：可以在许多并不相关的进程之间进行通讯。
- 消息队列。消息队列是由消息的链表，存放在内核中并由消息队列标识符标识。消息队列克服了信号传递信息少、管道只能承载无格式字节流以及缓冲区大小受限等缺点。
- 信号。用于通知接收进程某个事件已经发生 
- 信号量。信号量是一个计数器，可以用来控制多个进程对共享资源的访问。它常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。
- 共享内存。共享内存就是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。共享内存是最快的 IPC（进程间通信） 方式，它往往与其他通信机制如信号量配合使用，来实现进程间的同步和通信。
- 套接字。可用于不同机器间的进程通信。

线程间通信的方式：

[推荐阅读这篇博客](https://www.cnblogs.com/xh0102/p/5710074.html)

- 锁机制。包括互斥锁、条件变量、读写锁。互斥锁以排他方式方式数据被并发修改；读写锁允许多个线程同时读取，对写操作互斥；条件变量可以以原子的方式阻塞进程，直到某个特定条件为真为止。对条件的测试是在互斥锁的保护下进行的。条件变量始终与互斥锁一起使用。 
- 信号量（Semaphore) 机制。包括无名线程信号量和命名线程信号量 
- 信号（Signal）机制。类似进程间的信号处理 

#### 10、原子类比如AtomicInteger为什么能保证原子性？

JDK并发包下有一个atomic包，里面实现了一些直接使用CAS操作的线程安全的类型。AtomicInteger就是其中之一。得益于CAS操作，因此保证了原子性。CAS操作具体说一说是什么？

CAS(Compare And Swap)，即“比较并交换”。CAS基于乐观的态度，是无锁操作，它操作包含三个参数，当前要更新的变量、期望值、新值，仅当：当前值和预期值一样时，才会将当前值设置为新值；如果当前值和预期值不一样，说明这个变量已经被其他线程修改过了。如果有多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新。其他线程允许放弃操作，也允许再次尝试，直到修改成功为止。CAS操作是由硬件支持的，现在的处理器基本支持原子化的CAS指令。

CAS由什么缺点？如何解决？

可能引发"ABA"问题，即一个变量原来是A，先被修改成B后又修改回了A，由于CAS操作只是比较当前值和预期值是否一样（只比较结果，不在乎过程中状态的变化），在其他线程来看，该变量就好像没有发生过变化。

可以为数据添加时间戳，每次成功修改数据时，不仅更新数据的值，同时要更新时间戳的值。CAS操作时，不仅要比较当前值和预期值，还要比较当前时间戳和预期时间戳。两者都必须满足预期值才能修改成功。

#### 11、实现一个简单的线程池？

实现一个类似于`Executors.newFixedThreadPool(n)`的固定大小线程池，当小于corePoolSize时候，优先创建线程去执行该任务；当超过该值时，将任务提交到任务队列中，然后各个线程从任务队列中取任务来执行。

```java
import java.util.HashSet;
import java.util.Set;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;

public class MyThreadPool {
    private int workerCount;
    private int corePoolSize;
    private BlockingQueue<Runnable> workQueue;
    private Set<Worker> workers;
    private volatile boolean RUNNING = true;
    public MyThreadPool(int corePoolSize) {
        this.corePoolSize = corePoolSize;
        workQueue = new LinkedBlockingQueue<>();
        workers = new HashSet<>();
    }

    public void execute(Runnable r) {
        if (workerCount < corePoolSize) {
            addWorker(r);
        } else {
            try {
                workQueue.put(r);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    private void addWorker(Runnable r) {
        workerCount++;
        Worker worker = new Worker(r);
        Thread t = worker.thread;
        workers.add(worker);
        t.start();
    }

    class Worker implements Runnable {
        Runnable task;
        Thread thread;

        public Worker(Runnable task) {
            this.task = task;
            this.thread = new Thread(this);
        }

        @Override
        public void run() {
            while (RUNNING) {
                Runnable task = this.task;
                // 执行当前的任务，所以把这个任务置空，以免造成死循环
                this.task = null;
                if (task != null || (task = getTask()) != null) {
                    task.run();
                }
            }
        }
    }

    private Runnable getTask() {
        Runnable r = null;
        try {
            r = workQueue.take();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return r;
    }


    public static void main(String[] args) {
        MyThreadPool threadPool = new MyThreadPool(5);
        Runnable r = new Writer();
        for (int i = 0; i < 10; i++) {
            threadPool.execute(r);
        }
    }


}

class Writer implements Runnable {

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " ");
    }
}

```

Worker实现了Runnale，是真正执行任务的类。当线程池中工作线程小于核心线程时候，调用addWorker直接start线程执行它的第一个任务。否则，将任务放入任务队列中，等线程来执行它们。Worker中的run方法是一个死循环，执行第一个任务（addWorker时调用start方法执行的那个任务），或者通过getTask方法不断从任务队列中取得任务来执行。正是getTask方法实现了线程的复用，即一个线程虽然只能调用一次start方法，但是后续的任务可以在Worker的run方法里直接调用任务的run方法得以执行。简单来说就是在Worker的run里调用任务的run方法。

任务全部执行完毕后，线程池需要被关闭，否则程序一直死循环。上述代码中并没有实现`shutdown()`方法。

#### 12、实现生产者-消费者模型？

可以有几种方式实现生产者-消费者模型：

- wait()/notify()

- await()/signal()

- BlockingQueue

生产者-消费者问题的关键在于：

- 没有“产品”时，消费者不能消费
- “产品”线满时，生产者不能生产

如果用队列来存放“产品”：

- 队列为空时，消费者需要一直等待，不为空时消费者才能取走。
- 队列为满时，生产者需要一直等待，不为满时生产者才能进行生产。

等待和唤醒可以使用wait()/notify()实现。Java中的阻塞队列BlockingQueue，其`take()`和和`put()`方法就是阻塞的，内部其实就是await()/signal()方法的配合使用，非常适合作为数据传输的通道。

以ArrayBlockingQueue来说，同步是重入锁保证的。和该lock绑定了两个Condition，一个是notEmpty一个是notFull。简单说明下take和put方法。注意这并不是源码，只是方便理解把核心部分抽取出来。

```java
public E take() {
    lock.lock();
    try {
        // 当队列为空，不能取，必须等待
        while (count==0) {
            notEmpty.await();
        }
        // 不再阻塞说明队列有元素了，直接删除并返回
        return dequeue();
    } finally ｛
        lock.unlock();
	}
}

private void enqueue(E x) {
    // ...insert element
    // 因为插入了元素，说明队列不为空，唤醒在notEmpty上等待的线程
    notEmpty.signal();
}

public void put(E e) {
    lock.lock();
    try {
        // 队列满了，不能放入，必须等待
        while (count == items.length) {
            notFull.await();
        }
        // 此时队列不为满，可以放入
        enqueue(e);
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    // ...delete element
    // 移除了元素，因而队列不为满，唤醒在notFull上等待的线程
    E x = (E) items[takeIndex];
    notFull.signal();
    return x;
}
```

了解了原理，现在用阻塞队列实现生产者-消费者模型

```java
package easy;

import java.util.Random;

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.LinkedBlockingQueue;

public class ProducerConsumer {
    public static class Producer implements Runnable {
        private BlockingQueue<Integer> blockingQueue;
        private Random random = new Random();

        public Producer(BlockingQueue<Integer> blockingQueue) {
            this.blockingQueue = blockingQueue;
        }

        @Override
        public void run() {
            while (true) {
                Integer a = makeProduct();
                try {
                    blockingQueue.put(a);
                    System.out.println(Thread.currentThread().getName() + "生产了" + a + " 队列大小" + blockingQueue.size());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

        public Integer makeProduct() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return random.nextInt(10);
        }
    }

    public static class Consumer implements Runnable {
        private BlockingQueue<Integer> blockingQueue;

        public Consumer(BlockingQueue<Integer> blockingQueue) {
            this.blockingQueue = blockingQueue;
        }


        @Override
        public void run() {
            while (true) {
                Integer a = useProduct();
                System.out.println(Thread.currentThread().getName() + "消费产品" + a + " 队列大小" + blockingQueue.size());
            }
        }

        public Integer useProduct() {
            Integer a = null;
            try {
                a = blockingQueue.take();
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return a;
        }
    }

    public static void main(String[] args) {
        BlockingQueue<Integer> blockingQueue = new LinkedBlockingQueue<>(5);
        ExecutorService exec = Executors.newCachedThreadPool();
        int producerNum = 3;
        int consumerNum = 5;

        for (int i = 0; i < producerNum; i++) {
            exec.submit(new Producer(blockingQueue));
        }
        for (int i = 0; i < consumerNum; i++) {
            exec.submit(new Consumer(blockingQueue));
        }

    }
}
```

#### 13、介绍下J.U.C.下的类？

- java.util.concurrent.locks包下的ReentrantLock，即重入锁。可以实现同步，是synchronized的扩展。还有ReadWriteLock即读写锁，读-读非阻塞，读和读之间可并行，因此适合读多写少的场合。
- ConcurrentHashMap，线程安全的HashMap；
- CopyOnWriteArrayList，核心是在写入操作时，先用一个副本复制原数组，然后新值写入到副本中，写入完成后再将修改完的副本替换掉原来的数组。这种实现使得写入操作也不会阻塞读操作了，只有写-写会同步等待。适合读多写少的场合。
- 阻塞队列BlockingQueue，有基于数组实现的有界队列ArrayBlockingQueue，还有基于链表实现的无界队列LinkedBlockingQueue。
- ConcurrentLinkedQueue，高效地并发队列。
- 信号量Semaphore，允许多个线程同时访问，synchronized和重入锁都只允许在一个时刻只有一个线程可以进入临界区访问共享资源，而信号量允许多个线程同时访问某一个资源。
- 倒计时器CountDownLatch，给倒计时器设定一个计数个数，每完成一个任务计数减1，某一个线程等待在倒计时器上，当计数完毕后才能继续该线程的执行。强调一个线程等待其他线程执行完成后才能继续执行。
- 循环栅栏CyclicBarrier，和CountDownLatch比较类似，可传入一个Runnable，该计数器可循环使用，每次计数完成会执行该Runnable。更强调线程之间的互相等待，必须所有线程等准备完毕（一次计数完成），才能执行某个任务。
- 跳表ConcurrentSkipListMap，跳表是一个多层的链表结构，最下层拥有全部的键值数据，越往上越少；查找时从顶层开始查找，在本层没找到转到下一层接着查找，用于快速查找，而且跳表中的数据是已排序的。
- 线程池，Executor、Executors、EexcutorService、ThreadPoolExecutor等。Executors是一个线程池工厂，其静态方法通过返回new ThreadPool，以EexcutorService接收，而EexcutorService和ThreadPoolExecutor都是Executor的实现类。
- Fork/Join框架，如ForkJoinPool线程池
- atomic包下的各种原子类，如AtomicInteger，主要使用了CAS操作实现无锁
- LockSupport类，线程阻塞工具类，主要有park和unpark方法表示线程的挂起和唤醒，

#### 14、读写锁用过没？

ReadWriteLock即读写锁，它有两个方法如下，分别返回一个读锁和写锁，即读写锁分离。

```java
ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
Lock readLock = readWriteLock.readLock();
Lock writeLock = readWriteLock.writeLock();
```

在读时使用readLock进行加锁，在写时使用writeLock进行加锁。使得读-读不阻塞，读线程完全并行，适合读多写少的场合。

#### 15、自旋锁是什么，为什么要用自旋锁？自选锁的缺点？

指当一个线程在获取锁的时候，锁已经被其它线程获取，但是有可能锁的状态只会持续很短的一段时间，为此将线程挂起、恢复并不值得，因为线程的挂起和恢复都需要转入到内核态。系统假定未请求到锁的线程在不久之后就能获得这锁，于是让后面请求锁的那个线程执行一个忙循环，然后不断的判断锁是否能够被成功获取，直到获取到锁才会退出循环 。这就是自旋锁。

自旋锁的好处：不会使线程状态发生改变，即一直处于用户态，不会转入内核态（用户态和内核态的切换系统开销很大）。不会使线程进入阻塞状态，减少了不必要的上下文切换，执行速度快。

自旋锁的缺点：一直占用CPU时间，如果锁被占用时间很短，自旋等待效果就很好，如果锁占用时间太长，自旋的线程只会白白消耗CPU资源。

后来引入了自适应自旋锁，自旋时间不再是固定的了，由上一次在同一个锁上的自旋时间和锁的拥有者的状态决定。如果在同一个锁对象上，自旋等待刚刚成功获得过锁，且持有锁的线程正在运行中，虚拟机认为这次自旋也会成功，而且它将允许自旋等待持续更长的时间；相反，如果对于某个锁，自旋很少成功获得过，在以后要获取这个锁时可能就会省略自旋过程。

#### 16、进程和线程的区别？

进程是资源分配的最小单位，线程是程序执行的最小单位。 进程是线程的容器，即进程里面可以容纳多个线程，多个线程之间可以共享数据。

#### 17、线程的死锁指什么？如何检测死锁？如何解决死锁？

是指两个或两个以上的线程在执行过程中，互相占用着对方想要的资源但都不释放，造成了互相等待，结果线程都无法向前推进。

死锁的检测：可以采用等待图（wait-for graph）。采用深度优先搜索的算法实现，如果图中有环路就说明存在死锁。

解决死锁：

- 破环锁的四个必要条件之一，可以预防死锁。
- 加锁顺序保持一致。不同的加锁顺序很可能导致死锁，比如哲学家问题：A先申请筷子1在申请筷子2，而B先申请筷子2在申请筷子1，最后谁也得不到一双筷子（同时拥有筷子1和筷子2）
- 撤消或挂起进程，剥夺资源。终止参与死锁的进程，收回它们占有的资源，从而解除死锁。 

#### 18、CPU线程调度？

- **协同式线程调度**：线程的执行时间以及线程的切换都是由线程本身来控制，线程把自己的任务执行完后，主动通知系统切换到另一个线程。优点是没有线程安全的问题，缺点是线程执行的时间不可控，可能因为某一个线程不让出CPU，而导致整个程序被阻塞。

- **抢占式调度模式**：线程的执行时间和切换都是由系统来分配和控制的。不过可以通过设置线程优先级，让优先级高的线程优先占用CPU。 

  Java虚拟机默认采用抢占式调度模型。

#### 19、HashMap在多线程下有可能出现什么问题？

- JDK8之前，并发put下可能造成死循环。原因是多线程下单链表的数据结构被破环，指向混乱，造成了链表成环。JDK 8中对HashMap做了大量优化，已经不存在这个问题。
- 并发put，有可能造成键值对的丢失，如果两个线程同时读取到当前node，在链表尾部插入，先插入的线程是无效的，会被后面的线程覆盖掉。

#### 20、ConcurrentHashMap是如何保证线程安全的？

JDK 7中使用的是分段锁，内部分成了16个Segment即分段，每个分段可以看作是一个小型的HashMap，每次put只会锁定一个分段，降低了锁的粒度：

- 首先根据key计算出一个hash值，找到对应的Segment
- 调用Segment的lock方法（Segment继承了重入锁），锁住该段内的数据，所以并没有锁住ConcurrentHashMap的全部数据
- 根据key计算出hash值，找到Segment中数组中对应下标的链表，并将该数据放置到该链表中
- 判断当前Segment包含元素的数量大于阈值，则Segment进行扩容（Segment的个数是不能扩容的，但是单个Segment里面的数组是可以扩容的）

多线程put的时候，只要被加入的键值不属于 同一个分段，就可以做到真正的并行put。**对不同的Segment则无需考虑线程同步，对于同一个Segment的操作才需考虑。**

JDK 8中使用了CAS+synchronized保证线程安全，也采取了数组+链表/红黑树的结构。 

put时使用synchronized锁住了桶中链表的头结点。

数组的扩容，被问到了我在看吧.....我只知道多个线程可以协助数据的迁移。

有这么一个问题，ConcurrentHashMap，有三个线程，A先put触发了扩容，扩容时间很长，此时B也put会怎么样？此时C调用get方法会怎么样？C读取到的元素是旧桶中的元素还是新桶中的

A先触发扩容，ConcurrentHashMap迁移是在锁定旧桶的前提下进行迁移的，并没有去锁定新桶。

- 在某个桶的迁移过程中，别的线程想要对该桶进行put操作怎么办？一旦某个桶在迁移过程中了，必然要获取该桶的锁，所以其他线程的put操作要被阻塞。**因此B被阻塞**。
- 某个桶已经迁移完成（其他桶还未完成），别的线程想要对该桶进行put操作怎么办？该线程会首先检查是否还有未分配的迁移任务，如果有则先去执行迁移任务，如果没有即全部任务已经分发出去了，那么此时该线程可以直接对新的桶进行插入操作（映射到的新桶必然已经完成了迁移，所以可以放心执行操作）

ConcurrentHashMap的get操作没有加锁，所以可以读取到值，不过是旧桶中的值。

```java
if (finishing) {
    nextTable = null;
    table = nextTab;
    sizeCtl = (n << 1) - (n >>> 1);
    return;
}
```

从table = nextTable可以看出，当所有数据迁移完成时，才将用nextTab新数组去覆盖旧数组table。所以在A扩容过程中，**C读取到的是旧数组中的元素**。

#### 21、ThreadLocal的作用和实现原理？

对于共享变量，一般采取同步的方式保证线程安全。而ThreadLocal是为每一个线程都提供了一个线程内的局部变量，每个线程只能访问到属于它的副本。

实现原理，下面是set和get的实现

 ```java
// set方法
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

// 上面的getMap方法
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// get方法
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
 ```

从源码中可以看出：每一个线程拥有一个ThreadLocalMap，这个map存储了该线程拥有的所有局部变量。

set时先通过Thread.currentThread()获取当前线程，进而获取到当前线程的ThreadLocalMap，然后以ThreadLocal自己为key，要存储的对象为值，存到当前线程的ThreadLocalMap中。

get时也是先获得当前线程的ThreadLocalMap，以ThreadLocal自己为key，取出和该线程的局部变量。

题话外，一个线程内可以设置多个ThreadLocal，这样该线程就拥有了多个局部变量。比如当前线程为t1，在t1内创建了两个ThreadLocal分别是tl1和tl2，那么t1的ThreadLocalMap就有两个键值对。

```java
t1.threadLocals.set(tl1, obj1) // 等价于在t1线程中调用tl1.set(obj1)
t1.threadLocals.set(tl2, obj2) // 等价于在t1线程中调用tl2.set(obj1)

t1.threadLocals.getEntry(tl1) // 等价于在t1线程中调用tl1.get()获得obj1
t1.threadLocals.getEntry(tl2) // 等价于在t1线程中调用tl2.get()获得obj2
```

#### 22、ArrayBlockingQueue和LinkedBlockingQueue的区别？

- ArrayBlockingQueue基于数组，是有界的阻塞队列，初始化时需要指定容量且不可扩容；LinkedBlockingQueue基于链表，是无界的阻塞队列，容量无限制。
- ArrayBlockingQueue读写共用一把锁，因此put和take是互相阻塞的；而LinkedBlockingQueue使用了两把锁，一把putLock和一把takeLock，实现了锁分离，使得put和take写数据和读数据可以并发的进行。

#### 23、CountDownLatch和CyclicBarrier的区别？

- CountDownLatch强调一个线程等待其他所有线程，通过cdl.await()让当前线程等待在倒计数器上，每有一个线程执行完，cdl.countDown()，将计数减1，减到0时通知当前线程执行。简单的说就是一个线程等待，直到他所等待的其他线程都执行完成，当前线程才可以继续执行。
- cyclicBarrier强调线程之间互相等待，只要有一个线程还没到来，所有线程会一起等待。可以传入一个Runnable作为计数完成要执行的任务。每有一个线程调用cyc.await()计数减1，减到0时会执行一次该Runnable。简单地说就是线程之间互相等待，等所有线程都准备好，即调用await()方法之后，执行一次Runnable，此时所有线程开始同时执行！
- CountDownLatch的计数器只能使用一次。而CyclicBarrier的计数器可以使用reset() 方法重置。

#### 24、synchronized内部实现原理？

推荐阅读下面两篇博客。

[倔强中的小白](https://blog.csdn.net/tryingpfq/article/details/82115612)

[zejian_的博客](https://blog.csdn.net/javazejian/article/details/72828483)

对同步方法，JVM采用`ACC_SYNCHRONIZED`标记符来实现同步。 对于同步代码块。JVM采用`monitorenter`、`monitorexit`两个指令来实现同步。

同步方法通过`ACC_SYNCHRONIZED`关键字隐式的对方法进行加锁。当线程要执行的方法被标注上`ACC_SYNCHRONIZED`时，需要先获得锁才能执行该方法。

同步代码块通过`monitorenter`和`monitorexit`执行来进行加锁。当线程执行到`monitorenter`的时候要先获得所锁，才能执行后面的方法。当线程执行到`monitorexit`的时候则要释放锁。

每个对象自身维护这一个被加锁次数的计数器，当计数器数字为0时表示可以被任意线程获得锁。当计数器不为0时，只有获得锁的线程才能再次获得锁，即可重入锁。换句话说，一个线程获取到锁之后可以无限次地进入该临界区。

Synchronized原子性

原子性是指一个操作是不可中断的，要全部执行完成，要不就都不执行。

在Java中，为了保证原子性，提供了两个高级的字节码指令`monitorenter`和`monitorexit`。而这两个字节码指令，在Java中对应的关键字就是`synchronized`。通过`monitorenter`和`monitorexit`指令，可以保证被`synchronized`修饰的代码在同一时间只能被一个线程访问，在锁未释放之前，无法被其他线程访问到。因此，在Java中可以使用`synchronized`来保证方法和代码块内的操作是原子性的。

Synchronized可见性

可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。`synchronized`修饰的代码，在开始执行时会加锁，执行完成后会进行解锁。而为了保证可见性，有一条规则是这样的：对一个变量解锁之前，必须先把此变量同步回主存中。这样解锁后，后续线程就可以访问到被修改后的值。

Synchronized有序性

有序性即程序执行的顺序按照代码的先后顺序执行。由于`synchronized`修饰的代码，同一时间只能被同一线程访问。那么也就是单线程执行的。所以，可以保证其有序性。

#### 25、什么叫做锁的可重入？

同一个线程可以多次获得同一个锁，即一个线程获取到锁之后可以无限次地进入该临界区 (对于ReentrantLock来说，通过调用`lock.lock()`)；当然锁的释放也需要相同次数的unlock()操作。注意：除了ReentrantLock，synchronized的锁也是可重入的。

#### 26、Java线程生命周期的状态？

```java
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WATING,
    TERMINATED;
}
```

线程的所有状态都在枚举类State中定义了，其中：

NEW表示刚刚创建的线程，此时线程还没有开始执行。调用`start()`后线程进入RUNNABLE状态，线程在执行过程中遇到synchronized同步块，就进入BLOCKED阻塞状态，此时线程暂停执行，直到获得请求的锁。WAITING和TIMED_WAITING都表示等待，区别是前者是无时间限制的等待，后者是有时限的等待。等待可以是执行`wait()`方法后等待`notify()`方法将其唤醒，也可以是通过`join()`方法等待的线程等待目标线程的执行结束。一旦等待了期望事件，线程再次执行，从等待状态变成RUNNABLE状态。线程执行结束后，进入TERMINATED状态。

#### 27、被notify()唤醒的线程可以立即得到执行吗？

被notify唤醒的线程不是立刻可以得到执行的，因为`notify()`不会立刻释放锁，`wait()`状态的线程也不能立刻获得锁；等到执行`notify()`的线程退出同步块后，才释放锁，此时其他处于`wait()`状态的线程才能获得该锁。

#### 28、ThreadLocal的应用场景？

数据库连接

```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {  
    public Connection initialValue() {  
        return DriverManager.getConnection(DB_URL);  
    }  
};  
  
public static Connection getConnection() {  
    return connectionHolder.get();  
}  
```

Session管理

```java
private static final ThreadLocal threadSession = new ThreadLocal();  
  
public static Session getSession() throws InfrastructureException {  
    Session s = (Session) threadSession.get();  
    try {  
        if (s == null) {  
            s = getSessionFactory().openSession();  
            threadSession.set(s);  
        }  
    } catch (HibernateException ex) {  
        throw new InfrastructureException(ex);  
    }  
    return s;  
}  
```

#### 29、sleep、wait、yield的区别和联系？

sleep() 允许指定以毫秒为单位的一段时间作为参数，它使得线程在指定的时间内进入阻塞状态，不能得到CPU 时间，指定的时间一过，线程重新进入可执行状态。调用sleep后不会释放锁。

yield() 使得线程放弃CPU执行时间，但是不使线程阻塞，线程从运行状态进入就绪状态，随时可能再次分得 CPU 时间。有可能当某个线程调用了yield()方法暂停之后进入就绪状态，它又马上抢占了CPU的执行权，继续执行。

wait()是Object的方法，会使线程进入阻塞状态，和sleep不同，wait会同时释放锁。wait/notify在调用之前必须先获得对象的锁。

#### 30、Thread类中的start和run方法区别？

run方法只是一个普通方法调用，还是在调用它的线程里执行。

start才是开启线程的方法，run方法里面的逻辑会在新开的线程中执行。
