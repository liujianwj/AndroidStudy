### 滑动时的优化处理

在滑动时停止加载图片，在滑动停止时开始加载图片，这里用了Glide.pause 和Glide.resume.这里为了避免重复设置增加开销，设置了一个标志变量来做判断。

```java
mRecyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
@Override
public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
    super.onScrollStateChanged(recyclerView, newState);
    if (newState == RecyclerView.SCROLL_STATE_DRAGGING || newState == RecyclerView.SCROLL_STATE_SETTLING) {
        sIsScrolling = true;
        Glide.with(VipMasterActivity.this).pauseRequests();
    } else if (newState == RecyclerView.SCROLL_STATE_IDLE) {
        if (sIsScrolling == true) {
            Glide.with(VipMasterActivity.this).resumeRequests();
 
        }
        sIsScrolling = false;
    }
}
 
@Override
public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
    super.onScrolled(recyclerView, dx, dy);
}
});
```



### setHasFixedSize(true)

如果 Item 高度是固定的话，可以使用 `RecyclerView.setHasFixedSize(true);` 来避免 `requestLayout` 浪费资源

```java
public void setHasFixedSize(boolean hasFixedSize) {
        mHasFixedSize = hasFixedSize;
}
```

原理是，mHasFixedSizez主要用于onMeasure()方法和triggerUpdateProcessor()方法，onMeasure()这个和自定义的LayoutManager相关，先不管它，主要看triggerUpdateProcessor()方法：

```java
void triggerUpdateProcessor() {
            if (POST_UPDATES_ON_ANIMATION && mHasFixedSize && mIsAttached) {
                ViewCompat.postOnAnimation(RecyclerView.this, mUpdateChildViewsRunnable);
            } else {
                mAdapterUpdateDuringMeasure = true;
                requestLayout();
            }
}
```

看过源码，可以找到triggerUpdateProcessor()被调用的地方有：

- onItemRangeChanged()
- onItemRangeInserted()
- onItemRangeRemoved()
- onItemRangeMoved()

当调用Adapter的增删改插方法，最后就会根据mHasFixedSize这个值来判断需要不需要requestLayout()，而notifyDataSetChanged()方法最后是调用了onChanged，调用了requestLayout()，会去重新测量宽高。

总结：当我们确定Item的改变不会影响RecyclerView的宽高的时候可以设置setHasFixedSize(true)，并通过Adapter的增删改插方法去刷新RecyclerView，而不是通过notifyDataSetChanged()。（其实可以直接设置为true，当需要改变宽高的时候就用notifyDataSetChanged()去整体刷新一下）。



### 处理刷新闪烁

SetHasStableIds(true) + getItemId 





https://www.jianshu.com/p/aedb2842de30

https://www.jianshu.com/p/29352def27e6