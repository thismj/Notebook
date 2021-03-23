# JVM

## 内存模型

**JMM**：Java内存模型，即 `Java Memory Model`，JMM屏蔽掉各种硬件与操作系统的内存访问差异，让 Java 程序在各种平台下达到一致的内存访问效果

![](https://pic3.zhimg.com/80/v2-f36f366c07a6188ea3fdefc794ba021a_720w.jpg)

每个线程的工作内存都是独立的，工作内存保存了该线程用到的变量（局部变量、方法参数等）和主内存的副本拷贝（实例变量，静态变量等），线程操作数据只能在工作内存中进行，然后刷回到主存。这是 JMM 定义的线程基本工作方式。

### 原子性

原子性指的是一个操作是不可分割，不可中断的，要么执行，要么不执行，没有执行到一半的状态。

```java
//原子操作
int i = 2;
//非原子操作，包含读取i的值，赋值给j两步操作
int j = i;
//非原子操作，包括读取i的值，i+1、赋值给i三步操作
i++;
//非原子操作，与i++等效
i = i + 1;
```

### 可见性

可见性是指某个线程修改共享变量的值之后，其它线程能够立即知道值被修改了。在 Java 中，当一个变量被 volatile  修饰时，如果变量被修改会立刻刷新回主内存，并且其它线程读取该变量时，需要重新从主内存获取。synchronized 也能保证可见性，在线程释放锁之前，会把共享变量同步回主内存。

### 有序性

在Java中，可以使用 synchronized 或者 volatile 保证多线程之间操作的有序性。

### 内存间交互的八种方式

![](https://pic4.zhimg.com/80/v2-42d8f894f17ccf13252d8d8d6285f86b_720w.jpg)



lock：线程锁定**主内存**的变量。

unlock：线程释放**主内存**的变量，而后可以被其它线程锁定。

read：读取**主内存**变量的值，传输到**工作内存**中，以便 load 操作。

load：把上述 read 的值放入**工作内存**的变量副本。

use：把**工作内存**中变量副本的值传输到执行引擎，每当虚拟机遇到需要使用变量值的字节码指令时将会执行这个操作

assign：把从执行引擎接受到的值赋值给**工作内存**的变量副本，每当虚拟机遇到给变量赋值的字节码指令时将会执行这个操作。

store：把**工作内存**中变量副本的值传输到**主内存**中，以便 write 操作。

write：把上述 store 的值放入**主内存**的变量中。

JMM 直接保证的原子性变量操作包括 read、load、assign、use、store 和 write 这六个，我们大致可以认为，基本数据类型（ long、double 除外）的访问、读写都是具备原子性的，或者通过 lock 和 unlock 操作来保证复杂逻辑的原子性（synchronized(java) -> monitorenter、monitorexit(jvm) -> lock、unlock(jmm) ）

对于 long、double 类型的读取与赋值操作，在 32 位 JVM 中是非原子性的，多线程下写（每次只能写 32 位）会出现高低位问题，需要使用 volatile 关键字确保原子性；对于 64 位虚拟机来说，一般的商用虚拟机对 long 与 double 的读写都是原子性的。

如果要把一个变量从主内存拷贝到工作内存，那就要按顺序执行 read 和 load 操作，如果要把变量从工作内存同步回主内存，就要按顺序执行 store 和 write 操作。注意，Java内存模型只要求上述两个操作必须按顺序执行，但不要求是连续执行。也就是说 read 与 load 之间、store 与 write 之间是可插入其他指令的，如对主内存中的变量 a、b 进行访问时，一种可能出现的顺序是 read a、read b、load b、load a。

JMM 对 8 种内存交互操作制定的规则：

1. 不允许read、load、store、write操作之一单独出现，也就是read操作后必须load，store操作后必须write。
2. 不允许线程丢弃他最近的 assign 操作，即工作内存中的变量数据改变了之后，必须告知主存。
3. 不允许线程将没有 assign 的数据从工作内存同步到主内存。
4. 一个新的变量只能在主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（ load 或 assign ）的变量，换句话说就是对一个变量实施 use、store 操作之前，必须先执行 load 和 assign 操作。
5. 一个变量在同一个时刻只允许一条线程对其进行 lock 操作，但 lock 操作可以被同一条线程重复执行多次，多次执行 lock 后，只有执行相同次数的 unlock 操作，变量才会被解锁。
6. 如果对一个变量执行 lock 操作，那将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行 load 或 assign 操作以初始化变量的值。
7. 如果一个变量事先没有被 lock 操作锁定，那就不允许对它执行 unlock 操作，也不允许去 unlock 一个被其他线程锁定的变量。
8. **对一个变量执行 unlock 操作之前，必须先把此变量同步回主内存中（执行store、write操作）**。

### Volatile

volatile 的作用主要有以下两个：

* 保证线程间变量的可见性（并不能保证线程安全）
* 禁止 CPU 进行指令重排序

指令的重排序：为了使指令更加符合 CPU 的执行特性，提高执行效率，指令的执行顺序可以跟代码逻辑不一致。在单线程环境下是不影响执行结果的，但是多线程环境下就不一定了，所以在多线程环境下可以使用 volatile 关键字来禁用指令重排序。对具有 volatile  修饰的变量做读写操作时，其前面的操作必须执行完毕且对后面的操作可见，并且后面的操作一定还没有执行；在进行指令优化时，不能把访问 volatile  变量前的语句放到其后面，也不能把后面的语句放到前面，当然前面或者后面那些语句的顺序是不能保证的。

volatile 禁止指令重排序原理，内存屏障（Memory Barrier）：

| **屏障类型** | **指令示例**               | **说明**                                                     |
| ------------ | -------------------------- | ------------------------------------------------------------ |
| LoadLoad     | Load1; LoadLoad; Load2     | 保证load1的读取操作在load2及后续读取操作之前执行             |
| StoreStore   | Store1; StoreStore; Store2 | 在store2及其后的写操作执行前，保证store1的写操作已刷新到主内存 |
| LoadStore    | Load1; LoadStore; Store2   | 在stroe2及其后的写操作执行前，保证load1的读操作已读取结束    |
| StoreLoad    | Store1; StoreLoad; Load2   | 保证store1的写操作已刷新到主内存之后，load2及其后的读操作才能执行 |

volatile 读操作后插入 LoadLoad 、LoadStore内存屏障，即后续的 Load 跟 Store 都要等前面的 Load 完成。

![](https://pic4.zhimg.com/80/v2-437ebf30028ea4364533da27460a3077_720w.jpg)

volatile 写操作前插入 StoreStore 内存屏障，确保前面的 Store 完成了；后面插入 StoreLoad 屏障，确保后续的 Load 要等前面的 Store 完成。

![](https://pic4.zhimg.com/80/v2-52d9c8897b541c9e4e5a41be59b482ef_720w.jpg)

## 运行时数据区

Java 虚拟机在执行 Java 程序中会把它所管理的内存划分为若干个不同的数据区域，如下图所示（虚拟机规范，具体虚拟机的实现或者叫法都有区别）：

![](https://upload-images.jianshu.io/upload_images/2614605-246286b040ad10c1.png)

**程序计数器**：线程私有的，位于工作内存，可以看作是当前线程执行字节码的行号指示器。如果线程正在执行一个 Java 方法，则记录的是正在执行的虚拟机字节码指令的地址。

**Java虚拟机栈**：线程私有的，位于工作内存。每个 Java 方法执行时都会创建一个栈帧，用于储存局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至完成的过程，就对应着一个栈帧在虚拟机栈中入栈和出栈的过程。局部变量表存放了编译器可知的各种基本类型，在编译期间完成分配，一个方法需要在栈帧中分配多大的局部变量空间时完全确定的。该区域会触发 StackOverFlow 和 OutOfMemoryError 异常。

**本地方法栈**：与 Java 虚拟机栈类似，但是执行的是 Native 方法（HotSpot虚拟机中并不区分虚拟机栈和本地方法栈）

**Java堆**：所有线程共享，位于主内存，几乎所有的对象实例都在堆中分配。从内存回收的角度来看，Java 堆可以细分为新生代和老年代，再细致一点的有 Eden 空间、From Survivor空间，To Survivor空间等。从内存分配的角度来看，线程共享的 Java 堆中可能分配出多个线程私有的分配缓冲区(Thread Local Allocations Buffer，TLAB )。Java 堆可以处于物理上不连续的内存空间中，只需要逻辑上是连续的即可，当前主流的虚拟机通过（-Xmx 最大堆内存 和 -Xms 初始化堆内存 ）控制堆的大小。该区域会触发 OutOfMemoryError 异常。

**方法区**：所有线程共享，用于储存已被虚拟机加载的类信息，常量、静态变量、即时编译器编译后的代码等数据（在 HotSpot 中称之为永久代 Permanent Generation -> PermGen）。该区域会触发 OutOfMemoryError 异常。

**运行时常量池**：是方法区的一部分，具备动态性，并不要求常量一定只有在编译器产生，在运行时也可以将新的常量放进池中，例如 String.intern()。

**直接内存**：NIO 基于通道（Channel）和缓冲区（Buffer）的I/O 方式，可以通过 DirectByteBuffer 对象引用堆外内存，并不属于虚拟机运行时数据区的一部分。

![](https://segmentfault.com/img/remote/1460000038862246)

[JDK版本运行时数据区对比](https://segmentfault.com/a/1190000038862239)

[[Java8内存模型—永久代(PermGen)和元空间(Metaspace)](https://www.cnblogs.com/paddix/p/5309550.html)](https://www.cnblogs.com/paddix/p/5309550.html)

- JDK 1.6：有永久代，静态变量存放在永久代上。
- JDK 1.7：有永久代，但已经把字符串常量池、静态变量，存放在堆上。逐渐的减少永久代的使用。
- JDK 1.8：无永久代，运行时常量池、类常量池，都保存在元数据区，也就是常说的`元空间`（MetaSpace）。但字符串常量池仍然存放在堆上。



## 类加载机制

### 类加载过程

虚拟机把 Class 文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型，这就是虚拟机的类加载机制。

Java 可以在运行过程中动态加载从网络或者其他地方的 Class 文件的二进制流作为程序代码的一部分。

![](https://img-blog.csdnimg.cn/2019062014564165.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3poYW9jdWl0,size_16,color_FFFFFF,t_70)



#### 加载

* 通过类的全名获取 Class 文件的二进制流
* 将二进制流所代表的静态储存结构转换成方法区的运行时数据结构
* 在内存中生成一个 java.lang.Class 对象，作为方法区的这个类的各种数据的访问入口

#### 验证

验证是连接阶段的第一步，确保 Class 文件符合 Java 虚拟机规范：

* 文件格式验证，是否以魔数0xCAFEBABE开头、主次版本号是否在当前 Java 虚拟机接受范围内等等
* 元数据验证，对类的信息进行语义校验，例如是否错误地覆盖了 final 类，是否实现接口中所有的方法等等
* 字节码验证，对类的方法体进行校验分析
* 符号引用验证， 确保该类可以访问它依赖的某些外部类、方法、字段等资源

#### 准备

给静态变量分配内存，并设置类变量初始值，通常情况下是数据类型的零值，例如：

```java
public static int value = 123;
```

`value` 在准备阶段过后的初始值是0而不是123

```java
public static final int value = 123;
```

`value` 在准备阶段过后的初始值是123

#### 解析

把 Class 常量池内的符号引用替换为直接引用的过程

* 类或接口的解析
* 字段解析
* 方法解析
* 接口方法解析

#### 初始化

类加载过程的最后一个步骤，真正开始执行类中编写的程序代码，将主导权移交给应用程序，实际上就是执行 `<clinit>() `方法的过程，`<clinit>() `方法是由编译器自动收集类中的所有静态变量和静态语句块合并产生的，执行的顺序按照语句在源文件中出现的顺序决定，静态语句块只能访问定义在它前面的静态变量，定义在它后面的静态变量只能进行赋值操作。Java虚拟机会保证子类在执行`<clinit>() `方法前，父类的 `<clinit>() ` 方法已经执行过了。

Java 虚拟机必须保证一个类的 `<clinit>() ` 方法在多线程环境中被正确地加锁同步，确保只会有一个线程去执行初始化方法，其它线程都会阻塞等待。



###双亲委托机制

![](https://segmentfault.com/img/bVcHO1J)



Bootstrap ClassLoader：C++实现的（仅仅 HotSpot 虚拟机），虚拟机自身的一部分，负责加载 `<JAVA_HOME>/lib` 目录的类文件，`rt.jar`、`tools.jar`

Extension ClassLoader：ExtClassLoader，继承自 `java.lang.ClassLoader`，负责加载 `<JAVA_HOME>/lib/ext` 目录的类文件

Application ClassLoader：AppClassLoader，继承自 `java.lang.ClassLoader`，负责加载 `ClassPath` 上所有的类库，程序中默认的类加载器

```java
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 先检查是否已经加载过，加载过就不会再重复加载了
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //委托给父加载器加载
                        c = parent.loadClass(name, false);
                    } else {
                        //没有父父加载器则去Bootstrap ClassLoader查看是否加载过
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    //父加载器加载失败之后，再尝试自己加载
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                //类加载机制的连接过程
                resolveClass(c);
            }
            return c;
        }
    }
```



### 动态代理

[动态代理](https://www.liaoxuefeng.com/wiki/1252599548343744/1264804593397984)
[[装饰模式与代理模式的区别](https://www.cnblogs.com/jaredlam/archive/2011/11/08/2241089.html)](https://www.cnblogs.com/jaredlam/archive/2011/11/08/2241089.html)

在程序运行期间，根据被代理的接口动态生成实现类并加载运行的过程，称之为动态代理。

#### 基本使用

基本使用方法如下，首先定义需要动态代理的接口：

```java
public interface IService {
    void execute(String message);
}
```

定义动态代理的回调类：

```java
public class ServiceInvocationHandler implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("invoke method=" + method);
        System.out.println("invoke args=" + Arrays.toString(args));
        return null;
    }
}
```

生成动态代理对象并执行调用方法：

```java
public class Main {
    public static void main(String... args) {
        IService service = (IService) Proxy.newProxyInstance(
                IService.class.getClassLoader()
                , new Class[]{IService.class}, //可以动态代理多个接口，不能超过65535个......
                new ServiceInvocationHandler());
        service.execute("hello world");
    }
}
```

运行程序，打印信息如下：

```bash
method=public abstract void proxy.IService.execute(java.lang.String)
args=[hello world]
```

#### 底层原理

JVM在运行期间自动生成了实现了代理接口的class字节码并加载运行，在 IDEA 或者 AS 上通过 `run->Edit Configurations` 添加如下虚拟机参数，JVM会把生成的代理class文件保存到磁盘中：

```bash
-Dsun.misc.ProxyGenerator.saveGeneratedFiles=true
```

如上面程序会在项目根目录的 `com.sun.proxy` 包名下生成 `$Proxy0.class` 文件，反编译这个class文件：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package com.sun.proxy;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import proxy.IService;
//生成的代理类需要继承 Proxy 类，由于 Java 无法做到多继承，所以动态代理无法代理类，只能代理接口
public final class $Proxy0 extends Proxy implements IService {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    //带有 InvocationHandler 参数的构造函数
    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void execute(String var1) throws  {
        try {
            //方法执行时委托给 InvocationHandler
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("proxy.IService").getMethod("execute", Class.forName("java.lang.String"));
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

## GC

