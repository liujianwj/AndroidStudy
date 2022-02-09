<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220121112829013.png" alt="image-20220121112829013" style="zoom:50%;" />

### 源码分析

在Androidx时，AppCompatActivity的继承关系是，AppCompatActivity->FragmentActivity->androidx.activity.ComponentActivity->androidx.core.app.ComponentActivity->Activity，而在ComponentActivity实现了LifecycleOwner，成为了被观察者，我们先分析观察者和被观察者是如何关联的：

```java 
//实现称为被观察者
public interface LifecycleOwner {
  
    //与观察者绑定，事件分发都在Lifecycle中执行
    @NonNull
    Lifecycle getLifecycle();
}

//ComponentActivity中默认实现LifecycleOwner
ComponentActivity implements LifecycleOwner
  //实现getLifecycle方法，返回new LifecycleRegistry(this)
  ->getLifecycle()
  ->onCreate()
     //跟Glide一样，通过添加一个无UIFragment去监听activity生命周期
     //为什么不直接在activity生命周期方法里面监听？-》封装，便于维护，以及迁移，职责分明
     ->ReportFragment.injectIfNeededIn(this)


  //当我们需要监听Activity生命周期方法时，只需自己实现一个LifecycleObserver观察者类
  class MyLifecycleObserver : LifecycleEventObserver {
    override fun onStateChanged(lifecycleOwner: LifecycleOwner, event: Lifecycle.Event) {
        when(event){
            Lifecycle.Event.ON_CREATE-> Log.d("LifecycleEventObserver", "ON_CREATE")
            Lifecycle.Event.ON_START-> Log.d("LifecycleEventObserver", "ON_START")
            Lifecycle.Event.ON_RESUME-> Log.d("LifecycleEventObserver", "ON_RESUME")
            Lifecycle.Event.ON_PAUSE-> Log.d("LifecycleEventObserver", "ON_RESUME")
            Lifecycle.Event.ON_STOP-> Log.d("LifecycleEventObserver", "ON_RESUME")
            Lifecycle.Event.ON_DESTROY-> Log.d("LifecycleEventObserver", "ON_RESUME")
        }
    }
 }

 //然后在activity中添加监听
getLifecycle().addObserver(new MyLifecycleObserver())

//在LifecycleRegistry中进行绑定
public void addObserver(@NonNull LifecycleObserver observer) {
        this.enforceMainThreadIfNeeded("addObserver");
        //初始化状态为State.INITIALIZED
        State initialState = this.mState == State.DESTROYED ? State.DESTROYED : State.INITIALIZED;
        //将observer，state包装成ObserverWithState
        LifecycleRegistry.ObserverWithState statefulObserver = new LifecycleRegistry.ObserverWithState(observer, initialState);
        //将ObserverWithState添加进map存储
        LifecycleRegistry.ObserverWithState previous = (LifecycleRegistry.ObserverWithState)this.mObserverMap.putIfAbsent(observer, statefulObserver);
        //如果之前map中已经存在，则返回previous不为null，需用处理state，与被观察者状态对齐
        if (previous == null) {
            //获取被观察者，mLifecycleOwner为WeakReference<LifecycleOwner>类型，在LifecycleRegistry中将LifecycleOwner传入
            LifecycleOwner lifecycleOwner = (LifecycleOwner)this.mLifecycleOwner.get();
            if (lifecycleOwner != null) {
                boolean isReentrance = this.mAddingObserverCounter != 0 || this.mHandlingEvent;
                //计算需要跳转到的targetState
                State targetState = this.calculateTargetState(observer);
                ++this.mAddingObserverCounter;
                //小于目标状态，需要前进
                while(statefulObserver.mState.compareTo(targetState) < 0 && this.mObserverMap.contains(observer)) {
                    this.pushParentState(statefulObserver.mState);
                    //获取对应的event
                    Event event = Event.upFrom(statefulObserver.mState);
                    if (event == null) {
                        throw new IllegalStateException("no event up from " + statefulObserver.mState);
                    }
                    //分发事件给观察者，后续分析
                    statefulObserver.dispatchEvent(lifecycleOwner, event);
                    this.popParentState();
                    //继续计算需要跳转到的targetState
                    targetState = this.calculateTargetState(observer);
                }

                if (!isReentrance) {
                    this.sync();
                }

                --this.mAddingObserverCounter;
            }
        }
    }
```

接着我们分析下事件监听然后通知到观察者的流程：

```java
//ReportFragment在生命周期方法中对事件进行分发
ReportFragment
  ->onActivityCreated()
     ->dispatch(Event.ON_CREATE)
  ->onStart()
     ->dispatch(Event.ON_START)
  ->onResume
     ->dispatch(Event.ON_RESUME)
  ->onPause
     ->dispatch(Event.ON_PAUSE)
  ->onStop
     ->dispatch(Event.ON_STOP)
  ->onDestroy
     ->dispatch(Event.ON_DESTROY)
  -》dispatch(Activity activity, Event even)
     ->LifecycleRegistry lifecycle = ((LifecycleOwner)activity).getLifecycle()
     ->lifecycle.handleLifecycleEvent(event)
  
LifecycleRegistry.handleLifecycleEvent(event)
  //通过event获取对应的state
  ->State next = getStateAfter(event)
  //同步state
  ->moveToState(event.getTargetState())
    //同步自己的state
    ->this.mState = next
    //同步观察者的state
    ->sync()
       //if (this.mState.compareTo(((LifecycleRegistry.ObserverWithState)this.mObserverMap.eldest().getValue()).mState) < 0),  状态对比，需要后退状态
       ->backwardPass(lifecycleOwner)
       //if (!this.mNewEventOccurred && newest != null && this.mState.compareTo(((LifecycleRegistry.ObserverWithState)newest.getValue()).mState) > 0)，状态对比，需要前进状态
       ->forwardPass(lifecycleOwner)

//前进和后退逻辑差不多
private void forwardPass(LifecycleOwner lifecycleOwner) {
        //获取mObserverMap迭代器，mObserverMap存储的都是观察者
        IteratorWithAdditions ascendingIterator = this.mObserverMap.iteratorWithAdditions();
        //遍历一一同步状态
        while(ascendingIterator.hasNext() && !this.mNewEventOccurred) {
            Entry<LifecycleObserver, LifecycleRegistry.ObserverWithState> entry = (Entry)ascendingIterator.next();
            LifecycleRegistry.ObserverWithState observer = (LifecycleRegistry.ObserverWithState)entry.getValue();

            while(observer.mState.compareTo(this.mState) < 0 && !this.mNewEventOccurred && this.mObserverMap.contains(entry.getKey())) {
                this.pushParentState(observer.mState);
                //分发对应状态的事件
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                this.popParentState();
            }
        }

    }

static class ObserverWithState {
        State mState;
        LifecycleEventObserver mLifecycleObserver;

        ObserverWithState(LifecycleObserver observer, State initialState) {
            this.mLifecycleObserver = Lifecycling.lifecycleEventObserver(observer);
            this.mState = initialState;
        }

        void dispatchEvent(LifecycleOwner owner, Event event) {
            //根据event获取目标State
            State newState = LifecycleRegistry.getStateAfter(event);
            this.mState = LifecycleRegistry.min(this.mState, newState);
            //调用onStateChanged方法，即我们自定义的MyLifecycleObserver里面的方法（有些监听者需要通过反色射调用其监听方法）
            this.mLifecycleObserver.onStateChanged(owner, event);
            this.mState = newState;
        }
    }
```



#### event、state状态对齐问题：

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220121145105432.png" alt="image-20220121145105432" style="zoom:50%;" />

首先状态变化有两个方向，一个是往前推进，一个是往后

往前：onCreate->onStart->onResume

往后：onPause->onStop->onDestory

 状态分被观察者状态、观察者状态，推动流程如下：

被观察者生命周期方法--->分发event---->被观察者state对齐---->同步观察者state，获取目标state----->对齐观察者state，通过目标state获取eventc触发监听

```java
//通过event获取对应state    
static State getStateAfter(Event event) {
        switch(event) {
        case ON_CREATE: //推进
        case ON_STOP: //后退
            return State.CREATED;
        case ON_START:  //推进
        case ON_PAUSE: //后退
            return State.STARTED;
        case ON_RESUME: //推进
            return State.RESUMED;
        case ON_DESTROY: //后退
            return State.DESTROYED;
        case ON_ANY:
        default:
            throw new IllegalArgumentException("Unexpected event value " + event);
        }
    }

    //往后对齐，，例如当前状态是STARTED，需要往后回退到CREATED状态，则需要触发ON_STOP事件
    private static Event downEvent(State state) {
        switch(state) {
        case INITIALIZED:
            throw new IllegalArgumentException();
        case CREATED:
            return Event.ON_DESTROY;
        case STARTED:
            return Event.ON_STOP;
        case RESUMED:
            return Event.ON_PAUSE;
        case DESTROYED:
            throw new IllegalArgumentException();
        default:
            throw new IllegalArgumentException("Unexpected state value " + state);
        }
    }

    //往前对齐，，，例如当前状态是STARTED，需要前进到RESUMED状态，则需要触发ON_RESUME事件
    private static Event upEvent(State state) {
        switch(state) {
        case INITIALIZED:
        case DESTROYED:
            return Event.ON_CREATE;
        case CREATED:
            return Event.ON_START;
        case STARTED:
            return Event.ON_RESUME;
        case RESUMED:
            throw new IllegalArgumentException();
        default:
            throw new IllegalArgumentException("Unexpected state value " + state);
        }
    }
```

Q: 为什么不直接改变观察者状态，调用观察对应方法？

A: 因为LiveData等框架都是基于Lifecycle获取生命周期感知能力的，所有被观察需要自己保存生命周期方法，方便获取当前状态与对齐。