---
title: JVM
date: 2020/4/30 20:46:25
updated: 2020/7/27 23:00:25
categories:
- 后端工程师
tags:
- Java
- JVM
---

#### class文件信息

```c
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

魔数+副版本号+主版本号+常量池计数器+常量池数据区+访问标志(access flags)+类索引(this_class)+父类索引(super_clas)+接口计数器(interfaces_count)+fields+methiods+类文件属性(attributes)

- **[Magic Number](https://en.wikipedia.org/wiki/Magic_number_(programming))**: 0xCAFEBABE
- **Version of Class File Format**: the minor and major versions of the class file
- **Constant Pool**: Pool of constants for the class
- **Access Flags**: for example whether the class is abstract, static, etc.
- **This Class**: The name of the current class
- **[Super Class](https://en.wikipedia.org/wiki/Superclass_(computer_science))**: The name of the super class
- **[Interfaces](https://en.wikipedia.org/wiki/Interface_(object-oriented_programming))**: Any interfaces in the class
- **Fields**: Any fields in the class
- **[Methods](https://en.wikipedia.org/wiki/Method_(computing))**: Any methods in the class
- **Attributes**: Any attributes of the class (for example the name of the sourcefile, etc.)

#### 类的加载过程

1. Loading

   通过类的全限名获取此类的二进制字节流

   将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构

   在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口

2. Linking(Verification => Preparation => Resolution)

   Verification ：

   * 目的在于确保Class文件的字节流中包含信息符合当前虚拟机的要求，保证被加载类的正确性，不会危害虚拟机自身安全。
   * 主要包括四种验证，文件格式验证，元数据验证，字节码验证，符号引用验证

   Preparation ：

   * 为类变量分配内存并且设置该类变量的默认初始值，即零值。

   Resolution：

   * 讲常量池内的符号引用转换为直接应用的过程。
   * 事实上，解析操作往往会伴随着JVM在执行完初始化之后再执行。

3. Initialization


## JVM

#### 1、Java内存区域（注意不是Java内存模型JMM）的划分？

- 程序计数器。
- 虚拟机栈。
- 本地方法栈。
- Java堆。
- 方法区。

前三个是线程私有的，后两个是线程共享的。

字节码解释器通过改变程序计数器的值来决定下一条要执行的指令，为了在线程切换后每条线程都能正确回到上次执行的位置，因为每条线程都有自己的程序计数器。

虚拟机栈是存放Java方法内存模型，每个方法在执行时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法返回地址等信息。方法的开始调用对应着栈帧的进栈，方法执行完成对应这栈帧的出栈。位于栈顶被称为“当前方法”。

本地方法栈和虚拟机栈类似，不过虚拟机栈针对Java方法，而本地方法栈针对Native方法。

Java堆。对象实例被分配内存的地方，也是垃圾回收的主要区域。

方法区。存放被虚拟机加载的**类信息、常量（final）、静态变量（static）、即时编译期编译后的代码**。方法区是用永久代实现的，这个区域的内存回收目标主要是针对常量池的回收和类型的卸载。**运行时常量池是方法区的一部分**，运行时常量池是Class文件中的一项信息，存放编译期生成的各种字面量和符号引用。

#### 2、新生代和老年代。对象如何进入老年代，新生代怎么变成老年代？

Java堆分为新生代和老年代。在新生代又被划分为Eden区，From Sruvivor和To Survivor区，比例是8:1:1，所以新生代可用空间其实只有其容量的90%。对象优先被分配在Eden区。

- 不过大对象比如长字符串、数组由于需要大量连续的内存空间，所以直接进入老年代。这是对象进入老年代的一种方式，
- 还有就是长期存活的对象会进入老年代。在Eden区出生的对象经过一次Minor GC会若存活，且Survivor区容纳得下，就会进入Survivor区且对象年龄加1，当对象年龄达到一定的值，就会进入老年代。
- 在上述情况中，若Survivor区不能容纳存活的对象，则会通过分配担保机制转移到老年代。
- 同年龄的对象达到suivivor空间的一半，大于等于该年龄的对象会直接进入老年代。

#### 3、新生代的GC和老年代的GC？

发生在新生代的GC称为Minor GC，当Eden区被占满了而又需要分配内存时，会发生一次Minor GC，一般使用复制算法，将Eden和From Survivor区中还存活的对象一起复制到To Survivor区中，然后一次性清理掉Eden和From Survivor中的内存，使用复制算法不会产生碎片。

老年代的GC称为Full GC或者Major GC：

- 当老年代的内存占满而又需要分配内存时，会发起Full GC
- 调用System.gc()时，可能会发生Full GC，并不保证一定会执行。
- 在Minor GC后survivor区放不下，通过担保机制进入老年代的对象比老年代的内存空间还大，会发生Full GC；
- 在发生Minor GC之前，会先比较历次晋升到老年代的对象平均年龄，如果大于老年代的内存，也会触发Full GC。如果不允许担保失败，直接Full GC。

#### 4、对象在什么时候可以被回收，调用finalize方法后一定会被回收吗？

在经过可达性分析后，到GC Roots不可达的对象可以被回收（但并不是一定会被回收，至少要经过两次标记），此时对象被第一次标记，并进行一次判断：

- 如果该对象没有调用过或者没有重写finalize()方法，那么在第二次标记后可以被回收了；
- 否则，该对象会进入一个FQueue中，稍后由JVM建立的一个Finalizer线程中去执行回收，此时若对象中finalize中“自救”，即和引用链上的任意一个对象建立引用关系，到GC Roots又可达了，在第二次标记时它会被移除“即将回收”的集合；如果finalize中没有逃脱，那就面临被回收。

因此finalize方法被调用后，对象不一定会被回收。

#### 5、哪些对象可以作为GC Roots？

- 虚拟机栈中引用的对象
- 方法区中类静态属性引用的对象（static）
- 方法区中常量引用的对象（final）
- 本地方法栈中引用的对象

#### 6、讲一讲垃圾回收算法？

- 复制算法，一般用于新生代的垃圾回收
- 标记清除， 一般用于老年代的垃圾回收
- 标记整理，一般用于老年代的垃圾回收
- 分代收集：根据对象存活周期的不同把Java堆分为新生代和老年代。新生代中又分为Eden区、from survivor区和to survivor区，默认8:1:1，对象默认创建在Eden区，每次垃圾收集时新生代都会有大量对象死亡。此时利用复制算法将Eden区和from survivor区还存活的对象一并复制到to survivor区。老年代的对象存活率高，没有额外空间进行分配担保，因此采用标记-清除或者标记-整理的算法进行回收。前者会产生空间碎片，而后者不会。

#### 7、介绍下类加载器和类加载过程？

**先说类加载器**。

在Java中，系统提供了三种类加载器。

- 启动类加载器（Bootstrap ClassLoader），启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要委派给启动类加载器，直接使用null。
- 扩展类加载器（Extension ClassLoader）
- 应用程序类加载器（Application ClassLoader），负责加载用户类路径（ClassPath）上锁指定的类库。是程序中默认的类加载器。

当然用户也可以自定义类加载器。

**再说类加载的过程**。

主要是以下几个过程：

```html
加载 -> 验证 -> 准备 -> 解析 -> 初始化 -> 使用 -> 卸载
```

**加载**

- 通过一个类的全限定名获取定义该类的二进制字节流
- 将字节流表示的静态存储结构转化为方法区的运行时数据结构
- 在内存中生成这个类的Class对象，作为方法区这个类的各种数据的访问入口

**验证**

- 文件格式验证：比如检查是否以魔数0xCAFEBABE开头
- 元数据验证：对类的元数据信息进行语义校验，保证不存在不符合Java语言规范的元数据信息。比如检查该类是否继承了被final修饰的类。
- 字节码验证，通过数据流和控制流的分析，验证程序语义是合法的、符合逻辑的。

**准备**。
为类变量（static）分配内存并设置默认值。比如static int a = 123在准备阶段的默认值是0，但是如果有final修饰，在准备阶段就会被赋值为123了。

**解析**。

将常量池中的符号引用替换成直接引用的过程。包括类或接口、字段、类方法、接口方法的解析。

**初始化**。

按照程序员的计划初始化类变量。如static int a = 123，在准备阶段a的值被设置为默认的0，而到了初始化阶段其值被设置为123。

#### 8、什么是双亲委派模型，有什么好处？如何打破双亲委派模型？

类加载器之间满足双亲委派模型，即：除了顶层的启动类加载器外，其他所有类加载器都必须要自己的父类加载器。当一个类加载器收到类加载请求时，自己首先不会去加载这个类，而是不断把这个请求委派给父类加载器完成，因此所有的加载请求最终都传递给了顶层的启动类加载器。只有当父类无法完成这个加载请求时，子类加载器才会尝试自己去加载。

双亲委派模型的好处？使得Java的类随着它的类加载器一起具备了一种带有优先级的层次关系。Java的Object类是所有类的父类，因此无论哪个类加载器都会加载这个类，因为双亲委派模型，所有的加载请求都委派给了顶层的启动类加载器进行加载。所以Object类在任何类加载器环境中都是同一个类。

如何打破双亲委派模型？使用OSGi可以打破。*OSGI*(Open Services Gateway Initiative)，或者通俗点说JAVA动态模块系统。可以实现代码热替换、模块热部署。在OSGi环境下，类加载器不再是双亲委派模型中的树状结构，而是进一步发展为更加复杂的网状结构。

#### 9、说一说CMS和G1垃圾收集器？各有什么特点。

CMS(Concurrent Mark Sweep) 从名字可以看出是可以进行并发标记-清除的垃圾收集器。针对老年代的垃圾收集器，目的是尽可能地减少用户线程的停顿时间。

收集过程有如下几个步骤：

- 初始标记：标记从GC Roots能直接关联到的对象，会暂停用户线程
- 并发标记：即在堆中堆对象进行可达性分析，从GC Roots开始找出存活的对象，可以和用户线程一起进行
- 重新标记：修正并发标记期间因用户程序继续运作导致标记产生变动的对象的标记记录
- 并发清除：并发清除标记阶段中确定为不可达的对象

CMS的缺点：

- 由于是基于标记-清除算法，所以会产生空间碎片
- 无法处理浮动垃圾，即在清理期间由于用户线程还在运行，还会持续产生垃圾，而这部分垃圾还没有被标记，在本次无法进行回收。
- 对CPU资源敏感

CMS比较类似适合用户交互的场景，可以获得较小的响应时间。

G1(Garbage First)，有如下特点：

- 并行与并发
- 分代收集
- 空间整合 ：整体上看是“标记-整理”算法，局部（两个Region之间 ）看是复制算法。确保其不会产生空间碎片。（这是和CMS的区别之一）
- 可预测的停顿：G1除了追求低停顿外，还能建立可预测的时间模型，主要原因是它可以有计划地避免在整个Java堆中进行全区域的垃圾收集。

在使用G1收集器时，Java堆的内存划分为多个大小相等的独立区域，新生代和老年代不再是物理隔离。G1跟踪各个区域的垃圾堆积的价值大小，在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的区域。

G1的收集过程和CMS有些类似：

- 初始标记：标记与GC Roots直接关联的对象，会暂停用户线程（Stop the World）
- 并发标记：并发从GC Roots开始找出存活的对象，可以和用户线程一起进行
- 最终标记：修正并发标记期间因用户程序继续运作导致标记产生变动的对象的标记记录
- 筛选回收：清除标记阶段中确定为不可达的对象，具体来说对各个区域的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划。

G1的优势：可预测的停顿；实时性较强，大幅减少了长时间的gc；一定程度的高吞吐量。

#### 10、CMS和G1的区别？

由上一个问题可总结出CMS和G1的区别：

- G1堆的内存布局和其他垃圾收集器不同，它将整个Java堆划分成多个大小相等的独立区域(Region)。G1依然保留了分代收集，但是新生代和老年代不再是物理隔离的，它们都属于一部分Region的集合，因此仅使用G1就可以管理整个堆。
- CMS基于标记-清除，会产生空间碎片；G1从整体看是标记-整理，从局部（两个Region之间）看是复制算法，不会产生空间碎片。
- G1能实现可预测的停顿。

#### 11、GC一定会导致停顿吗，为什么一定要停顿？任意时候都可以GC吗还是在特定的时候？

GC进行时必须暂停所有Java执行线程，这被称为Stop The World。为什么要停顿呢？因为可达性分析过程中不允许对象的引用关系还在变化，否则可达性分析的准确性就无法得到保证。所以需要STW以保证可达性分析的正确性。

程序执行时并非在所有地方都能停顿下来开始GC，只有在“安全点”才能暂停。安全点指的是：HotSpot没有为每一条指令都生成OopMap（Ordinary Object Pointer），而是在一些特定的位置记录了这些信息。这些位置就叫安全点。
