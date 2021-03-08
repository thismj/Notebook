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

上面分析了异步请求的流程，如果是同步请求，则直接调用 `RealCall.execute()`：

```java
public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    timeout.enter();
    eventListener.callStart(this);
    try {
      //分发同步请求任务
      client.dispatcher().executed(this);
      //跟异步任务一样，通过责任链递归调用获取返回结果
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      e = timeoutExit(e);
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      //移出同步执行队列
      client.dispatcher().finished(this);
    }
  }
```

`Dispatcher.executed` ：

```java
  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    //直接添加进同步执行队列
    runningSyncCalls.add(call);
  }
```



总结流程图如下：

<img src="https://images2018.cnblogs.com/blog/809143/201808/809143-20180816162138233-707076029.png" style="zoom:80%;" />



### RetryAndFollowUpInterceptor

除了我们自定义的普通拦截器之外，这个拦截器在链中先被执行，它的作用是重试&跟进请求（RetryAndFollowUp，不好用一个词翻译啊），包含TCP连接失败、请求超时等的重试、资源重定向，服务器认证的跟进请求等等。

创建时机是在创建请求的时候 :

```java
  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    //创建RetryAndFollowUp拦截器
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    this.timeout = new AsyncTimeout() {
      @Override protected void timedOut() {
        cancel();
      }
    };
    this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
  }
```

`RetryAndFollowUpInterceptor.intercept(Chain chain)` :

```java
public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;
    //记录重试&跟进请求的次数
    int followUpCount = 0;
    //刚开始的时候肯定是null，当存在重试&跟进请求时会赋值为前一次请求的返回结果
    Response priorResponse = null;
    while (true) {
      if (canceled) {
        ////请求被外部取消
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      //释放连接标志位
      boolean releaseConnection = true;
      try {
        //正常递归地调用责任链
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        //连接服务器出错，此时请求还没发送，判断是否要重试
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getFirstConnectException();
        }
        //如果在外部设置了retryOnConnectionFailure为true，并且发生的异常不是永久性的，则尝试去重新连接，不释放资源
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        //跟服务器交互时出异常了，这种情况请求是可能已经发送出去了的
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        //同样的方法判断需不要重试
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      //如果当前请求结果之前存在重试&跟进，则把上一次的返回结果附加在当前结果里面
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Request followUp;
      try {
        //查看是否要进行跟进请求，主要有以下几种情况
        //1、http认证请求 401、407
        //2、外部设置了followRedirects为true，资源重定向 301、302、303（具有Location返回头）307、308（仅仅当重定向为GET、HEAD请求）等
        //3、外部设置了retryOnConnectionFailure为true，408错误码，服务器超时，还有一系列条件。。。
        //4、如果之前的请求正常，而本次返回了503 服务器维护，并且携带了返回头 "Retry-After=0"
        followUp = followUpRequest(response, streamAllocation.route());
      } catch (IOException e) {
        streamAllocation.release();
        throw e;
      }
      //followUp为null，说明没有后续的跟进请求，直接返回Response
      if (followUp == null) {
        streamAllocation.release();
        return response;
      }

      //followUp不为null，后续会执行跟进请求，关闭前一个返回结果的流资源
      closeQuietly(response.body());

      //重试&跟进请求次数超过了最大20次，就放弃了
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      //Response标记了UnrepeatableRequestBody，则不允许跟进请求
      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        //跟进的请求的主机是否与之前一致，如果不一致的话，不一致的话需要重新创建新的连接
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      //重置请求跟返回，开始下一次跟进请求
      request = followUp;
      priorResponse = response;
    }
  }
```

### BridgeInterceptor

用来处理设置一些 Request 未指定，但是 OkHttp 默认的请求头，例如 `Host`、`User-Agent`、`Accept-Encoding` 默认 gzip 编码；处理 Cookie 的保存和恢复

### CacheInterceptor

具体的缓存策略依赖于 HTTP 本身的缓存机制，并且只会缓存 GET 请求；采用 DiskLruCache 来作为磁盘缓存。

>1.通过Request尝试到Cache中拿缓存（里面非常多流程），当然前提是OkHttpClient中配置了缓存，默认是不支持的。
 2.根据response,time,request创建一个缓存策略，用于判断怎样使用缓存。
 3.如果缓存策略中设置禁止使用网络，并且缓存又为空，则构建一个Resposne直接返回，注意返回码=504
 4.缓存策略中设置不使用网络，但是又缓存，直接返回缓存
 5.接着走后续过滤器的流程，chain.proceed(networkRequest)
 6.当缓存存在的时候，如果网络返回的Resposne为304，则使用缓存的Resposne。
 7.构建网络请求的Resposne
 8.当在OKHttpClient中配置了缓存，则将这个Resposne缓存起来。
 9.缓存起来的步骤也是先缓存header，再缓存body。
 10.返回Resposne。



### ConnectInterceptor

用来和远端服务器建立连接，TCP、TLS的握手操作等

`StreamAllocation`：协调请求、连接、流，负责为一次请求(`Call`) 寻找连接(`Connection`)并建立流(`HttpCodec`)，从而完成远程通信。

`RouteSelector`：路由选择器，即对代理服务器、IP地址的选择

`Route`：路由实例，包含选择的代理服务器和IP地址。

`RealConnection`：连接(`Connection`)的实现类，对 Socket 的封装

`ConnectionPool`：连接(`Connection`)池，Socket连接复用，对于HTTP1.x来说，host相同直接复用；默认最多存在5个空闲的连接，如果在5分钟没有使用的话则会被释放

`HttpCodec`：负责 HTTP 请求的编码和返回的解码操作

### CallServerInterceptor

负责发送请求到服务端然后接受返回结果。

```java
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    realChain.eventListener().requestHeadersStart(realChain.call());
    //通过httpCodec写入请求头
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        //如果请求头包含 "Expect:100-continue" 则直接flush发送给服务端
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        //读取服务端的返回头，如果此处返回null，证明服务器返回了状态码100，客户端可继续发送请求
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        //继续发送请求
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        //写入请求body
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        //如果100 continue服务器未满足，禁止在这个连接上创建新流
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();

    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      //读取返回头，构建返回结果Response.Builder
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    //把请求信息、握手信息、返回时间等写入responseBuilder
    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    ......

    realChain.eventListener()
            .responseHeadersEnd(realChain.call(), response);

    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      //把返回体写入responseBuilder
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    ......
    //最终返回到上一个拦截器，即 ConnectInterceptor
    return response;
  }
```

