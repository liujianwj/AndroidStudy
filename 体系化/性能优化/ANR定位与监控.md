## ANR概念

ANR(Application Not Responding) 应用程序无响应，如果你的应用程序在UI线程被阻塞太长时间，就会出现ANR，通常出现ANR，系统会弹出一个提示框，让用户知道该程序正在被阻塞，是否继续等待或者关闭。

## ANR类型

- InputDispatchTimeOut（常见）

  输入时间分发超时5s，包括按键和触摸事件

  logcat日志关键字：Input event dispatching timed out

- Broadcast Timeout

  前台Broadcast：onReceiver在10s内没有处理完成发生ANR。

  后台Broadcast：onReceiver在60s内没有处理完成发生ANR。

  logcat日志关键字：Timeout of broadcast BroadcastRecord

- Service Timeout

  前台Service：onCreate，onStart，onBind等生命周期在20s内没有处理完成发生ANR。

  后台Service：onCreate，onStart，onBind等生命周期在200s内没有处理完成发生ANR。

  logcat日志关键字：Timeout executing service

- ContentProvider Timeout

  ContentProvider在10s内没有处理完成发生ANR。

  logcat日志关键字：Timeout publishing content providers

##  为什么出现ANR

1.  主线程频繁进行耗时的IO操作，如数据库读写
2. 多线程操作的死锁，主线程被block，held by
3. 主线程被Binder对端block
4. System Server 中WatchDog出现ANR
5. Service binder的连接达到上线无法和system server通信
6. 系统资源已耗尽（管道、CPU、IO）

## 如何避免ANR发生

1. 主线程尽量只做UI相关的操作，避免耗时操作，比如过度复杂的UI绘制，网络操作，文件IO操作；

2. 避免主线程跟工作线程发生锁的竞争，减少系统耗时binder的调用，请谨慎使用sharepreference，注意主线程执行provider query 操作

   > 总之,尽可能减少主线程的负载，让其空闲待命，以期可随时响应用户的操作

##  ANR分析技巧

1. 通过logcat日志，traces文件确认ANR发生时间点
2. traces文件和CPU使用率
3. traces文件地址/data/anr/traces.txt
4. 主线程状态
5. 其他线程状态

traces文件关键信息：

main：main标识是主线程，如果是线程，那么命名成“Thread-X”的格式,x表示线程id,逐步递增。
prio：线程优先级,默认是5
tid：tid不是线程的id，是线程唯一标识ID
group：是线程组名称
sCount：该线程被挂起的次数
dsCount：是线程被调试器挂起的次数
obj：对象地址
self：该线程Native的地址
sysTid：是线程号(主线程的线程号和进程号相同)
nice：是线程的调度优先级
sched：分别标志了线程的调度策略和优先级
cgrp：调度归属组
handle：线程处理函数的地址。
state：是调度状态
schedstat：从 /proc/[pid]/task/[tid]/schedstat读出，三个值分别表示线程在cpu上执行的时间、线程的等待时间和线程执行的时间片长度，不支持这项信息的三个值都是0；
utm：是线程用户态下使用的时间值(单位是jiffies）
stm：是内核态下的调度时间值
core：是最后执行这个线程的cpu核的序号

## 线上监控方案

1. Watchdog

    系统中watchdog流程：

   ![watchdog](/Users/liujian/Documents/study/books/markdown/图片/watchdog.png)

   其实就是向主消息Handler队列中添加一个消息，检测一定时间看该消息是否被执行来判断是否有发生ANR，依据这个流程，我们可以自己写一个watchdog获取发生ANR时日志信息：

   ![watchdog2](/Users/liujian/Documents/study/books/markdown/图片/watchdog2.png)

   代码如下：

   ```java
   package com.example.anr;
   
   import android.annotation.TargetApi;
   import android.os.Build;
   import android.os.Debug;
   import android.os.Handler;
   import android.os.Looper;
   import android.os.Process;
   import android.os.SystemClock;
   import android.util.Log;
   
   
   public class ANRWatchDog extends Thread {
   
       private static final String TAG = "ANR";
       private int timeout = 5000;
       private boolean ignoreDebugger = true;
   
       static ANRWatchDog sWatchdog;
   
       private Handler mainHandler = new Handler(Looper.getMainLooper());
   
   
       private class ANRChecker implements Runnable {
   
           private boolean mCompleted;
           private long mStartTime;
           private long executeTime = SystemClock.uptimeMillis();
   
           @Override
           public void run() {
               synchronized (ANRWatchDog.this) {
                   mCompleted = true;
                   executeTime = SystemClock.uptimeMillis();
               }
           }
   
           void schedule() {
               mCompleted = false;
               mStartTime = SystemClock.uptimeMillis();
               mainHandler.postAtFrontOfQueue(this);
           }
   
           boolean isBlocked() {
               return !mCompleted || executeTime - mStartTime >= 5000;
           }
       }
   
       public interface ANRListener {
           void onAnrHappened(String stackTraceInfo);
       }
   
       private ANRChecker anrChecker = new ANRChecker();
   
       private ANRListener anrListener;
   
       public void addANRListener(ANRListener listener){
           this.anrListener = listener;
       }
   
       public static ANRWatchDog getInstance(){
           if(sWatchdog == null){
               sWatchdog = new ANRWatchDog();
           }
           return sWatchdog;
       }
   
       private ANRWatchDog(){
           super("ANR-WatchDog-Thread");
       }
   
       @TargetApi(Build.VERSION_CODES.JELLY_BEAN)
       @Override
       public void run() {
           Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND); // 设置为后台线程
           while(true){
               while (!isInterrupted()) {
                   synchronized (this) {
                       anrChecker.schedule();
                       long waitTime = timeout;
                       long start = SystemClock.uptimeMillis();
                       while (waitTime > 0) {
                           try {
                               wait(waitTime);
                           } catch (InterruptedException e) {
                               Log.w(TAG, e.toString());
                           }
                           waitTime = timeout - (SystemClock.uptimeMillis() - start);
                       }
                       if (!anrChecker.isBlocked()) {
                           continue;
                       }
                   }
                   if (!ignoreDebugger && Debug.isDebuggerConnected()) {
                       continue;
                   }
                   String stackTraceInfo = getStackTraceInfo();
                   if (anrListener != null) {
                       anrListener.onAnrHappened(stackTraceInfo);
                   }
               }
               anrListener = null;
           }
       }
   
       private String getStackTraceInfo() {
           StringBuilder stringBuilder = new StringBuilder();
           for (StackTraceElement stackTraceElement : 
                Looper.getMainLooper().getThread().getStackTrace()) {
               stringBuilder
                       .append(stackTraceElement.toString())
                       .append("\r\n");
           }
           return stringBuilder.toString();
       }
   }
   
   ```

   

2. FileOberver

   官方链接：[https://developer.android.google.cn/reference/android/os/FileObserver.html](https://link.jianshu.com/?t=https://developer.android.google.cn/reference/android/os/FileObserver.html)

   Android系统在此基础上封装了一个FileObserver类来方便使用Inotify机制。FileObserver是一个抽象类，需要定义子类实现该类的onEvent抽象方法，当被监控的文件或者目录发生变更事件时，将回调FileObserver的onEvent()函数来处理文件或目录的变更事件:

   ```java
   package com.example.anr;
   
   import android.os.FileObserver;
   import android.util.Log;
   
   import androidx.annotation.Nullable;
   
   public class ANRFileObserver extends FileObserver {
   
   
       public ANRFileObserver(String path) {//data/anr/
           super(path);
       }
   
       public ANRFileObserver(String path, int mask) {
           super(path, mask);
       }
   
       @Override
           public void onEvent(int event, @Nullable String path) {
               switch (event)
           {
               case FileObserver.ACCESS://文件被访问
                   Log.i("Zero", "ACCESS: " + path);
                   break;
               case FileObserver.ATTRIB://文件属性被修改，如 chmod、chown、touch 等
                   Log.i("Zero", "ATTRIB: " + path);
                   break;
               case FileObserver.CLOSE_NOWRITE://不可写文件被 close
                   Log.i("Zero", "CLOSE_NOWRITE: " + path);
                   break;
               case FileObserver.CLOSE_WRITE://可写文件被 close
                   Log.i("Zero", "CLOSE_WRITE: " + path);
                   break;
               case FileObserver.CREATE://创建新文件
                   Log.i("Zero", "CREATE: " + path);
                   break;
               case FileObserver.DELETE:// 文件被删除，如 rm
                   Log.i("Zero", "DELETE: " + path);
                   break;
               case FileObserver.DELETE_SELF:// 自删除，即一个可执行文件在执行时删除自己
                   Log.i("Zero", "DELETE_SELF: " + path);
                   break;
               case FileObserver.MODIFY://文件被修改
                   Log.i("Zero", "MODIFY: " + path);
                   break;
               case FileObserver.MOVE_SELF://自移动，即一个可执行文件在执行时移动自己
                   Log.i("Zero", "MOVE_SELF: " + path);
                   break;
               case FileObserver.MOVED_FROM://文件被移走，如 mv
                   Log.i("Zero", "MOVED_FROM: " + path);
                   break;
               case FileObserver.MOVED_TO://文件被移来，如 mv、cp
                   Log.i("Zero", "MOVED_TO: " + path);
                   break;
               case FileObserver.OPEN://文件被 open
                   Log.i("Zero", "OPEN: " + path);
                   break;
               default:
                   //CLOSE ： 文件被关闭，等同于(IN_CLOSE_WRITE | IN_CLOSE_NOWRITE)
                   //ALL_EVENTS ： 包括上面的所有事件
                   Log.i("Zero", "DEFAULT(" + event + "): " + path);
                   break;
           }
       }
   }
   
   ```

   

