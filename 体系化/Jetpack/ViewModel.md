https://blog.csdn.net/jackzhouyu/article/details/109031202

作用：感应生命周期，屏幕旋转页面重建的时候，缓存数据

1、是如何做到屏幕旋转页面重建时，数据的恢复？

### 使用

```java
//在ComponentActivity子类中调用
private val downloadViewModel: DownloadViewModel by lazy {
        ViewModelProvider(this).get(DownloadViewModel::class.java)
}

//处理逻辑
class DownloadViewModel : ViewModel() {
}
```



### 源码分析

大致流程ComponentActivity实现了ViewModelStoreOwner接口：

```java
public interface ViewModelStoreOwner {
    @NonNull
    ViewModelStore getViewModelStore();
}

public class ViewModelStore {

    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }
  
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();
        }
        mMap.clear();
    }
}
```

ViewModelStoreOwner接口提供了getViewModelStore，返回ViewModelStore，ViewModelProvider初始化的时候将ComponentActivity对象传入，这样便可以获取ViewModelStore，而ViewModelStore中其实就是简单map存储着ViewModel。

在获取ViewModel时，先从ViewModelStore缓存列表中获取，获取不到则通过Factory利用反射新建ViewModel，然后放入ViewModelStore中：

```java
    public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            if (mFactory instanceof OnRequeryFactory) {
                ((OnRequeryFactory) mFactory).onRequery(viewModel);
            }
            return (T) viewModel;
        } else {
            //noinspection StatementWithEmptyBody
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
        } else {
            viewModel = (mFactory).create(modelClass);
        }
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }
```

在ComponentActivity的destory时，将viewmodel列表清空：

```java
public ComponentActivity() {
  //....省略。。。
  getLifecycle().addObserver(new LifecycleEventObserver() {
            @Override
            public void onStateChanged(@NonNull LifecycleOwner source,
                    @NonNull Lifecycle.Event event) {
                if (event == Lifecycle.Event.ON_DESTROY) {
                    if (!isChangingConfigurations()) {
                        getViewModelStore().clear();
                    }
                }
            }
        });
  //....省略。。。
}
```

此时我们会发现在清空的条件，不仅是状态为ON_DESTORY，还有!isChangingConfigurations() = true，

```java
    /**
     * Check to see whether this activity is in the process of being destroyed in order to be
     * recreated with a new configuration. This is often used in
     * {@link #onStop} to determine whether the state needs to be cleaned up or will be passed
     * on to the next instance of the activity via {@link #onRetainNonConfigurationInstance()}.
     *
     * @return If the activity is being torn down in order to be recreated with a new configuration,
     * returns true; else returns false.
     */
    public boolean isChangingConfigurations() {
        return mChangingConfigurations;
    }
```

通过注释我们知道，当一些配置信息改变，如横竖屏切换时，会走onRetainNonConfigurationInstance()方法，并且mChangingConfigurations状态会为true，也就是说当发生横竖屏切换时，ViewModelStore不会清空。但是activity会重新创建，既然数据会恢复，那ViewModelStore肯定是需要保存下来，我们可以看下ComponentActivity的onRetainNonConfigurationInstance():

```java
  public final Object onRetainNonConfigurationInstance() {
        Object custom = onRetainCustomNonConfigurationInstance();

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
        nci.viewModelStore = viewModelStore;
        return nci;
    }
```

onRetainNonConfigurationInstance中会将viewModelStore保存在NonConfigurationInstances中返回，接下来分析下onRetainNonConfigurationInstance的调用：

```java
//ActivityThread
performDestroyActivity
  ->ActivityClientRecord r = mActivities.get(token);
  -> r.lastNonConfigurationInstances = r.activity.retainNonConfigurationInstances();

//Activity
retainNonConfigurationInstances(){
  HashMap<String, Object> children = onRetainNonConfigurationChildInstances();
  NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.activity = activity;
        nci.children = children;
        nci.fragments = fragments;
        nci.loaders = loaders;
        if (mVoiceInteractor != null) {
            mVoiceInteractor.retainInstance();
            nci.voiceInteractor = mVoiceInteractor;
        }
        return nci;
}
```

所以可以发现，viewModelStore最终是保存在ActivityClientRecord的lastNonConfigurationInstances中，而ActivityClientRecord是在mActivities中获取，mActivities是ActivityThread中的成员变量，final ArrayMap<IBinder, ActivityClientRecord> mActivities = new ArrayMap<>()，跟随整个项目的生命周期，每个Activity都有对应的ActivityClientRecord，key = token

```java
handleLaunchActivity(ActivityClientRecord r)
     ->performLaunchActivity(ActivityClientRecord r, Intent customIntent)
         ->mActivities.put(r.token, r);
```

接着我们来看下ComponetActivity中getViewModelStore的实现：

```java
@NonNull
    @Override
    public ViewModelStore getViewModelStore() {
        if (getApplication() == null) {
            throw new IllegalStateException("Your activity is not yet attached to the "
                    + "Application instance. You can't request ViewModel before onCreate call.");
        }
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                // Restore the ViewModelStore from NonConfigurationInstances
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        return mViewModelStore;
    }

performLaunchActivity
  ->activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);

final void attach(...NonConfigurationInstances lastNonConfigurationInstances,...)
  ->mLastNonConfigurationInstances = lastNonConfigurationInstances;

getLastNonConfigurationInstance->mLastNonConfigurationInstances
```



个人总结：

1. ViewModel的作用是生命周期感应，可以在横竖屏切换的时候，数据自动保存，并且在生命周期结束的时候，会自动移除。
2. ViewModel累本身很简单，只是暴露clear方法，供开发在使用的时候，做一些额外的回收操作，主要的是怎么做到生命周期感应和横竖屏切换时如何保存数据和恢复数据。
3. 生命周期感应：在基类中存有一个成员变量viewmodelStore，理由有个集合，保存viewmodel。同时在activity创建的时候会向lifecycle中注册监听，当destroy且不是横竖屏切换时，会遍历viewmodelStore中的viewmodel清除。
4. 横竖屏切换： 而横竖屏切换时，会回调系统方法，onRetainNonConfigurationInstaces方法，里面会将viewmodelStore保存到nonco ngfigrurationinstace中，在系统执行performDestory时，会调用onRetainNonConfigurationInstaces，把它保存在activityClientRecode 中的lastNonConfigurationInstace中。在activity中会实现viewmo delStoreOnwer，实现里面的方法获取viewmo delStore，获取是先从lastNonConfigurationInstace中获取，获取不到再通过反射创建，并添加进viewmo delStore。