https://juejin.cn/post/7022624191723601928

https://blog.51cto.com/u_15200109/2786117

### `MVC`架构介绍

`MVC`架构主要分为以下几部分

1. 视图层（`View`）：对应于`xml`布局文件和`java`代码动态`view`部分
2. 控制层（`Controller`）：主要负责业务逻辑，在`android`中由`Activity`承担，同时因为`XML`视图功能太弱，所以`Activity`既要负责视图的显示又要加入控制逻辑，承担的功能过多。
3. 模型层（`Model`）：主要负责网络请求，数据库处理，`I/O`的操作，即页面的数据来源

因此`MVC`架构在`android`平台上的主要存在以下问题：

1. `Activity`同时负责`View`与`Controller`层的工作，违背了单一职责原则
2. `Model`层与`View`层存在耦合，存在互相依赖，违背了最小知识原则

### `MVP`架构介绍

`MVP`架构主要分为以下几个部分

1. `View`层：对应于`Activity`与`XML`,只负责显示`UI`,只与`Presenter`层交互，与`Model`层没有耦合
2. `Presenter`层： 主要负责处理业务逻辑，通过接口回调`View`层
3. `Model`层：主要负责网络请求，数据库处理等操作，这个没有什么变化

我们可以看到，`MVP`解决了`MVC`的两个问题，即`Activity`承担了两层职责与`View`层与`Model`层耦合的问题

但`MVP`架构同样有自己的问题

1. `Presenter`层通过接口与`View`通信，实际上持有了`View`的引用
2. 但是随着业务逻辑的增加，一个页面可能会非常复杂，这样就会造成`View`的接口会很庞大。

### `MVVM`架构介绍

`MVVM` 模式将 `Presenter` 改名为 `ViewModel`，基本上与 `MVP` 模式完全一致。
 唯一的区别是，它采用双向数据绑定（`data-binding`）：`View`的变动，自动反映在 `ViewModel`，反之亦然
 `MVVM`架构图如下所示：

 可以看出`MVVM`与`MVP`的主要区别在于,你不用去主动去刷新`UI`了，只要`Model`数据变了，会自动反映到`UI`上。换句话说，`MVVM`更像是自动化的`MVP`。

`MVVM`的双向数据绑定主要通过`DataBinding`实现，不过相信有很多人跟我一样，是不喜欢用`DataBinding`的，这样架构就变成了下面这样

1. `View`观察`ViewModle`的数据变化并自我更新,这其实是单一数据源而不是双向数据绑定，所以其实`MVVM`的这一大特性我其实并没有用到
2. `View`通过调用`ViewModel`提供的方法来与`ViewMdoel`交互

### 小结

`MVC`架构的主要问题在于`Activity`承担了`View`与`Controller`两层的职责，同时`View`层与`Model`层存在耦合

`MVP`引入`Presenter`层解决了`MVC`架构的两个问题，`View`只能与`Presenter`层交互，业务逻辑放在`Presenter`层

`MVP`的问题在于随着业务逻辑的增加，`View`的接口会很庞大，`MVVM`架构通过双向数据绑定可以解决这个问题

`MVVM`与`MVP`的主要区别在于,你不用去主动去刷新`UI`了，只要`Model`数据变了，会自动反映到`UI`上。换句话说，`MVVM`更像是自动化的`MVP`。

`MVVM`的双向数据绑定主要通过`DataBinding`实现，但有很多人(比如我)不喜欢用`DataBinding`，而是`View`通过`LiveData`等观察`ViewModle`的数据变化并自我更新,这其实是单一数据源而不是双向数据绑定

### `MVVM`架构有什么不足?

要了解`MVI`架构，我们首先来了解下`MVVM`架构有什么不足
 相信使用`MVVM`架构的同学都有如下经验，为了保证数据流的单向流动，`LiveData`向外暴露时需要转化成`immutable`的，这需要添加不少模板代码并且容易遗忘，如下所示

```kotlin
class TestViewModel : ViewModel() {
    //为保证对外暴露的LiveData不可变，增加一个状态就要添加两个LiveData变量
    private val _pageState: MutableLiveData<PageState> = MutableLiveData()
    val pageState: LiveData<PageState> = _pageState
    private val _state1: MutableLiveData<String> = MutableLiveData()
    val state1: LiveData<String> = _state1
    private val _state2: MutableLiveData<String> = MutableLiveData()
    val state2: LiveData<String> = _state2
    //...
}
复制代码
```

如上所示，如果页面逻辑比较复杂，`ViewModel`中将会有许多全局变量的`LiveData`,并且每个`LiveData`都必须定义两遍，一个可变的，一个不可变的。这其实就是我通过`MVVM`架构写比较复杂页面时最难受的点。
 其次就是`View`层通过调用`ViewModel`层的方法来交互的，`View`层与`ViewModel`的交互比较分散，不成体系

小结一下，在我的使用中，`MVVM`架构主要有以下不足

1. 为保证对外暴露的`LiveData`是不可变的，需要添加不少模板代码并且容易遗忘
2. `View`层与`ViewModel`层的交互比较分散零乱，不成体系

### `MVI`架构是什么?

`MVI` 与 `MVVM` 很相似，其借鉴了前端框架的思想，更加强调数据的单向流动和唯一数据源,架构图如下所示

 其主要分为以下几部分

1. `Model`: 与`MVVM`中的`Model`不同的是，`MVI`的`Model`主要指`UI`状态（`State`）。例如页面加载状态、控件位置等都是一种`UI`状态
2. `View`: 与其他`MVX`中的`View`一致，可能是一个`Activity`或者任意`UI`承载单元。`MVI`中的`View`通过订阅`Model`的变化实现界面刷新
3. `Intent`: 此`Intent`不是`Activity`的`Intent`，用户的任何操作都被包装成`Intent`后发送给`Model`层进行数据请求

### 单向数据流

`MVI`强调数据的单向流动，主要分为以下几步：

1. 用户操作以`Intent`的形式通知`Model`
2. `Model`基于`Intent`更新`State`
3. `View`接收到`State`变化刷新UI。


