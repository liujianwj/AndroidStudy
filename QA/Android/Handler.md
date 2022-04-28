- Handler的实现原理

  > 切换线程的原理：线程间共享内存。
  >
  > 主要成员：Handler、Message、Looper、MessageQueue、Thread、ThreadLocal、ThreadLocalMap
  >
  > Handler：负责发送消息；
  >
  > Message：消息载体，存储数据、执行体Runnable，Handler引用target，消息类型等信息
  >
  > Looper：负责处理MessageQueue创建，消息读取和退出，主要方法初始化prepare()，loop()开始执行循环读数据。
  >
  > 以及主要成员变量static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
  >
  > MessageQueue：消息队列，优先级队列，单链表结构，重要方法enqueueMessage，next。
  >
  > Thread：线程类，主要关心ThreadLocalMap threadLocals变量，每个线程都有个threadLocals变量
  >
  > ThreadLocal：用于操作内部类ThreadLocalMap
  >
  > ThreadLocalMap：key-value存储，在整个环节中，key->this->Looper.sThreadLocal，value->new Looper()，sThreadLocal是static final全局唯一，但是ThreadLocalMap在每个线程中都有一个，所以可以保证一个线程只有一个Looper。
  >
  > ```java
  > Thread1.   ThreadLocalMap map1 = new ThreadLocalMap()  map1.put(Looper.sThreadLocal, Looper1)
  > Thread2.   ThreadLocalMap map2 = new ThreadLocalMap()  map1.put(Looper.sThreadLocal, Looper2)
  > ```
  >
  > 主要工作原理：首先Looper.prepare()中通过ThreadLocal.get保证每个线程中只有一个Looper，且一个Looper都只有一个MessageQueue对象，然后开启Looper.loop()开启循环读取消息。然后在子线程中通过handler发送消息到MessageQueue，Looper.loop()通过queue.next()获取到消息，通过延迟消息处理/屏障处理后，最后通过msg.target.dispatchMessage(msg)分发消息，handler.dispatchMessage中调用handleMessage(msg)处理消息。因为发送消息在子线程，而Looper.loop()是在主线程一直循环执行，故handleMessage的处理都在主线程中，而MessageQueue是线程共享的，函数执行是区分线程，所以达到了切换线程处理的效果。
  >
  > 细节：当消息延迟时间没到，或者消息队列为空时，MessageQueue.next需要阻塞，阻塞原理是通过native方法，调用Linux底层的epoll.await，唤醒是通过底层wake方法

- 子线程中能不能直接new一个Handler,为什么主线程可以？主线程的Looper第一次调用loop方法,什么时候,哪个类

  > 不可以直接new一个Handler，需要先调用Looper.prepare()（创建Looper ），最后调用Looper.loop（开启循环读取消息）；
  >
  > 主线程在ActivityThread中的main方法中已经完成上面两步操作；
  >
  > ActivityThread中的main方法。

- Handler导致的内存泄露原因及其解决方案

  > Handler内部类持有外部类对象的引用导致
  >
  > Looper持有messageQueue，messageQueue持有message，message持有handler（target，在handler.enqueueMessage中赋值this），handler持有activity。
  >
  > 因为Looper的生命周期更长，根据jvm可达性分析，导致activity一直无法被回收

- 一个线程可以有几个Handler,几个Looper,几个MessageQueue对象

  > 无数个handler，一个Looper，一个MessageQueue

- Message对象创建的方式有哪些 & 区别？Message.obtain()怎么维护消息池的
  
  > 直接new或者通过Message.obtain()获取；
  >
  > 区别在于Message.obtain()当sPool不会空的时候，不用new对象直接获取，为空的时候就直接new；
  >
  > 在Looper.loop中消息处理完之后，会调用msg.recycleUnchecked();来回收消息，即将消息成员变量置空，然后添加到sPool链表头，Message.obtain()时获取第一个message。
  
- Handler 有哪些发送消息的方法

  > post(Runnable)、postAtTime(Runnable, long)、postDelayed(Runnable, long)、postAtFrontOfQueue(Runnable)
  >
  > sendMessage(Message)、sendMessageDelayed( Message msg, long delayMillis)、sendMessageAtTime( Message msg, long uptimeMillis)、sendMessageAtFrontOfQueue(Message msg)

- Handler的post与sendMessage的区别和应用场景

  > 两者后面都是调用到sendMessageAtTime(@NonNull Message msg, long uptimeMillis)->enqueueMessage()->queue.enqueueMessage(msg, uptimeMillis);
  >
  > 区别在于post是：
  >
  > ```java
  > public final boolean post(@NonNull Runnable r) {
  > return  sendMessageDelayed(getPostMessage(r), 0);
  > }
  > private static Message getPostMessage(Runnable r) {
  > Message m = Message.obtain();
  > m.callback = r;
  > return m;
  > }
  > ```
  >
  > 会将Runnable包装进Message；
  >
  > 场景：只需简单处理一个情况下的消息时可以使用post(Runnable)更为简单，sendMessage适合处理多种消息类型。

- handler postDealy后消息队列有什么变化，假设先 postDelay 10s, 再postDelay 1s, 怎么处理这2条消息

  > postDealy会更具delay的时间大小插入到队列，时间越少的在前面，优先处理；
  >
  > postDelay 10s会对从链表头开始比队列中的消息，如果10s比当前对比消息的时间大，则对比下一个，直到找到时间比自己大的，插入到其前面，再postDelay 1s也是一样的对比，最后postDelay 1s的消息会在postDelay 10s的前面优先被处理。

- MessageQueue是什么数据结构

  > 单链表结构，优先级队列，时间越小的排在前面

- Handler怎么做到的一个线程对应一个Looper，如何保证只有一个MessageQueue ThreadLocal在Handler机制中的作用

  > 首先Looper的创建都是通过Looper.prepare()方法，而在此方法中会先直接从threadLocal 中去获取Looper，如果能获取到则直接抛出异常，说明已经创建过Looper，否则创建Looper对象，同时set进ThreadLocal，而Looper构造方法中创建MessageQueue。在ThreadLocal.set和get中，都是先获取当前线程，然后获取线程的ThreadLocalMap threadLocals成员变量，
  >
  > 存取都是在threadLocals中，key是threadLocal，value是Looper对象。
  >
  > 关键代码：
  >
  > ```java
  > //Looper.java
  > class Looper{
  > //静态final对象，全局唯一，作为每个线程中ThreadLocal.ThreadLocalMap threadLocals中的key，存储对应的looper
  > static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
  > 
  > public static void prepare() {
  >     prepare(true);
  > }
  > 
  > private static void prepare(boolean quitAllowed) {
  >    //先获取，检查是否已经创建过
  >    if (sThreadLocal.get() != null) {
  >      throw new RuntimeException("Only one Looper may be created per thread");
  >    }
  >    //如果没有创建过，则创建，并添加进ThreadLocal，其实是添加到当前线程的ThreadLocalMap中，key是sThreadLocal，value是Looper对象
  >    sThreadLocal.set(new Looper(quitAllowed));
  > }
  > 
  > private Looper(boolean quitAllowed) {
  >      //创建MessageQueue
  >      mQueue = new MessageQueue(quitAllowed);
  >      mThread = Thread.currentThread();
  > }
  > }
  > 
  > //ThreadLocal.java
  > class ThreadLocal{
  > public void set(T value) {
  >    //获取当前线程
  >    Thread t = Thread.currentThread();
  >    //获取当前线程的ThreadLocalMap
  >    ThreadLocalMap map = getMap(t);
  >    if (map != null)
  >      map.set(this, value);
  >    else
  >      createMap(t, value);
  > }
  > 
  > public T get() {
  >    //获取当前线程
  >    Thread t = Thread.currentThread();
  >    //获取当前线程的ThreadLocalMap
  >    ThreadLocalMap map = getMap(t);
  >    if (map != null) {
  >      ThreadLocalMap.Entry e = map.getEntry(this);
  >      if (e != null) {
  >        @SuppressWarnings("unchecked")
  >        T result = (T)e.value;
  >        return result;
  >      }
  >    }
  >    return setInitialValue();
  > }
  > 
  > ThreadLocalMap getMap(Thread t) {
  >    return t.threadLocals;
  > }
  > }
  > 
  > //Thread.java
  > class Thread{
  > ThreadLocal.ThreadLocalMap threadLocals = null;
  > }
  > ```
  >
  > 

- HandlerThread是什么 & 好处 &原理 & 使用场景

  > 继承至Thread，自动帮我们在子线程中处理Looper.prepare()/Looper.loop() 等动作。线程启动后，我们可以通过获取looper，往子线程的looper中发送任务交给子线程处理。
  >
  > 好处：1、使用方便，自动处理Looper   2、线程安全，在创建looper，和获取looper时加锁，当发现线程已经启动，但是looper为空时，等待，looper创建成功后会发出通知唤醒。
  >
  > 使用场景：耗时任务需要交给子线程处理的都可以使用，典型的有IntentService内部就是使用的HandlerThread

- IdleHandler及其使用场景

  > 空闲队列，在信息队列为空，或者延时消息还在等待的时候执行
  >
  > 场景：在空闲的时候做一些日志上传等工作，或者可以用于优化app启动流程，把一些不必要第一时间启动的任务放到IdleHandler中等主线程空闲时处理。

- 消息屏障,同步屏障机制

  > https://www.cnblogs.com/renhui/p/12875589.html

- 子线程能不能更新UI

  > 可以， ViewRootImpl创建之前都可以，checkThread是在ViewRootImpl中检查，ViewRootImpl是在performResumeActivity中创建。

- 为什么Android系统不建议子线程访问UI

  > 首先，UI控件不是线程安全的，如果多线程并发访问UI控件可能会出现不可预期的状态，比如ui绘制错乱、交互错乱等，出了问题也很难去排查到底是哪个线程更新时出了问题。
  >
  > 那为什么系统不对UI控件的访问加上锁机制呢？
  > 缺点有两个：
  >
  > - 加上锁机制会让UI访问的逻辑变得复杂；
  > - 锁机制会降低UI访问的效率，因为锁机制会阻塞某些线程的执行

- Android中为什么主线程不会因为Looper.loop()里的死循环卡死

  > Android应用就是个消息驱动的体系，在ActivityThread的main方法中一个是就Looper.loop()进入死循环，处理activity的生命周期消息通知，绘图消息等等，正是因为主线程的死循环，程序才不会退出，一直处理消息，或者等待消息，这样应用才能正常的在前台显示和交互。

- MessageQueue#next 在没有消息的时候会阻塞，如何恢复？

  > 在没有消息的时候，会调用native方法，进入底层调用Linux的epoll.wait方法进行阻塞，恢复也是底层发出awake消息。
  >
  > enqueueMessage方法会将传入的消息对象根据触发时间（when）插入到message queue中。然后判断是否要唤醒等待中的队列。
  > 如果插在队列中间。说明该消息不需要马上处理，不需要由这个消息来唤醒队列。
  > 如果插在队列头部（或者when=0），则表明要马上处理这个消息。如果当前队列正在堵塞，则需要唤醒它进行处理。
  > 如果需要唤醒队列，则通过nativeWake方法，往前面提到的管道中写入一个"W"字符，令nativePollOnce方法返回。

- Handler消息机制中，一个looper是如何区分多个Handler的

  > 根据Message是由哪个handler发送的，则该Message会保存对应的handler，即保存到message.target成员变量，然后在looper.loop中，获取到要处理的message，调用其msg.target.dispatchMessage(msg)处理消息。

- 当Activity有多个Handler的时候，怎么样区分当前消息由哪个Handler处理

  > 同上，Message成员变量 Handler target，会在发送的时候保存对应的handler。

- 处理message的时候怎么知道是去哪个callback处理的

  > msg.target.dispatchMessage(msg)处理消息，target是对应的handler，如果mCallback不为空，即该消息是runnable，则直接处理mCallback，否则调用handler.handleMessage(msg)
  >
  > ```java
  > public void dispatchMessage(@NonNull Message msg) {
  >      if (msg.callback != null) {
  >          handleCallback(msg);
  >      } else {
  >          if (mCallback != null) {
  >              if (mCallback.handleMessage(msg)) {
  >                  return;
  >              }
  >          }
  >          handleMessage(msg);
  >      }
  > }
  > ```
  >
  > 

- Looper.quit/quitSafely的区别

  > Looper的quit/quitSafely方法源码如下：
  >
  > ```java
  > public void quit() {
  >  mQueue.quit(false);
  > }
  > 
  > public void quitSafely() {
  >  mQueue.quit(true);
  > }
  > 
  > void quit(boolean safe) {
  >      if (!mQuitAllowed) {
  >          throw new IllegalStateException("Main thread not allowed to quit.");
  >      }
  > 
  >      synchronized (this) {
  >          if (mQuitting) {
  >              return;
  >          }
  >          mQuitting = true;
  > 
  >          if (safe) {
  >              removeAllFutureMessagesLocked();
  >          } else {
  >              removeAllMessagesLocked();
  >          }
  > 
  >          // We can assume mPtr != 0 because mQuitting was previously false.
  >          nativeWake(mPtr);
  >      }
  > }
  > ```
  >
  > 通过观察以上源码我们可以发现:
  >
  > 当我们调用Looper的quit方法时，实际上执行了MessageQueue中的removeAllMessagesLocked方法，该方法的作用是把MessageQueue消息池中所有的消息全部清空，无论是延迟消息（延迟消息是指通过sendMessageDelayed或通过postDelayed等方法发送的需要延迟执行的消息）还是非延迟消息。
  >
  > 当我们调用Looper的quitSafely方法时，实际上执行了MessageQueue中的removeAllFutureMessagesLocked方法，通过名字就可以看出，该方法只会清空MessageQueue消息池中所有的延迟消息，并将消息池中所有的非延迟消息派发出去让Handler去处理，quitSafely相比于quit方法安全之处在于清空消息之前会派发所有的非延迟消息。
  >
  > 无论是调用了quit方法还是quitSafely方法只会，Looper就不再接收新的消息。即在调用了Looper的quit或quitSafely方法之后，消息循环就终结了，这时候再通过Handler调用sendMessage或post等方法发送消息时均返回false，表示消息没有成功放入消息队列MessageQueue中，因为消息队列已经退出了。
  >
  > 需要注意的是Looper的quit方法从API Level 1就存在了，但是Looper的quitSafely方法从API Level 18才添加进来。

- Handler 如何与 Looper 关联的

  > 在Handler new出来的时候，就要确定好对应的Looper，即主要是要确定后面的message需要发送到哪个messageQueue。
  >
  > Handler的构造方法主要分两种类型，一种是直接
  >
  > 1. Handler(Looper looper)：直接设置looper为成员变量mLooper，这种就直接和设置的looper关联，消息方式到对应的messageQueue
  >
  > 2. Handler()、Handler(Callback callback)等，都是未设置looper，则会在构造方法中通过ThreadLocal去获取，即获取当前线程，然后获取线程中的ThreadLocalMap对象去获取looper对象，如果为空，则会抛出异常，提示先调用Looper.prepare()，
  >
  >    如果获取成功，则设置到Handler的成员变量mLooper

- Looper 如何与 Thread 关联的

  > ThreadLocal、ThreadLocalMap

- Looper.loop()源码

  > 开启死循环从MessageQueue.next中获取消息，然后调用msg.target.dispatchMessage(msg)处理消息，如果获取到的消息为空，则退出循环。消息为空只会在looper.quit方法调用后才会出现。当MessageQueue中获取的当前消息时间未到，或者消息队列为空时，会阻塞。然后等待唤醒获取到message。

- MessageQueue的enqueueMessage()方法如何进行线程同步的

  > 通过锁synchronized(this)，锁的是MessageQueue对象，故不管是在enqueueMessage存消息，还是next取消息时，只要有线程在操作，都要等待。（next时也会加synchronized(this)）

- MessageQueue的next()方法内部原理

  > ```java
  > Message next() {
  >      // Return here if the message loop has already quit and been disposed.
  >      // This can happen if the application tries to restart a looper after quit
  >      // which is not supported.
  >      final long ptr = mPtr;
  >      if (ptr == 0) {
  >          return null;
  >      }
  > 
  >      int pendingIdleHandlerCount = -1; // -1 only during first iteration
  >      int nextPollTimeoutMillis = 0;
  >      for (;;) {
  >          if (nextPollTimeoutMillis != 0) {
  >              Binder.flushPendingCommands();
  >          }
  > 
  >          //没有消息，或者延时消息时间未到，则一直阻塞等待，底层是调用linux的epoll.wait等待
  >          nativePollOnce(ptr, nextPollTimeoutMillis);
  > 
  >          synchronized (this) {
  >              // Try to retrieve the next message.  Return if found.
  >              final long now = SystemClock.uptimeMillis();
  >              Message prevMsg = null;
  >              Message msg = mMessages;
  >              //如果当前消息是消息屏障，即消息屏障的区别是msg.target == null，则去遍历寻找最近的异步消息
  >              if (msg != null && msg.target == null) {
  >                  // Stalled by a barrier.  Find the next asynchronous message in the queue.
  >                  do {
  >                      prevMsg = msg;
  >                      msg = msg.next;
  >                  } while (msg != null && !msg.isAsynchronous()); //判断是否为异步消息
  >              }
  >              //如果找到了消息
  >              if (msg != null) {
  >                  //消息延时时间未到，设置阻塞时间nextPollTimeoutMillis
  >                  if (now < msg.when) {
  >                      // Next message is not ready.  Set a timeout to wake up when it is ready.
  >                      nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
  >                  } else {
  >                      // Got a message.
  >                      mBlocked = false;
  >                      if (prevMsg != null) {
  >                          prevMsg.next = msg.next;
  >                      } else {
  >                          mMessages = msg.next;
  >                      }
  >                      msg.next = null;
  >                      if (DEBUG) Log.v(TAG, "Returning message: " + msg);
  >                      msg.markInUse();
  >                      return msg;
  >                  }
  >              } else {
  >                  //没找到消息，则设置需要阻塞标志
  >                  // No more messages.
  >                  nextPollTimeoutMillis = -1;
  >              }
  > 
  >              // Process the quit message now that all pending messages have been handled.
  >              //Looper.quit->messageQueue.quit设置mQuitting=true，则返回null消息，Looper.loop中接收到会结束死循环
  >              if (mQuitting) {
  >                  dispose();
  >                  return null;
  >              }
  > 
  >              //如果当前无消息处理，或者需要等待延时，则处理空闲队列
  >              // If first time idle, then get the number of idlers to run.
  >              // Idle handles only run if the queue is empty or if the first message
  >              // in the queue (possibly a barrier) is due to be handled in the future.
  >              if (pendingIdleHandlerCount < 0
  >                      && (mMessages == null || now < mMessages.when)) {
  >                  pendingIdleHandlerCount = mIdleHandlers.size();
  >              }
  >              if (pendingIdleHandlerCount <= 0) {
  >                  // No idle handlers to run.  Loop and wait some more.
  >                  mBlocked = true;
  >                  continue;
  >              }
  > 
  >              if (mPendingIdleHandlers == null) {
  >                  mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
  >              }
  >              mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
  >          }
  > 
  >          // Run the idle handlers.
  >          // We only ever reach this code block during the first iteration.
  >          for (int i = 0; i < pendingIdleHandlerCount; i++) {
  >              final IdleHandler idler = mPendingIdleHandlers[i];
  >              mPendingIdleHandlers[i] = null; // release the reference to the handler
  > 
  >              boolean keep = false;
  >              try {
  >                  keep = idler.queueIdle();
  >              } catch (Throwable t) {
  >                  Log.wtf(TAG, "IdleHandler threw exception", t);
  >              }
  > 
  >              if (!keep) {
  >                  synchronized (this) {
  >                      mIdleHandlers.remove(idler);
  >                  }
  >              }
  >          }
  > 
  >          // Reset the idle handler count to 0 so we do not run them again.
  >          pendingIdleHandlerCount = 0;
  > 
  >          // While calling an idle handler, a new message could have been delivered
  >          // so go back and look again for a pending message without waiting.
  >          nextPollTimeoutMillis = 0;
  >      }
  >  }
  > ```
  >
  > 

- 子线程中是否可以用MainLooper去创建Handler，Looper和Handler是否一定处于一个线程

  > 可以的，不一定
  >
  > ```java
  > thread{
  > Handler(Looper.getMainLooper()).sendMessage(Message())
  > }
  > 
  > public Handler(@NonNull Looper looper) {
  > this(looper, null, false);
  > }
  > public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
  >      mLooper = looper;
  >      mQueue = looper.mQueue;
  >      mCallback = callback;
  >      mAsynchronous = async;
  > }
  > ```

- ANR和Handler的联系

  > 任何事物都是有handler消息处理的，anr的出现其实就是该消息在主线程中处理的时间长过时长。
  >
  > Handler是线程间通讯的机制，Android中，网络访问、文件处理等耗时操作必须放到子线程中去执行，否则将会造成ANR异常。
  >
  > ANR异常：Application Not Response 应用程序无响应
  >
  > 产生ANR异常的原因：在主线程执行了耗时操作，对Activity来说，主线程阻塞5秒将造成ANR异常，对BroadcastReceiver来说，主线程阻塞10秒将会造成ANR异常。
  >
  > 解决ANR异常的方法：耗时操作都在子线程中去执行
  >
  > 但是，Android不允许在子线程去修改UI，可我们又有在子线程去修改UI的需求，因此需要借助Handler。