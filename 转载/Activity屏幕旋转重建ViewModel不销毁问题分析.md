Activity屏幕旋转重建ViewModel不销毁问题分析
ViewModel是数据与页面解耦的一大利器，使用ViewModel无须关系其与Activity的生命周期引起的问题，在Activity生命周期的尽头会自动移除掉ViewModel，切断与Activity的联系，即使后续有ViewModel的事件传递过来，也无法到到Activity页面，不会引起任何bug

疑惑？？？
但是在某种情况下，却出现这样一个情况！Activity配置横竖屏自动切换后，当Activity屏幕切换时，其生命周期已经走到尽头，onDestroy方法已执行，新的Activity却仍然使用之前的ViewModel，这是为什么呢？与前面的矛盾了呢？
如下图所示：

第一个红框是旋转之前的ViewModel和Activity，第二个红框是执行了onDestroy方法后，屏幕旋转后的新的ViewModel和Activity，ViewModel没变，Activity变了，那ViewModel为什么没有被销毁呢？

先看看ViewModel与Activity之间的关系

以上是ViewModel的存储路线图，以键值对存储在HashMap数据模型中，而这个Map数据模型存储在ViewModelStore内，而ComponentActivity持有ViewModelStore引用，这个ComponentActivity就是我们编写Activity时会继承；我们都知道创建ViewModel一般会用以下方法:

model = new ViewModelProvider(this).get(NotifyDeModel.class);
1
以上代码不具体分析了，感兴趣的自行去查看源码！大致意思会先去ComponentActivity的ViewModelStore内的HashMap去查找，如果找到就直接复用，反之则会利用反射创建一个ViewModel并保存到HashMap中去，以后需要时复用即可！
但是，注意ComponentActivity这一段代码：

getLifecycle().addObserver(new LifecycleEventObserver() {
    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,@NonNull Lifecycle.Event event) {
        if (event == Lifecycle.Event.ON_DESTROY) {
            if (!isChangingConfigurations()) {
                getViewModelStore()返回的是ViewModelStore
                getViewModelStore().clear();
            }
        }
    }
});

也就是Activity销毁时，执行onDestroy会自动调用ViewModelStore的clear方法清除掉ViewModel，这也就是我们不用担心ViewModel生命周期的问题，因为Activity会自动帮我们处理掉；按照正常来说，一旦清除掉，ViewModel和ViewModelStore都会被JVM垃圾回收掉，下次Activity启动后将使用新的实例；

答案是无情的！横竖屏切换，onDestroy也执行了，ViewModel和ViewModelStore却沿用了切换之前的实例

ViewModelStore由来
既然ViewModelStore仍然是沿用之前的，那我们看看ComponentActivity里面的ViewModelStore是如何被赋值的，查看代码：

public ViewModelStore getViewModelStore() {
    ...
    if (mViewModelStore == null) {
    	既然ViewModelStore复用了，那么getLastNonConfigurationInstance返回肯定不为空
        NonConfigurationInstances nc =
                (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // Restore the ViewModelStore from NonConfigurationInstances
            mViewModelStore = nc.viewModelStore;
        }
        因为ViewModelStore复用了之前的，所以这里无法执行
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
    return mViewModelStore;
}

public Object getLastNonConfigurationInstance() {
    简单返回了mLastNonConfigurationInstances成员，为了满足复用条件，这个肯定不为null
    return mLastNonConfigurationInstances != null
            ? mLastNonConfigurationInstances.activity : null;
}

那么这个mLastNonConfigurationInstances是在什么地方赋值的呢？答案就在Activity的attach方法中：

final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
	...
	mLastNonConfigurationInstances = lastNonConfigurationInstances;
	...

attach这个方法是由framework调用了，这就必须要深入源码了解了；查看何时调用这个attach方法，并且传入的这个值lastNonConfigurationInstances；

AMS framework源码分析
这种Activity旋转重建操作，都会走RELAUNCH事件，我们查看ActivityThread源码发现，大致会走如下流程

sendMessage(H.RELAUNCH_ACTIVITY, token);
Handler收到RELAUNCH_ACTIVITY事件后，执行handleRelaunchActivityLocally(token)方法，会走到ClientTransaction
ClientTransaction一套交互逻辑完成后，最后又走回到ActivityThread的handleLaunchActivity(ActivityClientRecord)方法
在执行performLaunchActivity(ActivityClientRecord)，其内部就会调用attach方法
上述部分函数执行时，其函数参数由token逐渐转换到ActivityClientRecord，熟悉AMS原理的都知道，ActivityThread有一个ArrayMap类型的成员，key就是token，value就是ActivityClientRecord，而这个ActivityClientRecord就是保存了Activity需要的参数，当然也包含lastNonConfigurationInstances这个参数，看看performLaunchActivity函数就知道了：

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    Activity activity = null;
        try {
            获取classLoader
            java.lang.ClassLoader cl = appContext.getClassLoader();
            new一个Activity，当然要根据其正确的名字
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
                        }
        ​        }
    ​    这里就是我们要查找的地方，他的lastNonConfigurationInstances来源于r
	activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

到这里基本弄清楚了，ActivityClientRecord里面就保存了lastNonConfigurationInstances，但是ActivityClientRecord的lastNonConfigurationInstances从哪里来，答案是在onDestroy之前，会保存Activity的retainNonConfigurationInstances()方法返回值，而这个retainNonConfigurationInstances()就来源于Activity的onRetainNonConfigurationInstance()方法，而onRetainNonConfigurationInstance方法就保存了NonConfigurationInstances实例，这个实例里面就保存了ViewModelStore；这里我们在看看onDestroy之前AMS Framework干了哪些事情：

ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing,
            int configChanges, boolean getNonConfigInstance, String reason) {
    获取到ActivityClientRecord
	ActivityClientRecord r = mActivities.get(token);
	看到没，保存了来自Activity的东西，它里面就包含了viewModelStore
 	r.lastNonConfigurationInstances
                            = r.activity.retainNonConfigurationInstances();
}

NonConfigurationInstances retainNonConfigurationInstances() {
		调用方法,activity包含了viewModelStore
        Object activity = onRetainNonConfigurationInstance();
        HashMap<String, Object> children = onRetainNonConfigurationChildInstances();
        FragmentManagerNonConfig fragments = mFragments.retainNestedNonConfig();
        ....
        NonConfigurationInstances nci = new NonConfigurationInstances();
        赋值
        nci.activity = activity;
        nci.children = children;
        nci.fragments = fragments;
        nci.loaders = loaders;
        return nci;
    }
public final Object onRetainNonConfigurationInstance() {
        Object custom = onRetainCustomNonConfigurationInstance();
		这个mViewModelStore成员就是Activity里面的ViewModelStore
        ViewModelStore viewModelStore = mViewModelStore;
        if (viewModelStore == null) {
            // No one called getViewModelStore(), so see if there was an existing
            // ViewModelStore from our last NonConfigurationInstance
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                viewModelStore = nc.viewModelStore;
            }
        }

        if (viewModelStore == null && custom == null) {
            return null;
        }
    	
        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.custom = custom;
        赋值并返回，保存好了
        nci.viewModelStore = viewModelStore;
        return nci;
    }

到这里，我们也就明白了为什么ViewModelStore为什么没有被清理；因为在执行onDestroy之前，从ActivityClientRecord持有一条到ViewModelStore引用链，所以当Activity被销毁时，ViewModelStore不会被垃圾回收，也就不会被销毁，而Activity的reLaunch并不会销毁对应的ActivityClientRecord，下次仍然会复用ActivityClientRecord，进而复用保存的ViewModelStore，这样就解释通了；

那么，就算ViewModelStore可以复用，但是Activity在OnDestroy事件时执行了ViewModelStore的onClear方法，清除掉内部存储ViewModel了，那这个ViewModel如何能被复用呢？看onDestroy事件

if (event == Lifecycle.Event.ON_DESTROY) {
   if (!isChangingConfigurations()) {
       getViewModelStore().clear();
   }
}

答案只有一种可能了，就是isChangingConfigurations返回true，使ViewModelStore没有执行clear方法；这句话的意思就是如果是因为配置更改导致的，比如横竖屏配置，不会去执行clear方法，这样也说得通，ViewModel保存的只是数据，界面配置变化，并不需要数据重新加载，使用之前的也可以
————————————————
版权声明：本文为CSDN博主「jackzhous_」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/jackzhouyu/article/details/109031202