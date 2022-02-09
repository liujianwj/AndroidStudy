正常页面结构图：

继承activity：

![继承Activity示意图](/Users/liujian/Documents/study/books/AndroidStudy/图片/继承Activity示意图.png)

继承AppCompatActivity:

![继承AppCompatActivity示意图](/Users/liujian/Documents/study/books/AndroidStudy/图片/继承AppCompatActivity示意图.png)

以继承AppCompatActivity为例，从setContextView分析源码：

```java
AppCompatActivity.setContextView()
    ->setContextView()@AppCompatDelegateImpl
      ->ensureSubDecor()
         ->createSubDecor()
            ->mWindow.getDecorView
```

此处的mWindow实现类是哪个呢？我们需要从activity创建的时候找起：
 ```java
performLaunchActivity@ActivityThread
    ->activity.attach
       ->mWindow = new PhoneWindow() @Activity
 ```

由此可见mWindow的实现类是PhoneWindow，且每一个activity都有一个PhoneWindow，在activity初始化的时候创建。

接着继续从mWindow.getDecorView开始分析，进入PhoneWindow：
```java
PhoneWindow.getDecorView
    ->installDecor()
       ->mDecor = generateDecor()
          ->new DecorView()
       ->mContentParent = generateLayout(mDecor)
          ->R.layout.screen_simple->@android:id/content->mContentParent
          // R.layout.screen_simple --》 添加到 DecorView（FrameLayout）
          ->mDecor.onResourcesLoaded(mLayoutInflater, layoutResource); 
          ->ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
```

 这样我们就拿到了contentParent，接着回到AppCompatDelegateImpl.setContentView

 ```java
public void setContentView(int resId) {
        this.ensureSubDecor();
        ViewGroup contentParent = (ViewGroup)this.mSubDecor.findViewById(16908290);
        contentParent.removeAllViews();
        LayoutInflater.from(this.mContext).inflate(resId, contentParent);
        this.mAppCompatWindowCallback.getWrapped().onContentChanged();
    }
 ```

拿到contentParent后，需要通过LayoutInflater加载XML布局，添加进contentParent

```java
// R.layout.activity_main View 创建
--> LayoutInflater.from(mContext).inflate(resId, contentParent);
	// 通过反射创建View  --- 布局的rootView
	--> final View temp = createViewFromTag(root, name, inflaterContext, attrs);
			// 是否是sdk
		--》if (-1 == name.indexOf('.')) {  // LinearLayout 
                view = onCreateView(context, parent, name, attrs);
                	--》 PhoneLayoutInflater.onCreateView(name, attrs);
                		--> View view = createView(name, prefix, attrs);

            } else { // name = androidx.constraintlayout.widget.ConstraintLayout
                view = createView(context, name, null, attrs);
                		// 通过反射创建 View 对象
                	--> clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                        mContext.getClassLoader()).asSubclass(View.class);
                	--> constructor = clazz.getConstructor(mConstructorSignature);
                	--> final View view = constructor.newInstance(args);
            }
    --> rInflateChildren(parser, temp, attrs, true); // 创建子View
    	--> rInflate
    		--> View view = createViewFromTag(parent, name, context, attrs);
```

然后在ActivityThread.performResume时，创建ViewRootImpl，然后ViewRootImpl.setView时调用requestLayout开始渲染布局

LayoutInflate的参数的作用

```java
// 方式一:将布局添加成功
View view = inflater.inflate(R.layout.inflate_layout, ll, true);

// 方式二：报错，一个View只能有一个父亲（The specified child already has a parent.）
View view = inflater.inflate(R.layout.inflate_layout, ll, true); // 已经addView
ll.addView(view);

// 方式三：布局成功，第三个参数为false
// 目的：想要 inflate_layout 的根节点的属性（宽高）有效，又不想让其处于某一个容器中
View view = inflater.inflate(R.layout.inflate_layout, ll, false);
ll.addView(view);

// 方式四：root = null，这个时候不管第三个参数是什么，显示效果一样
// inflate_layout 的根节点的属性（宽高）设置无效，只是包裹子View，
// 但是子View（Button）有效，因为Button是出于容器下的
View view = inflater.inflate(R.layout.inflate_layout, null, false);
ll.addView(view);
```

merge
1. 优化布局
2. 必须作为rootView


include
1. id 要注意
2. 不能作为 root_View

ViewStub 标签

1. 跟include 差不多  --》 隐藏作用，懒加载

面试题

---

1.为什么requestWindowFeature()要在setContentView()之前调用
	requestWindowFeature 实际调用的是 PhoneWindow.requestFeature,
	在这个方法里面会判断如果变量 mContentParentExplicitlySet 为true则报错，
	而这个变量会在 `PhoneWindow.setContentView` 调用的时候设置为true。
2. 为什么这么设计呢？
	DecorView的xml布局是通过设置的窗口特征进行选择的。
3. 为什么 requestWindowFeature(Window.FEATURE_NO_TITLE);设置无效？
	需要用 supportRequestWindowFeature(Window.FEATURE_NO_TITLE);，因为继承的是AppCompatActivity，这个类里面会覆盖设置。

---


2.LayoutInflate几个参数的作用？


LayoutInflater inflater = LayoutInflater.from(this);
// 方式一:布局添加成功，里面执行了 ll.addView(view)
View view = inflater.inflate(R.layout.inflate_layout, ll, true);

// 方式二：报错，一个View只能有一个父亲（The specified child already has a parent.）
View view = inflater.inflate(R.layout.inflate_layout, ll, true);
ll.addView(view);

// 方式三：布局成功，第三个参数为false
// 目的：想要 inflate_layout 的根节点的属性（宽高）有效，又不想让其处于某一个容器中
View view = inflater.inflate(R.layout.inflate_layout, ll, false);
ll.addView(view);

// 方式四：root = null，这个时候不管第三个参数是什么，显示效果一样
// inflate_layout 的根节点的属性（宽高）设置无效，只是包裹子View，
// 但是子View（Button）有效，因为Button是处于容器下的
View view = inflater.inflate(R.layout.inflate_layout, null, false);
ll.addView(view);

---

3.描述下merge、include、ViewStub标签的特点
include:
1. 不能作为根元素，需要放在 ViewGroup中
2. findViewById查找不到目标控件，这个问题出现的前提是在使用include时设置了id，而在findViewById时却用了被include进来的布局的根元素id。
   1. 为什么会报空指针呢？
   如果使用include标签时设置了id，这个id就会覆盖 layout根view中设置的id，从而找不到这个id
   代码：LayoutInflate.parseInclude 
   			--》final int id = a.getResourceId(R.styleable.Include_id, View.NO_ID);
   			--》if (id != View.NO_ID) {
                    view.setId(id);
                } 

merge:
1. merge标签必须使用在根布局
2. 因为merge标签并不是View,所以在通过LayoutInflate.inflate()方法渲染的时候,第二个参数必须指定一个父容器,
	且第三个参数必须为true,也就是必须为merge下的视图指定一个父亲节点.
3. 由于merge不是View所以对merge标签设置的所有属性都是无效的.

ViewStub:就是一个宽高都为0的一个View，它默认是不可见的
1. 类似include，但是一个不可见的View类，用于在运行时按需懒加载资源，只有在代码中调用了viewStub.inflate()
	或者viewStub.setVisible(View.visible)方法时才内容才变得可见。
2. 这里需要注意的一点是，当ViewStub被inflate到parent时，ViewStub就被remove掉了，即当前view hierarchy中不再存在ViewStub，
	而是使用对应的layout视图代替。