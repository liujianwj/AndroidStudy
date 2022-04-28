### 切换线程工作原理

主要成员：Handler、Message、Looper、MessageQueue、Thread、ThreadLocal、ThreadLocalMap

- Handler：负责发送消息，创建的时候就保存了对应的Looper、MessageQueue；

- Message：消息载体，存储数据、执行体Runnable，Handler引用target，消息类型等信息

- Looper：负责处理MessageQueue创建，消息读取和退出，主要方法初始化prepare()，loop()开始执行循环读数据。

  以及主要成员变量static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

- MessageQueue：消息队列，优先级队列，单链表结构，重要方法enqueueMessage，next。

- Thread：线程类，主要关心ThreadLocalMap threadLocals变量，每个线程都有个threadLocals变量

- ThreadLocal：用于操作内部类ThreadLocalMap

- ThreadLocalMap：key-value存储，在整个环节中，key->this->Looper.sThreadLocal，value->new Looper()，sThreadLocal是static final全局唯一，但是ThreadLocalMap在每个线程中都有一个，所以可以保证一个线程只有一个Looper。

子线程、handler、主线程 其实构成了线程模型中的经典问题 生产者-消费者模型。 生产者-消费者模型：生产者和消费者在同一时间段内共用同一个存储空间，生产者往存储空间中添加数据，消费者从存储空间中取走数据。

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220209102618203.png" alt="image-20220209102618203" style="zoom:50%;" />

其中就有几个细节问题：

1. 一个线程中有几个Looper、几个MessageQueue、几个Handler，怎么保障的？

   > 一个线程只有一个Looper、Looper中只有一个MessageQueue、可以有无数个Handler。
   >
   > 我们要使用Handler，第一步是创建Looper，而Looper的创建是通过Looper.prepare()
   >
   > ```java
   > class Looper{
   >   static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();    
   > 
   >   private static void prepare(boolean quitAllowed) {
   >         //保障一个线程只有一个Looper
   >         if (sThreadLocal.get() != null) {
   >             throw new RuntimeException("Only one Looper may be created per thread");
   >         }
   >         sThreadLocal.set(new Looper(quitAllowed));
   >   }
   >   
   >   private Looper(boolean quitAllowed) {
   >         mQueue = new MessageQueue(quitAllowed);
   >         mThread = Thread.currentThread();
   >    }
   > }
   > ```
   >
   > 这里会先从sThreadLocal中去获取Looper，如果能获取到说明已经创建过了，则直接抛出异常，否则新建Looper，且在Looper的构造方法中新建MessageQueue，然后将Looper放入sThreadLocal。
   >
   > 接下来我们看下sThreadLocal是怎么做到保障当前线程只有一个Looper的：
   >
   > ```java
   > class ThreadLocal{
   >   //取数据
   >   public T get() {
   >         Thread t = Thread.currentThread();
   >         ThreadLocalMap map = getMap(t);
   >         if (map != null) {
   >             ThreadLocalMap.Entry e = map.getEntry(this);
   >             if (e != null) {
   >                 @SuppressWarnings("unchecked")
   >                 T result = (T)e.value;
   >                 return result;
   >             }
   >         }
   >         return setInitialValue();
   >  }
   >   //设置数据
   >   public void set(T value) {
   >         Thread t = Thread.currentThread();
   >         ThreadLocalMap map = getMap(t);
   >         if (map != null)
   >             map.set(this, value);
   >         else
   >             createMap(t, value);
   >     }
   >   
   >   ThreadLocalMap getMap(Thread t) {
   >         return t.threadLocals;
   >     }
   > }
   > 
   > class Thread{
   >   ThreadLocal.ThreadLocalMap threadLocals = null;
   > }
   > ```
   >
   > 根据源码我们可知，其实Looper是保存到了ThreadLocal的内部类ThreadLocalMap中，而每个线程都有个ThreadLocal.ThreadLocalMap变量，在存储Looper时，key=this=static final ThreadLocal<Looper> sThreadLocal，value=Looper，可知对应关系是：
   >
   > ```java
   > Thread1 -> ThreadLocalMap map1 = new ThreadLocalMap() -> map1.put(sThreadLocal, looper1)
   > Thread1 -> ThreadLocalMap map2 = new ThreadLocalMap() -> map1.put(sThreadLocal, looper2)
   > ```
   >
   > 这样就保障了每个线程都有自己的唯一的一份Looper。

2. 当消息队列为空时，怎么阻塞处理？同步屏障怎么处理？怎么退出？

   > 这些问题都是在Looper.loop()中，调用messageQueue.next()方法中处理，我们分析下next()即可：
   >
   > ```java
   > class MessageQueue{
   >   Message next() {
   >         // Return here if the message loop has already quit and been disposed.
   >         // This can happen if the application tries to restart a looper after quit
   >         // which is not supported.
   >         final long ptr = mPtr;
   >         if (ptr == 0) {
   >             return null;
   >         }
   > 
   >         int pendingIdleHandlerCount = -1; // -1 only during first iteration
   >         int nextPollTimeoutMillis = 0;
   >         for (;;) {
   >             if (nextPollTimeoutMillis != 0) {
   >                 Binder.flushPendingCommands();
   >             }
   >             //当消息队列为空，或者延时消息时间未到时，调用native方法，调用底层Linux的epoll.wait方法阻塞，只要管道有写入，则会唤醒，在enqueueMessage中会有nativeWake写入操作
   >             nativePollOnce(ptr, nextPollTimeoutMillis);
   > 
   >             synchronized (this) {
   >                 // Try to retrieve the next message.  Return if found.
   >                 final long now = SystemClock.uptimeMillis();
   >                 Message prevMsg = null;
   >                 Message msg = mMessages;
   >                 //处理消息屏障，屏障特别msg.target == null
   >                 if (msg != null && msg.target == null) {
   >                     // Stalled by a barrier.  Find the next asynchronous message in the queue.
   >                     do {
   >                         prevMsg = msg;
   >                         msg = msg.next;
   >                     } while (msg != null && !msg.isAsynchronous()); //判断是否找到异步消息
   >                 }
   >                 if (msg != null) {
   >                     if (now < msg.when) {
   >                         // Next message is not ready.  Set a timeout to wake up when it is ready.
   >                         nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
   >                     } else {
   >                         // Got a message.
   >                         mBlocked = false;
   >                         if (prevMsg != null) {
   >                             prevMsg.next = msg.next;
   >                         } else {
   >                             mMessages = msg.next;
   >                         }
   >                         msg.next = null;
   >                         if (DEBUG) Log.v(TAG, "Returning message: " + msg);
   >                         msg.markInUse();
   >                         return msg;
   >                     }
   >                 } else {
   >                     // No more messages.
   >                     nextPollTimeoutMillis = -1;
   >                 }
   > 
   >                 // Process the quit message now that all pending messages have been handled.
   >                 if (mQuitting) {
   >                     dispose();
   >                     return null;
   >                 }
   > 
   >                 // If first time idle, then get the number of idlers to run.
   >                 // Idle handles only run if the queue is empty or if the first message
   >                 // in the queue (possibly a barrier) is due to be handled in the future.
   >                 if (pendingIdleHandlerCount < 0
   >                         && (mMessages == null || now < mMessages.when)) {
   >                     pendingIdleHandlerCount = mIdleHandlers.size();
   >                 }
   >                 if (pendingIdleHandlerCount <= 0) {
   >                     // No idle handlers to run.  Loop and wait some more.
   >                     mBlocked = true;
   >                     continue;
   >                 }
   > 
   >                 if (mPendingIdleHandlers == null) {
   >                     mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
   >                 }
   >                 mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
   >             }
   > 
   >             // Run the idle handlers.
   >             // We only ever reach this code block during the first iteration.
   >             for (int i = 0; i < pendingIdleHandlerCount; i++) {
   >                 final IdleHandler idler = mPendingIdleHandlers[i];
   >                 mPendingIdleHandlers[i] = null; // release the reference to the handler
   > 
   >                 boolean keep = false;
   >                 try {
   >                     keep = idler.queueIdle();
   >                 } catch (Throwable t) {
   >                     Log.wtf(TAG, "IdleHandler threw exception", t);
   >                 }
   > 
   >                 if (!keep) {
   >                     synchronized (this) {
   >                         mIdleHandlers.remove(idler);
   >                     }
   >                 }
   >             }
   > 
   >             // Reset the idle handler count to 0 so we do not run them again.
   >             pendingIdleHandlerCount = 0;
   > 
   >             // While calling an idle handler, a new message could have been delivered
   >             // so go back and look again for a pending message without waiting.
   >             nextPollTimeoutMillis = 0;
   >         }
   >     }
   > }
   > ```
   >
   > 

3. 队列怎么维护执行顺序？

   ```java
   boolean enqueueMessage(Message msg, long when) {
           if (msg.target == null) {
               throw new IllegalArgumentException("Message must have a target.");
           }
           if (msg.isInUse()) {
               throw new IllegalStateException(msg + " This message is already in use.");
           }
   
           synchronized (this) {
               if (mQuitting) {
                   IllegalStateException e = new IllegalStateException(
                           msg.target + " sending message to a Handler on a dead thread");
                   Log.w(TAG, e.getMessage(), e);
                   msg.recycle();
                   return false;
               }
   
               msg.markInUse();
               msg.when = when;
               Message p = mMessages;
               boolean needWake;
               //根据时间大小插入，发送消息的时候都是
               //sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis)
               //SystemClock.uptimeMillis()为开机到当前的时间，所以如果不设置延时，那先发送的消息会插入到前面，先执行
               if (p == null || when == 0 || when < p.when) {
                   // New head, wake up the event queue if blocked.
                   msg.next = p;
                   mMessages = msg;
                   needWake = mBlocked;
               } else {
                   // Inserted within the middle of the queue.  Usually we don't have to wake
                   // up the event queue unless there is a barrier at the head of the queue
                   // and the message is the earliest asynchronous message in the queue.
                   needWake = mBlocked && p.target == null && msg.isAsynchronous();
                   Message prev;
                   for (;;) {
                       prev = p;
                       p = p.next;
                       if (p == null || when < p.when) {
                           break;
                       }
                       if (needWake && p.isAsynchronous()) {
                           needWake = false;
                       }
                   }
                   msg.next = p; // invariant: p == prev.next
                   prev.next = msg;
               }
   
               // We can assume mPtr != 0 because mQuitting is false.
               //唤醒next()中的等待方法
               if (needWake) {
                   nativeWake(mPtr);
               }
           }
           return true;
       }
   ```

   

### 同步屏障

同步屏障的概念，在Android开发中非常容易被人忽略，因为这个概念在我们普通的开发中太少见了，很容易被忽略。

大家经过上面的学习应该知道，线程的消息都是放到同一个MessageQueue里面，取消息的时候是互斥取消息，而且只能从头部取消息，**而添加消息是按照消息的执行的先后顺序进行的排序**，那么问题来了，同一个时间范围内的消息，如果它是需要立刻执行的，那我们怎么办，按照常规的办法，我们需要等到队列轮询到我自己的时候才能执行哦，那岂不是黄花菜都凉了。所以，我们需要给紧急需要执行的消息一个绿色通道，这个绿色通道就是同步屏障的概念。

屏障的意思即为阻碍，顾名思义，**同步屏障就是阻碍同步消息，只让异步消息通过**。如何开启同步屏障呢？如下而已：

```java
MessageQueue#postSyncBarrier()
  
public int postSyncBarrier() {
  return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
  // Enqueue a new sync barrier token.
  // We don't need to wake the queue because the purpose of a barrier is to stall it.
  synchronized (this) {
    final int token = mNextBarrierToken++;
    //从消息池中获取Message
    final Message msg = Message.obtain();
    msg.markInUse();
    //就是这里！！！初始化Message对象的时候，并没有给target赋值，因此 target==null
    msg.when = when;
    msg.arg1 = token;

    Message prev = null;
    Message p = mMessages;
    if (when != 0) {
      while (p != null && p.when <= when) {
        //如果开启同步屏障的时间（假设记为T）T不为0，且当前的同步消息里有时间小于T，则prev也不为null
        prev = p;
        p = p.next;
      }
    }
    //根据prev是不是为null，将 msg 按照时间顺序插入到 消息队列（链表）的合适位置
    if (prev != null) { // invariant: p == prev.next
      msg.next = p;
      prev.next = msg;
    } else {
      msg.next = p;
      mMessages = msg;
    }
    return token;
  }
}
```

可以看到，Message 对象初始化的时候并没有给 target 赋值，因此， target == null 的 来源就找到了。上面消息的插入也做了相应的注释。这样，一条 target == null 的消息就进入了消息队列。那么，开启同步屏障后，所谓的异步消息又是如何被处理的呢？

如果对消息机制有所了解的话，应该知道消息的最终处理是在消息轮询器 Looper#loop() 中，而 loop() 循环中会 调用MessageQueue#next() 从消息队列中获取消息。

**同步消息的应用场景**

似乎在日常的应用开发中，很少会用到同步屏障。那么，同步屏障在系统源码中有哪些使用场景呢？Android 系统中的 UI 更新相关的消息即为异步消息，需要优先处理。

比如，在 View 更新时，draw、requestLayout、invalidate 等很多地方都调用了ViewRootImpl#scheduleTraversals() ，如下：

```java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
          //开启同步屏障
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
          //发送异步消息
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```

postCallback() 最终走到了 Choreographer#postCallbackDelayedInternal() ：、

```java
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        if (DEBUG_FRAMES) {
            Log.d(TAG, "PostCallback: type=" + callbackType
                    + ", action=" + action + ", token=" + token
                    + ", delayMillis=" + delayMillis);
        }

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true); //异步消息
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
```

这里就开启了同步屏障，并发送异步消息，由于 UI 更新相关的消息是优先级最高的，这样系统就会优先处理这些异步消息。

最后，当要移除同步屏障的时候需要调用 ViewRootImpl#unscheduleTraversals() 。

```java
    void unscheduleTraversals() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
          //移除同步屏障
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
            mChoreographer.removeCallbacks(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        }
    }
```

小结

同步屏障的设置可以方便地处理那些优先级较高的异步消息。当我们调用Handler.getLooper().getQueue().postSyncBarrier() 并设置消息的 setAsynchronous(true) 时，target 即 为 null ，也就开启了同步屏障。当在消息轮询器 Looper 在 loop() 中循环处理消息时，如若开启了同步屏障，会优先处理其中的异步消息，而阻碍同步消息。



### HandlerThread

> 继承Thread，在run 中自己就完成了Looper.perpare()，Looper.loop()操作，，用户只需创建HandlerThread，然后start启动，，后续通过获取其looper，利用handler发送消息，这样消息处理都在子线程中执行了。
>
> 优势：使用简单，线程安全：注意其run()、getLooper()中都加了synchronized(this)
>
> ```java
> public void run() {
>         mTid = Process.myTid();
>         Looper.prepare();
>         synchronized (this) {
>             mLooper = Looper.myLooper();
>             //mLooper创建成功，唤醒，
>             notifyAll();
>         }
>         Process.setThreadPriority(mPriority);
>         onLooperPrepared();
>         Looper.loop();
>         mTid = -1;
>  }
> 
> public Looper getLooper() {
>         //先判断线程是否已经启动
>         if (!isAlive()) {
>             return null;
>         }
>         
>         // If the thread has been started, wait until the looper has been created.
>         synchronized (this) {
>             //如果此时mLooper还没创建成功，则wait等待，当有notifyAll调用后，wait处不会马上获取执行权力，只是具备了争夺锁的权力，不在等待
>             while (isAlive() && mLooper == null) {
>                 try {
>                     wait();
>                 } catch (InterruptedException e) {
>                 }
>             }
>         }
>         return mLooper;
> }
> ```
>
> 使用场景：我们知道，HandlerThread 所做的就是在新开的子线程中创建了 Looper，那它的使用场景就是 Thread + Looper 使用场景的结合，即：**在子线程中执行耗时的、可能有多个任务的操作**。比如说多个网络请求操作，或者多文件 I/O 等等。
>
> 使用 HandlerThread 的典型例子就是 `IntentService`

### IntentService

IntentService是继承并处理异步请求的一个类，在IntentService内有一个工作线程来处理耗时操作，启动IntentService的方式和启动传统的Service一样，同时，当任务执行完后，IntentService会自动停止，而不需要我们手动去控制或stopSelf()。另外，可以启动IntentService多次，而每一个耗时操作会以工作队列的方式在IntentService的onHandleIntent回调方法中执行，并且，每次只会执行一个工作线程，执行完第一个再执行第二个，以此类推。

场景：在Android开发中，我们或许会碰到这么一种业务需求，一项任务分成几个子任务，子任务按顺序先后执行，子任务全部执行完后，这项任务才算成功。那么，利用几个子线程顺序执行是可以达到这个目的的，但是每个线程必须去手动控制，而且得在一个子线程执行完后，再开启另一个子线程。或者，全部放到一个线程中让其顺序执行。这样都可以做到，但是，如果这是一个后台任务，就得放到Service里面，由于Service和Activity是同级的，所以，要执行耗时任务，就得在Service里面开子线程来执行。那么，有没有一种简单的方法来处理这个过程呢，答案就是IntentService。

