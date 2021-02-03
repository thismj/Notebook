<!-- toc -->

# Java

## Java基础

### 基本类型

| 类型    | 封装类型  | 长度  | 取值范围                       |
| ------- | --------- | ----- | ------------------------------ |
| byte    | Byte      | 1字节 | -2^7 ~ 2^7-1                   |
| short   | Short     | 2字节 | -2^15 ~ 2^15-1                 |
| int     | Integer   | 4字节 | -2^31 ~ 2^31-1                 |
| long    | Long      | 8字节 | -2^63 ~ 2^63-1                 |
| float   | Float     | 4字节 | 1.401298e-45 ~ 3.402823e+38    |
| double  | Double    | 8字节 | 4.9000000e-324 ~ 1.797693e+308 |
| char    | Character | 2字节 | 0 ~ 65535                      |
| boolean | Boolean   | ?     | true 、false                   |

[参考文章]((https://www.zhihu.com/question/39462340)
上面基本类型的字节长度是从 Java 语言角度来表述的，其明确了不同类型的取值范围。但是 JVM 在字节码层面上支持的整数类型只有 int 和 long（为啥？JVM 字节码指令是单字节的，最多只有256条，为了节省指令数目，不可能每种类型都占用一个指令），所有比 int 小的整数类型都会被提升为 int 来运算，所以单纯从运算效率来看，使用byte、 short  跟 int、long 没有啥区别。但是基本类型作为对象的字段或者数组元素储存时，确实是占用上表中对应的字节长度，所以使用合适的基本类型是可以节省空间的。

boolean占几个字节？作为对象的字段或者数组元素储存时，占一个字节；在运算的时候（栈帧，局部变量表）会提升到 int，即4个字节（依赖 JVM 具体的实现）

Integer.parseInt() 跟 Integer.valueOf()  方法的异同？parseInt() 返回的是基本类型 int，valueOf() 返回的是包装类型 Integer，当方法参数是字符串类型时，valueOf() 内部调用了parseInt() 方法。valueOf() 方法，有 IntegerCache 的缓存，默认是 -128 ～ 127，可以通过 JVM 参数来指定最大值

1.1 字面量属于 double 类型，1 字面量属于 int 类型

```java
float f = 1.1; //无法隐式向下转型，编译错误
byte b1 = 129; //无法隐式向下转型，编译错误
byte b2 = 12;
b2 += 128; //隐式向下转型，被编译成 b2 = (byte)(b2 + 128)
```

> A compound assignment expression of the form E1 op= E2 is equivalent to E1 = (T)((E1) op >(E2)), where T is the type of E1, except that E1 is evaluated only once.


###  自动装箱与拆箱

[Java自动装箱/拆箱](https://zhuanlan.zhihu.com/p/27657548)
以 Integer 为例子，Java在编译的时候，当我们变量声明为对象类型而赋值为基本数据类型时，则使用 Integer.valueOf() 进行装箱。当我们的变量声明为基本类型而赋值为对象类型时，则使用 Integer.intValue() 进行拆箱。

### String、StringBuffer、StringBuilder

[Java中String、StringBuffer、StringBuilder的区别详解](https://blog.csdn.net/zhangqiluGrubby/article/details/69257901)

[深入理解StringBuffer和StringBuilder](https://zhuanlan.zhihu.com/p/27324173)

[JAVA 中的 StringBuilder 和 StringBuffer 适用的场景是什么？](https://www.zhihu.com/question/20101840)

[Java 中 String 类为什么要设计成不可变的](https://zhuanlan.zhihu.com/p/115322967)

[请别再拿“String s = new String("xyz");创建了多少个String实例”来面试了吧](https://www.iteye.com/blog/rednaxelafx-774673)

[java中这条语句创建了几个对象？](https://www.zhihu.com/question/39372793)

[[深入解析String#intern](https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html)](https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html)

上面的文章 intern 正确使用的例子，不是说创建的字符串会少 100w 倍，而是gc之后，堆中存留的字符串少 100w 倍。

[string转换成integer的方式及原理](https://www.jianshu.com/p/9eebb4f2ccb1)


### Java是值传递还是引用传递？

[参考文章](https://www.zhihu.com/question/31203609)
对基本类型来说，变量的值就是其对应的值；对引用类型来说，变量的值是其指向对象的地址值。

在Java中本质上都是值传递，即把实参变量的值拷贝给形参变量，只不过意义不一样，基本类型拷贝的是它代表的值，引用类型拷贝的是它所指向对象的地址值.

### ==、equal()、和hashcode()

==：比较的是值是否相等，对基本类型来说，就是比较它们所代表的值；对引用类型来说，就是比较它们所指向对象的地址值。
equal()： 比较的是两个对象的内容是否相同，如果没有重写这个方法，则与 == 作用一样。
hashcode()：获取对象的hash值，一个 int 类型的整数

[参考文章](https://blog.csdn.net/javazejian/article/details/51348320)
equal()重写规则：

1. 自反性。对于任何非 null 的引用值 x，x.equals(x) 应返回 true。
2. 对称性。对于任何非 null 的引用值 x 与 y ，当且仅当 y.equals(x) 返回 true 时，x.equals(y) 才返回 true
3. 传递性。对于任何非 null 的引用值 x、y 与 z，如果 y.equals(x) 返回 true，y.equals(z) 返回true，那么 x.equals(z) 也应返回true。
4. 一致性。对于任何非 null 的引用值 x 与 y ，假设对象上 equals 比较中的信息没有被修改，则多次调用 x.equals(y) 始终返回 true 或者始终返回 false。
5. 对于任何非空引用值 x，x.equal(null) 应返回 false。

父类与子类的混合比较，如果不遵守上述规则，就会出问题。

为什么重写了 equal() 之后要重写 hashcode() 方法？
hashcode() 默认返回跟对象内存地址有关，如果重写了 equal() 但是不重写 hashcode() 的话，则可能会导致两个不同地址的对象，equal() 是相等的，但是 hashcode() 不相等，导致在使用 Hash 集合的时候（HashMap、HashSet）不能正常工作。

### 泛型擦除、协变、逆变、通配符

[聊聊Java泛型擦除那些事](https://zhuanlan.zhihu.com/p/64583822)
[秒懂Java类型（Type）系统](https://blog.csdn.net/ShuSheng0007/article/details/89520530)
[Java 不能实现真正泛型的原因是什么?](https://www.zhihu.com/question/28665443)
[Java中的协变与逆变](https://extremegtr.github.io/2016/07/11/Covariance-And-Contravariance-In-Java/)

### 四种引用类型

[Java中的强引用，软引用，弱引用，虚引用有什么用？](https://www.zhihu.com/question/37401125)

### final、finally、finalize区别

[Java 中的 final、finally、finalize 有什么不同？](https://zhuanlan.zhihu.com/p/88957765)
[Java的finalizer，cleaner等如何实现](https://www.zhihu.com/question/62953438)
[为什么Java有GC还需要自己来关闭某些资源？](https://www.zhihu.com/question/29265003/answer/43745406)
[finalize方法](https://zhuanlan.zhihu.com/p/29522201)

### 反射

[Java 反射由浅入深 | 进阶必备](https://juejin.im/post/6844904005294882830)
[学习java应该如何理解反射？](https://www.zhihu.com/question/24304289)
[java中的静态变量和Class对象究竟存放在哪个区域？](https://www.zhihu.com/question/59174759)
[Java6 的类反射瓶颈](https://blog.csdn.net/raintungli/article/details/6286701)
[Java 反射到底慢在哪里？](https://www.zhihu.com/question/19826278)

### 异常

[Java的异常](https://www.liaoxuefeng.com/wiki/1252599548343744/1264734349295520)
[Java异常处理和设计](https://www.cnblogs.com/dolphin0520/p/3769804.html)
[深入理解 Java try-with-resource 语法糖](https://juejin.im/entry/6844903446185951240)
[在 JDK 9 中更简洁使用 try-with-resources 语句](https://waylau.com/concise-try-with-resources-jdk9/)

### 注解

[JAVA 注解的基本原理](https://zhuanlan.zhihu.com/p/78182978)

### 其它

[为什么JDK源码中，无限循环大多使用for(;;)而不是while(true)?](https://www.zhihu.com/question/52311366)



### Java7 新特性



### Java8 新特性

https://juejin.im/post/6844903600301293581#heading-7

* 接口默认方法 default
* 函数式接口 FunctionInterface 和 lambda 表达式
* 方法引用 ::
* Stream 流式 API
* Optional
* Date/Time API
* Repeatable重复注解
* StringJoiner

## Java集合

### List

1. ArayList 底层通过动态数组实现，RandomAccess标记，支持快速随机访问，查询数据快，增删元素慢（数据需要移位，拷贝数组）
2. ArayList 调用空构造函数，第一次添加元素时会创建一个默认长度为10的数组，后续以 1.5 倍进行扩容；调用 int 参数函数，则默认创建参数大小的数组，后续以 1.5 倍 （oldCapacity + (oldCapacity >> 1) 进行扩容，参数传 0 跟默认空构造函数是不一样的，它会以1、2、3、4......大小进行扩容。（上述为Java 1.8 的实现，Java 1.6的空构造函数会默认创建一个容量为10的数组，并且是以 (oldCapacity * 3)/2 + 1 进行的）
3. ArayList底层数组使用 transient 修饰是因为 ArrayList 的底层数组扩容之后一般会有预留空间（这也是 为什么 List 元素个数叫 size() 而数组叫 length() 的原因），所以序列化整个底层数组没有意义，影响效率，浪费空间。ArrayList 重写了 writeObject() 方法，只序列化确实存在的 size() 个元素。
4. Vector 和 ArayList ：都是动态数组实现，但是Vector是线程安全的，关键方法都加了 synchronized 修饰；Vector 的扩容策略是 oldCapacity + ((capacityIncrement > 0) ? capacityIncrement : oldCapacity) ，可以通过构造函数指定每次扩容的大小，如果未指定则以 2 倍进行扩容（Java 1.8）
5. [SynchronizedList和Vector的区别](https://blog.csdn.net/w372426096/article/details/80679665)
6. [Java里用Iterator遍历一个ArrayList里的数据比直接用ArrayList里的方法访问整个List里的数据要好吗？](https://www.zhihu.com/question/29819559)
7. 增强性 for 循环原理是 iterator，在迭代遍历时，多线程环境下其它线程对集合做增删操作时（单线程在循环体里面做增删操作也会导致，但是可以用 iterator remove方法防止），会导致 modCount != expectedModCount 从而 throw ConcurrentModificationException（并发修改异常）
8. [CopyOnWriteArrayList你都不知道，怎么拿offer？](https://zhuanlan.zhihu.com/p/48784500)
9. Stack LIFO继承自 Vector，被 Deque 取代了
10. LinkedList：双向链表实现，增删元素时首先判断 index 是处于当前集合 size>>1 的前面还是后面。前面从 first 头指针开始遍历查找，后面从 last 指针开始遍历查找，增删操作比 ArrayList 效率高，只需要更新引用即可；LInkedList 实现了 Deque 的接口，也相当于是双端队列的链式储存结构，顺序储存结构可以用 ArrayDeque

### Queue

1. [Java之Queue接口中add()/offer()、remove()/poll()、element()/peek()的区别](https://www.jianshu.com/p/c41053d16713)
2. LinkedList是链式结构，ArrayDeque是动态数组会自动扩容，不会出现满队列的情况，所以这两个容器的 add()/offer() 方法无区别
3. [Stack and Queue](https://github.com/CarpenterLee/JCFInternals/blob/master/markdown/4-Stack%20and%20Queue.md)
4. ArrayDeque：双端循环队列，循序储存结构
5. BlockQueue添加了 put()/take() 这一对阻塞性方法,当队列满/空时会阻塞住线程,直到队列有空间/有元素,一些关键性的增删操作也加了锁,所以是线程安全的
6. ArrayBlockQueue：双端阻塞队列,循序储存结构,实现了 BlockQueue 的 put()/take() 方法 (必须指定队列长度并且不会自动扩容).
7. LinkedBlockingQueue: 双端阻塞队列,链式储存结构,实现了 BlockQueue 的 put()/take() 方法(如果不指定队列长度,默认容量为 Integer.MAX_VALUE)
8. PriorityBlockingQueue:二叉堆,顺序储存结构, 可以传入 Comparator,使队列的元素按照优先级排列, 添加元素 add/offer/put 会自动扩容,获取元素 take 阻塞.[二叉堆及优先级队列的实现原理](https://leetcode-cn.com/circle/article/bNtb4J/) [漫画：什么是二叉堆？（修正版）](https://www.cxyxiaowu.com/5352.html)
9. SynchronousQueue 两种模式，公平模式内部使用队列实现，非公平模式内部使用栈来实现，数据结构里面储存的结点包括数据元素、线程和区分是入队还是出队的标志位，使用CAS来实现非阻塞性的线程安全；如果是相同的操作则入队，如果不是相同的操作则出队。 [阻塞队列之SynchronousQueue源码分析](https://blog.csdn.net/u014634338/article/details/78419445) 
10. ConcurrentLinkedQueue：[ConcurrentLinkedQueue源码分析](https://juejin.im/post/6844903906288336909) 非阻塞队列，使用CAS来保证线程安全

### MAP

HashMap

[HashMap源码分析，基于1.8，对比1.7](https://blog.csdn.net/Leon_cx/article/details/81947991)

[[Java 8系列之重新认识HashMap](https://tech.meituan.com/2016/06/24/java-hashmap.html)](https://tech.meituan.com/2016/06/24/java-hashmap.html)

* key、value 都可以为 null，key 为 null 时 hashcode 为 0

* 1.7采用数组链表法来解决哈希冲突，1.8当链表长度达到 8 时会转为红黑树，当元素小于6的时候会转回链表

* 通过对数组 table size 取余来找到 hash 对应的数组位置 table[(size-1)&hashCode]，低位与(&)运算，即取余。计算 key 的 hashCode 时，通过扰动函数使哈希分配更均匀 (h = key.hashCode()) ^ (h >>> 16) ，使 hashCode 值的所有位都参与到上述取余运算（1.7对hashcode进行4次抖动...没太大意义）

* 数组 table 的 size 会被计算第一个大于 initialCapacity 并且满足为 2^n 的数；为啥要是 2^n？首先是通过取余运算能使 hash 平均分配在数组的各个位置，而通过 hash 值找到数组下标是通过位运算来的（table[(size-1)&hashCode]），当 size=2^n 时，size-1 二进制位全是1，& hashCode 后恰好保留了它的低位数据， 等价于取余 hashCode % size，效率高。（为了效率、为了 hash 位置的平均分配）

* table数组元素超过 threshold 时会进行扩容，table size扩容为两倍；threshold = 数组的长度legth*负载因子loadFactor（默认是0.75f）

* 1.7 扩容时遍历数组链表，根据结点 node 保存的 hash 值来找到在新数组的下标位置，链表元素转移时采用头插法（时间复杂度位O(1)）

* 1.8扩容时链表采用尾插法

* 1.7采用头插法将待转移元素插入到新hash桶链表中，高并发场景下会产生链表环，从而导致 Get 不存在元素时发生死循环 [高并发下的HashMap](https://zhuanlan.zhihu.com/p/31614195) JDK 1.8采用尾插法，规避了死循环的情况

LinkedHashMap
[搞懂 Java LinkedHashMap 源码](https://juejin.im/post/6844903590159450120#heading-5)
维护了一个双向链表，在一定条件下可以使遍历顺序跟添加顺序一致，取决于这个参数accessOrder，为 true 时遍历基于访问顺序，为 false 时基于插入顺序

HashTable

* 线程安全的HashMap，关键方法都加了 synchronized 修饰，阻塞性线程安全实现
* 不支持 key 与 value 为 null（并发情况不支持 key 跟 value 为 null）
* HashMap 是 Iterator 迭代器，支持 fail-fast （modCount）；Hashtable 是 Enumeration 迭代器，不支持  fail-fast （？？看HashTable 1.8 的代码，Enumeration 也实现了 Iterator 的方法？？ what？？？）

TreeMap
红黑树，二叉查找，排序场景使用，需要传入比较器（Comparator）或者类本身实现了 Comparable 接口

ConcurrentHashMap

* [什么是ConcurrentHashMap？](https://zhuanlan.zhihu.com/p/31614308)
* 1.7: 分段加锁技术，Segment 数组（Segment 继承自 ReentrantLock）+HashEntry数组（数组链表法），不同的 Segment 可以并发读写，同一个 Segment 读跟写也可以并发（读操作不加锁），同一个Segment 写跟写会阻塞（写操作加锁）；锁的粒度变小；size 方法循环统计 Segments 元素个数，通过判断对 segments 的修改次数，决定是否重新统计，超过重试次数，对每个 Segment  加锁，最后做一次统计。
* 1.8: 不再有 Segment 的概念，跟 1.8 hashMap的数据结构一致，数组+链表或红黑树，对每一个桶结点加锁（同步锁），锁的粒度更小了

### SET

HashSet 
内部有一个 HashMap 的成员变量以及 Object PRESENT = new Object()，每次添加元素的元素作为 key，PRESENT 作为 value

LinkedHashSet
内部为 LinkedHashMap 实现

TreeSet
内部为 TreeMap 实现

## Java IO

[ Java IO/NIO/AIO - Overview](https://www.pdai.tech/md/java/io/java-io-overview.html)

[NIO相关基础篇](https://mp.weixin.qq.com/s?__biz=MzU0MzQ5MDA0Mw==&mid=2247483907&idx=1&sn=3d5e1384a36bd59f5fd14135067af1c2&chksm=fb0be897cc7c61815a6a1c3181f3ba3507b199fd7a8c9025e9d8f67b5e9783bc0f0fe1c73903&scene=21#wechat_redirect)

阻塞性I/O

非阻塞性I/O

IO多路复用

单线程处理多个I/O请求，select/poll/epoll

信号驱动I/O

异步I/O

[【IO】深入理解管道(PipedInputStream、PipedOutputStream、PipedReader、PipedWriter)](https://blog.csdn.net/yhl_jxy/article/details/79283851)

[文件 I/O 的内核缓冲](https://github.com/raxxarr/note/issues/2)

[如何学习Java的NIO？](https://www.zhihu.com/question/29005375)

零拷贝实现

[面试被问到“零拷贝”！你真的理解吗？](https://zhuanlan.zhihu.com/p/146873631)

所有现代操作系统都使用虚拟内存，使用虚拟的地址取代物理地址，这样做的好处是：1.一个以上的虚拟地址可以指向同一个物理内存地址， 2.虚拟内存空间可大于实际可用的物理地址；利用第一条特性可以把内核空间地址和用户空间的虚拟地址映射到同一个物理地址，这样DMA就可以填充对内核和用户空间进程同时可见的缓冲区了

1、mmap+write

内存映射文件 

[[认真分析mmap：是什么 为什么 怎么用](https://www.cnblogs.com/huxiao-tee/p/4660352.html)](https://www.cnblogs.com/huxiao-tee/p/4660352.html)

2、senfile

Java实现，FileChannel.transferTo .transferFrom，两次拷贝DMA拷贝，从磁盘拷贝到内存缓冲区，从内存缓冲区拷贝到磁盘，没有CPU拷贝

## Java并发

### 进程与线程

进程是操作系统进行资源分配和调度的基本单位，每个进程都有自己独立的内存空间，进程间的切换开销比较大。
线程是 cpu 调度的最小单位，一个进程包括1个或多个线程，一般来说线程共享所在进程的内存空间和资源，每个线程有自己独立的运行栈和程序计数器，线程切换开销小。

Unix系统中通过 fork、vfork 创建进程，通过 pthread_create 创建线程。

### 创建线程

jvm把线程分为前台线程和后台线程（守护线程），通过 Thread 类正常创建的都是前台线程，可以在线程 start() 前调用 setDaemon(true) 方法将线程改变为守护线程，jvm结束的条件是所有的前台线程结束。

可以通过以下方法在 Java 中创建线程：

```java
public class JavaTest {
    private static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("MyThread.run");
        }
    }

    private static class MyRunnable implements Runnable {
        @Override
        public void run() {
            System.out.println("MyRunnable.run");
        }
    }

    private static class MyCallable implements Callable<Integer> {

        @Override
        public Integer call() throws Exception {
            return 0;
        }
    }

    public static void main(String[] args) {
        //扩展Thread类
        MyThread threadA = new MyThread();

        //实现Runnable接口
        Thread threadB = new Thread(new MyRunnable());

        threadA.start();
        threadB.start();

        //线程池
        ExecutorService executor = Executors.newCachedThreadPool();
        executor.submit(new MyRunnable());

        Future<Integer> future = executor.submit(new MyCallable());
    }
}

```

线程创建规则：
在 A 线程中创建 B 线程，则默认情况下 B 的优先级、是否守护线程与 A 相同。

### 守护线程

[从 JVM 视角看看 Java 守护线程](https://juejin.im/post/6844903966539513870)

### 线程的状态

Java 线程的 6 种状态定义在 java.lang.Thread$State 枚举类，这些状态是 JVM 状态，它不反映任何操作系统的线程状态。
[Java线程的6种状态及切换(透彻讲解)](https://blog.csdn.net/pange1991/article/details/53860651)
[Java中Thread类的join方法到底是如何实现等待的？](https://www.zhihu.com/question/44621343)

一个锁对应一个同步队列。

WAITTING对应的是等待队列，BLOCKED对应的是同步队列；等待队列中的线程收到 notify/notifyAll 的通知后会进入同步队列，当锁释放之后，同步队列中争抢到锁的线程进入 RUNNABLE 状态，等待系统调度获取 CPU 时间分片继续执行。

join() 方法用 synchronized 修饰，并且内部是用 wait() 方法来实现。在 A 线程中调用 B.join() 时，A 首先要获取 B 对象锁，然后实际上调用了 B 的 wait() 方法，此时 A 释放 B 对象锁，A 进入等待队列，等 B 线程执行完毕后，JVM 内部会调用 B 的 notifyAll 方法，此时 A 线程从等待队出队进入RUNNABLE 状态，等待系统调度获取 CPU 时间分片。


1. NEW

   新创建的线程对象，还没有调用 start() 方法

2. RUNNABLE

   处于 RUNNABLE 状态下的线程正在 JVM 中执行，但它可能正在等待来自于操作系统的其它资源。

   >Java 中的 RUNNABLE 状态实际上是包含了 Ready 与 Running 状态的，线程位于可运行线程池中，等待被线程调度选中，获取 CPU 的使用权，此时处于 Ready 状态。Ready状态的线程在获得 CPU 时间片后变为 Running 状态.

   >为何 JVM 中没有去区分这两种状态? CPU时间分片通常是很小的，一个线程一次最多只能在CPU 上运行比如 10-20ms 的时间（此时处于 Ready 状态），也即大概只有 0.01 秒这一量级，时间片用后就要被切换下来放入调度队列的末尾等待再次调度（也即回到 Ready 状态）。通常，Java的线程状态是服务于监控的，如果线程切换得是如此之快，那么区分 Ready 与 Running 就没什么太大意义了。现今主流的 JVM 实现都把 Java 线程一一映射到操作系统底层的线程上，把调度委托给了操作系统，我们在虚拟机层面看到的状态实质是对底层状态的映射及包装。JVM本身没有做什么实质的调度，把底层的 Ready 及 Running 状态映射上来也没多大意义，因此，统一成为 RUNNABLE 状态是不错的选择。

3. BLOCKED

4. WAITING

5. TIMED_WAITING

6. TERMINATED

### 中断线程

[java高并发程序设计1-线程停下来（stop,wait,suspend,await,interrupt,join,yield，sleep）的操作](https://blog.csdn.net/tangyuan_sibal/article/details/88695265)

sleep

Thread.sleep(0)的作用：使当前线程进入 RUNNABLE 的 Ready 状态，CPU重新调度线程

stop

过时，这方法会立刻释放线程所持有的锁，如果正在做同步操作，会产生很多未知的错误。

suspend/resume

过时，这玩意儿有死锁倾向，因为 suspend 之后不会释放持有的锁，这个时候只有调用线程的 resume 方法，让该线程继续工作，

但是如果调用线程的 resume 方法需要它本身持有的锁，则变成死锁了。

wait/notify

调用这两个方法时必须获取该对象的锁，否则会抛出 IllegalMonitorStateException 异常

interrupt/interrupted/isInterrupted

[[Java多线程：中断机制interrupt以及InterruptedException出现的原因](https://www.cnblogs.com/2015110615L/p/6736323.html)](https://www.cnblogs.com/2015110615L/p/6736323.html)

[[一文搞懂 Java 线程中断](https://segmentfault.com/a/1190000016537529)](https://segmentfault.com/a/1190000016537529?utm_source=sf-related)

线程中断方式，调用线程的 interrupt 方法，给线程做个中断标记，此时通过 isInterrupted 判断是否有中断标志位；interrupted 方法是 static 方法，返回当前线程是否有中断标志位，并清除当前的标志位。

若是我们调用线程的中断方法，当程序即将进入或是已经进入阻塞调用的时候，那么这个中断信号应该由InterruptedException捕获并进行重置；当run()方法程序段中不会出现阻塞操作的时候，这时候中断并不会抛出异常，我们需要通过interrupted()方法进行中断检查和中断标志的重置。另外，知道IO操作和synchronized上的阻塞不可中断也是必要的。

[[自己动手写把”锁”---LockSupport深入浅出](https://www.cnblogs.com/qingquanzi/p/8228422.html)](https://www.cnblogs.com/qingquanzi/p/8228422.html)

每个线程都有一个许可(permit)，permit只有两个值1和0,默认是0。

1. 当调用unpark(thread)方法，就会将thread线程的许可permit设置成1(注意多次调用unpark方法，不会累加，permit值还是1)。
2. 当调用park()方法，如果当前线程的permit是1，那么将permit设置为0，并立即返回。如果当前线程的permit是0，那么当前线程就会阻塞，直到别的线程将当前线程的permit设置为1.park方法会将permit再次设置为0，并返回。

注意：因为permit默认是0，所以一开始调用park()方法，线程必定会被阻塞。调用unpark(thread)方法后，会自动唤醒thread线程，即park方法立即返回。

带 blocker 的 park 方法可以提供传递给开发人员更多的信息，帮助监视工具和诊断工具确定线程收阻塞的原因。

#### ThreadLocal

每个线程在使用 ThreaLocal 变量时都会初始化一个完全独立的副本来保证线程安全。

ThreaLocal 是一个泛型类，可以接受任何类型的对象。Thread 实例里面有个类型为 ThreaLocal.ThreadLocalMap 的 threadLocals 变量，实际上 ThreaLocal 是对这个  ThreadLocalMap 集合的封装，这个Map是专门给 ThreaLocal 而设计的，它的元素 Entry 继承自 WeakReference<ThreadLocal>，key 就是 ThreaLocal  对象的弱引用，所以当外部不存在对 ThreaLocal 对象的强引用时，在下一次GC的时候会自动回收 ThreadLocal 对象。

### 锁

[面试必问的CAS，你懂了吗？](https://zhuanlan.zhihu.com/p/34556594)

[死磕Synchronized底层实现](https://github.com/farmerjohngit/myblog/issues/12)

[[不可不说的Java“锁”事](https://tech.meituan.com/2018/11/15/java-lock.html)](https://tech.meituan.com/2018/11/15/java-lock.html)

Java 1.6之前都是 Synchronized 都是重量级锁，利用操作系统底层的同步机制实现，对象头的 mark word 指向一个堆中 monitor 对象的指针

无锁->偏向锁->轻量级锁->重量级锁

[谈谈 synchronized 和 ReentrantLock 的区别](https://cloud.tencent.com/developer/article/1459414)

####volatile

JMM：Java内存模型

为什么Java的内存模型规范要这样定义导致出现线程本地内存和主存的值不同步呢？为啥线程要有自己的本地内存？

答案是利用缓存和改变执行代码顺序达到程序执行效率优化。

[关键字: volatile详解](https://www.pdai.tech/md/java/thread/java-thread-x-key-volatile.html)

#### 线程池

[[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)

[关于线程池你不得不知道的一些设置](https://objcoding.com/2019/04/14/threadpool-some-settings/#%E7%BA%BF%E7%A8%8B%E6%B1%A0%E7%9A%84%E6%A0%B8%E5%BF%83%E7%BA%BF%E7%A8%8B%E5%8F%AF%E4%BB%A5%E8%A2%AB%E5%9B%9E%E6%94%B6%E5%90%97)

Timer、ScheduledThreadPoolExecutor 的区别

Timer使用的是绝对时间，系统时间的改变会对Timer产生一定的影响；而ScheduledThreadPoolExecutor使用的是相对时间，所以不会有这个问题。

Timer使用单线程来处理任务，长时间运行的任务会导致其他任务的延时处理，而ScheduledThreadPoolExecutor可以自定义线程数量。

Timer没有对运行时异常进行处理，一旦某个任务触发运行时异常，会导致整个Timer崩溃，而ScheduledThreadPoolExecutor对运行时异常做了捕获（可以在`afterExecute()`回调方法中进行处理），所以更加安全。

