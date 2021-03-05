# Android常用第三方库

## OkHttp

### 主流程

创建 `OkHttpClient`，通过 `newCall(Request request)` 方法传入`request` 请求参数新建 http 请求 `RealCall`。涉及到的设计模式有创建者模式（Builder）、外观模式。

```java
public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory {
  ......
  //请求策略、异步请求分发
  final Dispatcher dispatcher;
  ......
  //自定义的普通拦截器，首先执行
  final List<Interceptor> interceptors;
  //自定义的网络拦截器，在真正的网络请求拦截器CallServerInterceptor之前执行
  final List<Interceptor> networkInterceptors;
  //http请求事件流程监听
  final EventListener.Factory eventListenerFactory;
  ......
  public OkHttpClient() {
    this(new Builder());
  }

  OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    ......
    this.interceptors = Util.immutableList(builder.interceptors);
    this.networkInterceptors = Util.immutableList(builder.networkInterceptors);
    this.eventListenerFactory = builder.eventListenerFactory;
    ......
  }
  
  ......
  //新建http请求
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
  ......
  public static final class Builder {
    Dispatcher dispatcher;
    ......
    final List<Interceptor> interceptors = new ArrayList<>();
    final List<Interceptor> networkInterceptors = new ArrayList<>();
    EventListener.Factory eventListenerFactory;
    ......

    public Builder() {
      dispatcher = new Dispatcher();
      ......
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      ......
    }

    Builder(OkHttpClient okHttpClient) {
      this.dispatcher = okHttpClient.dispatcher;
      ......
      this.interceptors.addAll(okHttpClient.interceptors);
      this.networkInterceptors.addAll(okHttpClient.networkInterceptors);
      this.eventListenerFactory = okHttpClient.eventListenerFactory;
      ......
    }
    
    ......
      
    /**
     * Sets the dispatcher used to set policy and execute asynchronous requests. Must not be null.
     */
    public Builder dispatcher(Dispatcher dispatcher) {
      if (dispatcher == null) throw new IllegalArgumentException("dispatcher == null");
      this.dispatcher = dispatcher;
      return this;
    }
    
    ......
      
    public Builder addInterceptor(Interceptor interceptor) {
      if (interceptor == null) throw new IllegalArgumentException("interceptor == null");
      interceptors.add(interceptor);
      return this;
    }
    
    ......
      
    public Builder addNetworkInterceptor(Interceptor interceptor) {
      if (interceptor == null) throw new IllegalArgumentException("interceptor == null");
      networkInterceptors.add(interceptor);
      return this;
    }

    ......

    public Builder eventListenerFactory(EventListener.Factory eventListenerFactory) {
      if (eventListenerFactory == null) {
        throw new NullPointerException("eventListenerFactory == null");
      }
      this.eventListenerFactory = eventListenerFactory;
      return this;
    }

    public OkHttpClient build() {
      return new OkHttpClient(this);
    }
  }
}
```

`RealCall.newRealCall()`

```java
 private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    this.timeout = new AsyncTimeout() {
      @Override protected void timedOut() {
        cancel();
      }
    };
    this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
}

static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    //设置请求事件监听器
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
```

异步请求，`RealCall.enqueue()`

```java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      //通过同步机制来保证请求只能执行一次，否则抛异常
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    //回调http请求开始事件
    eventListener.callStart(this);
    //分发该请求，用AsyncCall包裹外部传入的Callback
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

`AsyncCall`继承自 `NamedRunnable`，间接实现了 `Runnable`接口：

``AsyncCall.java``:

```java
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;
    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }
    ......
  }
```

`NamedRunnable.java`

```java
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = Util.format(format, args);
  }

  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    //执行异步请求任务前修改当前线程名字为 "OkHttp "+请求的url
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      //异步请求执行完毕，把名字改回去
      Thread.currentThread().setName(oldName);
    }
  }
  //AsyncCall实现具体的执行方法
  protected abstract void execute();
}
```

`Dispatcher`类的部分成员变量：

```java
public final class Dispatcher {
  //最大并发请求数
  private int maxRequests = 64;
  //单个主机的最大并发请求数
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;

  //请求的线程池，懒加载方式
  private @Nullable ExecutorService executorService;
  //异步请求等待队列
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
  //异步请求执行队列
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();
  //同步请求执行队列
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
  ......
}
```

`Dispatcher.enqueue()`

```java
  void enqueue(AsyncCall call) {
    synchronized (this) {
      //添加进异步等待队列，有加锁同步
      readyAsyncCalls.add(call);
    }
    promoteAndExecute();
  }
```

`Dispatcher.promoteAndExecute()`

```java
private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));
   
    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      //遍历异步请求等待队列
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();
        //正在执行的异步请求超过最大并发数64，跳出循环
        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        //该异步请求的主机目前正在执行的请求超过5个，则跳过该异步请求继续遍历
        if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.
        i.remove();
        executableCalls.add(asyncCall);
        //添加进异步请求执行队列
        runningAsyncCalls.add(asyncCall);
      }
      //runningCallsCount()是此时正在执行的异步请求和同步请求的和
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      //新加进异步请求执行队列的任务开始丢进线程池
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }
```

`Dispatcher.executorService()`

```java
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      //懒加载方式，创建一个核心线程数为0，阻塞队列为非公平方式的同步队列，所有丢过来的任务按顺序执行
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```

`AsyncCall.executeOn`

```java
    void executeOn(ExecutorService executorService) {
      assert (!Thread.holdsLock(client.dispatcher()));
      boolean success = false;
      try {
        //任务丢进线程池
        executorService.execute(this);
        success = true;
      } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        //如果触发线程池的拒绝策略，则回调http失败事件
        eventListener.callFailed(RealCall.this, ioException);
        responseCallback.onFailure(RealCall.this, ioException);
      } finally {
        if (!success) {
          //执行没成功，从异步请求执行队列中移除
          client.dispatcher().finished(this); // This call is no longer running!
        }
      }
    }
```

当线程池调度该任务执行时，调用 `AsyncCall.execute()` 方法：

```java
  @Override protected void execute() {
      boolean signalledCallback = false;
      timeout.enter();
      try {
        //通过责任链模式递归调用所有拦截器，并返回请求结果Response
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          //一次http请求完成，回调请求结果
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        e = timeoutExit(e);
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        //从异步请求执行队列中移除
        client.dispatcher().finished(this);
      }
    }
```

`RealCall.getResponseWithInterceptorChain()`

```java
Response getResponseWithInterceptorChain() throws IOException {
    List<Interceptor> interceptors = new ArrayList<>();
    // 依次添加各个自定义的拦截器和内置的拦截器，先添加的先执行
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));
    //构建初始拦截器链对象，触发执行，参数0代表拦截器列表的index
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

`RealInterceptorChain.proceed`

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    ......

    //构建下一个拦截器链对象
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    //取出第index个拦截器
    Interceptor interceptor = interceptors.get(index);
    //调用拦截器的 intercept(Chain chain) 方法，这里是精髓，该方法传入了上面构建的Chain对象，如果该方法直接返回了Response对象，则调用结束，如果里面继续返回了 chain.proceed()，则会形成一个递归地调用，直到某个拦截器最终返回了真正的Response对象，这也称之为责任链模式
    Response response = interceptor.intercept(next);

    ......

    return response;
  }
```









