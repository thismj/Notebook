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

## 集合

List转map



##异常

[浅谈Kotlin的Checked Exception机制](https://blog.csdn.net/guolin_blog/article/details/108817286)



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

`withContext` 串行

```kotlin
fun main() = runBlocking {
    val time = measureTimeMillis {
        val one: Int = withContext(Dispatchers.Default) {
            delay(1000)
            13
        }
        val two: Int = withContext(Dispatchers.Default) {
            delay(1000)
            14
        }
        println("sum is ${one + two}")
    }
    println("cost $time ms")
}
```

输出：

```bash
sum is 27
cost 2027 ms
```



`asyn` 并发

```kotlin
fun main() = runBlocking {
    val time = measureTimeMillis {
        val one: Deferred<Int> = async {
            delay(1000)
            13
        }
        val two: Deferred<Int> = async {
            delay(1000)
            14
        }
        println("sum is ${one.await() + two.await()}")
    }
    println("cost $time ms")
}
```

输出：

```bash
sum is 27
cost 1024 ms
```


|  关键函数   | 返回  | 上下文 | 作用域 |
|  ----  | ----  | ---- | ---- |
| launch  | Job |可选CoroutineContext参数|可以全局开启协程 GlobalScope.launch{}|
| runBlocking  | lambda返回值 |可选CoroutineContext参数|可以全局开启协程 runBlocking{}|
| withContext  | lambda返回值 |必须指定CoroutineContext参数|只能在协程作用域里面调用|
| withTimeout  | lambda返回值 |承袭外作用域CoroutineContext|只能在协程作用域里面调用|
| withTimeoutOrNull | lambda返回值或者超时后返回null | 承袭外作用域CoroutineContext |只能在协程作用域里面调用|
| async  | Deferred<lambda返回值类型> |可选CoroutineContext参数|只能在协程作用域里面调用|


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

---

[协程上下文与调度器](https://www.kotlincn.net/docs/reference/coroutines/coroutine-context-and-dispatchers.html)

调度器使用

```kotlin
fun main(): Unit = runBlocking {
    var index = 0
    val fixThreadDispatcher =
        Executors.newFixedThreadPool(3) { r -> Thread(r).also { it.name = "fixSize${++index}" } }
            .asCoroutineDispatcher()

    val singleThreadDispatcher =
        Executors.newSingleThreadExecutor { r -> Thread(r).also { it.name = "single" } }
            .asCoroutineDispatcher()

    coroutineScope {
        launch {
            println("A ${Thread.currentThread()}")
        }

        launch(Dispatchers.Unconfined) {
            println("B ${Thread.currentThread()}")
        }

        launch(Dispatchers.Default) {
            println("C ${Thread.currentThread()}")
        }

        launch(Dispatchers.IO) {
            println("D ${Thread.currentThread()}")
        }

        launch(singleThreadDispatcher) {
            println("E ${Thread.currentThread()}")
        }

        launch(fixThreadDispatcher) {
            println("F ${Thread.currentThread()}")
        }

        launch(fixThreadDispatcher) {
            println("G ${Thread.currentThread()}")
        }

        launch(fixThreadDispatcher) {
            println("H ${Thread.currentThread()}")
        }

        launch(fixThreadDispatcher) {
            println("I ${Thread.currentThread()}")
        }
    }

    singleThreadDispatcher.close()
    fixThreadDispatcher.close()
}
```

输出

```bash
B Thread[main @coroutine#3,5,main]
C Thread[DefaultDispatcher-worker-1 @coroutine#4,5,main]
D Thread[DefaultDispatcher-worker-1 @coroutine#5,5,main]
E Thread[single @coroutine#6,5,main]
F Thread[fixSize1 @coroutine#7,5,main]
G Thread[fixSize2 @coroutine#8,5,main]
H Thread[fixSize3 @coroutine#9,5,main]
I Thread[fixSize1 @coroutine#10,5,main]
A Thread[main @coroutine#2,5,main]
```

为什么 A 最后打印？A 协程没有指定上下文，在这里会继承父协程 runblocking 的调度器即 `BlockingEventLoop`，所以需要等父协程执行完毕才会轮到 A 协程执行，即 `fixThreadDispatcher.close()` 执行完毕之后，A协程才会被 BlockingEventLoop 调度执行。

上下文调度器比较：

|  调度器   | 说明  |  |
|  ----  | ----  | ---- |
| 不指定 | GlobalScope.launch则默认为Dispatchers.Default，其它情况继承自父协程 | |
| Dispatchers.Default  | 默认的调度器，核心线程数为cpu核心数（最低为2）最大线程数为cpu核心数*128，但是需要在[核心线程数，（1 shl 21）-2 = 2097150] 范围内 | |
| Dispatchers.Unconfined  | 非受限的调度器，在调用线程执行initial continuation（第一个挂起点之前），之后由继承自外部的调度器执行，通常代码不应该使用，因为在第一个挂起点之前不用进行协程调度 | |
| Dispatchers.IO | 也是用Dispatchers.Default调度，但是会限制并发数（默认为64），可以通过`kotlinx.coroutines.io.parallelism` 指定 | |
| Dispatchers.Main  | Android平台特有的主线程调度器 | |
| Executors.newSingleThreadExecutor().asCoroutineDispatcher()  | 单个线程的线程池调度器 | |
| Executors.newFixedThreadPool(x).asCoroutineDispatcher() | 固定线程数的线程池调度器 | |

IDEA&Android Studio 的 Kotlin 插件带有协程调试器，可以在 **Coroutines** 标签中查看各个调度器下的协程的状态，比如 `runBlocking` 这个协程的调度器叫 `BlockingEventLoop`，AS 进入 run->Edit Configuration 指定 JVM 参数 `-Dkotlinx.coroutines.debug` 开启调试选项

![](https://www.kotlincn.net/assets/externals/docs/reference/coroutines/images/coroutine-idea-debugging-1.png)

协程之间的关系，在父协程的 CoroutineScope 启动新的协程时，则新的协程通过 `CoroutineScope.coroutineContext` 继承了父协程的上下文调度器，为父协程的子协程，当父协程被取消时，所有的子协程都会被递归地取消，当父协程 `join` 的时候，会等待其所有的子协程执行完毕。而在 CoroutineScope 下用 GlobalScope 来启动一个新协程时，这个新协程是独立运行的，外层的协程取消时对它没有影响。

```kotlin
fun main() = runBlocking {

    val parent = launch {
        GlobalScope.launch {
            println("independent start")
            delay(1000)
            println("independent end")
        }

        launch {
            println("child start")
            delay(1000)
            println("child end")
        }
    }

    delay(500)
    parent.cancel()
    delay(1000)
    println("child not end??")
}
```

输出：

```bash
independent start
child start
independent end
child not end??
```

如何同时指定协程名字以及上下文调度器？CoroutineContext 重写了 `+` 操作符，可以通过相加的方式合并多个 CoroutineContext，CoroutineScope 也扩展了 `+` 方法，可以直接跟 CoroutineContext 相加得到新的 CoroutineScope 作用域。

```kotlin
val scope = CoroutineScope(Dispatchers.Default) + CoroutineName("Scope Coroutine")

fun main(){
    GlobalScope.launch(Dispatchers.Default + CoroutineName("Launch Coroutine")) {
        println("A ${Thread.currentThread()}")
    }

    scope.launch {
        println("B ${Thread.currentThread()}")
    }
    
    Thread.sleep(500)
}
```

输出：

```bash
A Thread[DefaultDispatcher-worker-1 @Launch Coroutine#1,5,main]
B Thread[DefaultDispatcher-worker-2 @Scope Coroutine#2,5,main]
```

---

如何用挂起函数封装现有的回调接口：

```kotlin
fun main() = runBlocking {
    val job = launch {
        try {
            val data = getData()
            println("data=$data")
        } catch (e: Exception) {
            println("e=$e")
        }
    }
    delay(1000)
    println("after 1s")
    job.join()
    println("end!!")
}

suspend fun getData() = suspendCoroutine<Int?> { continuation ->
    getData(object : Callback {
        override fun onSuccess(value: Int) {
            continuation.resume(value)
        }

        override fun onFailed(throwable: Throwable) {
            continuation.resumeWithException(throwable)
        }
    })
}

//模拟Java的回调方式，在新线程阻塞2s，返回数据
fun getData(callback: Callback) {
    Thread {
        Thread.sleep(2000)
        val random = (0..10).random()
        if (random >= 5) {
            callback.onSuccess(random)
        } else {
            callback.onFailed(IllegalAccessException("random is $random"))
        }
    }.start()
}

interface Callback {
    fun onSuccess(value: Int)
    fun onFailed(throwable: Throwable)
}
```

正常输出：

```bash
after 1s
data=7
end!!
```

异常输出:

```bash
after 1s
e=java.lang.IllegalAccessException: random is 3
end!!
```

有个问题，如果外部协程取消了，`getData(callback: Callback)` 方法还是会继续执行，无法感知到协程的取消，此时可以用 suspendCancellableCoroutine 来感知协程的取消，从而取消对应的异步执行逻辑

```kotlin
suspend fun getData() = suspendCancellableCoroutine<Int?> { continuation ->
    continuation.invokeOnCancellation { 
        //执行的协程取消了
        println("coroutine canceled!")
        //取消下面异步执行的逻辑
    }
    getData(object : Callback {
        override fun onSuccess(value: Int) {
            continuation.resume(value)
        }

        override fun onFailed(throwable: Throwable) {
            continuation.resumeWithException(throwable)
        }
    })
}
```

### 异步流（Flow）

先看如何创建一个同步流，在主线程通过阻塞线程模拟获得一个同步的事件流，sequence传入的 lambda SequenceScope作为接受者，内部不允许调用其它挂起函数。

```kotlin
fun main() {
    sequence {
        for (i in 1..3) {
            Thread.sleep(1000)
            println("yield i=$i ${Thread.currentThread()}")
            yield(i)
        }
    }.forEachIndexed { index, i ->
        println("index=$index i=$i ${Thread.currentThread()}")
    }
}
```

输出

```bash
yield i=1 Thread[main,5,main]
index=0 i=1 Thread[main,5,main]
yield i=2 Thread[main,5,main]
index=1 i=2 Thread[main,5,main]
yield i=3 Thread[main,5,main]
index=2 i=3 Thread[main,5,main]
```

创建异步流：

```kotlin
fun main() = runBlocking {
    flow {
        for (i in 1..3) {
            try {
                //getData()是上面定义的挂起函数
                val data = getData()
                println("emit $data ${Thread.currentThread()}")
                emit(data)
            } catch (e: IllegalAccessException) {
                println("invalid data ${Thread.currentThread()}")
            }
        }
        //指定异步流发送的协程
    }.flowOn(Dispatchers.Default)
        .collect {
            println("collect $it ${Thread.currentThread()}")
        }
}
```

输出，在 worker线程协程#2 发送数据流，在主线程协程#1收集数据流；流的发送总是在调用的协程上下文执行，不能通过 withContext 等切换协程，只能通过 flowOn 方法切换发送数据的协程。

```bash
emit 6 Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main]
collect 6 Thread[main @coroutine#1,5,main]
emit 9 Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main]
collect 9 Thread[main @coroutine#1,5,main]
invalid data Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main]
```

创建 Flow 的方式：

```kotlin
    flow {
        for (i in 1..3) {
            emit(i)
        }
    }

    flowOf(1, "2", 6.0f)

    (1..3).asFlow()

    listOf(2, "3").asFlow()
```

Flow的对比：

| Flow   | 用法 |
| ------ | ---- |
| flow | 基本的流构建器，`emit()`发送数据，是 suspend 的，所有数据的发送都在同一个协程，只能通过 `flowOn` 切换发送协程 |
| channelFlow{} | 通道流构建器，`send()`发送数据，是 suspend 的；`offer()` 非 suspend 的，只要在发送buffer范围内，就能offer成功，否则失败。可以在多个协程里面发送数据，是并发的，可以使用 `withContext`切换协程发送数据 |
| callbackFlow{} | 实际也是 ChannelFlow，只是命名不一样来代表不同的意图，很明显这个专门用来在 Callback 回调中实现异步流的，并且从1.3.4开始，callbackFlow 必须显式调用 `awaitClose()` |



过渡流操作符

|  操作符 | 作用|
| -- | --|
| filter |  过滤数据流 |
| distinctUntilChanged |   过滤相同数据 |
| transform |  转换，一对多映射，可以转换为多个其它数据 |
| map |  数据转换，一对一映射，转换为一个其它数据     |
| flatMapConcat |  数据转换，一对一映射，映射为一个新的数据流Flow，顺序收集，等新的Flow收集完毕再发送下一个值     |
| flatMapMerge |  数据转换，一对一映射，映射为一个新的数据流Flow，并发收集，可以指定最大的并发流的个数    |
| take |  限制数据流长度    |
| conflate |  当收集比发送慢时，只收集目前最新的值  |
| zip |  组合多个流，每个流都有新值产生的时候再组合 |
| combine |  组合多个流，与zip的区别在于无论哪个数据流产生了新的值都进行组合 |
| filter |  过滤数据流 |
| buffer |  当收集比发送慢时，开启新的协程去做发送操作，并缓存数据流，提升效率 |
| mapLatest、transferLatest、collectLatest、flatMapLatest |  如果有新的数据过来，但是上次的数据还在处理（挂起），则丢弃这次数据，重新发送最新的值 |
| onEach | 传入一个lambda，对发送的每个数据做一些操作  |

末端流操作符

| 操作符       | 作用                                                   |
| ------------ | ------------------------------------------------------ |
| collect      | 收集数据流                                             |
| toList/toSet | 数据流转换为集合                                       |
| reduce       |                                                        |
| fold         |                                                        |
| launchIn     | 在单独的协程中收集流，对比 flowOn 在单独的协程中发送流 |
|              |                                                        |
|              |                                                        |
|              |                                                        |

流异常
| 操作符       | 作用             |
| ------------ | ---------------- |
| catch      | 捕获并处理上游异常       |
| onCompletion | 数据流发送并收集完成（正常或者异常），能观察到所有异常，但是不处理|
| cancellable | 协程只有在挂起状态才能被cancel，所以异步流也一样，可以通过这个操作符在不挂起的情况下，也可以直接cancel |

### 通道Channel

类似于 BlockingQueue，提供了非阻塞的 `send` 和 `receive` 方法，生产者消费者模型，扇出即一对多的生产者消费者模式，扇入即多对一的生产者消费者模式。通道可以指定缓冲区，缓冲区内数据无需等待消费者。



## 注解

### @RestrictsSuspension

初步理解，当一个类或接口具有该注解并作为函数的接受者时，在函数里面只能调用自己的成员挂起函数或扩展挂起函数，不能调用任意其它的挂起函数。



## 其它

获取随机数：
