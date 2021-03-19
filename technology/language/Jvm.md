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

???

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

## 类加载机制

###双亲委托机制



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

