1. RecyclerView的多级缓存机制,每一级缓存具体作用是什么,分别在什么场景下会用到哪些缓存

   四级缓存：

   mAttachedScrap/mChangedScrap：屏幕内缓存，和动画相关

   mCacheViews：屏幕外缓存，当item刚滑出屏幕时缓存，再滑回来的时候，直接复用，不需再绑定数据

   mViewCacheExtension：回收和复用都需要自己处理

   RecycleViewPool：缓存池，根据viewType存储，每个type下默认大小5个，都经过漂白，数据需要重新绑定。

2. RecyclerView的滑动回收复用机制

3. RecyclerView的刷新回收复用机制

4. RecyclerView 为什么要预布局

   https://www.jianshu.com/p/b89bee49f1c0

   这个问题其实我们可以先阅读完下面的代码 然后再来回想这个问题 所以我先给出结论
    首先预布局就是先布局一次(🤣凑个字数) 然后会形成一个快照(`pre-layout`) 如`item1234` 然后再布局一次(`post-layout`) 形成另外一张快照 `item134` 这样我们其实就知道了整个动画轨迹 就可以生成动画
    上面看不懂 没关系 看下图👇

   <img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/WeChat9a090470c6ecd755e8dc47feb3bad7c7.png" alt="WeChat9a090470c6ecd755e8dc47feb3bad7c7" style="zoom:50%;" />

   下面分别是三种状态

   - 初始状态
   - pre-layout(预布局阶段 生成一个快照 详细代码我们会在下面分析)
   - post-layout(布局阶段 生成另一个快照)

   然后我们就知道了item3的初始位置和终止位置 就可以生成动画并执行

5. ListView 与 RecyclerView区别

6. RecyclerView性能优化

