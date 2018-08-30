---
title: OkHttp源码分析(二)
date: 2018-08-16
tags:
- OkHttp
categories:
- 框架源码
---

<!-- toc -->

#### 拦截器

每一个Call请求都需要经过一些列的拦截器处理，Ok通过这些拦截器实现了对Call请求的重试、重定向、缓存
、设置请求头、压缩，甚至是最终对服务端的请求和接收返回数据也都是在拦截器中进行的；

RealCall.getResponseWithInterceptorChain
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
请求的发起和接收，都是在拦截器中进行的，不管是异步还是同步请求，都会调用该方法，来启动拦截器链；


#### RealInterceptorChain

```
public final class RealInterceptorChain implements Interceptor.Chain {
  // 当前Call请求对应的所有拦截器；
  private final List<Interceptor> interceptors;

  // 用来从连接池找到合适的连接(新建，重用)，创建数据流对象，用来进行数据通信
  private final StreamAllocation streamAllocation;

  // okhttp数据流对象,由StreamAllocation创建，Ok与服务端的通信就是通过这个对象来实现的
  // (Socket通信)
  private final HttpCodec httpCodec;

  // 与服务端的socket的链路，用来实现三次握手，建立通信
  private final RealConnection connection;

  // 表示当前正在执行的拦截器索引
  private final int index;

  // 本次Http请求
  private final Request request;

  // 用来标记同一个拦截器被执行的次数，同一个拦截器不能够被调用两次
  private int calls;

  // 第一次创建在RealCall中进行，此时只是提供了拦截器集合和Request，其他都是null
  // 这些参数在后面的拦截器流程中会被逐个进行创建
  public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, RealConnection connection, int index, Request request) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
  }

  @Override public Connection connection() {
    return connection;
  }

  public StreamAllocation streamAllocation() {
    return streamAllocation;
  }

  public HttpCodec httpStream() {
    return httpCodec;
  }

  @Override public Request request() {
    return request;
  }

  // 每一个拦截器的拦截流程，最终都是调用该方法用来推送下一个拦截器的执行
  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

  // RealInterceptorChain核心，整个拦截器链的调用是通过这个方法推动的
  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();
    // 记录当前拦截器被调用的次数
    calls++;

    // 判断当前连接的端口和host是否支持该url
    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // 如果同一个拦截器实例被调用超过1次，则抛出异常
    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // 为当前拦截器创建一个RealInterceptorChain，将Http请求所需的所有信息都包装进去
    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);

    // 拿到当前需要执行的拦截器
    Interceptor interceptor = interceptors.get(index);

    // 执行拦截器的拦截方法
    Response response = interceptor.intercept(next);

    // 检查当前拦截器是否在最后执行了Chain的proceed方法，来推动下一个拦截器的执行
    // 调用了proceed方法，RealInterceptorChain的calls变量应该等于1
    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // 检查每一个拦截器处理后返回的Response是否为null
    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
  }
}
```

RealInterceptorChain是Ok拦截器链，其中包含了Ok中所有的拦截器；

RealInterceptorChain贯穿了整个拦截链的始终，其中包含了Http请求需要的重要数据：request请求、
连接客户端(RealConnection)、以及连接和流(StreamAllocation)等；

每开启一个拦截器进行操作时，都会创建相应的RealInterceptorChain，用来将Http请求的数据封装，传
递给拦截器；

每一个拦截器的操作，都会通过RealInterceptorChain来获取其中需要的信息，并且最终会通过
RealInterceptorChain来推动，进入到下一个拦截器中进行拦截操作；

#### RetryAndFollowUpInterceptor

RetryAndFollowUpInterceptor是Call请求流程中第一个系统拦截器，其起到的主要作用是实现Call
请求的失败重试和重定向；

1、成员变量：
```
public final class RetryAndFollowUpInterceptor implements Interceptor {
  /**
   * How many redirects and auth challenges should we attempt? Chrome follows 21 redirects; Firefox,
   * curl, and wget follow 20; Safari follows 16; and HTTP/1.0 recommends 5.
   */
  // 最大的重定向和重试次数
  private static final int MAX_FOLLOW_UPS = 20;

  private final OkHttpClient client;
  // 是否使用WebSocket协议
  private final boolean forWebSocket;

  // 用来从连接池找到合适的连接(新建，重用)，创建数据流对象，用来进行数据通信
  private StreamAllocation streamAllocation;

  // 打印日志用的调用栈的跟踪对象
  private Object callStackTrace;

  // 标记当前请求是否已经取消了
  private volatile boolean canceled;
```

2、cancel

我们可以通过Call的Cancel方法来取消当前正在执行的请求，则这个Cancel方法实际上就是调用
RetryAndFollowUpInterceptor的cancel方法；
```
public void cancel() {
  // 标记当前的请求状态为取消
  canceled = true;

  // 通过StreamAllocation关闭连接和数据流
  StreamAllocation streamAllocation = this.streamAllocation;
  if (streamAllocation != null) streamAllocation.cancel();
}
```

3、createAddress

通过请求的链接信息(HttpUrl)，构造出即将访问的服务器地址信息，包含了主机名，端口等，如果是通过
代理来请求的，还会包含代理的信息；如果是通过Https来请求的，则会为其创建SSL验证信息；

```
private Address createAddress(HttpUrl url) {
  SSLSocketFactory sslSocketFactory = null;
  HostnameVerifier hostnameVerifier = null;
  CertificatePinner certificatePinner = null;
  // 如果是基于Https协议的链接，则为其创建相应的SSL协议证书验证
  if (url.isHttps()) {
    sslSocketFactory = client.sslSocketFactory();
    hostnameVerifier = client.hostnameVerifier();
    certificatePinner = client.certificatePinner();
  }
  // 构造一个代表服务端地址的信息类
  return new Address(url.host(), url.port(), client.dns(), client.socketFactory(),
      sslSocketFactory, hostnameVerifier, certificatePinner, client.proxyAuthenticator(),
      client.proxy(), client.protocols(), client.connectionSpecs(), client.proxySelector());
}
```

4、recover

当链接失败时，RetryAndFollowUpInterceptor拦截器会调用该方法尝试进行重连操作，如果请求带有
请求体，则只有在请求体已经被缓存的情况下，才能进行链接恢复。或者错误是在request请求被发送之前
出现的。

```
// 参数2表示当前request请求是否已经发送了
private boolean recover(IOException e, boolean requestSendStarted, Request userRequest) {
  // 向连接池抛出异常，并关闭Socket
  streamAllocation.streamFailed(e);

  // 如果我们在配置OK连接时已经设置失败不重连，则不进行重连
  // The application layer has forbidden retries.
  if (!client.retryOnConnectionFailure()) return false;

  // 如果请求已经发送了，并且这个request的请求体是不能重复使用的请求体(比如请求体是不进行缓存的
  // 请求)，则不进行重试
  // We can't send the request body again.
  if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;

  // 判断该异常是否是严重的IO异常，在该异常下是否能够连接重试
  // This exception is fatal.
  if (!isRecoverable(e, requestSendStarted)) return false;

  // 是否还有路由可以尝试重连
  // No more routes to attempt.
  if (!streamAllocation.hasMoreRoutes()) return false;

  // 可以进行连接重试，但是创建一个新的链接来重试路由选择器；
  // For failure recovery, use the same route selector with a new connection.
  return true;
}
```

5、isRecoverable

用来判断连接失败时，发出的异常是否是不可恢复的异常；

```
private boolean isRecoverable(IOException e, boolean requestSendStarted) {
  // If there was a protocol problem, don't recover.
  // 协议异常，不恢复重试
  if (e instanceof ProtocolException) {
    return false;
  }

  // 如果是中断异常，则检查是否是连接超时导致的，如果是并且存在下一个Route，则进行重试
  // If there was an interruption don't recover, but if there was a timeout connecting to a route
  // we should try the next route (if there is one).
  if (e instanceof InterruptedIOException) {
    return e instanceof SocketTimeoutException && !requestSendStarted;
  }

  // 证书安全导致的SSL握手异常，不进行恢复
  // Look for known client-side or negotiation errors that are unlikely to be fixed by trying
  // again with a different route.
  if (e instanceof SSLHandshakeException) {
    // If the problem was a CertificateException from the X509TrustManager,
    // do not retry.
    if (e.getCause() instanceof CertificateException) {
      return false;
    }
  }

  // 证书认证失败异常，不恢复
  if (e instanceof SSLPeerUnverifiedException) {
    // e.g. a certificate pinning error.
    return false;
  }

  // An example of one we might want to retry with a different route is a problem connecting to a
  // proxy and would manifest as a standard IOException. Unless it is one we know we should not
  // retry, we return true and try a new route.
  return true;
}
```

如果异常是在我们连接到代理时发生的，并且是一个标准的IO异常，则判断是否是isRecoverable方法中指定要
拦截的异常，如果不是，则尝试一个新的路由；

6、followUpRequest

该方法用来检查服务端返回的response的状态，根据状态构造出新的包含安全认证的header，或者进行重定
向，或者处理为客户端连接超时(重新执行request请求)。如果接收该response之后不需要进行后续操作（
即直接返回），则返回null，否则返回更新后的request进行重新请求。

```
private Request followUpRequest(Response userResponse) throws IOException {
  if (userResponse == null) throw new IllegalStateException();
  Connection connection = streamAllocation.connection();
  Route route = connection != null
      ? connection.route()
      : null;

  // 获取服务端response对应的响应码
  int responseCode = userResponse.code();

  // 获取到原先request的请求方法类型
  final String method = userResponse.request().method();
  switch (responseCode) {
    // request需要重新认证
    case HTTP_PROXY_AUTH:
      Proxy selectedProxy = route != null
          ? route.proxy()
          : client.proxy();
      if (selectedProxy.type() != Proxy.Type.HTTP) {
        throw new ProtocolException("Received HTTP_PROXY_AUTH (407) code while not using proxy");
      }
      return client.proxyAuthenticator().authenticate(route, userResponse);

    // request需要重新认证
    case HTTP_UNAUTHORIZED:
      return client.authenticator().authenticate(route, userResponse);

    // 不需要客户端处理的重定向请求
    case HTTP_PERM_REDIRECT:
    case HTTP_TEMP_REDIRECT:
      // "If the 307 or 308 status code is received in response to a request other than GET
      // or HEAD, the user agent MUST NOT automatically redirect the request"
      if (!method.equals("GET") && !method.equals("HEAD")) {
        return null;
      }
      // fall-through

    // 进行重定向
    case HTTP_MULT_CHOICE:
    case HTTP_MOVED_PERM:
    case HTTP_MOVED_TEMP:
    case HTTP_SEE_OTHER:
      // Does the client allow redirects?
      if (!client.followRedirects()) return null;

      // 获取重定向指定的url
      String location = userResponse.header("Location");
      if (location == null) return null;
      // 解析为对应的HttpUrl
      HttpUrl url = userResponse.request().url().resolve(location);

      // Don't follow redirects to unsupported protocols.
      if (url == null) return null;

      // 判断是否重定向的url与原来的url是否是相同的scheme(即http请求还是https请求)
      // 如果不相同，我们又配置了不允许带SSL验证的重定向，则不进行重定向
      // If configured, don't follow redirects between SSL and non-SSL.
      boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
      if (!sameScheme && !client.followSslRedirects()) return null;

      // 注意，这里是从原来的request复制出一份，所以配置是一样的
      // Most redirects don't include a request body.
      Request.Builder requestBuilder = userResponse.request().newBuilder();

      // 是否是允许带有请求体的请求方式(Post允许带有请求体，而Get不允许)
      if (HttpMethod.permitsRequestBody(method)) {
        // 是否允许在重定向过程中，保留之前请求的请求体
        final boolean maintainBody = HttpMethod.redirectsWithBody(method);

        // 通过Get方式进行重定向，肯定不需要保留请求体
        if (HttpMethod.redirectsToGet(method)) {
          requestBuilder.method("GET", null);
        } else {
          // 设置请求体
          RequestBody requestBody = maintainBody ? userResponse.request().body() : null;
          requestBuilder.method(method, requestBody);
        }

        // 没有请求体，则移除头部对应的请求体信息
        if (!maintainBody) {
          requestBuilder.removeHeader("Transfer-Encoding");
          requestBuilder.removeHeader("Content-Length");
          requestBuilder.removeHeader("Content-Type");
        }
      }

      // 移除掉原来请求header中的安全认证信息
      // When redirecting across hosts, drop all authentication headers. This
      // is potentially annoying to the application layer since they have no
      // way to retain them.
      if (!sameConnection(userResponse, url)) {
        requestBuilder.removeHeader("Authorization");
      }

      return requestBuilder.url(url).build();

    // 客户端认证超时，则进行重新请求
    case HTTP_CLIENT_TIMEOUT:
      // 408's are rare in practice, but some servers like HAProxy use this response code. The
      // spec says that we may repeat the request without modifications. Modern browsers also
      // repeat the request (even non-idempotent ones.)
      if (userResponse.request().body() instanceof UnrepeatableRequestBody) {
        return null;
      }

      return userResponse.request();

    default:
      return null;
  }
}
```

7、intercept

```
@Override public Response intercept(Chain chain) throws IOException {
  Request request = chain.request();

  // 此处创建了StreamAllocation，这个StreamAllocation在后续的拦截链流程都需要
  // 用到
  streamAllocation = new StreamAllocation(
      client.connectionPool(), createAddress(request.url()), callStackTrace);

  // 记录总的重定向的次数
  int followUpCount = 0;
  // 记录上一次请求返回的response
  Response priorResponse = null;
  while (true) {
    if (canceled) {
      streamAllocation.release();
      throw new IOException("Canceled");
    }

    Response response = null;
    boolean releaseConnection = true;
    try {
      // 推动一下个拦截器的执行，在此将StreamAllocation传递给了下一个拦截器，后续拦截器
      // 都可以拿到该实例
      response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);

      releaseConnection = false;
    } catch (RouteException e) {
      // Route匹配异常，此时request请求是还没有被发送出去的
      // The attempt to connect via a route failed. The request will not have been sent.
      // 检查在该异常下是否允许再进行重试操作
      if (!recover(e.getLastConnectException(), false, request)) {
        throw e.getLastConnectException();
      }
      releaseConnection = false;
      continue;
    } catch (IOException e) {
      // IOException异常下，可能已经向服务端发送了request请求
      // An attempt to communicate with a server failed. The request may have been sent.
      boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
      // 检查在该异常下是否允许再进行重试操作
      if (!recover(e, requestSendStarted, request)) throw e;
      releaseConnection = false;
      continue;
    } finally {
      // We're throwing an unchecked exception. Release any resources.
      if (releaseConnection) {
        streamAllocation.streamFailed(null);
        streamAllocation.release();
      }
    }

    // 注意，到这里，Call请求的整个拦截链已经走了一遍，又回到本拦截器中

    // 如果存在再次请求操作，则重新请求后，返回的response将会保存之前请求的response，这样
    // 多次请求之后，就形成了一个response链，通过当前的response可以查询获取到
    // 之前请求的response
    // Attach the prior response if it exists. Such responses never have a body.
    if (priorResponse != null) {
      response = response.newBuilder()
          .priorResponse(priorResponse.newBuilder()
                  .body(null)
                  .build())
          .build();
    }

    // 是否要进行再次请求(不一定是重定向响应导致的，其他情况也可能造成再次请求)
    Request followUp = followUpRequest(response);

    // 无需进行再次请求，则直接返回response
    if (followUp == null) {
      // 不是WebSocket协议，则直接释放连接
      if (!forWebSocket) {
        streamAllocation.release();
      }
      return response;
    }
    // 需要进行再次请求，则关闭掉当前response响应体的连接(IO连接等)；
    closeQuietly(response.body());

    // 检查总的请求的次数是否超过了最大限制
    if (++followUpCount > MAX_FOLLOW_UPS) {
      streamAllocation.release();
      throw new ProtocolException("Too many follow-up requests: " + followUpCount);
    }

    // 如果再次请求携带的之前的request的请求体是不能重复使用的
    if (followUp.body() instanceof UnrepeatableRequestBody) {
      streamAllocation.release();
      throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
    }

    // 再次请求的url与之前请求的url是同一个，则释放调用当前的连接
    // 重新创建一个
    if (!sameConnection(response, followUp.url())) {
      streamAllocation.release();
      streamAllocation = new StreamAllocation(
          client.connectionPool(), createAddress(followUp.url()), callStackTrace);
    } else if (streamAllocation.codec() != null) {
      throw new IllegalStateException("Closing the body of " + response
          + " didn't close its backing stream. Bad interceptor?");
    }

    // 保存最新的request请求和response
    request = followUp;
    priorResponse = response;
  }
}
```

RetryAndFollowUpInterceptor虽然是整个系统拦截链的第一个拦截器，但是其作用时期却是在整个Http
请求的最后环节；

RetryAndFollowUpInterceptor的一个作用就是捕获整个拦截器链处理过程的异常，判断该异常下是否
要重走整个请求流程(注意，不是请求重试和重定向，因为此时followUpCount是重新置为0的，相当于之前
的请求都抛弃了，重新再来)。

RetryAndFollowUpInterceptor的另一个作用是在服务端返回response，经过了多个拦截器处理之后，
在最终到达RetryAndFollowUpInterceptor这里时，解析response中的响应码，如果需要进行重新请求
(可能是要进行重定向，也可能是其他情况，比如客户端超时，导致要进行重新请求)，则会重新创建一个
request请求，进行请求，直到请求结束，则将response返回给客户端进行处理；

https://blog.csdn.net/sdfdzx/article/details/78186164
https://www.jianshu.com/p/e3b6f821acb8

https://yq.aliyun.com/articles/78104?spm=a2c4e.11153940.blogcont78101.13.30493cbfGzKQVY
https://www.jianshu.com/p/9deec36f2759
https://www.jianshu.com/p/6166d28983a2
https://www.jianshu.com/p/671a123ec163
https://www.jianshu.com/p/92ce01caa8f0

#### BridgeInterceptor

顾名思义，BridgeInterceptor起到桥接的作用。BridgeInterceptor将应用程序的用户请求数据转化为
对应网络请求数据；在网络请求之后，又将对应的服务端返回数据，转化为对应的用户相应数据；

BridgeInterceptor为request设置各种请求报头，从response中读取并保存cookie，并解压服务端返回
的gzip格式的数据作为response的相应体数据；


```
public final class BridgeInterceptor implements Interceptor {
  // Ok提供的Cookie管理类默认实现是NO_COOKIES，不接受任何cookie
  // 如果要实现cookie保存，则可以实现CookieJar，做自己的实现
  private final CookieJar cookieJar;

  public BridgeInterceptor(CookieJar cookieJar) {
    this.cookieJar = cookieJar;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    // 获取当前请求request，构造出一个副本
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    // 下面开始构造请求头

    // request带有请求体
    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      // 构造Content-Type
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      // 构造Content-Length
      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    // 补充Host
    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    // 补充Connection为Keep-Alive，即保持连接，是多路复用的重要步骤
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // 声明客户端支持的压缩编码格式
    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    // 根据url读取出本地保存的cookie信息，添加到请求头中
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      // 构造请求头
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    // 设置对应浏览器的User-Agent，ok有自己的User-Agent，比如"okhttp/3.7.0"
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    // request请求头已经设置完毕，构造出一个新的request，传递给下一个拦截器
    Response networkResponse = chain.proceed(requestBuilder.build());

    // 到此，已经完成了一次对服务端的访问，下面开始读取响应头信息

    // 解析请求头，如果服务端需要往客户端写cookie，则会在响应头中增加"Set-Cookie"，用来向客
    // 户端传递cookie信息；
    // 默认情况下，ok的cookie管理器是NO_COOKIES，即不接收任何cookie信息
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    // 构造出一个信息的response副本
    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    // 如果响应体中包含了gzip格式的压缩数据，则进行解压，重新赋给副本response的响应体
    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      // 这里用到一个新的开源库——Okio 来处理io数据流
      responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }

  // 根据本地读取到的cookie信息，构造一个cookie请求头
  /** Returns a 'Cookie' HTTP request header with all cookies, like {@code a=b; c=d}. */
  private String cookieHeader(List<Cookie> cookies) {
    StringBuilder cookieHeader = new StringBuilder();
    for (int i = 0, size = cookies.size(); i < size; i++) {
      if (i > 0) {
        cookieHeader.append("; ");
      }
      Cookie cookie = cookies.get(i);
      cookieHeader.append(cookie.name()).append('=').append(cookie.value());
    }
    return cookieHeader.toString();
  }
}
```

BridgeInterceptor的一个职责是构造出一个新的request副本，并解析旧request中的信息，作为请求头
添加进新的request中(这就是将应用程序的用户请求数据转化为对应网络请求数据的过程)；

BridgeInterceptor对cookie进行了管理，但ok默认是不接收服务端设置的cookie的，需要我们自己实现
cookieJar，完成存取逻辑；

BridgeInterceptor会对服务端返回的gzip格式的响应体进行解压，并重新赋给响应体，这样免去了应用层
解压的麻烦；

BridgeInterceptor的另一个职责是构造出一个新的response副本，解析旧response的cookie信息进行
保存，解压响应体，赋给新response的响应体(这就是将服务端响应数据转化为用户数据的过程)；


https://www.jianshu.com/p/e3b6f821acb8
https://yq.aliyun.com/articles/78104?spm=a2c4e.11153940.blogcont78105.12.563837beQ4QiO4

#### CacheInterceptor

CacheInterceptor用来决策当前网络请求是否使用缓存，并且会根据缓存策略决定是否进行缓存和删除缓
存。

##### CacheStrategy

CacheStrategy接收当前request请求和可能存在的response缓存，并根据这两者设置的头数据来决策是
使用缓存，还是进行网络请求，还是两者都进行；

1、CacheStrategy

根据response的状态码判断当前的response是否能被缓存；

```
public static boolean isCacheable(Response response, Request request) {
  // Always go to network for uncacheable response codes (RFC 7231 section 6.1),
  // This implementation doesn't support caching partial content.
  switch (response.code()) {
    // 这种状态码的response允许被缓存
    case HTTP_OK:
    case HTTP_NOT_AUTHORITATIVE:
    case HTTP_NO_CONTENT:
    case HTTP_MULT_CHOICE:
    case HTTP_MOVED_PERM:
    case HTTP_NOT_FOUND:
    case HTTP_BAD_METHOD:
    case HTTP_GONE:
    case HTTP_REQ_TOO_LONG:
    case HTTP_NOT_IMPLEMENTED:
    case StatusLine.HTTP_PERM_REDIRECT:
      // These codes can be cached unless headers forbid it.
      break;

    // 这两种状态码下如果响应头是合适的，则会进行缓存
    case HTTP_MOVED_TEMP:
    case StatusLine.HTTP_TEMP_REDIRECT:
      // These codes can only be cached with the right response headers.
      // http://tools.ietf.org/html/rfc7234#section-3

      // s-maxage这种Cache-Control(会覆盖max-age和Expires效果)标识被直接忽略，因为这种标识
      // 只能作用于共享缓存，而ok进行的是私有缓存(个人觉得这个应该理解为接收私有和共享两种缓存
      // 对于s-maxage这种只能作用于共享缓存的情况，ok选择不缓存该response，否则缓存是私有的，
      // 将不知道何时过期);
      // s-maxage is not checked because OkHttp is a private cache that should ignore s-maxage.
      if (response.header("Expires") != null
          || response.cacheControl().maxAgeSeconds() != -1
          || response.cacheControl().isPublic()
          || response.cacheControl().isPrivate()) {
        break;
      }
      // Fall-through.

    default:
      // All other codes cannot be cached.
      return false;
  }

  // Cache-Control标志是no-store，表示不缓存任何客户端请求和服务端响应
  // A 'no-store' directive on request or response prevents the response from being cached.
  return !response.cacheControl().noStore() && !request.cacheControl().noStore();
}
```

2、Factory

根据request、response和系统当前事件构造相应的CacheStrategy实例；

```
public static class Factory {
  final long nowMillis;
  final Request request;
  final Response cacheResponse;

  // 对应响应头中的Date标识，记录服务端产生响应数据的时间
  /** The server's time when the cached response was served, if known. */
  private Date servedDate;
  private String servedDateString;

  // 对应响应头中的Last-Modified标识，记录缓存在服务端最后一次的修改时间，用于对比缓存
  /** The last modified date of the cached response, if known. */
  private Date lastModified;
  private String lastModifiedString;

  // 对应响应头的expires标识，记录服务端设置的缓存过期时间，因为服务端和客户端可能存在时间差异
  // 所以存在误差，推荐max-age来记录过期时间(expires在HTTP/1.1已经被抛弃)
  /**
   * The expiration date of the cached response, if known. If both this field and the max age are
   * set, the max age is preferred.
   */
  private Date expires;

  // 记录Ok第一次发出该request时的时间，在有缓存的情况下，则不是当前系统时间，而是之前发送获取缓存的
  // request的发送时间
  /**
   * Extension header set by OkHttp specifying the timestamp when the cached HTTP request was
   * first initiated.
   */
  private long sentRequestMillis;

  // 记录Ok第一次接收到该response的时间，可能是缓存response的时间
  /**
   * Extension header set by OkHttp specifying the timestamp when the cached HTTP response was
   * first received.
   */
  private long receivedResponseMillis;

  // 对应响应头的ETag标识，用于唯一识别缓存，用于对比缓存
  /** Etag of the cached response. */
  private String etag;

  // 对应响应头的Age标识，记录响应消息在代理服务器中存储的时长
  /** Age of the cached response. */
  private int ageSeconds = -1;
```

3、构造方法

接收当前系统时间、request请求和缓存的response，解析缓存和有效期有关的响应头数据；

```
public Factory(long nowMillis, Request request, Response cacheResponse) {
  this.nowMillis = nowMillis;
  this.request = request;
  this.cacheResponse = cacheResponse;

  if (cacheResponse != null) {
    // 拿到缓存response对应的request发出时间
    this.sentRequestMillis = cacheResponse.sentRequestAtMillis();
    // 拿到从服务端接收缓存response时的时间
    this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();

    // 解析缓存的响应头
    Headers headers = cacheResponse.headers();
    for (int i = 0, size = headers.size(); i < size; i++) {
      String fieldName = headers.name(i);
      String value = headers.value(i);

      // 服务端生成响应时的时间
      if ("Date".equalsIgnoreCase(fieldName)) {
        servedDate = HttpDate.parse(value);
        servedDateString = value;

      // 缓存过期时间
      } else if ("Expires".equalsIgnoreCase(fieldName)) {
        expires = HttpDate.parse(value);

      // 服务端返回的记录缓存数据在服务端上次修改的时间
      } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {
        lastModified = HttpDate.parse(value);
        lastModifiedString = value;

      // 缓存的唯一标识
      } else if ("ETag".equalsIgnoreCase(fieldName)) {
        etag = value;

      // 缓存在代理服务器保存的时间
      } else if ("Age".equalsIgnoreCase(fieldName)) {
        ageSeconds = HttpHeaders.parseSeconds(value, -1);
      }
    }
  }
}
```

4、getCandidate

解析缓存响应头中的关于缓存使用期限的标识，构造出新的request、response或者为空的response；

```
private CacheStrategy getCandidate() {
  // No cached response.
  if (cacheResponse == null) {
    return new CacheStrategy(request, null);
  }

  // Drop the cached response if it's missing a required handshake.
  if (request.isHttps() && cacheResponse.handshake() == null) {
    return new CacheStrategy(request, null);
  }

  // 检查缓存response中的状态码，如果对应状态码下是不允许缓存的，则返回空的response
  // 这种情况正常情况下是不会出现，因为在第一次从服务端请求到数据后，在缓存之前就会进行
  // 一次检查，所以这里的操作是多余的
  // If this response shouldn't have been stored, it should never be used
  // as a response source. This check should be redundant as long as the
  // persistence store is well-behaved and the rules are constant.
  if (!isCacheable(cacheResponse, request)) {
    return new CacheStrategy(request, null);
  }

  // request设置了不需要走缓存或者这个request实际是对比缓存中发给服务端的验证请求
  CacheControl requestCaching = request.cacheControl();
  if (requestCaching.noCache() || hasConditions(request)) {
    return new CacheStrategy(request, null);
  }

  // 这个response存活的时间(当前系统时间 - 首次request请求时的时间)
  long ageMillis = cacheResponseAge();
  // 这个response的有效时长，如果设置了多种，首推max-age
  long freshMillis = computeFreshnessLifetime();

  // 获取request请求中设置的max-age值
  if (requestCaching.maxAgeSeconds() != -1) {
    freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
  }

  // 获取request请求中设置的min-fresh值
  long minFreshMillis = 0;
  if (requestCaching.minFreshSeconds() != -1) {
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
  }

  // 缓存response中没有强制缓存过期就不能用，并且request中设置了允许最大的过期时间
  long maxStaleMillis = 0;
  CacheControl responseCaching = cacheResponse.cacheControl();
  if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
  }

  // 缓存不是对比缓存，(ageMillis + minFreshMillis)为缓存实际存活时间，包含了已存活时间ageMillis
  // 和还能存活时间minFreshMillis，在缓存有效期内，(ageMillis + minFreshMillis)的值是固定的，
  // 超过有效期，此时minFreshMillis为0，ageMillis大于原来(ageMillis + minFreshMillis)，注意
  // 这里的有效期是绝对的有效期，超过绝对有效期不一定失效，还有看是否大于(freshMillis + maxStaleMillis)
  if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    Response.Builder builder = cacheResponse.newBuilder();
    // 缓存过期了，但没有超过最大的过期期限，还是能用的
    if (ageMillis + minFreshMillis >= freshMillis) {
      builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
    }

    // 当缓存设置的有效期非常长时，超过了一天，虽然能够使用缓存，但必须进行警告
    long oneDayMillis = 24 * 60 * 60 * 1000L;
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
      builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
    }
    return new CacheStrategy(null, builder.build());
  }

  // 下面是对比缓存的情况

  // 如果缓存是对比缓存，则在使用缓存之前，必须先发一个request带上缓存响应头对应的标识值于服务端
  // 进行验证，验证缓存有效，才能使用，否则必须重新请求
  // Find a condition to add to the request. If the condition is satisfied, the response body
  // will not be transmitted.
  String conditionName;
  String conditionValue;
  if (etag != null) {
    conditionName = "If-None-Match";
    conditionValue = etag;
  } else if (lastModified != null) {
    conditionName = "If-Modified-Since";
    conditionValue = lastModifiedString;
  } else if (servedDate != null) {
    conditionName = "If-Modified-Since";
    conditionValue = servedDateString;
  } else {
    return new CacheStrategy(request, null); // No condition! Make a regular request.
  }

  // 构造出对比缓存的验证request
  Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
  Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

  Request conditionalRequest = request.newBuilder()
      .headers(conditionalRequestHeaders.build())
      .build();
  return new CacheStrategy(conditionalRequest, cacheResponse);
}
```

5、get

调用getCandidate方法，获取CacheStrategy实例，但并不一定使用，还要进行多一层判断；

```
public CacheStrategy get() {
  CacheStrategy candidate = getCandidate();

  // 如果networkRequest不为空，表明必须进行一次请求网络才能获取数据了(可能是验证，不一定是请求)
  // 如果检查到request设置了强制走缓存，不能进行网络请求，则返回空的response内容和504响应码
  if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
    // We're forbidden from using the network and the cache is insufficient.
    return new CacheStrategy(null, null);
  }

  return candidate;
}
```

##### CacheInterceptor

成员变量：

```
  final InternalCache cache;
```
CacheInterceptor 只有一个成员变量cache，提供了缓存、读取、删除、更新response的功能，并且以
请求的url为key；

_ok提供了默认的cache(基于DiskLruCache)，使用时我们只需要创建对应Cache实例即可。当然，我们
还可以实习自己的缓存类，只需要实现InternalCache接口即可。_

intercept

```
@Override public Response intercept(Chain chain) throws IOException {
  Response cacheCandidate = cache != null
      ? cache.get(chain.request())
      : null;

  long now = System.currentTimeMillis();

  // 根据request请求和缓存的response的头数据决策出新的request和response
  // 如果networkRequest不为空，则表示需要进行网络请求，如果cacheResponse不会null，则是于服
  // 务端进行验证，检查本地缓存的有效性,否则cacheResponse为null，则是需要向服务端请求数据；

  // 如果networkRequest为null，则表明不进行网络请求，此时如果cacheResponse不为null，则说明
  // 有缓存可以使用；否则cacheResponse为null，表明cache存在问题，无法满足请求，报504错误
  CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
  Request networkRequest = strategy.networkRequest;
  Response cacheResponse = strategy.cacheResponse;

  // 记录通过缓存取数据的次数
  if (cache != null) {
    cache.trackResponse(strategy);
  }

  // 缓存response不可用，关闭
  if (cacheCandidate != null && cacheResponse == null) {
    closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
  }

  // 不允许进行网络请求(only-if-cached)，又没有缓存数据，则构造504错误
  // If we're forbidden from using the network and the cache is insufficient, fail.
  if (networkRequest == null && cacheResponse == null) {
    return new Response.Builder()
        .request(chain.request())
        .protocol(Protocol.HTTP_1_1)
        .code(504)
        .message("Unsatisfiable Request (only-if-cached)")
        .body(Util.EMPTY_RESPONSE)
        .sentRequestAtMillis(-1L)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
  }

  // 走缓存
  // If we don't need the network, we're done.
  if (networkRequest == null) {
    return cacheResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .build();
  }

  // 走网络请求
  Response networkResponse = null;
  try {
    networkResponse = chain.proceed(networkRequest);
  } finally {
    // If we're crashing on I/O or otherwise, don't leak the cache body.
    if (networkResponse == null && cacheCandidate != null) {
      closeQuietly(cacheCandidate.body());
    }
  }

  // 对比缓存，于服务端进行对比验证后，发现缓存没有过期，可以继续使用
  // If we have a cache response too, then we're doing a conditional get.
  if (cacheResponse != null) {
    if (networkResponse.code() == HTTP_NOT_MODIFIED) {
      // 合并缓存response和此次response
      // 更新sentRequestAtMillis和receivedResponseAtMillis应该是说明缓存刷新了
      Response response = cacheResponse.newBuilder()
          .headers(combine(cacheResponse.headers(), networkResponse.headers()))
          .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
          .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
          .cacheResponse(stripBody(cacheResponse))
          .networkResponse(stripBody(networkResponse))
          .build();
      networkResponse.body().close();

      // Update the cache after combining headers but before stripping the
      // Content-Encoding header (as performed by initContentStream()).
      cache.trackConditionalCacheHit();
      // 更新本地缓存
      cache.update(cacheResponse, response);
      return response;
    } else {
      closeQuietly(cacheResponse.body());
    }
  }

  // 普通的服务端请求，从服务端请求数据
  // stripBody的作用是剔除掉response中的body数据
  Response response = networkResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .networkResponse(stripBody(networkResponse))
      .build();

  if (cache != null) {
    // 存在响应体，并且response是可以被缓存的，则对response进行缓存
    if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
      // Offer this request to the cache.
      CacheRequest cacheRequest = cache.put(response);

      // 因为缓存response和用户读取response数据存在冲突(不能被同时持有)，所以要构造出一个包
      // 含新的data source的response，专门用来支持缓存读取；
      return cacheWritingResponse(cacheRequest, response);
    }

    // 缓存是无效的，进行移除
    if (HttpMethod.invalidatesCache(networkRequest.method())) {
      try {
        cache.remove(networkRequest);
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
    }
  }

  return response;
}
```

CacheInterceptor的主要职责是管理response缓存，正常情况下，除非在response和request中显式
指定了no-store，否则只要提供了Cache，则ok会为我们缓存存在响应体的response数据；

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Age
https://www.jianshu.com/p/b32d13655be7
https://yq.aliyun.com/articles/78104?spm=a2c4e.11153940.blogcont78105.12.563837beQ4QiO4
https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Age
https://blog.csdn.net/cominglately/article/details/77685214


#### ConnectInterceptor


> ConnectInterceptor职责很简单，就是管理当前Call请求与服务端的连接；

##### RealConnection

RealConnection负责与服务端建立连接(通过三次握手、四次挥手)；

RealConnection创建、包装和操作Socket，每一个RealConnection对应一个连接的Socket；

RealConnection代表连接Socket的链路，如果创建了RealConnection实例，则表明我们已经跟服务端建立
了一条通信链路；

RealConnection采用的连接协议有HTTP1.x、HTTPS和HTTP2三种网络协议进行连接；

当RealConnection是基于HTTP2协议是，则可以承载多个数据流(对应多个StreamAllocation)；

建立HTTPS和HTTP2的流程是先创建一个普通的rawSocket，可以用来完成HTTP1.x请求，如果当前连接支持
HTTPS或HTTP2，则会在构造SSLSocket，用来完成带安全验证的HTTP请求；

属性：

```
private final ConnectionPool connectionPool;
private final Route route;

// The fields below are initialized by connect() and never reassigned.
// 下面的变量在调用connect方法进行连接时被初始化
/** The low-level TCP socket. */
// rawSocket可以认为是普通的Socket，基于HTTP协议
private Socket rawSocket;

/**
 * The application layer socket. Either an {@link SSLSocket} layered over {@link #rawSocket}, or
 * {@link #rawSocket} itself if this connection does not use SSL.
 */
// 应用层的Socket，如果没有使用SSL协议，则会使用SSLSocket或者直接是rawSocket
// 否则将是基于HTTPS安全协议的SSLSocket，应用与HTTPS和HTTP2协议
private Socket socket;
// 基于HTTPS，描述握手协议的
private Handshake handshake;
// 描述协议类型(HTTP1.0/HTTP1.1/HTTP2等)
private Protocol protocol;
// Http2使用的连接
private Http2Connection http2Connection;

与服务端数据交互的输入输出流
private BufferedSource source;
private BufferedSink sink;

// The fields below track connection state and are guarded by connectionPool.
// 下面的变量是用来跟踪连接状态的，并且由连接池进行管理

/** If true, no new streams can be created on this connection. Once true this is always true. */
// 如果为true，表示当前连接池不能再创建新的stream流
public boolean noNewStreams;

// 记录连接成功的次数
public int successCount;

/**
 * The maximum number of concurrent streams that can be carried by this connection. If {@code
 * allocations.size() < allocationLimit} then new streams can be created on this connection.
 */
// 当前连接可以承载的Stream流个数，如果个数小于这里声明的值，则该连接可以创建新的Stream流
// HTTP1.x只能够承载1个，而HTTP2能够承载多个
public int allocationLimit = 1;

// 当前连接承载的Stream流集合
/** Current streams carried by this connection. */
public final List<Reference<StreamAllocation>> allocations = new ArrayList<>();

/** Nanotime timestamp when {@code allocations.size()} reached zero. */
public long idleAtNanos = Long.MAX_VALUE;
```
###### connectSocket

创建一个最基本的Socket，支持HTTP连接，并且可以扩展为HTTPS连接

```
/** Does all the work necessary to build a full HTTP or HTTPS connection on a raw socket. */
private void connectSocket(int connectTimeout, int readTimeout) throws IOException {
  Proxy proxy = route.proxy();
  Address address = route.address();

  // 创建直连或者有代理的Socket
  rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
      ? address.socketFactory().createSocket()
      : new Socket(proxy);

  rawSocket.setSoTimeout(readTimeout);
  try {
    // 连接Socket
    Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
  } catch (ConnectException e) {
    ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
    ce.initCause(e);
    throw ce;
  }

  // 创建Socket的输入/输出流
  source = Okio.buffer(Okio.source(rawSocket));
  sink = Okio.buffer(Okio.sink(rawSocket));
}
```

###### createTunnel

创建一个连接管道；

```
private Request createTunnel(int readTimeout, int writeTimeout, Request tunnelRequest,
    HttpUrl url) throws IOException {
  // Make an SSL Tunnel on the first message pair of each SSL + proxy connection.
  String requestLine = "CONNECT " + Util.hostHeader(url, true) + " HTTP/1.1";
  while (true) {
    // 基于HTTP1.1的Socket数据通讯封装类
    Http1Codec tunnelConnection = new Http1Codec(null, null, source, sink);
    source.timeout().timeout(readTimeout, MILLISECONDS);
    sink.timeout().timeout(writeTimeout, MILLISECONDS);
    // 发送建立管道连接请求
    tunnelConnection.writeRequest(tunnelRequest.headers(), requestLine);
    tunnelConnection.finishRequest();
    Response response = tunnelConnection.readResponseHeaders(false)
        .request(tunnelRequest)
        .build();
    // The response body from a CONNECT should be empty, but if it is not then we should consume
    // it before proceeding.

    // 检查管道是否建立
    long contentLength = HttpHeaders.contentLength(response);
    if (contentLength == -1L) {
      contentLength = 0L;
    }
    Source body = tunnelConnection.newFixedLengthSource(contentLength);
    Util.skipAll(body, Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
    body.close();

    switch (response.code()) {
      case HTTP_OK:
        // Assume the server won't send a TLS ServerHello until we send a TLS ClientHello. If
        // that happens, then we will have buffered bytes that are needed by the SSLSocket!
        // This check is imperfect: it doesn't tell us whether a handshake will succeed, just
        // that it will almost certainly fail because the proxy has sent unexpected data.
        if (!source.buffer().exhausted() || !sink.buffer().exhausted()) {
          throw new IOException("TLS tunnel buffered too many bytes!");
        }
        return null;

      case HTTP_PROXY_AUTH:
        tunnelRequest = route.address().proxyAuthenticator().authenticate(route, response);
        if (tunnelRequest == null) throw new IOException("Failed to authenticate with proxy");

        if ("close".equalsIgnoreCase(response.header("Connection"))) {
          return tunnelRequest;
        }
        break;

      default:
        throw new IOException(
            "Unexpected response code for CONNECT: " + response.code());
    }
  }
}
```

###### connectTunnel

以管道方式建立连接(就是建立一个TCP长连接，将所有的请求都交给这个连接处理，这些请求被按顺序发送和
接收，当一个请求耗时过长时，后续请求将会受到阻塞)；

```
private void connectTunnel(int connectTimeout, int readTimeout, int writeTimeout)
    throws IOException {
  // 构造一个管道请求
  Request tunnelRequest = createTunnelRequest();
  HttpUrl url = tunnelRequest.url();
  int attemptedConnections = 0;
  int maxAttempts = 21;
  while (true) {
    if (++attemptedConnections > maxAttempts) {
      throw new ProtocolException("Too many tunnel connections attempted: " + maxAttempts);
    }
    // 创建Socket
    connectSocket(connectTimeout, readTimeout);
    // 创建一个管道连接
    tunnelRequest = createTunnel(readTimeout, writeTimeout, tunnelRequest, url);

    if (tunnelRequest == null) break; // Tunnel successfully created.

    // The proxy decided to close the connection after an auth challenge. We need to create a new
    // connection, but this time with the auth credentials.
    closeQuietly(rawSocket);
    rawSocket = null;
    sink = null;
    source = null;
  }
}
```

###### establishProtocol

如果是基于HTTPS或者HTTP2协议的连接，则必须经过下面的方法。

```
private void establishProtocol(ConnectionSpecSelector connectionSpecSelector) throws IOException {
  // 无法创建SSLSocket(基于SSL和TLS协议的Socket，提供HTTPS连接支持)
  if (route.address().sslSocketFactory() == null) {
    protocol = Protocol.HTTP_1_1;
    socket = rawSocket;
    return;
  }

  // 创建SSLSocket
  // 如果HTTPS支持TLS协议，则采用TLS协议，否则是SSL协议
  connectTls(connectionSpecSelector);

  // 采用HTTP2协议进行连接
  // HTTP2协议默认基于TLS和SSL协议安全加密的
  if (protocol == Protocol.HTTP_2) {
    socket.setSoTimeout(0); // HTTP/2 connection timeouts are set per-stream.
    http2Connection = new Http2Connection.Builder(true)
        .socket(socket, route.address().url().host(), source, sink)
        .listener(this)
        .build();
    http2Connection.start();
  }
}
```

###### connect

```
public void connect(
    int connectTimeout, int readTimeout, int writeTimeout, boolean connectionRetryEnabled) {
  if (protocol != null) throw new IllegalStateException("already connected");

  RouteException routeException = null;
  // ConnectionSpec是描述Socket连接的协议信息
  List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
  // ConnectionSpecSelector用来选择合适的ConnectionSpec，当握手或者协议出现问题的时候，需
  // 要尝试不同的协议进行连接
  ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);

  // Https安全连接需要的证书验证
  if (route.address().sslSocketFactory() == null) {
    if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
      throw new RouteException(new UnknownServiceException(
          "CLEARTEXT communication not enabled for client"));
    }
    String host = route.address().url().host();
    if (!Platform.get().isCleartextTrafficPermitted(host)) {
      throw new RouteException(new UnknownServiceException(
          "CLEARTEXT communication to " + host + " not permitted by network security policy"));
    }
  }

  while (true) {
    try {
      // 基于持久化的管道连接，使用较少
      if (route.requiresTunnel()) {
        connectTunnel(connectTimeout, readTimeout, writeTimeout);
      } else {
        // 一般情况都是这种，创建并连接Socket
        // 此时rawSocket被创建
        connectSocket(connectTimeout, readTimeout);
      }
      // 构建安全传输协议，如果支持SSL或者TLS，则在rawSocket基础上进行包装为SSLSocket
      establishProtocol(connectionSpecSelector);
      break;
    } catch (IOException e) {
      closeQuietly(socket);
      closeQuietly(rawSocket);
      socket = null;
      rawSocket = null;
      source = null;
      sink = null;
      handshake = null;
      protocol = null;
      http2Connection = null;

      if (routeException == null) {
        routeException = new RouteException(e);
      } else {
        routeException.addConnectException(e);
      }

      if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
        throw routeException;
      }
    }
  }

  if (http2Connection != null) {
    synchronized (connectionPool) {
      allocationLimit = http2Connection.maxConcurrentStreams();
    }
  }
}

```


##### StreamAllocation

StreamAllocation的作用是协调连接(Connections)、数据流(Streams)和请求(Calls)之间的关系；

连接指的是Socket与远程服务端的连接。创建连接的过程可能很缓慢，所以我们可以直接取消掉一个
正在创建的连接。

数据流表示建立在连接之上的逻辑请求和响应数据。每一个连接能够携带的数据流个数都是有限制的，HTTP1.x
协议，每一个连接都只能够携带一个数据流，而HTTP2采用了连接复用技术，所以同一连接可以携带多个数据流。

请求，即Call请求，通过request发起。一个请求对应了一些列的数据流，典型的是一个初始化请求和其后的
一系列重定向请求，每一个请求对应一个数据流。我们更希望一个请求的所有数据流都使用一个相同的连接，
这样比在不同的连接上将有更好的行为和位置优势(同一个位置更便于管理和操作)。


> HTTP2相对于HTTP1.x解决了连接无法复用的问题。所谓的连接复用，就是多个stream流都采用同一Socket
进行连接，这些Stream都有一个相同的特征，就是它们的host和port必须是相同的，每一个stream代表一次
request请求。通过复用同一个Socket连接，能够显著的减少Socket建立TCP连接时，三次握手的时间。


StreamAllocation的实例代表这一个使用一个或多个基于一个或多个连接的数据流(说白了就是一个请求的
数据流管理类)，这个类提供了基于这些连接资源的释放API：

**noNewStreams**

noNewStreams方法能够防止一个连接被再次创建一个新的数据流，这个方法一般用在连接的close方法被调
用，或者连接与需求不符的情况下。

**streamFinished**

streamFinished用来释放当前allocation中保存的活跃的数据流。

> 在同一时刻，只有一个连接可能是处于活跃状态，所以在创建其他的数据流之前，必须确保已经调用了本方
法将活跃的连接关闭掉。

**release**

release方法的作用是移除当前连接上的请求(多个Stream)；

> 如果当前连接上任然存在一个活跃的数据流，则调用这个方法不会立马释放连接。比如，当一个请求已经
结束了，但是，我们还没有使用完response的响应体数据(通过response读取网络文件输出流)。

上面三种管理连接的方式，在StreamAllocation中对应了三个方法，这三个方法都调用了：
```
  private Socket deallocate(boolean noNewStreams, boolean released, boolean streamFinished) {
      ...
  }
```
来进行连接的管理。无论是通过哪种方式，都无法单独做到将连接置为空闲的效果。只有在streamFinished
为true，即数据流是关闭或者为创建的状态，才能将当前的连接置为空闲状态(Socket也会同时关闭)。


**cancel**

cancel 方法用来取消掉当前的数据流和连接，支持异步的调用。如果数据流是基于HTTP2协议的，则在
调用该方法时，只会关闭该数据流，连接和共享该连接的其他数据流不会受到影响。

如果连接正处于TLS握手阶段(即还没正式建立起来)，调用该方法可能会打断整个连接。


###### findConnection

该方法将返回一个持有新的数据流的连接，如果已经存在现成可用的连接，则会直接使用；否则会先创建
连接，放入连接池；

```
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
    boolean connectionRetryEnabled) throws IOException {
  Route selectedRoute;
  synchronized (connectionPool) {
    if (released) throw new IllegalStateException("released");
    if (codec != null) throw new IllegalStateException("codec != null");
    if (canceled) throw new IOException("Canceled");

    // 检查是否已经有现成的连接可用
    // 注意这个连接必须是可以创建新的数据流的，即noNewStreams为false
    // Attempt to use an already-allocated connection.
    RealConnection allocatedConnection = this.connection;
    if (allocatedConnection != null && !allocatedConnection.noNewStreams) {
      return allocatedConnection;
    }

    // 根据address和Route从连接池中查找是否有现成可用的连接
    // Attempt to get a connection from the pool.
    Internal.instance.get(connectionPool, address, this, null);
    // 如果找到连接，则会调用StreamAllocation的acquire方法，赋给connection变量
    if (connection != null) {
      return connection;
    }

    // 更换路由
    selectedRoute = route;
  }

  // 确保更新了路由
  // If we need a route, make one. This is a blocking operation.
  if (selectedRoute == null) {
    selectedRoute = routeSelector.next();
  }

  RealConnection result;
  synchronized (connectionPool) {
    if (canceled) throw new IOException("Canceled");

    // 根据新的路由，从连接池中查找是否有新的连接
    // Now that we have an IP address, make another attempt at getting a connection from the pool.
    // This could match due to connection coalescing.
    Internal.instance.get(connectionPool, address, this, selectedRoute);
    if (connection != null) return connection;

    // 根据路由，创建一个新的连接，并立刻将其赋给StreamAllocation，这样能够确保我们在异步调用
    // cancel方法时能够直接打断连接进行握手。
    // Create a connection and assign it to this allocation immediately. This makes it possible
    // for an asynchronous cancel() to interrupt the handshake we're about to do.
    route = selectedRoute;
    refusedStreamCount = 0;
    result = new RealConnection(connectionPool, selectedRoute);
    acquire(result);
  }

  // 进行TCP连接和TLS握手
  // Do TCP + TLS handshakes. This is a blocking operation.
  result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);

  // 将已建立的连接缓存
  routeDatabase().connected(result.route());

  Socket socket = null;
  synchronized (connectionPool) {
    // 将连接放到连接池中
    // 该方法最终将调用ConnectionPool的put方法，而put方法在添加连接的同时，会执行其中的
    // cleanupRunnable，清除连接池中保存时间最长的空闲连接
    // Pool the connection.
    Internal.instance.put(connectionPool, result);

    // 在异步的情况下，如果连接是基于HTTP2协议的，并且有其他的请求已经建立了相同的连接
    // 则复用那个连接，并把当前建立的连接释放掉
    // If another multiplexed connection to the same address was created concurrently, then
    // release this connection and acquire that one.
    // 检查是否是基于HTTP2的连接
    if (result.isMultiplexed()) {
      // 检查是否已经建立相同的连接，相同则把旧连接绑定到当前StreamAllocation，并
      // 释放当前连接，把当前连接的Socket返回
      // 该方法最终调用的是ConnectionPool的deduplicate，而在该方法中，当匹配到相同的
      // 连接时，将回调StreamAllocation的releaseAndAcquire方法进行释放和设置
      socket = Internal.instance.deduplicate(connectionPool, address, this);
      result = connection;
    }
  }
  // 关闭Socket
  closeQuietly(socket);

  return result;
}
```

###### releaseAndAcquire

释放当前Allocation持有的连接，并用传递进来的连接进行代替。在同时有多个HTTP2异步请求发出时，
如果发现Allocation新建的连接已经存在了，可以调用该方法，实现线程安全的去重操作；

```
public Socket releaseAndAcquire(RealConnection newConnection) {
  assert (Thread.holdsLock(connectionPool));
  // StreamAllocation刚创建连接时，该连接中包含的StreamAllocation个数只能是1(即当前实例)
  // 多于1，表明当前StreamAllocation持有的连接不是新建的，已经被其他StreamAllocation持有了
  if (codec != null || connection.allocations.size() != 1) throw new IllegalStateException();

  // 新连接中只有一个StreamAllocation，所以只需取第一个
  // Release the old connection.
  Reference<StreamAllocation> onlyAllocation = connection.allocations.get(0);

  // 释放当前StreamAllocation持有的连接
  Socket socket = deallocate(true, false, false);

  // 将新连接(其实是旧连接，其他StreamAllocation已经持有了)赋给当前StreamAllocation
  // Acquire the new connection.
  this.connection = newConnection;
  // 将当前StreamAllocation交给连接管理(这也是为什么前面的判断多于1会抛异常，旧连接会持有多个
  // StreamAllocation)
  newConnection.allocations.add(onlyAllocation);

  return socket;
}
```

###### deallocate

deallocate方法用来把当前StreamAllocation持有的连接释放掉。

```
private Socket deallocate(boolean noNewStreams, boolean released, boolean streamFinished) {
  assert (Thread.holdsLock(connectionPool));

  // 结束数据流传输
  if (streamFinished) {
    this.codec = null;
  }
  if (released) {
    this.released = true;
  }
  Socket socket = null;
  if (connection != null) {
    // 标记连接不能再创建新的数据流
    if (noNewStreams) {
      connection.noNewStreams = true;
    }
    // 如果当前StreamAllocation不存在活跃的数据流，并且可以释放或者不能再创建新数据流
    if (this.codec == null && (this.released || connection.noNewStreams)) {
      // 将当前StreamAllocation从连接中移除
      release(connection);

      // 如果当前StreamAllocation持有的连接没有被其他的StreamAllocation持有
      // 则尝试释放掉该连接
      if (connection.allocations.isEmpty()) {
        connection.idleAtNanos = System.nanoTime();
        // 将连接置为空闲状态
        // 连接池默认允许5个空闲的连接存在
        // 如果自定义连接池，空闲连接个数设置为0，则连接置为空闲后，会立刻被从连接池中移除，
        // 如果连接被移除，则移除的连接的Socket会被返回，进行关闭操作

        // 默认情况下，当我们在findConnection中添加新的连接进入连接池时，就会推动连接池清除
        // 空闲的连接
        if (Internal.instance.connectionBecameIdle(connectionPool, connection)) {
          socket = connection.socket();
        }
      }
      connection = null;
    }
  }
  return socket;
}
```

###### findHealthyConnection

获取一个可以使用的连接；

```
private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
    int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
    throws IOException {
  // 循环获取一个有效的连接
  while (true) {
    // 获取一个连接，可能是从缓存池中获取，也可能是新建的
    RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
        connectionRetryEnabled);

    // 如果是新建的连接，则直接跳过连接的健康检查
    // If this is a brand new connection, we can skip the extensive health checks.
    synchronized (connectionPool) {
      if (candidate.successCount == 0) {
        return candidate;
      }
    }

    // 如果是旧连接，检查连接是否还有效，无效则将连接的noNewStreams置为false(不能再创建新连接)
    // 再次循环获取连接
    // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
    // isn't, take it out of the pool and start again.
    if (!candidate.isHealthy(doExtensiveHealthChecks)) {
      noNewStreams();
      continue;
    }

    return candidate;
  }
}
```

###### noNewStreams

这个方法的主要作用是设置当前的StreamAllocation的连接不能再创建新的数据流；

```
public void noNewStreams() {
  Socket socket;
  synchronized (connectionPool) {
    // 这里只有参数1 noNewStreams为true，大多数情况下只是起到让连接不能再创建数据流的作用
    // 当StreamAllocation没有数据流或者处于释放状态下，则会关闭连接
    socket = deallocate(true, false, false);
  }
  closeQuietly(socket);
}
```

###### newStream

创建一个新的数据流；

```
public HttpCodec newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {
  int connectTimeout = client.connectTimeoutMillis();
  int readTimeout = client.readTimeoutMillis();
  int writeTimeout = client.writeTimeoutMillis();
  boolean connectionRetryEnabled = client.retryOnConnectionFailure();

  try {
    // 获取一个健康可用的连接(可能是新建的，也可能是从连接池中获取的)
    RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
        writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);

    // 在连接上创建一个新的数据流，可能是基于HTTP1.x协议的Http1Codec
    // 也可能是基于HTTP2协议的Http2Codec
    // 如果是基于HTTP1.x协议的，必须确保连接是新的连接，只有新连接的noNewStream才为false
    // 而HTTP2.x则可以进行连接的复用，所以可以是旧连接，但同样noNewStream必须为false
    HttpCodec resultCodec = resultConnection.newCodec(client, this);

    synchronized (connectionPool) {
      codec = resultCodec;
      return resultCodec;
    }
  } catch (IOException e) {
    throw new RouteException(e);
  }
}
```


##### ConnectInterceptor.intercept

至此，ConnectInterceptor的intercept方法的流程也就很明确了！

```
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;

  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    // 获取从RetryAndFollowUpInterceptor中就已经创建的StreamAllocation
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // 对于非Get的请求，需要对连接进行健康检查，检查是否可用
    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```

拿到在RetryAndFollowUpInterceptor中创建的StreamAllocation实例(在这里才算是第一使用了
StreamAllocation)，通过StreamAllocation创建新的数据流，获取可用的连接(连接也可能是在创建
新数据流的过程中创建的，也可能是从连接池中拿的)，传递给RealInterceptorChain的proceed方法，
推动拦截链进入系统拦截器的最后一个环节——CallServerInterceptor。

![image](/images/connectinterceptor转换关系.png)

> 每一个请求都对应这一个StreamAllocation、一个RealConnection、一个HttpCodec；<br>


![image](/images/StreamAllocation与Connection和httpcodec关系.png)

> 在StreamAllocation中，在创建新的数据流时，会从连接池中查找是否有可复用的连接，没有才会创建
新的连接，并把新的连接放入到连接池中，在放入连接池的同时，连接池会把当前空闲最久的连接的给移除掉；

> 在StreamAllocation中，根据Address和Route信息，可能从缓存池中获取出一个连接RealConnection，
也可能是新建一个RealConnection(创建后会把RealConnection放入到ConnectionPool中)，最终的目的
是将Request转化为一个对应数据流(HttpCodec)，用来与服务端进行数据通信.


![image](/images/StreamAllocation与RealConnection.png)

> 对于基于HTTP1.x协议的连接，一个连接只能同时被一个请求使用，也就是说一个RealConnection中
同时只会存在一个SteamAllocation，而一个SteamAllocation对应一个数据流，当同时有多个请求对
同一个地址路由发起请求时，需要建立多个连接，所以Http1.x效率是低下的。(这并不以为着每次请求都
需要进行三次握手，因为有连接池的存在，对于同一个地址路由的请求，能够复用连接，但这种方式，拿到
缓存的连接可能会失效！)

> HTTP1.x协议中有一种建立管道通信方式的长连接，这种方式虽然也能够省去建立连接时的握手时间，并且
不像缓存一样，连接会失效，但这种方式会一直占用着连接，对于通信不频繁的情况，将会浪费资源。

> HTTP2协议的连接，支持连接复用，也就是说，一个连接可以被多个请求同时使用，对应的就是一个
RealConnection可以对应(保存)多个StreamAllocation。一个连接中将会同时有多个StreamAllocation
在进行数据通信(实际上，虽然实现了连接复用，但多个数据流并不是在通道中并发传输的，而是需要排队
顺序通过连接通道，到达服务端后，根据序号再各自组成完整数据请求)；
