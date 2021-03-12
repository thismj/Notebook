# Android Binder

Binder 是 Android 系统实现进程间通信（IPC）的方式之一，它基于开源的 [OpenBinder](https://link.zhihu.com/?target=http%3A//www.angryredplanet.com/~hackbod/openbinder/docs/html/BinderIPCMechanism.html)，是一种 C/S 架构的通信方式。相比于 Linux 的管道、消息队列、Socket共享内存等方式相比，Binder 具有以下优势：

1. 性能：传输数据只需要拷贝一次，优于管道、消息队列、Socket（两次） 等，仅次于共享内存（无需拷贝）
2. 稳定性：基于 C/S 架构，通信双方相对独立，稳定性较好，优于共享内存（资源同步问题）
3. 安全性，在内核中根据 UID 来鉴别进程的身份，进行系统权限控制，并且可以建立私有通道（匿名Binder）
4. 语言层面角度：Binder机制比较契合面向对象的思想



## 相关类职能（Java层）

`IBinder`：跨进程通信对象的接口，实现该接口就具备了跨进程通信的能力

`Binder`：跨进程通信对象的基类，实现了 `IBinder` 接口，代表 Server 端本地的 Binder 对象

`BinderProxy`：`Binder` 的内部类，也实现了 `IBinder` 接口，代表 Client 端的 Binder 代理对象。在跨进程通信时，Binder 驱动会完成  `Binder`

跟 `BinderProxy` 的相互转换。

`IInterface`：Server 端本地的 Binder 类需要实现该接口，代表它能提供哪些方法（服务）给客户端。

`ServiceManager`：添加和获取实名 Binder 对象，相当于 DNS，把字符串形式的 Binder 名字转换为 Binder 对象的引用。



## 实现Binder

定义接口，表明 Server 端提供的方法：

```kotlin
interface ServerInterface : IInterface {
    /**
     * 提供两个数字相加并返回结果的方法
     */
    fun plus(a: Int, b: Int): Int
}
```

定义 Client 端的 Binder 类，实现上述定义的 ServerInterface  接口：

```kotlin
/**
 * @param remote 通过Binder驱动把Server端Binder转换而来的BinderProxy
 */
class ClientBinder(private val remote: IBinder) : ServerInterface {
    companion object {
        //定义plus方法的transaction code
        const val PLUS_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION
    }
  
    override fun plus(a: Int, b: Int): Int {
        //Client向Server端跨进程写数据
        val data = Parcel.obtain()
        //Server向Client端跨进程回数据
        val reply = Parcel.obtain()
        val result: Int
        try {
            //向Server端写入描述符，表明自己想调用的Server端Binder对象
            data.writeInterfaceToken(DESCRIPTOR)
            //向Server写入参数a，b
            data.writeInt(a)
            data.writeInt(b)
            //通过代理对象BinderProxy远程调用Server端的接口，PLUS_TRANSACTION作为调用plus方法的映射
            remote.transact(PLUS_TRANSACTION, data, reply, 0)
            //远程调用返回，读取Server端是否写入了异常
            reply.readException()
            //读取Server返回的结果
            result = reply.readInt()
        } finally {
            //回收Parcel
            data.recycle()
            reply.recycle()
        }
        return result
    }

    override fun asBinder(): IBinder {
        return remote
    }
}
```

定义 Server 端的 Binder 类：

1. 继承自 Binder 基类，表明这是一个 Server 端的 Binder 对象，且具备跨进程通信的能力
2. 实现上面定义的 ServerInterface 接口

```kotlin
class ServerBinder() : android.os.Binder(), ServerInterface {
    companion object {
        //定义该Binder的描述符
        const val DESCRIPTOR = "cn.thismj.android.demo.binder.ServerBinder"
    }

    init {
        //attch Server本地Binder对象以及描述符，与Client端的代理对象BinderProxy做区分
        attachInterface(this, DESCRIPTOR)
    }

    override fun asBinder(): IBinder {
        return this
    }
    
  /**
   * Client端的代理对象BinderProxy通过远程调用transact()方法最终经过Binder驱动的转换会调用到Server端本地Binder对象的onTransact方法
   */
    override fun onTransact(code: Int, data: Parcel, reply: Parcel?, flags: Int): Boolean {
        when (code) {
            //code为PLUS_TRANSACTION，表明Client端想调用plus()方法
            PLUS_TRANSACTION -> {
                //验证Client端确实是想调用该Binder，如果描述符对不上，会抛异常SecurityException
                data.enforceInterface(DESCRIPTOR)
                //读取Client端传过来的参数a、b
                val a = data.readInt()
                val b = data.readInt()
                //执行Server端实现的plus()方法
                val result = plus(a, b)
                //表示Server正常执行，向Client端写入无异常的数据
                reply?.writeNoException()
                //向Client端返回结果
                reply?.writeInt(result)
                return true
            }
        }
        return super.onTransact(code, data, reply, flags)
    }

    /**
     * Server端实现功能接口
     */
    override fun plus(a: Int, b: Int): Int {
        return a + b
    }
}
```

通过远程 Service 跨进程调用：

```kotlin
    private val serviceConnection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            Log.d(TAG, "onServiceConnected $service")
            val clientBinder = ClientBinder(service!!)
            Log.d(TAG, "1+2=${clientBinder.plus(1, 2)}")
        }

        override fun onServiceDisconnected(name: ComponentName?) {
        }
    }
```

查看打印数据：

```bash
2021-03-11 21:25:27.650 30962-30962/? D/MainActivity: onServiceConnected android.os.BinderProxy@498eed6
2021-03-11 21:25:27.651 30962-30962/? D/MainActivity: 1+2=3
```

## transaction code

transaction code 定义为 Binder 跨进程通信时的方法标识，其范围为 [0x00000001～0x00ffffff]，Binder定义了一些默认内置的 transaction code ：

```java
public interface IBinder {
    /**
     * The first transaction code available for user commands.
     */
    int FIRST_CALL_TRANSACTION  = 0x00000001;
    /**
     * The last transaction code available for user commands.
     */
    int LAST_CALL_TRANSACTION   = 0x00ffffff;
    /**
     * IBinder protocol transaction code: pingBinder().
     */
    int PING_TRANSACTION        = ('_'<<24)|('P'<<16)|('N'<<8)|'G';
    
    /**
     * 打印Binder内部状态
     */
    int DUMP_TRANSACTION        = ('_'<<24)|('D'<<16)|('M'<<8)|'P';
    
    /**
     * IBinder protocol transaction code: execute a shell command.
     * @hide
     */
    int SHELL_COMMAND_TRANSACTION = ('_'<<24)|('C'<<16)|('M'<<8)|'D';

    /**
     * 询问服务端Binder的描述符，例如上面例子即返回 "cn.thismj.android.demo.binder.ServerBinder"
     */
    int INTERFACE_TRANSACTION   = ('_'<<24)|('N'<<16)|('T'<<8)|'F';
    
}
```

 

## Binder跨进程通信原理

Android 通过 Linux 内核的**动态内核可加载模块**（Loadable Kernel Module，LKM）的机制，添加了 Binder 驱动，用户进程之间通过这个 Binder 驱动作为桥梁来实现通信。

操作系统从逻辑上将虚拟空间划分为用户空间（User Space）和内核空间（Kernel Space）, 并且进程与进程间内存是不共享的，传统的 IPC 方式需要经历两次拷贝：

<img src="https://markdown-1258186581.cos.ap-shanghai.myqcloud.com/20190618235640.png" style="zoom:30%;" />

Binder IPC 方式基于内存映射（mmap）来实现，只需要拷贝一次：

![](https://markdown-1258186581.cos.ap-shanghai.myqcloud.com/20190618235518.png)

## 参考资料

[为什么 Android 要采用 Binder 作为 IPC 机制？](https://www.zhihu.com/question/39440766)

[写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)

[Android Bander设计与实现 - 设计篇](https://blog.csdn.net/universus/article/details/6211589)

