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
  >      setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
  >              getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
  > }
  > ```
  >
  > 而getDefaultSize()代码为:
  >
  > ```java
  > public static int getDefaultSize(int size, int measureSpec) {
  >  int result = size;
  >  int specMode = MeasureSpec.getMode(measureSpec);
  >  int specSize = MeasureSpec.getSize(measureSpec);
  > 
  >  switch (specMode) {
  >    case MeasureSpec.UNSPECIFIED:
  >      result = size;
  >      break;
  >    case MeasureSpec.AT_MOST:
  >    case MeasureSpec.EXACTLY:
  >      result = specSize; // 这里是问题的关键
  >      break;
  >  }
  >  return result;
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
  >      final AttachInfo attachInfo = mAttachInfo;
  >      //当attachInfo不为空，即已经执行完view.onAttachToWindow时，直接post到UI队列
  >      if (attachInfo != null) {
  >          return attachInfo.mHandler.post(action);
  >      }
  > 
  >      // Postpone the runnable until we know on which thread it needs to run.
  >      // Assume that the runnable will be successfully placed after attach.
  >      //否则先缓存到mRunQueue队列
  >      getRunQueue().post(action);
  >      return true;
  > }
  > ```
  >
  > attachInfo是在View.dispatchAttachedToWindow(AttachInfo info, int visibility)中赋值的，而dispatchAttachedToWindow的执行顺序是：
  >
  > ```java
  > // ViewRootImpl.java
  > private void performTraversals() {
  >  // 省略其他代码
  >  host.dispatchAttachedToWindow(mAttachInfo, 0);
  > 
  >  // 省略其他代码
  >  getRunQueue().executeActions(mAttachInfo.mHandler);
  > 
  >  // 省略其他代码
  >  performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
  >  // 省略其他代码
  >  performLayout(lp, mWidth, mHeight);
  >  // 省略其他代码
  >  performDraw();
  > }
  > ```
  >
  > getRunQueue().executeActions(mAttachInfo.mHandler)就是执行之前mRunQueue中的Runnable，但是我们都知道在View#post中可以获取控件的宽高，但是看代码却是在measure,layout和draw之前，所以我们要继续看下getRunQueue().executeActions(mAttachInfo.mHandler)的源码：
  >
  > ```java
  > public void executeActions(Handler handler) {
  > synchronized (this) {
  >  final HandlerAction[] actions = mActions;
  >  for (int i = 0, count = mCount; i < count; i++) {
  >    final HandlerAction handlerAction = actions[i];
  >    handler.postDelayed(handlerAction.action, handlerAction.delay);
  >  }
  > 
  >  mActions = null;
  >  mCount = 0;
  > }
  > }
  > ```
  >
  > 这里其实就是把Runnable发送到UI主线程的handler，但是此时我们的主线程handler已经在执行绘制等Runnable，并且有同步屏障的处理，所以mRunQueue中的消息总是在绘制完成之后执行，故我们可以正常获取宽高。
  >
  > https://blog.csdn.net/u014761700/article/details/79908801

- Android绘制和屏幕刷新机制原理

  > 为了保障画面的流程，需要1.6ms刷新一次屏幕，
  >
  > 屏幕刷新机制：从底层往上，屏幕渲染->

- Choreography原理

  > 底层提供刷新监听，1.6秒发送一次sync信息，，上层可以注册监听，注册后可以接受到下一次sync信息将绘制信息发送给底层渲染。

- 什么是双缓冲

  > 最开始是cpu处理好buffer，交给gpu处理，gpu处理好后再给屏幕渲染，但是此时会存在gpu处理时，cpu等待的过程，浪费cpu，而且cpu+gpu处理时间过长，未达到1.6ms刷新出下一帧，导致卡顿
  >
  > 这个时候出现双缓冲，两个buffer对象，cpu处理使用一个buffer，gpu处理使用一个buffer，gpu处理的时候，cpu可以处理下一帧，都处理完后，交换buffer，gpu处理cpu给过来的buffer，cpu接到空的buffer继续处理下一帧

- 为什么使用SurfaceView

  > 大多数情况下我们的自定义View都会选择去继承View或ViewGroup来实现，但是为什么系统还要为我们提供一个SurfaceView呢？
  > 首先我们知道View类如果需要更新视图，必须我们主动的去调用invalidate()或者postInvalidate()方法来再走一次onDraw()完成更新。但是呢，Android系统规定屏幕的刷新间隔为16ms，如果这个View在16ms内更新完毕了，就不会卡顿，但是如果逻辑操作太多，16ms内没有更新完毕，剩下的操作就会丢到下一个16ms里去完成，这样就会造成UI线程的阻塞，造成View的运动过程掉帧，自然就会卡顿了。
  > 所以这些原因也就促使了SurfaceView的存在。毕竟，如果是一个游戏，它有可能相当频繁的有更新画面的需求。
  >
  > 
  >
  > SuraceView的主要优势
  >
  > 1、SurfaceView的刷新处于主动，有利于频繁的更新画面。
  >
  > 2、SurfaceView的绘制在子线程进行，避免了UI线程的阻塞。
  >
  > 3、SurfaceView在底层实现了一个双缓冲机制，效率大大提升。
  
- 什么是SurfaceView

- View和SurfaceView的区别

  > View在主线程刷新，SurfaceView在子线程刷新
  >
  > View需要刷新时，必须调用invalidate()或者requestLayout()，通过choreography注册下一次sync信息，下一次信号到来的时候才开始绘制信息，然后等下一个信息到的时候，绘制信息才能展示在屏幕上
  >
  > 而SurfaceView可以自己往内存中写数据
  >
  > SurfaceView类的成员变量mRequestedType描述的是SurfaceView的绘图表面Surface的类型，一般来说，它的值可能等于SURFACE_TYPE_NORMAL或者SURFACE_TYPE_PUSH_BUFFERS，
  > 当一个SurfaceView的绘图表面的类型等于SURFACE_TYPE_NORMAL的时候，就表示该SurfaceView的绘图表面所使用的内存是一块普通的内存，一般来说，这块内存是由SurfaceFlinger服务来分配的，我们可以在应用程序内部自由地访问它，即可以在它上面填充任意的UI数据，然后交给SurfaceFlinger服务来合成，并且显示在屏幕上，在这种情况下，在SurfaceFlinger服务一端使用一个Layer对象来描述该SurfaceView的绘图表面。
  >
  > 当一个SurfaceView的绘图表面的类型等于SURFACE_TYPE_PUSH_BUFFERS的时候，就表示该SurfaceView的绘图表面所使用的内存不是由SurfaceFlinger服务分配的，我们不能够在应用程序内部对它进行操作，所以不能调用lockCanvas来获取Canvas对象进行绘制了，例如当一个SurfaceView是用来显示摄像头预览或者视频播放的时候，我们就会将它的绘图表面的类型设置为SURFACE_TYPE_PUSH_BUFFERS，这样摄像头服务或者视频播放服务就会为该SurfaceView绘图表面创建一块内存，并且将采集的预览图像数据或者视频帧数据源源不断地填充到该内存中去，在这种情况下，在SurfaceFlinger服务一端使用一个LayerBuffer对象来描述该SurfaceView的绘图表面。
  >

- SurfaceView为什么可以直接子线程绘制

  > 没有checkThread检测，这也是SurfaceView的优势，避免UI线程的阻塞

- SurfaceView、TextureView、SurfaceTexture、GLSurfaceView

- getWidth()方法和getMeasureWidth()区别

  > getWidth() = right - left；
  >
  > getMeasureWidth()是测量后的宽度；
  >
  > getMeasuredWidth()获取的是view原始的大小，也就是这个view在XML文件中配置或者是代码中设置的大小。getWidth（）获取的是这个view layout后最终显示的大小，这个大小有可能等于原始的大小也有可能不等于原始大小。

- invalidate() 和 postInvalidate() 的区别

  > postInvalidate可以在子线程调用，其实内部也是通过handler切换到主线程然后调用invalidate

- Requestlayout，onlayout，onDraw，DrawChild区别与联系

- LinearLayout、FrameLayout 和 RelativeLayout 哪个效率高

- LinearLayout的绘制流程

- 自定义 View 的流程和注意事项

  > - 继承View重写onDraw方法
  >
  > 主要用于实现不规则的效果，即这种效果不方便通过布局的组合方式来实现。相当于就是得自己“画”了。采用这种方式需要自己支持wrap_content，padding也需要自己处理
  >
  > - 继承ViewGroup派生特殊的Layout
  >
  > 主要用于实现自定义的布局，看起来很像几种View组合在一起的时候，可以使用这种方式。这种方式需要合适地处理ViewGroup的测量和布局，并同时处理子元素的测量和布局过程。比如自定义一个自动换行的LinerLayout等。
  >
  > - 继承特定的View，比如TextView
  >
  > 这种方法主要是用于扩展某种已有的View，增加一些特定的功能。这种方法比较简单，也不需要自己支持wrap_content和padding。
  >
  > - 继承特定的ViewGroup，比如LinearLayout
  >
  > 这种方式也比较常见，和上面的第2种方法比较类似，第2种方法更佳接近View的底层。
  >
  > > 自定义View有多种方式，需要根据实际需要选择一种简单低成本的方式来实现
  >
  > 
  >
  > 注意点：
  >
  > - 让View支持wrap_content
  > - 让View支持padding
  > - 避免在onMeasure中创建对象，容易出现内存抖动，内存碎片太多，容易oom，

- 自定义View如何考虑机型适配

  > 合理使用warp_content，match_parent.
  >
  > 尽可能的是使用RelativeLayout
  >
  > 针对不同的机型，使用不同的布局文件放在对应的目录下，android会自动匹配。
  >
  > 尽量使用点9图片。
  >
  > 使用与密度无关的像素单位dp，sp
  >
  > 引入android的百分比布局。
  >
  > 切图的时候切大分辨率的图，应用到布局当中。在小分辨率的手机上也会有很好的显示效果。

- 自定义控件优化方案

- invalidate怎么局部刷新

  > invalidate(Rect)，脏区域，失效，重绘

- View加载流程（setContentView）

