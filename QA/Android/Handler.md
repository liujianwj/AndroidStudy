1.Handler的实现原理

2.子线程中能不能直接new一个Handler,为什么主线程可以

  主线程的Looper第一次调用loop方法,什么时候,哪个类

3.Handler导致的内存泄露原因及其解决方案

4.一个线程可以有几个Handler,几个Looper,几个MessageQueue对象

5.Message对象创建的方式有哪些 & 区别？

  Message.obtain()怎么维护消息池的

6.Handler 有哪些发送消息的方法

7.Handler的post与sendMessage的区别和应用场景

8.handler postDealy后消息队列有什么变化，假设先 postDelay 10s, 再postDelay 1s, 怎么处理这2条消息

9.MessageQueue是什么数据结构

10.Handler怎么做到的一个线程对应一个Looper，如何保证只有一个MessageQueue

​    ThreadLocal在Handler机制中的作用

11.HandlerThread是什么 & 好处 &原理 & 使用场景

12.IdleHandler及其使用场景

13.消息屏障,同步屏障机制

14.子线程能不能更新UI

15.为什么Android系统不建议子线程访问UI

16.Android中为什么主线程不会因为Looper.loop()里的死循环卡死

MessageQueue#next 在没有消息的时候会阻塞，如何恢复？

17.Handler消息机制中，一个looper是如何区分多个Handler的

当Activity有多个Handler的时候，怎么样区分当前消息由哪个Handler处理

处理message的时候怎么知道是去哪个callback处理的

18.Looper.quit/quitSafely的区别

19.通过Handler如何实现线程的切换

20.Handler 如何与 Looper 关联的

21.Looper 如何与 Thread 关联的

22.Looper.loop()源码

23.MessageQueue的enqueueMessage()方法如何进行线程同步的

24.MessageQueue的next()方法内部原理

25.子线程中是否可以用MainLooper去创建Handler，Looper和Handler是否一定处于一个线程

26.ANR和Handler的联系


