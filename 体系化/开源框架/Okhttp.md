### OkHttp介绍

由Square公司贡献的一个处理网络请求的开源项目，是目前Android使用最广泛的网络框架。从Android4.4开始 HttpURLConnection的底层实现采用的是OkHttp。

- 支持HTTP/2并允许对同一主机的所有请求共享一个套接字
- 连接池复用底层TCP(Socket)，减少请求延时
- 默认通过GZip压缩数据
- 缓存相应数据减少重复的网络请求
- 请求失败自动重试主机的其他ip，自动重定向

### 使用方法

```java
class Test {
    //    OkHttpClient client = new OkHttpClient();
      OkHttpClient client = new OkHttpClient.Builder().build();

    void get(String url) throws IOException {
        Request request = new Request.Builder()
                .url(url)
                .get()
                .build();
        Call call = client.newCall(request);
        //执行同步请求
//        Response response = call.execute();
//        ResponseBody body = response.body();
//        System.out.println(body.toString());
        call.enqueue(new Callback() {
            @Override
            public void onResponse(Call call, Response response) throws IOException {
                ResponseBody body = response.body();
                System.out.println(body.toString());
            }

            @Override
            public void onFailure(Call call, IOException e) {
            }
        });
    }

    void post(String url) throws IOException{
        RequestBody requestBody = new FormBody.Builder()
                .add("name", "liujian")
                .build();
        Request request = new Request.Builder()
                .url(url)
                .post(requestBody)
                .build();
        Call call = client.newCall(request);
        //执行同步请求
//        Response response = call.execute();
//        ResponseBody body = response.body();
//        System.out.println(body.toString());
        call.enqueue(new Callback() {
            @Override
            public void onResponse(Call call, Response response) throws IOException {
                ResponseBody body = response.body();
                System.out.println(body.toString());
            }

            @Override
            public void onFailure(Call call, IOException e) {
            }
        });
    }

}
```

### 调用流程

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/okhttp.jpg" alt="okhttp" style="zoom:50%;" />

在使用OkHttp发起一次请求时，对于使用者最少存在 OkHttpClient 、 Request 与 Call 三个角色。其中OkHttpClient 和 Request 的创建可以使用它为我们提供的 Builder （建造者模式）。而 Call 则是把 Request 交 给 OkHttpClient 之后返回的一个已准备好执行的请求。

同时OkHttp在设计时采用的门面模式，将整个系统的复杂性给隐藏起来，将子系统接口通过一个客户端OkHttpClient统一暴露出来。

OkHttpClient 中全是一些配置，比如代理的配置、ssl证书的配置等。而 Call 本身是一个接口，我们获得的实现为: RealCall。Call 的 execute 代表了同步请求，而 enqueue 则代表异步请求。两者唯一区别在于一个会直接发起网络请求，而另一个使用OkHttp内置的线程池来进行。这就涉及到OkHttp的任务分发器。

### 分发器

Dispatcher，分发器就是用来调配请求任务的，内部会包含一个线程池。可以在创建 OkHttpClient 时，传递我们自己定义的线程池来创建分发器。这个Dispatcher中的成员有:

```java
//异步请求同时存在的最大请求 
private int maxRequests = 64; 
//异步请求同一域名同时存在的最大请求 
private int maxRequestsPerHost = 5; 
//闲置任务(没有请求时可执行一些任务，由使用者设置) 
private @Nullable Runnable idleCallback; 
//异步请求使用的线程池 
private @Nullable ExecutorService executorService; 
//异步请求等待执行队列 
private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>(); 
//异步请求正在执行队列 
private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>(); 
//同步请求正在执行队列 
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

#### 同步请求

```java
public Response execute() throws IOException {
   //···省略···
  try {
       this.client.dispatcher().executed(this);
       //真正执行请求
       Response result = this.getResponseWithInterceptorChain();
  }
  //···省略···
   finally {
       this.client.dispatcher().finished(this);
   }
}
```

因为同步请求不需要线程池，也不存在任何限制。所以分发器仅做一下记录。

#### 异步请求

```java
//RealCall
public void enqueue(Callback responseCallback) {
    synchronized(this) {
      if (this.executed) {
        throw new IllegalStateException("Already Executed");
      }

      this.executed = true;
    }

    this.captureCallStackTrace();
    this.eventListener.callStart(this);
    //---->1
    this.client.dispatcher().enqueue(new RealCall.AsyncCall(responseCallback));
}

//---->1 Dispatcher
void enqueue(AsyncCall call) {
    synchronized(this) {
      this.readyAsyncCalls.add(call);
    }

    this.promoteAndExecute();
}

private boolean promoteAndExecute() {
    assert !Thread.holdsLock(this);
    List<AsyncCall> executableCalls = new ArrayList();
    boolean isRunning;
    AsyncCall asyncCall;
    synchronized(this) {
      Iterator i = this.readyAsyncCalls.iterator();

      while(true) {
        if (i.hasNext()) {
          asyncCall = (AsyncCall)i.next();
          //判断是否需要将ready队列中的任务移动到running队列中
          //条件是请求中的任务数小于64，且同一host任务小于5
          if (this.runningAsyncCalls.size() < this.maxRequests) {
            if (this.runningCallsForHost(asyncCall) < this.maxRequestsPerHost) {
              i.remove();
              executableCalls.add(asyncCall);
              this.runningAsyncCalls.add(asyncCall);
            }
            continue;
          }
        }

        isRunning = this.runningCallsCount() > 0;
        break;
      }
    }

    int i = 0;

    for(int size = executableCalls.size(); i < size; ++i) {
      asyncCall = (AsyncCall)executableCalls.get(i);
      //在线程池中执行，asyncCall继承Runnable
      asyncCall.executeOn(this.executorService());
    }

    return isRunning;
}

//RealCall
final class AsyncCall extends NamedRunnable {
        //···省略···
        void executeOn(ExecutorService executorService) {
            assert !Thread.holdsLock(RealCall.this.client.dispatcher());

            boolean success = false;

            try {
                executorService.execute(this);
                success = true;
            } catch (RejectedExecutionException var8) {
                InterruptedIOException ioException = new InterruptedIOException("executor rejected");
                ioException.initCause(var8);
                RealCall.this.eventListener.callFailed(RealCall.this, ioException);
                this.responseCallback.onFailure(RealCall.this, ioException);
            } finally {
                if (!success) {
                    RealCall.this.client.dispatcher().finished(this);
                }

            }

        }

        protected void execute() {
            boolean signalledCallback = false;
            RealCall.this.timeout.enter();

            try {
                //真正执行请求
                Response response = RealCall.this.getResponseWithInterceptorChain();
                if (RealCall.this.retryAndFollowUpInterceptor.isCanceled()) {
                    signalledCallback = true;
                    this.responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
                } else {
                    signalledCallback = true;
                    this.responseCallback.onResponse(RealCall.this, response);
                }
            } 
            //···省略···
          finally {
                RealCall.this.client.dispatcher().finished(this);
            }

        }
    }

public abstract class NamedRunnable implements Runnable {
    protected final String name;

    public NamedRunnable(String format, Object... args) {
        this.name = Util.format(format, args);
    }

    public final void run() {
        String oldName = Thread.currentThread().getName();
        Thread.currentThread().setName(this.name);

        try {
            this.execute();
        } finally {
            Thread.currentThread().setName(oldName);
        }

    }

    protected abstract void execute();
}
```

当正在执行的任务未超过最大限制64，同时 runningCallsForHost(call) < maxRequestsPerHost 同一Host的请求不超过5个，则会添加到正在执行队列，同时提交给线程池。否则先加入等待队列。加入线程池直接执行没啥好说的，但是如果加入等待队列后，就需要等待有空闲名额才开始执行。因此每次执行完一个请求后，都会调用分发器的 finished 方法

```java
void finished(AsyncCall call) {
  this.finished(this.runningAsyncCalls, call);
}

void finished(RealCall call) {
  this.finished(this.runningSyncCalls, call);
}

private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized(this) {
      if (!calls.remove(call)) {
        throw new AssertionError("Call wasn't in-flight!");
      }

      idleCallback = this.idleCallback;
    }

    //再次进行调度分配
    boolean isRunning = this.promoteAndExecute();
    if (!isRunning && idleCallback != null) {
      idleCallback.run();
    }
}
```

在满足条件下，会把等待队列中的任务移动到 runningAsyncCalls 并交给线程池执行。所以分发器到这里就完了。逻辑上还是非常简单的。不管是同步请求，还是异步请求，最后都会通过Response response = RealCall.this.getResponseWithInterceptorChain()执行真正的请求工作，这个就是OkHttp的核心：拦截器责任链。但是在介绍责任链之前，我们再来看一下分发器中的线程池。

#### 分发器线程池

```java
/**
     * public ThreadPoolExecutor(int corePoolSize,
     *                               int maximumPoolSize,
     *                               long keepAliveTime,
     *                               TimeUnit unit,
     *                               BlockingQueue<Runnable> workQueue,
     *                               ThreadFactory threadFactory,
     *                               RejectedExecutionHandler handler)
     */
//Dispatcher.class
public synchronized ExecutorService executorService() {
    if (this.executorService == null) {
      this.executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue(), Util.threadFactory("OkHttp Dispatcher", false));
    }

    return this.executorService;
}
```

线程池构造函数的参数含义如下：

**corePoolSize**:指定了线程池中的线程数量，它的数量决定了添加的任务是开辟新的线程去执行，还是放到\**workQueue任务队列中去；*（allowCoreThreadTimeOut（boolean）可以设置核心线程是否回收）

***maximumPoolSize**:指定了线程池中的最大线程数量，这个参数会根据你使用的\**workQueue任务队列的类型，决定线程池会开辟的最大线程数量；

**keepAliveTime**:当线程池中空闲线程数量超过corePoolSize时，多余的线程会在多长时间内被销毁；

**unit**:keepAliveTime的单位

**workQueue**:任务队列，被添加到线程池中，但尚未被执行的任务；它一般分为直接提交队列、有界任务队列、无界任务队列、优先任务队列几种；

**threadFactory**:线程工厂，用于创建线程，一般用默认即可；

**handler:拒绝策略**；当任务太多来不及处理时，如何拒绝任务；

在OkHttp的分发器中的线程池定义如上，其实就和 Executors.newCachedThreadPool() 创建的线程一样。首先核心线程为0，表示线程池不会一直为我们缓存线程，线程池中所有线程都是在60s内没有工作就会被回收。而最大线程 Integer.MAX_VALUE 与等待队列 SynchronousQueue 的组合能够得到最大的吞吐量。即当需要线程池执行任务时，如果不存在空闲线程不需要等待，马上新建线程执行任务！等待队列的不同指定了线程池的不同排队机制。一般来说，等待队列 BlockingQueue 有： ArrayBlockingQueue 、 LinkedBlockingQueue 与 SynchronousQueue 。

假设向线程池提交任务时，核心线程都被占用的情况下：

**ArrayBlockingQueue** ：基于数组的阻塞队列，初始化需要指定固定大小。当使用此队列时，向线程池提交任务，会首先加入到等待队列中，当等待队列满了之后，再次提交任务，尝试加入队列就会失败，这时就会检查如果当前线程池中的线程数未达到最大线程，则会新建线程执行新提交的任务。所以最终可能出现后提交的任务先执行，而先提交的任务一直在等待。

**LinkedBlockingQueue** ：基于链表实现的阻塞队列，初始化可以指定大小，也可以不指定。当指定大小后，行为就和 ArrayBlockingQueu 一致。而如果未指定大小，则会使用默认的 Integer.MAX_VALUE 作为队列大小。这时候就会出现线程池的最大线程数参数无用，因为无论如何，向线程池提交任务加入等待队列都会成功。最终意味着所有任务都是在核心线程执行。如果核心线程一直被占，那就一直等待。

**SynchronousQueue** : 无容量的队列。使用此队列意味着希望获得最大并发量。因为无论如何，向线程池提交任务，往队列提交任务都会失败。而失败后如果没有空闲的非核心线程，就会检查如果当前线程池中的线程数未达到最大线程，则会新建线程执行新提交的任务。完全没有任何等待，唯一制约它的就是最大线程数的个数。因此一般配合 Integer.MAX_VALUE 就实现了真正的无等待。

但是需要注意的时，我们都知道，进程的内存是存在限制的，而每一个线程都需要分配一定的内存。所以线程并不能无限个数。那么当设置最大线程数为 Integer.MAX_VALUE 时，OkHttp同时还有最大请求任务执行个数: 64的限制。这样即解决了这个问题同时也能获得最大吞吐。

### 拦截器

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20211223154153303.png" alt="image-20211223154153303" style="zoom:50%;" />

- 重试重定向拦截器：RetryAndFollowUpInterceptor

  第一个拦截器，主要完成两件事情：重试与重定向

  **重试**：请求阶段发生了 RouteException 或者 IOException会进行判断是否重新发起请求。

  ```java
  try {
      response = realChain.proceed(request, streamAllocation, (HttpCodec)null, (RealConnection)null);
      releaseConnection = false;
  } catch (RouteException var19) {
      if (!this.recover(var19.getLastConnectException(), streamAllocation, false, request)) {
        throw var19.getFirstConnectException();
      }
  
      releaseConnection = false;
      continue;
  } catch (IOException var20) {
      boolean requestSendStarted = !(var20 instanceof ConnectionShutdownException);
      if (!this.recover(var20, streamAllocation, requestSendStarted, request)) {
        throw var20;
      }
  
      releaseConnection = false;
      continue;
  }
  ```

  可以看到两个异常都是根据 recover 方法判断是否能够进行重试，如果返回 true ，则表示允许重试。

  ```java
  private boolean recover(IOException e, StreamAllocation streamAllocation, boolean requestSendStarted, Request userRequest) {
      streamAllocation.streamFailed(e);
      //todo 1、在配置OkhttpClient是设置了不允许重试（默认允许），则一旦发生请求失败就不再重试
      if (!this.client.retryOnConnectionFailure()) {
        return false;
      } 
      //todo 2、如果是RouteException，不用管这个条件， 
      // 如果是IOException，由于requestSendStarted只在http2的io异常中可能为false，所以主要是第二个条件
      else if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) {
        return false;
      } 
      //todo 3、判断是不是属于重试的异常
      else if (!this.isRecoverable(e, requestSendStarted)) {
        return false;
      } else {
        //todo 4、有没有可以用来连接的路由路线
        return streamAllocation.hasMoreRoutes();
      }
  }
  ```

  所以首先使用者在不禁止重试的前提下，如果出现了**某些异常**，并且存在更多的路由线路，则会尝试换条线路进行

  请求的重试。其中**某些异常**是在 isRecoverable 中进行判断:

  ```java
  private boolean isRecoverable(IOException e, boolean requestSendStarted) {
      // 出现协议异常，不能重试
      if (e instanceof ProtocolException) {
        return false;
      } 
      // 如果不是超时异常，不能重试
      else if (!(e instanceof InterruptedIOException)) {
        // SSL握手异常中，证书出现问题，不能重试
        if (e instanceof SSLHandshakeException && e.getCause() instanceof CertificateException) {
          return false;
        } else {
          // SSL握手未授权异常 不能重试
          return !(e instanceof SSLPeerUnverifiedException);
        }
      } else {
        return e instanceof SocketTimeoutException && !requestSendStarted;
      }
  }
  ```

  1、**协议异常**，如果是那么直接判定不能重试;（你的请求或者服务器的响应本身就存在问题，没有按照http协议来

  定义数据，再重试也没用）

  2、**超时异常**，可能由于网络波动造成了Socket连接的超时，可以使用不同路线重试。

  3、**SSL证书异常/SSL验证失败异常**，前者是证书验证失败，后者可能就是压根就没证书，或者证书数据不正确，

  那还怎么重试？

  经过了异常的判定之后，如果仍然允许进行重试，就会再检查当前有没有可用路由路线来进行连接。简单来说，比

  如 DNS 对域名解析后可能会返回多个 IP，在一个IP失败后，尝试另一个IP进行重试。

​       **重定向**：如果请求结束后没有发生异常并不代表当前获得的响应就是最终需要交给用户的，还需要进一步来判断是否需要重定向的判断。重定向的判断位于 followUpRequest 方法。整个是否需要重定向的判断内容很多，记不住，这很正常，关键在于理解他们的意思。如果此方法返回空，那就表示不需要再重定向了，直接返回响应；但是如果返回非空，那就要重新请求返回的 Request ，但是需要注意的是，我们的 followup 在拦截器中定义的最大次数为**20**次。

- 桥接拦截器：BridgeInterceptor

  连接应用程序和服务器的桥梁，我们发出的请求将会经过它的处理才能发给服务器，比如设置请求内容长度，编码，gzip压缩，cookie等，获取响应后保存Cookie等操作。这个拦截器相对比较简单。

  在补全了请求头后交给下一个拦截器处理，得到响应后，主要干两件事情：

  1、保存cookie，在下次请求则会读取对应的数据设置进入请求头，默认的 CookieJar 不提供实现

  2、如果使用gzip返回的数据，则使用 GzipSource 包装便于解析。

  

- 缓存拦截器：CacheInterceptor

  在发出请求前，判断是否命中缓存。如果命中则可以不请求，直接使用缓存的响应。 (只会存在Get请求的缓存)

- 连接拦截器：ConnectInterceptor

  主要是获取请求服务器用的socket，可以从连接池ConnectionPool中获取，获取不到则new出来。

  ConnectionPool主要三个方法：put(RealConnection connection)，RealConnection get()，cleanup()

  put():

  ```java
  void put(RealConnection connection) {
      assert Thread.holdsLock(this);
       
      //如果没有启动连接池清除任务，则启动
      if (!this.cleanupRunning) {
        this.cleanupRunning = true;
        //在线程池中执行清除任务
        executor.execute(this.cleanupRunnable);
      }
      //直接添加到连接池
      this.connections.add(connection);
  }
  ```

  put方法直接先将连接添加进连接池，同时检查清除任务是否开启，清除任务会循环清理连接。

  cleanup()：

  ```java
  //cleanupRunnable
  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
      this.cleanupRunnable = new Runnable() {
        public void run() {
          while(true) {
            long waitNanos = ConnectionPool.this.cleanup(System.nanoTime());
            if (waitNanos == -1L) {
              return;
            }
  
            if (waitNanos > 0L) {
              long waitMillis = waitNanos / 1000000L;
              waitNanos -= waitMillis * 1000000L;
              synchronized(ConnectionPool.this) {
                try {
                  ConnectionPool.this.wait(waitMillis, (int)waitNanos);
                } catch (InterruptedException var8) {
                }
              }
            }
          }
        }
      };
      //··
  }
  ```

  

- 请求服务器拦截器：CallServerInterceptor

  














