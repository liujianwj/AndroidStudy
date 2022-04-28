- 说下Activity生命周期


 > onCreate、onStart、onResume、onPause、onStop、onDestory


- Activity A 启动另一个Activity B 会调用哪些方法？如果B是透明主题的又或则是个DialogActivity呢

> A启动B，B为正常界面：A.onPause->B.onCreate->B.onStart->B.onResume->A.onStop
>                  返回：B.onPause->A.reStart->A.onStart->A.onResume->B.onStop->B.Destory
>
> A启动B，B为透明主题：A.onPause->B.onCreate->B.onStart->B.onResume
>                     返回：B.onPause->A.onResume->B.onStop->B.Destory  
>
> A启动Dialog：无影响
>
> 锁屏：只会调用onPause，解锁：调用onResume
>
> 横竖屏切换：onPause -> onStop ->onSaveInstanceState -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume


- 说下onSaveInstanceState()方法的作用 ? 何时会被调用？onSaveInstanceState(),onRestoreInstanceState的调用时机？

  > Activity中为我们提供了onSaveInstanceState()方法，用于保证在系统回收Activity前被调用，以此来解决被回收数据丢失的问题，并通过onRestoreInstanceState()方法恢复数据。
  > onSaveInstanceState()和onRestoreInstanceState()并不是成对出现，也就是说onSaveInstanceState()调用了并不能保证onRestoreInstanceState()方法调用，但onRestoreInstanceState()调用了则页面必然被系统回收了，onSaveInstanceState()肯定被调用了。
  > 1）onSaveInstanceState()肯定被执行时机
  > 1、当用户按下HOME键时
  > 2、从最近应用中选择运行其他的程序时
  > 3、按下电源按键（关闭屏幕显示）时
  > 4、从当前activity启动一个新的activity时
  > 5、屏幕方向切换时(竖屏切横屏或横屏切竖屏)
  > 如果用户主动销毁Activity，如：按下返回键，或者调用了finish()方法销毁activity，则onSaveInstanceState不会被调用
  > onPause  -> onStop ->onSaveInstanceState。
  >
  > 2）onRestoreInstanceState()调用时机
  >   当系统存在 “未经你允许” 销毁了你的的activity的时，即重建Activity时，onSaveInstanceState()方法会执行，onRestoreInstanceState()方法也会执行，否则onRestoreInstanceState()方法不会执行。
  >   比如第5种情况屏幕方向切换时，activity生命周期如下：
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
  >                startActivity(intent);
  >
  > 隐式启动可以启动其他app的activity。

- scheme使用场景,协议格式,如何使用

  > 使用场景：
  >
  > 1.服务器下发跳转路径，客户端根据服务器下发跳转路径跳转相应的页面
  >
  > 2.H5页面点击锚点，根据锚点具体跳转路径APP端跳转具体的页面
  >
  > 3.APP端收到服务器端下发的PUSH通知栏消息，根据消息的点击跳转路径跳转相关页面
  >
  > 4.APP根据URL跳转到另外一个APP指定页面
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