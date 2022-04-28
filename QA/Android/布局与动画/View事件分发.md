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