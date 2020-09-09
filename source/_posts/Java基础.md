### String

1、为什么String类设计为final

1、String类型不可变才能维护一个String常量池。当两个String变量拥有同样的值时，可以将变量直接指向String常量池。

2、String类型不可变，所以创建的时候就计算了hashcode且不会再变，是得Hashmap中可以使用String作为可靠的Key。

#### 1、Hashmap是怎么实现的，底层原理？

HashMap的底层使用数组+链表/红黑树实现。

`transient Node<K,V>[] table;`这表示HashMap是Node数组构成，其中Node类的实现如下，可以看出这其实就是个链表，链表的每个结点是一个<K,V>映射。

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
```

HashMap的每个下标都存放了一条链表。

**常量/变量定义**

```java
/* 常量定义 */

// 初始容量为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;
// 负载因子，当键值对个数达到DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR会触发resize扩容 
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 当链表长度大于8，且数组长度大于MIN_TREEIFY_CAPACITY，就会转为红黑树
static final int TREEIFY_THRESHOLD = 8;
// 当resize时候发现链表长度小于6时，从红黑树退化为链表
static final int UNTREEIFY_THRESHOLD = 6;
// 在要将链表转为红黑树之前，再进行一次判断，若数组容量小于该值，则用resize扩容，放弃转为红黑树
// 主要是为了在建立Map的初期，放置过多键值对进入同一个数组下标中，而导致不必要的链表->红黑树的转化，此时扩容即可，可有效减少冲突
static final int MIN_TREEIFY_CAPACITY = 64;

/* 变量定义 */

// 键值对的个数
transient int size;
// 键值对的个数大于该值时候，会触发扩容
int threshold;
// 非线程安全的集合类中几乎都有这个变量的影子，每次结构被修改都会更新该值，表示被修改的次数
transient int modCount;
```

关于modCount的作用见[这篇blog](https://blog.csdn.net/u012926924/article/details/50452411)

> 在一个迭代器初始的时候会赋予它调用这个迭代器的对象的modCount，如果在迭代器遍历的过程中，一旦发现这个对象的modCount和迭代器中存储的modCount不一样那就抛异常。
> **Fail-Fast机制**：java.util.HashMap不是线程安全的，因此如果在使用迭代器的过程中有其他线程修改了map，那么将抛出ConcurrentModificationException，这就是所谓fail-fast策略。这一策略在源码中的实现是通过modCount域，modCount顾名思义就是修改次数，对HashMap内容的修改都将增加这个值，那么在迭代器初始化过程中会将这个值赋给迭代器的expectedModCount。在迭代过程中，判断modCount跟expectedModCount是否相等，如果不相等就表示已经有其他线程修改了Map。

**注意初始容量和扩容后的容量都必须是2的次幂**，为什么呢?

**hash方法**

先看散列方法

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

HashMap的散列方法如上，其实就是将hash值的高16位和低16位异或，我们将马上看到hash在与n - 1相与的时候，高位的信息也被考虑了，能使碰撞的概率减小，散列得更均匀。

在JDK 8中，HashMap的putVal方法中有这么一句

```java
if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
```

关键就是这句`(n - 1) & hash`，这行代码是把待插入的结点散列到数组中某个下标中，其中hash就是通过上面的方法的得到的，为待插入Node的key的hash值，n是table的容量即`table.length`，2的次幂用二进制表示的话，只有最高位为1，其余为都是0。减去1，刚好就反了过来。比如16的二进制表示为10000，减去1后的二进制表示为01111，除了最高位其余各位都是1，保证了在相与时，可以使得散列值分布得更均匀（因为如果某位为0比如1011，那么结点永远不会被散列到1111这个位置），且当n为2的次幂时候有`(n - 1) & hash == hash % n`, 举个例子，比如hash等于6时候，01111和00110相与就是00110，hash等于16时，相与就等于0了，多举几个例子便可以验证这一结论。最后来回答为什么HashMap的容量要始终保持2的次幂

- **使散列值分布均匀**
- **位运算的效率比取余的效率高**

注意table.length是数组的容量，而`transient int size`表示存入Map中的键值对数。

`int threshold`表示临界值，当键值对的个数大于临界值，就会扩容。threshold的更新是由下面的方法完成的。

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

```

该方法返回大于等于cap的最小的二次幂数值。比如cap为16，就返回16，cap为17就返回32。

**put方法**

put方法主要由putVal方法实现：

- 如果没有产生hash冲突，直接在数组`tab[i = (n - 1) & hash]`处新建一个结点；
- 否则，发生了hash冲突，此时key如果和头结点的key相同，找到要更新的结点，直接跳到最后去更新值
- 否则，如果数组下标中的类型是TreeNode，就插入到红黑树中
- 如果只是普通的链表，就在链表中查找，找到key相同的结点就跳出，到最后去更新值；到链表尾也没有找到就在尾部插入一个新结点。接着判断此时链表长度若大于8的话，还需要将链表转为红黑树（注意在要将链表转为红黑树之前，再进行一次判断，若数组容量小于64，则用resize扩容，放弃转为红黑树）

**get方法**

get方法由getNode方法实现：

- 如果在数组下标的链表头就找到key相同的，那么返回链表头的值
- 否则如果数组下标处的类型是TreeNode，就在红黑树中查找
- 否则就是在普通链表中查找了
- 都找不到就返回null

remove方法的流程大致和get方法类似。

**HashMap的扩容，resize()过程？**

``` java
newCap = oldCap << 1
```

resize方法中有这么一句，说明是扩容后数组大小是原数组的两倍。

```java
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 如果数组中只有一个元素，即只有一个头结点，重新哈希到新数组的某个下标
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        // 数组下标处的链表长度大于1，非红黑树的情况
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // oldCap是2的次幂，最高位是1，其余为是0，哈希值和其相与，根据哈希值的最高位是1还是0，链表被拆分成两条，哈希值最高位是0分到loHead。
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 哈希值最高位是1分到hiHead
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            // loHead挂到新数组[原下标]处；
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            // hiHead挂到新数组中[原下标+oldCap]处
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
```

举个例子，比如oldCap是16，二进制表示是10000，hash值的后五位和oldCap相与，因为oldCap的最高位（从右往左数的第5位）是1其余位是0，因此hash值的该位是0的所有元素被分到一条链表，挂到新数组中原下标处，hash值该位为1的被分到另外一条链表，挂到新数组中原下标+oldCap处。举个例子：桶0中的元素其hash值后五位是0XXXX的就被分到桶0种，其hash值后五位是1XXXX就被分到桶4中。

#### 2、Java中的错误和异常？

Java中的所有异常都是Throwable的子类对象，Error类和Exception类是Throwable类的两个直接子类。

Error：包括一些严重的、程序不能处理的系统错误类。这些错误一般不是程序造成的，比如StackOverflowError和OutOfMemoryError。

Exception：异常分为运行时异常和检查型异常。

- 检查型异常要求必须对异常进行处理，要么往上抛，要么try-catch捕获，不然不能通过编译。这类异常比较常见的是IOException。
- 运行时异常，可处理可不处理，在编译时可以通过，异常在运行时才暴露。比如数组下标越界，除0异常等。

#### 3、Java的集合类框架介绍一下？

首先接口Collection和Map是平级的，Map没有实现Collection。

Map的实现类常见有HashMap、TreeMap、LinkedHashMap和HashTable等。其中HashMap使用散列法实现，低层是数组，采用链地址法解决哈希冲突，每个数组的下标都是一条链表，当长度超过8时，转换成红黑树。TreeMap使用红黑树实现，可以按照键进行排序。LinkedHashMap的实现综合了HashMap和双向链表，可保证以插入时的顺序（或访问顺序，LRU的实现）进行迭代。HashTable和HashMap比，前者是线程安全的，后者不是线程安全的。HashTable的键或者值不允许null，HashMap允许。

Collection的实现类常见的有List、Set和Queue。List的实现类有ArrayList和LinkedList以及Vector等，ArrayList就是一个可扩容的对象数组，LinkedList是一个双向链表。Vector是线程安全的（ArrayList不是线程安全的）。Set里的元素不可重复，实现类常见的有HashSet、TreeSet、LinkedHashSet等，HashSet的实现基于HashMap，实际上就是HashMap中的Key，同样TreeSet低层由TreeMap实现，LinkedHashSet低层由LinkedHashMap实现。Queue的实现类有LinkedList，可以用作栈、队列和双向队列，另外还有PriorityQueue是基于堆的优先队列。

#### 4、Java反射是什么？为什么要用反射，有什么好处，哪些地方用到了反射？

反射：允许任意一个类在运行时获取自身的类信息，并且可以操作这个类的方法和属性。这种动态获取类信息和动态调用对象方法的功能称为Java的反射机制。

反射的核心是JVM在运行时才动态加载类或调用方法/访问属性。它不需要事先（写代码的时候或编译期）知道运行对象是谁，如`Class.ForName()`根本就没有指定某个特定的类，完全由你传入的类全限定名决定，而通过new的方式你是知道运行时对象是哪个类的。 反射避免了将程序“写死”。

反射可以降低程序耦合性，提高程序的灵活性。new是造成紧耦合的一大原因。比如下面的工厂方法中，根据水果类型决定返回哪一个类。

```java
public class FruitFactory {
    public Fruit getFruit(String type) {
        Fruit fruit = null;
        if ("Apple".equals(type)) {
            fruit = new Apple();
        } else if ("Banana".equals(type)) {
            fruit = new Banana();
        } else if ("Orange".equals(type)) {
            fruit = new Orange();
        }
        return fruit;
    }
}

class Fruit {}
class Banana extends Fruit {}
class Orange extends Fruit {}
class Apple extends Fruit {}
```

但是我们事先并不知道之后会有哪些类，比如新增了Mango，就需要在if-else中新增；如果以后不需要Banana了就需要从if-else中删除。这就是说只要子类变动了，我们必须在工厂类进行修改，然后再编译。如果用反射呢？

```java
public class FruitFactory {
    public Fruit getFruit(String type) {
        Fruit fruit = null;
        try {
            fruit = (Fruit) Class.forName(type).newInstance();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return fruit;
    }
}

class Fruit {}
class Banana extends Fruit {}
class Orange extends Fruit {}
class Apple extends Fruit {}
```

如果再将子类的全限定名存放在配置文件中。

```properties
class-type=com.fruit.Apple
```

那么不管新增多少子类，根据不同的场景只需修改文件就好了，上面的代码无需修改代码、重新编译，就能正确运行。

哪些地方用到了反射？举几个例子

- 加载数据库驱动时
- Spring的IOC容器，根据XML配置文件中的类全限定名动态加载类
- 工厂方法模式中（如上）

#### 5、说说你对面向对象、封装、继承、多态的理解？

- 封装：隐藏实现细节，明确标识出允许外部使用的所有成员函数和数据项。 防止代码或数据被破坏。
- 继承：子类继承父类，拥有父类的所有功能，并且可以在父类基础上进行扩展。实现了代码重用。子类和父类是兼容的，外部调用者无需关注两者的区别。
- 多态：一个接口有多个子类或实现类，在运行期间（而非编译期间）才决定所引用的对象的实际类型，再根据其实际的类型调用其对应的方法，也就是“动态绑定”。

Java实现多态有三个必要条件**：继承、重写、向上转型。**

- 继承：子类继承或者实行父类
- 重写：在子类里面重写从父类继承下来的方法
- 向上转型：父类引用指向子类对象

```java
public class OOP {
    public static void main(String[] args) {
        /*
         * 1. Cat继承了Animal
         * 2. Cat重写了Animal的eat方法
         * 3. 父类Animal的引用指向了子类Cat。
         * 在编译期间其静态类型为Animal;在运行期间其实际类型为Cat，因此animal.eat()将选择Cat的eat方法而不是其他子类的eat方法
         */
        Animal animal = new Cat();
        printEating(animal);
    }

    public static void printEating(Animal animal) {
        animal.eat();
    }
}

abstract class Animal {
    abstract void eat();
}
class Cat extends Animal {
    @Override
    void eat() {
        System.out.println("Cat eating...");
    }
}
class Dog extends Animal {
    @Override
    void eat() {
        System.out.println("Dog eating...");
    }
}
```

#### 6、实现不可变对象的策略？比如JDK中的String类。

- 不提供setter方法（包括修改字段、字段引用到的的对象等方法）
- 将所有字段设置为final、private
- 将类修饰为final，不允许子类继承、重写方法。可以将构造函数设为private，通过工厂方法创建。
- 如果类的字段是对可变对象的引用，不允许修改被引用对象。 1）不提供修改可变对象的方法；2）不共享对可变对象的引用。对于外部传入的可变对象，不保存该引用。如要保存可以保存其复制后的副本；对于内部可变对象，不要返回对象本身，而是返回其复制后的副本。

#### 7、Java序列话中如果有些字段不想进行序列化，怎么办？

对于不想进行序列化的变量，使用transient关键字修饰。功能是：阻止实例中那些用此关键字修饰的的变量序列化；当对象被反序列化时，被transient修饰的变量值不会被持久化和恢复。transient只能修饰变量，不能修饰类和方法。

#### 8、==和equals的区别？

== 对于基本类型，比较值是否相等，对于对象，比较的是两个对象的地址是否相同，即是否是指相同一个对象。

equals的默认实现实际上使用了==来比较两个对象是否相等，但是像Integer、String这些类对equals方法进行了重写，比较的是两个对象的内容是否相等。

对于Integer，如果依然坚持使用==来比较，有一些要注意的地方。对于[-128,127]区间里的数，有一个缓存。因此

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b); // true

Integer a = 128;
Integer b = 128;
System.out.println(a == b); // false

// 不过采用new的方式，a在堆中，这里打印false
Integer a = new Integer(127);
Integer b = 127;
System.out.println(a == b);
```

对于String，因为它有一个常量池。所以

```java
String a = "gg" + "rr";
String b = "ggrr";
System.out.println(a == b); // true

// 当然牵涉到new的话，该对象就在堆上创建了，所以这里打印false
String a = "gg" + "rr";
String b = new String("ggrr");
System.out.println(a == b);
```

#### 9、接口和抽象类的区别？

- Java不能多继承，一个类只能继承一个抽象类；但是可以实现多个接口。
- 继承抽象类是一种IS-A的关系，实现接口是一种LIKE-A的关系。
- 继承抽象类可以实现对父类代码的复用，也可以重写抽象方法实现子类特有的功能。实现接口可以为类新增额外的功能。
- 抽象类定义基本的共性内容，接口是定义额外的功能。
- 调用者使用动机不同, 实现接口是为了使用他规范的某一个行为；继承抽象类是为了使用这个类属性和行为.

#### 10、给你一个Person对象p，如何将该对象变成JSON表示？

本质是考察Java反射，因为要实现一个通用的程序。实现可能根本不知道该类有哪些字段，所以不能通过get和set等方法来获取键-值。使用反射的getDeclaredFields()可以获得其声明的字段。如果字段是private的，需要调用该字段的`f.setAccessible(true);`，才能读取和修改该字段。

```java
import java.lang.reflect.Field;
import java.util.HashMap;

public class Object2Json {
    public static class Person {
        private int age;
        private String name;

        public Person(int age, String name) {
            this.age = age;
            this.name = name;
        }
    }

    public static void main(String[] args) throws IllegalAccessException {
        Person p = new Person(18, "Bob");
        Class<?> classPerson = p.getClass();
        Field[] fields = classPerson.getDeclaredFields();
        HashMap<String, String> map = new HashMap<>();
        for (Field f: fields) {
            // 对于private字段要先设置accessible为true
            f.setAccessible(true);
            map.put(String.valueOf(f.getName()), String.valueOf(f.get(p)));
        }
        System.out.println(map);
    }
}

```

得到了map，再弄成JSON标准格式就好了。

#### 11、JDBC中sql查询的完整过程？操作事务呢？

```java
@Test
public void fun2() throws SQLException, ClassNotFoundException {
    // 1. 注册驱动
    Class.forName("com.mysql.jdbc.Driver");
    String url = "jdbc:mysql://localhost:3306/xxx?useUnicode=true&characterEncoding=utf-8";
    // 2.建立连接
    Connection connection = DriverManager.getConnection(url, "root", "admin");
    // 3. 执行sql语句使用的Statement或者PreparedStatment
    Statement statement = connection.createStatement();
    String sql = "select * from stu;";
    ResultSet resultSet = statement.executeQuery(sql);

    while (resultSet.next()) {
        // 第一列是id，所以从第二行开始
        String name = resultSet.getString(2); // 可以传入列的索引，1代表第一行，索引不是从0开始
        int age = resultSet.getInt(3);
        String gender = resultSet.getString(4);
        System.out.println("学生姓名：" + name + " | 年龄：" + age + " | 性别：" + gender);
    }
    // 关闭结果集
    resultSet.close();
    // 关闭statemenet
    statement.close();
    // 关闭连接
    connection.close();
}
```

**ResultSet维持一个指向当前行记录的cursor（游标）指针**

- 注册驱动
- 建立连接
- 准备sql语句
- 执行sql语句得到结果集
- 对结果集进行遍历
- 关闭结果集（ResultSet）
- 关闭statement
- 关闭连接（connection）

由于JDBC默认自动提交事务，每执行一个update ,delete或者insert的时候都会自动提交到数据库，无法回滚事务。所以若需要实现事务的回滚，要指定`setAutoCommit(false)`。

- `true`：sql命令的提交（commit）由驱动程序负责
- `false`：sql命令的提交由应用程序负责，程序必须调用commit或者rollback方法

JDBC操作事务的格式如下，在捕获异常中进行事务的回滚。

```java
try {
  con.setAutoCommit(false);//开启事务…
  ….
  …
  con.commit();//try的最后提交事务
} catch() {
  con.rollback();//回滚事务
}
```

#### 12、实现单例，有哪些要注意的地方？

就普通的实现方法来看。

- 不允许在其他类中直接new出对象，故构造方法私有化
- 在本类中创建唯一一个static实例对象
- 定义一个public static方法，返回该实例

```java
public class SingletonImp {
    // 饿汉模式
    private static SingletonImp singletonImp = new SingletonImp();
    // 私有化（private）该类的构造函数
    private SingletonImp() {
    }

    public static SingletonImp getInstance() {
        return singletonImp;
    }
}
```

饿汉模式：线程安全，不能延迟加载。

```java
public class SingletonImp4 {
    private static volatile SingletonImp4 singletonImp4;

    private SingletonImp4() {}

    public static SingletonImp4 getInstance() {
        if (singletonImp4 == null) {
            synchronized (SingletonImp4.class) {
                if (singletonImp4 == null) {
                    singletonImp4 = new SingletonImp4();
                }
            }
        }

        return singletonImp4;
    }
}
```

双重检测锁+volatile禁止语义重排。因为`singletonImp4 = new SingletonImp4();`不是原子操作。

```java
public class SingletonImp6 {
    private SingletonImp6() {}

    // 专门用于创建Singleton的静态类
    private static class Nested {
        private static SingletonImp6 singletonImp6 = new SingletonImp6();
    }

    public static SingletonImp6 getInstance() {
        return Nested.singletonImp6;
    }
}
```

静态内部类，可以实现延迟加载。

最推荐的是单一元素枚举实现单例。

- 写法简单
- 枚举实例的创建默认就是线程安全的
- 提供了自由的序列化机制。面对复杂的序列或反射攻击，也能保证是单例

```java
public enum Singleton {
    INSTANCE;
    public void anyOtherMethod() {}
}
```




## 海量数据处理

首先要了解几种数据结构和算法：

- HashMap，记住对于同一个键，哈希出来的值一定是一样的，不同的键哈希出来也可能一样，这就是发生了冲突（或碰撞）。
- BitMap，可以看成是bit数组，数组的每个位置只有0或1两种状态。Java中可以使用int数组表示位图，arr[0]是一个int，一个int是32位，故可以表示0-31的数，同理arr[1]可表示32-63...实际上就是用一个32位整型表示了32个数。
- 大/小根堆，O(1)时间可在堆顶得到最大值/最小值。利用小根堆可用于求Top K问题。
- 布隆过滤器。使用长度为m的bit数组和k个Hash函数，某个键经过k个哈希函数得到k个下标，将k个下标在bit数组中对应的位置设置为1。对于每个键都重复上述过程，得到最终设置好的布隆过滤器。对于新来的键，使用同样的过程，得到k个下标，判断k个下标在bit数组中的值是否为1，若有一个不为1，说明这个键一定不在集合中。若全为1，也可能不在集合中。就是说：查询某个键，判断不属于该集合是绝对正确的；判断属于该集合是低概率错误的。因为多个位置的1可能是由不同的键散列得到。

对上亿个无重复数字的排序，或者找到没有出现过数字，注意因为无重复数字，而BitMap的0和1正好可以表示该数字有没有出现过。如果要求更小的内存，可以先分出区间，对落入区间的进行计数。必然有的区间数量未满，再遍历一次数组，只看该区间上的数字，使用BitMap，遍历完成后该区间中必然有没被设置成0的的地方，这些地方就是没出现的数。

数据在小范围内波动，比如人类年龄，而且数据允许重复，可用计数排序处理数值排序或查找没有出现过的值，计数的桶中频次为0的就是没有出现过的数。

数据是数字，要找最大的Top K，直接用大小为K的小根堆，不断淘汰最小元素即可。

数据是数字或非数字，要找频次最高的Top K。可使用HashMap统计频次，统计出频次最大的前K个即可。统计频次选出前K的过程可以用小根堆。还可以用Hash分流的方法，即用一个合适的hash函数将数据分到不同的机器或者文件中，因为对于同样的数据，由于hash函数的性质，必然被分配到同一个文件中，因此不存在相同的数据分布在不同的文件这种情况。对每个文件采用HashMap统计频次，用小根堆选出Top K，然后汇总全部文件，从所有部分结果的Top K中再利用小根堆得到最终的Top K。

查找数值的排名，比如找到中位数。比如将数划分区间，对落入每个区间的数进行计数。然后可以得知中位数落在哪个区间，再遍历所有数，这次只关心落在该区间的数，不划分区间的对其进行计数，就可以找出中位数。

