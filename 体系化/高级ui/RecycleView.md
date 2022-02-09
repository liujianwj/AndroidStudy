#### 对比listview优势

主要是解耦、功能颗粒度细，灵活度高，分割线可以自定义ItemDecoration，布局可以自定义LayoutManager

#### 四级缓存：

```java
    public final class Recycler {
        //一级缓存中用来存储屏幕中显示的ViewHolde
        final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
        ArrayList<ViewHolder> mChangedScrap = null;
        //二级缓存中用来存储屏幕外的缓存
        final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
        //暂可忽略 mAttachedScrap的不可变视图
        private final List<ViewHolder> mUnmodifiableAttachedScrap =      Collections.unmodifiableList(mAttachedScrap);
        //当前屏幕外缓存大小，数量为2
        private int mRequestedCacheMax = DEFAULT_CACHE_SIZE;
        int mViewCacheMax = DEFAULT_CACHE_SIZE;
        //四级缓存当二级缓存超过2时候启用
        RecycledViewPool mRecyclerPool;
        //三级缓存扩展类（默认不启用）
        private ViewCacheExtension mViewCacheExtension;

    }
```

1. mChangeScrap和mAttachedScrap

   用来缓存还在屏幕的ViewHolder

   mAttachedScrap：存储的是当前屏幕中的ViewHolder，mAttachedScrap的对应数据结构是ArrayList，在调用LayoutManager#onLayoutChildren方法时对views进行布局，此时会将RecyclerView上的Views全部暂存到该集合中，该缓存中的ViewHolder的特性是，如果和RV上的position或者itemId匹配上了那么可以直接拿来使用的，无需调用onBindViewHolder方法。目的：职责分明，Recycler存放ViewHolder，LayoutManager只管怎么布局，需要ViewHolder就从Recycler中获取。

   mChangedScrap和mAttachedScrap属于同一级别的缓存，不过mChangedScrap的调用场景是notifyItemChanged和notifyItemRangeChanged，只有发生变化的ViewHolder才会放入到mChangedScrap中。mChangedScrap缓存中的ViewHolder是需要调用onBindViewHolder方法重新绑定数据的。

2. mCachedViews

   mCachedViews缓存滑动时即将与RecyclerView分离的ViewHolder，按子View的position或id缓存，默认最多存放2个。mCachedViews对应的数据结构是ArrayList，但是该缓存对集合的大小是有限制的。

   该缓存中ViewHolder的特性和mAttachedScrap中的特性是一样的，只要position或者itemId对应就无需重新绑定数据。开发者可以调用setItemViewCacheSize(size)方法来改变缓存的大小，该层级缓存触发的一个常见的场景是滑动RecyclerView。当然调用notify()也会触发该缓存。

3. mViewCacheExtension

   这个的创建和缓存完全由开发者自己控制，系统未往这里添加数据

4. RecycledViewPool 

    本质上是一个SparseArray，其中key是ViewType(int类型)，value存放的是 ArrayList< ViewHolder>，mCacheViews满了的时候，从mCacheViews中移除第一个放入到对应ViewType的ArrayList< ViewHolder>，默认每个ArrayList中最多存放5个ViewHolder。

##### 复用过程（重要方法fill）

两个入口，一个是滑动时复用onTouchEvent中，一个是布局时复用onLayout中

滑动时复用：

```java
onTouchEvent@RecyclerView
  ->scrollByInternal
    ->scrollStep
       ->mLayout.scrollHorizontallyBy
       ->mLayout.scrollVerticallyBy
//横向滑动和纵向滑动复用逻辑一样，这里从纵向滑动分析，mLayout为LinearLayoutManager
LinearLayoutManager.scrollVerticallyBy
  ->scrollBy
    ->fill(recycler, this.mLayoutState, state, false)
       ->layoutChunk 
          ->layoutState.next(recycler)
             ->View view = recycler.getViewForPosition(this.mCurrentPosition)
Recycler.getViewForPosition
  //主要的复用代码
  ->tryGetViewHolderForPositionByDeadline
     ->1. getChangedScrapViewForPosition --》 mChangedScrap （position、StableId）
       2. getScrapOrHiddenOrCachedHolderForPosition --》 mAttachedScrap、mCachedViews（ForPosition）
       3. getScrapOrCachedViewForId --》 mAttachedScrap、mCachedViews（StableId）
       4. mViewCacheExtension.getViewForPositionAndType --》 自定义复用，缓存需要自己实现
       5. getRecycledViewPool().getRecycledView
       6. mAdapter.createViewHolder --> onCreateViewHolder --> 创建 ViewHolder 对象
       7. tryBindViewHolderByDeadline --> onBindViewHolder --> 处理数据
  
```



布局时复用：

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/image-20220117110349208.png" alt="image-20220117110349208" style="zoom:50%;" />

缓存过程

<img src="/Users/liujian/Documents/study/books/AndroidStudy/图片/WeChat2483378a774f8453ea7a3229c5b421dc.png" alt="WeChat2483378a774f8453ea7a3229c5b421dc" style="zoom: 33%;" />

##### 分割线ItemDecortaion（以绘制类似城市列表，顶部吸顶效果为例）

自定义分割线，继承ItemDecortaion，一般需要重写一下几个方法：

- getItemOffsets
- onDraw
- onDrawOver

getItemOffsets：预留分割线的位置

```java
    @Override
    public void getItemOffsets(@NonNull Rect outRect, @NonNull View view,
                               @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        super.getItemOffsets(outRect, view, parent, state);

        if (parent.getAdapter() instanceof StarAdapter) {
            StarAdapter adapter = (StarAdapter) parent.getAdapter();
            // 当前Item的位置
            int position = parent.getChildLayoutPosition(view);

            // 如何判断 item 是头部
            boolean isGroupHeader = adapter.isFirstItemOfGroup(position);
            // 是第一个
            if (isGroupHeader) {
                outRect.set(0, headerHeight, 0, 0);
            } else {
                // padding，margin
                outRect.set(0, 4, 0, 0);
            }
        }
    }
```



onDraw和onDrawOver区别，先来看下RecyclerView和分割线的绘制流程

```java
ReyclerView.draw 
--> super.draw(c);
	--> 绘制自己 --> ReyclerView.onDraw
		--> ItemDecoration.onDraw
	--> 绘制孩子(ItemView)
--> ItemDecoration.onDrawOver()
```

由此可见：ItemDecoration.onDraw  --》 绘制孩子(ItemView) --》 ItemDecoration.onDrawOver，后面绘制的，会覆盖前面绘制的

onDraw中绘制的，会被ItemView覆盖，onDrawOver中绘制的会覆盖ItemView。

```java
// 都是绘制 -- 绘制后的效果 -- 分割线
    @Override
    public void onDraw(@NonNull Canvas c, @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        super.onDraw(c, parent, state);

        if (parent.getAdapter() instanceof StarAdapter) {
            StarAdapter adapter = (StarAdapter) parent.getAdapter();

            // 当前屏幕上的
            int count = parent.getChildCount();

            // 实现itemView的宽度和分割线的宽度一样
            int left = parent.getPaddingLeft();
            int right = parent.getWidth() - parent.getPaddingRight();
            for (int i = 0; i < count; i++) {
                View view = parent.getChildAt(i);

                if (view.getTop() - headerHeight - parent.getPaddingTop() >= 0) {
                    // 当前Item的位置
                    int position = parent.getChildLayoutPosition(view);
                    // 如何判断 item 是头部
                    boolean isGroupHeader = adapter.isFirstItemOfGroup(position);
                    // 判断是否是头部
                    if (isGroupHeader) {
                        c.drawRect(left, view.getTop() - headerHeight, right, view.getTop(), headPaint);
                        String groupName = adapter.getGroupName(position);
                        drawTextPaint.getTextBounds(groupName, 0, groupName.length(), textRect);
                        // 绘制文字
                        c.drawText(groupName, left + 20,
                                view.getTop() - headerHeight / 2 + textRect.height() / 2, drawTextPaint);

                    } else { // 普通的itemView的分割线
                        c.drawRect(left, view.getTop() - 4, right, view.getTop(), headPaint);
                    }
                }
            }
        }
    }

    @Override
    public void onDrawOver(@NonNull Canvas c, @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        super.onDrawOver(c, parent, state);
        if (parent.getAdapter() instanceof StarAdapter) {
            StarAdapter adapter = (StarAdapter) parent.getAdapter();

            int left = parent.getPaddingLeft();
            int right = parent.getWidth() - parent.getPaddingRight();
            int top = parent.getPaddingTop();

            // 当前显示在界面的 第一个item
            int position = ((LinearLayoutManager) parent.getLayoutManager()).findFirstVisibleItemPosition();
            View itemView = parent.findViewHolderForAdapterPosition(position).itemView;

            boolean isFirstItemOfGroup = adapter.isFirstItemOfGroup(position + 1);
            if (isFirstItemOfGroup) {
                int bottom = Math.min(top + headerHeight, itemView.getBottom());
                c.drawRect(left, top, right, bottom, headOverPaint);

                String groupName = adapter.getGroupName(position);
                drawOverTextPaint.getTextBounds(groupName, 0, groupName.length(), textRect);

                c.clipRect(left, top, right, bottom);

                c.drawText(groupName, left + 20,
                        bottom - headerHeight / 2 + textRect.height() / 2, drawOverTextPaint);
            } else {// 固定的
                c.drawRect(left, top, right, top + headerHeight, headOverPaint);

                String groupName = adapter.getGroupName(position);
                drawTextPaint.getTextBounds(groupName, 0, groupName.length(), textRect);
                // 绘制文字
                c.drawText(groupName, left + 20,
                        top + headerHeight / 2 + textRect.height() / 2, drawOverTextPaint);
            }
        }
    }

```


outRect  --》 onDraw，onDrawOver 没有区别

##### 自定义LayoutManager（实现探探看好友效果）

```java
package com.leo.slidecard;

import android.view.View;
import android.view.ViewGroup;

import androidx.recyclerview.widget.RecyclerView;

public class SlideCardLayoutManager extends RecyclerView.LayoutManager {

    @Override
    public RecyclerView.LayoutParams generateDefaultLayoutParams() {
        return new RecyclerView.LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT,
                ViewGroup.LayoutParams.WRAP_CONTENT);
    }

    // 必须要重写, super里面是空实现
    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        // 回收
        detachAndScrapAttachedViews(recycler);

        // 总共的item个数：8个
        int itemCount = getItemCount();
        // 当前底部的 position
        int bottomPosition;

        // 总个数少于4个的时候
        if (itemCount < CardConfig.MAX_SHOW_COUNT) {
            bottomPosition = 0;
        } else {
            bottomPosition = itemCount - CardConfig.MAX_SHOW_COUNT;
        }

        // 自定义View -- onMeasure  onLayout
        // 4.5.6.7
        for (int i = bottomPosition; i < itemCount; i++) {
            // 那View --》 复用
            View view = recycler.getViewForPosition(i);

            addView(view);

            // onMeasure
            measureChildWithMargins(view, 0, 0);

            int widthSpace = getWidth() - getDecoratedMeasuredWidth(view);
            int heightSpace = getHeight() - getDecoratedMeasuredHeight(view);

            // onLayout -- 布局所有子View
            layoutDecoratedWithMargins(view, widthSpace / 2, heightSpace / 2,
                    widthSpace / 2 + getDecoratedMeasuredWidth(view),
                    heightSpace / 2 + getDecoratedMeasuredHeight(view));

            // 8-4-1  8-5-1 --> 3,2,1,0
            int level = itemCount - i - 1;

            // 对子View进行缩放平移处理
            if (level > 0) {
                // level < 3
                if (level < CardConfig.MAX_SHOW_COUNT - 1) {// 2,1
                    view.setTranslationY(CardConfig.TRANS_Y_GAP * level);
                    view.setScaleX(1 - CardConfig.SCALE_GAP * level);
                    view.setScaleY(1 - CardConfig.SCALE_GAP * level);
                } else {//3
                    // 如果是最底下那张，则效果与前一张一样
                    view.setTranslationY(CardConfig.TRANS_Y_GAP * (level - 1));
                    view.setScaleX(1 - CardConfig.SCALE_GAP * (level - 1));
                    view.setScaleY(1 - CardConfig.SCALE_GAP * (level - 1));
                }
            }
        }
    }
}

```

配合ItemTouchHelper.SimpleCallback使用上下左右滑动查看

```java
package com.leo.slidecard;

import android.graphics.Canvas;
import android.view.View;

import androidx.annotation.NonNull;
import androidx.recyclerview.widget.ItemTouchHelper;
import androidx.recyclerview.widget.RecyclerView;

import com.leo.slidecard.adapter.UniversalAdapter;

import java.util.List;

public class SlideCardCallback extends ItemTouchHelper.SimpleCallback {

    private RecyclerView mRv;
    private UniversalAdapter<SlideCardBean> adapter;
    private List<SlideCardBean> mDatas;

    public SlideCardCallback(RecyclerView mRv,
                             UniversalAdapter<SlideCardBean> adapter, List<SlideCardBean> mDatas) {
        // 设置方向，拖拽，滑动  上下左右 0x1111
        super(0, 15);//1111
        this.mRv = mRv;
        this.adapter = adapter;
        this.mDatas = mDatas;
    }

//    public SlideCardCallback(int dragDirs, int swipeDirs) {
//        super(dragDirs, swipeDirs);
//    }

    // drag 用的
    @Override
    public boolean onMove(@NonNull RecyclerView recyclerView, @NonNull RecyclerView.ViewHolder viewHolder, @NonNull RecyclerView.ViewHolder target) {
        return false;
    }

    // 动画结束后的操作；滑出上面的item，添加下面的item
    @Override
    public void onSwiped(@NonNull RecyclerView.ViewHolder viewHolder, int direction) {
        // 数据循环使用
        // 移除最上面的
        SlideCardBean remove = mDatas.remove(viewHolder.getLayoutPosition());
        // 添加到数组的第一个位置
        mDatas.add(0, remove);
        // 刷新
        adapter.notifyDataSetChanged();
    }

    // 绘制
    @Override
    public void onChildDraw(@NonNull Canvas c, @NonNull RecyclerView recyclerView,
                            @NonNull RecyclerView.ViewHolder viewHolder, float dX,
                            float dY, int actionState, boolean isCurrentlyActive) {
        super.onChildDraw(c, recyclerView, viewHolder, dX, dY, actionState, isCurrentlyActive);

        double maxDistance = recyclerView.getWidth() * 0.5f;
        double distance = Math.sqrt(dX * dX + dY * dY);
        // 放大的系数
        double fraction = distance / maxDistance;

        if (fraction > 1) {
            fraction = 1;
        }

        // 当前显示在界面的item数，0，1，2，3
        int itemCount = recyclerView.getChildCount();
        for (int i = 0; i < itemCount; i++) {
            View view = recyclerView.getChildAt(i);

            int level = itemCount - i - 1;

            // 对子View进行缩放平移处理
            if (level > 0) {
                // level < 3
                if (level < CardConfig.MAX_SHOW_COUNT - 1) {// 2,1
                    // 最大达到它的上一个Item的效果
                    view.setTranslationY((float) (CardConfig.TRANS_Y_GAP * level - fraction * CardConfig.TRANS_Y_GAP));
                    view.setScaleX((float) (1 - CardConfig.SCALE_GAP * level + fraction * CardConfig.SCALE_GAP));
                    view.setScaleY((float) (1 - CardConfig.SCALE_GAP * level + fraction * CardConfig.SCALE_GAP));
                }
            }
        }
    }

    // 回弹时间
    @Override
    public long getAnimationDuration(@NonNull RecyclerView recyclerView, int animationType, float animateDx, float animateDy) {
        return 1000;
    }

    // 回弹距离
    @Override
    public float getSwipeThreshold(@NonNull RecyclerView.ViewHolder viewHolder) {
        return 0.2f;
    }
}

```

与RecyclerView绑定

```java
// 创建 ItemTouchHelper ，必须要使用 ItemTouchHelper.Callback
ItemTouchHelper.Callback callback = new SlideCardCallback(rv, adapter, mDatas);
ItemTouchHelper helper = new ItemTouchHelper(callback);

// 绑定rv
helper.attachToRecyclerView(rv);
```





#### 优化

https://blog.csdn.net/smileiam/article/details/88396546

- 刷新时图片闪烁问题

  Recylerview的item是 ImageView 和  TextView构成，当数据改变时，我们会调用 notifyDataSetChanged，这个时候列表会刷新，为了使 url 没变的 ImageView 不重新加载（图片会一闪），我们可以用 

  setHasStableIds(true);
  使用这个，相当于给ImageView加了一个tag，tag不变的话，不用重新加载图片。但是加了这句话，会使得 列表的 数据项 重复！！ 我们需要在我们的Adapter里面重写 getItemId就好了。

  @Override
  public long getItemId(int position) {
      return position;

  }

- setHasFixedSize(true)

  当Item的高度如是固定的，设置这个属性为true可以提高性能，尤其是当RecyclerView有条目插入、删除时性能提升更明显。RecyclerView在条目数量改变，会重新测量、布局各个item，如果设置了setHasFixedSize(true)，由于item的宽高都是固定的，adapter的内容改变时，RecyclerView不会整个布局都重绘。具体可用以下伪代码表示：

  ```java
  void onItemsInsertedOrRemoved() {
     if (hasFixedSize) layoutChildren();
     else requestLayout();
  }
  ```

- 使用getExtraLayoutSpace为LayoutManager设置更多的预留空间

  在RecyclerView的元素比较高，一屏只能显示一个元素的时候，第一次滑动到第二个元素会卡顿。  

  RecyclerView (以及其他基于adapter的view，比如ListView、GridView等)使用了缓存机制重用子 view（即系统只将屏幕可见范围之内的元素保存在内存中，在滚动的时候不断的重用这些内存中已经存在的view，而不是新建view）。

  这个机制会导致一个问题，启动应用之后，在屏幕可见范围内，如果只有一张卡片可见，当滚动的时 候，RecyclerView找不到可以重用的view了，它将创建一个新的，因此在滑动到第二个feed的时候就会有一定的延时，但是第二个feed之 后的滚动是流畅的，因为这个时候RecyclerView已经有能重用的view了。

  如何解决这个问题呢，其实只需重写getExtraLayoutSpace()方法。根据官方文档的描述 getExtraLayoutSpace将返回LayoutManager应该预留的额外空间（显示范围之外，应该额外缓存的空间）。

  ```java
  LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this) {
      @Override
      protected int getExtraLayoutSpace(RecyclerView.State state) {
          return 300;
      }
  };
  ```

- 局部刷新

- 避免创建过多 OnClickListener 对象

- 复用RecycledViewPool

  在TabLayout+ViewPager+RecyclerView的场景中，当多个RecyclerView有相同的item布局结构时，多个RecyclerView共用一个RecycledViewPool可以避免创建ViewHolder的开销，避免GC。RecycledViewPool对象可通过RecyclerView对象获取，也可以自己实现。

  ```java
  RecycledViewPool mPool = mRecyclerView1.getRecycledViewPool();
  //下一个RecyclerView可直接进行setRecycledViewPool
  
  mRecyclerView2.setRecycledViewPool(mPool);
  
  mRecyclerView3.setRecycledViewPool(mPool);
  ```

- DiffUtil

https://mp.weixin.qq.com/s/auphzaQF6_wJx6dGFY6niA