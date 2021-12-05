[toc]

# OKHttp3-- 调用对象 RealCall 源码解析【四】

## 结构

Call(接口而已) >>>>RealCall(真正的实现类)>>>>执行同步请求，ReCall调用**excute**方法，异步调用**enqueue**方法。

## Call

OkHttp执行**Request**时并不是直接执行的，而是把**Request**封装一层为**Call**对象。一个**Call**对象既可以代表一个准备好执行的请求(Request)，又可以代表一个request/response对(Stream)，因此一个**Call**无法被执行两次。

## 实例化**ReCall**

实例化**ReCall**首先是通过 OkHttpClient 的 newCall 方法获取一个 Call 对象

```java
Call call = httpClient.newCall(request);
```

```java
  @Override 
  public Call newCall(Request request) {
    return new RealCall(this, request, false /* for web socket */);
  }
```

这里只是 new 了一个 RealCall 对象，然后返回，进入到 RealCall 对象的构造方法看看。

> 构造方法参考：[构造方法 - 廖雪峰的官方网站 (liaoxuefeng.com)](https://www.liaoxuefeng.com/wiki/1252599548343744/1260454185794944) 这里简单介绍构造方法。我们创建实例的时候，常常需要先new一个对象，然后使用初始化方法来初始化对象。比如。
>
> ```java
> Person ming = new Person();
> ming.setName("小明");
> ming.setAge(12);
> ```
>
> 这里就需要``setName``和``setAge``两个方法来初始化，而且每new一个对象，都要调用初始化方法，不然就会出错。
>
> 但是我们可以通过定义一个构造函数，实现在new对象时直接初始化，避免每次都要输入``对象.setName()``和``对象.setAge()``来初始化，更加方便。
>
> ```java
> public class Main {
>     public static void main(String[] args) {
>         Person p = new Person("Xiao Ming", 15);
>         System.out.println(p.getName());
>         System.out.println(p.getAge());
>     }
> }
> 
> class Person {
>     private String name;
>     private int age;
> public Person(String name, int age) {
>     this.name = name;
>     this.age = age;
> }
> 
> public String getName() {
>     return this.name;
> }
> 
> public int getAge() {
>     return this.age;
> }
> }
> ```
> 之后我们在创建``Person``实例的时候，就可以只写一句代码``Person xxx=new Person("xxx",xxx)``

```java
  RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    final EventListener.Factory eventListenerFactory = client.eventListenerFactory();
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    this.eventListener = eventListenerFactory.create(this);
  }
```

咱们可以看到以上代码第6行，实例化了拦截器链中的第一个拦截器 RetryAndFollowUpInterceptor，也就是**重试**和**重定向**拦截器。重试好理解，请求失败了重新请求；那**重定向**什么意思呢，咱们在以下进行解释。

* 对于转发过程：客户端发送一个请求到服务器，服务器匹配Servlet，并执行。当这个 Servlet 执行完后，在服务器端它要调用 getRequestDispacther() 方法，把请求转发给指定的 okhttp.jsp。
* 对于重定向过程：客户端发送一个请求到服务器端，服务器匹配 Servlet，这都和请求转发一样。Servlet 处理完之后调用了 sendRedirect() 这个方法，这个方法是 response 的方法。response.sendRedirect() 方法执行完立即向客户端返回响应，响应行告诉客户端你必须再重新发送一个请求，去访问 okhttp.jsp；紧接着客户端收到这个请求后，立刻发出一个新的请求，去请求 okhttp.jsp。

Forward 是在服务器端的跳转，就是客户端发一个请求给服务器，服务器直接将请求相关参数的信息原封不动的传递到该服务器的其他 jsp 或 Servlet 去处理。而 sendRedirect() 是客户端的跳转，服务器会返回客户端一个响应报头和新的 URL 地址，原来的参数信息如果服务器没有特殊处理就不存在了，浏览器会访问新的 URL 所指向的 Servlet 或 jsp，这可能不是原来服务器上的 webService 了。

总结：

- 转发是在服务器端完成的，重定向是在客户端发生的
- 转发的速度快，重定向速度慢
- 转发是同一次请求，重定向是两次请求
- 转发时浏览器地址栏没有变化，重定向时地址栏有变化
- 转发必须是在同一台服务器下完成，重定向可以在不同的服务器下完成

## 同步请求

```java
@Override 
public Response execute throws IOException{
    //第一步
    synchronized(this){
        if(executed) throw new IllegalStateException("Already Executed");
        executed=true;
    }
    //第二步
    captureCallStackTrace();
    try{
    //第三步
        client.dispatcher().executed(this);
        //第四步
        Responsed result=getResponseWithInterceptor("Canceled");
        if(result==null)throw new IOException("Canceled");
        return result;
    }finally{
    //第五步
        client.dispatcher().finished(this);
    }
}
```

* 第一步：首先出现一个同步代码块，对当前对象加锁，通过一个标志位 executed 判断该对象的 execute 方法是否已经执行过，如果执行过就抛出异常；这也就是同一个 Call 只能执行一次的原因
* 第二步：这个是用来捕获 okhttp 的请求堆栈信息，不是重点
* 第三步：调用 Dispatcher 的 executed 方法，将请求放入分发器，这是非常重要的一步
* 第四步：通过拦截器连获取返回结果 Response
* 第五步：调用 dispatcher 的 finished 方法，回收同步请求

## 异步请求

异步请求会走到内部的 enqueue 方法



```java
  @Override 
  public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

首先也是加同步，判断这个 Call 是否执行过；然后实例化了一个 AsyncCall，最后调用分发器的 enqueue 方法

## RealCall源码

```java
/**
 * 一个Call封装一对Request和Response，能且仅能被执行一次。并且Call可以被取消。
 */
final class RealCall implements Call {
  //OKHttp客户端
  final OkHttpClient client;
  //重试和重定向拦截器
  final RetryAndFollowUpInterceptor retryAndFollowUpInterceptor;
  //RealCall状态监听器
  final EventListener eventListener;

  //请求对象
  final Request originalRequest;
  final boolean forWebSocket;

  // RealCall是否执行过
  private boolean executed;

  RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    final EventListener.Factory eventListenerFactory = client.eventListenerFactory();

    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);

    this.eventListener = eventListenerFactory.create(this);
  }

  /** 返回初始化此Call的原始请求 */
  @Override 
  public Request request() {
    return originalRequest;
  }

  /** 进行同步请求
  * 立即发出请求，一直阻塞当前线程，直到返回结果或者报错
  * 可以使用Response.body获取结果
  * 为了复用连接，需要调用Response.close关闭响应体
  */
  @Override 
  public Response execute() throws IOException {
    synchronized (this) {
      //同一个RealCall，只能执行一次
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    //打印堆栈信息
    captureCallStackTrace();
    try {
      //加入分发器中的正在执行的同步请求队列
      client.dispatcher().executed(this);
      //获取请求结果
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      //通知分发器请求完成，从队列移除
      client.dispatcher().finished(this);
    }
  }

  private void captureCallStackTrace() {
    Object callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()");
    retryAndFollowUpInterceptor.setCallStackTrace(callStackTrace);
  }
  
  /**
  * 进行异步请求
  * 至于何时执行这个请求由分发器决定
  * 通常是立即执行，除非当前有任务在执行或不符合限制条件
  * 如果不能立即执行，会被保存到等待执行的异步请求队列
  * 请求结束后，会通过回调接口将结果返回
  */
  @Override 
  public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    //实例化一个线程AsyncCall交给分发器，由分发器中的线程池执行这个线程
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }

  //取消这个RealCall 如果请求已经有返回了，那么就不能被取消了
  @Override 
  public void cancel() {
    retryAndFollowUpInterceptor.cancel();
  }

  //判断是否执行过
  @Override 
  public synchronized boolean isExecuted() {
    return executed;
  }

  //判断是否取消了
  @Override 
  public boolean isCanceled() {
    return retryAndFollowUpInterceptor.isCanceled();
  }

  //复制一个相同的RealCall
  @SuppressWarnings("CloneDoesntCallSuperClone")
  @Override public RealCall clone() {
    return new RealCall(client, originalRequest, forWebSocket);
  }

  //StreamAllocation协调Connections，Streams，Calls三者之间的关系
  StreamAllocation streamAllocation() {
    return retryAndFollowUpInterceptor.streamAllocation();
  }

  //异步请求线程
  final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    //主机名
    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    RealCall get() {
      return RealCall.this;
    }

    //当分发器的线程池执行该对象时，该方法被调用
    @Override 
    protected void execute() {
      //保证onFailure只被回调一次
      boolean signalledCallback = false;
      try {
        //通过拦截器获取返回结果
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          //如果请求被取消，回调onFailure
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          // 正常情况，调用onResponse
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        // 如果上面回调过，这里就不再进行回调，保证onFailure只会被调用一次
        if (signalledCallback) {
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        //通知分发器请求结束，从队列中移除该请求
        client.dispatcher().finished(this);
      }
    }
  }

  /**
   * Returns a string that describes this call. Doesn't include a full URL as that might contain
   * sensitive information.
   */
  String toLoggableString() {
    return (isCanceled() ? "canceled " : "")
        + (forWebSocket ? "web socket" : "call")
        + " to " + redactedUrl();
  }

  //返回包含此URL的字符串，无用户名，密码，查询信息
  String redactedUrl() {
    return originalRequest.url().redact();
  }
  
  //依次执行拦截器链中的拦截器获取结果
  Response getResponseWithInterceptorChain() throws IOException {
    List<Interceptor> interceptors = new ArrayList<>();
    //添加自定义拦截器
    interceptors.addAll(client.interceptors());
    //添加重试和重定向拦截器
    interceptors.add(retryAndFollowUpInterceptor);
    //添加桥接拦截器
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    //添加缓存拦截器
    interceptors.add(new CacheInterceptor(client.internalCache()));
    //添加链接拦截器
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      //添加网络拦截器
      interceptors.addAll(client.networkInterceptors());
    }
    //添加连接服务器拦截器，主要负责将我们的Http请求写进网络的IO流中
    interceptors.add(new CallServerInterceptor(forWebSocket));

    //构建拦截器链依次执行每一个拦截器
    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
}
```

## 参考文章

* [(33条消息) OKHttp3--调用对象RealCall源码解析【四】_Mango先生的博客-CSDN博客](https://blog.csdn.net/qq_30993595/article/details/86686105)

