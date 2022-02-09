1.View绘制流程

​     ActivityThread->performResume->mWindow.addView->mGlobal.addView->new ViewRootImpl->setView->request->shecherTrachs->choreography.post->runable->performTrachs->onMer

2.MeasureSpec是什么

   MeasureSpec代表的是一个32位的int值，高两位对应的是SpecMode，指测量模式，

   低30位对应的是SpecSize，指的是某中测量模式下的大小 。

​    SpecMode：精准模式（EXACTLY）、AT_MOST、UNSPECIFIED

3.子View创建MeasureSpec创建规则是什么

​    结合SpecMode + childDimension (child.getLayoutParams())，准确值，LayoutParams.MATCH_PARENT，LayoutParams.WRAP_CONTENT)

4.自定义View wrap_content不起作用的原因

   受SpecMode影响

5.在Activity中获取某个View的宽高有几种方法

- onWindowFocusChanges

  当Activity的窗口得到焦点和失去焦点时均会被调用一次

  如果频繁的进行onResume和onPause，那么onWindowFocusChanged也会被频繁调用

- ViewThreeObserve

  当view树的状态发生改变或者view树内部的view可见发生变化时，onGlobalLayout方法将被回调

- post

  通过post可以将一个runnable投递到[消息队列](https://so.csdn.net/so/search?q=消息队列)的尾部,然后等待Looper调用次Runnable的时候,View已经初始化好了

6.为什么onCreate获取不到View的宽高

  view的测量是在activityThread.performResume中开始执行的

7.View#post与Handler#post的区别

   view.post是在view执行完之后，有消息屏障处理

8.Android绘制和屏幕刷新机制原理

9.Choreography原理

  https://ljd1996.github.io/2020/09/07/Android-Choreographer%E5%8E%9F%E7%90%86/

10.什么是双缓冲

   解决gpu处理的时候，cpu不会闲置，可以同时执行，然后再交换数据，渲染到屏幕上，避免出现卡顿

11.为什么使用SurfaceView

12.什么是SurfaceView

13.View和SurfaceView的区别

14.SurfaceView为什么可以直接子线程绘制

   view刷新到时候会调用checkThread()，SurfaceView不会

15.SurfaceView、TextureView、SurfaceTexture、GLSurfaceView

16.getWidth()方法和getMeasureWidth()区别

​    getWidth()：right-left，是layout()后的值，view布局后的宽度

   getMeasureWidth()：view根据类容测量后的宽高 

17.invalidate() 和 postInvalidate() 的区别

​    postInvalidate（）允许在子线程执行，handler.post到主线程执行invalidate

18.Requestlayout，onlayout，onDraw，DrawChild区别与联系

19.LinearLayout、FrameLayout 和 RelativeLayout 哪个效率高

20.LinearLayout的绘制流程

21.自定义 View 的流程和注意事项

22.自定义View如何考虑机型适配

23.自定义控件优化方案

24.invalidate怎么局部刷新

25.View加载流程（setContentView）