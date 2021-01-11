# Kotlin

Kotlin 名字来源于俄罗斯芬兰湾中一个岛屿的名字，全称是 Kotlin Island，是英语「科特林岛」之意。

## 变量和属性

Java 中的一些关于变量的概念定义：

| 概念                           | 描述                                             |
| ------------------------------ | ------------------------------------------------ |
| *Field*（*字段*）              | 类的成员变量，包括非静态的实例变量、静态的类变量 |
| *Local Variable*（*局部变量*） | 方法体或者静态代码块内定义的变量                 |
| *Parameter*（*参数*）          | 方法调用声明的参数                               |

什么是属性？属性的概念比较抽象，它是一个用户可以改变的对象特征，其本质上而言是可以从对象外部公开访问的Field，Kotlin的属性提供了这样一种一致性，即对象的属性只能通过访问器访问。

Kotlin 用关键字 var 定义可变属性，可以重新赋值；val 定义不可变属性，只能赋值一次。属性可以声明在代码文件顶层，作为顶层属性。

声明一个属性的完整写法：

```kotlin
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]
```

为什么会有 *幕后字段* ？首先，在 Kotlin 中得把属性跟 Java 中字段的含义区分开，不管是在类内部还是外部，属性都是通过 getter、setter 来访问的，它不是字段，举个例子：

在 Kotlin 中定义一个属性 name，此时自定义实现它的 getter、setter 访问器，并且不使用 field 字段，此时查看字节码，是没有生成 name 这个字段的，而只有 getName、setName 两个方法而已，从这儿可以看出，虽然有 name 这个属性，但是并没有 name 这个字段。但是不能说它们没有关系，只要 getter、setter 至少有一个使用了默认实现，或者在自定义的 getter、setter 中使用了 field 这个变量，此时就会生成一个 name 属性对应的 name 字段，而此字段则是 kotlin 中所谓的 *幕后字段*，并且幕后字段只能在 getter、setter 里面使用。如果 *幕后字段*  存在，并且没有指定为延迟初始化或延迟加载，也不是抽象的，则必须具有初始化器。

还有一个 *幕后属性* 的概念，官方文档举例的使用场景是，如果类中需要一个属性，对外表现为 val 只读，对内表现为 var 可读写，则可以这么写：

```kotlin
private var _table: Map<String, Int>? = null
public val table: Map<String, Int>
    get() {
        if (_table == null) {
            _table = HashMap() // 类型参数已推断出
        }
        return _table ?: throw AssertionError("Set to null by another thread")
    }
```

如果属性被声明为 private 的话，则默认不会生成 getter、setter，所以使用 *_table*  属性的时候不会产生函数调用开销。 结合 *幕后字段* 的理解，可以认为上述代码只会生成一个名字为 *_table*  的字段，以及一个 *getTable()*  的方法。

疑问，如果仅仅只是要属性对外只读对内可读写的话，下面的方法不是也可以吗：

```kotlin
var table: Map<String, Int>? = null
private set
```

## 

## 协程

### 相关概念

[聊聊协程的发展历程](https://juejin.cn/post/6844904040170520590)

协程最早诞生于 1958 年，被应用于汇编语言中，完整定义在 1963 年；线程是伴随着操作系统的出现，于 1967 年被提出。

协程是协作式多任务，通过代码执行的恢复和暂停来实现多任务程序；线程是操作系统调度的最小执行单元，属于抢占式的多任务。

协程的调度切换由用户控制，运行在用户态；线程的调度切换由操作系统控制，运行在内核态。

协程是基于语言层面的，并没有定义统一的接口，因此在不同的编程语言中的实现以及使用差异很大；线程通过操作系统的统一接口，定义了大体相同的线程使用方式，使不同的编程语言都对线程的使用是基本一致。

线程的痛点：两个线程之间如何方便的交互，callback的形式会使代码难以阅读和理解，最终让项目变得难以维护！！

协程：使用同步的方式来写异步的代码，易于理解和维护

在协程诞生之初，只是用来解决编程中的某些特殊问题的编程组件，它的多任务更像多个函数的组合协作执行，那个时候，协程其实更像是一种具备暂停恢复的函数。但是这种功能似乎并不受欢迎，因此协程在很长一段时间内都是比较小众的。（此时协程和线程关系并不大）

如今它成为底层支持多线程的协作式多任务组件，很好的解决了线程协作的痛点，同时也逐渐变得越来越受欢迎，协程和线程的关系更加亲密，它们似乎也变得更加相似。（如今你可以把协程视作一种轻量级线程）

Kotlin中的协程:

[开始使用Kotlin协程](https://www.jianshu.com/p/9f720b9ccdea)

一个协程被 suspension point 分割为多个Continuation，内部维护了一个具有 suspension point + 1（初始状态）个的状态的状态机，通过调用 Continuation 的 resume() 方法，就会执行到下一个 Continuation，每一个 Continuation 可以（指定）运行在不同的线程上。

### 基本使用

[协程基础](https://www.kotlincn.net/docs/reference/coroutines/basics.html)

```kotlin
fun main() {
    //Dispatchers指定为Unconfined
    GlobalScope.launch(Dispatchers.Unconfined) {
        //initial continuation 执行在主线程
        println("1 ${Thread.currentThread()} isDaemon=${Thread.currentThread().isDaemon}")
        //Continuation 1 不确定会在那个线程执行
        delay(1000)
        //Continuation 2 不确定会在那个线程执行
        println("2 ${Thread.currentThread()} isDaemon=${Thread.currentThread().isDaemon}")
    }
    println("3 ${Thread.currentThread()} isDaemon=${Thread.currentThread().isDaemon}")
    //因为协程默认创建的线程是守护线程，所以为了防止主线程执行完毕之后虚拟机退出，在此sleep 2s
    Thread.sleep(2000)
}
```

上面代码执行结果：

```bash
1 Thread[main @coroutine#1,5,main] isDaemon=false
3 Thread[main,5,main] isDaemon=false
2 Thread[kotlinx.coroutines.DefaultExecutor @coroutine#1,5,main] isDaemon=true
```

`launch` 方法传入的参数为类型为 `CoroutineScope.() -> Unit` 的 lambda 表达式，即该 lambda 中 `this` 为协程作用域对象（CoroutineScope）

---

runBlocking、coroutineScope共同点：这两个作用域里面的协程体以及所有子协程结束之后，才会执行外层后续的逻辑。

runBlocking、coroutineScope区别：

```kotlin
fun main() {
    GlobalScope.launch(Dispatchers.Unconfined) {
        coroutineScope {
            launch {
                println("A ${Thread.currentThread()}")
                delay(200)
                println("B ${Thread.currentThread()}")
            }
        }
        println("C ${Thread.currentThread()}")
    }
    println("D ${Thread.currentThread()}")
    Thread.sleep(500)
}
```

`coroutineScope`作用域不会阻塞主线程，所以 `delay` 挂起协程之后，D 在主线程得以打印，然后需要等 `coroutineScope` 里面所有的协程体以及子协程执行结束之后，才会打印 C，输出如下：

```bash
A Thread[main @coroutine#2,5,main]
D Thread[main,5,main]
B Thread[kotlinx.coroutines.DefaultExecutor @coroutine#2,5,main]
C Thread[kotlinx.coroutines.DefaultExecutor @coroutine#1,5,main]
```

```kotlin
fun main() {
    runBlocking {
        launch {
            println("A ${Thread.currentThread()}")
            delay(200)
            println("B ${Thread.currentThread()}")
        }
    }
    println("C ${Thread.currentThread()}")
}
```

`runBlocking` 作用域会阻塞主线程，所以协程 `delay` 挂起之后，协程不会释放主线程，而是使之阻塞，导致 C 无法提前打印，输出如下：

```bash
A Thread[main @coroutine#2,5,main]
B Thread[main @coroutine#2,5,main]
C Thread[main,5,main]
```

---

[协程的取消与超时](https://www.kotlincn.net/docs/reference/coroutines/cancellation-and-timeouts.html#%E5%8F%96%E6%B6%88%E4%B8%8E%E8%B6%85%E6%97%B6)：

`Job`是`launch`方法的返回值，它就是用来控制协程的运行状态的。`Job`中有几个关键方法：

- start。如果是`CoroutineStart.LAZY`创建出来的协程，调用该方法开启协程。
- cancel。取消正在执行的协程。如协程处于运算状态，则不能被取消。也就是说，只有协程处于阻塞状态时才能够取消。
- join。阻塞父协程，直到本协程执行完。
- cancelAndJoin。等价于`cancel` + `join`。

在协程作用域中，可以通过扩展属性 `isActive` 判断 `Job` 是否存活（即没有执行完成或者没有被取消）







