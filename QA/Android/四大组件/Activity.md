#### 1. Activity的启动流程
   https://blog.csdn.net/u012267215/article/details/91406211
   - Activity的启动流程都是通过startActivityForResult，startActivity也是，只是requestCode默认为-1

   - 进入到Instrumentation（Instrumentation类似于管家，听从Activity命令，执行各种生命周期方法，完成Binder通信）中的execStartActivity方法，这里会调用ActivityTaskManager.getService().startActivity，ActivityTaskManager.getService()返回的是IBinder对象，与系统进程AMS通信

   - AMS接收到信息后会保存activity信息，然后发送onPause通知上一个activity暂停，如果是通过launcher启动，则是通知launcher组件

   - 接收到onPause信息后，会通过handler消息发送到ActivityThread，执行performActivityPause，然后通知AMS已经执行完onPause 

   - AMS接收到通知后，会继续执行activity启动，此时会检测Activity所属的进程是否存在

     1）不存在的话，则首先启动进程，及通过socket向zygote进程发送通知fork出一个新的进程，并执行ActivityThread的main方法，然后通过binder向ASM发送通知，ASM接收到通知后执行初始化操作，并检测当前进程是否有需要创建的activity，有则执行创建

     2）不存在的话，则直接执行创建
     
   - AMS将执行创建Activity的通知到ActivityThread，通过反射创建Activity对象，并执行onCreate、onStart、onResume方法
     
   - ActivityThread执行完onResume方法，通知AMS，开始执行上一个Activity的onStop方法
     
   - ActivityThread执行真正的onStop方法   

#### 2.  onSaveInstanceState(),onRestoreInstanceState的调用时

 https://blog.csdn.net/fenggering/article/details/53907654

​      Activity中为我们提供了onSaveInstanceState()方法，用于保证在系统回收Activity前被调用，以此来解决被回收数据丢失的问题，并通过onRestoreInstanceState()方法恢复数据。
​      onSaveInstanceState()和onRestoreInstanceState()并不是成对出现，也就是说onSaveInstanceState()调用了并不能保证onRestoreInstanceState()方法调用，但onRestoreInstanceState()调用了则页面必然被系统回收了，onSaveInstanceState()肯定被调用了。
​    1）**onSaveInstanceState()肯定被执行时机**
​      1、当用户按下HOME键时
​      2、从最近应用中选择运行其他的程序时
​      3、按下电源按键（关闭屏幕显示）时
​      4、从当前activity启动一个新的activity时
​      5、屏幕方向切换时(竖屏切横屏或横屏切竖屏)
​     如果用户主动销毁Activity，如：按下返回键，或者调用了finish()方法销毁activity，则onSaveInstanceState不会被调用
onPause  -> onStop ->onSaveInstanceState。
  2）**onRestoreInstanceState()调用时机**
​     当系统存在 “未经你允许” 销毁了你的的activity的时，即重建Activity时，onSaveInstanceState()方法会执行，onRestoreInstanceState()方法也会执行，否则onRestoreInstanceState()方法不会执行。
​     比如第5种情况屏幕方向切换时，activity生命周期如下：
onPause -> onStop ->onSaveInstanceState -> onDestroy -> onCreate -> onStart -> onRestoreInstanceState -> onResume
在这里onRestoreInstanceState被调用，是因为屏幕切换时原来的activity确实被系统回收了，又重新创建了一个新的activity

#### 3. activity的启动模式和使用场景

​    https://blog.csdn.net/sinat_14849739/article/details/78072401

   Activity的启动模式有四种：**standard、singleTop、singleTask和singleInstance**

- **standard：标准模式**

  标准模式下，只要启动一次Activity，系统就会在当前任务栈新建一个实例。正常的去打开一个新的页面，这种启动模式使用最多，最普通。

- **singleTop：栈顶复用模式**

  这种启动模式下，如果要启动的Activity已经处于栈的顶部，那么此时系统不会创建新的实例，而是直接打开此页面，同时它的onNewIntent()方法会被执行，我们可以通过Intent进行传值，而且它的onCreate()，onStart()方法不会被调用，因为它并没有发生任何变化。

  特点：
  1、当前栈中已有该Activity的实例并且该实例位于栈顶时，不会创建实例，而是复用栈顶的实例，并且会将Intent对象传入，回调onNewInten()方法；
  2、当前栈中已有该Activity的实例但是该实例不在栈顶时，其行为和standard启动模式一样，依然会创建一个新的实例；
  3、当前栈中不存在该Activity的实例时，其行为同standard启动模式。

  使用场景：
  这种模式应用场景的话，假如一个新闻客户端，在通知栏收到了3条推送，点击每一条推送会打开新闻的详情页，如果为默认的启动模式的话，点击一次打开一个页面，会打开三个详情页，这肯定是不合理的。如果启动模式设置为singleTop，当点击第一条推送后，新闻详情页已经处于栈顶，当我们第二条和第三条推送的时候，只需要通过Intent传入相应的内容即可，并不会重新打开新的页面，这样就可以避免重复打开页面了。

- **singleTask：站内复用模式**

  在这个模式下，如果栈中存在这个Activity的实例就会复用这个Activity，不管它是否位于栈顶，复用时，会将它上面的Activity全部出栈，因为singleTask本身自带clearTop这种功能。并且会回调该实例的onNewIntent()方法。其实这个过程还存在一个任务栈的匹配，因为这个模式启动时，会在自己需要的任务栈中寻找实例，这个任务栈就是通过taskAffinity属性指定。如果这个任务栈不存在，则会创建这个任务栈。不设置taskAffinity属性的话，默认为应用的包名。
  
  特点：
  在复用的时候，首先会根据taskAffinity去找对应的任务栈：
  1、如果不存在指定的任务栈，系统会新建对应的任务栈，并新建Activity实例压入栈中。
  2、如果存在指定的任务栈，则会查找该任务栈中是否存在该Activity实例
        a、如果不存在该实例，则会在该任务栈中新建Activity实例。
        b、如果存在该实例，则会直接引用，并且回调该实例的onNewIntent()方法。并且任务栈中该实例之上的Activity会被全部销毁。

  使用场景：
  SingleTask这种启动模式最常使用的就是一个APP的首页，因为一般为一个APP的第一个页面，且长时间保留在栈中，所以最适合设置singleTask启动模式来复用。
  
- **singleInstance：单实例模式**

  单实例模式，顾名思义，只有一个实例。该模式具备singleTask模式的所有特性外，与它的区别就是，这种模式下的Activity会单独占用一个Task栈，具有全局唯一性，即整个系统中就这么一个实例，由于栈内复用的特性，后续的请求均不会创建新的Activity实例，除非这个特殊的任务栈被销毁了。以singleInstance模式启动的Activity在整个系统中是单例的，如果在启动这样的Activiyt时，已经存在了一个实例，那么会把它所在的任务调度到前台，重用这个实例。
  
  特点：
  启动该模式Activity的时候，会查找系统中是否存在：
  1、不存在，首先会新建一个任务栈，其次创建该Activity实例。
  2、存在，则会直接引用该实例，并且回调onNewIntent()方法。
  特殊情况：该任务栈或该实例被销毁，系统会重新创建。

  使用场景：
  很常见的是，电话拨号盘页面，通过自己的应用或者其他应用打开拨打电话页面 ，只要系统的栈中存在该实例，那么就会直接调用。

4. Activity A跳转Activity B，再按返回键，生命周期执行的顺序

   onPause->onCreate->onStart->onResume->onStop (->onSaveInstanceState)

   onPause->onRestart->onResume->onStop->onDestory

5. 横竖屏切换,按home键,按返回键,锁屏与解锁屏幕,跳转透明Activity界面,启动一个 Theme 为 Dialog 的 Activity，弹出Dialog时Activity的生命周期

6. onStart 和 onResume、onPause 和 onStop 的区别

   onStart是activity从不可见到可见，onResume是activity获取用户焦点，可进行交互；

   onPause是失去用户焦点，不可交互，但此时可见或部分可见，onStop不可见。

7. Activity之间传递数据的方式Intent是否有大小限制，如果传递的数据量偏大，有哪些方案

   有的.

   一. 限制传递数据量

   二. 改变传输数据方式（參见[Activity之间传递数据的方式](http://blog.csdn.net/rflyee/article/details/47431633)）

   \1. 静态static

   \2. 单例

   \3. Application

   \4. 持久化

8. Activity的onNewIntent()方法什么时候会执行

   当Activity的launchMode为singleTask的时候，通过Intent启到一个Activity，如果系统已经存在一个实例，系统就会将请求发送到这个实例上，但这个时候，系统就不会再调用通常情况下我们处理请求数据的onCreate方法，而是调用onNewIntent方法。这时的Activity执行的生命周期为：onNewIntent()——>onRestart()——>onStart()——>onResume();

   当然也不要忘记，系统可能会随时杀掉后台运行的Activity，如果这一切发生，那么系统就会调用onCreate方法，而不调用onNewIntent方法，一个好的解决方法就是在onCreate和onNewIntent方法中调用同一个处理数据的方法。

9. 显示启动和隐式启动

10. scheme使用场景,协议格式,如何使用

11. ANR 的四种场景

12. onCreate和onRestoreInstance方法中恢复数据时的区别

    onCreate中的bundle可能为空，onRestoreInstance中的一定不为空

13. activty间传递数据的方式

14. 跨App启动Activity的方式,注意事项

15. Activity任务栈是什么

16. 有哪些Activity常用的标记位Flags

    **FLAG_ ACTIVITY_NEW_TASK：**这个标记位的作用是指定Activity启动模式为”singleTask“，其作用等同于在AndroidManifest中指定android:launchMode="singleTask"相同。

    **FLAG_ACTIVITY_SINGLE_TOP：**这个标记位的作用是指定Activity启动模式为”singleTop“，其作用等同于在AndroidManifest中指定android:launchMode="singleTop"相同。

    **FLAG_ACTIVITY_CLEAR_TOP：**具有此标记位的Activity，当它启动的时候，在同一个任务栈中所有位于它上面的Activity都要出栈。这个模式一般需要和FLAG_ ACTIVITY_NEW_TASK配合使用，在这种情况下，如果调用的Activity的实例已存在，那么系统就会回调方法onNewIntent()。
    如果被启动的Activity采用的是standard，那么它连同它之上的Activity都要出栈，系统会重新创建Activity实例并放入栈顶。singleTask模式其实默认就具有此标记位的效果。

    **FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS：**具有这个标记的Activity不会出现在历史Activity的列表中，当某些情况我们不希望用户通过历史列表回到我们这个Activity的时候，这个标记将起作用。它等同于在AndroidManifest中为Activity添加属性：android:excludeFromRecents=“true”。

17. Activity的数据是怎么保存的,进程被Kill后,保存的数据怎么恢复的


