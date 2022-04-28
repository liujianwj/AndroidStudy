











#### ViewPager1懒加载问题

主要解决方法：setUserVisibleHint(boolean isVisibleToUser)

部分源码分析

```java
//在setOffscreenPageLimit、onMeasure、scroll中都会调用这个方法
ViewPager.populate()  //这里主要是通过madapter适配数据
  ->mAdapter.startUpdate(this) //开始适配
  ->addNewItem
     //创建item
     ->ii.object = this.mAdapter.instantiateItem(this, position)
         ->setUserVisibleHint(false)
  //销毁item
  ->mAdapter.destroyItem
  //设置当前显示的Item
  ->mAdapter.setPrimaryItem  
     -> mCurrentPrimaryItem.setUserVisibleHint(false);
	   -> fragment.setUserVisibleHint(true);
  //通过事务的方式，执行fragment生命周期函数
  ->mAdapter.finishUpdate(this)
```

mAdapter：FragmentStatePagerAdapter 不可以缓存，FragmentPagerAdapter 可以缓存 https://www.jianshu.com/p/44a592732501

**FragmentStatePageAdapter**：

FragmentStatePagerAdapter会销毁不需要的Fragment，一般来说，ViewHolder会保存正在显示的Fragment和它左右两边第一个Fragment，分别为A、B、C，那么当显示的Fragment变成C时，保存的Fragment就会变成B、C、D了，而A此时就会被销毁，但是需要注意的是，此时A在销毁的时候，会通过onSaveInstanceState方法来保存Fragment中的Bundle信息，当再次切换回来的时候，就可以利用保存的信息来恢复到原来的状态。

**FragmentPageAdapter**：

FragmentPageAdapter会调用事务的detach方法来处理，而不是使用remove方法。因此，FragmentPageAdapter只是销毁了Fragment的视图，其实例还是保存在FragmentManager中。

**如何选择Adapter呢**：

从上文得知，FragmentStatePageAdapter适用于Fragment较多的情况，例如月圆之夜这个卡牌游戏，需要展示数十个不同的场景，需要数十个不同的Fragment，如果这些Fragment都保存在FragmentManager中的话，对应用的性能会造成很大的影响。
 而FragmentPageAdapter则适用于固定的，少量的Fragment情况，例如和TabLayout共同使用时，典型的一个例子就是知乎上方的TabLayout。

```java
package com.leo.vp1_lazyload3.base;

import android.os.Bundle;
import android.util.Log;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;

import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.fragment.app.Fragment;
import androidx.fragment.app.FragmentManager;

import com.leo.vp1_lazyload3.FragmentDelegater;

import java.util.List;

public abstract class LazyFragment extends Fragment {

    FragmentDelegater mFragmentDelegater;
    private View rootView = null;
    private boolean isViewCreated = false;

    private boolean isVisibleStateUP = false; //  记录上一次可见的状态

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {
        E("onCreateView: ");
        if (rootView == null) {
            rootView = inflater.inflate(getLayoutRes(), container, false);
        }
        isViewCreated = true; //  解决奔溃1.1
        initView(rootView); // 初始化控件 findvxxx

        //  解决 loading 一直显示的问题
        if (getUserVisibleHint()) {
            // 手动来分发下
            setUserVisibleHint(true);
        }

        return rootView;
    }

    //  判断 Fragment 是否可见 【第一版 1.1】
    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);
        E("setUserVisibleHint：" + isVisibleToUser);

        if (isViewCreated) { //  解决崩溃1.2
            //  记录上一次可见的状态: && isVisibleStateUP
            // 现在准备显示 && 之前不可见
            if (isVisibleToUser && !isVisibleStateUP) {
                dispatchUserVisibleHint(true);
            } else if (!isVisibleToUser && isVisibleStateUP) { // 现在不准备显示 && 之前可见
                dispatchUserVisibleHint(false);
            }
        }
    }

    //  分发 可见 不可见 的动作 【第一版 1.2】
    private void dispatchUserVisibleHint(boolean visibleState) {
        //  记录上一次可见的状态: && isVisibleStateUP
        this.isVisibleStateUP = visibleState;

        if (visibleState && isParentInvisible()) {
            return;
        }

        if (visibleState) {
            // 加载数据
            onFragmentLoad();

            // 当父容器加载数据时，设置子ViewPager的Visible状态为true
            dispatchChildVisibleState(true);
        } else {
            // 停止一切操作
            onFragmentLoadStop();

            // 当父容器加载数据时，设置子ViewPager的Visible状态为false
            dispatchChildVisibleState(false);
        }
    }

    protected void dispatchChildVisibleState(boolean state) {
        FragmentManager fragmentManager = getChildFragmentManager();
        List<Fragment> fragments = fragmentManager.getFragments();
        if (fragments != null) {
            for (Fragment fragment : fragments) { // 循环遍历 嵌套里面的 子 Fragment 来分发事件操作
                if (fragment instanceof LazyFragment &&
                        !fragment.isHidden() &&
                        fragment.getUserVisibleHint()) {
                    ((LazyFragment) fragment).dispatchUserVisibleHint(state);
                }
            }
        }
    }

    // 判断父控件是否课件，如果父控件不显示，为true
    private boolean isParentInvisible() {
        Fragment parentFragment = getParentFragment();
        if (parentFragment instanceof LazyFragment) {
            LazyFragment fragment = (LazyFragment) parentFragment;
            return !fragment.isVisibleStateUP;
        }
        return false;
    }


    // 让子类完成，初始化布局，初始化控件
    protected abstract void initView(View rootView);

    protected abstract int getLayoutRes();

    // -->>>停止网络数据请求
    public void onFragmentLoadStop() {
        E("onFragmentLoadStop");
    }

    // -->>>加载网络数据请求
    public void onFragmentLoad() {
        E("onFragmentLoad");
    }

    @Override
    public void onResume() {
        super.onResume();
        E("onResume");

        //  不可见 到 可见 变化过程  说明可见
        if (getUserVisibleHint() && !isVisibleStateUP) {
            dispatchUserVisibleHint(true);
        }
    }

    @Override
    public void onPause() {
        super.onPause();
        E("onPause");

        //  可见 到 不可见  变化过程  说明 不可见
        if (getUserVisibleHint() && isVisibleStateUP) {
            dispatchUserVisibleHint(false);
        }
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        E("onDestroyView");
    }

    // 工具相关而已
    public void setFragmentDelegater(FragmentDelegater fragmentDelegater) {
        mFragmentDelegater = fragmentDelegater;
    }

    private void E(String string) {
        if (mFragmentDelegater != null) {
            mFragmentDelegater.dumpLifeCycle(string);
        }
    }
}

```

ViewPager2
1. 内部使用RecyclerView -- VP2 vs VP1  性能更好
2. adapter只支持FragmentStateAdapter
3. RecyclerView.Adapter , FragmentStateAdapter其实就是继承至RecyclerView.Adapter
4. VP2可以垂直方向滚动
5. Lifecycle 对 Fragment 的生命周期的管理
6. 默认不预加载