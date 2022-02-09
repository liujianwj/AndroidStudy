重要点：

1、生命周期感应：active = { start、resume}

2、粘性事件： observer.mLastVersion >= mVersion 为false

3、数据倒灌 ：observer.mLastVersion >= mVersion

4、postValue连续调用，前面的事件可能会丢失

### 基本用法

```kotlin
//1、初始化liveData，MutableLiveData为LiveData的装饰类，将复杂逻辑隐藏，只提供用户关心的postValue、setValue
object Event{
    val liveData: MutableLiveData<String> by lazy {
        MutableLiveData<String>()
    }
}

//2、注册监听
//在实现LifecycleOwner的对象中注册监听（有生命周期感知能力）
Event.liveData.observe(this, object: Observer<String>{
            override fun onChanged(t: String?) {
                //do something
            }
})
//无生命周期感知能力，等价于handler
Event.liveData.observeForever(object: Observer<String>{
            override fun onChanged(t: String?) {
                //do something
            }
})

//3、发送事件
Event.liveData.value = "a"   //只能在主线程
Event.liveData.postValue("a") //可以在子线程，内部还是用切换到主线程，然后调用setValue
```



### 源码分析

首先分析注册流程，入口observe()

```java
//1----LiveData
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    //如果当前观察者环境生命周期为DESTROYED，直接返回
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
      // ignore
      return;
    }
    //LifecycleBoundObserver装饰，处理主要逻辑
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    //添加到mObservers集合
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
      throw new IllegalArgumentException("Cannot add the same observer"
                                         + " with different lifecycles");
    }
    if (existing != null) {
      return;
    }
    //注册lifecycle监听
    owner.getLifecycle().addObserver(wrapper);
}

//2---LifecycleBoundObserver
//LifecycleBoundObserver实现了LifecycleEventObserver，注册lifecycle监听后，便可以监听生命周期方法
@Override
public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
    //当DESTROYED，直接移除监听
    if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
      removeObserver(mObserver);
      return;
    }
    //如果当前是active=true,进行事件分发
    activeStateChanged(shouldBeActive());
}
@Override
boolean shouldBeActive() {
    //active=true 对应 STARTED、RESUME
    return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
}
void activeStateChanged(boolean newActive) {
      if (newActive == mActive) {
        return;
      }
      // immediately set active state, so we'd never dispatch anything to inactive
      // owner
      mActive = newActive;
      boolean wasInactive = LiveData.this.mActiveCount == 0;
      LiveData.this.mActiveCount += mActive ? 1 : -1;
      if (wasInactive && mActive) {
        onActive();
      }
      if (LiveData.this.mActiveCount == 0 && !mActive) {
        onInactive();
      }
      //当mActive=true时，分发事件
      if (mActive) {
        dispatchingValue(this);
      }
}
```

事件发送流程

```java
//LiveData
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

//在onStateChanged监听到mActive=true时，initiator！= null，即initiator为mActive的observer
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
      mDispatchInvalidated = true;
      return;
    }
    mDispatchingValue = true;
    do {
      mDispatchInvalidated = false;
      //mActive=true的observer会走这里
      if (initiator != null) {
        considerNotify(initiator);
        initiator = null;
      } else { //setVaule/postValue时触发这里
        //遍历观察者列表，一一通知
        for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
             mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
          considerNotify(iterator.next().getValue());
          if (mDispatchInvalidated) {
            break;
          }
        }
      }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}


private void considerNotify(ObserverWrapper observer) {
      if (!observer.mActive) {
        return;
      }
      // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
      //
      // we still first check observer.active to keep it as the entrance for events. So even if
      // the observer moved to an active state, if we've not received that event, we better not
      // notify for a more predictable notification order.
      if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
      }
      //粘性事件原因
      if (observer.mLastVersion >= mVersion) {
        return;
      }
      observer.mLastVersion = mVersion;
      //noinspection unchecked
      observer.mObserver.onChanged((T) mData);
}
```



问题分析：

1、生命周期感应

   A : 通过注册Lifecycle监听，使其拥有感知生命周期方法，这样就比以前的handler处理事件更为高级，事件只会在mActive = true时分发，当DESTROYED时自动移除。

2、粘性事件

一般我们都是先注册监听，然后发送事件。如果反过来，先发送事件，然后在注册监听时，会收到上一个发送的事件，这个称为粘性事件。主要原因是observer.mLastVersion >= mVersion = false，mLastVersion和mVersion初始值都是-1，当先执行setValue时，mVersion++，mVersion = 0，然后我们在onCreate中，添加observer监听，observer.mLastVersion = -1，然后生命周期方法到onStart->o nResule，观察到状态为mActive，且observer.mLastVersion >= mVersion = false，所有会将上次一次mData分发。

3、数据倒灌 https://www.jianshu.com/p/cb6a80865805

我们在MVVM架构中，会在viewModel中使用livedata处理UI与数据的通信，此时会有一个问题，在页面重建时，LiveData自动推送最后一次数据。比如说，我们屏幕旋转后，LiveData会自动推送最后一次数据。根据粘性事件的分析，也是因为observer.mLastVersion >= mVersion造成的。主要是viewModel在屏幕旋转前，会将自己保存到ActivityClientRecord中，所以当页面重建时，mVersion可能大于-1，而observer在页面中新建，observer.mLastVersion = -1，即导致数据倒灌。



#### postValue连续调用，前面的事件可能会丢失问题

看postValue源码：

```java
protected void postValue(T value) {
        boolean postTask;
        synchronized (mDataLock) {
            //mPendingData没被设置value，即mPendingData == NOT_SET时，才能分发事件
            postTask = mPendingData == NOT_SET;
            mPendingData = value;
        }
        if (!postTask) {
            return;
        }
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
        @Override
        public void run() {
            Object newValue;
            synchronized (mDataLock) {
                newValue = mPendingData;
                //将mPendingData置回NOT_SET
                mPendingData = NOT_SET;
            }
            //noinspection unchecked
            setValue((T) newValue);
        }
};
```

根据源码我们可以知道主要是mPendingData标志位的原因，mPendingData在初始时，以及Runnable执行时，会被置回NOT_SET，这样在postValue时，可以发送消息，发送的时候会将value设置给mPendingData，所以在前一个消息没处理的时候，后一个消息进来直接return，而newValue = mPendingData，消息每次进来会mPendingData = value，这样newValue就被后面的消息覆盖了，所以说Runnable是执行的前面的，但是newValue使用的时候最后的。

