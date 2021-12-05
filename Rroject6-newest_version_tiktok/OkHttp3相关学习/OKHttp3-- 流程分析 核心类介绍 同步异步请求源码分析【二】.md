[toc]

# **OKHttp3-- 流程分析 核心类介绍 同步异步请求源码分析【二】.md**

## 系列

* [OKHttp3–详细使用及源码分析系列之初步介绍【一】](https://blog.csdn.net/qq_30993595/article/details/82079586)
* [OKHttp3–流程分析 核心类介绍 同步异步请求源码分析【二】](https://blog.csdn.net/qq_30993595/article/details/86438864)
* [OKHttp3–Dispatcher 分发器源码解析【三】](https://blog.csdn.net/qq_30993595/article/details/86681210)
* [OKHttp3–调用对象 RealCall 源码解析【四】](https://blog.csdn.net/qq_30993595/article/details/86686105)
* [OKHttp3–拦截器链 RealInterceptorChain 源码解析【五】](https://blog.csdn.net/qq_30993595/article/details/87475028)
* [OKHttp3–重试及重定向拦截器 RetryAndFollowUpInterceptor 源码解析【六】](https://blog.csdn.net/qq_30993595/article/details/87523713)
* [OKHttp3–桥接拦截器 BridgeInterceptor 源码解析及相关 http 请求头字段解析【七】](https://blog.csdn.net/qq_30993595/article/details/87738481)
* [OKHttp3–缓存拦截器 CacheInterceptor 源码解析【八】](https://blog.csdn.net/qq_30993595/article/details/87886518)
* [OKHttp3-- HTTP 缓存机制解析 缓存处理类 Cache 和缓存策略类 CacheStrategy 源码分析 【九】](https://blog.csdn.net/qq_30993595/article/details/87941893)

## OkHttp请求流程

![img](https://img-blog.csdnimg.cn/20190113214751753.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMwOTkzNTk1,size_16,color_FFFFFF,t_70)

## 核心类 

* OkHttpClient：整个框架的客户端类，OkHttpClient 强烈建议全局单例使用，因为每一个 OkHttpClient 都有自己单独的连接池和线程池，复用连接池和线程池能够减少延迟、节省内存
* ConnectionPool：管理 HTTP 和 HTTP / 2 连接的重用，以减少网络延迟。 相同 Address 的 HTTP 请求可以共享 Connection。 此类实现了哪些连接保持打开以供将来使用的策略
* 。。。。。(参加原文)

## 使用方法切入点

* 同步请求方法：Call.execute，在当前线程执行请求，会阻塞当前线程
* 异步请求方法：Call.enqueue，新开一个线程执行请求，不会阻塞当前线程

## 同步和异步请求代码

* 第一步：实例化 OKHttpClient 对象
* 第二步：实例化网络请求对象 Request
* 第三步：实例化一个准备执行的请求的对象
* 第四步：执行请求方法

### 同步请求

```java
OkHttpClient httpClient = new OkHttpClient
                    .Builder()
                    .connectTimeout(3000, TimeUnit.SECONDS)
                    .readTimeout(3000,TimeUnit.SECONDS)
                    .build();

    public String syncRequest(String url){

            String result = null;

            Request request = new Request
                    .Builder()
                    .url(url)
                    .build();

            Call call = httpClient.newCall(request);
            Response response = null;

            try {
                response = call.execute();

                if(response.isSuccessful()){
                    result = response.body().string();
                }

            } catch (IOException e) {
                e.printStackTrace();
            }

            return result;
        }
```

### 异步请求 

```java
OkHttpClient httpClient = new OkHttpClient
                    .Builder()
                    .connectTimeout(3000, TimeUnit.SECONDS)
                    .readTimeout(3000,TimeUnit.SECONDS)
                    .build();
     
    public void asyncRequest(String url){

        Request request = new Request
                .Builder()
                .url(url)
                .build();

        Call call = httpClient.newCall(request);

        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //请求失败了，这里是在子线程回调，不能在这里直接更新UI
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                //获取请求结果，这里是在子线程回调
                String result = response.body().string();
            }
        });
    }
```

## 同步和异步请求源码分析

从上面的代码可以看出来，不管是同步和异步，在真正执行请求前，也就是 execute 和 enqueue 方法调用前所做的操作都是一样的，接下来我们就从这相同部分开始看

### OKHttpClient 实例化

我们这里是通过它的静态内部类 Builder 构建 OKHttpClient，也就是使用常见的构建者模式，代码如下

```java
public Builder() {
      //实例化分发器
      dispatcher = new Dispatcher();
      //设置支持的网络协议，默认支持HTTP/2和http/1.1
      protocols = DEFAULT_PROTOCOLS;
      //设置支持的连接，默认是使用SNI和ALPN等扩展的现代TLS连接和未加密、未认证的http连接
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      //Call状态监听器
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      //使用默认的代理选择器
      proxySelector = ProxySelector.getDefault();
      //默认没有cookie
      cookieJar = CookieJar.NO_COOKIES;
      //创建socket工厂类
      socketFactory = SocketFactory.getDefault();
      //下面四个是安全相关的配置
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      //实例化连接池 使用连接池技术减少请求的延迟(如果SPDY是可用的话)
      connectionPool = new ConnectionPool();
      //域名解析系统
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      //为了保持长连接，我们必须间隔一段时间发送一个ping指令进行保活
      pingInterval = 0;
    }
```

这里有两个很重要的操作

* 一个就是实例化 Dispatcher，由这个对象对 OKHttp 中的所有网络请求任务进行调度，我们发送的同步或异步请求都会由它进行管理；它里面维护了三个队列，上面有说到，后面在分析这个类的时候会详细介绍其操作
* 还有一个就是实例化了连接池 ConnectionPool，我们常说 OKHttp 有多路复用机制和减少网络延迟功能就是由这个类去实现；我们将客户端与服务端之间的连接抽象为一个 connection（接口），其实现类是 RealConnection，为了管理所有 connection，就产生了 ConnectionPool 这个类；当一些 connection 共享相同的地址，这时候就可以复用 connection；同时还实现了某些 connection 保持连接状态，以备后续复用（不过是在一定时间限制内，不是永远保持打开的）

### Request 实例化

虽然 Request 构建如下方所示



```java
Request request = new Request
                .Builder()
                .url(url)
                .build();
```



但是整个代码实现流程是



- 先通过 Request 构建内部 builder 对象，在构建它的过程，默认指定请求方法为 get，然后又构建了一个请求头 Headers 的 Builder 对象

  ```java
      public Builder() {
        this.method = "GET";
        this.headers = new Headers.Builder();
      }
  ```

- Headers 类的 Builder 对象没有构造方法，只不过内部定义了一个 List 数组，用来存放头部信息

  ```java
  public static final class Builder {
      final List<String> namesAndValues = new ArrayList<>(20);
  }
  ```

- 然后通过 Request 的 Builder 对象设置 url，再调用 build 方法，该方法实例化了 Request 对象并返回

  ```java
      public Request build() {
        if (url == null) throw new IllegalStateException("url == null");
        return new Request(this);
      }
  ```

- 在实例化 Request 的时候，Request 的构造方法里又调用了内部 Builder 对象内部的 Headers.Builder 对象的 build 方法，这个方法实例化了 Headers 对象，Headers 对象的构造方法将 List 数组转换成了 String 数组

  ```java
    Request(Builder builder) {
      this.url = builder.url;
      this.method = builder.method;
      this.headers = builder.headers.build();
      this.body = builder.body;
      this.tag = builder.tag != null ? builder.tag : this;
    }
  
  public Headers build() {
    return new Headers(this);
  }
  Headers(Builder builder) {
  this.namesAndValues = builder.namesAndValues.toArray(new String[builder.namesAndValues.size()]);
  }
  ```

### Call实例化

Call 在 OKHttp 中是一个接口，它抽象了用户对网络请求的一些操作，比如执行请求 (enqueue 方法和 execute 方法)，取消请求(cancel 方法) 等，真正实现是 RealCall 这个类



它的实例化是由 OkHttpClient 对象的 newCall 方法去完成的



```java
 /**
   * 准备在将来某个时间点执行Request
   */
  @Override 
  public Call newCall(Request request) {
    return new RealCall(this, request, false /* for web socket */);
  }
```



这里 RealCall 的构造方法第一个参数是 OkHttpClient 对象，第二个参数是 Request 对象，第三个参数是跟 WebSocket 请求有关，这个后续在讲到这点的时候再说；至于在 RealCall 里真正做了哪些事需要到构造方法看看



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



- 首先 RealCall 内部会持有 OkHttpClient 和 Request 这两个对象的引用
- 实例化了 RetryAndFollowUpInterceptor 对象，这是一个拦截器，上面说到，它是一个请求重试和重定向的拦截器，后续在讲到拦截器的时候具体分析它
- 最后持有一个 Call 状态监听器的引用

总结：不管是同步请求还是异步请求，在执行请求前都要做这三个操作，也就是实例化 OKHttp 客户端对象 OkHttpClient，包含请求信息的 Request 对象，一个准备执行请求的对象 Call；接下来就要分道扬镳了

### 执行同步请求 execute

```java
 Call.execute();

  @Override 
  public Response execute() throws IOException {
    //第一步
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    //第二步
    captureCallStackTrace();
    try {
    //第三步
      client.dispatcher().executed(this);
      //第四步
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
    //第五步
      client.dispatcher().finished(this);
    }
  }
```



- 第一步：首先出现一个同步代码块，对当前对象加锁，通过一个标志位 executed 判断该对象的 execute 方法是否已经执行过，如果执行过就抛出异常；这也就是同一个 Call 只能执行一次的原因
- 第二步：这个是用来捕获 okhttp 的请求堆栈信息不是重点
- 第三步：调用 Dispatcher 的 executed 方法，将请求放入分发器，这是非常重要的一步
- 第四步：通过拦截器连获取返回结果 Response
- 第五步：调用 dispatcher 的 finished 方法，回收同步请求



这里我们来看下第三步源码：



```java
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```



可以看到这里的逻辑很简单，只是将请求添加到了 runningSyncCalls 中，它是一个队列，看看它的定义



```java
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```



如果大家看过 AsyncTask 的源码，一定对 ArrayDeque 这个数据结构熟悉，不熟悉的可以看看博主的[从源码解析 - Android 数据结构之双端队列 ArrayDeque](https://blog.csdn.net/qq_30993595/article/details/80799444)；这个队列会保存正在进行的同步请求（包含了已取消但未完成的请求）

接下来我们就要想了，executed 方法只是将请求存到了队列当中，那什么时候去执行这个请求呢？上面说到过拦截器和拦截器链的概念，那么我们就回到上面的第四步去看看它的真面目

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

我们通过 Call.execute() 方法进行同步请求获取最终的返回结果就是通过这个方法实现的，看看内部逻辑：

- 实例化一个 List，用来存放 Interceptor 对象
- addAll(client.interceptors()) 这一步用来添加用户自定义的拦截器
- 依次添加 OKHttp 内部的五个拦截器
- 创建了一个 RealInterceptorChain 对象，它构建了一个拦截器链，通过 proceed 方法将这些拦截器一一执行

具体逻辑来看它的内部实现

#### ``RealInterceptorChain``

```java
private final List<Interceptor> interceptors;
private final StreamAllocation streamAllocation;
private final HttpCodec httpCodec;
private final RealConnection connection;
private final int index;
private final Request request;
private int calls;
  
public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, RealConnection connection, int index, Request request) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
  }
```

- interceptors 存放了众多拦截器
- 其中第二，第三，第四个参数都是 null
- index 默认传的是 0，其实是 interceptors 数组的索引，在 proceed 方法获取数组中的拦截器用的
- request 就是包含请求信息的对象



#### ``RealInterceptorChain.proceed``

```java
  @Override 
  public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }
```

这里的设计有点奇怪，构造方法中已经传了 Request ，然后在 proceed 方法又传一遍

没啥逻辑，就是调用另一个重载的方法

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
}
```

- 如果索引 index 大于 interceptors 的大小，就抛出异常；然后将 calls 加一
- 接下来的几个关于 httpCodec 的 if 判断暂时不用管

我们重点看中间的几行代码：
又实例化了一个 RealInterceptorChain 对象，然后根据 index 从 interceptors 列表中取出下一个拦截器，最后调用该拦截器的 intercept 方法，并将拦截器链对象传递进去，最终获取结果返回

这里我们要看到一个很重要的点就是在实例化新的 RealInterceptorChain 时候，传入的参数是 index+1；这样当第一个拦截器执行自己的 intercept 方法，做相关的逻辑，如果能获取结果就返回，不再继续执行接下来的拦截器；如果需要通过下一个拦截器获取结果，那就通过传入的参数 RealInterceptorChain 调用 proceed 方法，这样在这个方法里根据递增的索引 index 就能不断的从 interceptors 列表中取出下一个拦截器，执行每个拦截器自己的逻辑获取结果，直到所有拦截器执行完毕；这就是 OKHttp 拦截器链的由来，也是它的精妙所在

至于每个拦截器具体逻辑，在后续文章给出

接下来看看第五步



#### ``dispatcher.finished``

当队列中的请求执行完毕后该怎么处理呢？难道继续存放在队列中？我们来看看

```java
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }
```

当同步请求执行到这里说明这个请求已经完成了，需要进行收尾工作

```java
  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }
```

第一个参数是正在进行同步请求的队列，第二个参数是 Call 对象，第三个参数是调整异步请求队列的标志

这里也有一个同步代码块，因为队列是线程不安全的实现；首先将这个 Call 对象从队列中移出，不能移出就抛出异常；这里 promoteCalls 是 false，所以 promoteCalls 方法不执行；接下来调用 runningCallsCount 方法，计算目前正在执行的请求，看看具体实现

```java
  public synchronized int runningCallsCount() {
    return runningAsyncCalls.size() + runningSyncCalls.size();
  }
```

返回的值是正在进行的同步请求数和正在进行的异步请求数

然后给 idleCallback 赋值

最后判断 runningCallsCount 是否为 0，这说明整个 dispatcher 分发器内部没有维护正在进行的请求了，同时 idleCallback 不为 null，那就执行它的 run 方法（每当 dispatcher 内部没有正在执行的请求时进行的回调）

可以看出来，在进行同步请求的时候 dispatcher 分发器做的事很简单，就是添加请求到队列，从队列移除请求，那异步请求的时候是如何呢？详情参考原文。

