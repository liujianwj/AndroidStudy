### 一、相关概念

* **View:** 视图，绘制到屏幕上的内容，如TextView，ImageView 等。
* **Window:** View的载体，对Window进行添加和删除需要通过WindowManager来操作。Window并不是真实的存在，View才是Android中的视图呈现形式，View不能单独存在，它必须依附在Window这个抽象概率上面。
* **WindowManager:** 管理系统中的Window，实际功能通过Binder IPC借由WindowManagerService实现。
* **Canvas:** 提供一些对Surface绘图对API用来进行实际的绘图操作。如果是软件绘制，其drawXXX 方法会将内容绘制到Bitmap上；如果是硬件绘制，其drawXXX方法会抽象成DrawOp操作，然后添加到DisplayList中被GPU渲染。
* **Surface:** 一个Window对应一个Surface(当存在SurfaceView则例外，Java Surface实例存在ViewRootImpl中，对应native层的surface对象)。Surface内部持有一个BufferQueueProducer指针(在Layer中创建)可以生产图像缓存区用来绘制，与App和SurfaceFlinger形成一个生产者消费者模型。
* **Layer:** App请求创建Surface时SurfaceFlinger会创建Layer对象，它是SurfaceFlinger合成的基本操作单元，因此一个Surface对应一个Layer。它创建有一个BufferQueueProducer生产者和BufferQueueConsumer消费者，这两个对象与像素数据的存储与转移相关。用户最终看到的屏幕内容是许多Layer按照z-order混合的结果。
* **Skia:** 一个2D绘图引擎，绘图步骤：定义一个位图32位像素并初始化SkBitmap bitmap->分配位图所占空间->指定输出设备->设备绘制的风格。整理大致绘图机制为：封装图形层Layer，用一个skia的bitmap来存储UI数据，用Canvas将UI绘制到bitmap上->应用可以创建多个Layer来绘制图形->要刷新时将多个Layer图形叠加到Surface上，由SurfaceFlinger来输出显示。
* **SurfaceView:** 一个较之 TextView，Button等更为特殊的View，它不与其宿主的Window共享一个Surface，而是有自己的独立Surface。并且它可以在一个独立的线程中绘制UI。因此SurfaceView一般用来实现比较复杂的图像或动画/视频的显示。
* **Choreographer:** 编舞者，用来控制当收到VSync信号后才开始绘制任务，保证绘制拥有完整的16.6ms。通常应用层不会直接使用Choreographer，而是使用更高级的API，如View.invalidate()等，可以通过Choreographer来监控应用的帧率。

