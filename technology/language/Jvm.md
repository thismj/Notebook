# JVM

## 内存模型

**JMM**：Java内存模型，即 `Java Memory Model`，JMM屏蔽掉各种硬件与操作系统的内存访问差异，让 Java 程序在各种平台下达到一致的内存访问效果



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

