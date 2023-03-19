---
title: OkHttp原理探析
date: 2019-10-18 14:34:28
tags: 
  - android
categories:
  - 技术
description: 本文是对 OkHttp 原理的简要分析（以异步为例）。
---

`OkHttp`是最近几年在安卓开发中运用比较广泛的开源网络框架，支持同步和异步请求，本文主要分析平常开发中运用得比较多的异步请求流程。首先来看下开启一个异步请求的流程
```java
OkHttpClient okHttpClient = new OkHttpClient();
```
第一步创建`OkHttpClient`实例
```java
Request request = new Request.Builder().url("https://lucien623.github.io/").build();
```
第二步创建`Request`实例
```java
okHttpClient.newCall(request).enqueue(new okhttp3.Callback() {
    @Override
    public void onFailure(okhttp3.Call call, IOException e) {

    }

    @Override
    public void onResponse(okhttp3.Call call, okhttp3.Response response) throws IOException {

    }
});
```
第三步调用`okHttpClient`的`newCall`方法，并把`request`传入，再接着调用`enqueue`方法即开始异步请求了。
先暂时不分析`OkHttpClient`这个类，来看看`Request`的创建过程
```java
public static class Builder {
    HttpUrl url;
    String method;
    Headers.Builder headers;
    RequestBody body;
    Object tag;

    //...此处省略部分代码
    public Builder url(String url) {
      if (url == null) throw new NullPointerException("url == null");

      if (url.regionMatches(true, 0, "ws:", 0, 3)) {
        url = "http:" + url.substring(3);
      } else if (url.regionMatches(true, 0, "wss:", 0, 4)) {
        url = "https:" + url.substring(4);
      }

      HttpUrl parsed = HttpUrl.parse(url);
      if (parsed == null) throw new IllegalArgumentException("unexpected url: " + url);
      return url(parsed);
    }

   //...此处省略部分代码

    public Request build() {
      if (url == null) throw new IllegalStateException("url == null");
      return new Request(this);
    }
  }
```
就是通过建造者模式构建一个实例，对我们传入的`URL`地址进行了一个判断，如果是`web socket`形式的地址，会被转换成`http`或`https`形式的地址，
接着我们来看看`newCall(request)`过程
```java
@Override public Call newCall(Request request) {
  return new RealCall(this, request, false /* for web socket */);
}
```
可以看到构建返回了一个`RealCall`实例，`newCall`方法其实是定义在`Call`接口中的工厂方法接口，接着看
```java
RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
  final EventListener.Factory eventListenerFactory = client.eventListenerFactory();

  this.client = client;
  this.originalRequest = originalRequest;
  this.forWebSocket = forWebSocket;
  this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);

  // TODO(jwilson): this is unsafe publication and not threadsafe.
  this.eventListener = eventListenerFactory.create(this);
}
```
`RealCall`实例包含了之前创建的`OkHttpClient`和`Request`实例，并创建了一个`RetryAndFollowUpInterceptor`拦截器，这个我们之后可以分析到，`RealCall`实现了`Call`接口中的方法，包括接下来要调用`RealCall`中的`enqueue`方法
```java
@Override public void enqueue(Callback responseCallback) {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```
先会判断当前的`RealCall`对象是否已经执行过同步或者异步方法，如果执行过的会就会抛出"Already Executed"的异常，这一步可以得出每一个创建的`Call`都只能够执行一次请求，接着我们可以看到调用`client.dispatcher()`执行了`enqueue`方法，就是通过`OkHttpClient`实例获取到了一个`Dispatcher`实例，然后再调用这个方法，我们暂时不管`Dispatcher`，先看`new AsyncCall(responseCallback)`是个啥，`responseCallback`是我们在调用异步请求方法的时候传递的一个回调，接着看`AsyncCall`
```java
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
      Response response = getResponseWithInterceptorChain();
      if (retryAndFollowUpInterceptor.isCanceled()) {
        signalledCallback = true;
        responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
      } else {
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      }
    } catch (IOException e) {
      if (signalledCallback) {
        // Do not signal the callback twice!
        Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
      } else {
        responseCallback.onFailure(RealCall.this, e);
      }
    } finally {
      client.dispatcher().finished(this);
    }
  }
}
```
`NamedRunnable`其实就是个实现了`Runnable`接口的抽象类，在`NamedRunnable`重写的`run()`方法中调用了`AsyncCall`中的`execute();`方法
```java
@Override public final void run() {
  String oldName = Thread.currentThread().getName();
  Thread.currentThread().setName(name);
  try {
    execute();
  } finally {
    Thread.currentThread().setName(oldName);
  }
}

protected abstract void execute();
```
这个时候我们就大致可以知道`Dispatcher`里面肯定有个`Thread`，或者线程池了，其实他就是个任务调度器，英文的字面意思不就是这样么= =，我们接着看它调用`enqueue`方法是怎么执行的
```java
synchronized void enqueue(AsyncCall call) {
  if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
    runningAsyncCalls.add(call);
    executorService().execute(call);
  } else {
    readyAsyncCalls.add(call);
  }
}
```
`Dispatcher`实例是在创建`OkHttpClient`时 new 的，初始化时创建了三个队列
```java
/** Ready async calls in the order they'll be run. */
private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

/** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

/** Running synchronous calls. Includes canceled calls that haven't finished yet. */
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```
`readyAsyncCalls`表示等待执行异步请求队列，`runningAsyncCalls`是正在执行异步请求中的队列，`runningSyncCalls`是正在执行同步请求的队列，再回到上面先判断当前正在执行异步请求的数量有没有小于最大的异步请求数`maxRequests`即 64 个，然后判断当前`host`的异步请求数是否超过了`maxRequestsPerHost`即 5 个，如果不满足其中一个条件就把`call`放入待执行异步任务队列中，如果同时满足的话，就把当前的`call`放入正在执行异步请求队列中，然后执行`executorService().execute(call);`
```java
public synchronized ExecutorService executorService() {
  if (executorService == null) {
    executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
  }
  return executorService;
}
```
终于线程池的真面目露出来了，这里创建了一个线程池大小为`Integer.MAX_VALUE`即 2^31 - 1，如果有线程闲置 60 秒即被回收，内部没有任何容量的阻塞队列的线程池，`AsyncCall`交由线程池之后会开启子线程执行异步请求，然后会执行`execute()`方法，接着来看`Response response = getResponseWithInterceptorChain();`
```java
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors());
  interceptors.add(retryAndFollowUpInterceptor);
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  interceptors.add(new CacheInterceptor(client.internalCache()));
  interceptors.add(new ConnectInterceptor(client));
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(forWebSocket));

  Interceptor.Chain chain = new RealInterceptorChain(
      interceptors, null, null, null, 0, originalRequest);
  return chain.proceed(originalRequest);
}
```
我们可以看到这里创建了一堆拦截器放到了集合里，然后和`originalRequest`实例一起构建了`RealInterceptorChain`，最后将执行`chain.proceed(originalRequest)`的结果返回
```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
    RealConnection connection) throws IOException {
  if (index >= interceptors.size()) throw new AssertionError();

  calls++;

   //...此处省略部分代码

  // Call the next interceptor in the chain.
  RealInterceptorChain next = new RealInterceptorChain(
      interceptors, streamAllocation, httpCodec, connection, index + 1, request);
  Interceptor interceptor = interceptors.get(index);
  Response response = interceptor.intercept(next);

   //...此处省略部分代码

  return response;
}
```
`proceed`方法中又构建了一个`RealInterceptorChain`实例，`index`值加 1，接着获取对应 index 位置的`Interceptor`实例，前面我们可以看到第一个拦截器是`retryAndFollowUpInterceptor`
```java
@Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();

    streamAllocation = new StreamAllocation(
        client.connectionPool(), createAddress(request.url()), callStackTrace);

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response = null;
      boolean releaseConnection = true;
      try {
        response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } 
      //...此处省略部分代码
      finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Request followUp = followUpRequest(response);

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      closeQuietly(response.body());
	  //...此处省略部分代码

      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(
            client.connectionPool(), createAddress(followUp.url()), callStackTrace);
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }
```
这个拦截器功能是根据`response`来做相应的处理，`followUpRequest(response)`这个方法里会根据`responseCode`判断是否需要重新构建`Request`，默认请求成功的话会返回`null`，如果发生类似请求重定向之类的，那么便会重新构建`Request`实例返回，可以看到这里面是一个`while (true)`循环，那么用重新构建的`request`再走循环里的逻辑，包括`response = ((RealInterceptorChain) chain).proceed(request, streamAllocation, null, null);`也就是说会再次进行网络请求获取`response`。那么继续来看`proceed`，此时的`chain`是之前创建的`index = 1`的`RealInterceptorChain`实例，此时在`proceed`方法中获取的到的`Interceptor`是`BridgeInterceptor`实例，这个拦截器的作用就是把我们构造的请求转换成发送至服务器的请求以及将服务端返回的响应转换成用户友好的响应，这个拦截器的代码就不详细分析了，拦截器这里其实就是运用了责任链模式，每一个拦截器都可以对`request`和`response`进行处理，这样的设计是`OkHttp`比较精妙的一个地方。ok，那么接下来我们再看看缓存拦截器`CacheInterceptor`里的`intercept(Chain chain)`方法
```java
Response cacheCandidate = cache != null
    ? cache.get(chain.request())
    : null;
```
首先判断`cache`实例是否为`null`，如果我们创建`OkHttpClient`实例时传入了自己的缓存实例的话，会调用`get`方法，缓存其实是用`DiskLruCache`形式保存的，key 是以请求`url`的 md5 值的形式。
```java
long now = System.currentTimeMillis();

CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
Request networkRequest = strategy.networkRequest;
Response cacheResponse = strategy.cacheResponse;
```
```java
public CacheStrategy get() {
  CacheStrategy candidate = getCandidate();

  if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
    // We're forbidden from using the network and the cache is insufficient.
    return new CacheStrategy(null, null);
  }

  return candidate;
}
```
这一步是根据请求和缓存`Response`获取缓存策略，继续看`getCandidate()`方法
```java
private CacheStrategy getCandidate() {
  // No cached response.
  // 没有缓存
  if (cacheResponse == null) {
    return new CacheStrategy(request, null);
  }

  // Drop the cached response if it's missing a required handshake.
  // 丢失握手信息
  if (request.isHttps() && cacheResponse.handshake() == null) {
    return new CacheStrategy(request, null);
  }

  // 根据返回状态码和 Cache-control 策略判断是否可以缓存
  // 这里的状态码包括 200（请求成功）404 （服务器找不到请求的网页）等等
  // Cache-control 判断了 cacheResponse 和 request 中的缓存控制策略是否为no-store
  if (!isCacheable(cacheResponse, request)) {
    return new CacheStrategy(request, null);
  }

  //根据请求头判断是否是不需要 cache
  CacheControl requestCaching = request.cacheControl();
  if (requestCaching.noCache() || hasConditions(request)) {
    return new CacheStrategy(request, null);
  }

  long ageMillis = cacheResponseAge();
  long freshMillis = computeFreshnessLifetime();

  if (requestCaching.maxAgeSeconds() != -1) {
    freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
  }

  long minFreshMillis = 0;
  if (requestCaching.minFreshSeconds() != -1) {
    minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
  }

  long maxStaleMillis = 0;
  CacheControl responseCaching = cacheResponse.cacheControl();
  if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
    maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
  }

  if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
    Response.Builder builder = cacheResponse.newBuilder();
    if (ageMillis + minFreshMillis >= freshMillis) {
      builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
    }
    long oneDayMillis = 24 * 60 * 60 * 1000L;
    if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
      builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
    }
    //这里表示如果缓存没有过期，那么根据 cacheResponse 创建一个 Response
    return new CacheStrategy(null, builder.build());
  }

  //缓存过期
  String conditionName;
  String conditionValue;
  //如果 etag 不为空，则向网络请求带 If-None-Match
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

  Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
  Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

  Request conditionalRequest = request.newBuilder()
      .headers(conditionalRequestHeaders.build())
      .build();
  return new CacheStrategy(conditionalRequest, cacheResponse);
}
```
接下来就是根据对`CacheStrategy`做相应逻辑了
```java
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
```
这里表示如果没有`networkRequest`并且没有缓存的话，就自己构造一个`Response` 504 返回
```java
if (networkRequest == null) {
  return cacheResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .build();
}
```
这一步表示如果不进行网络请求那么返回缓存的`cacheResponse`，接下来如果需要网络就会执行`networkResponse = chain.proceed(networkRequest);`了
```java
if (cacheResponse != null) {
  if (networkResponse.code() == HTTP_NOT_MODIFIED) {
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
    cache.update(cacheResponse, response);
    return response;
  } else {
    closeQuietly(cacheResponse.body());
  }
}
```
拿到网络请求返回的`networkResponse`后，判断是不是返回 304，如果是的话就把`cacheResponse`的请求头和`networkResponse`的请求头合并，然后更新`Response`缓存，接下来
```java
Response response = networkResponse.newBuilder()
    .cacheResponse(stripBody(cacheResponse))
    .networkResponse(stripBody(networkResponse))
    .build();

if (cache != null) {
  if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
    // Offer this request to the cache.
    CacheRequest cacheRequest = cache.put(response);
    return cacheWritingResponse(cacheRequest, response);
  }

  if (HttpMethod.invalidatesCache(networkRequest.method())) {
    try {
      cache.remove(networkRequest);
    } catch (IOException ignored) {
      // The cache cannot be written.
    }
  }
}

return response;
```
接下来根据`response`里的返回码、method 和内容长度判断是否有需要缓存的 body 内容以及根据`response`的返回码缓存策略和`networkRequest`中的缓存策略判断是否需要缓存，如果 body 可以缓存以及策略需要缓存的话，那么会将`response`存放到`cache`中。接下来一步是根据网络请求的方式来判断是否需要缓存啦，像类似于"DELETE"、"PUT"之类的请求就没必要添加到缓存里。还有一个`CallServerInterceptor`就不多讲了，这个拦截器的功能就是向服务器发送请求数据以及获取响应数据。经过这一连串的拦截器处理最后可以获取到`Response`然后回调返回了～