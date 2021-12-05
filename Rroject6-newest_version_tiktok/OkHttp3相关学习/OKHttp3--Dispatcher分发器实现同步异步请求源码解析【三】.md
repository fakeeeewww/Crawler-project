[toc]

# OKHttp3--Dispatcher分发器实现同步异步请求源码解析【三】.md

## 数据结构

在 Dispatcher 中维护了三个队列，队列类型是 ArrayDeque，这是一个双端队列(双端队列可以从队列头部和尾部进行操作，都满足FIFO原则，即先进去可以先执行)。具体细节可参考[从源码解析 - Android 数据结构之双端队列 ArrayDeque 实现 FIFO 和 LIFO 队列](https://blog.csdn.net/qq_30993595/article/details/80799444)

* 第一个队列是 runningSyncCalls，是一个**正在执行**的同步请求队列，所有我们添加的同步请求都会被添加到这里面，包括已被取消但没执行完的请求，队列泛型(泛型的本质是参数化类型，也就是说所操作的数据类型被指定为一个参数，参数类型不定)是 RealCall 对象。
* 第二个队列是 runningAsyncCalls，是一个**正在执行**的异步请求队列，所有我们添加的异步请求都会被添加到这里，包括已被取消但没执行完的请求，队列泛型是 AsyncCall 对象，实现了 Runnable 接口。
* 第三个队列是 readyAsyncCalls，是一个**等待执行**的异步请求队列。

要知道通过这些队列，OKHttp 可以轻松的实现并发请求，更方便的维护请求数以及后续对这些请求的操作（比如取消请求），大大提高网络请求效率；同时可以更好的管理请求数，防止同时运行的线程过多，导致 OOM，同时限制了同一 hostname 下的请求数，防止一个应用占用的网络资源过多，优化用户体验。

## 线程池

OKHttp 在其内部维护了一个线程池，用于执行异步请求 AsyncCall，其构造如下。

```java
private ExecutorService executorService;
public synchronized ExecutorService executorService() {
  if (executorService == null) {
    executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
        new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
  }
  return executorService;
}
```

我们需要关注的就是 ThreadPoolExecutor 前三个参数

* 第一个是 0，说明该线程池没有核心线程，所有线程都是工作线程，即所有线程超过一定空闲时间会被回收
* 第二个参数是 Integer.MAX_VALUE，即最大线程数，虽然设置值这么大，但是无须担心性能消耗过大问题，因为有队列去维护请求数
* 第三个参数是 60，即工作线程空闲 60s 后就会被回收

关于线程池更多信息可参考 [Android 开发 - 通过 ExecutorService 构建一个 APP 使用的全局线程池](https://blog.csdn.net/qq_30993595/article/details/84324681)

## 请求管理

```java
private int maxRequests = 64;
private int maxRequestsPerHost = 5;
```

在类里定义了这两个变量，其含义是默认支持的最大并发请求数量是 64 个，单个 host 并发请求的最大数量是 5 个；这两个值是可以通过后续设置进行更改的，并且这个要求只针对异步请求，对于同步请求数量不做限制

当异步请求进入 Dispatcher 中，如果满足上面两个数量要求，该请求会被添加到 runningAsyncCalls 中，然后执行它；如果不满足就将其添加到 readyAsyncCalls 中；当一个异步请求结束时，会遍历 readyAsyncCalls 队列，再进行条件判断，符合条件就将请求从该队列移到 runningAsyncCalls 队列中并执行它

## Dispatcher源码
```
public final class Dispatcher {
  //最大并发请求数
  private int maxRequests = 64;
  //单个主机最大并发请求数
  private int maxRequestsPerHost = 5;

  private Runnable idleCallback;

  /** 执行AsyncCall的线程池 */
  private ExecutorService executorService;

  /** 等待执行的异步请求队列. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** 正在执行的异步请求队列，包含已取消但为执行完的请求 */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** 正在执行的同步请求队列，包含已取消但为执行完的请求 */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  //构造方法 接收一个线程池
  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  //构建一个线程池
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }

  /**
   * 设置并发执行的最大请求数
      */

    public synchronized void setMaxRequests(int maxRequests) {
        if (maxRequests < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequests);
        }
        this.maxRequests = maxRequests;
        promoteCalls();
    }

  public synchronized int getMaxRequests() {
    return maxRequests;
  }

  /**
   * 设置每个主机同时执行的最大请求数
      */

    public synchronized void setMaxRequestsPerHost(int maxRequestsPerHost) {
        if (maxRequestsPerHost < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequestsPerHost);
        }
        this.maxRequestsPerHost = maxRequestsPerHost;
        promoteCalls();
    }

  public synchronized int getMaxRequestsPerHost() {
    return maxRequestsPerHost;
  }

  /**
   * 当分发器处于空闲状态下，即没有正在运行的请求，设置回调
      */

    public synchronized void setIdleCallback(Runnable idleCallback) {
        this.idleCallback = idleCallback;
    }

  /**
   * 执行异步请求
   * 当正在执行的异步请求数量小于64且单个host正在执行的请求数量小于5的时候，就执行该请求，并添加到队列
   * 否则添加到等待队列中
      */

    synchronized void enqueue(AsyncCall call) {
        if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
        } else {
      readyAsyncCalls.add(call);
        }
    }

  /**
   * 取消所有请求
      */

    public synchronized void cancelAll() {
        for (AsyncCall call : readyAsyncCalls) {
      call.get().cancel();
        }

​    for (AsyncCall call : runningAsyncCalls) {
​      call.get().cancel();
​    }

​    for (RealCall call : runningSyncCalls) {
​      call.cancel();
​    }
  }

  //调整请求队列，将等待队列中的请求放入正在请求的队列
  private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // 如果正在执行请求的队列已经满了，那就不用调整了.
    if (readyAsyncCalls.isEmpty()) return; // 如果等待队列是空的，也不需要调整

​    //遍历等待队列
​    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
​      AsyncCall call = i.next();
​      //单个host正在执行的请求数量小于5的时候，将该请求添加到runningAsyncCalls中并执行它
​      //同时从等待队列中删除它
​      if (runningCallsForHost(call) < maxRequestsPerHost) {
​        i.remove();
​        runningAsyncCalls.add(call);
​        executorService().execute(call);
​      }

​      if (runningAsyncCalls.size() >= maxRequests) return; // 如果正在执行请求的队列已经满了，就退出循环
​    }
  }

  /** 返回单个host的请求数 */
  private int runningCallsForHost(AsyncCall call) {
    int result = 0;
    for (AsyncCall c : runningAsyncCalls) {
      if (c.host().equals(call.host())) result++;
    }
    return result;
  }

  /** 执行同步请求，只是将其添加到队列中 */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

  /** 异步请求执行完成调用. */
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }

  /** 同步请求执行完成调用. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      //从队列中移除
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      //异步请求才会走这个，调整队列
      if (promoteCalls) promoteCalls();
      //计算当前正在执行的同步和异步请求数量
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

​    if (runningCallsCount == 0 && idleCallback != null) {
​      idleCallback.run();
​    }
  }

  /** 返回当前正在等待执行的异步请求的快照. */
  public synchronized List<Call> queuedCalls() {
    List<Call> result = new ArrayList<>();
    for (AsyncCall asyncCall : readyAsyncCalls) {
      result.add(asyncCall.get());
    }
    return Collections.unmodifiableList(result);
  }

  /** 返回当前正在执行的异步请求的快照. */
  public synchronized List<Call> runningCalls() {
    List<Call> result = new ArrayList<>();
    result.addAll(runningSyncCalls);
    for (AsyncCall asyncCall : runningAsyncCalls) {
      result.add(asyncCall.get());
    }
    return Collections.unmodifiableList(result);
  }

  //返回等待执行的异步请求数量
  public synchronized int queuedCallsCount() {
    return readyAsyncCalls.size();
  }

  //计算正在执行的请求数量
  public synchronized int runningCallsCount() {
    return runningAsyncCalls.size() + runningSyncCalls.size();
  }
}
```

## 参考文献

[(33条消息) OKHttp3--Dispatcher分发器实现同步异步请求源码解析【三】_Mango先生的博客-CSDN博客](https://blog.csdn.net/qq_30993595/article/details/86681210)
