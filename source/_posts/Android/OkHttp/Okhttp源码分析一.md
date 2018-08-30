---
title: OkHttp源码分析(一)
date: 2018-08-14
tags:
- OkHttp
categories:
- 框架源码
---

<!-- toc -->

#### Call

> Call 在OKHttp中代表着一个已经准备好可以执行的Http请求；<br>
> Call请求是能够被取消的；<br>
> Cal代表着一个独立的request和response的Stream流，并且是不能够被执行两次的；

```
// 同步
OkHttpClient().newCall(Request.Builder().url("").build()).execute()

// 异步
OkHttpClient().newCall(Request.Builder().url("").build()).enqueue(object:Callback{
    override fun onFailure(call: Call?, e: IOException?) {
    }
    override fun onResponse(call: Call?, response: Response?) {

    }
})
```

在OKHttp中，无论是异步的请求还是同步的，最终都是转化为执行一个Call请求，而这个Call的真正实现类
实际上是RealCall；

1、OkHttpClient.newCall
```
/**
 * Prepares the {@code request} to be executed at some point in the future.
 */
 // 接收一个Request，并将其包装进RealCall中
@Override public Call newCall(Request request) {
  // 参数3用来指定使用使用WebSocket方式进行请求，默认是false
  return new RealCall(this, request, false /* for web socket */);
}
```

2、RealCall
```
RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
  // EventListener可以让我们更加细致的监听到Http请求的过程，但在3.7的版本中没有放出来
  // 原因下面说了，线程不安全；后续版本已经放出来给使用了
  final EventListener.Factory eventListenerFactory = client.eventListenerFactory();

  this.client = client;
  this.originalRequest = originalRequest;
  this.forWebSocket = forWebSocket;
  // 实现请求重试和重定向的拦截器
  this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);

  // 正如这里说明的，EventListener目前不放出来
  // TODO(jwilson): this is unsafe publication and not threadsafe.
  this.eventListener = eventListenerFactory.create(this);
}
```

3、RealCall.execute

> execute方法用来执行同步的网络请求；

```
@Override public Response execute() throws IOException {
  // 使用同步锁，标记executed，表示当前Call已经处于执行状态
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  // 捕获一个属于当前Call的stacktrace，用来打印调用栈日志，这可以帮助我们分析连接泄露的原因
  captureCallStackTrace();
  try {
    // 拿到Dispatcher，执行当前的Call请求
    // 实际上Dispatcher在执行同步请求时，是向加入到队列中进行等待，后续在拿出来执行
    // Dispatcher在OkHttpClient.Builder实例化时被创建
    client.dispatcher().executed(this);
    // 经过一系列拦截器之后，返回的执行结果
    Response result = getResponseWithInterceptorChain();
    if (result == null) throw new IOException("Canceled");
    return result;
  } finally {
    // 执行结束后，从执行队列中将Call移除掉
    client.dispatcher().finished(this);
  }
}
```

4、RealCall.enqueue

> enqueue方法用来执行异步的网络请求；

```
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  // 和同步最大的区别在此，将一个AsyncCall交给Dispatcher执行，实际上也是
  // 加到Dispatcher中的执行队列中进行执行
  // AsyncCall实际就是一个Runnable，可以猜测，在Dispatcher中肯定是要通过
  // 线程池来执行这个Runnable的
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

5、RealCall.AsyncCall

> AsyncCall是RealCall的内部类，用来包装异步回调；<br>
AsyncCall继承自NamedRunnable，实际就是一个指定当前线程名称的Runnable，并且在执行run方法时
会调用execute方法；<br>


```
final class AsyncCall extends NamedRunnable {
  private final Callback responseCallback;

  AsyncCall(Callback responseCallback) {
    super("OkHttp %s", redactedUrl());
    this.responseCallback = responseCallback;
  }

  String host() {
    return originalRequest.url().host();
  }

  Request request() {
    return originalRequest;
  }

  RealCall get() {
    return RealCall.this;
  }

  @Override protected void execute() {
    boolean signalledCallback = false;
    try {
      // 通过拦截器链调用后返回的Response
      Response response = getResponseWithInterceptorChain();
      // 如果一个请求已经被用户取消了
      // 在Call中调用Cancel方法，实际是执行retryAndFollowUpInterceptor的cancel方法
      if (retryAndFollowUpInterceptor.isCanceled()) {
        signalledCallback = true;
        // 调用responseCallback的回调方法
        responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
      } else {
        signalledCallback = true;
        // 调用responseCallback的回调方法
        responseCallback.onResponse(RealCall.this, response);
      }
    } catch (IOException e) {
      if (signalledCallback) {
        // Do not signal the callback twice!
        // 打印轨迹日志
        Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
      } else {
        // 调用responseCallback的回调方法
        responseCallback.onFailure(RealCall.this, e);
      }
    } finally {
      // 执行结束后，从执行队列中将Call移除掉
      client.dispatcher().finished(this);
    }
  }
}
```

_可以看到，在异步回调时，并没有为我们回调回主线程，所以我们在使用时需要注意线程问题；_

6、RealCall.getResponseWithInterceptorChain

> getResponseWithInterceptorChain方法中，为每一个Call请求增加了拦截器链，其中除了Ok默认
的拦截器，还有我们通过addInterceptor添加的应用拦截器和addNetworkInterceptor添加的网络
拦截器；

```
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  // 添加我们通过addInterceptor添加的应用拦截器
  interceptors.addAll(client.interceptors());
  // 添加失败重试和请求重定向拦截器
  interceptors.add(retryAndFollowUpInterceptor);
  // 添加请求头信息、对接收内容进行gzip压缩
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  // 对请求进行缓存管理
  interceptors.add(new CacheInterceptor(client.internalCache()));
  // 从连接池中获取一个有效的连接(可能是重用的，也可能是新建的)
  interceptors.add(new ConnectInterceptor(client));
  // 如果不是通过WebSocket协议连接的，则还会添加网络拦截器，即我们通过addNetworkInterceptor
  // 添加的拦截器
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors());
  }
  // 负责向服务器发起访问，并拿到最原始的返回数据往回传
  interceptors.add(new CallServerInterceptor(forWebSocket));

  // 创建拦截器链，用来处理每一个请求
  Interceptor.Chain chain = new RealInterceptorChain(
      interceptors, null, null, null, 0, originalRequest);
  // 开始执行调用链
  return chain.proceed(originalRequest);
}
```

#### Dispatcher请求任务分发器

> Dispatcher可以认为是Ok中，Call请求的分发、管理和执行的对象；

成员变量：
```
public final class Dispatcher {
  // 限制同时执行的请求数
  private int maxRequests = 64;
  // 限制同一个Host下同时能够执行的请求数
  private int maxRequestsPerHost = 5;
  // 当在执行的请求数量(包括同步和异步)为0时，将会回调这个Runnable
  private Runnable idleCallback;

  /** Executes calls. Created lazily. */
  // 执行异步请求的线程池
  private ExecutorService executorService;

  // 异步请求下，将会有两个队列来保存请求，一个用来保存正在执行的请求，
  // 当异步同时请求数量超过最大请求数时，Call请求将会被先放入另一队列中进行等待
  /** Ready async calls in the order they'll be run. */
  // 异步请求下，超过最大同时请求数时，将会把Call先暂时保存在这个队列中
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  // 异步请求下，用来保存正在执行的请求的队列
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  // 用来保存同步请求的队列
  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  ...
}
```

##### Dispatcher执行同步请求

在同步请求的情况下，我们RealCall.execute方法中，是通过调用：
```
client.dispatcher().executed(this);
```
来执行的。对应的Dispatch代码：
```
synchronized void executed(RealCall call) {
  // 将Call添加到同步队列中
  runningSyncCalls.add(call);
}
```
可以看到只是将Call添加到同步队列中；

Dispatcher对同步Call的支持，只是通过队列来进行同一管理请求，并没有其他特别的操作：

1、Dispatcher.cancelAll:取消OkHttp的所有网络请求；
```
public synchronized void cancelAll() {
  // 遍历异步准备队列，取消每一个请求
  for (AsyncCall call : readyAsyncCalls) {
    call.get().cancel();
  }

  // 遍历异步执行队列，取消所有请求
  for (AsyncCall call : runningAsyncCalls) {
    call.get().cancel();
  }

  // 遍历同步请求队列，取消所有请求
  for (RealCall call : runningSyncCalls) {
    call.cancel();
  }
}
```

2、Dispatcher.finish：在同步请求结束时，将其从队列中移除；
```
void finished(RealCall call) {
  finished(runningSyncCalls, call, false);
}

// 参数3promoteCalls，在异步情况下才为true，用来推动队列中异步请求的执行
private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
  // 记录当前正在执行请求数，包括同步和异步
  int runningCallsCount;
  // 当正在执行的请求数为0时，会执行该Runnable
  Runnable idleCallback;
  synchronized (this) {
    if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
    // 推动异步队列中请求的执行
    if (promoteCalls) promoteCalls();
    // 统计正在执行的请求数
    runningCallsCount = runningCallsCount();
    idleCallback = this.idleCallback;
  }

  // 当正在执行的请求数为0时，会执行该Runnable
  if (runningCallsCount == 0 && idleCallback != null) {
    idleCallback.run();
  }
}
```

3、获取正在执行的请求、统计正在执行的请求数；
```
// 拿到当前正在执行的所有请求，包括了异步和同步
public synchronized List<Call> runningCalls() {
  List<Call> result = new ArrayList<>();
  result.addAll(runningSyncCalls);
  for (AsyncCall asyncCall : runningAsyncCalls) {
    result.add(asyncCall.get());
  }
  return Collections.unmodifiableList(result);
}

// 统计正在执行的请求数
public synchronized int runningCallsCount() {
  return runningAsyncCalls.size() + runningSyncCalls.size();
}
```

##### Dispatcher执行异步请求

Dispatcher对于异步的请求，采用了两个队列来进行管理，一个用来保存正在执行的所有Call请求，因为
同一时间能够执行的Call请求的数量是受到限制的(maxRequests)，所以当请求数量操作了这个限制，
Dispatcher就会将多出的请求放入到等待队列中进行等待；

异步请求在RealCall.enqueue方法中被执行时：
```
client.dispatcher().enqueue(new AsyncCall(responseCallback));
```
对应的Dispatch代码：
1、Dispatcher.enqueue
```
synchronized void enqueue(AsyncCall call) {
  // 正在执行的异步请求数小于最大的限制数并且，同一个host的请求数也没有达到最大限制
  if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
    // 将异步请求添加到队列中
    runningAsyncCalls.add(call);
    // 通过线程池开启一个线程来执行Call请求
    executorService().execute(call);
  } else {
    // 如果正在执行的异步请求数多于限制数，则先放入到准备队列中
    readyAsyncCalls.add(call);
  }
}
```

2、Dispatcher.runningCallsForHost
```
private int runningCallsForHost(AsyncCall call) {
  int result = 0;
  // 根据异步请求Call的host来统计同一host下的请求数
  for (AsyncCall c : runningAsyncCalls) {
    if (c.host().equals(call.host())) result++;
  }
  return result;
}
```
3、Dispatcher.executorService

> Dispatcher通过创建一个线程池来执行异步请求；

```
public synchronized ExecutorService executorService() {
  if (executorService == null) {
    // 创建一个线程池实例
    // 参数1指定保留的空闲线程数为0
    // 参数2指定线程池创建的最大线程数为Integer.MAX_VALUE
    // 参数3、4指定每一个空闲线程存活的时间为60秒
    // 参数5指定了一个同步队列来管理所有需要线程执行的Runnable
    // 参数6指定了线程池的名称和创建的线程为非后台线程
    executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
  }
  return executorService;
}
```

_SynchronousQueue是一种无缓存的同步队列，当有一个线程添加进队列后，必须等待另一线程取出，才能
够继续入队。同样，一个线程要从队列中取出数据，也必须等待另一个线程来插入数据(可以认为队列的容量
为0)。这种无队列缓存的方式，能够让插入的数据以最快的方式被取出(无需进行入队和出队操作)，这在多任
务队列中是最快的任务处理方式。_

对于暂时保存在readyAsyncCalls的Call请求，Dispatcher通过promoteCalls方法来推动执行；

4、Dispatcher.promoteCalls
```
private void promoteCalls() {
  // 如果当前正在执行的异步请求数已经达到最大值，则不执行
  if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
  // 如果等待队列中没有请求需要执行，同样不执行该方法
  if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

  // 遍历等待队列的请求数，逐一取出
  for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
    AsyncCall call = i.next();

    // 判断当前取出的请求是否执行的条件是同一host执行的请求数是否超过最大限制
    if (runningCallsForHost(call) < maxRequestsPerHost) {
      // 没有操作，则从队列中移除
      i.remove();
      // 添加到执行队列中
      runningAsyncCalls.add(call);
      // 执行请求
      executorService().execute(call);
    }

    // 判断是否达到最大请求数了，是则退出
    if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
  }
}
```

有三种情况下，Dispatcher会调用promoteCalls方法来推动异步请求的执行，其中两种分别是调用者
使用Dispatcher设置自定义的最大请求数(maxRequests)和每个host的最大并发数请求(maxRequestsPerHost);

最后一种则是在有一个异步请求被结束时，会通过该方法，从准备队列中取出新的请求进行执行；

```
void finished(AsyncCall call) {
  finished(runningAsyncCalls, call, true);
}

// 异步请求被结束时，promoteCalls为true
private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
  int runningCallsCount;
  Runnable idleCallback;
  synchronized (this) {
    if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
    // 当结束了一个异步请求后，就会调用一个promoteCalls方法，推动准备队列中一个Call请求
    // 进入到执行队列中进行执行
    if (promoteCalls) promoteCalls();
    runningCallsCount = runningCallsCount();
    idleCallback = this.idleCallback;
  }

  if (runningCallsCount == 0 && idleCallback != null) {
    idleCallback.run();
  }
}
```

#### 总结

至此，最基本的OK源码执行流程分析完毕，对于异步的请求，大概的可以通过下图来进行概括：

![image](/images/Dispatcher.png)
![image](/images/okhttp异步请求.png)


参考：

https://www.jianshu.com/p/9deec36f2759
https://yq.aliyun.com/articles/78103?spm=a2c4e.11153940.blogcont78105.13.292a37bevfJZ5A
