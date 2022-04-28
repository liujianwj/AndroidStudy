## OOM是什么

​     OOM（OutOfMemoryError）内存溢出错误，在常见的 Crash疑难排行榜上，OOM绝对可以名列前茅并且经久不衰。因为它发生时的Crash堆栈信息往往不是导致问题的根本原因，而只是压死骆驼的最后一根稻草。

#### 发生OOM的条件

- Android 2.x系统GC LOG中的dalvik allocated + external allocated +  新分配的大小 >= getMemoryClass() 值的时候就会发生OOM。例如，假如有这么一段 Dalvik输出的GL LOG: GC_FOR_MALLOC free 2K, 13% free 32586K/37455K, external 8989K/10356K, paused 20ms，那么32586+8989+(新分配23975)=65550>64M时，就会发生OOM。

- Android 4.x系统 Android 4.x的系统废除了external的计数器，类似bitmap的分配改到dalvik的java heap中申请，只要allocated + 新分配的内存 >= getMemoryClass()的时候就会发生OOM。

#### OOM原因分类

- Java堆内存溢出
- 无足够连续内存空间
- FD数量超出限制
- 线程数量超出限制
- 虚拟内存不足

## Android 内存分析命令介绍

常用的内存调优分析命令：

1. dumpsys meminfo

   适合场景：查看进程的oom adj，或者dalvik/native等区域内存情况，或者某个进程或apk的内存情况，功能非常强大。

2. procrank

   适合场景：查看进程的VSS/RSS/PSS/USS各个内存指标。

3. cat /proc/meminfo

   适合场景：查看系统的详尽内存信息，包含内核情况。

4. free

   适合场景：只查看系统的可用内存。

5. showmap

   适合场景：查看进程的虚拟地址空间的内存分配情况。

6. vmstat

   适合场景：周期性的打印出进程运行队列、系统切换、 cpu时间占比等情况。

## Android内存泄漏分析工具

#### MAT（Memory Analyzer Tool）

#### Android Studio Memory-profiler

https://developer.android.com/studio/profile/memory-profiler#performance

#### LeakCanary

https://github.com/square/leakcanary

## 内存泄漏常见场景及解决方案

#### 1. 资源性对象未关闭

   对于资源性对象不再使用时，应该立即调用它的close()函数，将其关闭，然后再置为null。例如Bitmap等资源未关闭会造成内存泄漏，此时我们应在Activity销毁时及时关闭。

#### 2. **注册对象未注销**

   例如BroadcastReceiver、EventBus等造成的内存泄漏，应该在Activity销毁时及时注销。

#### 3. 类的静态变量持有大数据对象

   尽量避免静态变量存储数据，尤其是大数据，可以使用数据库存储。

#### 4. 单例造成的内存泄漏

   优先使用Application的Context，如需使用Activity的Context时，可以在传入Context时用弱引用进行封装，在使用的时候从弱引用中获取Context。

#### 5. 非静态内部类的静态实例

   该实例的生命周期和应用一样长，这就导致该静态实例一直持有该Activity的引用，Activity的内存资源不能正常回收。此时，我们可以将该内部类设为静态内部类或将该内部类抽取出来封装成一个单例，如果需要使用Context，尽量使用Application Context，如果需要使用Activity Context，就记得用完后置空让GC可以回收，否则还是会内存泄漏。

#### 6.  Handler临时性内存泄漏

   Message发出之后存储在MessageQueue中，在Message中存在一个target，它是Handler的一个引用，Message在Queue中存在的时间过长，就会导致Handler无法被回收。如果Handler是非静态的，则会导Activity或者Service不会被回收。并且消息队列是在一个Looper线程中不断地轮询处理消息，当这个Activity退出时，消息队列中还有未处理的消息或者正在处理的消息，并且消息队列中的Message持有Handler实例的引用，Handler又持有Activity的引用，所以导致该Activity的内存资源无法及时回收，引发内存泄漏。

   解决方案如下所示：

   1. 使用一个静态Handler内部类，然后对Handler持有的对象（一般是Activity）使用弱引用，这样在回收时，也可以回收Handler持有的对象。

   2. 在Activity的Destroy或者Stop时，应该移除消息队列中的消息，避免Looper线程的消息队列中有待处理的消息需要处理。

   需要注意的是，AsyncTask内部也是Handler机制，同样存在内存泄漏风险，但其一般是临时性的。对于类似AsyncTask或是线程造成的内存泄漏，我们也可以将AsyncTask和Runnable类独立出来或者使用静态内部类。

#### 7. 容器中的对象没清除造成的内存泄漏

   在退出程序之前，将集合里的东西clear，然后置为null，再退出程序

#### 8. WebView内存泄漏

   WebView都存在内存泄漏的问题，在应用中只要使用一次WebView，内存就不会被释放掉。我们可以为WebView开启一个独立的进程，使用AIDL与应用的主进程进行通信，WebView所在的进程可以根据业务的需要选择合适的时机进行销毁，达到正常释放内存的目的。

#### 9. 使用ListView时造成的内存泄漏

   在构造Adapter时，使用缓存的convertView。





Java: Memory from objects allocated from Java or Kotlin code.【Java内存区域，主要是对象和Bitmap】

Native: Memory from objects allocated from C or C++ code.【C相关】

Stack: Memory used by both native and Java stacks in your app. This usually relates to how many threads your app is running.【这是一些方法栈和线程占用的区域】

Code: 就是写代码库.so等等；

Other: Memory used by your app that the system isn't sure how to categorize.【系统也不确定是个啥】

**Graphics: Memory used for graphics buffer queues to display pixels to the screen, including GL surfaces, GL textures, and so on. (Note that this is memory shared with the CPU, not dedicated GPU memory.)【像素图像图形渲染我的理解，这是之前我们没见过的】**



Memory Profile 

![image-20220323163411504](/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220323163411504.png)

Java表示Java代码或Kotlin代码分配的内存；

Native表示C或C++代码分配的内存(即使App没有native层，调用framework代码时，也有可能触发分配native内存)；

Graphics表示图像相关缓存队列占用的内存；

Stack表示native和java占用的栈内存；

Code表示代码、资源文件、库文件等占用的内存；

Others表示无法明确分类的内存；

Allocated表示Java或Kotlin分配对象的数量(Android8.0以下时，仅统计Memory Profiler启动后，进程再分配的对象数量； 
8.0以上时，由于系统内置了统计工具，Memory Profiler可以得到整个app启动后分配对象的数量)。