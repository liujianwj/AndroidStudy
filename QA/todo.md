https://juejin.cn/post/6990332687063990286

http://www.xiangxueketang.cn/enjoy/removal/article_7?bd_vid=9470290798457506851

> 内容概要：包括 Android、Java、数据结构与算法、计算机网路四大模块、

> Android内含：Activity、Fragment、service、布局优化、AsyncTask相关、Android 事件分发机制、 Binder、Android 高级必备 ：AMS,WMS,PMS、Glide、 Android 组件化与插件化等面试题和技术栈！

> Java内含：HashMap、ArrayList、LinkedList 、Hashset 源码分析、内存模型、垃圾回收算法（JVM）、多线程、注解、反射、泛型、设计模式等面试题和技术栈！



### 一. Android相关

#### 1.Activity

- 说下Activity生命周期


 > onCreate、onStart、onResume、onPause、onStop、onDestory


- Activity A 启动另一个Activity B 会调用哪些方法？如果B是透明主题的又或则是个DialogActivity呢

> A启动B，B为正常界面：A.onPause->B.onCreate->B.onStart->B.onResume->A.onStop
>                     返回：B.onPause->A.reStart->A.onStart->A.onResume->B.onStop->B.Destory
>
> A启动B，B为透明主题：A.onPause->B.onCreate->B.onStart->B.onResume
>                        返回：B.onPause->A.onResume->B.onStop->B.Destory  
>
> A启动Dialog：无影响
>
> 锁屏：只会调用onPause，解锁：调用onResume
>
> 横竖屏切换：onPause -> onStop ->onSaveInstanceState -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume


- 说下onSaveInstanceState()方法的作用 ? 何时会被调用？onSaveInstanceState(),onRestoreInstanceState的调用时机？

  > Activity中为我们提供了onSaveInstanceState()方法，用于保证在系统回收Activity前被调用，以此来解决被回收数据丢失的问题，并通过onRestoreInstanceState()方法恢复数据。
  >    onSaveInstanceState()和onRestoreInstanceState()并不是成对出现，也就是说onSaveInstanceState()调用了并不能保证onRestoreInstanceState()方法调用，但onRestoreInstanceState()调用了则页面必然被系统回收了，onSaveInstanceState()肯定被调用了。
  >  1）onSaveInstanceState()肯定被执行时机
  >    1、当用户按下HOME键时
  >    2、从最近应用中选择运行其他的程序时
  >    3、按下电源按键（关闭屏幕显示）时
  >    4、从当前activity启动一个新的activity时
  >    5、屏幕方向切换时(竖屏切横屏或横屏切竖屏)
  >   如果用户主动销毁Activity，如：按下返回键，或者调用了finish()方法销毁activity，则onSaveInstanceState不会被调用
  > onPause  -> onStop ->onSaveInstanceState。
  >
  >  2）onRestoreInstanceState()调用时机
  >      当系统存在 “未经你允许” 销毁了你的的activity的时，即重建Activity时，onSaveInstanceState()方法会执行，onRestoreInstanceState()方法也会执行，否则onRestoreInstanceState()方法不会执行。
  >      比如第5种情况屏幕方向切换时，activity生命周期如下：
  > onPause -> onStop ->onSaveInstanceState -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume
  > 在这里onRestoreInstanceState被调用，是因为屏幕切换时原来的activity确实被系统回收了，又重新创建了一个新的activity

- Activity的启动流程

  > 两种情况：应用内启动Activity，桌面点击图标启动应用时启动Activity，第二种情况更为详细，以下分析第二种
  >
  > 1. 点击桌面应用，launcherAPP会调用startActivity->startActivityForResult->Instrumentation.execStartActivity->AMS.startActivity
  > 2. AMS接收到信息主要做以下事情：
  >    - 对参数intent的内容进行解析，保存到ResolveInfo对象中
  >    - 从传进来的参数得到调用者的进程信息，保存到ProcessRcord对象中，这里获取的就是Launcher应用程序的进程
  >    - 创建即将要启动的Activity的相关信息 保存到ActivityRecord对象
  >    - 根据intent的参数设置启动模式，并创建一个新的Task来启动这个Activity
  >    - 将创建的Task保存到ActivityManagerService
  >    - 将Launcher推入Paused状态    
  > 3. Launcher收到pause通知（ApplicationThread.schedulePauseActivity接收到通知，然后通过handler发送消息），然后执行performPauseActivity，调用onPause，通知AMS
  > 4. AMS接收到通知，通过socket通知zygote fork一个新进程，新进程中执行ActivityThread里的main方法
  > 5. ActivityThread调用attachApplication方法向AMS传递一个Application对象，后续通过这个对象进行通信
  > 6. AMS接收到对象后，向ActivityThread发送sheduleLaunchActivity，ApplicationThread接收到信息，通过handler发送消息，handler接收到消息后，执行performLaunchActivity->Instrumentation.newActivity（通过反射获取activity）->attach-> onCreate->onStart->onResume

- Activity的启动模式和使用场景

  > Activity的启动模式有四种：**standard、singleTop、singleTask和singleInstance**
  >
  > - **standard：标准模式**
  >
  >   标准模式下，只要启动一次Activity，系统就会在当前任务栈新建一个实例。正常的去打开一个新的页面，这种启动模式使用最多，最普通。
  >
  > - **singleTop：栈顶复用模式**
  >
  >   这种启动模式下，如果要启动的Activity已经处于栈的顶部，那么此时系统不会创建新的实例，而是直接打开此页面，同时它的onNewIntent()方法会被执行，我们可以通过Intent进行传值，而且它的onCreate()，onStart()方法不会被调用，因为它并没有发生任何变化。
  >
  >   特点：
  >   1、当前栈中已有该Activity的实例并且该实例位于栈顶时，不会创建实例，而是复用栈顶的实例，并且会将Intent对象传入，回调onNewInten()方法；
  >   2、当前栈中已有该Activity的实例但是该实例不在栈顶时，其行为和standard启动模式一样，依然会创建一个新的实例；
  >   3、当前栈中不存在该Activity的实例时，其行为同standard启动模式。
  >
  >   使用场景：
  >   这种模式应用场景的话，假如一个新闻客户端，在通知栏收到了3条推送，点击每一条推送会打开新闻的详情页，如果为默认的启动模式的话，点击一次打开一个页面，会打开三个详情页，这肯定是不合理的。如果启动模式设置为singleTop，当点击第一条推送后，新闻详情页已经处于栈顶，当我们第二条和第三条推送的时候，只需要通过Intent传入相应的内容即可，并不会重新打开新的页面，这样就可以避免重复打开页面了。
  >
  > - **singleTask：站内复用模式**
  >
  >   在这个模式下，如果栈中存在这个Activity的实例就会复用这个Activity，不管它是否位于栈顶，复用时，会将它上面的Activity全部出栈，因为singleTask本身自带clearTop这种功能。并且会回调该实例的onNewIntent()方法。其实这个过程还存在一个任务栈的匹配，因为这个模式启动时，会在自己需要的任务栈中寻找实例，这个任务栈就是通过taskAffinity属性指定。如果这个任务栈不存在，则会创建这个任务栈。不设置taskAffinity属性的话，默认为应用的包名。
  >
  >   特点：
  >   在复用的时候，首先会根据taskAffinity去找对应的任务栈：
  >   1、如果不存在指定的任务栈，系统会新建对应的任务栈，并新建Activity实例压入栈中。
  >   2、如果存在指定的任务栈，则会查找该任务栈中是否存在该Activity实例
  >         a、如果不存在该实例，则会在该任务栈中新建Activity实例。
  >         b、如果存在该实例，则会直接引用，并且回调该实例的onNewIntent()方法。并且任务栈中该实例之上的Activity会被全部销毁。
  >
  >   使用场景：
  >   SingleTask这种启动模式最常使用的就是一个APP的首页，因为一般为一个APP的第一个页面，且长时间保留在栈中，所以最适合设置singleTask启动模式来复用。
  >
  > - **singleInstance：单实例模式**
  >
  >   单实例模式，顾名思义，只有一个实例。该模式具备singleTask模式的所有特性外，与它的区别就是，这种模式下的Activity会单独占用一个Task栈，具有全局唯一性，即整个系统中就这么一个实例，由于栈内复用的特性，后续的请求均不会创建新的Activity实例，除非这个特殊的任务栈被销毁了。以singleInstance模式启动的Activity在整个系统中是单例的，如果在启动这样的Activiyt时，已经存在了一个实例，那么会把它所在的任务调度到前台，重用这个实例。
  >
  >   特点：
  >   启动该模式Activity的时候，会查找系统中是否存在：
  >   1、不存在，首先会新建一个任务栈，其次创建该Activity实例。
  >   2、存在，则会直接引用该实例，并且回调onNewIntent()方法。
  >   特殊情况：该任务栈或该实例被销毁，系统会重新创建。
  >
  >   使用场景：
  >   很常见的是，电话拨号盘页面，通过自己的应用或者其他应用打开拨打电话页面 ，只要系统的栈中存在该实例，那么就会直接调用。

- onStart 和 onResume、onPause 和 onStop 的区别

  > onStart：界面由不可见到可见
  >
  > onResume：界面获取焦点，可以交互
  >
  > onPause：界面失去焦点，不可交互
  >
  > onStop：界面由可见到不可见

- Activity之间传递数据的方式Intent是否有大小限制，如果传递的数据量偏大，有哪些方案

  > 有的.
  >
  > 一. 限制传递数据量
  >
  > 二. 改变传输数据方式（參见[Activity之间传递数据的方式](http://blog.csdn.net/rflyee/article/details/47431633)）
  >
  > \1. 静态static
  >
  > \2. 单例
  >
  > \3. Application
  >
  > \4. 持久化

- Activity的onNewIntent()方法什么时候会执行

  > 当Activity的launchMode为singleTask的时候，通过Intent启到一个Activity，如果系统已经存在一个实例，系统就会将请求发送到这个实例上，但这个时候，系统就不会再调用通常情况下我们处理请求数据的onCreate方法，而是调用onNewIntent方法。这时的Activity执行的生命周期为：onNewIntent()—>onRestart()—>onStart()—>onResume();
  >
  > 当然也不要忘记，系统可能会随时杀掉后台运行的Activity，如果这一切发生，那么系统就会调用onCreate方法，而不调用onNewIntent方法，一个好的解决方法就是在onCreate和onNewIntent方法中调用同一个处理数据的方法。

- 显示启动和隐式启动

  > 显示启动：startActivity(Intent(MainActivity.this,OtherActivity.class))
  >
  > 隐式启动：Intent intent = new Intent("com.ghost.deng.OTHER");
  >
  > ​                  startActivity(intent);
  >
  > 隐式启动可以启动其他app的activity。

- scheme使用场景,协议格式,如何使用

  > 使用场景：
  >
  >  1.服务器下发跳转路径，客户端根据服务器下发跳转路径跳转相应的页面
  >
  >  2.H5页面点击锚点，根据锚点具体跳转路径APP端跳转具体的页面
  >
  >  3.APP端收到服务器端下发的PUSH通知栏消息，根据消息的点击跳转路径跳转相关页面
  >
  >  4.APP根据URL跳转到另外一个APP指定页面
  >
  > 协议格式：[scheme:][//authority][path][?query][#fragment]
  >
  > 在android中，除了scheme、authority是必须要有的，其它的几个path、query、fragment，它们每一个可以选择性的要或不要，但顺序不能变
  >
  > scheme跳转在微信被禁止了，解决方案有两个，
  >
  > 1）提示在浏览器中打开，然后浏览器中可以通过scheme跳转
  >
  > 2）通过App Link，类似iOS的Universal Links，在xml中需要配置android:autoVerify="true"，并且要在域名下面添加assetlinks.json文件


- ANR 的四种场景

  > - InputDispatchTimeOut（常见）
  >
  >   输入时间分发超时5s，包括按键和触摸事件
  >
  >   logcat日志关键字：Input event dispatching timed out
  >
  > - Broadcast Timeout
  >
  >   前台Broadcast：onReceiver在10s内没有处理完成发生ANR。
  >
  >   后台Broadcast：onReceiver在60s内没有处理完成发生ANR。
  >
  >   logcat日志关键字：Timeout of broadcast BroadcastRecord
  >
  > - Service Timeout
  >
  >   前台Service：onCreate，onStart，onBind等生命周期在20s内没有处理完成发生ANR。
  >
  >   后台Service：onCreate，onStart，onBind等生命周期在200s内没有处理完成发生ANR。
  >
  >   logcat日志关键字：Timeout executing service
  >
  > - ContentProvider Timeout
  >
  >   ContentProvider在10s内没有处理完成发生ANR。
  >
  >   logcat日志关键字：Timeout publishing content providers

- onCreate和onRestoreInstance方法中恢复数据时的区别

  > 因为onSaveInstanceState 不一定会被调用，所以onCreate()里的Bundle参数可能为空，如果使用onCreate()来恢复数据，一定要做非空判断。而onRestoreInstanceState的Bundle参数一定不会是空值，因为它只有在上次activity被回收了才会调用。
  >
  > onRestoreInstanceState是在onStart()之后被调用的。有时候我们需要onCreate()中做的一些初始化完成之后再恢复数据，用onRestoreInstanceState会比较方便。

- Activty间传递数据的方式

  > Intent()、静态变量、全局变量、sp、文件

- 跨App启动Activity的方式,注意事项

  >1、隐式Intent调起方式，当使用action匹配规则，目标组件不要忘记添加默认的category规则。
  >
  >2、关于android:exported属性是否需要设置
  >
  >android:exported 是Android中的四大组件 Activity，Service，Provider，Receiver 四大组件中都会有的一个属性。总体来说它的主要作用是：是否支持其它应用调用当前组件。 如果包含有intent-filter 默认值为true; 没有intent-filter默认值为false。

- Activity任务栈是什么

  > Activity任务栈是Android对Activity界面的一种管理方式。任务栈，顾名思义就是“后进先出”，也就是说，当从一个Activity中启动一个新的Activity界面时，新界面将位于Activity栈的栈顶；当用户按下返回键时，系统将弹出栈顶的Activity并将上一个Activity置为栈顶，此时应用界面也就回到该Activity界面了，如果应用只包含一个Activity或者当前任务栈只存在一个Activity时，按下返回键，系统将退出应用。
  >

- 有哪些Activity常用的标记位Flags

  > FLAG_ ACTIVITY_NEW_TASK：这个标记位的作用是指定Activity启动模式为”singleTask“，其作用等同于在AndroidManifest中指定android:launchMode="singleTask"相同。
  >
  > FLAG_ACTIVITY_SINGLE_TOP：这个标记位的作用是指定Activity启动模式为”singleTop“，其作用等同于在AndroidManifest中指定android:launchMode="singleTop"相同。
  >
  > FLAG_ACTIVITY_CLEAR_TOP：具有此标记位的Activity，当它启动的时候，在同一个任务栈中所有位于它上面的Activity都要出栈。这个模式一般需要和FLAG_ ACTIVITY_NEW_TASK配合使用，在这种情况下，如果调用的Activity的实例已存在，那么系统就会回调方法onNewIntent()。
  > 如果被启动的Activity采用的是standard，那么它连同它之上的Activity都要出栈，系统会重新创建Activity实例并放入栈顶。singleTask模式其实默认就具有此标记位的效果。
  >
  > FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS：具有这个标记的Activity不会出现在历史Activity的列表中，当某些情况我们不希望用户通过历史列表回到我们这个Activity的时候，这个标记将起作用。它等同于在AndroidManifest中为Activity添加属性：android:excludeFromRecents=“true”。

- Activity的数据是怎么保存的,进程被Kill后,保存的数据怎么恢复的

  >1、保存数据的方法:onSaveInstanceState(Bundle outState)
  >
  >触发条件：Activity未执行finish，比如按了home键，电源键，旋转Activity，内存不足等。这种数据保存都是临时的。如果想保存一些持久数据，用onPause
  >
  >
  >
  >2、恢复数据的方法：onRestoreInstanceState(BundlesavedInstanceState)
  >
  >触发条件：onSaveInstanceState已经触发，Activity被系统回收之后，再次打开。
  >
  >
  >
  >3、恢复数据的方法：Create(Bundle savedInstanceState)
  >
  >触发条件：创建Activity实例的时候

#### 2.Service

- service 的生命周期，两种启动方式的区别

  > - 启动服务
  >
  >   startService执行的生命周期：onCreate()->onStartCommand()->onDestory()
  >
  >   1）启动服务/startService() 
  >
  >   ​      单次启动：onCreate()->onStartCommand()
  >
  >   ​      多次启动：onCreate()->onStartCommand()->onStartCommand()...
  >
  >   ​      注意：多次启动onCreate() 只会调用一次
  >
  >   2）停止服务/stopService()
  >
  >   ​     onDestory()
  >
  > - 绑定服务
  >
  >   bindService执行的生命周期：onCreate()->onBind()->unbindService()->onDestory()
  >
  >   1）绑定服务/bindService()
  >
  >   ​      onCreate()->onBind()
  >
  >   2）解绑服务/unbindService()
  >
  > 启动服务：不会随调用者摧毁而摧毁
  >
  > 绑定服务：可以与调用者绑定实现一些交互，但是与调用者共生死

- 如何保证Service不被杀死 ？

  > 没办法优雅的处理，优先级？自启动？alertmanager?

- Service与Activity怎么实现通信

  >bindService，通过serviceConnection获取ibinder交互

- IntentService是什么,IntentService原理，应用场景及其与Service的区别

  > 应用场景启动app后，后台自动下载资源，不用管生死，完成任务自己销毁。

- Service 的 onStartCommand 方法有几种返回值?各代表什么意思?

  > 1、START_STICKY： 如果service进程被kill掉，保留service的状态为开始状态，但不保留递送的intent对象。随后系统会尝试重新创建service，由 于服务状态为开始状态，所以创建服务后一定会调用onStartCommand(Intent,int,int)方法。如果在此期间没有任何启动命令被传 递到service，那么参数Intent将为null。
  >
  > 2、START_NOT_STICKY：“非粘性的”。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统不会自动重启该服务
  >
  > 3、START_REDELIVER_INTENT：重传Intent。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统会自动重启该服务，并将Intent的值传入。
  >
  > 4、START_STICKY_COMPATIBILITY：START_STICKY的兼容版本，但不保证服务被kill后一定能重启。

- bindService和startService混合使用的生命周期以及怎么关闭

  > 混合使用可以结合两者优点，既能与调用者实现一些交互，又不随调用者共生死
  >
  > onCreate – onStartCommand – onBind – onUnbind–onDestroy
  >
  > 关闭：先unbindService，然后stopService

- 用过哪些系统Service ？

  > wms、pkms、alaretmanager

- 了解ActivityManagerService吗？发挥什么作用

  > AMS，四大组件的管理

#### 3.BroadcastReceiver

- 广播的分类和使用场景

  > 全局广播和局内广播：全局广播表示该广播可以通知/接收其他应用，局内广播是指广播的通知和接收只能在在同一个应用中处理；
  > 有序广播和无序广播：有序广播是指按照一定顺序的优先级，一个一个的进行传递，可以修改广播数据，终止广播传递；无序广播是指没有顺序优先级，任何接收者都能无时效性的接收传递的广播，这种类型的广播不能被修改数据和终止传递；
  > 自定义广播和系统广播：设置自定义广播不具有约束性，自己想怎样设置都行；设置系统广播具有约束性，只能按照定义的规则设置；（注意：实现自定义/系统广播的步骤都是一样的，都是通过设置action来标识广播，只不过系统广播中对action的设置只能按照约束的规则来设置，自定义广播则不需要）；

- 广播的两种注册方式的区别

  > 共同点：他们都可以用于自定义广播或系统广播注册；
  >
  > 不同点：动态注册最终通过registerReceiver()方法进行注册，所以它相应的也有释放/销毁广播的方法（unregisterReceiver）；而静态注册则不能手动释放/销毁广播；
  >
  > unregisterReceiver(testBReceiver); //这样就可以销毁广播了
  >
  > 建议：对于系统级别的，重复使用频率高的广播，例如网络变化广播，建议使用静态注册；相反，对于自定义，使用频率低的广播，建议使用动态注册；这样不用了了可以通过unregisterReceiver方法进行释放资源，防止内存泄漏；
  >
  > （注意：对于unregisterReceiver()方法的使用建议在Activity中的onPause()声明周期中使用，因为当手机内存不足要回收Activity资源时，有可能执行到onPause()生命周期就销毁Activity，而不会执行onStop()和onDetory()这两个生命周期方法，也就是说，如果在onStop()或onDetory()调用unregisterReceiver()方法时，在出现上述情况下，有可能回收失败，赵成内存泄漏，所以最安全的方式是在onPause()方法中使用）

- 广播发送和接收的原理

  > AMS，类似于观察者模式，在AMS注册监听
  >
  > 被观察者消息发出的时候，在AMS查找是否有观察者，找到的话调用观察者回调方法。

- 本地广播和全局广播的区别

  > LocalBroadcastManager

#### 4.ContentProvider

- 什么是ContentProvider及其使用

  > ContentProvider是Android中跨进程数据交换的重要类，Android为数据交换提供了一个标准ContentProvider.
  >
  > 
  >
  > 那么应该如何完整实现开发一个contentProvider呢
  > 1、首先需要定义自己的ContentProvider，该类需要继承Android提供的ContentProvider的基类
  >
  >      ```java
  > public class DictProvider extends ContentProvider {
  >     String TAG  = "=======TAG=====" ;
  >     @Override
  >     public boolean onCreate() {
  >         Log.v(TAG,"onCreate");
  >         return false;
  >     }
  >     @Override
  >     public Cursor query(@NonNull Uri uri, @Nullable String[] projection, @Nullable String selection, @Nullable String[] selectionArgs, @Nullable String sortOrder) {
  >         Log.v(TAG,"query");
  >         return null;
  >     }
  >     @Override
  >     public String getType(@NonNull Uri uri) {
  >         Log.v(TAG,"getType");
  >         return null;
  >     }
  >     @Override
  >     public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
  >         Log.v(TAG,"insert");
  >         return null;
  >     }
  > 
  >     @Override
  >     public int delete(@NonNull Uri uri, @Nullable String selection, @Nullable String[] selectionArgs) {
  >         Log.v(TAG,"delete");
  >         return 0;
  >     }
  > 
  >     @Override
  >     public int update(@NonNull Uri uri, @Nullable ContentValues values, @Nullable String selection, @Nullable String[] selectionArgs) {
  >         Log.v(TAG,"update");
  >         return 0;
  >     }
  > }
  > 
  >      ```
  >
  > 
  >
  > 2、需要在AndroidManifest.xml文件中注册这个ContentProvider
  >
  > ```java
  > <provider
  >             android:authorities="com.jewelermobile.gangfu.zdydemo1.DictProvider"
  >             android:name=".activity.DictProvider"
  >             android:exported="true"/>// true 允许外部应用启动 反之 不能启动
  > ```

- ContentProvider的权限管理

  > 1. 无限制：
  >
  >    如果一个provider 用了exported=true，那么任意三方调用方都能对其进行调用；
  >
  > 2. 只对满足特定条件三方才进行暴露：
  >
  >    provider方：
  >
  >    <provider
  >        android:name=".MyContentProvider"
  >        android:authorities="com.anddle.mycontentprovider"
  >        android:enabled="true"
  >        android:exported="true"
  >        android:permission="com.anddle.provideraccess" />
  >
  >       <permission
  >            android:name="com.anddle.provideraccess"
  >            android:label="provider pomission"
  >            android:protectionLevel="normal" />
  >
  >    1).配置清单中加android:permission="com.anddle.provideraccess" 
  >
  >    2).再定义permission，其对应的name与声明的权限对应
  >
  >    调用方：
  >
  >    清单文件中声明： <uses-permission android:name="com.anddle.provideraccess" /> 即可调用provider
  >
  > 3. 在②只对满足特定条件基础上并区分读写
  >
  >    provider方：
  >
  >    <provider
  >        android:name=".MyContentProvider"
  >        ......
  >        android:readPermission="com.anddle.provideraccess.read" />
  >
  >    <permission
  >            android:name="com.anddle.provideraccess.read"
  >            android:label="provider pomission"
  >            android:protectionLevel="normal" />
  >
  >    1.配置清单中加android:readpermission(writepermission)="com.anddle.provideraccess.read" 
  >
  >    2.再定义permission，其对应的name与声明的权限对应
  >
  >    调用方：
  >
  >    清单文件中声明： <uses-permission android:name="com.anddle.provideraccess.read" /> 即可调用provider

- ContentProvider,ContentResolver,ContentObserver之间的关系

  >ContentProvider: 四大组件之一，内容提供者，使用`ContentProvider`对外共享数据的好处是统一了数据的访问方式。
  >
  >- 四大组件的内容提供者，主要用于对外提供数据
  >- 实现各个应用程序之间的（跨应用）数据共享，比如联系人应用中就使用了ContentProvider（我在上篇ContentProvider是如何实现数据共享的？—— ContentProvider数据共享案例中提供了一个简单的删除联系人的案例）。其实它也只是一个中间人，真正的数据源是文件或者SQLite等。
  >- 一个应用实现`ContentProvider`来提供内容给别的应用来操作，通过`ContentResolver`来操作别的应用数据，当然在自己的应用中也可以。
  >- 负责管理结构化数据的访问；
  >- 封装数据并且提供一套定义数据安全的机制；
  >- 是一套在不同进程间进行数据访问的接口；
  >- 为数据跨进程访问提供了一套安全的访问机制，对数据组织和安全访问提供了可靠的保证；
  >
  >
  >
  >ContentResolver: ContentResolver可以不同URI操作不同的ContentProvider中的数据，外部进程可以通过ContentResolver与ContentProvider进行交互。
  >
  >- 内容解析者，用于获取内容提供者提供的数据
  >
  >- ContentResolver.notifyChange（uri）发出消息
  >
  >  
  >
  >ContentObserver: 观察ContentProvider中的数据变化，并将变化通知给外界。
  >
  >- 内容监听器，可以监听数据的改变状态
  >
  >- 目的是观察（捕捉）特定Uri引起的数据库的变化，继而做一些相应的处理，它类似于数据库技术中的触发器（Trigger），当ContentObserver所观察的Uri发生变化时，便会触发它。触发器分为表触发器、行触发器，相应地ContentObsever也分为表ContentObserver、行ContentObserver，当然这是与它所监听的Uri MIME Type有关的
  >
  >- ContentResolver.registerContentObserver()监听消息

  

- ContentProvider的实现原理

  > 1. application初始化的时候会installProvider
  > 2. 向AMS请求provider的时候如果对端进程不存在则请求的那个线程需要一直等待
  > 3. 当对方的进程启动之后并publish之后，请求provider的线程才可返回，所以尽量不要在主线程请求provider
  > 4. 请求provider分为stable以及unsbale，stable类型链接在server进程挂掉之后，client进程会跟着被株连kill
  > 5. insert/delete/update默认建立stable链接，query默认建立unstable链接，如果失败会重新建立stable链接
  > 6. AMS作为一个中间管理员的身份，所有的provider会向它注册
  > 7. 向AMS请求到provider之后，就可以在client和server之间自行binder通信，不需要再经过systemserver

- ContentProvider的优点

  > 1.封装数据，提供统一接口，当项目需求需要修改数据源的时候，节省时间和人力
  >
  > 2.提供一种跨进程数据共享的方式
  >
  > 3.数据更新通知机制。

- Uri 是什么

  > Content://com.jewelermobile.gangfu.zdydemo1.DictProvider/资源部分
  >
  > （1）Content:// 这部分是Android的ContentProvider规定，并且固定
  > （2）com.jewelermobile.gangfu.zdydemo1.DictProvider 这部分是注册时的android:authorities，注册之后并且固定
  > （3）资源部分，这个根据查询资源改变所以不固定
  >
  > Uri其实就是一个地址，只有通过Uri才能对应去操作ContentProvider，就像一条唯一通往胜利道路。

#### 5.Handler

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
  >    return  sendMessageDelayed(getPostMessage(r), 0);
  > }
  > private static Message getPostMessage(Runnable r) {
  >    Message m = Message.obtain();
  >    m.callback = r;
  >    return m;
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
  >   //静态final对象，全局唯一，作为每个线程中ThreadLocal.ThreadLocalMap threadLocals中的key，存储对应的looper
  >   static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
  > 
  >   public static void prepare() {
  >        prepare(true);
  >   }
  > 
  >   private static void prepare(boolean quitAllowed) {
  >       //先获取，检查是否已经创建过
  >       if (sThreadLocal.get() != null) {
  >         throw new RuntimeException("Only one Looper may be created per thread");
  >       }
  >       //如果没有创建过，则创建，并添加进ThreadLocal，其实是添加到当前线程的ThreadLocalMap中，key是sThreadLocal，value是Looper对象
  >       sThreadLocal.set(new Looper(quitAllowed));
  >   }
  >   
  >   private Looper(boolean quitAllowed) {
  >         //创建MessageQueue
  >         mQueue = new MessageQueue(quitAllowed);
  >         mThread = Thread.currentThread();
  >   }
  > }
  > 
  > //ThreadLocal.java
  > class ThreadLocal{
  >    public void set(T value) {
  >       //获取当前线程
  >       Thread t = Thread.currentThread();
  >       //获取当前线程的ThreadLocalMap
  >       ThreadLocalMap map = getMap(t);
  >       if (map != null)
  >         map.set(this, value);
  >       else
  >         createMap(t, value);
  >   }
  > 
  >   public T get() {
  >       //获取当前线程
  >       Thread t = Thread.currentThread();
  >       //获取当前线程的ThreadLocalMap
  >       ThreadLocalMap map = getMap(t);
  >       if (map != null) {
  >         ThreadLocalMap.Entry e = map.getEntry(this);
  >         if (e != null) {
  >           @SuppressWarnings("unchecked")
  >           T result = (T)e.value;
  >           return result;
  >         }
  >       }
  >       return setInitialValue();
  >   }
  > 
  >   ThreadLocalMap getMap(Thread t) {
  >       return t.threadLocals;
  >   }
  > }
  > 
  > //Thread.java
  > class Thread{
  >    ThreadLocal.ThreadLocalMap threadLocals = null;
  > }
  > ```
  >
  > 

- HandlerThread是什么 & 好处 &原理 & 使用场景

- IdleHandler及其使用场景

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
  >         if (msg.callback != null) {
  >             handleCallback(msg);
  >         } else {
  >             if (mCallback != null) {
  >                 if (mCallback.handleMessage(msg)) {
  >                     return;
  >                 }
  >             }
  >             handleMessage(msg);
  >         }
  >  }
  > ```
  >
  > 

- Looper.quit/quitSafely的区别

  > Looper的quit/quitSafely方法源码如下：
  >
  > ```java
  > public void quit() {
  >     mQueue.quit(false);
  > }
  > 
  > public void quitSafely() {
  >     mQueue.quit(true);
  > }
  > 
  > void quit(boolean safe) {
  >         if (!mQuitAllowed) {
  >             throw new IllegalStateException("Main thread not allowed to quit.");
  >         }
  >  
  >         synchronized (this) {
  >             if (mQuitting) {
  >                 return;
  >             }
  >             mQuitting = true;
  >  
  >             if (safe) {
  >                 removeAllFutureMessagesLocked();
  >             } else {
  >                 removeAllMessagesLocked();
  >             }
  >  
  >             // We can assume mPtr != 0 because mQuitting was previously false.
  >             nativeWake(mPtr);
  >         }
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
  >             
  >             //没有消息，或者延时消息时间未到，则一直阻塞等待，底层是调用linux的epoll.wait等待
  >             nativePollOnce(ptr, nextPollTimeoutMillis);
  > 
  >             synchronized (this) {
  >                 // Try to retrieve the next message.  Return if found.
  >                 final long now = SystemClock.uptimeMillis();
  >                 Message prevMsg = null;
  >                 Message msg = mMessages;
  >                 //如果当前消息是消息屏障，即消息屏障的区别是msg.target == null，则去遍历寻找最近的异步消息
  >                 if (msg != null && msg.target == null) {
  >                     // Stalled by a barrier.  Find the next asynchronous message in the queue.
  >                     do {
  >                         prevMsg = msg;
  >                         msg = msg.next;
  >                     } while (msg != null && !msg.isAsynchronous()); //判断是否为异步消息
  >                 }
  >                 //如果找到了消息
  >                 if (msg != null) {
  >                     //消息延时时间未到，设置阻塞时间nextPollTimeoutMillis
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
  >                     //没找到消息，则设置需要阻塞标志
  >                     // No more messages.
  >                     nextPollTimeoutMillis = -1;
  >                 }
  > 
  >                 // Process the quit message now that all pending messages have been handled.
  >                 //Looper.quit->messageQueue.quit设置mQuitting=true，则返回null消息，Looper.loop中接收到会结束死循环
  >                 if (mQuitting) {
  >                     dispose();
  >                     return null;
  >                 }
  > 
  >                 //如果当前无消息处理，或者需要等待延时，则处理空闲队列
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
  > ```
  >
  > 

- 子线程中是否可以用MainLooper去创建Handler，Looper和Handler是否一定处于一个线程

  > 可以的，不一定
  >
  > ```java
  > thread{
  >   Handler(Looper.getMainLooper()).sendMessage(Message())
  > }
  >   
  > public Handler(@NonNull Looper looper) {
  >   this(looper, null, false);
  > }
  > public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
  >         mLooper = looper;
  >         mQueue = looper.mQueue;
  >         mCallback = callback;
  >         mAsynchronous = async;
  > }
  > ```

- ANR和Handler的联系

  > Handler是线程间通讯的机制，Android中，网络访问、文件处理等耗时操作必须放到子线程中去执行，否则将会造成ANR异常。
  >
  > ANR异常：Application Not Response 应用程序无响应
  >
  > 产生ANR异常的原因：在主线程执行了耗时操作，对Activity来说，主线程阻塞5秒将造成ANR异常，对BroadcastReceiver来说，主线程阻塞10秒将会造成ANR异常。
  >
  > 解决ANR异常的方法：耗时操作都在子线程中去执行
  >
  > 但是，Android不允许在子线程去修改UI，可我们又有在子线程去修改UI的需求，因此需要借助Handler。

#### 6.View绘制

- View绘制流程

  > handlerResumeActivity->windowManagerGlobal.addView->new ViewRootImpl->viewRootImpl.setView->requestLayout->scheduleTravalsers->chorographe.postCallback(traversalRunable)->travalsersRunable->doTraversal->performTraversals->预测量->performMer

- MeasureSpec是什么

  > MeasureSpec代表的是一个32位的int值，高两位对应的是SpecMode，指测量模式，
  >
  > 低30位对应的是SpecSize，指的是某中测量模式下的大小 。
  >
  > SpecMode：精准模式（EXACTLY）、AT_MOST、UNSPECIFIED

- 子View创建MeasureSpec创建规则是什么

  > 通过父View的MeasureSpec和子View的LayoutParams来确定子View的MeasureSpec的
  >
  > <img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220207161006516.png" alt="image-20220207161006516" style="zoom:50%;" />

- 自定义View wrap_content不起作用的原因

  > 因为view中的onMeasure()方法如下：
  >
  > ```java
  > protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  >         setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
  >                 getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
  > }
  > ```
  >
  > 而getDefaultSize()代码为:
  >
  > ```java
  > public static int getDefaultSize(int size, int measureSpec) {
  >     int result = size;
  >     int specMode = MeasureSpec.getMode(measureSpec);
  >     int specSize = MeasureSpec.getSize(measureSpec);
  > 
  >     switch (specMode) {
  >       case MeasureSpec.UNSPECIFIED:
  >         result = size;
  >         break;
  >       case MeasureSpec.AT_MOST:
  >       case MeasureSpec.EXACTLY:
  >         result = specSize; // 这里是问题的关键
  >         break;
  >     }
  >     return result;
  > }
  > ```
  >
  > 从上面的代码可以看出，android.view.View在measure时，父view会传入MeasureSpec.AT_MOST或者MeasureSpec.EXACTLY，但是View居然不关注自身的属性，而是一律使用父view给的最大可用specSize，这就导致了View填充父view的现象。
  >
  > Android源码中TextView这些对wrap_content都有做处理，所以我们在自定义view的时候，也可以对wrap_content做处理，比如按情况设置默认大小。

- 在Activity中获取某个View的宽高有几种方法

  > - onWindowFocusChanges
  >
  >   当Activity的窗口得到焦点和失去焦点时均会被调用一次
  >
  >   如果频繁的进行onResume和onPause，那么onWindowFocusChanged也会被频繁调用
  >
  > - ViewThreeObserve
  >
  >   当view树的状态发生改变或者view树内部的view可见发生变化时，onGlobalLayout方法将被回调
  >
  > - post
  >
  >   通过post可以将一个runnable投递到[消息队列](https://so.csdn.net/so/search?q=消息队列)的尾部,然后等待Looper调用次Runnable的时候,View已经初始化好了

- 为什么onCreate获取不到View的宽高

  > 这时候还没有进行测量

- View#post与Handler#post的区别

  > 总结：当view.onAttachToWindow执行完之后，View#post就是将任务post到UI主线程的消息队列，而在此之前会先存放到mRunQueue队列中，然后在performTraversals()方法中将这些任务都post到UI主线程的消息队列。
  >
  > 
  >
  > View#post源码分析：
  >
  > ```java
  > public boolean post(Runnable action) {
  >         final AttachInfo attachInfo = mAttachInfo;
  >         //当attachInfo不为空，即已经执行完view.onAttachToWindow时，直接post到UI队列
  >         if (attachInfo != null) {
  >             return attachInfo.mHandler.post(action);
  >         }
  > 
  >         // Postpone the runnable until we know on which thread it needs to run.
  >         // Assume that the runnable will be successfully placed after attach.
  >         //否则先缓存到mRunQueue队列
  >         getRunQueue().post(action);
  >         return true;
  > }
  > ```
  >
  > attachInfo是在View.dispatchAttachedToWindow(AttachInfo info, int visibility)中赋值的，而dispatchAttachedToWindow的执行顺序是：
  >
  > ```java
  > // ViewRootImpl.java
  > private void performTraversals() {
  >     // 省略其他代码
  >     host.dispatchAttachedToWindow(mAttachInfo, 0);
  >   
  >     // 省略其他代码
  >     getRunQueue().executeActions(mAttachInfo.mHandler);
  >   
  >     // 省略其他代码
  >     performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
  >     // 省略其他代码
  >     performLayout(lp, mWidth, mHeight);
  >     // 省略其他代码
  >     performDraw();
  > }
  > ```
  >
  > getRunQueue().executeActions(mAttachInfo.mHandler)就是执行之前mRunQueue中的Runnable，但是我们都知道在View#post中可以获取控件的宽高，但是看代码却是在measure,layout和draw之前，所以我们要继续看下getRunQueue().executeActions(mAttachInfo.mHandler)的源码：
  >
  > ```java
  > public void executeActions(Handler handler) {
  >   synchronized (this) {
  >     final HandlerAction[] actions = mActions;
  >     for (int i = 0, count = mCount; i < count; i++) {
  >       final HandlerAction handlerAction = actions[i];
  >       handler.postDelayed(handlerAction.action, handlerAction.delay);
  >     }
  > 
  >     mActions = null;
  >     mCount = 0;
  >   }
  > }
  > ```
  >
  > 这里其实就是把Runnable发送到UI主线程的handler，但是此时我们的主线程handler已经在执行绘制等Runnable，所以mRunQueue中的消息总是在绘制完成之后执行，故我们可以正常获取宽高。
  >
  > https://blog.csdn.net/u014761700/article/details/79908801

- Android绘制和屏幕刷新机制原理

- Choreography原理

- 什么是双缓冲

- 为什么使用SurfaceView

- 什么是SurfaceView

- View和SurfaceView的区别

- SurfaceView为什么可以直接子线程绘制

- SurfaceView、TextureView、SurfaceTexture、GLSurfaceView

- getWidth()方法和getMeasureWidth()区别

- invalidate() 和 postInvalidate() 的区别

- Requestlayout，onlayout，onDraw，DrawChild区别与联系

- LinearLayout、FrameLayout 和 RelativeLayout 哪个效率高

- LinearLayout的绘制流程

- 自定义 View 的流程和注意事项

- 自定义View如何考虑机型适配

- 自定义控件优化方案

- invalidate怎么局部刷新

  > invalidate(Rect)，脏区域，失效，重绘

- View加载流程（setContentView）

#### 7.View事件分发

- View事件分发机制

  > ViewRootImpl->setView->设置事件监听
  >
  > 点击事件触发->DoctorView.dispatchTouchEvent->activity.dispatchTouchEvent->PhoneWindow.superDispatcherTouchEvent->DoctorView.superDispatchTouchEvent->ViewGroup.dispatchTouchEvent->onInterceptTouchEvent
  >
  > dow事件或者mFirstTouchTarget不为空时，会走onInterceptorTouchEvent，如果返回true，则直接交给自己super.dispatchTouchEvent(event)处理即子view的处理流程，否则调用子view的dispatchTouchEvent，先判断是否设置了onTouch事件，如果有的话，先执行，看其返回值，再。...

- view的onTouchEvent，OnClickListerner和OnTouchListener的onTouch方法 三者优先级

  > onTouch->onTouchEvent->OnClickListerner

- onTouch 和onTouchEvent 的区别

  > dispatchTouchEvent 中判断是否

- ACTION_CANCEL什么时候触发

  > 移除屏幕外， 或者mFiretTouchTarget != null && interceptor = true

- 事件是先到DecorView还是先到Window

  > 先到 DecorView

- 点击事件被拦截，但是想传到下面的View，如何操作

  > 内部拦截，在子view中需要的地方调用getParent().requestDisallowInterceptTouchEvent(false);

- 如何解决View的事件冲突

  > 内部拦截、外部拦截、类似于nestedscrollerView和recyclerView使用回调

- 在 ViewGroup 中的 onTouchEvent 中消费 ACTION_DOWN 事件，ACTION_UP事件是怎么传递

  > 直接交给ViewGroup的super.dispatcherTouchEvent处理

- Activity ViewGroup和View都不消费ACTION_DOWN,那么ACTION_UP事件是怎么传递的

- 同时对父 View 和子 View 设置点击方法，优先响应哪个

  > 点击在子view上的话，响应子view

- requestDisallowInterceptTouchEvent的调用时机

  > 子view中需要让出事件处理，允许父view拦截

#### 8.RecycleView

- RecyclerView的多级缓存机制,每一级缓存具体作用是什么,分别在什么场景下会用到哪些缓存
- RecyclerView的滑动回收复用机制
- RecyclerView的刷新回收复用机制
- RecyclerView 为什么要预布局
- ListView 与 RecyclerView区别
- RecyclerView性能优化

#### 9.Viewpager&Fragment

- Fragment的生命周期 & 结合Activity的生命周期
- Activity和Fragment的通信方式， Fragment之间如何进行通信
- getFragmentManager、getSupportFragmentManager 、getChildFragmentManager之间的区别？
- 为什么使用Fragment.setArguments(Bundle)传递参数
- FragmentPagerAdapter与FragmentStatePagerAdapter的区别与使用场景
- FragmentPageAdapter和FragmentStatePageAdapter区别及使用场景
- Fragment懒加载
- ViewPager2与ViewPager区别
- Fragment嵌套问题

#### 10.WebView

- 如何提高WebView加载速度
- WebView与 js的交互
- WebView的漏洞
- JsBridge原理

#### 11.动画

- 动画的类型
- 补间动画和属性动画的区别
- ObjectAnimator，ValueAnimator及其区别
- TimeInterpolator插值器，自定义插值器
- TypeEvaluator估值器

#### 12.Bitmap

- Bitmap 内存占用的计算
- getByteCount() & getAllocationByteCount()的区别
- Bitmap的压缩方式
- LruCache & DiskLruCache原理
- 如何设计一个图片加载库
- 有一张非常大的图片,如何去加载这张大图片
- 如果把drawable-xxhdpi下的图片移动到drawable-xhdpi下，图片内存是如何变的。
- 如果在hdpi、xxhdpi下放置了图片，加载的优先级。如果是400_800，1080_1920，加载的优先级。

#### 13.mvc&mvp&mvvm

- MVC及其优缺点
- MVP及其优缺点
- MVVM及其优缺点
- MVP如何管理Presenter的生命周期，何时取消网络请求

#### 14.Binder

- Android中进程和线程的关系,区别
- 为何需要进行IPC,多进程通信可能会出现什么问题
- Android中IPC方式有几种、各种方式优缺点
- 为何新增Binder来作为主要的IPC方式
- 什么是Binder
- Binder的原理
- Binder Driver 如何在内核空间中做到一次拷贝的？
- 使用Binder进行数据传输的具体过程
- Binder框架中ServiceManager的作用
- 什么是AIDL
- AIDL使用的步骤
- AIDL支持哪些数据类型
- AIDL的关键类，方法和工作流程
- 如何优化多模块都使用AIDL的情况
- 使用 Binder 传输数据的最大限制是多少，被占满后会导致什么问题
- Binder 驱动加载过程中有哪些重要的步骤
- 系统服务与bindService启动的服务的区别
- Activity的bindService流程
- 不通过AIDL，手动编码来实现Binder的通信

#### 15.内存泄漏&内存溢出

- 什么是OOM & 什么是内存泄漏以及原因
- Thread是如何造成内存泄露的，如何解决？
- Handler导致的内存泄露的原因以及如何解决
- 如何加载Bitmap防止内存溢出
- MVP中如何处理Presenter层以防止内存泄漏的

#### 16.性能优化

- 内存优化
- 启动优化
- 布局加载和绘制优化
- 卡顿优化
- 网络优化

#### 17.Window&WindowManager

- 什么是Window
- 什么是WindowManager
- 什么是ViewRootImpl
- 什么是DecorView
- Activity，View，Window三者之间的关系
- DecorView什么时候被WindowManager添加到Window中

#### 18.WMS

- 什么是WMS
- WMS是如何管理Window的
- IWindowSession是什么，WindowSession的创建过程是怎样的
- WindowToken是什么
- WindowState是什么
- Android窗口大概分为几种？分组原理是什么
- Dialog的Context只能是Activity的Context，不能是Application的Context
- App应用程序如何与SurfaceFlinger通信的
- View 的绘制是如何把数据传递给 SurfaceFlinger 的
- 共享内存的具体实现是什么
- relayout是如何向SurfaceFlinger申请Surface
- 什么是Surface

#### 19.AMS

- ActivityManagerService是什么？什么时候初始化的？有什么作用？
- ActivityThread是什么?ApplicationThread是什么?他们的区别
- Instrumentation是什么？和ActivityThread是什么关系？
- ActivityManagerService和zygote进程通信是如何实现的
- ActivityRecord、TaskRecord、ActivityStack，ActivityStackSupervisor，ProcessRecord
- ActivityManager、ActivityManagerService、ActivityManagerNative、ActivityManagerProxy的关系
- 手写实现简化版AMS

#### 20.系统启动

- android系统启动流程
- SystemServer，ServiceManager，SystemServiceManager的关系
- 孵化应用进程这种事为什么不交给SystemServer来做，而专门设计一个Zygote
- Zygote的IPC通信机制为什么使用socket而不采用binder

#### 21.App启动&打包&安装

- 应用启动流程
- apk组成和Android的打包流程
- Android的签名机制，签名如何实现的,v2相比于v1签名机制的改变
- APK的安装流程

#### 22.序列化

- 什么是序列化
- 为什么需要使用序列化和反序列化
- 序列化的有哪些好处
- Serializable 和 Parcelable 的区别
- 什么是serialVersionUID
- 为什么还要显示指定serialVersionUID的值?

#### 23.模块化&组件化

- 什么是模块化
- 什么是组件化
- 组件化优点和方案
- 组件独立调试
- 组件间通信
- Aplication动态加载
- ARouter原理

#### 24.热修复&插件化

- 插件化的定义
- 插件化的优势
- 插件化框架对比
- 插件化流程
- 插件化类加载原理
- 插件化资源加载原理
- 插件化Activity加载原理
- 热修复和插件化区别
- 热修复原理

#### 25.AOP

- AOP是什么
- AOP的优点
- AOP的实现方式,APT,AspectJ,ASM,epic,hook

#### 26.Jectpack

- Navigation
- DataBinding
- Viewmodel
- livedata
- liferecycle

#### 27.开源框架

- Okhttp源码流程,线程池

- Okhttp拦截器,addInterceptor 和 addNetworkdInterceptor区别

- Okhttp责任链模式

- Okhttp缓存怎么处理

- Okhttp连接池和socket复用

- Glide怎么绑定生命周期

- Glide缓存机制,内存缓存，磁盘缓存

- Glide与Picasso的区别

- LruCache原理

- Retrofit源码流程,动态代理

- LeakCanary弱引用,源码流程

- Eventbus

- Rxjava

### 二. Java相关

#### 1.HashMap

- HashMap原理

- HashMap 有用过吗？您能给我说说他的主要用途吗？

- 您能说说 HashMap 常用操作的底层实现原理吗？如存储 put(K key, V value)，查找 get(Object key)，删除 remove(Object key)，修改 replace(K key, V value)等操作

- hash 冲突（或者叫 hash 碰撞）是什么？为什么会出现这种现象，如何解 决 hash 冲突？

- HashMap 的容量为什么一定要是 2 的 n 次方？

- 您能说说 HashMap 和 HashTable 的区别吗？

- HashMap中put()如何实现的

- HashMap中get()如何实现的

- 为什么HashMap线程不安全

- HashMap1.7和1.8有哪些区别

- 解决hash冲突的时候，为什么用红黑树

- 红黑树的效率高，为什么一开始不用红黑树存储

- 不用红黑树，用二叉查找树可以不

- 为什么阀值是8才转为红黑树

- 为什么退化为链表的阈值是6

- hash冲突有哪些解决办法

- HashMap在什么条件下扩容

- HashMap中hash函数怎么实现的，还有哪些hash函数的实现方式

- 为什么不直接将hashcode作为哈希值去做取模,而是要先高16位异或低16位

- 为什么扩容是2的次幂

- 链表的查找的时间复杂度是多少

- 红黑树

#### 2.ArrayList

- ArrayList定义

- ArrayList 的构造器

- add 方法源码分析

- get 方法源码分析

- set 方法源码分析

- ArrayList和LinkedList的区别，以及应用场景

#### 3. LinkedList

- LinkedList 定义

- LinkedList 支持的操作

- Node 类

- addFirst 源码分析

- getFirst 方法源码分析

- removeFirst 方法源码分析

- add(int index, E e)方法源码分析
#### 4. Hashset 源码

- 属性

- 构造方法

- 添加元素

- 删除元素

- 查询元素

- 遍历元素

- 全部源码

#### 5. 内存模型

- 内存模型产生背景
- 物理机的并发问题
- Java 内存模型的组成分析
- Java 内存间的交互操作
- Java 内存模型运行规则

#### 6. 垃圾回收算法（JVM）

- Jvm的内存模型,每个里面都保存的什么
- 类加载机制的几个阶段加载、验证、准备、解析、初始化、使用、卸载
- 对象实例化时的顺序
- 类加载器,双亲委派及其优势
- 垃圾回收机制
- 谈谈对 JVM 的理解?
- JVM 内存区域，开线程影响哪块区域内存？
- 对 Dalvik、ART 虚拟机有什么了解？对比

#### 7.多线程

- 谈一谈java线程模型
- Java中创建线程的方式,Callable,Runnable,Future,FutureTask
- 线程的几种状态
- 谈谈线程死锁，如何有效的避免线程死锁？
- 如何实现多线程中的同步
- synchronized和Lock的使用、区别,原理；
- volatile，synchronized和volatile的区别？为何不用volatile替代synchronized？
- 锁的分类，锁的几种状态，CAS原理
- 为什么会有线程安全？如何保证线程安全
- sleep()与wait()区别,run和start的区别,notify和notifyall区别,锁池,等待池
- Java多线程通信
- 为什么Java用线程池
- Java中的线程池参数,共有几种
- 说下 Java 中的线程创建方式，线程池的工作原理

#### 8.注解

- 注解的分类和底层实现原理
- 自定义注解

#### 9.反射

- 什么是反射
- 反射机制的相关类
- 反射中如何获取Class类的实例
- 如何获取一个类的属性对象 & 构造器对象 & 方法对象
- Class.getField和getDeclaredField的区别，getDeclaredMethod和getMethod的区别
- 反射机制的优缺点

#### 10.泛型

- 泛型概念的提出（为什么需要泛型）？
- 什么是泛型？
- 自定义泛型接口、泛型类和泛型方法
- 类型通配符

#### 11.设计模式

- 你所知道的设计模式有哪些
- 单例设计模式
- 工厂设计模式
- 建造者模式（Builder）
- 适配器设计模式
- 装饰模式（Decorator）
- 策略模式（strategy）
- 观察者模式（Observer）

