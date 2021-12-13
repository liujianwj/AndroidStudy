# 提出问题

用Flutter写界面写了一段时间了，感觉很爽，尤其是热加载功能，节省了大把时间，声明式的编程方式也是以后的趋势。现在基本熟练以后一些简单的效果能很快写出来，即使没见过的也可以在网上搜一下找到答案，但是感觉没有深入底层了解，有些问题还是一知半解，这些问题比如以下几个：

1. createState 方法在什么时候调用？state 里面为啥可以直接获取到 widget 对象？
2. build 方法是在什么时候调用的？
3. BuildContext 是什么？
4. Widget 频繁更改创建是否会影响性能？复用和更新机制是什么样的？
5. 创建 Widget 里面的 Key 到底是什么作用？

后面抽时间看了一些关于 Flutter 渲染的文章，重点了解了 Widget、Element 和 RenderObject 方面的内容，终于有了一些了解，对上面几个问题也有了清晰的答案，因此通过这篇文章记录一下。

# 三棵树

首先先了解三棵树，这是我们的核心，需要首先建立一个概念。

### Widget 树

我们平时用 Widget 使用声明式的形式写出来的界面，可以理解为 Widget 树，这是要介绍的第一棵树。

### RenderObject 树

Flutter 引擎需要把我们写的 Widget 树的信息都渲染到界面上，这样人眼才能看到，跟渲染有关的当然有一颗渲染树 RenderObject tree，这是第二颗树，渲染树节点叫做 RenderObject，这个节点里面处理布局、绘制相关的事情。这两个树的节点并不是一一对应的关系，有些 Widget是要显示的，有些 Widget ，比如那些继承自 StatelessWidget & StatefulWidget 的 Widget 只是将其他 Widget 做一个组合，这些 Widget 本身并不需要显示，因此在 RenderObject 树上并没有相对应的节点。

### Element 树

Widget 树是非常不稳定的，动不动就执行 build方法，一旦调用 build 方法意味着这个 Widget 依赖的所有其他 Widget 都会重新创建，如果 Flutter 直接解析 Widget树，将其转化为 RenderObject 树来直接进行渲染，那么将会是一个非常消耗性能的过程，那对应的肯定有一个东西来消化这些变化中的不便，来做cache。因此，这里就有另外一棵树 Element 树。Element 树这一层将 Widget 树的变化（类似 React 虚拟 DOM diff）做了抽象，可以只将真正需要修改的部分同步到真实的 RenderObject 树中，最大程度降低对真实渲染视图的修改，提高渲染效率，而不是销毁整个渲染视图树重建。

这三棵树如下图所示，是我们讨论的核心内容。



![img](https://upload-images.jianshu.io/upload_images/548341-328ce76cece3eab7.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

从上图可以看出，widget 树和 Element 树节点是一一对应关系，每一个 Widget 都会有其对应的 Element，但是 RenderObject 树则不然，只有需要渲染的 Widget 才会有对应的节点。Element 树相当于一个中间层，大管家，它对 Widget 和 RenderObject 都有引用。当 Widget 不断变化的时候，将新 Widget 拿到 Element 来进行对比，看一下和之前保留的 Widget 类型和 Key 是否相同，如果都一样，那完全没有必要重新创建 Element 和 RenderObject，只需要更新里面的一些属性即可，这样可以以最小的开销更新 RenderObject，引擎在解析 RenderObject 的时候，发现只有属性修改了，那么也可以以最小的开销来做渲染。

以上只是引出了非常重要的三棵树和他们之间的关系，简而言之，Widget 树就是配置信息的树，我们平时写代码写的就是这棵树，RenderObject 树是渲染树，负责计算布局，绘制，Flutter 引擎就是根据这棵树来进行渲染的，Element 树作为中间者，管理着将 Widget 生成 RenderObject和一些更新操作。

前面只是从概念角度粗略来介绍，下面我们从源码层面来看一看。

# 从源码来了解 Widget、Element 和 RenderObject

### Widget

下面对 Widget 的概述截图来自官网



![img](https:////upload-images.jianshu.io/upload_images/548341-7f075bc985cf2856.png?imageMogr2/auto-orient/strip|imageView2/2/w/532/format/webp)

翻译一下就是，Widget 描述 Element 的配置信息，是 Flutter 框架里的核心类层次结构，一个 Widget 是用户界面某一部分的不可变描述。Widgets 可以转为 Elements，Elements 管理着底层的渲染树。

有这么多 Widget，我们来简单分各类吧，前面已经提到 Widget 有可渲染和不可渲染的分别了。可渲染里面分为多孩子和单孩子，也就是属性为 child 或 children，在不可渲染的 Widgets 里面又分为有状态和无状态，也就是 StatefullWidget 和  StatelessWidget。我们选择四个典型的Widgets来看看吧，如 Padding、RichText、Container、TextField。通过查阅源码，我们看到这几个类的继承关系如下图所示。

![img](https:////upload-images.jianshu.io/upload_images/548341-fb386ff7c0dbf142.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

来到Widget 类里面可以看到有以下方法



```cpp
  @protected
  Element createElement();
```

Widget 是个抽象类，所有的 Widgets 都是它的子类，其抽象方法 createElement 需要子类实现，这里体现了之前我们说的 Widget 和 Element 的一一对应关系。来到 StatelessWidget、StatefulWidget、MultiChildRenderObjectWidget、SingleChildRenderObjectWidget 里面我们可以找到 createElement 的实现。

SingleChildRenderObjectWidget



```csharp
@override
 SingleChildRenderObjectElement createElement() => SingleChildRenderObjectElement(this);
```

MultiChildRenderObjectWidget



```csharp
@override
MultiChildRenderObjectElement createElement() => MultiChildRenderObjectElement(this);
```

StatefulWidget



```csharp
@override
StatefulElement createElement() => StatefulElement(this);
```

StatelessWidget



```csharp
@override
StatelessElement createElement() => StatelessElement(this);
```

可以发现规律，创建 Element 都会传入 this，也就是当前 Widget，然后返回对应的 Element，这些 Element 都是继承自 Element，Element 会有引用指向当前 Widget。

我们继续来到 RichText 和 Padding 类定义里面，他们都是继承自 RenderObjectWidget，可以看到他们都有 createRenderObject 方法，如下

Padding



```csharp
  @override
  RenderPadding createRenderObject(BuildContext context) {
    return RenderPadding(
      padding: padding,
      textDirection: Directionality.of(context),
    );
  }
```

RichText



```java
@override
  RenderParagraph createRenderObject(BuildContext context) {
    assert(textDirection != null || debugCheckHasDirectionality(context));
    return RenderParagraph(text,
      textAlign: textAlign,
      textDirection: textDirection ?? Directionality.of(context),
      softWrap: softWrap,
      overflow: overflow,
      textScaleFactor: textScaleFactor,
      maxLines: maxLines,
      strutStyle: strutStyle,
      textWidthBasis: textWidthBasis,
      locale: locale ?? Localizations.localeOf(context, nullOk: true),
    );
  }
```

RenderPadding 和 RenderParagraph 最终都是继承自 RenderObject。通过以上源码分析，我们可以看出来 Widget 里面有生成 Element 和 RenderObject 的方法，所以我们平时只需要埋头写好 Widget 就行，Flutter 框架会帮我们生成对应的 Element 和 RenderObject。但是在什么时候调用 createElement 和 createRenderObject呢， 后面继续分析。

### Element

以下对 Element 描述来自官网



![img](https:////upload-images.jianshu.io/upload_images/548341-8690dd0aabad5860.png?imageMogr2/auto-orient/strip|imageView2/2/w/428/format/webp)

直接翻译过来就是，Element 是 树中特定位置 Widget 的一个实例化对象。这句话有两层意思：1. 表示 Widget 是一个配置，Element 才是最终的对象；2. Element 是通过遍历 Widget 树时，调用 Widget 的方法创建的。Element 承载了视图构建的上下文数据，是连接结构化的配置信息到完成最终渲染的桥梁。

上面从源码里面介绍 Widget 都会生成对应的 Element，这里我们也对 Element 简单做一个分类，和 Widget 相对应，如下图所示。

![img](https:////upload-images.jianshu.io/upload_images/548341-65db18b942e46010.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

首先还是进入 Element 类里面看看，这是个抽象类，可以看到一些关键的方法和属性。



```dart
  /// Typically called by an override of [Widget.createElement].
  Element(Widget widget)
    : assert(widget != null),
      _widget = widget;
```

上面介绍Widget 里面 createElement 方法的时候可以看到会传入 this，这里从 Element 的构造方法中可以看到，this 最后传给了 Element 里面的 _widget。也就是说每个 Element 里面都会有一个 Widget 的引用。_widget 在 Element 里面定义如下



```dart
  /// The configuration for this element.
  @override
  Widget get widget => _widget;
  Widget _widget;
```

从源码里面知道 Element 里面的 widget 是一个 get 方法，直接返回 _widget。从上面的注释信息也再一次提到 Widget 和 Element 的关系，Widget 是 Element 的配置。

对于 Element 的构造方法，StatelessfulElement 有一些特殊的地方，如下



```java
class StatefulElement extends ComponentElement {
  /// Creates an element that uses the given widget as its configuration.
  StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),
        super(widget) {
    ... 省略断言 ...
    assert(_state._element == null);
    _state._element = this;
     ... 省略断言 ...
    _state._widget = widget;
    assert(_state._debugLifecycleState == _StateLifecycle.created);
  }
  
  /// The [State] instance associated with this location in the tree.
  ///
  /// There is a one-to-one relationship between [State] objects and the
  /// [StatefulElement] objects that hold them. The [State] objects are created
  /// by [StatefulElement] in [mount].
  State<StatefulWidget> get state => _state;
  State<StatefulWidget> _state;
}
```

StatefulElement 的构造方法中还调用了对应 Widget 的 createState 方法，并赋值给 _state，这也解答了我们在文章开头提出的问题（createState 方法在什么时候调用？）。StatefulElement 里面不仅有对 Widget 的引用，也有对 StatefulWidget 的 State 的引用。并且在构造函数里面还将 widget 赋值给了 _state 里面的 _widget。所以我们在 State 里面可以直接使用 widget 就可以拿到 State 对应的 Widget。原来是在 StatefulElement 构造函数的时候赋值的。解释了开头提到的问题（state 里面为啥可以直接获取到 widget 对象？）。

Element 还有一个关键的方法 mount，如下



```dart
  @mustCallSuper
  void mount(Element parent, dynamic newSlot) {
    ... 省略断言 ...  
    _parent = parent;
    _slot = newSlot;
    _depth = _parent != null ? _parent.depth + 1 : 1;
    _active = true;
    if (parent != null) // Only assign ownership if the parent is non-null
      _owner = parent.owner;
    if (widget.key is GlobalKey) {
      final GlobalKey key = widget.key;
      key._register(this);
    }
    _updateInheritance();
    ... 省略断言 ...
  }
```

Flutter 框架会根据 Widget 创建对应的 Element，Element 生成以后会调用 Element 的 mount 方法，将生成的 Element 挂载到 Element 树上。这里的 createElement 和 mount 都是 Flutter 框架自动调用的，不需要开发者手动调用。因此我们平时可能没关注这些过程。Element 里面的 mount 方法需要子类实现，我们来看看ComponentElement 里的 mount 方法。



```dart
  @override
  void mount(Element parent, dynamic newSlot) {
    super.mount(parent, newSlot);
    assert(_child == null);
    assert(_active);
    _firstBuild();
    assert(_child != null);
  }
```

这里一步一步看源码，发现执行链路如下：
 _firstBuild()【ComponentElement】 -> rebuild() 【Element】-> performRebuild()【ComponentElement】 -> build()【StatelessElement】
 看一看最后 StatelessElement build() 的源码



```dart
  @override
  StatelessWidget get widget => super.widget;

  @override
  Widget build() => widget.build(this);
```

StatefulElement 的 build() 的源码如下



```csharp
  @override
  Widget build() => state.build(this);  
```

可以看出ComponentElement 的 mount 最后执行的是  build 方法。不过 StatelessElement 和 StatefulElement 是有区别的，StatelessElement 执行的是 Widget 里的 build 方法，而 StatefulElement 里面执行的是 state 的 build 方法。因此，这里也解决了文章开始提到的一个问题（build 方法是在什么时候调用的？）。也知道了 StatefulWidget 和 它的 State 是如何联系起来的。

另外，我们看到上面执行执行build 方法传递的参数 this，也就是当前 Element，而我们在写代码的时候 build 方法是这样的



```csharp
  @override
  Widget build(BuildContext context) {
  }
```

因此我们知道了，这个 BuildContext 其实就是这个 Widget 所对应的 Element。看看 Element 的定义就更清楚了。这也解释了开始提到的问题（BuildContext 是什么？）。



```dart
abstract class Element extends DiagnosticableTree implements BuildContext {
}
```

再来看看 RenderObjectElement 里的 mount 方法



```csharp
  @override
  void mount(Element parent, dynamic newSlot) {
    ... 省略断言 ...
    _renderObject = widget.createRenderObject(this);
    ... 省略断言 ...
    attachRenderObject(newSlot);
    _dirty = false;
  }
```

对比一下 ComponentElement 和 RenderObjectElement 里面的 mount 方法，前面介绍过，ComponentElement 是非渲染 Widget 对应的 Element，而 RenderObjectElement 是渲染 Widget 对应的 Element，前者的mount 方法主要是负责执行 build 方法，而后者的 mount 方法主要是调用 Widget 里面的 createRenderObject 方法生成 RenderObject，然后赋值给自己的 _renderObject。

因此可以总结，ComponentElement 的 mount 方法主要作用是执行 build，而 RenderObjectElement 的 mount 方法主要作用是生成 RenderObject。

Widget 类里面有一个很重要的静态方法，本来可以放到上面讲 Widget 的时候说，但是还是放到 Element 里面吧。就是这个



```cpp
  /// Whether the `newWidget` can be used to update an [Element] that currently
  /// has the `oldWidget` as its configuration.
  ///
  /// An element that uses a given widget as its configuration can be updated to
  /// use another widget as its configuration if, and only if, the two widgets
  /// have [runtimeType] and [key] properties that are [operator==].
  ///
  /// If the widgets have no key (their key is null), then they are considered a
  /// match if they have the same type, even if their children are completely
  /// different.
  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
```

Element 里面有一个 _widget 作为其配置信息，当widget变化或重新生成以后，Element 要不要销毁重建呢，还是直接将新生成的 Widget 替换旧的 Widget。答案就是通过这个方法判断的，上面的注释可以翻译如下

判断新 Widget 是否可以用来取代 Element 当前的配置信息 _widget。
 Element 使用特定的 widget 作为其配置信息，如果 runtimeType 和 key 和之前的 widget 相同，那么可以使用一个新的 widget 更新 Element  里面旧的 widget。
 如果这两个widget 都没有赋值 key，那么只要 runtimeType 相同也可以更新，即使这两个 widget 的孩子 widget 都完全不一样。

因此可以看出，即使外面的 widget 树经常变换重建，我们的 Element 可以维持相对稳定，不会重复创建，当然也就不会重复 mount， 生成 RenderObject，只需要以最小代价更新相关属性即可，最大可能减小了性能消耗。Widget 本身只是一些配置信息，简单的对象，它的变更重建不直接影响渲染，对性能影响很小。这就解决了上文提到的另外一个问题（Widget 频繁更改创建是否会影响性能？复用和更新机制是什么样的？）。

### RenderObject

从 RenderObject 的名字，我们就能很直观地知道，RendreObject 是主要负责实现视图渲染的对象。从上文中我们知道了一下几点

1. RenderObject 和 widget 并不是一一对应的，只有继承自 RenderObjectWidget 的 widget 才有对应的 RenderObject；
2. 生成 RenderObject 的方法 createRenderObject 是在 Widget 里面定义的；
3. 在 RenderObjectElement 执行 mount 方法的时候调用的 widget 里面的 createRenderObject 方法的；
4. RenderObjectElement 里面既有对 Widget 的引用也有对 RenderObject 的引用，它作为中间者，管理着双方。

RenderObject 在 Flutter 的展示分为四个阶段，即布局、绘制、合成和渲染。其中，布局和绘制在 RenderObject 中完成，Flutter 采用深度优先机制遍历渲染对象树，确定树中各个对象的位置和尺寸，并把它们绘制在不同的图层上。绘制完毕后，合成和渲染的工作则交给 Skia 搞定。

# 总结

上面通过源码讲解了一下 Widget、Element、RenderObject 的联系。下面简单来个总结。

我们写好 Widget 树后，Flutter 会在遍历 Widget 树时调用 Widget 里面的 createElement 方法去生成对应节点的 Element 对象，同时 Element 里面也有了对 Widget 的引用。特别的是当 StatefulElement 创建的时候也执行 StatefulWidget 里面的 createState 方法创建 state，并且赋值给 Element 里的 _state 属性，当前 widget 也同时赋值给了 state 里的_widget。Element 创建好以后 Flutter 框架会执行 mount 方法，对于非渲染的 ComponentElement 来说 mount 主要执行 widget 里的 build 方法，而对于渲染的 RenderObjectElement 来说 mount 里面会调用 widget 里面的 createRenderObject 方法 生成 RenderObject，并赋值给 RenderObjectElement 里的相应属性。StatefulElement 执行 build 方法的时候是执行的 state 里面的 build 方法，并且将自身传入，也就是 常见的 BuildContext。

如果 Widget 的配置数据发生了改变，那么持有该 Widget 的 Element 节点也会被标记为 dirty。在下一个周期的绘制时，Flutter 就会触发该 Element 树的更新，通过 canUpdate 方法来判断是否可以使用新的 Widget 来更新 Element 里面的配置，还是重新生成 Element。并使用最新的 Widget 数据更新自身以及关联的 RenderObject对象。布局和绘制完成后，接下来的事情交给 Skia 了。在 VSync 信号同步时直接从渲染树合成 Bitmap，然后提交给 GPU。

# 回答开头提出的问题

1. createState 方法在什么时候调用？state 里面为啥可以直接获取到 widget 对象？

答：Flutter 会在遍历 Widget 树时调用 Widget 里面的 createElement 方法去生成对应节点的 Element 对象，同时执行 StatefulWidget 里面的 createState 方法创建 state，并且赋值给 Element 里的 _state 属性，当前 widget 也同时赋值给了 state 里的_widget，state 里面有个 widget 的get 方法可以获取到 _widget 对象。

1. build 方法是在什么时候调用的？

答：Element 创建好以后 Flutter 框架会执行 mount 方法，对于非渲染的 ComponentElement 来说 mount 主要执行 widget 里的 build 方法，StatefulElement 执行 build 方法的时候是执行的 state 里面的 build 方法，并且将自身传入，也就是常见的 BuildContext

1. BuildContext 是什么？

答：StatefulElement 执行 build 方法的时候是执行的 state 里面的 build 方法，并且将自身传入，也就是 常见的 BuildContext。简而言之 BuidContext 就是 Element。

1. Widget 频繁更改创建是否会影响性能？复用和更新机制是什么样的？

答：不会影响性能，widget 只是简单的配置信息，并不直接涉及布局渲染相关。Element 层通过判断新旧 widget 的runtimeType 和 key 是否相同决定是否可以直接更新之前的配置信息，也就是替换之前的 widget，而不必每次都重新创建新的 Element。

1. 创建 Widget 里面的 Key 到底是什么作用？

答：Key 作为 Widget 的标志，在widget 变更的时候通过判断 Element 里面之前的 widget 的 runtimeType 和 key来决定是否能够直接更新。需要了解更多 Key 的作用可以阅读这边文章[Flutter渲染之通过demo了解Key的作用](https://www.jianshu.com/p/25aa9294acc5)。通过几个demo可以好好巩固这篇文章的知识。



作者：黑化肥发灰
链接：https://www.jianshu.com/p/71bb118517b1
来源：简书